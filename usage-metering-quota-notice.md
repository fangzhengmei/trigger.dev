# Trigger.dev 用量计量与套餐配额告警机制

本文档基于代码分析，梳理 trigger.dev 如何统计任务运行量、按组织维度聚合、与套餐限额对比，以及在 dashboard / 邮件 / API 中展示进度和告警的完整处理流程。

---

## 一、整体架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                         用户触发任务                                  │
│                    (SDK / API / Schedule)                           │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  TriggerTaskService / TriggerTaskServiceV1                          │
│  ┌─────────────────────┐  ┌───────────────────────┐                │
│  │ 1. Entitlement 检查  │  │ 2. Queue Size 守卫    │                │
│  │    (getEntitlement)  │  │   (guardQueueSize)    │                │
│  └──────────┬──────────┘  └───────────┬───────────┘                │
│             │                         │                             │
│             ▼                         ▼                             │
│       超限 → OutOfEntitlementError  超限 → ServiceValidationError   │
│       (阻断触发)                    (阻断触发)                      │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ 通过检查
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Run Engine (内部包)                               │
│  ┌────────────────┐   ┌──────────────────┐   ┌──────────────────┐  │
│  │  BillingCache   │   │  DequeueSystem   │   │  RateLimiter     │  │
│  │  (计划缓存)     │   │  (出队调度)      │   │  (API 速率限制)  │  │
│  └───────┬────────┘   └────────┬─────────┘   └──────────────────┘  │
│          │                     │                                   │
│          ▼                     ▼                                   │
│   从 Platform API          出队时读取 BillingCache                  │
│   获取当前计划              判断 isPaying / hasPrivateLink           │
│   缓存到 Redis             附加 placementTags                      │
│   (fresh: 5min)                                                  │
│   (stale: 10min)                                                 │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│              创建 Attempt → 上报用量                                 │
│  CreateTaskRunAttemptService                                        │
│    → reportInvocationUsage(orgId, baseCostInCents)                  │
│    → POST /api/v1/usage/ingest/compute (compute 用量)               │
│    → Platform Billing API (外部计费服务)                             │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│           Platform Billing API (独立计费微服务)                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐  │
│  │ getCurrentPlan() │  │ usage()          │  │ getEntitlement()│  │
│  │ (当前套餐+限额)   │  │ (用量时序数据)   │  │ (是否有访问权)  │  │
│  └──────────────────┘  └──────────────────┘  └─────────────────┘  │
│  ┌──────────────────┐  ┌──────────────────┐                        │
│  │ reportInvocation │  │ BillingAlerts    │                        │
│  │ Usage()          │  │ (告警阈值)       │                        │
│  └──────────────────┘  └──────────────────┘                        │
│            ↕                                                        │
│        Stripe (支付与订阅)                                           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、计量周期与窗口

### 2.1 计费周期：自然月 (Calendar Month)

代码位置：[getCurrentPlan](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L221-L256)

```ts
const firstDayOfMonth = new Date();
firstDayOfMonth.setUTCDate(1);
firstDayOfMonth.setUTCHours(0, 0, 0, 0);

const firstDayOfNextMonth = new Date();
firstDayOfNextMonth.setUTCDate(1);
firstDayOfNextMonth.setUTCMonth(firstDayOfNextMonth.getUTCMonth() + 1);
firstDayOfNextMonth.setUTCHours(0, 0, 0, 0);
```

- **周期起点**：每月 1 日 UTC 00:00:00
- **周期终点**：下月 1 日 UTC 00:00:00
- **周期自动重置**：每月自动滚动，无需手动重置计数器
- **periodRemainingDuration**：当前时刻到周期结束的剩余毫秒数

### 2.2 用量聚合：按组织维度，按天窗口

代码位置：[getTaskUsageByOrganization](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/internal-packages/clickhouse/src/taskRuns.ts)

ClickHouse 查询从 `trigger_dev.task_runs_v2` 表聚合：

