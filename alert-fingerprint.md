# 告警 Error Fingerprint 聚合处理机制分析

## 一、核心概念

系统采用 **Sentry 风格**的错误指纹（fingerprint）机制，将相似错误聚合为同一错误组（Error Group），避免重复告警。整个流程串联了三个核心环节：

1. **指纹生成**：任务运行出错时计算唯一标识
2. **告警去重**：基于指纹 + 状态机判断是否需要告警
3. **通知派发**：通过多通道（Email/Slack/Webhook）发送告警

---

## 二、Fingerprint 生成规则

### 2.1 算法入口

文件：`apps/webapp/app/utils/errorFingerprinting.ts`

```typescript
// 核心函数
calculateErrorFingerprint(error: unknown): string
```

### 2.2 指纹输入构成

指纹由三部分拼接后进行 SHA-256 哈希，取前 16 位字符：

```
指纹输入 = errorType + ":" + normalizedMessage + ":" + normalizedStack
```

**示例**：
```
"Error:Connection timeout:at db.connect (db.ts:_:_)|at query (query.ts:_:_)"
→ SHA-256 → 取前16位 → "a1b2c3d4e5f6a7b8"
```

### 2.3 消息规范化（normalizeErrorMessage）

**核心思想**：将动态值替换为占位符，使同类错误归为一组。

| 匹配模式 | 替换为 | 示例 |
|---------|--------|------|
| UUID (8-4-4-4-12) | `<uuid>` | `550e8400-e29b-41d4-a716-446655440000` → `<uuid>` |
| Run ID (`run_xxx`) | `<run-id>` | `run_abc1234xyz` → `<run-id>` |
| 任务友好 ID (`xxx_xxxxxx`) | `<id>` | `task_abc12345678` → `<id>` |
| ISO 8601 时间戳 | `<timestamp>` | `2024-03-01T15:30:45Z` → `<timestamp>` |
| Unix 时间戳 (10/13 位) | `<timestamp>` | `1234567890` → `<timestamp>` |
| URL | `<url>` | `https://api.example.com/users/123` → `<url>` |
| 4+ 位数字 ID | `<id>` | `12345` → `<id>` |
| 文件路径 (Unix) | `<path>` | `/home/user/project/file.ts` → `<path>` |
| 文件路径 (Windows) | `<path>` | `C:\Users\John\file.ts` → `<path>` |
| 邮箱 | `<email>` | `user@example.com` → `<email>` |
| 内存地址 | `<addr>` | `0x7fff5fbffab0` → `<addr>` |
| 长引号字符串 (20+ 字符) | `<string>` | `"very long dynamic content"` → `"<string>"` |

**执行顺序注意**：特定模式（URL、时间戳）必须在通用模式（数字、路径）之前执行，避免提前被错误替换。

### 2.4 堆栈规范化（normalizeStackTrace）

对堆栈轨迹做以下处理：

1. **只取前 5 帧**：忽略深层调用栈，聚焦错误发生点
2. **移除行号列号**：`file.ts:123:45` → `file.ts:_:_`
3. **移除独立数字**：所有数字替换为 `_`
4. **保留文件名，移除路径**：`/home/user/project/src/file.ts` → `file.ts`
5. **用 `|` 连接各帧**：形成紧凑字符串

**示例**：
```
原始堆栈:
  at functionName (/home/user/project/src/file.ts:123:45)
  at anotherFunction (/home/user/project/src/other.ts:67:89)

规范化后:
  "at functionName (file.ts:_:_)|at anotherFunction (other.ts:_:_)"
```

### 2.5 指纹稳定性测试保证

文件：`apps/webapp/test/errorFingerprinting.test.ts`

测试覆盖的关键场景：
- 仅时间戳不同的错误 → 相同指纹
- 仅 URL 路径不同的错误 → 相同指纹
- 仅用户 ID 不同的错误 → 相同指纹
- 不同错误类型 → 不同指纹
- 不同堆栈轨迹 → 不同指纹
- 堆栈行号变化 → 相同指纹

