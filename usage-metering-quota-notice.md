# Trigger.dev 用量计量与套餐配额告警机制

本文档基于代码分析，深入梳理 trigger.dev 在配额接近上限时的提示机制，重点剖析免费用户侧边栏 FreePlanUsage 组件、SideMenu 渲染条件、75%/95%/100% 颜色阈值表现、各提示组件的职责边界，以及缓存窗口导致的提示与阻断不同步问题。

---

## 一、FreePlanUsage 组件：侧边栏配额进度条

### 1.1 组件代码

代码位置：[FreePlanUsage](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/FreePlanUsage.tsx)

```ts
export function FreePlanUsage({ to, percentage }: { to: string; percentage: number }) {
  const cappedPercentage = Math.min(percentage, 1);
  const widthProgress = useMotionValue(cappedPercentage * 100);
  const color = useTransform(
    widthProgress,
    [0, 74, 75, 95, 100],
    ["#22C55E", "#22C55E", "#F59E0B", "#F43F5E", "#F43F5E"]
  );

  const hasHitLimit = cappedPercentage >= 1;

  return (
    <div className={cn(
      "rounded border border-charcoal-700 bg-charcoal-750 p-2.5",
      hasHitLimit && "border-error/40"
    )}>
      <div className="flex items-center justify-between gap-2">
        <div className="flex items-center gap-1">
          <ArrowUpCircleIcon className="h-5 w-5 text-text-dimmed" />
          <span className="text-2sm text-text-bright">Free Plan</span>
        </div>
        <Link to={to} className="text-2sm text-text-link focus-custom">
          Upgrade
        </Link>
      </div>
      <div className="relative mt-3 h-1 rounded-full bg-background-dimmed">
        <motion.div
          initial={{ width: 0 }}
          animate={{ width: cappedPercentage * 100 + "%" }}
          style={{ backgroundColor: color }}
          transition={{ duration: 1, type: "spring" }}
          className="absolute left-0 top-0 h-full rounded-full"
        />
      </div>
    </div>
  );
}
```

### 1.2 三段颜色阈值

使用 Framer Motion 的 `useTransform` 实现连续颜色插值：

```ts
const color = useTransform(
  widthProgress,         // 输入：0-100 的进度值
  [0, 74, 75, 95, 100], // 输入断点
  ["#22C55E", "#22C55E", "#F59E0B", "#F43F5E", "#F43F5E"]  // 输出颜色
);
```

| 使用率范围        | 进度条颜色              | Tailwind 近似色  | 含义         |
|------------------|------------------------|-----------------|-------------|
| 0% – 74%         | `#22C55E`              | green-500       | 正常（绿色） |
| 75% – 94%        | `#F59E0B`              | amber-500       | 警告（琥珀色）|
| 95% – 100%+      | `#F43F5E`              | rose-500        | 危险（红色） |

颜色变化特点：
- **75% 是从绿到琥珀的转折点**，是所有组件中最早的视觉预警
- **95% 是从琥珀到红的转折点**，比 Limits 页面的 90% 更晚
- 75%-95% 之间颜色平滑过渡（`useTransform` 线性插值）
- 0%-74% 始终绿色，75% 瞬间跳变到琥珀色

### 1.3 超限时的额外视觉效果

```ts
const hasHitLimit = cappedPercentage >= 1;
// 容器边框变为错误色
className={cn(
  "rounded border border-charcoal-700 bg-charcoal-750 p-2.5",
  hasHitLimit && "border-error/40"
)}
```

- **使用率 ≥ 100%**：容器边框变为 `border-error/40`（半透明红色边框）
- 进度条宽度上限 `cappedPercentage = Math.min(percentage, 1)`，始终不会超过 100%

### 1.4 组件 UI 结构

```
┌─────────────────────────────────────┐
│  ↑ Free Plan               Upgrade │  ← 头部：图标+文字+升级链接
│  ═════════════════════════════      │  ← 进度条：颜色随阈值变化
└─────────────────────────────────────┘
         ↑ 边框变红 (percentage ≥ 100%)
```

