# Webhook 订阅与回调全链路梳理

> 目标：从"外部系统注册 webhook 订阅"到"trigger.dev 递送事件回调"，再到"失败重试/最终丢弃"，逐环节梳理代码路径，定位可能导致"丢事件"的关键风险点。

---

## 1. 整体架构概览

trigger.dev 的 webhook 通知体系由两条主线组成：

| 方向 | 描述 |
|------|------|
| **出站（Outbound）** | trigger.dev 内部事件 → 排队 → HTTP POST 回调到集成方指定的 URL |
| **入站（Inbound）** | 外部系统发 HTTP 请求到 trigger.dev 的 HTTP Endpoint → 触发 Task Run |

本文重点梳理**出站方向**（Alert Webhook），因为"丢事件"的问题集中在此。

```
外部系统 ──注册──▶ API Route ──▶ DB (ProjectAlertChannel)
                                    │
内部事件(TaskRun完成/部署) ──▶ PerformAlertsService ──▶ DB (ProjectAlert, PENDING)
                                    │
                              alertsWorker (Redis)
                                    │
                          DeliverAlertService ──▶ #deliverWebhook()
                                    │               │
                              HTTP POST ──▶ 集成方 URL
                                    │
                          成功 → DB status=SENT
                          失败 → 重试(max 3次) → DLQ(最终丢弃)
```

---

## 2. 订阅注册：外部系统注册 Webhook 订阅

### 2.1 API 入口