---

## 三、数据写入：从任务出错到 ClickHouse 存储

### 3.1 写入流程

任务运行出错时，`error_fingerprint` 随任务运行数据一起写入 `task_runs_v2` 表。通过**物化视图（Materialized View）**自动聚合到两张错误表：

```
task_runs_v2（原始任务运行表）
    ↓
    ├─ 物化视图 errors_mv_v1 → errors_v1（错误组预聚合）
    └─ 物化视图 error_occurrences_mv_v1 → error_occurrences_v1（分钟级桶表）
```

### 3.2 两张 ClickHouse 表的职责区分

| 维度 | `errors_v1` | `error_occurrences_v1` |
|------|------------|-----------------------|
| **引擎** | `AggregatingMergeTree()` | `SummingMergeTree(count)` |
| **主键** | `(org, project, env, task, fingerprint)` | `(org, project, env, task, fingerprint, version, minute)` |
| **聚合粒度** | 错误组级别（跨时间） | 分钟 + 任务版本级别 |
| **存储内容** | 聚合状态（State）：<br>`first_seen`(minState)<br>`last_seen`(maxState)<br>`occurrence_count`(sumState)<br>`sample_*`(anyState) | 原始计数字段：<br>`count UInt64` |
| **查询方式** | 用 `-Merge` 函数合并状态：<br>`sumMerge(occurrence_count)`<br>`minMerge(first_seen)`<br>`maxMerge(last_seen)` | 直接 `sum(count)` 聚合 |
| **适用场景** | 错误组列表、首次/末次出现时间、总次数 | 时间范围统计、忽略阈值检查、趋势图表 |
| **TTL** | 90 天 | 90 天 |

### 3.3 `errors_v1` 详解

**表定义**：`internal-packages/clickhouse/schema/022_create_errors_v1_table.sql`

```sql
CREATE TABLE trigger_dev.errors_v1 (
  occurrence_count AggregateFunction(sum, UInt64),  -- 存储 sumState 二进制状态
  first_seen SimpleAggregateFunction(min, DateTime64(3)),
  last_seen SimpleAggregateFunction(max, DateTime64(3)),
  ...
) ENGINE = AggregatingMergeTree()
ORDER BY (organization_id, project_id, environment_id, task_identifier, error_fingerprint)
```

**关键特点**：
- `AggregatingMergeTree` 存储的是**聚合函数的状态（State）**，不是最终结果
- 写入时使用 `sumState()`、`minState()`、`maxState()` 等函数
- 查询时必须使用对应的 `sumMerge()`、`minMerge()`、`maxMerge()` 合并状态
- 按错误组主键自动合并，相同组的多行数据会在后台合并为一行

### 3.4 `error_occurrences_v1` 详解

**表定义**：`internal-packages/clickhouse/schema/023_create_error_occurrences_v1.sql`

```sql
CREATE TABLE trigger_dev.error_occurrences_v1 (
  count UInt64,  -- 直接存储数值
  ...
) ENGINE = SummingMergeTree(count)
ORDER BY (..., minute, task_version)
```

**关键特点**：
- `SummingMergeTree` 直接存储数值，按主键合并时自动求和 `count` 字段
- 按分钟分桶，支持按时间范围精确统计
- 包含 `task_version`，支持按版本拆分统计

### 3.5 告警评估时的查询使用

**查询活跃错误（errors_v1）**：
```sql
SELECT
  environment_id, task_identifier, error_fingerprint,
  any(error_type) as error_type,
  any(error_message) as error_message,
  toString(toUnixTimestamp64Milli(minMerge(first_seen))) as first_seen,
  toString(toUnixTimestamp64Milli(maxMerge(last_seen))) as last_seen,
  toUInt64(sumMerge(occurrence_count)) as occurrence_count
FROM trigger_dev.errors_v1
GROUP BY environment_id, task_identifier, error_fingerprint
HAVING toInt64(maxMerge(last_seen)) > {scheduledAt}
```