```sql
SELECT
  task_identifier,
  count() AS run_count,
  avg(usage_duration_ms) AS average_duration,
  sum(usage_duration_ms) AS total_duration,
  avg(cost_in_cents) / 100.0 AS average_cost,
  sum(cost_in_cents) / 100.0 AS total_cost,
  sum(base_cost_in_cents) / 100.0 AS total_base_cost
FROM trigger_dev.task_runs_v2 FINAL
WHERE
  environment_type != 'DEVELOPMENT'
  AND created_at >= fromUnixTimestamp64Milli({startTime:Int64})
  AND created_at <  fromUnixTimestamp64Milli({endTime:Int64})
  AND organization_id = {organizationId:String}
  AND _is_deleted = 0
GROUP BY task_identifier
ORDER BY total_cost DESC
```

关键点：
- **排除 DEVELOPMENT 环境**：只计算 Staging / Production 的用量
- **按 organization_id 过滤**：组织维度聚合
- **时间范围由调用方传入**：通常是当月 1 日到月末

### 2.3 Usage 时序数据

代码位置：[UsagePresenter](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/presenters/v3/UsagePresenter.server.ts#L29-L118)

- 调用 `getUsageSeries(organizationId, { from, to, window: "DAY" })` 获取按天聚合的美元花费
- 使用线性回归做月度花费预测（projected spend）
- 按任务标识符分组的费用明细来自 ClickHouse

### 2.4 缓存策略

代码位置：[platformCache](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L75-L110)

| 缓存项      | Fresh TTL | Stale TTL | 用途                    |
|------------|-----------|-----------|------------------------|
| limits     | 5 min     | 10 min    | 套餐限额                |
| usage      | 5 min     | 10 min    | 当前周期用量            |
| entitlement| 1 min     | 2 min     | 访问权限（更短，及时拦截）|

Run Engine 内部还有独立的 [BillingCache](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/internal-packages/run-engine/src/engine/billingCache.ts#L17-L18)：
- Fresh: 5 min / Stale: 10 min
- 使用 LRU 内存 + Redis 双层缓存
- 计划变更时调用 `invalidate(orgId)` 主动失效

---

## 三、任务运行计数与组织维度聚合

### 3.1 计量触发点：Attempt 创建时上报

代码位置：[CreateTaskRunAttemptService](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/services/createTaskRunAttempt.server.ts#L177-L179)

```ts
if (taskRunAttempt.number === 1 && taskRun.baseCostInCents > 0) {
  await reportInvocationUsage(environment.organizationId, taskRun.baseCostInCents, {
    runId: taskRun.id,
```

- 仅在**第一次 attempt**（非重试）时上报 `baseCostInCents`
- 上报目标：Platform Billing API 的 `reportInvocationUsage` 端点

### 3.2 Compute 用量上报

代码位置：[api.v1.usage.ingest](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/api.v1.usage.ingest.ts)

```ts
const result = await reportComputeUsage(request);
// → POST ${BILLING_API_URL}/api/v1/usage/ingest/compute
```

这是一个透传路由，将 Supervisor 发来的 compute 用量直接转发给 Platform Billing API。

### 3.3 Run 完成时更新用量

代码位置：[runAttemptSystem](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/internal-packages/run-engine/src/engine/systems/runAttemptSystem.ts#L728-L765)

```ts
const updatedUsage = this.#calculateUpdatedUsage({
  runId,
  currentUsageDurationMs: currentRun.usageDurationMs,
  currentCostInCents: currentRun.costInCents,
  attemptDurationMs: completion.usage?.durationMs ?? 0,
  machinePresetName: currentRun.machinePreset,
  environmentType: latestSnapshot.environmentType,
});
```

- Run 完成时在数据库中更新 `usageDurationMs` 和 `costInCents`
- `costInCents` 基于 machine preset 的 `centsPerMs` × 运行时长计算
- 这些数据随后由 ClickHouse 同步用于聚合查询

### 3.4 计量数据流总结

```
Task 触发 → 创建 TaskRun 记录 (DB)
    ↓
创建 Attempt #1 → reportInvocationUsage(baseCostInCents) → Platform Billing API
    ↓
Run 执行中 → Supervisor 周期上报 compute → POST /api/v1/usage/ingest/compute
    ↓
Run 完成 → 更新 taskRun.usageDurationMs / costInCents (DB)
    ↓
ClickHouse 同步 → task_runs_v2 表 → 按 org 聚合的用量查询
    ↓
UsagePresenter → UsageBar / Usage 页面展示
```

---

## 四、超限后对任务调度的影响

### 4.1 Entitlement 检查：硬性阻断

代码位置：[TriggerTaskServiceV1](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/services/triggerTaskV1.server.ts#L115-L120)

```ts
if (environment.type !== "DEVELOPMENT" && !options.skipChecks) {
  const result = await getEntitlement(environment.organizationId);
  if (result && result.hasAccess === false) {
    throw new OutOfEntitlementError();
  }
}
```

- [OutOfEntitlementError](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/services/triggerTask.server.ts#L40-L44) 消息："You can't trigger a task because you have run out of credits."
- **仅在非 DEVELOPMENT 环境检查**
- Entitlement 检查失败 → **完全阻断触发**，无法创建新 run
- 缓存策略：entitlement 缓存仅 1 min fresh / 2 min stale，确保尽快感知到权限变更

### 4.2 Entitlement 容错机制（Fail-Open）

代码位置：[getEntitlement](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L565-L598)

```ts
if (result.err || result.val === undefined) {
  return {
    hasAccess: true as const,  // 默认放行
  };
}
```

- 如果 Billing API 不可用或出错，**默认允许访问**（fail-open）
- 防止计费服务故障导致所有客户无法使用

### 4.3 Queue Size 守卫：队列满时拒绝

代码位置：[guardQueueSizeLimitsForEnv](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/queueSizeLimits.server.ts)

```ts
const maximumSize = getMaximumSizeForEnvironment(environment);
const queueSize = await marqs.lengthOfEnvQueue(environment);
const projectedSize = queueSize + itemsToAdd;
return {
  isWithinLimits: projectedSize <= maximumSize,
  maximumSize,
  queueSize,
};
```

- Dev 环境：`organization.maximumDevQueueSize ?? env.MAXIMUM_DEV_QUEUE_SIZE`
- Staging/Production：`organization.maximumDeployedQueueSize ?? env.MAXIMUM_DEPLOYED_QUEUE_SIZE`
- 超限时返回 `ServiceValidationError`："Cannot trigger ... queue size limit for this environment has been reached"

各套餐队列大小限制：

| 套餐    | Development (per queue) | Staging/Production (per queue) |
|---------|------------------------|-------------------------------|
| Free    | 500                    | 10,000                        |
| Hobby   | 500                    | 250,000                       |
| Pro     | 5,000                  | 1,000,000                     |

### 4.4 API Rate Limiting：令牌桶限流

代码位置：[authorizationRateLimitMiddleware](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/authorizationRateLimitMiddleware.server.ts)

支持三种限流算法：
- **fixedWindow**：固定窗口计数
- **slidingWindow**：滑动窗口计数
- **tokenBucket**：令牌桶（默认）

默认 API 限流：1,500 requests/min。

Dashboard Limits 页面实时显示当前 token 余量。

### 4.5 并发限制

代码位置：[getDefaultEnvironmentConcurrencyLimit](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L286-L330)

并发限制按环境类型区分：
```ts
switch (environmentType) {
  case "DEVELOPMENT": return plan.v3Subscription.plan.limits.concurrentRuns.development;
  case "STAGING":     return plan.v3Subscription.plan.limits.concurrentRuns.staging;
  case "PREVIEW":     return plan.v3Subscription.plan.limits.concurrentRuns.preview;
  case "PRODUCTION":  return plan.v3Subscription.plan.limits.concurrentRuns.production;
}
```

| 套餐  | 并发运行数 |
|-------|----------|
| Free  | 10       |
| Hobby | 25       |
| Pro   | 100+     |

### 4.6 出队调度时的计费标签

代码位置：[dequeueSystem](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/internal-packages/run-engine/src/engine/systems/dequeueSystem.ts#L523-L629)

出队时查询 `BillingCache.getCurrentPlan(orgId)`：

```ts
const billingResult = await this.options.billingCache.getCurrentPlan(orgId);
// 失败时回退到 TaskRun.planType
isPaying = (lockedTaskRun.planType ?? "free") !== "free";
// ...
placementTags: [placementTag("paid", isPaying ? "true" : "false")],
```

- 付费/免费用户会被打上不同的 placement tag
- 这影响了 Supervisor 端的执行调度策略（如不同规格的机器）

---

## 五、Dashboard / 邮件 / API 中的进度与告警

### 5.1 Dashboard：Limits 页面

代码位置：[LimitsPresenter](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/presenters/v3/LimitsPresenter.server.ts) / [Limits 路由](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.limits/route.tsx)

展示内容包括：

| 区域         | 展示项                                                                 |
|-------------|-----------------------------------------------------------------------|
| Current Plan| 套餐名称 + 升级/联系 Enterprise 按钮                                   |
| Concurrency | 当前并发限制 + 购买更多并发入口                                        |
| Rate Limits | API / Batch 速率限制 + 当前剩余 token 实时显示                         |
| Quotas      | Projects / Schedules / Team Members / Alerts / Branches / Batch Concurrency / Queue Size / Metric Dashboards / Log Retention / Query Period 等 |
| Features    | Staging 环境 / Support 级别 / Included Usage                          |

每个 Quota 项的结构：
```ts
type QuotaInfo = {
  name: string;
  description: string;
  limit: number | null;
  currentUsage: number;
  source: "default" | "plan" | "override";
  canExceed?: boolean;     // 是否可超限（软限制 vs 硬限制）
  isUpgradable?: boolean;  // 是否可通过升级套餐提升
};
```

- `canExceed: true` → 软限制，超出不阻断但会计费
- `canExceed: false` → 硬限制，超出即阻断
- 超限时 UI 显示 "You've exceeded your limit" 对话框 + Upgrade 按钮

### 5.2 Dashboard：Usage 页面

代码位置：[Usage 路由](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.usage/route.tsx)

- 按月选择器（近 6 个月）
- [UsageBar](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/UsageBar.tsx) 组件可视化展示：
  - **Used** (绿色实心)：当前已用金额
  - **Included usage / Tier limit** (深绿色半透明)：套餐包含额度
  - **Billing limit** (如果设置了)：自定义账单上限
- 按任务分组的费用明细表（来自 ClickHouse）
- 月度花费趋势图 + 线性回归预测

### 5.3 Dashboard：组织级免费额度超限提示

代码位置：[组织路由](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug/route.tsx#L97-L103)

```ts
let hasExceededFreeTier = false;
let usagePercentage = 0;
if (plan?.v3Subscription && !plan.v3Subscription.isPaying && plan.v3Subscription.plan && usage) {
  hasExceededFreeTier = usage.cents > plan.v3Subscription.plan.limits.includedUsage;
  usagePercentage = usage.cents / plan.v3Subscription.plan.limits.includedUsage;
}
```

- 免费用户超过 includedUsage 时，侧边栏会显示超限提示

### 5.4 邮件告警：Billing Alerts

代码位置：[billing-alerts 路由](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx)

两种告警类型：

**标准告警**（百分比阈值）：
- 75%、90%、100%、200%、500%

**尖峰告警**（异常飙升检测）：
- 10x (1000%)、20x (2000%)、50x (5000%)、100x (10000%)

配置项：
- **Amount**：基准金额（当月预算）
- **Emails**：接收告警的邮箱列表
- **Alert Levels**：勾选哪些阈值触发告警

数据流：
```
getBillingAlerts(orgId)  → Platform Billing API
setBillingAlert(orgId, { amount, emails, alertLevels }) → Platform Billing API
```

Billing API 在后台监控用量，达到阈值时发送邮件通知。

### 5.5 API 层面的配额信息

通过 Platform Billing Client 暴露的 API：

| 方法                      | 返回值                    | 用途                    |
|--------------------------|--------------------------|------------------------|
| `client.currentPlan()`   | 套餐信息 + limits + usage | 获取当前计划与限额      |
| `client.usage()`         | 用量数据（美元）          | 查询指定时间范围的用量  |
| `client.usageSeries()`   | 按天/小时聚合的时序数据   | 用量趋势图              |
| `client.getEntitlement()`| `{ hasAccess: boolean }`  | 检查是否有权触发任务    |
| `client.getBillingAlerts()`| 告警配置               | 获取当前告警设置        |
| `client.updateBillingAlerts()`| 更新告警配置       | 修改告警阈值和邮箱      |

---

## 六、与计费系统（Stripe）之间的数据流

### 6.1 架构：三层分离

```
┌──────────────┐     ┌───────────────────┐     ┌─────────────┐
│  Webapp      │────→│ Platform Billing   │────→│  Stripe      │
│  (Remix)     │     │ API (独立微服务)    │     │  (支付/订阅) │
└──────────────┘     └───────────────────┘     └─────────────┘
       ↑                    ↑
       │                    │
┌──────────────┐     ┌───────────────────┐
│  Run Engine  │────→│  BillingCache     │
│  (内部包)     │     │  (Redis + LRU)    │
└──────────────┘     └───────────────────┘
```

- **Webapp** 不直接与 Stripe 通信，通过 Platform Billing API 间接操作
- **Run Engine** 只通过 BillingCache 获取缓存后的计划信息，减少对 Billing API 的调用

### 6.2 Stripe 交互入口

代码位置：[platform.v3.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L338-L349)

- **Customer Portal**：`client.createPortalSession(orgId, { returnUrl })` → Stripe 托管的客户自助门户
- **套餐变更**：`client.setPlan(orgId, plan)` → 创建订阅/切换套餐
- **AddOn 购买**：`client.setAddOn(orgId, { type, amount })` → 购买额外并发/席位/分支

### 6.3 套餐变更后的缓存失效

代码位置：[setPlan](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L367-L431)

```ts
case "free_connected":
  opts?.invalidateBillingCache?.(organization.id);       // Run Engine 缓存失效
  platformCache.entitlement.remove(organization.id);     // Entitlement 缓存失效
  break;
case "updated_subscription":
  opts?.invalidateBillingCache?.(organization.id);
  platformCache.entitlement.remove(organization.id);
  break;
case "canceled_subscription":
  opts?.invalidateBillingCache?.(organization.id);
  platformCache.entitlement.remove(organization.id);
  break;
```

套餐变更后立即失效两层缓存，确保 Run Engine 下次出队时能拿到最新计划。

### 6.4 用量上报到计费

```
Attempt 创建 → reportInvocationUsage(orgId, baseCostInCents)
                    ↓
           Platform Billing API.reportInvocationUsage()
                    ↓
           Stripe Metering API (按用量计费)

Supervisor  → POST /api/v1/usage/ingest/compute
                    ↓
           Platform Billing API → Stripe Metering
```

- 每次 Run 首次 attempt 时上报 `baseCostInCents`（基础费用）
- Compute 持续用量通过 Supervisor 周期性上报
- Platform Billing API 将这些用量汇总后通过 Stripe Metering/Subscription 计费

### 6.5 仅 Cloud 环境启用

代码位置：[isCloud](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L764-L780)

```ts
export function isCloud(): boolean {
  const acceptableHosts = [
    "https://cloud.trigger.dev",
    "https://test-cloud.trigger.dev",
    "https://internal.trigger.dev",
  ];
  return acceptableHosts.includes(env.LOGIN_ORIGIN);
}
```

- 仅在 Trigger.dev Cloud 环境中初始化 `BillingClient`
- 自托管环境中 `client` 为 `undefined`，所有计费相关函数返回 `undefined`
- 自托管环境的限额通过环境变量配置（如 `MAXIMUM_DEV_QUEUE_SIZE`）

---

## 七、完整数据流图

```
                          ┌──────────────────┐
                          │   Stripe         │
                          │  (支付/订阅)     │
                          └────────┬─────────┘
                                   │ 订阅事件
                                   ▼
┌────────────┐    getCurrentPlan    ┌──────────────────┐
│  Webapp    │◄────────────────────│ Platform Billing  │
│  (Remix)   │    getUsage          │ API              │
│            │    getEntitlement    │                  │
│            │    getBillingAlerts  │  ┌─────────────┐│
│            │────────────────────►│  │用量聚合/计费 ││
│            │ reportInvocationUsage│  │Stripe Meter ││
│            │ reportComputeUsage  │  └─────────────┘│
│            │    setPlan          │                  │
│            │    setAddOn         │                  │
│            │    createPortal     │                  │
└─────┬──────┘                    └──────────────────┘
      │                                     ▲
      │ UsageBar / LimitsPage /             │
      │ BillingAlertsPage / UsagePage       │
      ▼                                     │
┌──────────────┐   BillingCache (Redis)     │
│  Run Engine  │   ┌──────────────────┐     │
│  Dequeue     │◄──│ getCurrentPlan   │─────┘
│  System      │   │ (5min fresh)     │
│              │   └──────────────────┘
│  Trigger     │
│  Guard       │─── guardQueueSizeLimitsForEnv
│              │─── getEntitlement → OutOfEntitlementError
│              │─── RateLimiter → 429 Too Many Requests
└──────────────┘
      │
      │ 出队 + placementTags(paid/free)
      ▼
┌──────────────┐
│  Supervisor   │
│  执行 Run     │ → POST /api/v1/usage/ingest/compute
│  上报用量     │
└──────────────┘
      │
      ▼
┌──────────────┐
│  ClickHouse   │
│  task_runs_v2 │ → 按 org 聚合 → UsagePresenter
└──────────────┘
```

---

## 八、关键代码索引

| 功能                  | 文件                                                              |
|----------------------|-------------------------------------------------------------------|
| 计费周期计算          | [platform.v3.server.ts#L221-L256](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L221-L256) |
| BillingCache          | [billingCache.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/internal-packages/run-engine/src/engine/billingCache.ts) |
| Entitlement 检查      | [platform.v3.server.ts#L565-L598](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L565-L598) |
| OutOfEntitlementError | [triggerTask.server.ts#L40-L44](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/services/triggerTask.server.ts#L40-L44) |
| Queue Size 守卫       | [queueSizeLimits.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/queueSizeLimits.server.ts) |
| 触发时 Entitlement 检查| [triggerTaskV1.server.ts#L115-L120](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/services/triggerTaskV1.server.ts#L115-L120) |
| 用量上报 (Invocation) | [createTaskRunAttempt.server.ts#L177-L179](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/services/createTaskRunAttempt.server.ts#L177-L179) |
| 用量上报 (Compute)    | [api.v1.usage.ingest.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/api.v1.usage.ingest.ts) |
| ClickHouse 聚合查询   | [taskRuns.ts (getTaskUsageByOrganization)](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/internal-packages/clickhouse/src/taskRuns.ts) |
| LimitsPresenter       | [LimitsPresenter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/presenters/v3/LimitsPresenter.server.ts) |
| UsageBar 组件         | [UsageBar.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/UsageBar.tsx) |
| UsagePresenter        | [UsagePresenter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/presenters/v3/UsagePresenter.server.ts) |
| Billing Alerts 页面   | [billing-alerts/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx) |
| Limits 页面           | [limits/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.limits/route.tsx) |
| Dequeue 中的计费逻辑  | [dequeueSystem.ts#L523-L629](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/internal-packages/run-engine/src/engine/systems/dequeueSystem.ts#L523-L629) |
| 并发限制读取          | [platform.v3.server.ts#L286-L330](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L286-L330) |
| Rate Limit 中间件     | [authorizationRateLimitMiddleware.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/authorizationRateLimitMiddleware.server.ts) |
| 套餐变更与缓存失效    | [platform.v3.server.ts#L367-L431](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L367-L431) |
| Run Engine 初始化配置 | [runEngine.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/runEngine.server.ts) |
| 组织级免费额度超限    | [org route.tsx#L97-L103](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug/route.tsx#L97-L103) |
