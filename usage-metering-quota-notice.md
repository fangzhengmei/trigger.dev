# Trigger.dev 用量计量与套餐配额告警机制

本文档基于代码分析，深入梳理 Billing Alerts 100% 阈值的精确判断逻辑，包括 CheckboxWithLabel 的 checked 状态管理、readOnly 阻止切换但不强制选中的实现细节、alerts.alertLevels 默认值来源，以及"100% 不可取消"与"是否一定收到 100% 邮件"之间的关联；同时重新整理免费用户与付费用户的邮件提醒时间线。

---

## 一、CheckboxWithLabel 的 checked 状态管理

### 1.1 组件核心逻辑

代码位置：[CheckboxWithLabel](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/primitives/Checkbox.tsx#L68-L177)

```ts
export const CheckboxWithLabel = React.forwardRef<HTMLInputElement, CheckboxProps>(
  ({ defaultChecked, disabled, ...props }, ref) => {
    const [isChecked, setIsChecked] = useState<boolean>(defaultChecked ?? false);
    const [isDisabled, setIsDisabled] = useState<boolean>(disabled ?? false);

    useEffect(() => {
      setIsChecked(defaultChecked ?? false);
    }, [defaultChecked]);

    return (
      <div
        onClick={(e) => {
          if (isDisabled || props.readOnly === true) return false;
          setIsChecked((c) => !c);
        }}
      >
        <input
          type="checkbox"
          value={value}
          checked={isChecked}
          onChange={(e) => {
            if (isDisabled || props.readOnly === true) return false;
            setIsChecked(!isChecked);
          }}
        />
      </div>
    );
  }
);
```

### 1.2 状态初始化链

```
defaultChecked prop
       │
       ▼
useState(defaultChecked ?? false)   → isChecked 初始值
       │
       ▼
<input checked={isChecked} />       → 表单提交时根据 checked 决定是否提交 value
```

关键点：
- `isChecked` **完全由 `defaultChecked` 初始值决定**
- 后续状态变化仅通过 `setIsChecked` 触发
- `useEffect(() => setIsChecked(defaultChecked ?? false), [defaultChecked])` 会在 `defaultChecked` 变化时同步，但 billing-alerts 页面中 `defaultChecked` 不会变化（loader 数据在页面生命周期内固定）

### 1.3 readOnly 的精确行为

```ts
// onClick 阻止切换
onClick={(e) => {
  if (isDisabled || props.readOnly === true) return false;  // 直接返回，不调用 setIsChecked
  setIsChecked((c) => !c);
}}

// onChange 阻止切换
onChange={(e) => {
  if (isDisabled || props.readOnly === true) return false;  // 直接返回，不调用 setIsChecked
  setIsChecked(!isChecked);
}}
```

**`readOnly` 的行为：阻止从当前状态切换到任何其他状态。**

| 当前 isChecked | readOnly | 点击结果 | 含义 |
|---------------|----------|---------|------|
| `true`        | `true`   | 仍为 `true` | 不可取消 ✓ |
| `false`       | `true`   | 仍为 `false` | **不可选中** ✗ |
| `true`        | `false`  | 变为 `false` | 可取消 |
| `false`       | `false`  | 变为 `true`  | 可选中 |

**这是理解 100% 阈值的关键：`readOnly` 并不强制选中，而是冻结当前状态。**

### 1.4 readOnly 的视觉表现

```ts
// 容器样式
cn(
  props.readOnly || disabled ? "cursor-default" : "cursor-pointer",
  isChecked && isCheckedClassName,
  (isDisabled || props.readOnly) && isDisabledClassName  // "opacity-70"
)

// input 样式
"read-only:border-charcoal-650 read-only:!bg-charcoal-700"
```

- `readOnly` 时：鼠标变为 default（非 pointer），透明度降低到 70%，checkbox 背景变为 charcoal-700
- 如果 `isChecked === true`：仍有 `checked:!bg-indigo-500`（紫色填充）
- 如果 `isChecked === false`：仅有灰色边框和灰色背景

---

## 二、Billing Alerts 100% 阈值的精确判断逻辑

### 2.1 标准阈值复选框渲染

代码位置：[billing-alerts/route.tsx#L256-L275](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L256-L275)

```tsx
{checkboxLevels.map((level) => (
  <CheckboxWithLabel
    name={alertLevels.name}
    id={`level_${level}`}
    key={level}
    value={level.toString()}
    variant="simple/small"
    label={
      <span>
        {level * 100}%
        <span className="text-text-dimmed">
          ({formatCurrency(Number(dollarAmount) * level, false)})
        </span>
      </span>
    }
    defaultChecked={alerts.alertLevels.includes(level)}
    className="pr-0"
    readOnly={level === 1.0}
  />
))}
```

### 2.2 defaultChecked 的来源追踪

```
Billing API 返回
      │
      ▼
loader: const [error, alerts] = await tryCatch(getBillingAlerts(organization.id))
      │
      ▼
alerts = { amount: 美分, emails: [...], alertLevels: [...] }
      │
      ▼
return typedjson({ alerts: { ...alerts, amount: alerts.amount / 100 } })
      │
      ▼
组件: const { alerts } = useTypedLoaderData<typeof loader>()
      │
      ▼
defaultChecked={alerts.alertLevels.includes(level)}
```

**`defaultChecked` 的值完全取决于 Platform Billing API 返回的 `alertLevels` 数组。**

### 2.3 alertLevels 默认值的来源

代码位置：[getBillingAlerts](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L600-L610)

```ts
export async function getBillingAlerts(organizationId: string) {
  if (!client) return undefined;
  const result = await client.getBillingAlerts(organizationId);
  if (!result.success) {
    recordPlatformFailure("getBillingAlert", "no_success");
    throw new Error("Error getting billing alert");
  }
  return result;
}
```

- `getBillingAlerts` 直接调用 `@trigger.dev/platform`（v1.0.27）的 `BillingClient.getBillingAlerts()`
- `@trigger.dev/platform` 是外部 npm 包，不在本 monorepo 中
- **Webapp 代码无法控制 `alertLevels` 的默认值**
- `alertLevels` 的默认值由 Platform Billing API 在创建组织时初始化

### 2.4 100% 阈值的两种场景

根据 CheckboxWithLabel 的 readOnly 行为，100% 复选框有两种可能状态：

#### 场景 A：Billing API 返回 `alertLevels` 包含 `1.0`

```
alerts.alertLevels.includes(1.0) === true
  → defaultChecked={true}
  → isChecked 初始化为 true
  → readOnly={true} 锁定状态
  → 用户无法取消 100% 复选框
  → 表单提交时 value="1.0" 被包含在 alertLevels 中
  → ✅ 用户一定会收到 100% 邮件
```

#### 场景 B：Billing API 返回 `alertLevels` 不包含 `1.0`

```
alerts.alertLevels.includes(1.0) === false
  → defaultChecked={false}
  → isChecked 初始化为 false
  → readOnly={true} 锁定状态
  → 用户无法勾选 100% 复选框
  → 表单提交时 value="1.0" 不被包含在 alertLevels 中
  → ❌ 用户不会收到 100% 邮件
  → ❌ 用户无法通过 UI 启用 100% 邮件
```

**场景 B 会导致一个严重问题：readOnly 阻止了用户选中 100% 复选框，使得"100% 不可取消"变成了"100% 不可启用"。**

### 2.5 设计意图与代码实现的差距

| 设计意图                      | 代码实现                              | 差距 |
|------------------------------|--------------------------------------|------|
| 用户始终收到 100% 邮件通知     | `readOnly` 冻结当前状态，不强制选中     | 如果 API 默认不含 1.0，readOnly 会阻止启用 |
| 100% 阈值不可关闭             | 100% 复选框不可切换（选中或未选中都锁定）| "不可关闭"≠"始终开启" |
| 确保最低限度的告警覆盖         | 依赖 Billing API 默认包含 1.0          | 无客户端校验强制 1.0 在 alertLevels 中 |

**要实现"用户一定收到 100% 邮件"，需要同时满足两个条件：**
1. Billing API 在创建组织时默认将 `1.0` 加入 `alertLevels`
2. Webapp 端的 `readOnly` 防止用户取消

两个条件缺一不可。当前代码只实现了条件 2，条件 1 由外部服务保证，Webapp 无法验证。

### 2.6 服务端校验是否强制 1.0？

代码位置：[billing-alerts/route.tsx#L101-L104](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L101-L104)

```ts
alertLevels: z.preprocess((i) => {
  if (typeof i === "string") return [i];
  return i;
}, z.coerce.number().array().nonempty("At least one alert level is required")),
```

- 校验仅要求 `alertLevels` 是非空数组
- **没有校验 `1.0` 是否在数组中**
- 如果用户提交 `alertLevels: [0.75]`（不含 1.0），服务端校验会通过
- 而由于 readOnly 冻结了 100% 复选框为未选中状态，这确实可能发生

### 2.7 表单提交时 alertLevels 的组装机制

```tsx
<CheckboxWithLabel
  name={alertLevels.name}   // 所有标准阈值复选框共享同一个 name
  value={level.toString()}  // 每个复选框的 value 是阈值数字字符串
/>
```

HTML 表单提交规则：
- 同名复选框中，只有 `checked` 的复选框的 `value` 会被提交
- `isChecked` 状态决定是否提交
- 未选中的复选框不会提交任何值

所以 `alertLevels` 数组的值 = 所有 `isChecked === true` 的复选框的 `value` 组合。

---

## 三、"100% 不可取消"与"是否一定收到 100% 邮件"的关系

### 3.1 核心结论

**"100% 不可取消"≠"一定收到 100% 邮件"。**

`readOnly` 的语义是"冻结当前状态"，不是"强制选中"。能否收到 100% 邮件取决于：

```
能否收到 100% 邮件
  = 1.0 是否在提交的 alertLevels 数组中
  = 100% 复选框是否 isChecked
  = alerts.alertLevels.includes(1.0)  （初始值）
  = Billing API 返回的 alertLevels 是否包含 1.0
```

### 3.2 三层保障分析

| 保障层           | 实现方式                          | 是否保证 100% 邮件 | 备注                          |
|-----------------|-----------------------------------|-------------------|------------------------------|
| Webapp 前端      | `readOnly={level === 1.0}`        | ❌ 不保证          | 仅冻结，不强制选中             |
| Webapp 服务端    | `z.coerce.number().array().nonempty()` | ❌ 不保证     | 不校验 1.0 是否在数组中       |
| Billing API 后端 | 默认 alertLevels 包含 1.0          | ✅ 保证（如果确实如此）| 外部服务，代码不可见           |

### 3.3 可能的风险场景

```
1. 组织创建时 Billing API 初始化 alertLevels = [1.0]
   → 用户首次打开 Billing Alerts 页面 → 100% 已选中且不可取消 ✅
   → 用户提交表单 → alertLevels 包含 1.0 ✅
   → 100% 邮件一定会发送 ✅

2. Billing API 初始化 alertLevels = []
   → 用户首次打开页面 → 100% 未选中且不可选中 ❌
   → 用户提交表单 → alertLevels 不含 1.0 ❌
   → 100% 邮件不会发送 ❌

3. 之前的提交将 1.0 移除了（通过其他手段，如 API 直接调用）
   → 下次打开页面 → 100% 未选中且不可选中 ❌
   → readOnly 阻止恢复 ❌
```

### 3.4 尖峰告警的默认选中逻辑对比

代码位置：[billing-alerts/route.tsx#L303-L306](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L303-L306)

```tsx
defaultChecked={
  alerts.alertLevels.includes(level) ||
  !spikeAlertLevels.some((l) => alerts.alertLevels.includes(l))
}
```

尖峰告警有**回退默认值**：如果 `alertLevels` 中不包含任何尖峰阈值，则默认全部勾选。这是一种容错设计，确保即使用户从未配置过尖峰告警，也会默认启用。

**但标准阈值（包括 100%）没有这种回退逻辑。** `defaultChecked={alerts.alertLevels.includes(level)}` 完全依赖 API 返回值，没有"如果从未配置则默认启用"的逻辑。

---

## 四、免费用户与付费用户的邮件提醒时间线（修正版）

### 4.1 关键前提：Billing Alerts 需要主动配置邮箱

Billing Alerts 页面初始状态：
- `emails` 默认为 `[""]`（一个空字符串输入框）
- 表单校验要求至少一个有效邮箱
- **用户必须手动输入邮箱地址并点击 Update 才能启用邮件告警**

这意味着：**如果用户从未访问 Billing Alerts 页面并配置邮箱，即使 1.0 在 alertLevels 中，也不会收到任何邮件。** 邮件发送需要同时满足两个条件：
1. `alertLevels` 包含对应阈值
2. `emails` 中有有效的邮箱地址

### 4.2 免费用户完整时间线

```
用量 0%─────75%─────90%─────95%─────100%──────────→

产品内提示：
       │       │       │       │        │
       ▼       ▼       ▼       ▼        ▼
    FreePlan  FreePlan  FreePlan  FreePlan  UpgradePrompt
    绿色      琥珀色    琥珀色     红色      红色横幅
    (侧边栏)  (侧边栏)  (侧边栏)  (侧边栏)  (全局 NavBar)
                                         +红色边框

邮件提醒（前提：已配置邮箱 + alertLevels 包含对应阈值）：
       │               │       │        │
       ▼               ▼       ▼        ▼
    Billing          Billing  Billing  Billing
    Alert 75%        Alert    Alert    Alert 200%...
    (邮箱)           90%      100%     (邮箱)
                     (邮箱)   (邮箱)
                     
                                         │
                                         ▼
                                      API 阻断
                                      OutOfEntitlementError
                                      (仅非 DEV 环境)

注意：
- FreePlanUsage 75% 变琥珀色是产品内最早的提示（无需配置）
- Billing Alert 75% 邮件需要提前配置邮箱
- 如果未配置邮箱：唯一的产品内提示是 FreePlanUsage 颜色变化 + UpgradePrompt（超 100% 后）
```

### 4.3 付费用户完整时间线

```
用量 0%─────75%─────90%─────100%──────────→

产品内提示：
       │                       │       │
       │                       ▼       │
       │                    Limits    │
       │                    90%橙     │
       │                  (需主动查看) │

邮件提醒（前提：已配置邮箱 + alertLevels 包含对应阈值）：
       │               │       │        │
       ▼               ▼       ▼        ▼
    Billing          Billing  Billing  Billing
    Alert 75%        Alert    Alert    Alert 200%...
    (邮箱)           90%      100%     (邮箱)
                     (邮箱)   (邮箱)

付费用户注意：
  ✗ 没有 FreePlanUsage（侧边栏无进度条）
  ✗ 没有 UpgradePrompt（无超限横幅）
  ✓ 有 Billing Alerts（需配置邮箱）
  ✓ 有 Usage 页面（需主动查看，UsageBar 仅绿色深浅变化）
  ✓ 有 Limits 页面（需主动查看，90%/100% 数字变色）
  ✓ 付费用户超限时 API 不阻断（canExceed 机制）
```

### 4.4 免费用户 vs 付费用户的核心差异

| 维度                  | 免费用户                                   | 付费用户                                |
|----------------------|-------------------------------------------|----------------------------------------|
| 产品内持续提示         | FreePlanUsage 进度条（75%/95% 颜色变化）    | ❌ 无                                   |
| 产品内超限提示         | UpgradePrompt 红色横幅                      | ❌ 无                                   |
| 邮件提醒              | Billing Alerts（需配置邮箱）                 | Billing Alerts（需配置邮箱）             |
| 被动查看页面          | Usage / Limits                              | Usage / Limits                          |
| API 超限行为          | OutOfEntitlementError 阻断                   | canExceed=true 时不阻断，超出部分按量计费  |
| 首个产品内预警时机     | 75%（FreePlanUsage 变琥珀色）                | ❌ 无自动预警                            |
| 首个邮件预警时机      | 75%（如已配置 Billing Alerts）                | 75%（如已配置 Billing Alerts）            |

### 4.5 付费用户缺少持续提示的问题

付费用户在产品内**没有任何持续可见的用量提示**：
- 没有 FreePlanUsage 进度条
- 没有 UpgradePrompt 横幅
- UsageBar 仅在 Usage 页面可见，且只有绿色深浅变化
- Limits 页面需要主动导航
- 唯一的自动提醒是 Billing Alerts 邮件（需配置邮箱）

如果付费用户**没有配置 Billing Alerts 邮箱**，则：
- 75% 时：**无任何提示**
- 90% 时：**无任何提示**（除非主动查看 Limits 页面）
- 100% 时：**无任何提示**
- 仅当主动查看 Usage 或 Limits 页面才能发现用量状态

---

## 五、FreePlanUsage、UpgradePrompt、NotificationPanel、Billing Alerts 的职责边界

### 5.1 职责矩阵

| 通道                | 通道类型   | 目标用户    | 展示位置             | 触发时机         | 提醒方式          | 阈值体系                | 需要配置 | 产品内可见 |
|--------------------|-----------|------------|---------------------|-----------------|------------------|------------------------|---------|-----------|
| **FreePlanUsage**  | 产品内提示 | 仅免费用户  | 项目 SideMenu 底部   | 始终可见         | 进度条颜色变化     | 75% 绿→琥珀, 95%→红    | 否      | 是（展开时）|
| **UpgradePrompt**  | 产品内提示 | 仅免费用户  | 全局 NavBar 下方     | usage > included | 红色横幅+阻断文案  | 100%（无中间阈值）      | 否      | 是         |
| **Billing Alerts** | 邮件提醒   | 所有用户    | 组织设置页配置       | 阈值触发          | 邮件通知          | 75/90/100/200/500%+尖峰 | **是**  | 否（仅邮件）|
| **NotificationPanel**| 产品内提示| 所有用户    | 项目 SideMenu 底部   | 管理员手动创建    | 通知卡片          | 无内置配额逻辑          | 管理员  | 是         |

### 5.2 提示与邮件的功能边界

```
┌───────────────────────────────────────────────────────────────┐
│                    产品内提示（Dashboard 内）                    │
│                                                               │
│  ┌─────────────────┐    ┌─────────────────┐                  │
│  │ FreePlanUsage   │    │ UpgradePrompt   │                  │
│  │ 被动·持续·视觉   │    │ 被动·超限·强制   │                  │
│  │ 仅免费用户       │    │ 仅免费用户       │                  │
│  │ 不需配置         │    │ 不需配置         │                  │
│  └─────────────────┘    └─────────────────┘                  │
│           │                       │                          │
│           │  75%/95% 颜色          │ 100%+ 红色横幅            │
│           ▼                       ▼                          │
│  [预警] 进度条变色，无文字    [告知] 已被阻断，需升级           │
│                                                               │
│  ┌─────────────────┐    ┌─────────────────┐                  │
│  │ NotificationPanel│    │ Limits/Usage 页 │                  │
│  │ 可扩展·未启用    │    │ 被动·按需·详细   │                  │
│  │ 所有用户         │    │ 所有用户         │                  │
│  │ 管理员手动创建    │    │ 需主动导航       │                  │
│  └─────────────────┘    └─────────────────┘                  │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│                    邮件提醒（Dashboard 外）                      │
│                                                               │
│  ┌─────────────────────────────────────────────────┐         │
│  │ Billing Alerts                                   │         │
│  │ 主动推送·可配置·异步                               │         │
│  │ 所有用户                                          │         │
│  │ 需要用户配置邮箱 + 阈值                            │         │
│  │ 唯一脱离 Dashboard 的提醒通道                      │         │
│  └─────────────────────────────────────────────────┘         │
└───────────────────────────────────────────────────────────────┘
```

### 5.3 职责边界总结

**FreePlanUsage**：产品内·被动·持续的用量感知
- 职责：让免费用户在正常使用过程中随时感知用量进度
- 覆盖：仅免费用户，仅产品内，仅侧边栏展开时
- 局限：仅颜色变化无文字，折叠时不可见，不自动刷新，不发送邮件

**UpgradePrompt**：产品内·被动·强制的阻断告知
- 职责：在免费用户已被阻断后告知原因和解决方案
- 覆盖：仅免费用户，仅超 100% 后，全局可见
- 局限：无预警功能，不发送邮件，仅免费用户

**Billing Alerts**：邮件·主动·可配置的阈值推送
- 职责：在用户不看 Dashboard 时通过邮件获知用量状态
- 覆盖：**所有用户**，不依赖 Dashboard 在线
- 局限：**需要用户主动配置邮箱**，未配置则不会收到任何邮件
- 关键风险：100% 邮件是否必收取决于 Billing API 默认 alertLevels 是否包含 1.0

**NotificationPanel**：产品内·被动·可扩展的通知卡片
- 职责：展示管理员创建的产品内通知
- 覆盖：所有用户，SideMenu 内
- 现状：**未用于配额告警**，仅用于产品公告等

---

## 六、关键代码索引

| 功能                        | 文件                                                                                                              |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------|
| CheckboxWithLabel 组件      | [Checkbox.tsx#L68-L177](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/primitives/Checkbox.tsx#L68-L177) |
| isChecked 初始化             | [Checkbox.tsx#L87](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/primitives/Checkbox.tsx#L87) |
| readOnly 阻止 onClick       | [Checkbox.tsx#L123](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/primitives/Checkbox.tsx#L123) |
| readOnly 阻止 onChange      | [Checkbox.tsx#L135](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/primitives/Checkbox.tsx#L135) |
| readOnly 视觉样式            | [Checkbox.tsx#L115-L118](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/primitives/Checkbox.tsx#L115-L118) |
| 100% readOnly 设置           | [billing-alerts/route.tsx#L273](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L273) |
| 100% defaultChecked 来源    | [billing-alerts/route.tsx#L271](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L271) |
| alertLevels schema 校验     | [billing-alerts/route.tsx#L101-L104](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L101-L104) |
| 尖峰告警默认选中逻辑         | [billing-alerts/route.tsx#L303-L306](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L303-L306) |
| 标准阈值定义                | [billing-alerts/route.tsx#L185](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L185) |
| 尖峰阈值定义                | [billing-alerts/route.tsx#L187](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L187) |
| getBillingAlerts            | [platform.v3.server.ts#L600-L610](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L600-L610) |
| setBillingAlert             | [platform.v3.server.ts#L612-L623](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platform.v3.server.ts#L612-L623) |
| loader 中 alerts 数据处理   | [billing-alerts/route.tsx#L76-L81](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L76-L81) |
| isFree 判定                 | [billing-alerts/route.tsx#L194](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L194) |
| Amount 只读渲染             | [billing-alerts/route.tsx#L222-L251](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L222-L251) |
| 邮箱校验 schema            | [billing-alerts/route.tsx#L88-L100](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L88-L100) |
| 表单提交 amount × 100       | [billing-alerts/route.tsx#L134](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.billing-alerts/route.tsx#L134) |
| FreePlanUsage 组件          | [FreePlanUsage.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/FreePlanUsage.tsx) |
| SideMenu 渲染 FreePlanUsage | [SideMenu.tsx#L745-L752](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/SideMenu.tsx#L745-L752) |
| UpgradePrompt 横幅          | [UpgradePrompt.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/UpgradePrompt.tsx) |
| hasExceededFreeTier 计算    | [org route.tsx#L115-L120](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug/route.tsx#L115-L120) |
| NotificationPanel           | [NotificationPanel.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/NotificationPanel.tsx) |
| NotificationPanel 通知查询  | [platformNotifications.server.ts#L139-L205](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/services/platformNotifications.server.ts#L139-L205) |
| UsageBar 组件               | [UsageBar.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/billing/UsageBar.tsx) |
| Limits 页面颜色函数         | [limits/route.tsx#L809-L826](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.limits/route.tsx#L809-L826) |
| OrganizationSettings 侧边栏 | [OrganizationSettingsSideMenu.tsx#L107-L114](file:///d:/fz/0508-3/solo-dogfeeding/code/192-trigger.dev/apps/webapp/app/components/navigation/OrganizationSettingsSideMenu.tsx#L107-L114) |