**查询时间段内发生次数（error_occurrences_v1）**：
```sql
SELECT
  environment_id, task_identifier, error_fingerprint,
  sum(count) as occurrences_since
FROM trigger_dev.error_occurrences_v1
WHERE minute >= toStartOfMinute(fromUnixTimestamp64Milli({scheduledAt}))
GROUP BY environment_id, task_identifier, error_fingerprint
```

---

## 四、告警去重：状态机判断流程

### 4.1 核心评估器

文件：`apps/webapp/app/v3/services/alerts/errorAlertEvaluator.server.ts`

```typescript
class ErrorAlertEvaluator {
  async evaluate(projectId: string, scheduledAt: number): Promise<void>
}
```

### 4.2 错误组状态（ErrorGroupState）

Postgres 表 `errorGroupState` 维护每个错误组的生命周期状态：

| 状态 | 说明 | 触发条件 |
|------|------|----------|
| `UNRESOLVED` | 未解决，活跃错误 | 新问题、回归、取消忽略后 |
| `RESOLVED` | 已解决 | 用户手动标记解决 |
| `IGNORED` | 已忽略 | 用户手动忽略，支持三种忽略策略 |

**主键**：`(environmentId, taskIdentifier, errorFingerprint)`

### 4.3 告警触发分类（ErrorClassification）

只有以下三种情况会触发告警：

#### 1. `new_issue`（新问题）
- **条件**：Postgres 中无对应 `errorGroupState` 记录，且 `first_seen > scheduledAt`
- **含义**：该指纹在当前环境 + 任务中首次出现

#### 2. `regression`（回归）
- **条件**：状态为 `RESOLVED`，且 `last_seen > resolvedAt`
- **含义**：已解决的错误重新出现

#### 3. `unignored`（取消忽略）
- **条件**：状态为 `IGNORED`，且满足以下任一条件：
  - 忽略到期时间已过（`ignoredUntil <= now`）
  - 错误发生率超过阈值（`ignoredUntilOccurrenceRate`）
  - 累计发生次数超过阈值（`ignoredUntilTotalOccurrences`）

### 4.4 评估完整流程

```
1. 获取项目所有启用的 ERROR_GROUP 告警通道
2. 确定最小评估间隔（minIntervalMs，默认 300s）
3. 从 errors_v1 查询活跃错误（last_seen > scheduledAt）
   → 按 (env, task, fingerprint) 分组
4. 查询这些错误组的现有状态（Postgres errorGroupState）
5. 查询 error_occurrences_v1 获取时间段内发生次数
   → 用于检查 IGNORED 状态的阈值是否突破
6. 对每个活跃错误进行分类判断：
   - 无状态记录 + first_seen 是新的 → new_issue
   - RESOLVED 状态 + last_seen > resolvedAt → regression
   - IGNORED 状态 + 阈值突破 → unignored
7. 对需要告警的错误，为每个匹配的通道派发通知任务
8. 更新 errorGroupState 状态为 UNRESOLVED（或创建新记录）
9. 自链（self-chain）：安排下一次评估
```

### 4.5 去重键设计

**状态查找键**：
```
key = `${environmentId}:${taskIdentifier}:${errorFingerprint}`
```

**说明**：
- 同一错误在不同环境（production/staging）视为不同错误组
- 同一错误在不同任务中视为不同错误组
- 相同指纹、相同环境、相同任务 → 同一错误组

---

## 五、通知派发：多通道发送

### 5.1 任务编排（Alerts Worker）

文件：`apps/webapp/app/v3/alertsWorker.server.ts`

告警系统基于 Redis Worker 实现异步任务队列，关键任务类型：

| 任务类型 | 触发时机 | 说明 |
|---------|----------|------|
| `v3.evaluateErrorAlerts` | 周期性自链调度 | 错误告警评估入口 |
| `v3.deliverErrorGroupAlert` | 评估后派发 | 实际发送通知 |
| `v3.performTaskRunAlerts` | 任务运行结束 | 任务级告警（与错误组无关） |

