# Trigger.dev 审计事件与活动流 — 代码链路全解析

## 1. 事件分类体系

Trigger.dev 的可审计事件分为 **两大独立子系统**，各自拥有独立的数据模型、写入路径和查询方式。

### 1.1 任务运行事件（TaskEvent）— 主要活动流

这是系统中最核心、量级最大的事件流，记录了 Task 的完整生命周期，包括 Span、Log、异常、取消等。

**Prisma 模型**：[schema.prisma](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/internal-packages/database/prisma/schema.prisma#L1677-L1780) 中的 `TaskEvent` / `TaskEventPartitioned`

**分类维度**：

| 维度 | 枚举 | 值 | 用途 |
|------|------|----|------|
| Level | `TaskEventLevel` | `TRACE`, `DEBUG`, `LOG`, `INFO`, `WARN`, `ERROR` | 日志级别/显示样式 |
| Kind | `TaskEventKind` | `UNSPECIFIED`, `INTERNAL`, `SERVER`, `CLIENT`, `PRODUCER`, `CONSUMER`, `UNRECOGNIZED`, `LOG` | 事件来源分类 |
| Status | `TaskEventStatus` | `UNSET`, `OK`, `ERROR`, `UNRECOGNIZED` | 事件结果状态 |

**ClickHouse 中的 Kind 映射**（在 [clickhouseEventRepository.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/clickhouseEventRepository.server.ts#L668-L682) 中转换）：

| Prisma Kind/Level | ClickHouse Kind | 说明 |
|-------------------|-----------------|------|
| `level=TRACE` | `SPAN` | 追踪 Span |
| `isDebug=true` | `DEBUG_EVENT` | 调试日志 |
| `kind=UNSPECIFIED` | `ANCESTOR_OVERRIDE` | 祖先覆盖事件（不可见） |
| `level=INFO` | `LOG_INFO` | 信息日志 |
| `level=WARN` | `LOG_WARN` | 警告日志 |
| `level=ERROR` | `LOG_ERROR` | 错误日志 |
| Span Event (exception) | `SPAN_EVENT` | Span 附加事件 |
| Span Event (attempt_failed) | `ANCESTOR_OVERRIDE` | 重试失败覆盖 |

**ClickHouse Status 映射**：

| Prisma 字段组合 | ClickHouse Status |
|-----------------|-------------------|
| `isPartial=true` | `PARTIAL` |
| `isError=true` | `ERROR` |
| `isCancelled=true` | `CANCELLED` |
| 其他 | `OK` |

### 1.2 管理员模拟审计日志（ImpersonationAuditLog）— 合规专用

这是目前唯一的 **严格意义上** 的审计日志模型，专门记录管理员冒充操作。

**Prisma 模型**：[schema.prisma](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/internal-packages/database/prisma/schema.prisma#L2751-L2776) 中的 `ImpersonationAuditLog`

| 字段 | 类型 | 说明 |
|------|------|------|
| `action` | `ImpersonationAuditLogAction` (`START` / `STOP`) | 操作类型 |
| `adminId` | `String` → `User` | 执行冒充的管理员 |
| `targetId` | `String` → `User` | 被冒充的目标用户 |
| `ipAddress` | `String?` | 客户端 IP（从 `x-forwarded-for` 提取） |
| `createdAt` | `DateTime` | 自动时间戳 |

**索引**：`adminId`, `targetId`, `createdAt` — 支持按管理员、目标用户和时间范围查询。

**写入位置**：[admin.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/models/admin.server.ts#L213-L284) 中的 `redirectWithImpersonation()` 和 `clearImpersonation()`

---

## 2. 事件采集 → 归一 → 写入存储的完整链路

### 2.1 入口：两条事件采集通道

#### 通道 A：OTLP Exporter（SDK → 平台）

SDK 通过 OpenTelemetry Protocol (OTLP) 将 trace/log 数据推送到平台。

**入口文件**：[otlpExporter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/otlpExporter.server.ts)

```
SDK (OTLP Export)
  → otlpExporter.exportTraces(request)
    → convertSpansToCreateableEvents() — 将 OTLP ResourceSpans 转为 CreateEventInput[]
    → enrichCreatableEvents()          — LLM 成本增强
    → #exportEvents()                  — 选择存储后端写入
```

#### 通道 B：服务端事件录制（平台内部操作）

平台自身的服务端操作（如 run 完成、失败、取消等）通过 `EventRepository` 直接写入。

**入口文件**：[eventRepository.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/eventRepository.server.ts)

核心方法：
- `recordEvent()` — 录制单条事件（[L1075-L1148](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/eventRepository.server.ts#L1075-L1148)）
- `traceEvent()` — 录制带生命周期的追踪事件（[L1150-L1284](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/eventRepository.server.ts#L1150-L1284)）
- `completeSuccessfulRunEvent()` / `completeFailedRunEvent()` / `cancelRunEvent()` 等 — Run 生命周期事件

#### 通道 C：管理员审计（ImpersonationAuditLog）

独立通道，直接写入 PostgreSQL，不经过 EventRepository。

**写入代码**：[admin.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/models/admin.server.ts#L227-L241)

```typescript
await prisma.impersonationAuditLog.create({
  data: {
    action: "START",
    adminId: user.id,
    targetId: userId,
    ipAddress,
  },
});
```

### 2.2 归一化（Normalization）

无论哪条通道，事件最终都归一化为 `CreateEventInput` 类型。

**类型定义**：[eventRepository.types.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/eventRepository.types.ts#L51-L90)

关键字段归一化逻辑（在 `EventRepository.recordEvent` 中，[L1075-L1148](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/eventRepository.server.ts#L1075-L1148)）：

| 归一化步骤 | 说明 |
|------------|------|
| Trace Context 提取 | 从 `options.context` 中提取 `traceparent`/`tracestate`，获取 traceId 和 parentId |
| Span ID 生成 | 随机生成（`generateSpanId()`）或基于种子确定性生成（`generateDeterministicSpanId()`） |
| Metadata 组装 | 注入 `environmentId`, `environmentType`, `organizationId`, `projectId`, `projectRef`, `runId`, `taskSlug`, `serviceName`, `serviceNamespace` |
| Style 组装 | 注入 `icon`, `variant` 等展示元数据 |
| Properties 扁平化 | 使用 `flattenAttributes(metadata, SemanticInternalAttributes.METADATA)` 将嵌套属性展平 |

### 2.3 存储后端选择（Router）

**路由文件**：[index.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/index.server.ts)

支持三种存储后端：

| 后端 | 常量 | 实现类 | 适用场景 |
|------|------|--------|----------|
| PostgreSQL | `POSTGRES` | `EventRepository` | 默认/小规模 |
| ClickHouse V1 | `CLICKHOUSE` | `ClickhouseEventRepository` | 按时间分区 |
| ClickHouse V2 | `CLICKHOUSE_V2` | `ClickhouseEventRepository` | 按插入时间分区（避免 too many parts） |

**选择逻辑**（优先级从高到低）：

1. **Run 级别**：`run.taskEventStore` 字段指定（在 `findRunForEventCreation()` 中读取）
2. **组织级别 Feature Flag**：`organization.featureFlags.taskEventRepository`
3. **全局 Feature Flag**：`flag(FEATURE_FLAG.taskEventRepository)`
4. **环境变量兜底**：`env.EVENT_REPOSITORY_DEFAULT_STORE`

### 2.4 批量写入与资源隔离

写入不直接落盘，而是经过 `DynamicFlushScheduler` 批量调度。

**核心文件**：[dynamicFlushScheduler.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/dynamicFlushScheduler.server.ts)

**写入流程**：

```
CreateEventInput
  → addToBatch() — 入队 + 负载检查
  → currentBatch 达到 batchSize 或 flushInterval 到期
  → createBatch() — 生成批次
  → flushBatches() — 并发刷盘
    → limiter(pLimit) — 控制并发度
    → callback(flushId, batch) — 实际写入
      → EventRepository: #flushBatch → #doFlushBatch → TaskEventStore.createMany
      → ClickHouse: #flushBatch → clickhouse.taskEvents.insert
    → 写入后: #publishToRedis() — 通知实时订阅
```

**资源隔离机制**：

| 机制 | 参数 | 默认值 | 说明 |
|------|------|--------|------|
| 批量大小 | `EVENTS_BATCH_SIZE` | 100 | 每批事件数 |
| 刷盘间隔 | `EVENTS_BATCH_INTERVAL` | 1000ms | 定时刷盘周期 |
| 最小并发 | `EVENTS_MIN_CONCURRENCY` | 1 | 低负载时并发度 |
| 最大并发 | `EVENTS_MAX_CONCURRENCY` | 10 | 高负载时最大并发 |
| 最大批量 | `EVENTS_MAX_BATCH_SIZE` | 500 | 单批上限 |
| 内存压力阈值 | `EVENTS_MEMORY_PRESSURE_THRESHOLD` | 5000 | 触发扩容的队列深度 |
| 负载丢弃阈值 | `EVENTS_LOAD_SHEDDING_THRESHOLD` | 100000 | 触发丢事件的队列深度 |
| 负载丢弃开关 | `EVENTS_LOAD_SHEDDING_ENABLED` | "1" | 是否启用负载丢弃 |

**负载丢弃策略**（[L349-L367](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/dynamicFlushScheduler.server.ts#L349-L367)）：
- 当队列深度 ≥ `loadSheddingThreshold` 时启用
- **只丢弃 `LOG` 类型事件**（`isDroppableEvent` 检查 `kind === "LOG"` / ClickHouse 中 `kind === "DEBUG_EVENT"`）
- TRACE/SPAN 等关键事件始终保留
- 队列深度降至阈值 80% 以下时退出负载丢弃模式

**写入容错**（PostgreSQL 路径）：
- 二分法重试：`#doFlushBatch` 遇到可重试 Prisma 错误时，将批次对半拆分递归插入（最大深度 5 层）
- 单条失败不阻塞整批

**写入容错**（ClickHouse 路径）：
- JSON 解析恢复：遇到 `Cannot parse JSON object` 错误时，执行 UTF-16 清洗后重试一次
- 清洗仍失败则整批丢弃（`permanentlyDroppedBatches` 计数器追踪）

### 2.5 实时通知（Redis Pub/Sub）

写入成功后，通过 Redis Pub/Sub 通知订阅方。

**核心文件**：[tracePubSub.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/services/tracePubSub.server.ts)

- **发布**：`tracePubSub.publish(traceIds)` → 向 `events:{traceId}` 频道发布时间戳
- **订阅**：`tracePubSub.subscribeToTrace(traceId)` → 创建独立 Redis 连接订阅指定 trace
- 使用独立 Redis 实例（`PUBSUB_REDIS_*` 配置），与业务 Redis 隔离

---

## 3. Dashboard 活动流读取与按角色过滤

### 3.1 实时活动流（SSE）

**Presenter**：[RunStreamPresenter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/presenters/v3/RunStreamPresenter.server.ts)

```
浏览器 SSE 连接
  → RunStreamPresenter.createLoader()
    → tracePubSub.subscribeToTrace(run.traceId)
    → throttle(send, 1000ms) — 限流：最多 1 秒 1 次推送
    → SSE 心跳：5 秒间隔 ping
    → 超时：30 秒无活动断开
    → 清理：取消 Redis 订阅 + 关闭连接
```

### 3.2 Run 详情页面事件读取

**Presenter**：[RunPresenter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/presenters/v3/RunPresenter.server.ts)

```
RunPresenter.call({ userId, projectSlug, environmentSlug, runFriendlyId })
  → prisma.taskRun.findFirstOrThrow() — 先查 Run（含成员关系验证）
  → resolveEventRepositoryForStore(run.taskEventStore)
  → eventRepository.getTraceSummary() — 获取 Span 树
```

**关键**：`userId` + `projectSlug` + 组织成员关系用于 **数据隔离**（只返回用户所属组织的数据），而非基于角色的字段级过滤。

### 3.3 Logs 页面（全局日志视图）

**Presenter**：[LogsListPresenter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/presenters/v3/LogsListPresenter.server.ts)

查询强制走 ClickHouse V2（PostgreSQL 后端直接报错），支持：
- 按 task、runId、level、时间范围过滤
- 全文搜索（message + attributes_text）
- 游标分页
- **保留期过滤**：`retentionLimitDays` 参数限制查询时间范围不早于保留期

### 3.4 Span 详情

**Presenter**：[SpanPresenter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/presenters/v3/SpanPresenter.server.ts)

```
SpanPresenter.call({ userId, projectSlug, spanId, runFriendlyId })
  → 验证用户权限
  → eventRepository.getSpan() — 获取单 Span 详情
```

### 3.5 RBAC 与角色过滤

**RBAC 核心**：[rbac.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/services/rbac.server.ts) + [dashboardBuilder.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/services/routeBuilders/dashboardBuilder.server.ts)

- 使用 `@trigger.dev/rbac` 插件，基于 Prisma 的 `OrganizationMember` 角色
- **Dashboard 路由**统一通过 `authenticateAndAuthorize()` 检查：认证 → 授权（`ability.can(action, resource)`）
- 支持 `requireSuper` 超级管理员检查
- **读权限**：到组织/项目级别，非行级/字段级
- **事件数据**不按角色做字段级脱敏；同一组织内所有成员看到相同的 Span/Log 内容

### 3.6 导出能力

**日志下载**：[resources.runs.$runParam.logs.download.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/routes/resources.runs.$runParam.logs.download.ts)

- Gzip 压缩的纯文本流导出
- **角色过滤**：非 admin 用户不导出 `kind=LOG`（debug）事件（[L52-L54](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/routes/resources.runs.$runParam.logs.download.ts#L52-L54)）
- 格式：`<timestamp> <taskSlug> <level> <message> (<duration>)`

**API 导出**：`/api/v1/runs/$runId/events` 和 `/api/v1/runs/$runId/trace` 提供程序化访问。

---

## 4. 上下文字段的脱敏策略

### 4.1 私有属性过滤（$ 前缀与 ctx. 前缀）

**核心函数**：[common.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/common.server.ts#L145-L167) 中的 `removePrivateProperties()`

```typescript
export function removePrivateProperties(attributes: Attributes | undefined | null): Attributes | undefined {
  const result: Attributes = {};
  for (const [key, value] of Object.entries(attributes)) {
    if (key.startsWith("$") || key.startsWith("ctx.")) {
      continue; // 跳过私有属性
    }
    result[key] = value;
  }
  return Object.keys(result).length === 0 ? undefined : result;
}
```

**过滤规则**：
- **`$` 前缀属性**：包括 `$style.icon`, `$style.variant`, `$metadata.*`, `$output`, `$resource` 等内部元数据
- **`ctx.` 前缀属性**：运行时上下文数据

**应用位置**：
1. **写入时**：ClickHouse 路径中，`createEventToTaskEventV1InputAttributes()` 调用 `removePrivateProperties()` 后再 `unflattenAttributes()` 写入 `attributes` 列
2. **读取时**：`sanitizedAttributes()` 在 `EventRepository.getSpan()` 中（[L1717-L1728](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/eventRepository.server.ts#L1717-L1728)）再次过滤
3. **详情视图**：`getTraceDetailedSummary()` 中也调用 `removePrivateProperties()`（[L635](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/eventRepository.server.ts#L635)）

### 4.2 堆栈跟踪修正

**函数**：`transformException()` → `correctErrorStackTrace()`（[L1747-L1767](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/eventRepository.server.ts#L1747-L1767)）

- 使用 `SemanticInternalAttributes.PROJECT_DIR` 替换堆栈中的本地路径
- 在开发环境 (`isDev`) 中执行更激进的处理
- 避免暴露服务器文件系统路径

### 4.3 Debug 日志可控

**环境变量**：`EVENT_REPOSITORY_DEBUG_LOGS_DISABLED`（[env.server.ts L1487](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/env.server.ts#L1487)）

- 设为 `true` 时，`recordRunDebugLog()` 直接返回成功但不写入，静默丢弃 debug 事件
- 在查询路径中，`includeDebugLogs: false` 选项会排除 `kind=DEBUG_EVENT` / `kind=LOG` 的事件

### 4.4 当前不足（合规审计视角）

| 方面 | 现状 | 差距 |
|------|------|------|
| 字段级脱敏 | 仅过滤 `$`/`ctx.` 前缀 | 无 PII 自动识别/掩码 |
| 角色字段可见性 | 非admin不导出debug日志 | 无细粒度字段级 RBAC |
| 审计日志范围 | 仅 ImpersonationAuditLog | 缺少配置变更、权限变更、API 调用等审计事件 |
| 不可篡改性 | 无特殊保护 | 审计日志存储在可写的 PostgreSQL 表中 |

---

## 5. 保留期与过期机制

### 5.1 PostgreSQL 路径

- **配置**：`EVENTS_DEFAULT_LOG_RETENTION` 环境变量，默认 **7 天**
- 此值传入 `EventRepoConfig.retentionInDays`，但**在代码中未发现自动化清理逻辑**
- PostgreSQL 的 `TaskEvent` 表未配置分区或 TTL
- `TaskEventPartitioned` 表按 `createdAt` 范围查询以利用分区裁剪，但无自动过期

### 5.2 ClickHouse 路径

- **TTL 声明**（[007_add_task_events_v1.sql](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/internal-packages/clickhouse/schema/007_add_task_events_v1.sql#L48-L49)）：
  ```sql
  TTL toDateTime(expires_at) + INTERVAL 7 DAY
  SETTINGS ttl_only_drop_parts = 1;
  ```
- **`expires_at` 计算**（写入时设定）：
  - Run 生命周期事件：`run.createdAt + 30 天`
  - 其他事件：`Date.now() + 365 天`（1 年）
  - 代码中有 `// TODO: make sure configurable and by org` 注释，表明保留期目前不可按组织配置
- **实际 TTL**：`expires_at + 7 天` 后 ClickHouse 在后台合并时自动删除过期 Part
- V2 表同样使用此 TTL 策略（[010_add_task_events_v2.sql](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/internal-packages/clickhouse/schema/010_add_task_events_v2.sql#L50-L51)）

### 5.3 ImpersonationAuditLog

- **无 TTL/过期机制**：PostgreSQL 中无分区、无过期字段、无清理 Job
- 数据永久保留，直到手动删除

### 5.4 LogsListPresenter 保留期限制

在查询时通过 `retentionLimitDays` 参数限制查询范围（[L163-L169](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/presenters/v3/LogsListPresenter.server.ts#L163-L169)），将 `effectiveFrom` 钳制到保留截止日期之后。这是应用层面的软限制，非存储层面的硬过期。

---

## 6. 高并发下事件写入与查询的资源隔离

### 6.1 写入隔离

| 隔离层 | 机制 | 细节 |
|--------|------|------|
| **内存缓冲** | `DynamicFlushScheduler` | 事件先入内存队列，定时/定量刷盘，避免逐条写入 |
| **并发控制** | `pLimit` | 动态调整并发度（1~10），低负载降为 1，高压力自动扩至最大 |
| **负载丢弃** | `isDroppableEvent` | 只丢弃 LOG/DEBUG_EVENT，SPAN 等关键事件永不丢弃 |
| **批次大小** | 动态调整 | 压力高时 `currentBatchSize = min(maxBatchSize, batchSize × (1 + queuePressure))` |
| **容错降级** | 二分法重试（PG）/ JSON 清洗重试（CH） | 单条坏数据不阻塞整批 |
| **存储隔离** | PostgreSQL 分表 | `TaskEvent` vs `TaskEventPartitioned`，后者按 `createdAt` 分区 |
| **存储隔离** | ClickHouse 分区 | V1 按 `start_time` 分区，V2 按 `inserted_at` 分区（避免晚到事件导致 too many parts） |
| **异步写入** | ClickHouse `async_insert` | 可配置异步插入，不等待数据落盘即返回 |

### 6.2 查询隔离

| 隔离层 | 机制 | 细节 |
|--------|------|------|
| **读写分离** | Prisma `$replica` | 读操作走 Read Replica，写操作走 Primary |
| **租户隔离** | `organizationId` + `environmentId` + `projectId` | 所有查询均包含租户条件 |
| **时间范围裁剪** | `startCreatedAt` / `endCreatedAt` | 利用分区裁剪减少扫描量 |
| **Bloom Filter 索引** | ClickHouse `bloom_filter` | `run_id`, `span_id` 列建有 Bloom Filter 加速查询 |
| **全文索引** | ClickHouse `tokenbf_v1` | `attributes_text` 列建 Token Bloom Filter 支持搜索 |
| **查询限制** | `maximumTraceSummaryViewCount` / `maximumTraceDetailedSummaryViewCount` | 限制单次查询返回行数（默认 25000 / 5000） |
| **实时流限流** | `throttle(send, 1000ms)` | SSE 推送最多 1 秒 1 次 |

### 6.3 监控指标

通过 Prometheus Gauge 暴露：

| 指标名 | 说明 |
|--------|------|
| `event_repository_subscriber_count` | 实时订阅者数量 |
| `event_flush_scheduler_queued_items` | 队列中待刷盘事件数 |
| `event_flush_scheduler_batch_queue_length` | 待刷盘批次数 |
| `event_flush_scheduler_concurrency` | 当前并发度 |
| `event_flush_scheduler_active_flushes` | 活跃刷盘操作数 |
| `event_flush_scheduler_dropped_events` | 因负载丢弃的事件总数 |
| `event_flush_scheduler_is_load_shedding` | 是否正在负载丢弃 |
| `trace_pub_sub_subscribers` | Redis Pub/Sub 订阅者数量 |

---

## 7. 端到端代码链路总结

### 完整写入链路

```
┌─ SDK (Worker) ──────────────────────────────────────────────────────┐
│  otel.trace / otel.log                                              │
│  → OTLP HTTP POST /otel/v1/traces                                   │
└────────────────────────────┬────────────────────────────────────────┘
                             │
┌─ Webapp Server ────────────▼────────────────────────────────────────┐
│  otlpExporter.exportTraces()                                        │
│    → convertSpansToCreateableEvents() — OTLP → CreateEventInput[]   │
│    → enrichCreatableEvents() — LLM 成本增强                         │
│    → resolveEventRepositoryForStore() — 选择 PG / CH                │
│                                                                      │
│  ─── PostgreSQL 路径 ───                                            │
│  EventRepository.insertMany()                                       │
│    → flushScheduler.addToBatch()                                    │
│    → DynamicFlushScheduler.flushBatches()                           │
│    → #doFlushBatch() → TaskEventStore.createMany()                  │
│    → tracePubSub.publish(traceIds)                                  │
│                                                                      │
│  ─── ClickHouse 路径 ───                                           │
│  ClickhouseEventRepository.insertMany()                             │
│    → createEventToTaskEventV1Input() — 归一化 + removePrivateProps  │
│    → flushScheduler.addToBatch()                                    │
│    → #flushBatch() → clickhouse.taskEvents.insert()                 │
│    → tracePubSub.publish(traceIds)                                  │
└──────────────────────────────────────────────────────────────────────┘

┌─ Server-side Events ───────────────────────────────────────────────┐
│  completeSuccessfulRunEvent() / completeFailedRunEvent() / ...      │
│    → insertImmediate() → #flushBatch() → ...                       │
└──────────────────────────────────────────────────────────────────────┘

┌─ Impersonation Audit ──────────────────────────────────────────────┐
│  admin.server.ts:redirectWithImpersonation()                       │
│    → prisma.impersonationAuditLog.create() — 直写 PG              │
└──────────────────────────────────────────────────────────────────────┘
```

### 完整读取链路

```
┌─ Dashboard (浏览器) ───────────────────────────────────────────────┐
│                                                                      │
│  Run 详情页:                                                         │
│    RunPresenter.call({ userId, ... })                               │
│      → prisma.taskRun.findFirstOrThrow() — 成员验证                 │
│      → eventRepository.getTraceSummary() — Span 树                  │
│      → eventRepository.getSpan() — Span 详情                        │
│                                                                      │
│  实时流:                                                             │
│    RunStreamPresenter.createLoader()                                │
│      → tracePubSub.subscribeToTrace(traceId) — Redis 订阅          │
│      → SSE + throttle(1s)                                           │
│                                                                      │
│  Logs 页面:                                                          │
│    LogsListPresenter.call()                                         │
│      → clickhouse.taskEventsSearch.logsListQueryBuilder()           │
│      → retentionLimitDays 裁剪                                      │
│                                                                      │
│  日志下载:                                                           │
│    resources.runs.$runParam.logs.download.ts                        │
│      → eventRepository.getRunEvents()                               │
│      → 非admin跳过debug日志 → gzip 流                               │
│                                                                      │
│  RBAC:                                                               │
│    dashboardBuilder.server.ts:authenticateAndAuthorize()            │
│      → rbac.authenticateSession() → RbacAbility                    │
│      → ability.can(action, resource) — 组织/项目级授权             │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 8. 关键代码文件索引

| 文件 | 职责 |
|------|------|
| [eventRepository.types.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/eventRepository.types.ts) | 事件类型定义与接口 |
| [eventRepository.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/eventRepository.server.ts) | PostgreSQL 事件仓库实现 |
| [clickhouseEventRepository.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/clickhouseEventRepository.server.ts) | ClickHouse 事件仓库实现 |
| [index.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/index.server.ts) | 存储后端路由 |
| [common.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/common.server.ts) | 归一化/脱敏工具函数 |
| [dynamicFlushScheduler.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/dynamicFlushScheduler.server.ts) | 批量写入调度器 |
| [taskEventStore.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/taskEventStore.server.ts) | PostgreSQL 分表抽象 |
| [tracePubSub.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/services/tracePubSub.server.ts) | Redis 实时通知 |
| [otlpExporter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/otlpExporter.server.ts) | OTLP 入口 |
| [admin.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/models/admin.server.ts) | 管理员审计日志写入 |
| [rbac.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/services/rbac.server.ts) | RBAC 插件 |
| [dashboardBuilder.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/services/routeBuilders/dashboardBuilder.server.ts) | Dashboard 认证授权 |
| [RunPresenter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/presenters/v3/RunPresenter.server.ts) | Run 详情 Presenter |
| [RunStreamPresenter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/presenters/v3/RunStreamPresenter.server.ts) | 实时流 Presenter |
| [SpanPresenter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/presenters/v3/SpanPresenter.server.ts) | Span 详情 Presenter |
| [LogsListPresenter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/presenters/v3/LogsListPresenter.server.ts) | 日志列表 Presenter |
| [schema.prisma](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/internal-packages/database/prisma/schema.prisma#L1677) | TaskEvent / ImpersonationAuditLog 模型 |
| [007_add_task_events_v1.sql](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/internal-packages/clickhouse/schema/007_add_task_events_v1.sql) | ClickHouse task_events_v1 DDL + TTL |
| [010_add_task_events_v2.sql](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/internal-packages/clickhouse/schema/010_add_task_events_v2.sql) | ClickHouse task_events_v2 DDL + TTL |
| [env.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/env.server.ts#L469-L477) | 批量写入/保留期配置 |
