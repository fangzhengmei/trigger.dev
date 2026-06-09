# Trigger.dev 用量计量与套餐配额告警机制

本文档基于代码分析，深入梳理 trigger.dev 在配额接近上限时的提示机制，重点校准 Billing Alerts 对免费用户和付费用户的邮件告警差异，重新划定 FreePlanUsage、UpgradePrompt、NotificationPanel、Billing Alerts 四大提示通道在产品内提示和邮件提醒功能上的职责边界。

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
  // ...
}
```

### 1.2 三段颜色阈值

| 使用率范围        | 进度条颜色  | Tailwind 近似色  | 含义         |
|------------------|------------|-----------------|-------------|
| 0% – 74%         | `#22C55E`  | green-500       | 正常（绿色） |
| 75% – 94%        | `#F59E0B`  | amber-500       | 警告（琥珀色）|
| 95% – 100%+      | `#F43F5E`  | rose-500        | 危险（红色） |

- 75% 是所有组件中**最早的视觉预警点**
- 100% 时容器边框额外变为 `border-error/40`（半透明红色边框）
- 进度条宽度上限 `Math.min(percentage, 1)`，始终不超过 100%

### 1.3 SideMenu 渲染条件

代码位置：[SideMenu.tsx#L745-L752](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/SideMenu.tsx#L745-L752)

```tsx
{isFreeUser && (
  <CollapsibleHeight isCollapsed={isCollapsed}>
    <FreePlanUsage
      to={v3BillingPath(organization)}
      percentage={currentPlan.v3Usage.usagePercentage}
    />
  </CollapsibleHeight>
)}
```

- **仅免费用户**（`isPaying === false`）显示
- 侧边栏折叠时完全隐藏，无 mini 图标替代
- `percentage` 来自 org route loader，仅在页面导航时刷新（不自动轮询）

---

## 二、Billing Alerts：邮件告警的完整校准

### 2.1 页面入口与前置条件

代码位置：[billing-alerts/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx) / [OrganizationSettingsSideMenu](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/OrganizationSettingsSideMenu.tsx#L107-L114)

Billing Alerts 入口位于 **Organization Settings** 侧边菜单（非项目级 SideMenu），所有 Cloud 环境的用户都能看到该入口：

```tsx
// OrganizationSettingsSideMenu 中
{isManagedCloud && (
  <SideMenuItem
    name="Billing alerts"
    icon={BellAlertIcon}
    to={v3BillingAlertsPath(organization)}
  />
)}
```

前置条件：
- `isManagedCloud === true`（仅 Trigger.dev Cloud 环境）
- 自托管环境自动重定向到组织首页，无法访问

### 2.2 isFree 分支：amount 字段的差异化渲染

代码位置：[billing-alerts/route.tsx#L194-L251](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L194-L251)

```ts
const isFree = !plan?.v3Subscription?.isPaying;
```

**免费用户的 Amount 渲染**：

```tsx
{isFree ? (
  <>
    <Paragraph variant="small" className="text-text-dimmed">
      ${dollarAmount}
    </Paragraph>
    <input type="hidden" name={amount.name} value={dollarAmount} />
  </>
) : (
  <Input
    {...conform.input(amount, { type: "number" })}
    value={dollarAmount}
    onChange={(e) => { /* 可编辑 */ }}
    readOnly={isFree}
  />
)}
```

| 维度           | 免费用户                                     | 付费用户                         |
|---------------|---------------------------------------------|---------------------------------|
| 渲染方式       | `<Paragraph>` 只读文本 + hidden input         | `<Input>` 可编辑数字输入框        |
| Amount 来源   | Platform Billing API 返回值（美分→美元 `/100`）| 同左，但用户可修改                 |
| Amount 含义   | 固定为免费套餐的 `includedUsage` 金额           | 用户自定义的预算金额               |
| 能否编辑       | ❌ 不可编辑（显示为灰色文字）                   | ✅ 可编辑（`step={0.01}`）       |
| hidden input   | 有（确保表单提交时携带值）                      | 无（使用正常 input）              |

**关键理解**：免费用户的 Amount 是 Platform Billing API 根据免费套餐的 `includedUsage` 自动设置的，用户无法修改。这意味着免费用户的告警基准金额就是套餐包含额度本身，告警阈值都是相对于这个固定值计算的。

### 2.3 标准告警阈值（checkboxLevels）

代码位置：[billing-alerts/route.tsx#L185](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L185)

```ts
const checkboxLevels = [0.75, 0.9, 1.0, 2.0, 5.0];
```

| 阈值   | 金额（免费用户 $5 included 为例） | 含义                   | 默认行为                        |
|--------|----------------------------------|------------------------|--------------------------------|
| 75%    | $3.75                            | 接近预算                | `defaultChecked={alerts.alertLevels.includes(level)}` |
| 90%    | $4.50                            | 即将超预算              | 同上                           |
| 100%   | $5.00                            | 预算耗尽                | `readOnly={level === 1.0}` — **不可取消** |
| 200%   | $10.00                           | 超出预算 2 倍           | 同上                           |
| 500%   | $25.00                           | 超出预算 5 倍           | 同上                           |

**100% 阈值不可取消的逻辑**：

```tsx
<CheckboxWithLabel
  name={alertLevels.name}
  value={level.toString()}
  defaultChecked={alerts.alertLevels.includes(level)}
  readOnly={level === 1.0}
/>
```

- `readOnly={level === 1.0}` 使 100% 复选框变灰，用户无法取消勾选
- 这确保**所有用户（包括免费和付费）至少在用量达到预算 100% 时都会收到邮件**
- 如果这是首次访问（`alerts.alertLevels` 为空），100% 复选框默认不勾选，但 `readOnly` 使其无法取消

### 2.4 尖峰告警阈值（spikeAlertLevels）

代码位置：[billing-alerts/route.tsx#L187](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L187)

```ts
const spikeAlertLevels = [10.0, 20.0, 50.0, 100.0];
```

| 阈值     | 金额（免费用户 $5 included 为例） | 含义           |
|----------|----------------------------------|---------------|
| 10x      | $50.00                           | 异常飙升        |
| 20x      | $100.00                          | 严重异常        |
| 50x      | $250.00                          | 极端异常        |
| 100x     | $500.00                          | 灾难性异常      |

**尖峰告警的默认选中逻辑**：

```tsx
defaultChecked={
  alerts.alertLevels.includes(level) ||
  !spikeAlertLevels.some((l) => alerts.alertLevels.includes(l))
}
```

解读：
- 如果当前 `alertLevels` 已包含该阈值 → 保持勾选
- **如果 `alertLevels` 中不包含任何尖峰阈值 → 默认全部勾选**

这意味着：
- 首次访问的用户（没有配置过告警）：**所有尖峰告警默认勾选**
- 已手动取消某些尖峰告警的用户：仅保留之前勾选的
- 尖峰告警没有 `readOnly`，用户可以自由取消任何阈值

### 2.5 邮箱输入的表单行为

代码位置：[billing-alerts/route.tsx#L313-L332](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L313-L332)

```ts
const schema = z.object({
  emails: z.preprocess((i) => {
    if (typeof i === "string") return [i];
    if (Array.isArray(i)) {
      const emails = i.filter((v) => typeof v === "string" && v !== "");
      if (emails.length === 0) return [""];
      return emails;
    }
    return [""];
  }, z.string().email().array().nonempty("At least one email is required")),
  // ...
});
```

表单行为：
- 至少需要一个有效邮箱
- 当所有已有邮箱输入框都填满时，自动追加新的空输入框
- `defaultValue: { emails: [""] }` 初始只有一个空输入框
- 提交时 `amount` 乘以 100 转为美分（`submission.value.amount * 100`）

### 2.6 免费用户 vs 付费用户的邮件告警完整对比

| 维度                 | 免费用户                                         | 付费用户                              |
|---------------------|-------------------------------------------------|--------------------------------------|
| Amount              | 只读（`<Paragraph>`），值由 Billing API 设置        | 可编辑（`<Input>`），用户自定义预算     |
| Amount 含义         | 套餐的 includedUsage 金额                          | 用户设定的月度预算                     |
| 标准阈值            | 75%/90%/100%/200%/500%                            | 同左                                  |
| 100% 阈值           | `readOnly` 不可取消                                | `readOnly` 不可取消                    |
| 尖峰阈值            | 10x/20x/50x/100x                                  | 同左                                  |
| 尖峰默认勾选        | 无已有配置时全部勾选                                | 同左                                  |
| 邮箱配置            | 可编辑                                            | 可编辑                                |
| 是否能收到邮件       | ✅ 是（配置邮箱后）                                  | ✅ 是                                  |
| 邮件中的金额        | 基于 includedUsage 计算                             | 基于用户设置的 amount 计算              |

**关键校准**：**免费用户和付费用户都可以接收 Billing Alerts 邮件。** 之前文档中将 Billing Alerts 归类为"仅付费用户"是不准确的。两者的差异仅在 Amount 是否可编辑，告警机制本身对两种用户都可用。

### 2.7 免费用户的 Amount 值从何而来

代码位置：[billing-alerts/route.tsx#L76-L81](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L76-L81)

```ts
return typedjson({
  alerts: {
    ...alerts,
    amount: alerts.amount / 100,  // 美分 → 美元
  },
});
```

- `alerts` 来自 `getBillingAlerts(organization.id)` → Platform Billing API
- Platform Billing API 为免费用户自动设置 `amount = includedUsage`（以美分为单位）
- 页面将美分转为美元显示
- 免费用户看到的就是套餐包含额度的等值金额（如 Free 套餐 $5 → 显示 $5.00）

---

## 三、UpgradePrompt：超限后的强制提醒

代码位置：[UpgradePrompt](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/UpgradePrompt.tsx)

### 3.1 触发条件

代码位置：[org route.tsx#L115-L120](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug/route.tsx#L115-L120)

```ts
hasExceededFreeTier = usage.cents > plan.v3Subscription.plan.limits.includedUsage;
```

- **仅免费用户**，且 `usage.cents > includedUsage` 时触发
- 付费用户永远不会看到此横幅
- 替换 NavBar 下方的 EnvironmentBanner

### 3.2 与 Billing Alerts 100% 阈值的关系

Billing Alerts 的 100% 阈值判定条件是 `usage ≥ amount`。对于免费用户，`amount = includedUsage`，因此：
- Billing Alerts 100% 邮件 ≈ `usage.cents ≥ includedUsage`
- UpgradePrompt 出现 ≈ `usage.cents > includedUsage`

两者几乎同时触发，但：
- Billing Alerts 100% 是邮件通知（异步，用户可能在邮箱中看到）
- UpgradePrompt 是 Dashboard 横幅（同步，用户在产品内看到）
- 由于缓存差异，实际触发时间可能有数分钟偏差

---

## 四、NotificationPanel：产品内通知卡片

代码位置：[NotificationPanel](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/NotificationPanel.tsx) / [platformNotifications.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platformNotifications.server.ts)

### 4.1 定位与能力

- 位于 SideMenu 底部区域（FreePlanUsage 上方）
- 60 秒轮询，展示来自 `PlatformNotification` 数据库表的卡片
- 支持类型：`card`、`info`、`warn`、`error`、`success`、`changelog`
- 支持 scope：`GLOBAL`、`ORGANIZATION`、`PROJECT`、`USER`

### 4.2 配额通知现状

**当前代码中不存在自动创建配额相关 PlatformNotification 的逻辑。** 所有通知必须通过 admin 端点手动创建。

这意味着 NotificationPanel 的 `warn`/`error` 类型虽然**技术上可以展示配额告警**，但目前没有被用于此目的。它是一个**可扩展但未启用**的通知通道。

---

## 五、四大提示通道的职责边界

### 5.1 职责矩阵（校准版）

| 通道                | 通道类型   | 目标用户    | 展示位置            | 触发时机         | 提醒方式          | 阈值体系               | 可配置性        |
|--------------------|-----------|------------|--------------------|-----------------|------------------|-----------------------|----------------|
| **FreePlanUsage**  | 产品内提示 | 仅免费用户  | 项目 SideMenu 底部  | 始终可见         | 进度条颜色变化     | 75% 绿→琥珀, 95%→红   | 不可配置        |
| **UpgradePrompt**  | 产品内提示 | 仅免费用户  | 全局 NavBar 下方    | usage > included | 红色横幅+阻断文案  | 100%（无中间阈值）     | 不可配置        |
| **Billing Alerts** | 邮件提醒   | **所有用户**| 组织设置页配置      | 阈值触发          | 邮件通知          | 75/90/100/200/500% + 尖峰 | 邮箱+阈值可配置 |
| **NotificationPanel**| 产品内提示| 所有用户   | 项目 SideMenu 底部  | 管理员手动创建    | 通知卡片          | 无内置配额逻辑         | 仅管理员可配置  |

### 5.2 各通道的提示内容对比

| 通道                | 75% 时提示                   | 100% 时提示                          | 超限后提示                |
|--------------------|-----------------------------|--------------------------------------|-------------------------|
| **FreePlanUsage**  | 进度条变琥珀色（无文字）       | 进度条变红+边框变红（无文字）           | 同 100%（进度条不超 100%）|
| **UpgradePrompt**  | 不显示                       | 不显示                                | 红色横幅+明确阻断文案      |
| **Billing Alerts** | 邮件："compute spend crossed 75%" | 邮件："compute spend crossed 100%"  | 邮件（200%/500%/尖峰）    |
| **NotificationPanel**| 无                          | 无                                   | 无（除非管理员手动创建）   |

### 5.3 免费用户完整体验时间线（校准版）

```
用量 0%─────75%─────95%─────100%──────────→

       │       │       │        │
       ▼       ▼       ▼        ▼
    FreePlan  FreePlan  FreePlan   UpgradePrompt
    绿色      琥珀色    红色       红色横幅
    (侧边栏)  (侧边栏)  (侧边栏)  (全局 NavBar)
                       +红色边框
                       
    ◄── 邮件 (如配置了 Billing Alerts) ──►
    Billing   Billing   Billing    Billing
    Alert 75% Alert 90% Alert 100% Alert 200%...
    (邮箱)    (邮箱)    (邮箱)     (邮箱)
    
                                    │
                                    ▼
                                 API 阻断
                                 OutOfEntitlementError
                                 (仅非 DEV 环境)
```

### 5.4 付费用户完整体验时间线（校准版）

```
用量 0%─────75%─────90%─────100%──────────→

       │               │       │        │
       ▼               ▼       ▼        ▼
    Billing          Limits   Billing  Billing
    Alert 75%        90%橙    Alert    Alert 200%
    (邮箱)          (需主动    100%     (邮箱)
                     查看)    (邮箱)

    注意：付费用户
    ✗ 没有 FreePlanUsage
    ✗ 没有 UpgradePrompt
    ✓ 有 Billing Alerts（与免费用户相同的配置界面）
    ✓ 有 Usage 页面（需主动查看）
    ✓ 有 Limits 页面（需主动查看）
```

### 5.5 职责边界总结

**FreePlanUsage**：产品内·被动·持续的用量感知
- 职责：让免费用户在正常使用过程中**随时感知**用量进度
- 局限：仅颜色变化无文字，折叠时不可见，不自动刷新

**UpgradePrompt**：产品内·被动·强制的阻断告知
- 职责：在免费用户**已被阻断后**告知原因和解决方案
- 局限：仅超 100% 后出现，没有预警功能

**Billing Alerts**：邮件·主动·可配置的阈值推送
- 职责：在用户**不看 Dashboard** 时也能通过邮件获知用量状态
- 覆盖：**免费用户和付费用户均可使用**
- 局限：需要用户主动配置邮箱，邮件可能被忽略

**NotificationPanel**：产品内·被动·可扩展的通知卡片
- 职责：展示管理员创建的**产品内通知**（含配额告警的扩展点）
- 现状：**未用于配额告警**，仅用于产品公告等

---

## 六、Billing Alerts 的数据流与缓存

### 6.1 完整数据流

```
┌──────────────────────────┐
│ Billing Alerts 配置页面    │
│ (setBillingAlert)         │
│   amount × 100 → 美分     │
│   emails[]               │
│   alertLevels[]          │
└────────────┬─────────────┘
             │ POST
             ▼
┌──────────────────────────┐
│ Platform Billing API     │
│ .updateBillingAlerts()   │
│   存储: amount, emails,  │
│         alertLevels      │
│   后台监控: usage 对比    │
│   各 alertLevel × amount │
│   达到阈值 → 发送邮件     │
└──────────────────────────┘

读取:
┌──────────────────────────┐
│ loader → getBillingAlerts│
│   → client.getBillingAlerts()
│   → 返回 { amount(美分), emails[], alertLevels[] }
│   → 页面 amount / 100 → 美元显示
└──────────────────────────┘
```

### 6.2 Billing Alerts 不走 platformCache

与 `usage`、`limits`、`entitlement` 不同，`getBillingAlerts` 和 `setBillingAlert` **直接调用 BillingClient**，不经过 `platformCache` 的 SWR 缓存：

```ts
export async function getBillingAlerts(organizationId: string) {
  if (!client) return undefined;
  const result = await client.getBillingAlerts(organizationId);
  // 直接返回，无缓存
}
```

这意味着：
- 配置页面每次加载都直接查询 Billing API
- 修改后立即生效（无缓存延迟）
- 但邮件告警的触发判定在 Billing API 后台进行，与 Webapp 无关

### 6.3 邮件告警与 Dashboard 提示的独立运行

```
邮件告警路径（Billing API 后台）:
  Usage 上报 → Billing API 聚合 → 对比 alertLevel × amount → 发邮件
  (无 Webapp 缓存延迟，Billing API 内部实时判定)

Dashboard 提示路径（Webapp）:
  org loader → getCachedUsage (5min/10min) → FreePlanUsage / UpgradePrompt
  (有缓存延迟，见下文)
```

**邮件告警可能比 Dashboard 提示更早到达用户。** 这是一个合理的设计：邮件通道不受 Webapp 缓存 TTL 限制，Billing API 可以在自己的后台逻辑中实时判定阈值。

---

## 七、缓存窗口导致的提示与阻断不同步问题

### 7.1 缓存 TTL 对比

| 缓存项       | Fresh TTL | Stale TTL | 消费者                                  |
|-------------|-----------|-----------|----------------------------------------|
| `usage`     | 5 min     | 10 min    | org loader → FreePlanUsage / UpgradePrompt / UsageBar |
| `limits`    | 5 min     | 10 min    | getCurrentPlan → org loader              |
| `entitlement`| 1 min    | 2 min     | triggerTask / batchTrigger               |
| Billing Alerts| 无缓存   | —         | 直接查询 Billing API                     |

### 7.2 两条独立路径导致的不同步

```
路径 A: Dashboard 提示
  getCachedUsage → platformCache.usage (5min fresh / 10min stale)
  → usagePercentage → FreePlanUsage 颜色
  → hasExceededFreeTier → UpgradePrompt 显示

路径 B: API 阻断
  getEntitlement → platformCache.entitlement (1min fresh / 2min stale)
  → hasAccess === false → OutOfEntitlementError
```

**典型不同步场景**：entitlement 缓存（1-2 min）比 usage 缓存（5-10 min）更快反映超限状态。API 可能在 T+2 min 开始阻断，但 Dashboard 到 T+10 min 才显示超限。用户在 T+2~T+10 之间看到 Dashboard 正常但 API 报错。

加上 org loader 的 `shouldRevalidate` 仅在路径变化时触发，实际延迟可能更长。

### 7.3 邮件告警不受缓存影响

Billing Alerts 的阈值判定在 Platform Billing API 后台进行，不经过 Webapp 的 `platformCache`。这意味着：
- **邮件告警的到达时间与 Dashboard 提示和 API 阻断均独立**
- 邮件可能在 Dashboard 还没变红时就到达（Billing API 实时判定 vs Webapp 缓存延迟）
- 邮件也可能在 API 已阻断后才到达（邮件发送队列延迟）

---

## 八、关键代码索引

| 功能                        | 文件                                                                                                              |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------|
| FreePlanUsage 组件          | [FreePlanUsage.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/FreePlanUsage.tsx) |
| FreePlanUsage 颜色阈值      | [useTransform L10-L14](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/FreePlanUsage.tsx#L10-L14) |
| SideMenu 渲染 FreePlanUsage | [SideMenu.tsx#L745-L752](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/SideMenu.tsx#L745-L752) |
| isFreeUser 判定             | [SideMenu.tsx#L180](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/SideMenu.tsx#L180) |
| Billing Alerts 页面         | [billing-alerts/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx) |
| isFree 判定（Billing Alerts）| [billing-alerts/route.tsx#L194](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L194) |
| Amount 只读渲染             | [billing-alerts/route.tsx#L222-L251](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L222-L251) |
| 标准阈值定义                | [billing-alerts/route.tsx#L185](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L185) |
| 尖峰阈值定义                | [billing-alerts/route.tsx#L187](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L187) |
| 100% readOnly               | [billing-alerts/route.tsx#L273](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L273) |
| 尖峰默认勾选逻辑            | [billing-alerts/route.tsx#L303-L306](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L303-L306) |
| 邮箱验证 schema            | [billing-alerts/route.tsx#L88-L100](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L88-L100) |
| Amount 美分→美元            | [billing-alerts/route.tsx#L79](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L79) |
| Amount 美元→美分（提交）    | [billing-alerts/route.tsx#L134](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L134) |
| getBillingAlerts            | [platform.v3.server.ts#L600-L610](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L600-L610) |
| setBillingAlert             | [platform.v3.server.ts#L612-L623](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L612-L623) |
| OrganizationSettings 侧边栏 | [OrganizationSettingsSideMenu.tsx#L107-L114](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/OrganizationSettingsSideMenu.tsx#L107-L114) |
| UpgradePrompt 横幅          | [UpgradePrompt.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/UpgradePrompt.tsx) |
| hasExceededFreeTier 计算    | [org route.tsx#L115-L120](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug/route.tsx#L115-L120) |
| NotificationPanel           | [NotificationPanel.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/NotificationPanel.tsx) |
| 通知查询逻辑                | [platformNotifications.server.ts#L139-L205](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platformNotifications.server.ts#L139-L205) |
| 缓存 TTL 配置               | [platform.v3.server.ts#L91-L107](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L91-L107) |
| Entitlement 阻断            | [triggerTaskV1.server.ts#L115-L120](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/v3/services/triggerTaskV1.server.ts#L115-L120) |
| Billing 页面（套餐展示）     | [billing/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing/route.tsx) |