### 5.2 派发服务

文件：`apps/webapp/app/v3/services/alerts/deliverErrorGroupAlert.server.ts`

```typescript
class DeliverErrorGroupAlertService {
  async call(payload: ErrorAlertPayload): Promise<void>
}
```

支持三种通知通道：

#### 1. Email
- 使用 `ProjectAlertEmailProperties` 解析配置
- 调用 `sendAlertEmail` 发送邮件
- 包含错误详情、发生次数、跳转链接

#### 2. Slack
- 使用 `ProjectAlertSlackProperties` 解析配置
- 构建结构化 Slack 消息（blocks + attachments）
- 包含：错误分类标签、堆栈预览、任务/环境/项目信息、发生次数、Investigate 按钮
- 支持 Slack 速率限制和平台错误处理

#### 3. Webhook
- 使用 `ProjectAlertWebhookProperties` 解析配置
- 生成标准化 Webhook payload
- **安全签名**：使用 HMAC-SHA256 对 payload 签名，`x-trigger-signature-hmacsha256` 头部
- 5 秒超时保护

### 5.3 任务去重（Idempotency）

派发任务的 ID 设计保证幂等：

```
任务ID = `deliverErrorGroupAlert:${channelId}:${errorFingerprint}:${scheduledAt}`
```

- 同一通道、同一错误、同一评估周期 → 同一任务 ID
- Redis Worker 保证相同 ID 的任务不会重复入队

---

## 六、完整调用链

```
任务运行出错
    ↓
[计算 error_fingerprint] （任务执行时）
    ↓
写入 task_runs_v2 表
    ↓
[物化视图自动聚合]
    ├─ errors_v1（AggregatingMergeTree，按错误组聚合）
    └─ error_occurrences_v1（SummingMergeTree，按分钟桶聚合）
    ↓
[周期调度] evaluateErrorAlerts（默认每 5 分钟）
    ↓
查询 errors_v1 获取活跃错误（按 fingerprint 分组）
    ↓
查询 errorGroupState（Postgres）获取状态
    ↓
查询 error_occurrences_v1 获取时间段内次数（用于 IGNORED 阈值检查）
    ↓
分类判断：new_issue / regression / unignored
    ↓
是 → 入队 deliverErrorGroupAlert 任务
    ↓
更新 errorGroupState 为 UNRESOLVED
    ↓
Alerts Worker 消费任务
    ↓
按通道类型发送：Email / Slack / Webhook
```

---

## 七、关键设计要点

### 7.1 为什么这样设计？

| 设计决策 | 原因 |
|---------|------|
| 指纹取 SHA-256 前 16 位 | 16 位十六进制 = 64 位熵，足够唯一且存储紧凑 |
| 只取前 5 帧堆栈 | 错误发生点通常在调用栈顶部，深层栈帧变化不影响错误本质 |
| 状态机存储在 Postgres | 需要事务保证和频繁更新，ClickHouse 不适合 |
| AggregatingMergeTree + SummingMergeTree 双表 | `errors_v1` 存聚合状态用于快速列表查询；`error_occurrences_v1` 存分钟级计数用于精确时间范围统计 |
| 物化视图自动写入 | 数据写入 `task_runs_v2` 后自动聚合，无需业务代码处理 |
| 自链（self-chain）调度 | 避免使用外部 cron，每个项目独立调度，支持动态间隔 |

### 7.2 潜在注意点

1. **指纹碰撞**：16 位哈希理论上存在碰撞可能，但实际场景下可忽略
2. **规范化粒度**：过于激进的规范化可能导致不同错误被错误归组
3. **评估间隔**：默认 5 分钟，告警存在最多 5 分钟延迟
4. **忽略阈值计算**：基于滑动窗口，窗口大小 = 当前时间 - 上次调度时间
5. **AggregatingMergeTree 查询**：必须使用 `-Merge` 函数，否则得到的是二进制状态而非可读数值