**文件**: [api.v1.projects.$projectRef.alertChannels.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/routes/api.v1.projects.$projectRef.alertChannels.ts#L17-L95)

- **路由**: `POST /api/v1/projects/:projectRef/alertChannels`
- **认证**: Personal Access Token (`authenticateApiRequestWithPersonalAccessToken`)
- **请求体校验**: `ApiCreateAlertChannel` Zod schema，支持 `email` / `webhook` 两种 channel 类型
- 对于 `webhook` 类型，需要提供 `url`（必填）和 `secret`（可选）

### 2.2 服务层：CreateAlertChannelService

**文件**: [createAlertChannel.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/createAlertChannel.server.ts#L37-L103)

关键逻辑：

1. 通过 `projectRef` + `userId` 查找项目，验证权限
2. **幂等性支持**：如果提供了 `deduplicationKey` 且已存在同 key 的 channel，则**更新**而非创建
3. **Secret 处理**（第 122-126 行）：
   ```ts
   case "WEBHOOK":
     return {
       url: channel.url,
       secret: await encryptSecret(env.ENCRYPTION_KEY, channel.secret ?? nanoid()),
       version: "v2",
     };
   ```
   - 如果集成方未提供 secret，系统自动用 `nanoid()` 生成一个
   - Secret 使用 `ENCRYPTION_KEY` 加密后存储到 DB
   - **默认版本为 `v2`**（虽然 schema 中 `version` 默认值为 `"v1"`，但 `CreateAlertChannelService` 在创建时硬编码为 `"v2"`）

### 2.3 数据模型：ProjectAlertChannel

**文件**: [schema.prisma](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/internal-packages/database/prisma/schema.prisma#L2299-L2331)

```
ProjectAlertChannel {
  id                  String   @id @default(cuid())
  friendlyId          String   @unique
  deduplicationKey    String   @default(cuid())
  userProvidedDeduplicationKey Boolean @default(false)
  enabled             Boolean  @default(true)
  type                ProjectAlertChannelType  // EMAIL | SLACK | WEBHOOK
  name                String
  properties          Json     // 包含 url、加密的 secret、version
  alertTypes          ProjectAlertType[]       // TASK_RUN | DEPLOYMENT_FAILURE | DEPLOYMENT_SUCCESS | ERROR_GROUP
  environmentTypes    RuntimeEnvironmentType[] @default([STAGING, PRODUCTION])
  errorAlertConfig    Json?
  projectId           String
  @@unique([projectId, deduplicationKey])
}
```

**订阅作用域与环境隔离**：
- `projectId` 将 channel 绑定到项目
- `environmentTypes` 数组控制 channel 在哪些环境类型下生效（默认 `STAGING + PRODUCTION`）
- `alertTypes` 数组过滤 channel 关心的事件类型
- `enabled` 字段可禁用 channel

---

## 3. 事件触发：内部事件到告警创建

### 3.1 触发点一览

| 事件类型 | 触发位置 | 入队 Job |
|----------|----------|----------|
| Task Run 失败 | [finalizeTaskRun.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/finalizeTaskRun.server.ts#L149) | `v3.performTaskRunAlerts` |
| Task Run 失败 (runEngine) | [runEngineHandlers.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/runEngineHandlers.server.ts#L83) | `v3.performTaskRunAlerts` |
| 部署失败 | [failDeployment.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/failDeployment.server.ts#L69) | `v3.performDeploymentAlerts` |
| 部署超时 | [timeoutDeployment.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/timeoutDeployment.server.ts#L66) | `v3.performDeploymentAlerts` |
| 部署成功 | [finalizeDeployment.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/finalizeDeployment.server.ts#L148) | `v3.performDeploymentAlerts` |
| 部署索引失败 | [deploymentIndexFailed.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/deploymentIndexFailed.server.ts#L92) | `v3.performDeploymentAlerts` |
| 部署成功(Worker V3) | [createDeploymentBackgroundWorkerV3.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/createDeploymentBackgroundWorkerV3.server.ts#L212) | `v3.performDeploymentAlerts` |
| 错误组告警 | [errorAlertEvaluator.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/errorAlertEvaluator.server.ts) | `v3.deliverErrorGroupAlert` |

### 3.2 PerformTaskRunAlertsService

**文件**: [performTaskRunAlerts.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/performTaskRunAlerts.server.ts#L13-L48)

1. 查找 TaskRun，获取 `projectId` 和 `runtimeEnvironment.type`
2. 在 DB 中查找匹配的 AlertChannel：`projectId` + `alertTypes has TASK_RUN` + `environmentTypes has <环境类型>` + `enabled = true`
3. **对每个匹配的 channel** 调用 `DeliverAlertService.createAndSendAlert()`

### 3.3 PerformDeploymentAlertsService

**文件**: [performDeploymentAlerts.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/performDeploymentAlerts.server.ts#L6-L57)

逻辑类似，根据部署状态（`DEPLOYED` / 非 `DEPLOYED`）决定 alertType 为 `DEPLOYMENT_SUCCESS` 或 `DEPLOYMENT_FAILURE`。

### 3.4 错误组告警评估器

**文件**: [errorAlertEvaluator.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/errorAlertEvaluator.server.ts)

- **周期性自调度**：通过 `selfChain()` 在每次评估结束后将自己重新入队，默认间隔 5 分钟（300,000ms）
- 查询 ClickHouse 获取活跃错误，分类为 `new_issue` / `regression` / `unignored`
- 对每个可告警的错误，为每个匹配 channel 入队 `v3.deliverErrorGroupAlert` job

---

## 4. 消息排队：Redis Worker 队列

### 4.1 AlertsWorker

**文件**: [alertsWorker.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/alertsWorker.server.ts#L13-L141)

基于 `@trigger.dev/redis-worker` 的 `RedisWorker`，使用 Redis 作为消息队列。

**Job Catalog 定义**：

| Job | 用途 | visibilityTimeoutMs | retry.maxAttempts |
|-----|------|---------------------|-------------------|
| `v3.performTaskRunAlerts` | 查找匹配 channel 并创建告警 | 60,000 (60s) | 3 |
| `v3.performDeploymentAlerts` | 查找匹配 channel 并创建告警 | 60,000 | 3 |
| `v3.deliverAlert` | 实际递送告警 | 60,000 | 3 |
| `v3.evaluateErrorAlerts` | 错误组评估 | 300,000 (5min) | 3 |
| `v3.deliverErrorGroupAlert` | 递送错误组告警 | 60,000 | 3 |

**并发配置**（通过环境变量）：

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `ALERTS_WORKER_CONCURRENCY_WORKERS` | 2 | Worker 循环数 |
| `ALERTS_WORKER_CONCURRENCY_TASKS_PER_WORKER` | 10 | 每个 Worker 的任务数 |
| `ALERTS_WORKER_CONCURRENCY_LIMIT` | 50 | 总并发限制 |
| `ALERTS_WORKER_POLL_INTERVAL` | 1000ms | 空闲轮询间隔 |
| `ALERTS_WORKER_IMMEDIATE_POLL_INTERVAL` | 100ms | 有任务时轮询间隔 |

### 4.2 消息入队与去重

**文件**: [deliverAlert.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L1266-L1335)

`DeliverAlertService.enqueue()` 使用 `alert:{alertId}` 作为去重 ID 入队，确保同一 alert 不会被重复处理。

`DeliverAlertService.createAndSendAlert()` 流程：
1. **速率限制检查**（仅对非 WEBHOOK 类型）：通过 GCRA 限流器检查
2. 在 DB 中创建 `ProjectAlert` 记录，状态为 `PENDING`
3. 入队 `v3.deliverAlert` job

### 4.3 GCRA 速率限制器

**文件**: [GCRARateLimiter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/GCRARateLimiter.server.ts)

使用 Generic Cell Rate Algorithm (GCRA) 实现，基于 Redis Lua 脚本原子执行。

配置（环境变量）：

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `ALERT_RATE_LIMITER_EMISSION_INTERVAL` | 2,500ms | 最小请求间隔 |
| `ALERT_RATE_LIMITER_BURST_TOLERANCE` | 10,000ms | 突发容忍度 |

**重要：速率限制仅对非 WEBHOOK 类型生效**（见 [deliverAlert.server.ts#L1295-L1319](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L1295-L1319)）：

```ts
if (taskRunId && channelType !== "WEBHOOK") {
  const result = await alertsRateLimiter.check(channelId);
  if (!result.allowed) {
    return;  // 静默丢弃，不创建 alert 记录
  }
}
```

> ⚠️ **潜在丢事件风险点**：对 EMAIL/SLACK 类型，速率限制不通过时**静默返回**，不创建 ProjectAlert 记录，事件直接丢弃且无日志通知。但 WEBHOOK 类型不受此影响。

---

## 5. 对外回调：HTTP POST 递送

### 5.1 DeliverAlertService

**文件**: [deliverAlert.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L103-L198)

1. 从 DB 加载 alert 记录，**如果状态不是 `PENDING` 则直接返回**（防止重复递送）
2. 根据 channel 类型分发：`EMAIL` / `SLACK` / `WEBHOOK`
3. 成功后更新 DB 状态为 `SENT`
4. 失败时：如果异常是 `SkipRetryError` 则静默返回（不重试）；否则**抛出异常触发 Worker 重试**

### 5.2 #deliverWebhook() — 核心递送方法

**文件**: [deliverAlert.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L931-L970)

```ts
async #deliverWebhook<T>(payload: T, webhook: ProjectAlertWebhookProperties) {
  const rawPayload = JSON.stringify(payload);
  const hashPayload = Buffer.from(rawPayload, "utf-8");

  // 1. 解密 secret
  const secret = await decryptSecret(env.ENCRYPTION_KEY, webhook.secret);

  // 2. 生成 HMAC-SHA256 签名
  const hmacSecret = Buffer.from(secret, "utf-8");
  const key = await subtle.importKey("raw", hmacSecret, { name: "HMAC", hash: "SHA-256" }, false, ["sign"]);
  const signature = await subtle.sign("HMAC", key, hashPayload);
  const signatureHex = Buffer.from(signature).toString("hex");

  // 3. 发送 HTTP POST
  const response = await fetch(webhook.url, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "x-trigger-signature-hmacsha256": signatureHex,
    },
    body: rawPayload,
    signal: AbortSignal.timeout(5000),  // 5秒超时
  });

  // 4. 非 2xx 响应抛异常（触发重试）
  if (!response.ok) {
    throw new Error(`Failed to send alert webhook to ${webhook.url}`);
  }
}
```

**关键参数**：

| 参数 | 值 | 说明 |
|------|----|------|
| HTTP Method | POST | 固定 |
| Content-Type | application/json | 固定 |
| 签名头 | `x-trigger-signature-hmacsha256` | HMAC-SHA256 签名的十六进制表示 |
| 超时 | 5,000ms | `AbortSignal.timeout(5000)` |
| 成功判定 | `response.ok` (2xx) | 仅 2xx 视为成功 |
| 失败行为 | 抛异常 → Worker 重试 | 3xx/4xx/5xx 全部视为失败 |

### 5.3 错误组告警递送

**文件**: [deliverErrorGroupAlert.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/deliverErrorGroupAlert.server.ts#L197-L263)

递送逻辑与 `#deliverWebhook()` 完全一致：同样的签名算法、同样的超时时间、同样的成功判定标准。

### 5.4 Webhook Payload 格式

**文件**: [webhooks.ts (core schemas)](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/packages/core/src/v3/schemas/webhooks.ts)

使用 Zod discriminated union 定义四种 webhook 类型：

| type 字段 | 说明 | Payload 对象 |
|-----------|------|-------------|
| `alert.run.failed` | Task Run 失败 | `AlertWebhookRunFailedObject` |
| `alert.deployment.success` | 部署成功 | `AlertWebhookDeploymentSuccessObject` |
| `alert.deployment.failed` | 部署失败 | `AlertWebhookDeploymentFailedObject` |
| `alert.error` | 错误组告警 | `AlertWebhookErrorGroupObject` |

公共字段：`id` (alert ID)、`created` (创建时间)、`webhookVersion` (固定 "v1")

---

## 6. 签名验证

### 6.1 发送端签名生成

签名流程（见 [#deliverWebhook()](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L931-L970)）：

1. 将 payload JSON 序列化为字符串
2. 将 secret 解密
3. 使用 HMAC-SHA256 算法，以 secret 为密钥、payload 为消息，计算签名
4. 将签名结果转为十六进制字符串
5. 放入 `x-trigger-signature-hmacsha256` 请求头

### 6.2 接收端验证（SDK）

**文件**: [webhooks.ts (SDK)](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/packages/trigger-sdk/src/v3/webhooks.ts#L58-L109)

集成方可以使用 SDK 提供的 `webhooks.constructEvent()` 验证签名：

```ts
const event = await webhooks.constructEvent(request, "webhook_secret");
```

验证流程：
1. 从请求中提取 `x-trigger-signature-hmacsha256` 头
2. 读取请求 body 原始文本
3. 使用相同的 HMAC-SHA256 算法计算期望签名
4. 使用 **timing-safe比较** (`timingSafeEqual`) 防止时序攻击
5. 签名验证通过后，使用 `Webhook.parse()` 解析 payload（Zod 校验）

**注意**：
- 签名头缺失 → 抛出 `WebhookError("No signature header found")`
- 签名不匹配 → 抛出 `WebhookError("Invalid signature")`
- Payload 解析失败 → 抛出 `WebhookError("Webhook parsing failed: ...")`

---

## 7. 超时与退避策略

### 7.1 HTTP 请求超时

**固定 5 秒超时**：`AbortSignal.timeout(5000)`

这是一个硬超时，如果集成方在 5 秒内未响应：
- 请求被中止
- 抛出异常（TypeError: signal timed out）
- Worker 捕获异常 → 触发重试

### 7.2 Worker 重试机制

**文件**: [worker.ts (redis-worker)](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/packages/redis-worker/src/worker.ts#L897-L993)

当 `processItem` 失败时：

1. 计算新的 attempt = 当前 attempt + 1
2. 合并 catalog 的 retry 配置与默认配置
3. 调用 `calculateNextRetryDelay(retrySettings, newAttempt)` 计算延迟
4. 如果返回 `undefined`（已达最大重试次数）→ 移入 Dead Letter Queue (DLQ)
5. 否则，以 `availableAt = now + retryDelay` 重新入队

### 7.3 退避算法

**文件**: [retries.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/packages/core/src/v3/utils/retries.ts#L29-L44)

```ts
export function calculateNextRetryDelay(options: RetryOptions, attempt: number) {
  if (attempt >= opts.maxAttempts) return;
  const random = randomize ? Math.random() + 1 : 1;
  const timeout = Math.min(maxTimeoutInMs, random * minTimeoutInMs * Math.pow(factor, attempt - 1));
  return Math.round(timeout);
}
```

**Alerts Worker 的具体配置**：

| 参数 | 值 | 说明 |
|------|----|------|
| maxAttempts | 3 | 最多重试 3 次 |
| factor | 2 (默认) | 指数因子 |
| minTimeoutInMs | 1,000 (默认) | 最小延迟 1 秒 |
| maxTimeoutInMs | 60,000 (默认) | 最大延迟 60 秒 |
| randomize | true (默认) | 随机抖动 |

**实际退避时序**（理论值，实际会有 ±随机抖动）：

| Attempt | 延迟 |
|---------|------|
| 1 (首次失败) | ~1,000ms × 2^0 = 1s |
| 2 (第二次失败) | ~1,000ms × 2^1 = 2s |
| 3 (第三次失败) | → DLQ（不再重试） |

> ⚠️ **关键发现**：Alerts Worker 的 retry 配置只有 `maxAttempts: 3`，这意味着加上首次尝试，总共只有 **3 次机会**。如果集成方的 webhook 端点在 3 次（约 3 秒 + 2 秒 + 可见性超时恢复时间）内无法恢复，事件将进入 DLQ 被永久丢弃。

---

## 8. 最终丢弃：Dead Letter Queue

### 8.1 DLQ 机制

**文件**: [queue.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/packages/redis-worker/src/queue.ts#L328-L360)

当重试次数用尽后，消息被移入 Redis 中的 DLQ（Dead Letter Queue）：
- 存储在 Redis sorted set 中
- 保留原始消息 ID 和错误信息
- 可通过 `redriveFromDeadLetterQueue()` 重新入队

### 8.2 DLQ Redrive

**文件**: [worker.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/packages/redis-worker/src/worker.ts#L1126-L1153)

Worker 订阅 Redis Pub/Sub channel `{name}:redrive`，收到消息后调用 `redriveFromDeadLetterQueue(id)` 将消息从 DLQ 移回主队列。

> ⚠️ **重要**：DLQ redrive 需要手动触发，没有自动扫描机制。如果集成方不知道有 DLQ 消息，这些事件将永远不会被递送。

### 8.3 ProjectAlert 状态流转

```
PENDING → SENT     (递送成功)
PENDING → (无变化)  (递送失败且 SkipRetryError)
PENDING → (无变化)  (递送失败，Worker 重试)
         → DLQ     (重试耗尽，消息丢失)
```

> ⚠️ **关键发现**：当 Worker 重试耗尽进入 DLQ 时，**DB 中 ProjectAlert 的状态仍然是 `PENDING`**，不会更新为 `FAILED`。这导致：
> 1. 无法通过 DB 查询知道哪些 alert 递送失败
> 2. 没有 `FAILED` 状态的告警记录，集成方无法追溯丢失的事件
> 3. 没有自动补偿机制

---

## 9. 订阅作用域与环境隔离

### 9.1 多层过滤机制

告警的触发和递送经过以下多层过滤：

1. **项目级**：`ProjectAlertChannel.projectId` 绑定到项目
2. **环境类型级**：`ProjectAlertChannel.environmentTypes` 数组（`STAGING` / `PRODUCTION`）
3. **事件类型级**：`ProjectAlertChannel.alertTypes` 数组
4. **启用状态**：`ProjectAlertChannel.enabled` 字段
5. **速率限制**：GCRA 限流器（仅 EMAIL/SLACK，WEBHOOK 不受限制）

### 9.2 Branch 环境的处理

在 `PerformTaskRunAlertsService` 中，环境匹配使用 `parentEnvironment` 的类型：

```ts
environmentTypes: {
  has: run.runtimeEnvironment.parentEnvironment?.type ?? run.runtimeEnvironment.type,
}
```

这意味着 branch 环境会继承其父环境（STAGING/PRODUCTION）的告警配置。

### 9.3 版本隔离

Webhook payload 有 v1 和 v2 两个版本：
- **v1**：扁平结构，包含基本字段
- **v2**：嵌套结构，增加 `id`、`created`、`webhookVersion`、`type` 字段，payload 包裹在 `object` 中

在 `CreateAlertChannelService` 中创建时默认为 `"v2"`，但 `ProjectAlertWebhookProperties` schema 中 `version` 默认值为 `"v1"`。在 `DeliverAlertService.#sendWebhook()` 中根据 version 分发不同格式的 payload。

---

## 10. 丢事件风险点总结

### 🔴 高风险

| # | 风险点 | 代码位置 | 说明 |
|---|--------|----------|------|
| 1 | **重试次数仅 3 次** | [alertsWorker.server.ts#L36-L38](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/alertsWorker.server.ts#L36-L38) | Alert 递送 job 最多重试 3 次，退避仅 ~1s+2s。如果集成方短暂不可用（如部署中），极易丢事件 |
| 2 | **DLQ 无自动补偿** | [worker.ts#L933-L958](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/packages/redis-worker/src/worker.ts#L933-L958) | 进入 DLQ 的事件需要手动 redrive，无自动扫描/通知机制 |
| 3 | **ProjectAlert 状态不更新为 FAILED** | [deliverAlert.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/deliverAlert.server.ts) | 重试耗尽时 DB 中的 alert 永远停留在 PENDING，无法区分"正在重试"和"已丢弃" |
| 4 | **HTTP 超时仅 5 秒** | [deliverAlert.server.ts#L956](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L956) | `AbortSignal.timeout(5000)` 对慢速集成方不友好 |

### 🟡 中等风险

| # | 风险点 | 代码位置 | 说明 |
|---|--------|----------|------|
| 5 | **所有非 2xx 响应都触发重试** | [deliverAlert.server.ts#L959-L969](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L959-L969) | 4xx 错误（如 URL 不存在、签名验证失败）也会重试，浪费重试次数 |
| 6 | **EMAIL/SLACK 速率限制静默丢弃** | [deliverAlert.server.ts#L1296-L1309](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L1296-L1309) | 被 GCRA 限流的 alert 不创建记录，集成方完全无感知 |
| 7 | **Worker 可见性超时与 job 执行时间不匹配** | [alertsWorker.server.ts#L34-L35](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/alertsWorker.server.ts#L34-L35) | `visibilityTimeoutMs: 60_000` 但 `v3.evaluateErrorAlerts` 的实际执行可能超过 60 秒 |

### 🟢 低风险

| # | 风险点 | 代码位置 | 说明 |
|---|--------|----------|------|
| 8 | **无事件去重确认** | [deliverAlert.server.ts#L149](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L149) | 依赖 `status !== PENDING` 防重复递送，但 DLQ 消息重新入队后可能遇到已被标记为 SENT 的 alert |
| 9 | **Webhook secret 存储依赖 ENCRYPTION_KEY** | [createAlertChannel.server.ts#L125](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/createAlertChannel.server.ts#L125) | 如果 ENCRYPTION_KEY 丢失或轮换，已存储的 secret 将无法解密，webhook 递送将失败 |

---

## 11. 完整代码路径索引

### 注册路径
1. `POST /api/v1/projects/:projectRef/alertChannels` → [api.v1.projects.$projectRef.alertChannels.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/routes/api.v1.projects.$projectRef.alertChannels.ts)
2. → `CreateAlertChannelService.call()` → [createAlertChannel.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/createAlertChannel.server.ts)
3. → `encryptSecret()` → DB `ProjectAlertChannel`

### 事件触发路径
4. Task Run 完成/失败 → `PerformTaskRunAlertsService.enqueue()` → [performTaskRunAlerts.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/performTaskRunAlerts.server.ts)
5. 部署事件 → `PerformDeploymentAlertsService.enqueue()` → [performDeploymentAlerts.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/performDeploymentAlerts.server.ts)
6. 错误组 → `ErrorAlertEvaluator.evaluate()` → [errorAlertEvaluator.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/errorAlertEvaluator.server.ts)

### 消息排队路径
7. `DeliverAlertService.createAndSendAlert()` → GCRA 速率检查 → DB `ProjectAlert(PENDING)` → alertsWorker.enqueue()
8. alertsWorker (Redis RedisWorker) → [alertsWorker.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/alertsWorker.server.ts)

### 递送路径
9. `DeliverAlertService.call()` → [deliverAlert.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/apps/webapp/app/v3/services/alerts/deliverAlert.server.ts)
10. → `#sendWebhook()` → `#deliverWebhook()` → `fetch()` with HMAC signature
11. → 成功: DB `ProjectAlert(SENT)` | 失败: 抛异常 → Worker 重试

### 重试与丢弃路径
12. Worker `processItem()` catch → `calculateNextRetryDelay()` → [retries.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/packages/core/src/v3/utils/retries.ts)
13. 重试: 重新入队 `queue.enqueue({ availableAt: retryDate })` → [worker.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/packages/redis-worker/src/worker.ts)
14. 耗尽: `queue.moveToDeadLetterQueue()` → DLQ → [queue.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/packages/redis-worker/src/queue.ts)

### 签名验证路径（接收端 SDK）
15. `webhooks.constructEvent()` → [webhooks.ts (SDK)](file:///d:/fz/0508-3/solo-dogfeeding/code/191-trigger.dev/packages/trigger-sdk/src/v3/webhooks.ts)
16. → `verifySignature()` → timing-safe 比较 → `Webhook.parse()` (Zod 校验)