- "Free Plan" 文字始终显示，不随用量变化
- "Upgrade" 链接始终可见，指向 billing 页面
- 进度条高 1px（`h-1`），非常纤细
- 使用 spring 动画过渡，持续 1 秒

---

## 二、SideMenu 渲染 FreePlanUsage 的条件

### 2.1 渲染位置

代码位置：[SideMenu](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/SideMenu.tsx#L736-L753)

```tsx
<motion.div layout className={cn(
  "flex flex-col gap-1 border-t border-grid-bright p-1",
  isCollapsed && "items-center"
)}>
  <HelpAndAI isCollapsed={isCollapsed} ... />
  {isFreeUser && (
    <CollapsibleHeight isCollapsed={isCollapsed}>
      <FreePlanUsage
        to={v3BillingPath(organization)}
        percentage={currentPlan.v3Usage.usagePercentage}
      />
    </CollapsibleHeight>
  )}
</motion.div>
```

FreePlanUsage 位于 SideMenu 的**最底部区域**（`grid-rows-[2.5rem_1fr_auto]` 中的 `auto` 行），在以下组件下方：
1. IncidentStatusPanel（事故状态面板）
2. V3DeprecationPanel（V3 弃用警告）
3. NotificationPanel（平台通知卡片）
4. HelpAndAI（帮助与 AI 助手）

### 2.2 渲染条件

```ts
const isFreeUser = currentPlan?.v3Subscription?.isPaying === false;
```

| 条件                              | 是否渲染 FreePlanUsage |
|----------------------------------|----------------------|
| 免费用户 (`isPaying === false`)   | ✅ 显示               |
| 付费用户 (`isPaying === true`)    | ❌ 不显示             |
| 无计划信息 (`currentPlan` 为空)    | ❌ 不显示             |
| 自托管环境（`client` 为 undefined）| ❌ 不显示（currentPlan 为空）|

### 2.3 侧边栏折叠时的行为

```tsx
<CollapsibleHeight isCollapsed={isCollapsed}>
  <FreePlanUsage ... />
</CollapsibleHeight>
```

- 侧边栏折叠时，`CollapsibleHeight` 通过 CSS Grid `grid-rows-[0fr]` + `opacity-0` 将 FreePlanUsage 完全隐藏
- **折叠状态下没有替代的 mini 图标**，用户完全看不到配额信息
- 展开时通过 `grid-rows-[1fr]` + `opacity-100` 显示，带 200ms 动画

### 2.4 percentage 数据来源

代码位置：[org route.tsx loader](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug/route.tsx#L115-L120)

```ts
let hasExceededFreeTier = false;
let usagePercentage = 0;
if (plan?.v3Subscription && !plan.v3Subscription.isPaying && plan.v3Subscription.plan && usage) {
  hasExceededFreeTier = usage.cents > plan.v3Subscription.plan.limits.includedUsage;
  usagePercentage = usage.cents / plan.v3Subscription.plan.limits.includedUsage;
}

return typedjson({
  currentPlan: { ...plan, v3Usage: { ...usage, hasExceededFreeTier, usagePercentage } },
});
```

- `usagePercentage = usage.cents / includedUsage`（0 到 1+ 的浮点数）
- 此值通过 `useCurrentPlan()` hook 在整个应用共享
- 数据来自组织级 loader，**仅在页面导航时重新加载**（见下文 shouldRevalidate 分析）

### 2.5 shouldRevalidate：何时刷新 usagePercentage

代码位置：[org route.tsx shouldRevalidate](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug/route.tsx#L33-L59)

```ts
export const shouldRevalidate: ShouldRevalidateFunction = (params) => {
  // 仅在以下条件重新加载：
  // 1. 组织 slug 变化
  // 2. 项目 slug 变化
  // 3. 环境 slug 变化
  // 4. 环境 pause/resume 表单提交
  // 5. URL 路径名变化（不含 search params）
  return params.currentUrl.pathname !== params.nextUrl.pathname;
};
```

**关键发现：`usagePercentage` 不会自动定时刷新。** 它只在用户切换页面时重新加载。这与 Limits 页面的 5 秒自动轮询形成对比。

---

## 三、五大提示组件的职责对比

### 3.1 职责矩阵

| 组件              | 目标用户   | 显示位置          | 触发方式   | 何时可见               | 提醒类型   | 颜色阈值            |
|------------------|-----------|-------------------|-----------|----------------------|-----------|-------------------|
| **FreePlanUsage**| 免费用户   | SideMenu 底部      | 始终可见   | 侧边栏展开时始终显示    | 被动+持续  | 75% 绿→琥珀, 95%→红 |
| **UsageBar**     | 所有用户   | Usage 页面         | 用户导航到  | 用户主动查看 Usage 页   | 被动+按需  | 仅绿色深浅变化       |
| **Limits 页面**  | 所有用户   | Limits 页面表格    | 用户导航到  | 用户主动查看 Limits 页  | 被动+按需  | 90% 橙, 100% 红     |
| **UpgradePrompt**| 免费用户   | 全局 NavBar 下方    | 超限时     | 超 100% 后所有页面     | 被动+强制  | 红色横幅            |
| **Billing Alerts**| 付费用户  | 邮件               | 阈值触发   | 邮件客户端             | 主动推送   | 75%/90%/100%/...   |

### 3.2 各组件的具体职责

#### FreePlanUsage：持续可见的"仪表盘"

- **唯一始终可见的配额进度指示器**（只要侧边栏展开）
- 告诉用户：你用的是 Free Plan，用量到多少了
- 提供 Upgrade 快捷链接
- 不提供具体数字，只有进度条颜色
- **不提供 75%/95% 的文字提示**，仅通过颜色暗示

#### UsageBar：金额维度的详细可视化

- 展示美元金额（Used / Included usage / Billing limit）
- 按月选择器查看历史
- 配合按任务分组的费用明细表和日趋势图
- 超限时颜色从 `bg-green-600` 变为 `bg-green-700`（几乎看不出）
- **职责：让用户理解钱花在哪里**

#### Limits 页面：结构化限额清单

- 按行展示每个 Quota 的 Limit / Current / Source / Upgrade
- Rate Limit 实时 Token 余量（5 秒刷新）
- Quota 达 90% 时 Current 数字变橙，达 100% 变红
- **职责：让用户了解所有限制的精确数值**

#### UpgradePrompt：超限后的强制提醒

- 仅当 `hasExceededFreeTier === true` 时出现
- 替换 NavBar 下的 EnvironmentBanner
- 红色背景 + 错误图标 + 明确阻断文案
- **职责：告知用户已被阻断，引导升级**

#### Billing Alerts：脱离 Dashboard 的主动推送

- 唯一不依赖用户主动查看的提醒
- 通过邮件通知，阈值可配置
- 100% 阈值不可取消
- 含尖峰告警（10x/20x/50x/100x）
- **职责：在用户不看 Dashboard 时也能知道接近上限**

### 3.3 关键差异：FreePlanUsage vs UpgradePrompt

| 维度         | FreePlanUsage                         | UpgradePrompt                         |
|-------------|---------------------------------------|---------------------------------------|
| 出现时机     | 始终可见（只要免费+侧边栏展开）         | 仅超 100% 后                           |
| 显示位置     | SideMenu 底部                          | NavBar 下方（所有页面）                 |
| 视觉强度     | 小卡片+纤细进度条                       | 全宽红色横幅+错误图标                   |
| 文案内容     | "Free Plan" + "Upgrade" 链接           | 明确的阻断说明+下月重置日期             |
| 目标用户     | 免费用户                               | 免费用户                               |
| 折叠时       | 完全隐藏                               | 不受侧边栏折叠影响                     |
| 百分比显示   | 有颜色进度条（75%/95% 阈值）            | 无进度条                              |

**FreePlanUsage 是预警，UpgradePrompt 是告知阻断后果。** 两者之间存在"提示→阻断"的空档期：FreePlanUsage 在 75% 时变琥珀色但无文字，用户可能不理解含义；等到 UpgradePrompt 出现时已经超限被阻断了。

---

## 四、缓存窗口导致的提示与阻断不同步问题

### 4.1 两条独立的数据路径

```
路径 A: Dashboard 提示（FreePlanUsage / UpgradePrompt / UsageBar / Limits）
┌───────────────────┐     ┌───────────────────┐     ┌───────────────────┐
│ org route.tsx     │     │ platformCache     │     │ Platform Billing  │
│ loader            │────→│ .usage            │────→│ API               │
│                   │     │ fresh: 5min       │     │ client.usage()    │
│ usagePercentage   │     │ stale: 10min      │     │                   │
│ hasExceededFreeTier│    │                   │     │                   │
└───────────────────┘     └───────────────────┘     └───────────────────┘

路径 B: API 阻断（Entitlement 检查）
┌───────────────────┐     ┌───────────────────┐     ┌───────────────────┐
│ triggerTask       │     │ platformCache     │     │ Platform Billing  │
│ V1.server.ts      │────→│ .entitlement      │────→│ API               │
│                   │     │ fresh: 1min       │     │ client.getEntit.. │
│ getEntitlement()  │     │ stale: 2min       │     │                   │
│ → OutOfEntitle..  │     │                   │     │                   │
└───────────────────┘     └───────────────────┘     └───────────────────┘
```

### 4.2 缓存 TTL 对比

| 缓存项       | Fresh TTL | Stale TTL | 用途                          | 消费者              |
|-------------|-----------|-----------|------------------------------|--------------------|
| `usage`     | 5 min     | 10 min    | 用量数据（cents）              | org loader → FreePlanUsage / UpgradePrompt / UsageBar |
| `limits`    | 5 min     | 10 min    | 套餐限额（includedUsage 等）    | getCurrentPlan → org loader |
| `entitlement`| 1 min    | 2 min     | 访问权限（hasAccess）           | triggerTask / batchTrigger |

### 4.3 不同步场景分析

#### 场景 1：Dashboard 显示正常但 API 已阻断

```
时间线：
T+0    用户用量到达 includedUsage 的 100%
T+0    Platform Billing API 更新 hasAccess = false
T+0~1  entitlement 缓存仍在 fresh 期（1 min），API 查询返回旧值 hasAccess = true
T+1~2  entitlement 缓存进入 stale 期，后台重新验证，新值 hasAccess = false 入缓存
T+2    后续 API 调用读取到 hasAccess = false，开始阻断

同时：
T+0~5  usage 缓存仍在 fresh 期（5 min），FreePlanUsage 显示的 usagePercentage < 1
T+5~10 usage 缓存进入 stale 期，后台重新验证
T+10   FreePlanUsage 才可能显示更新后的百分比

结果：API 在 T+2 开始阻断，但 Dashboard 到 T+10 才反映真实用量
      用户在 T+2~T+10 之间看到 Dashboard 正常但 API 报错 OutOfEntitlementError
```

**最大延迟：usage 缓存最多比 entitlement 缓存晚 8 分钟反映用量变化。**

#### 场景 2：Dashboard 显示超限但 API 仍放行

```
时间线：
T+0    用量刚过 includedUsage
T+0    Platform Billing API 的 entitlement 判定尚未触发（可能有内部延迟）
T+0    usage 缓存重新验证后返回最新 cents > includedUsage
T+0    FreePlanUsage 变红、UpgradePrompt 出现
T+0~1  entitlement 缓存仍为 hasAccess = true（Billing API 内部尚未判定）
T+1~2  entitlement 缓存重新验证，hasAccess = false 入缓存

结果：Dashboard 显示超限，但 API 在 T+0~T+1 仍允许触发
      这是"假阳性"——用户看到警告但实际仍能触发任务
```

#### 场景 3：org loader 的 shouldRevalidate 进一步延迟

即使 usage 缓存已更新，org loader 的 `shouldRevalidate` 逻辑决定了何时重新执行 loader：

```ts
return params.currentUrl.pathname !== params.nextUrl.pathname;
```

- 仅在 URL 路径变化时重新加载
- **搜索参数变化不触发**（如 search params）
- **同页面内操作不触发**（如触发任务、查看日志）

这意味着：
- 用户如果停留在同一页面（如 Tasks 列表），`usagePercentage` 可能**长时间不更新**
- 直到用户导航到其他页面，org loader 才重新执行，读取（可能已过期的）usage 缓存
- **实际延迟 = org loader 未重新加载时间 + usage 缓存 TTL**

### 4.4 不同步的影响总结

| 不同步类型           | 最大延迟     | 影响                                       |
|---------------------|-------------|-------------------------------------------|
| Dashboard 滞后于阻断 | ~8 min      | 用户看到正常 Dashboard 但 API 报错，困惑    |
| 阻断滞后于 Dashboard | ~1 min      | 用户看到超限警告但 API 仍放行，产生误判      |
| org loader 不刷新    | 不确定（用户停留时间） | FreePlanUsage 长期不更新，颜色不变化     |
| 侧边栏折叠           | 用户操作决定  | FreePlanUsage 完全不可见                    |

### 4.5 Fail-Open 加剧不同步

代码位置：[getEntitlement](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L565-L598)

```ts
if (result.err || result.val === undefined) {
  return { hasAccess: true as const }; // Fail-Open
}
```

- 当 Billing API 不可用时，entitlement 默认放行
- 但 usage 缓存可能仍返回旧值，显示超限
- 用户看到超限提示但任务仍能正常触发——**这可能让用户忽视超限提示的可信度**

---

## 五、NotificationPanel：产品内通知系统

### 5.1 通知机制

代码位置：[NotificationPanel](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/NotificationPanel.tsx) / [platformNotifications.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platformNotifications.server.ts)

SideMenu 中有一个 `NotificationPanel`，位于 FreePlanUsage 上方，展示来自 `PlatformNotification` 数据库表的通知卡片。

### 5.2 通知查询

```ts
const POLL_INTERVAL_MS = 60000; // 1 分钟轮询
```

- 每 **60 秒**轮询一次 `/resources/platform-notifications`
- `shouldRevalidate: () => false`（不随页面导航重新验证）
- 查询活跃通知：`surface: "WEBAPP"` + `archivedAt: null` + 时间范围内

### 5.3 通知类型

```ts
type: z.enum(["card", "info", "warn", "error", "success", "changelog"])
```

通知支持多种类型，包括 `warn` 和 `error`。scope 可以是：
- `GLOBAL`：所有用户
- `ORGANIZATION`：特定组织
- `PROJECT`：特定项目
- `USER`：特定用户

### 5.4 是否有配额相关通知？

**当前代码中不存在自动创建配额通知的逻辑。** `PlatformNotification` 的创建和管理通过 admin 端点手动操作（`createPlatformNotification`），没有代码在用量达到阈值时自动插入通知记录。

这意味着：
- **没有代码自动在 NotificationPanel 中显示"接近上限"或"超限"通知**
- 配额通知只能通过管理员手动创建 PlatformNotification 来实现
- Billing Alerts 的邮件通知由 Platform Billing API 在后台发送，不走 NotificationPanel

### 5.5 NotificationPanel 与 FreePlanUsage 的位置关系

```
SideMenu 底部区域（从上到下）：
┌─────────────────────────────┐
│ IncidentStatusPanel          │  ← 事故状态
├─────────────────────────────┤
│ V3DeprecationPanel           │  ← V3 弃用警告
├─────────────────────────────┤
│ NotificationPanel            │  ← 平台通知（手动创建）
├─────────────────────────────┤
│ HelpAndAI                    │  ← 帮助 + AI
├─────────────────────────────┤
│ FreePlanUsage (仅免费用户)    │  ← 配额进度条
└─────────────────────────────┘
```

如果有管理员创建了配额相关的 PlatformNotification（scope=ORGANIZATION），它会出现在 NotificationPanel 中，位于 FreePlanUsage 上方。

---

## 六、完整的"提示→阻断"流程与时间窗口

### 6.1 免费用户体验时间线

```
用量 0%──75%──95%──100%────────────────────────→

       │     │     │      │
       ▼     ▼     ▼      ▼
    FreePlan  FreePlan  FreePlan    UpgradePrompt
    绿色      琥珀色    红色        红色横幅出现
    (侧边栏)  (侧边栏)  (侧边栏)   (全局 NavBar)
                       +红色边框
                       
                       │
                       ▼
                    API 阻断
                    OutOfEntitlementError
                    (仅非 DEV 环境)

       ◄── 5-10 min 缓存延迟 ──►
       Dashboard 可能还没变红时 API 已阻断
```

### 6.2 付费用户体验时间线

```
用量 0%──75%──90%──100%──200%──→

       │           │     │      │
       ▼           ▼     ▼      ▼
    Billing     Limits  Usage  Billing
    Alert 75%   90%橙   Bar    Alert 100%
    (邮件)      (需主动查看)    (邮件)
                            (邮件)

    注意：付费用户没有 FreePlanUsage 和 UpgradePrompt
    付费用户没有侧边栏进度条
    付费用户没有全局超限横幅
```

### 6.3 关键缺口

| 用户类型 | 75% 时看到什么            | 95% 时看到什么           | 100% 时看到什么          |
|---------|--------------------------|------------------------|------------------------|
| 免费用户 | FreePlanUsage 琥珀色进度条 | FreePlanUsage 红色进度条 | UpgradePrompt 红色横幅 + API 阻断 |
| 付费用户 | 仅邮件（如配置了 Billing Alert）| 仅邮件 + Limits 页面数字变色 | 仅邮件 + API 可能阻断（取决于计划）|

**免费用户在 75%-95% 之间只有侧边栏进度条变色，没有文字提示解释变色含义。**

---

## 七、关键代码索引

| 功能                        | 文件                                                                                                              |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------|
| FreePlanUsage 组件          | [FreePlanUsage.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/FreePlanUsage.tsx) |
| FreePlanUsage 颜色阈值      | [useTransform L10-L14](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/FreePlanUsage.tsx#L10-L14) |
| FreePlanUsage 超限边框      | [hasHitLimit L16-L23](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/FreePlanUsage.tsx#L16-L23) |
| SideMenu 渲染条件           | [SideMenu.tsx#L745-L752](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/SideMenu.tsx#L745-L752) |
| isFreeUser 判定             | [SideMenu.tsx#L180](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/SideMenu.tsx#L180) |
| CollapsibleHeight 折叠隐藏  | [SideMenu.tsx#L1172-L1192](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/SideMenu.tsx#L1172-L1192) |
| usagePercentage 计算        | [org route.tsx#L115-L120](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug/route.tsx#L115-L120) |
| shouldRevalidate 逻辑       | [org route.tsx#L33-L59](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug/route.tsx#L33-L59) |
| getCachedUsage 缓存         | [platform.v3.server.ts#L497-L512](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L497-L512) |
| 缓存 TTL 配置               | [platform.v3.server.ts#L91-L107](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L91-L107) |
| getEntitlement + Fail-Open  | [platform.v3.server.ts#L565-L598](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L565-L598) |
| NotificationPanel           | [NotificationPanel.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/NotificationPanel.tsx) |
| 通知轮询 (60s)              | [platform-notifications.tsx#L40-L61](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/resources.platform-notifications.tsx#L40-L61) |
| 通知查询逻辑                | [platformNotifications.server.ts#L139-L205](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platformNotifications.server.ts#L139-L205) |
| UpgradePrompt 横幅          | [UpgradePrompt.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/UpgradePrompt.tsx) |
| NavBar 中 UpgradePrompt 位置| [PageHeader.tsx#L17-L28](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/primitives/PageHeader.tsx#L17-L28) |
| UsageBar 组件               | [UsageBar.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/UsageBar.tsx) |
| Limits 页面颜色函数         | [limits/route.tsx#L809-L826](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.limits/route.tsx#L809-L826) |
| Billing Alerts 页面         | [billing-alerts/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx) |
| Storybook 测试页            | [storybook.free-plan-usage/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/storybook.free-plan-usage/route.tsx) |
| Entitlement 阻断            | [triggerTaskV1.server.ts#L115-L120](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/services/triggerTaskV1.server.ts#L115-L120) |
| Batch Entitlement 阻断      | [batchTriggerV3.server.ts#L199-L203](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/services/batchTriggerV3.server.ts#L199-L203) |
