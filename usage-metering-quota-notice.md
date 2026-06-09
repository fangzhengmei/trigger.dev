# Trigger.dev 用量计量与套餐配额告警机制

本文档基于代码分析，深入梳理 trigger.dev 在配额接近上限时的提示机制，包括 Limits 页面颜色和文案变化、Rate limit 剩余 token 的视觉显示、UsageBar 与 Billing Alerts 各自承担的提醒功能，以及这些提示与权益阻断之间的顺序关系。

---

## 一、Limits 页面：配额达到 90%/100% 时的颜色与文案变化

### 1.1 核心颜色函数 `getUsageColorClass`

代码位置：[getUsageColorClass](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.limits/route.tsx#L809-L826)

```ts
function getUsageColorClass(
  percentage: number | null,
  mode: "usage" | "remaining" = "usage"
): string {
  if (percentage === null) return "text-text-dimmed";

  if (mode === "remaining") {
    // For remaining tokens: 0 = bad (red), <=10% = warning (orange)
    if (percentage <= 0) return "text-error";
    if (percentage <= 0.1) return "text-warning";
    return "text-text-bright";
  } else {
    // For usage: 100% = bad (red), >=90% = warning (orange)
    if (percentage >= 1) return "text-error";
    if (percentage >= 0.9) return "text-warning";
    return "text-text-bright";
  }
}
```

该函数接收两个参数：
- `percentage`：0-1 范围的使用百分比
- `mode`：`"usage"` 表示使用量（越高越差），`"remaining"` 表示剩余量（越低越差）

### 1.2 Quota 行的颜色逻辑

代码位置：[QuotaRow](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.limits/route.tsx#L554-L684)

Quota 行计算 percentage 并在"Current"列应用颜色：

```ts
const percentage =
  !hideCurrentUsage && quota.limit && quota.limit > 0
    ? quota.currentUsage / quota.limit
    : null;

// 在 TableCell 上应用颜色
<TableCell
  alignment="right"
  className={cn(
    "tabular-nums",
    hideCurrentUsage ? "text-text-dimmed" : getUsageColorClass(percentage, "usage")
  )}
>
```

**颜色变化规则（Quota Usage 模式）：**

| 使用率范围        | CSS 类名           | 视觉效果  | 含义         |
|------------------|--------------------|-----------|-------------|
| < 90%            | `text-text-bright` | 亮白色    | 正常         |
| 90% – 99.99%     | `text-warning`     | 橙色/警告色 | 接近上限警告 |
| ≥ 100%           | `text-error`       | 红色/错误色 | 已达/超限    |
| 无 limit 或隐藏   | `text-text-dimmed` | 灰色      | 不适用       |

**关键发现：Limits 页面仅在 Current 列的数字文字上变色，没有进度条、背景色或其他更醒目的视觉提示。** 没有 75% 的提前预警色，只有 90% 和 100% 两个阈值。

### 1.3 不显示当前用量的 Quota

以下类型的 Quota 隐藏了当前用量，不适用颜色变化：

| Quota 名称              | 原因                                     |
|------------------------|------------------------------------------|
| Log retention          | 时长型指标，非计数                         |
| Query period           | 时长型指标，非计数                         |
| Charts per dashboard   | 每个 dashboard 不同，无法展示单一数值        |
| Max queued runs        | 实时队列大小，在 Limits 页面隐藏             |

### 1.4 Quota 超限时的文案和交互

各功能页面（非 Limits 页面）在用户尝试创建超限资源时弹出对话框：

**Schedules 页面**：[schedules/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.schedules/route.tsx#L192-L194)

```
DialogHeader: "You've exceeded your limit"
DialogDescription: "You've used {used}/{limit} of your schedules."
DialogFooter: [Upgrade] 按钮
```

**Branches 页面**：[branches/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.branches/route.tsx#L240-L246)

```ts
const requiresUpgrade =
  plan?.v3Subscription?.plan &&
  limits.used >= plan.v3Subscription.plan.limits.branches.number &&
  !plan.v3Subscription.plan.limits.branches.canExceed;

const atBranchLimit = limits.used >= limits.limit;
```

- `requiresUpgrade`（canExceed=false 且已达限额）→ 显示红色错误文案：
  ```
  "You've used all {limit} of your branches. Archive one or upgrade your plan to enable more."
  ```
- `atBranchLimit`（已达限额但 canExceed=true）→ 显示红色标题 + 普通用量文案：
  ```
  Header: "You've used {used}/{limit} of your branches" (text-error)
  ```

Branches 页面还有**环形进度图**，颜色为：
- 正常：`stroke-success`（绿色）
- 达到限额：`stroke-error`（红色）

---

## 二、Rate Limit 剩余 Token 低于 10% 的显示方式

### 2.1 Rate Limit 行的 Token 展示

代码位置：[RateLimitRow](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.limits/route.tsx#L344-L415)

```ts
const maxTokens = info.config.type === "tokenBucket"
  ? info.config.maxTokens
  : info.config.tokens;

const percentage =
  info.currentTokens !== null && maxTokens > 0
    ? info.currentTokens / maxTokens
    : null;
```

展示结构：
```
┌──────────────────────────────────────┐
│  {AnimatedNumber}  ← 当前剩余 token（带颜色）  │
│  of {maxTokens}    ← 灰色小字               │
└──────────────────────────────────────┘
```

颜色使用 `getUsageColorClass(percentage, "remaining")` 模式：

| 剩余 Token 百分比   | CSS 类名           | 视觉效果  | 含义           |
|--------------------|--------------------|-----------|---------------|
| > 10%              | `text-text-bright` | 亮白色    | 正常           |
| 1% – 10%           | `text-warning`     | 橙色/警告色 | 即将耗尽警告   |
| 0%                 | `text-error`       | 红色/错误色 | 已耗尽         |
| 无法获取            | `text-text-dimmed` | 灰色 "–"  | 不可用         |

### 2.2 Token 数值使用动画显示

代码位置：[AnimatedNumber](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.limits/route.tsx#L382)

```tsx
<AnimatedNumber value={info.currentTokens} />
```

使用 `AnimatedNumber` 组件，当 Token 数值变化时有动画过渡效果，让用户直观感知到 Token 正在消耗。

### 2.3 Limits 页面自动刷新

代码位置：[loader](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.limits/route.tsx#L97-L112)

```ts
const autoReloadPollIntervalMs = 5000;
useAutoRevalidate({ interval: data.autoReloadPollIntervalMs, onFocus: true });
```

- 每 **5 秒**自动重新请求 loader 数据
- 窗口获得焦点时也会刷新
- 确保 Rate Limit Token 余量近乎实时更新

### 2.4 Token 查询机制

代码位置：[getRateLimitRemainingTokens](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/presenters/v3/LimitsPresenter.server.ts#L458-L491)

```ts
const ratelimit = new Ratelimit({
  redis: rateLimitRedisClient,
  limiter,
  ephemeralCache: new Map(),
  analytics: false,
  prefix: `ratelimit:${keyPrefix}`,
});
const remaining = await ratelimit.getRemaining(hashedKey);
```

- 使用与 Rate Limit 中间件**相同的 Redis 实例和配置**查询剩余 Token
- API Key 经过 SHA-256 哈希后查询（与中间件一致）
- Batch Rate Limit 使用 environmentId 直接查询（不哈希）
- 查询失败返回 `null`，页面显示 "–"

---

## 三、UsageBar 承担的提醒功能

### 3.1 组件职责

代码位置：[UsageBar](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/UsageBar.tsx)

UsageBar 是一个**金额维度的进度可视化组件**，用于 Usage 页面，展示当月 compute 花费相对于套餐限额的进度。

### 3.2 三层可视化结构

```
┌─────────────────────────────────────────────────────────────────┐
│ ████████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░  │
│ ↑ Used (current)      ↑ Included usage / Tier limit            │
│     green-600/700         green-900/50                          │
│                          ↑ Billing limit (optional)             │
└─────────────────────────────────────────────────────────────────┘
```

| 层级                    | CSS 类                          | 颜色           | 含义                          |
|------------------------|---------------------------------|---------------|-------------------------------|
| Used                   | `bg-green-600` / `bg-green-700` | 亮绿/深绿      | 当前已花费金额                 |
| Included usage / Tier limit | `bg-green-900/50`          | 半透明深绿     | 套餐包含的免费额度              |
| Billing limit          | （无特定颜色类）                  | 透明/默认      | 用户自定义的账单上限            |
| Capped usage           | `bg-green-600`                  | 亮绿           | 用量在限额内的部分              |

### 3.3 超限时的颜色变化

```ts
className={cn(
  "absolute h-3 rounded-l-sm",
  tierLimit && current > tierLimit ? "bg-green-700" : "bg-green-600"
)}
```

- **未超限**（`current <= tierLimit`）：`bg-green-600`（标准绿）
- **已超限**（`current > tierLimit`）：`bg-green-700`（深绿，视觉更暗）

**关键发现：UsageBar 超限时仅从标准绿变为深绿，没有红色或橙色警告色。** 颜色变化非常微妙，不够醒目。

### 3.4 Legend 标签文案

| 标签                | 条件                   | 含义                 |
|--------------------|-----------------------|----------------------|
| "Used:"            | 始终显示               | 当前花费金额          |
| "Included usage:"  | `isPaying === true`   | 付费用户的套餐包含额度 |
| "Tier limit:"      | `isPaying === false`  | 免费用户的额度上限    |
| "Billing limit:"   | `billingLimit` 已设置  | 自定义账单上限        |

### 3.5 标签位置自适应

代码位置：[Legend](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/UsageBar.tsx#L107-L137)

```ts
const flipLegendPositionValue = 80;
const flipLegendPosition = percentage > flipLegendPositionValue ? true : false;
```

当进度条超过 80% 时，标签自动翻转到左侧显示，避免溢出右侧边界。

### 3.6 UsageBar 的局限性

| 局限点                     | 说明                                                 |
|---------------------------|------------------------------------------------------|
| 无阈值预警色               | 没有 90%/100% 的橙/红色过渡                           |
| 超限仅变色差               | green-600 → green-700 差异极小                        |
| 无文案提示                 | 不显示 "approaching limit" 或 "exceeded" 等文字       |
| 无百分比值                 | 不直接显示使用百分比                                  |
| 进度条按最大值 1.1 倍缩放  | 即使 100% 也不会充满进度条，可能让用户误以为还有余量    |

---

## 四、Billing Alerts 承担的提醒功能

### 4.1 定位与职责

代码位置：[billing-alerts/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx)

Billing Alerts 是**唯一的主动推送提醒机制**（邮件），与 UsageBar 的被动可视化形成互补。

### 4.2 标准告警阈值

```ts
const checkboxLevels = [0.75, 0.9, 1.0, 2.0, 5.0];
```

| 阈值   | 含义                    | 场景                           |
|--------|------------------------|-------------------------------|
| 75%    | 接近预算                | 早期预警，有时间调整            |
| 90%    | 即将超预算              | 紧急预警                       |
| 100%   | 达到预算                | 预算耗尽                       |
| 200%   | 超出预算 2 倍           | 超额使用                       |
| 500%   | 超出预算 5 倍           | 严重超额                       |

**100% 阈值不可取消**（`readOnly={level === 1.0}`），确保用户至少在预算耗尽时收到通知。

### 4.3 尖峰告警阈值

```ts
const spikeAlertLevels = [10.0, 20.0, 50.0, 100.0];
```

| 阈值     | 含义               | 场景                              |
|----------|-------------------|-----------------------------------|
| 10x      | 预算的 10 倍       | 异常飙升（bug 导致 runaway task）  |
| 20x      | 预算的 20 倍       | 严重异常                          |
| 50x      | 预算的 50 倍       | 极端异常                          |
| 100x     | 预算的 100 倍      | 灾难性异常                        |

默认勾选逻辑：
```ts
defaultChecked={
  alerts.alertLevels.includes(level) ||
  !spikeAlertLevels.some((l) => alerts.alertLevels.includes(l))
}
```
如果用户之前没有配置过任何尖峰告警，则默认全部勾选。

### 4.4 配置参数

| 参数       | 类型           | 说明                                    |
|-----------|---------------|-----------------------------------------|
| amount    | number ($USD)  | 基准预算金额，存储时 × 100 转为美分       |
| emails    | string[]       | 接收告警邮件的地址列表，至少一个          |
| alertLevels | number[]     | 已启用的阈值列表                         |

### 4.5 免费用户的限制

```ts
const isFree = !plan?.v3Subscription?.isPaying;
// 免费用户: amount 显示为只读文本，不可编辑
// 付费用户: amount 可编辑输入框
```

### 4.6 告警的触发机制

Billing Alerts 的阈值判定和邮件发送由 **Platform Billing API** 在后台完成：

```
用量上报 → Platform Billing API 聚合 → 对比 alert.amount × alertLevels
    → 达到阈值 → 发送邮件通知到配置的 emails
```

webapp 本身**不参与阈值判定和邮件发送**，仅负责配置的 CRUD。

---

## 五、提示与权益阻断之间的顺序关系

### 5.1 完整的"预警 → 阻断"时序

```
┌─────────────────────────────────────────────────────────────────────┐
│ 阶段 1: 被动可视化提示（Dashboard 内，用户需主动查看）               │
│                                                                     │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────┐   │
│  │ UsageBar     │   │ Limits 页面  │   │ Branches/Schedules   │   │
│  │ (Usage 页)   │   │ Quota 数字   │   │ 环形进度图+红色文案   │   │
│  │ green 变化    │   │ 颜色变化      │   │ stroke-error         │   │
│  └──────────────┘   └──────────────┘   └──────────────────────┘   │
│       ↓ 75%            ↓ 90%              ↓ 100%                   │
│   仅视觉提示          仅数字变色          弹窗+Upgrade按钮          │
└─────────────────────────────────────────────────────────────────────┘
         │
         │ 用量继续增长
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 阶段 2: 主动推送提醒（用户无需主动查看）                              │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ Billing Alerts (邮件)                                    │      │
│  │ 75% → 90% → 100% → 200% → 500% → 10x → 20x → ...      │      │
│  │ 到达阈值自动发送邮件到配置的邮箱                           │      │
│  └──────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
         │
         │ 用量继续增长（超免费额度）
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 阶段 3: 全局顶部横幅提示（所有页面可见）                              │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ UpgradePrompt (NavBar 内嵌)                              │      │
│  │ 红色背景 + ExclamationCircleIcon + 错误色文字             │      │
│  │ "You have exceeded the monthly $X free credits.          │      │
│  │  Existing runs will be queued and new runs won't be      │      │
│  │  created until {next month}, or you upgrade."            │      │
│  │ + [Upgrade] 按钮                                         │      │
│  └──────────────────────────────────────────────────────────┘      │
│  触发条件: hasExceededFreeTier === true (仅免费用户)                 │
│  显示位置: 替换 NavBar 下方的 EnvironmentBanner                     │
└─────────────────────────────────────────────────────────────────────┘
         │
         │ 忽略横幅，继续尝试触发任务
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 阶段 4: API 层面权益阻断（硬阻断）                                    │
│                                                                     │
│  触发任务时的检查顺序：                                               │
│  1. Entitlement 检查 → hasAccess === false → OutOfEntitlementError  │
│     "You can't trigger a task because you have run out of credits." │
│  2. Queue Size 守卫 → 超限 → ServiceValidationError                │
│     "Cannot trigger ... queue size limit has been reached"          │
│  3. Rate Limit → Token 耗尽 → HTTP 429 Too Many Requests           │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 UpgradePrompt 的触发逻辑

代码位置：[UpgradePrompt](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/UpgradePrompt.tsx#L11-L47)

```ts
if (!plan || !plan.v3Usage.hasExceededFreeTier) {
  return null;
}
```

触发条件计算：

代码位置：[org route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug/route.tsx#L115-L120)

```ts
hasExceededFreeTier = usage.cents > plan.v3Subscription.plan.limits.includedUsage;
```

- **仅对免费用户**（`!plan.v3Subscription.isPaying`）计算
- 当 `usage.cents > includedUsage` 时触发
- **没有 90% 的提前预警**，只有超过 100% 后才出现

显示位置：

代码位置：[NavBar](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/primitives/PageHeader.tsx#L17-L28)

```ts
const showUpgradePrompt = useShowUpgradePrompt(organization);
// ...
{showUpgradePrompt.shouldShow && organization
  ? <UpgradePrompt />
  : <EnvironmentBanner />
}
```

- UpgradePrompt **替换**了 EnvironmentBanner（环境标签），出现在 NavBar 下方
- 一旦显示，**所有使用 NavBar 的页面**都会看到此横幅

### 5.3 Entitlement 阻断的精确时机

代码位置：[triggerTaskV1.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/services/triggerTaskV1.server.ts#L115-L142)

触发任务时的检查顺序：

```
Step 1: 检查幂等键缓存命中 → 命中则跳过后续检查
Step 2: Entitlement 检查 (仅非 DEV 环境)
        → hasAccess === false → 抛出 OutOfEntitlementError (阻断)
Step 3: Queue Size 守卫 (所有环境)
        → 超限 → 抛出 ServiceValidationError (阻断)
Step 4: Tags 数量检查
        → 超过 MAX_TAGS_PER_RUN → 抛出 ServiceValidationError (阻断)
Step 5: 创建 TaskRun → 进入队列
```

Batch 触发也遵循相同顺序：

代码位置：[batchTriggerV3.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/services/batchTriggerV3.server.ts#L199-L258)

```
Step 1: 检查 parent run 状态
Step 2: Entitlement 检查 (仅非 DEV 环境)
Step 3: Queue Size 守卫 (考虑批量数量 newRunCount)
Step 4: 准备 run 数据 → 创建批量 TaskRun
```

### 5.4 阻断的容错策略

Entitlement 检查使用 **Fail-Open** 策略：

```ts
// getEntitlement 返回 undefined 或出错时
if (result.err || result.val === undefined) {
  return { hasAccess: true as const }; // 默认放行
}
```

这意味着：
- 如果 Platform Billing API 宕机 → **不阻断**用户
- 如果缓存读取失败 → **不阻断**用户
- 只有明确返回 `hasAccess: false` 时才阻断

### 5.5 DEV 环境的豁免

```ts
if (environment.type !== "DEVELOPMENT" && !options.skipChecks) {
  const result = await getEntitlement(environment.organizationId);
  // ...
}
```

- DEVELOPMENT 环境**跳过** Entitlement 检查
- 即使超限，开发环境仍可触发任务
- Queue Size 守卫仍然生效（`!options.skipChecks` 条件）

---

## 六、各提示机制的对比与差距分析

### 6.1 功能对比

| 提示机制            | 触发时机          | 展示位置         | 通知方式   | 颜色预警          | 文案提示      |
|--------------------|------------------|-----------------|-----------|------------------|-------------|
| Limits 页面 Quota  | 用户主动查看      | Limits 页表格    | 被动      | 90% 橙 / 100% 红 | 无           |
| Limits 页面 Rate   | 用户主动查看      | Limits 页表格    | 被动      | ≤10% 橙 / 0% 红  | 无           |
| UsageBar           | 用户主动查看      | Usage 页进度条   | 被动      | 仅绿色深浅变化     | 无           |
| Billing Alerts     | 自动触发          | 邮件             | 主动推送   | 不适用            | 阈值百分比    |
| UpgradePrompt 横幅 | 超免费额度        | 全局 NavBar 下方  | 被动(可见) | 红色背景+图标      | 明确阻断文案  |
| 功能页面 Dialog     | 尝试创建超限资源  | Schedules/Branches | 被动(交互) | 红色文案          | 超限对话框    |
| Entitlement 阻断   | API 触发任务时    | API 响应         | 主动阻断   | 不适用            | Error 消息   |

### 6.2 当前提示不够明确的方面

| 问题                        | 详细说明                                                               |
|-----------------------------|-----------------------------------------------------------------------|
| UsageBar 无阈值预警色        | 超限时仅 green-600 → green-700，视觉差异极小，缺乏橙色/红色过渡         |
| UsageBar 无文字提示          | 不显示 "approaching limit" 或 "exceeded limit" 等文案                   |
| Limits 页面仅数字变色        | 没有背景色高亮、进度条、图标等更醒目的视觉提示                           |
| 无 75% 提前预警（页面内）     | Limits 页面从正常直接跳到 90% 橙色，缺少 75% 的早期预警                |
| UpgradePrompt 仅限免费用户    | 付费用户超限时没有类似的顶部横幅提醒                                     |
| 无接近上限的 in-app 通知      | 没有在侧边栏、toast 或通知中心显示"即将达到上限"的提示                   |
| Rate limit 低 token 无补充提示 | 剩余 token 低于 10% 时仅数字变橙色，没有建议使用 batchTrigger 的提示     |

---

## 七、关键代码索引

| 功能                        | 文件                                                                                                              |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------|
| 颜色判定函数                | [getUsageColorClass](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.limits/route.tsx#L809-L826) |
| Quota 行颜色应用            | [QuotaRow](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.limits/route.tsx#L554-L684) |
| Rate Limit 行颜色应用       | [RateLimitRow](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.limits/route.tsx#L344-L415) |
| Limits 页面自动刷新         | [autoReloadPollIntervalMs](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.limits/route.tsx#L97-L112) |
| Token 剩余查询              | [getRateLimitRemainingTokens](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/presenters/v3/LimitsPresenter.server.ts#L458-L491) |
| UsageBar 组件               | [UsageBar](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/UsageBar.tsx) |
| UsageBar 超限颜色逻辑       | [UsageBar L52-L54](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/UsageBar.tsx#L52-L54) |
| UsageBar Legend 翻转        | [Legend L108-L109](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/UsageBar.tsx#L108-L109) |
| Billing Alerts 页面         | [billing-alerts/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx) |
| 标准告警阈值                | [checkboxLevels](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L185) |
| 尖峰告警阈值                | [spikeAlertLevels](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L187) |
| 100% 阈值不可取消           | [readOnly level 1.0](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L273) |
| UpgradePrompt 横幅          | [UpgradePrompt](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/UpgradePrompt.tsx#L11-L47) |
| NavBar 中 UpgradePrompt 位置| [NavBar](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/primitives/PageHeader.tsx#L17-L28) |
| hasExceededFreeTier 计算    | [org route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug/route.tsx#L115-L120) |
| Entitlement 阻断            | [triggerTaskV1.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/services/triggerTaskV1.server.ts#L115-L120) |
| Batch Entitlement 阻断      | [batchTriggerV3.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/services/batchTriggerV3.server.ts#L199-L203) |
| Entitlement Fail-Open       | [getEntitlement](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L565-L598) |
| Queue Size 守卫             | [queueSizeLimits.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/queueSizeLimits.server.ts) |
| Branches 超限文案+环形图     | [branches/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.branches/route.tsx#L240-L246) |
| Schedules 超限对话框         | [schedules/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.schedules/route.tsx#L192-L194) |
| LimitsPresenter 数据层       | [LimitsPresenter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/presenters/v3/LimitsPresenter.server.ts) |
| Usage 页面（UsageBar 使用处）| [usage/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.usage/route.tsx#L155-L163) |
