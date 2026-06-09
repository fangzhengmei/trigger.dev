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

**文件**: [api.v1.projects.$projectRef.alertChannels.ts](apps/webapp/app/routes/api.v1.projects.$projectRef.alertChannels.ts#L17-L95)

- **路由**: `POST /api/v1/projects/:projectRef/alertChannels`
- **认证**: Personal Access Token (`authenticateApiRequestWithPersonalAccessToken`)
- **请求体校验**: `ApiCreateAlertChannel` Zod schema，支持 `email` / `webhook` 两种 channel 类型
- 对于 `webhook` 类型，需要提供 `url`（必填）和 `secret`（可选）

### 2.2 服务层：CreateAlertChannelService

**文件**: [createAlertChannel.server.ts](apps/webapp/app/v3/services/alerts/createAlertChannel.server.ts#L37-L103)

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
   - **channel properties 中的 `version` 字段被硬编码为 `"v2"`**（虽然 `ProjectAlertWebhookProperties` Zod schema 中 `version` 默认值为 `"v1"`，但 `CreateAlertChannelService` 在创建时显式设为 `"v2"`）

### 2.3 数据模型：ProjectAlertChannel

**文件**: [schema.prisma](internal-packages/database/prisma/schema.prisma#L2299-L2331)

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
| Task Run 失败 | [finalizeTaskRun.server.ts](apps/webapp/app/v3/services/finalizeTaskRun.server.ts#L149) | `v3.performTaskRunAlerts` |
| Task Run 失败 (runEngine) | [runEngineHandlers.server.ts](apps/webapp/app/v3/runEngineHandlers.server.ts#L83) | `v3.performTaskRunAlerts` |
| 部署失败 | [failDeployment.server.ts](apps/webapp/app/v3/services/failDeployment.server.ts#L69) | `v3.performDeploymentAlerts` |
| 部署超时 | [timeoutDeployment.server.ts](apps/webapp/app/v3/services/timeoutDeployment.server.ts#L66) | `v3.performDeploymentAlerts` |
| 部署成功 | [finalizeDeployment.server.ts](apps/webapp/app/v3/services/finalizeDeployment.server.ts#L148) | `v3.performDeploymentAlerts` |
| 部署索引失败 | [deploymentIndexFailed.server.ts](apps/webapp/app/v3/services/deploymentIndexFailed.server.ts#L92) | `v3.performDeploymentAlerts` |
| 部署成功(Worker V3) | [createDeploymentBackgroundWorkerV3.server.ts](apps/webapp/app/v3/services/createDeploymentBackgroundWorkerV3.server.ts#L212) | `v3.performDeploymentAlerts` |
| 错误组告警 | [errorAlertEvaluator.server.ts](apps/webapp/app/v3/services/alerts/errorAlertEvaluator.server.ts) | `v3.deliverErrorGroupAlert` |

### 3.2 PerformTaskRunAlertsService

**文件**: [performTaskRunAlerts.server.ts](apps/webapp/app/v3/services/alerts/performTaskRunAlerts.server.ts#L13-L48)

1. 查找 TaskRun，获取 `projectId` 和 `runtimeEnvironment.type`
2. 在 DB 中查找匹配的 AlertChannel：`projectId` + `alertTypes has TASK_RUN` + `environmentTypes has <环境类型>` + `enabled = true`
3. **对每个匹配的 channel** 调用 `DeliverAlertService.createAndSendAlert()`

### 3.3 PerformDeploymentAlertsService

**文件**: [performDeploymentAlerts.server.ts](apps/webapp/app/v3/services/alerts/performDeploymentAlerts.server.ts#L6-L57)

逻辑类似，根据部署状态（`DEPLOYED` / 非 `DEPLOYED`）决定 alertType 为 `DEPLOYMENT_SUCCESS` 或 `DEPLOYMENT_FAILURE`。

### 3.4 错误组告警评估器

**文件**: [errorAlertEvaluator.server.ts](apps/webapp/app/v3/services/alerts/errorAlertEvaluator.server.ts)

- **周期性自调度**：通过 `selfChain()` 在每次评估结束后将自己重新入队，默认间隔 5 分钟（300,000ms）
- 查询 ClickHouse 获取活跃错误，分类为 `new_issue` / `regression` / `unignored`
- 对每个可告警的错误，为每个匹配 channel 入队 `v3.deliverErrorGroupAlert` job

---

## 4. 消息排队：Redis Worker 队列

### 4.1 AlertsWorker

**文件**: [alertsWorker.server.ts](apps/webapp/app/v3/alertsWorker.server.ts#L13-L141)

基于 `@trigger.dev/redis-worker` 的 `RedisWorker`，使用 Redis 作为消息队列。

**Job Catalog 定义**：

| Job | 用途 | visibilityTimeoutMs | retry.maxAttempts |
|-----|------|---------------------|-------------------|
| `v3.performTaskRunAlerts` | 查找匹配 channel 并创建告警 | 60,000 (60s) | 3 |
| `v3.performDeploymentAlerts` | 查找匹配 channel 并创建告警 | 60,000 (60s) | 3 |
| `v3.deliverAlert` | 实际递送告警 | 60,000 (60s) | 3 |
| `v3.evaluateErrorAlerts` | 错误组评估 | 300,000 (5min) | 3 |
| `v3.deliverErrorGroupAlert` | 递送错误组告警 | 60,000 (60s) | 3 |

**并发配置**（通过环境变量）：

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `ALERTS_WORKER_CONCURRENCY_WORKERS` | 2 | Worker 循环数 |
| `ALERTS_WORKER_CONCURRENCY_TASKS_PER_WORKER` | 10 | 每个 Worker 的任务数 |
| `ALERTS_WORKER_CONCURRENCY_LIMIT` | 50 | 总并发限制 |
| `ALERTS_WORKER_POLL_INTERVAL` | 1000ms | 空闲轮询间隔 |
| `ALERTS_WORKER_IMMEDIATE_POLL_INTERVAL` | 100ms | 有任务时轮询间隔 |

### 4.2 消息入队与去重

**文件**: [deliverAlert.server.ts](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L1266-L1335)

`DeliverAlertService.enqueue()` 使用 `alert:{alertId}` 作为去重 ID 入队，确保同一 alert 不会被重复处理。

`DeliverAlertService.createAndSendAlert()` 流程：
1. **速率限制检查**（仅对非 WEBHOOK 类型）：通过 GCRA 限流器检查
2. 在 DB 中创建 `ProjectAlert` 记录，状态为 `PENDING`
3. 入队 `v3.deliverAlert` job

### 4.3 GCRA 速率限制器

**文件**: [GCRARateLimiter.server.ts](apps/webapp/app/v3/GCRARateLimiter.server.ts)

使用 Generic Cell Rate Algorithm (GCRA) 实现，基于 Redis Lua 脚本原子执行。

配置（环境变量）：

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `ALERT_RATE_LIMITER_EMISSION_INTERVAL` | 2,500ms | 最小请求间隔 |
| `ALERT_RATE_LIMITER_BURST_TOLERANCE` | 10,000ms | 突发容忍度 |

**重要：速率限制仅对非 WEBHOOK 类型生效**（见 [deliverAlert.server.ts#L1295-L1319](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L1295-L1319)）：

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

**文件**: [deliverAlert.server.ts](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L103-L198)

1. 从 DB 加载 alert 记录，**如果状态不是 `PENDING` 则直接返回**（防止重复递送）
2. 根据 channel 类型分发：`EMAIL` / `SLACK` / `WEBHOOK`
3. 成功后更新 DB 状态为 `SENT`
4. 失败时：如果异常是 `SkipRetryError` 则静默返回（不重试）；否则**抛出异常触发 Worker 重试**

### 5.2 #deliverWebhook() — 核心递送方法

**文件**: [deliverAlert.server.ts](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L931-L970)

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
| 超时 | 5,000ms | `AbortSignal.timeout(5000)`，仅适用于出站 fetch |
| 成功判定 | `response.ok` (2xx) | 仅 2xx 视为成功 |
| 失败行为 | 抛异常 → Worker 重试 | 3xx/4xx/5xx 全部视为失败 |

### 5.3 错误组告警递送

**文件**: [deliverErrorGroupAlert.server.ts](apps/webapp/app/v3/services/alerts/deliverErrorGroupAlert.server.ts#L197-L263)

递送逻辑与 `#deliverWebhook()` 完全一致：同样的签名算法、同样的超时时间、同样的成功判定标准。

### 5.4 Webhook Payload 格式

**文件**: [webhooks.ts (core schemas)](packages/core/src/v3/schemas/webhooks.ts)

使用 Zod discriminated union 定义四种 webhook 类型：

| type 字段 | 说明 | Payload 对象 |
|-----------|------|-------------|
| `alert.run.failed` | Task Run 失败 | `AlertWebhookRunFailedObject` |
| `alert.deployment.success` | 部署成功 | `AlertWebhookDeploymentSuccessObject` |
| `alert.deployment.failed` | 部署失败 | `AlertWebhookDeploymentFailedObject` |
| `alert.error` | 错误组告警 | `AlertWebhookErrorGroupObject` |

公共字段：`id` (alert ID)、`created` (创建时间)、`webhookVersion` (硬编码为 `"v1"`)

> **关于 `webhookVersion` 字段**：在 `DeliverAlertService.#sendWebhook()` 构建 v2 格式的 payload 时，`webhookVersion` 字段被硬编码为 `"v1"`（见 [deliverAlert.server.ts#L393](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L393) 和 [#L609](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L609)）。这里的 `webhookVersion` 是 payload 中的协议版本标识，始终为 `"v1"`，与 channel properties 中的 `version` 字段（控制 payload 结构为 v1 还是 v2 格式）是两个不同的概念。

---

## 6. 签名验证

### 6.1 发送端签名生成

签名流程（见 [deliverAlert.server.ts#L931-L970](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L931-L970)）：

1. 将 payload JSON 序列化为字符串
2. 将 secret 解密
3. 使用 HMAC-SHA256 算法，以 secret 为密钥、payload 为消息，计算签名
4. 将签名结果转为十六进制字符串
5. 放入 `x-trigger-signature-hmacsha256` 请求头

### 6.2 接收端验证（SDK）

**文件**: [webhooks.ts (SDK)](packages/trigger-sdk/src/v3/webhooks.ts#L58-L109)

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

### 7.1 出站 HTTP 请求超时

**固定 5 秒超时**：`AbortSignal.timeout(5000)`

此超时仅适用于 trigger.dev **作为 HTTP 客户端**向外部 webhook URL 发起 `fetch()` 请求时（见 [deliverAlert.server.ts#L956](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L956) 和 [deliverErrorGroupAlert.server.ts#L252](apps/webapp/app/v3/services/alerts/deliverErrorGroupAlert.server.ts#L252)）。这是一个硬超时，如果集成方在 5 秒内未响应：
- 请求被中止
- 抛出异常（TypeError: signal timed out）
- Worker 捕获异常 → 触发重试

> **注意**：此 5 秒超时是**出站专用**的客户端超时，与入站路由的请求超时完全无关。trigger.dev 的 Express 服务器未设置 `requestTimeout`，入站请求的超时由调用方的 HTTP 客户端配置决定（见 [server.ts](apps/webapp/server.ts)）。

### 7.2 Worker 重试机制

**文件**: [worker.ts (redis-worker)](packages/redis-worker/src/worker.ts#L897-L993)

当 `processItem` 失败时：

1. 计算新的 attempt = 当前 attempt + 1
2. 合并 catalog 的 retry 配置与默认配置
3. 调用 `calculateNextRetryDelay(retrySettings, newAttempt)` 计算延迟
4. 如果返回 `undefined`（已达最大重试次数）→ 移入 Dead Letter Queue (DLQ)
5. 否则，以 `availableAt = now + retryDelay` 重新入队

### 7.3 退避算法

**文件**: [retries.ts](packages/core/src/v3/utils/retries.ts#L29-L44)

```ts
export function calculateNextRetryDelay(options: RetryOptions, attempt: number) {
  const opts = { ...defaultRetryOptions, ...options };
  if (attempt >= opts.maxAttempts) return;
  const random = randomize ? Math.random() + 1 : 1;
  const timeout = Math.min(maxTimeoutInMs, random * minTimeoutInMs * Math.pow(factor, attempt - 1));
  return Math.round(timeout);
}
```

**Alerts Worker 的实际重试配置**：

Alerts Worker catalog 中每个 job 仅指定了 `retry: { maxAttempts: 3 }`。Worker 的 `processItem` 在计算重试延迟时，将 catalog 的 retry 配置展开覆盖到 Worker 的 `defaultRetrySettings` 上：

```ts
const retrySettings = {
  ...defaultRetrySettings,   // Worker 内置默认值
  ...catalogItem?.retry,     // catalog 覆盖
};
```

Worker 的 `defaultRetrySettings` 定义（见 [worker.ts#L97-L105](packages/redis-worker/src/worker.ts#L97-L105)）：

| 参数 | Worker 默认值 | Alerts catalog 覆盖 | 最终生效值 |
|------|--------------|---------------------|-----------|
| maxAttempts | 12 | 3 | **3** |
| factor | 2 | (未覆盖) | **2** |
| minTimeoutInMs | 1,000 | (未覆盖) | **1,000** |
| maxTimeoutInMs | 3,600,000 (1h) | (未覆盖) | **3,600,000** |
| randomize | true | (未覆盖) | **true** |

> **注意**：`maxTimeoutInMs` 的最终生效值是 **3,600,000（1小时）**，来自 Worker 的 `defaultRetrySettings`，而非 `@trigger.dev/core` 中 `defaultRetryOptions` 的 60,000。这是因为 Worker 使用自己的 `defaultRetrySettings` 进行合并，`calculateNextRetryDelay` 内部的 `defaultRetryOptions` 会被 Worker 传入的完整 `retrySettings` 覆盖。

**实际退避时序**（理论值，实际会有 ±随机抖动）：

| Attempt | 延迟计算 | 延迟 |
|---------|---------|------|
| 1 (首次失败) | random × 1,000 × 2^0 | ~1-2s |
| 2 (第二次失败) | random × 1,000 × 2^1 | ~2-4s |
| 3 (第三次失败) | attempt(3) >= maxAttempts(3) | → DLQ（不再重试） |

> ⚠️ **关键发现**：Alerts Worker 的 retry 配置只有 `maxAttempts: 3`，这意味着加上首次尝试，总共只有 **3 次机会**。如果集成方的 webhook 端点在 3 次（约 1s + 2s + 可见性超时恢复时间）内无法恢复，事件将进入 DLQ 被永久丢弃。

---

## 8. 最终丢弃：Dead Letter Queue

### 8.1 DLQ 机制

**文件**: [queue.ts](packages/redis-worker/src/queue.ts#L328-L360)

当重试次数用尽后，消息被移入 Redis 中的 DLQ（Dead Letter Queue）：
- 存储在 Redis sorted set 中
- 保留原始消息 ID 和错误信息
- 可通过 `redriveFromDeadLetterQueue()` 重新入队

### 8.2 DLQ Redrive

**文件**: [worker.ts](packages/redis-worker/src/worker.ts#L1126-L1153)

Worker 订阅 Redis Pub/Sub channel `{name}:redrive`，收到消息后调用 `redriveFromDeadLetterQueue(id)` 将消息从 DLQ 移回主队列。

> ⚠️ **重要**：DLQ redrive 需要手动触发，没有自动扫描机制。如果集成方不知道有 DLQ 消息，这些事件将永远不会被递送。

### 8.3 ProjectAlert 状态流转

```
PENDING → SENT     (递送成功，DeliverAlertService.call() 第 192-197 行更新)
PENDING → (无变化)  (递送失败且 SkipRetryError，不进入重试)
PENDING → (无变化)  (递送失败，Worker 重试中)
PENDING → (无变化)  (重试耗尽，消息进入 DLQ)
```

> ⚠️ **关键发现**：当 Worker 重试耗尽将消息移入 DLQ 时，`DeliverAlertService.call()` 抛出的异常被 Worker 的 `processItem()` 捕获并处理（重入队或移 DLQ），但 **没有任何代码路径将 `ProjectAlert.status` 更新为 `FAILED`**。DB 中的 alert 永远停留在 `PENDING`。这导致：
> 1. 无法通过 DB 查询知道哪些 alert 递送失败
> 2. 没有 `FAILED` 状态的告警记录，集成方无法追溯丢失的事件
> 3. 没有自动补偿机制
>
> 此外，如果 DLQ 消息被手动 redrive 回主队列，Worker 会重新执行 `DeliverAlertService.call()`，该方法首先检查 `alert.status !== "PENDING"`（见 [deliverAlert.server.ts#L149](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L149)）。由于状态仍为 `PENDING`，redrive 后仍会尝试递送——这一点是正确的。但如果在 Worker 重试期间 alert 已因某种原因被更新为 `SENT`（极端情况），则 redrive 后会被跳过。

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

在 `CreateAlertChannelService` 中创建时 `version` 被硬编码为 `"v2"`，但 `ProjectAlertWebhookProperties` Zod schema 中 `version` 默认值为 `"v1"`。在 `DeliverAlertService.#sendWebhook()` 中根据 version 分发不同格式的 payload。

> **注意**：channel properties 的 `version` 字段（`"v1"` / `"v2"`）控制 payload 结构格式；而 payload 中的 `webhookVersion` 字段始终硬编码为 `"v1"`，表示协议版本。两者含义不同。

---

## 10. 丢事件风险点总结

### 🔴 高风险

| # | 风险点 | 代码位置 | 说明 |
|---|--------|----------|------|
| 1 | **重试次数仅 3 次** | [alertsWorker.server.ts#L36-L38](apps/webapp/app/v3/alertsWorker.server.ts#L36-L38) | Alert 递送 job 最多重试 3 次，退避仅 ~1s+2s。如果集成方短暂不可用（如部署中），极易丢事件 |
| 2 | **DLQ 无自动补偿** | [worker.ts#L933-L958](packages/redis-worker/src/worker.ts#L933-L958) | 进入 DLQ 的事件需要手动 redrive，无自动扫描/通知机制 |
| 3 | **ProjectAlert 状态不更新为 FAILED** | [deliverAlert.server.ts](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts) | 重试耗尽时 DB 中的 alert 永远停留在 PENDING，无法区分"正在重试"和"已丢弃" |
| 4 | **出站 fetch 超时仅 5 秒** | [deliverAlert.server.ts#L956](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L956) | `AbortSignal.timeout(5000)` 对慢速集成方不友好；此超时仅适用于 trigger.dev 出站 fetch，入站路由无此超时 |

### 🟡 中等风险

| # | 风险点 | 代码位置 | 说明 |
|---|--------|----------|------|
| 5 | **所有非 2xx 响应都触发重试** | [deliverAlert.server.ts#L959-L969](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L959-L969) | 4xx 错误（如 URL 不存在、签名验证失败）也会重试，浪费重试次数 |
| 6 | **EMAIL/SLACK 速率限制静默丢弃** | [deliverAlert.server.ts#L1296-L1309](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L1296-L1309) | 被 GCRA 限流的 alert 不创建记录，集成方完全无感知 |

### 🟢 低风险

| # | 风险点 | 代码位置 | 说明 |
|---|--------|----------|------|
| 7 | **Webhook secret 存储依赖 ENCRYPTION_KEY** | [createAlertChannel.server.ts#L125](apps/webapp/app/v3/services/alerts/createAlertChannel.server.ts#L125) | 如果 ENCRYPTION_KEY 丢失或轮换，已存储的 secret 将无法解密，webhook 递送将失败 |

---

## 11. 完整代码路径索引

### 注册路径
1. `POST /api/v1/projects/:projectRef/alertChannels` → [api.v1.projects.$projectRef.alertChannels.ts](apps/webapp/app/routes/api.v1.projects.$projectRef.alertChannels.ts)
2. → `CreateAlertChannelService.call()` → [createAlertChannel.server.ts](apps/webapp/app/v3/services/alerts/createAlertChannel.server.ts)
3. → `encryptSecret()` → DB `ProjectAlertChannel`

### 事件触发路径
4. Task Run 完成/失败 → `PerformTaskRunAlertsService.enqueue()` → [performTaskRunAlerts.server.ts](apps/webapp/app/v3/services/alerts/performTaskRunAlerts.server.ts)
5. 部署事件 → `PerformDeploymentAlertsService.enqueue()` → [performDeploymentAlerts.server.ts](apps/webapp/app/v3/services/alerts/performDeploymentAlerts.server.ts)
6. 错误组 → `ErrorAlertEvaluator.evaluate()` → [errorAlertEvaluator.server.ts](apps/webapp/app/v3/services/alerts/errorAlertEvaluator.server.ts)

### 消息排队路径
7. `DeliverAlertService.createAndSendAlert()` → GCRA 速率检查 → DB `ProjectAlert(PENDING)` → alertsWorker.enqueue()
8. alertsWorker (Redis RedisWorker) → [alertsWorker.server.ts](apps/webapp/app/v3/alertsWorker.server.ts)

### 递送路径
9. `DeliverAlertService.call()` → [deliverAlert.server.ts](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts)
10. → `#sendWebhook()` → `#deliverWebhook()` → `fetch()` with HMAC signature
11. → 成功: DB `ProjectAlert(SENT)` | 失败: 抛异常 → Worker 重试

### 重试与丢弃路径
12. Worker `processItem()` catch → `calculateNextRetryDelay()` → [retries.ts](packages/core/src/v3/utils/retries.ts)
13. 重试: 重新入队 `queue.enqueue({ availableAt: retryDate })` → [worker.ts](packages/redis-worker/src/worker.ts)
14. 耗尽: `queue.moveToDeadLetterQueue()` → DLQ → [queue.ts](packages/redis-worker/src/queue.ts)

### 签名验证路径（接收端 SDK）
15. `webhooks.constructEvent()` → [webhooks.ts (SDK)](packages/trigger-sdk/src/v3/webhooks.ts)
16. → `verifySignature()` → timing-safe 比较 → `Webhook.parse()` (Zod 校验)

---

## 12. 事实校验修订

### 12.1 原文描述修正

以下是对初版文档中与代码不符之处的逐项修正：

| # | 原文描述 | 修正 | 代码依据 |
|---|---------|------|---------|
| 1 | "公共字段：`webhookVersion` (固定 'v1')" | 表述不够准确，容易误解。`webhookVersion` 是 payload 中的**协议版本标识**，硬编码为 `"v1"`；channel properties 的 `version` 字段控制 payload 结构（v1 扁平 vs v2 嵌套），两者是不同概念 | [deliverAlert.server.ts#L393](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L393)：v2 格式 payload 中 `webhookVersion: "v1"` |
| 2 | "maxTimeoutInMs: 60,000 (默认)" | **错误**。Alerts Worker 的 `retry` 配置仅指定了 `maxAttempts: 3`，`maxTimeoutInMs` 来自 Worker 的 `defaultRetrySettings`（3,600,000 = 1小时），而非 `@trigger.dev/core` 的 `defaultRetryOptions`（60,000） | [worker.ts#L97-L105](packages/redis-worker/src/worker.ts#L97-L105)：`defaultRetrySettings = { maxTimeoutInMs: 3_600_000 }`；[worker.ts#L927-L930](packages/redis-worker/src/worker.ts#L927-L930)：`...defaultRetrySettings, ...catalogItem?.retry` |
| 3 | "Worker 可见性超时与 job 执行时间不匹配：`visibilityTimeoutMs: 60_000` 但 `v3.evaluateErrorAlerts` 的实际执行可能超过 60 秒" | **错误**。`v3.evaluateErrorAlerts` 的 `visibilityTimeoutMs` 实际为 `60_000 * 5 = 300,000`（5分钟），与错误评估的执行时间匹配，不存在不匹配风险。此条已从风险表中移除 | [alertsWorker.server.ts#L65](apps/webapp/app/v3/alertsWorker.server.ts#L65)：`visibilityTimeoutMs: 60_000 * 5` |
| 4 | "DLQ 消息重新入队后可能遇到已被标记为 SENT 的 alert" | 需要补充说明：由于进入 DLQ 时 ProjectAlert 状态仍为 PENDING（不会被更新为 FAILED），redrive 后 `DeliverAlertService` 检查 `status !== PENDING` 时仍会尝试递送。只有在极端情况下（如同时有另一条 Worker 成功递送了同一 alert），redrive 后才会遇到 SENT 状态而被跳过 | [deliverAlert.server.ts#L149](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L149) |

### 12.2 入站路径代码分析与风险适用性

trigger.dev 的入站事件处理涉及三条不同路径，它们与出站告警 Webhook 的架构完全不同。以下逐一进行代码级分析，并修正原矩阵中的错误判断。

#### 12.2.1 路径一：HTTP Endpoint / Source 回调

外部系统通过 HTTP POST 将事件推送到 trigger.dev 的回调 URL。API 速率限制白名单中包含两条相关路径（见 [apiRateLimit.server.ts#L49-L73](apps/webapp/app/services/apiRateLimit.server.ts#L49-L73)）：

- `/api/v1/http-endpoints/$httpEndpointId/env/$envType/$shortcode` — HTTP Endpoint 回调
- `/api/v1/sources/http/$id` — HTTP Source 回调
- `/api/v1/endpoints/$environmentId/$endpointSlug/index/$indexHookIdentifier` — Index Hook 回调

这三条路径均被列入 `pathWhiteList`，**完全绕过 API 速率限制**。这意味着外部系统向这些 URL 推送事件时不受 token bucket 限流约束。

> ⚠️ **重要发现**：在当前代码库中，这三个路由对应的 Remix route handler 文件**不存在**。`apiRateLimit.server.ts` 中保留了这些白名单条目，但路由文件已不再存在（可能在 run-engine 重构中被移除或合并）。这意味着当前版本中，这些回调 URL 的入站请求要么由其他机制（如反向代理、云平台专用 handler）处理，要么已经不再使用。

#### 12.2.2 路径二：Task Trigger API

**路由**: `POST /api/v1/tasks/:taskId/trigger`
**文件**: [api.v1.tasks.$taskId.trigger.ts](apps/webapp/app/routes/api.v1.tasks.$taskId.trigger.ts)

这是集成方主动触发任务执行的主要 API。代码分析如下：

**API 限流**：此路径**不在** `pathWhiteList` 中，受 API 速率限制保护。限流策略为 token bucket，参数由组织的 `apiRateLimiterConfig` 控制（见 [apiRateLimit.server.ts#L27-L45](apps/webapp/app/services/apiRateLimit.server.ts#L27-L45)）。限流不通过时返回 **429 状态码**（非静默丢弃）。

**请求幂等**：支持两层幂等机制：

1. **路由级**（`x-trigger-request-idempotency-key` header）：通过 `handleRequestIdempotency()` 检查 Redis 缓存中是否有相同 key 的已成功请求，有则直接返回缓存结果（见 [requestIdempotency.server.ts#L16-L71](apps/webapp/app/utils/requestIdempotency.server.ts#L16-L71)）
2. **服务级**（`idempotency-key` header）：通过 `IdempotencyKeyConcern.handleTriggerRequest()` 在 DB 中查找相同 idempotencyKey + taskId + environmentId 的 TaskRun，找到则返回已有 run（见 [idempotencyKeys.server.ts#L20-L79](apps/webapp/app/runEngine/concerns/idempotencyKeys.server.ts#L20-L79)）。幂等 key 有 TTL（默认 30 天），过期后 key 被清除，允许重新触发。如果已有 run 处于失败/过期等终态，key 也会被清除

**任务入队**：`TriggerTaskService.call()` 通过 `RunEngine.trigger()` 创建 TaskRun 记录并写入 run-engine 的 `RunQueue`。入队操作在同一请求中**同步完成**——如果入队成功，API 返回 200 + run ID；如果失败，API 返回 5xx。

**5 秒 fetch 超时**：**不适用**。此路径是入站 HTTP 请求（外部系统 → trigger.dev），不涉及 trigger.dev 向外发起 fetch 调用。请求处理是同步的 DB 写入 + Redis 排队，通常在毫秒级完成。

#### 12.2.3 路径三：TaskRun 的 Run-Engine 队列执行

TaskRun 创建后进入 `RunQueue`（见 [run-queue/index.ts](internal-packages/run-engine/src/run-queue/index.ts)），由 run-engine 的 Worker 异步执行。这是**trigger.dev 内部的异步执行路径**，与告警 Webhook 的 AlertsWorker 架构类似但参数差异很大。

**DLQ 与 redrive**：RunQueue 有自己的 DLQ 机制：

- 当 `nackMessage()` 被调用且 `incrementAttemptCount = true` 时，attempt 递增；如果 `attempt >= maxAttempts`，消息被移入 DLQ（见 [run-queue/index.ts#L922-L927](internal-packages/run-engine/src/run-queue/index.ts#L922-L927)）
- DLQ 消息可通过 `redriveMessage()` 重新入队（见 [run-queue/index.ts#L463-L466](internal-packages/run-engine/src/run-queue/index.ts#L463-L466)），同样基于 Redis Pub/Sub（`rq:redrive` channel）
- Redrive 时 attempt 重置为 0（见 [run-queue/index.ts#L1163](internal-packages/run-engine/src/run-queue/index.ts#L1163)），消息重新进入正常队列

**重试配置**：RunQueue 的默认重试配置（见 [run-queue/index.ts#L139-L145](internal-packages/run-engine/src/run-queue/index.ts#L139-L145)）：

| 参数 | 值 | 与 AlertsWorker 对比 |
|------|----|--------------------|
| maxAttempts | 12 | AlertsWorker: 3 |
| factor | 2 | 相同 |
| minTimeoutInMs | 1,000 | 相同 |
| maxTimeoutInMs | 3,600,000 (1h) | 相同 |
| randomize | true | 相同 |

RunQueue 的 `maxAttempts` 可通过 `RunQueueOptions.retryOptions` 覆盖，但默认为 **12 次**，远高于 AlertsWorker 的 3 次。退避时序理论上可达 ~1s → ~2s → ~4s → ~8s → ~16s → ~32s → ... → 上限 1h。

**Engine Rate Limit**：run-engine 的 `/engine/v1/worker-actions/*` 路径在 `engineRateLimiter` 中被**白名单豁免**（见 [engineRateLimit.server.ts#L28](apps/webapp/app/services/engineRateLimit.server.ts#L28)），即 Worker 与 engine 之间的通信不受速率限制。

#### 12.2.4 修正后的风险适用性矩阵

原矩阵将"入站 HTTP Endpoint"视为单一整体，忽略了三条路径的差异和 run-engine 队列执行阶段的存在。以下为修正后的逐项分析：

| # | 风险点 | 告警 Webhook（出站） | HTTP Endpoint / Source 回调 | Task Trigger API | Run-Engine 队列执行 |
|---|--------|------|------|------|------|
| 1 | 重试次数仅 3 次 | ✅ 适用 | ❌ 不适用（无 Redis Worker 队列） | ❌ 不适用（同步请求，重试由调用方控制） | ❌ **不适用**（默认 maxAttempts=12，远高于 3） |
| 2 | DLQ 无自动补偿 | ✅ 适用 | ❌ 不适用 | ❌ 不适用 | ✅ **适用**（RunQueue 同样依赖手动 redrive，无自动扫描） |
| 3 | ProjectAlert 状态不更新为 FAILED | ✅ 适用 | ❌ 不适用 | ❌ 不适用（TaskRun 持久化到 DB，状态可查询） | ❌ **不适用**（TaskRun 有完整的状态机：QUEUED → EXECUTING → COMPLETED/FAILED 等） |
| 4 | 出站 fetch 超时仅 5 秒 | ✅ 适用（trigger.dev 出站 fetch 的 `AbortSignal.timeout(5000)`） | ❌ **不适用**（入站路由无 5 秒超时，见下方说明） | ❌ **不适用**（同左，入站路由无 5 秒超时） | ❌ 不适用（内部队列处理，不涉及 HTTP fetch） |
| 5 | 所有非 2xx 响应都触发重试 | ✅ 适用 | ❌ 不适用 | ❌ 不适用 | ❌ 不适用 |
| 6 | 速率限制静默丢弃 | ✅ 适用（EMAIL/SLACK 类型） | ❌ **不适用**（白名单豁免限流） | ❌ 不适用（限流返回 429，非静默丢弃） | ❌ 不适用（Worker 通信被白名单豁免） |
| 7 | Webhook secret 存储依赖 ENCRYPTION_KEY | ✅ 适用 | ❌ 不适用 | ❌ 不适用（使用 API Key 认证） | ❌ 不适用 |

**修正要点**：

1. **原矩阵第 2 项"DLQ 无自动补偿"**：原判断为入站路径"不适用"，**部分修正**。Task Trigger API 本身确实不涉及 DLQ，但 TaskRun 创建后进入 Run-Engine 队列执行阶段时，RunQueue **同样存在 DLQ 无自动补偿的风险**。如果 run-engine 队列中的消息因执行失败耗尽重试次数进入 DLQ，同样需要手动 redrive，且**没有自动扫描/通知机制**。

2. **原矩阵第 4 项"HTTP 超时仅 5 秒"**：原判断为入站路径"适用但角色相反"，**修正为不适用**。"5 秒超时"是指 `DeliverAlertService.#deliverWebhook()` 和 `DeliverErrorGroupAlertService` 中 `fetch()` 调用的 `AbortSignal.timeout(5000)`（见 [deliverAlert.server.ts#L956](apps/webapp/app/v3/services/alerts/deliverAlert.server.ts#L956) 和 [deliverErrorGroupAlert.server.ts#L252](apps/webapp/app/v3/services/alerts/deliverErrorGroupAlert.server.ts#L252)），这是 **trigger.dev 作为 HTTP 客户端**对外发起请求时设置的超时。入站路由（HTTP Endpoint / Source 回调、Task Trigger API）中 **不存在 5 秒超时机制**：
   - Express 服务器未设置 `requestTimeout` 或 `headersTimeout`（见 [server.ts](apps/webapp/server.ts)），Node.js 18+ 默认 `requestTimeout` 为 `0`（无超时），仅设置了 `keepAliveTimeout = 65s`
   - 每个请求创建了 `AbortController`，但仅在客户端断开连接时 `abort()`（见 [server.ts#L148-L149](apps/webapp/server.ts#L148-L149)），**没有基于时间的超时**
   - 入站路由的请求处理是同步的 DB 写入 + Redis 排队，通常在毫秒级完成
   - 入站请求的超时由**调用方的 HTTP 客户端配置**决定，与 trigger.dev 的 5 秒出站超时无关

3. **原矩阵第 6 项"速率限制静默丢弃"**：对于 HTTP Endpoint / Source 回调路径，这些路径被**白名单豁免**API 限流（见 [apiRateLimit.server.ts#L57-L58](apps/webapp/app/services/apiRateLimit.server.ts#L57-L58)），因此不存在被限流静默丢弃的风险。但这同时意味着这些路径**完全没有速率保护**，如果外部系统发送大量请求，可能对 trigger.dev 造成过载。

#### 12.2.5 核心区别总结

| 维度 | 告警 Webhook（出站） | Task Trigger API（入站） | Run-Engine 队列执行 |
|------|------|------|------|
| 通信模式 | trigger.dev → 外部（异步） | 外部 → trigger.dev（同步） | 内部异步队列 |
| 消息队列 | AlertsWorker (Redis) | 无 | RunQueue (Redis) |
| 最大重试 | 3 次 | 由调用方决定 | 12 次（默认） |
| DLQ | 有，无自动补偿 | 无 | 有，无自动补偿 |
| 丢事件可追溯性 | 差（状态永远 PENDING） | 好（同步返回错误码 + 幂等支持） | 好（TaskRun 状态机完整） |
| 速率限制 | WEBHOOK 不受限，EMAIL/SLACK 静默丢弃 | 限流返回 429 | Worker 通信被白名单豁免 |
| 5 秒出站超时 | ✅ trigger.dev 出站 fetch `AbortSignal.timeout(5000)` | 无（入站路由无此超时，由调用方客户端控制） | 不涉及 HTTP |

因此，集成方反馈的"丢事件"问题需要根据方向分别排查：
- **出站告警 Webhook**：高风险点 1-3（重试仅 3 次、DLQ 无补偿、状态不更新）是直接原因
- **入站 Task Trigger API**：风险在于调用方是否正确处理了非 200 响应，trigger.dev 提供了幂等性和明确错误码
- **Run-Engine 队列执行**：虽然重试次数远高于 AlertsWorker（12 vs 3），但 DLQ 同样无自动补偿，极端情况下仍可能丢失
