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
---

## 9. Dashboard 首页任务 Activity 图 — 环境指标到 ClickHouse 查询的读取链路

### 9.1 前端入口

**路由**：[route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam._index/route.tsx#L109-L135)

Dashboard 首页（环境 `_index` 路由）使用 `TaskListPresenter` 获取任务列表与 Activity 图数据：

```
loader({ request, params })
  → requireUserId(request)
  → findProjectBySlug() — 组织成员验证
  → findEnvironmentBySlug() — 环境验证
  → taskListPresenter.call({ organizationId, projectId, environmentId, environmentType })
  → typeddefer({ tasks, activity, runningStats, durations })
```

**关键**：`activity` / `runningStats` / `durations` 是 **Promise 不 await**（[L80 注释](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/presenters/v3/TaskListPresenter.server.ts#L80)），通过 `typeddefer` 实现流式加载——先返回 tasks 列表，指标数据延迟推送。

### 9.2 Presenter → Repository

**Presenter**：[TaskListPresenter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/presenters/v3/TaskListPresenter.server.ts)

```
TaskListPresenter.call()
  → findCurrentWorkerFromEnvironment() — 查询当前活跃 Worker
  → backgroundWorkerTask.findMany() — 获取任务列表（排除 AGENT 类型）
  → environmentMetricsRepository.getDailyTaskActivity({ ..., days: 6 })    — 7天 Activity
  → environmentMetricsRepository.getCurrentRunningStats({ ..., days: 6 })  — 队列/运行统计
  → environmentMetricsRepository.getAverageDurations({ ..., days: 6 })     — 平均耗时
```

**实例化**（单例模式）：
```typescript
const environmentMetricsRepository = new ClickHouseEnvironmentMetricsRepository({
  clickhouse: clickhouseClient,
});
```

### 9.3 Repository → ClickHouse 查询

**Repository**：[environmentMetricsRepository.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/services/environmentMetricsRepository.server.ts)

三个查询均直接读 ClickHouse `task_runs_v2` 表（非 task_events 表）：

#### Activity 图查询

**ClickHouse SQL**（[taskRuns.ts L424-L453](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/internal-packages/clickhouse/src/taskRuns.ts#L424-L453)）：

```sql
SELECT
    task_identifier,
    status,
    toDate(created_at) as day,
    count() as count
FROM trigger_dev.task_runs_v2 FINAL
WHERE
    organization_id = {organizationId:String}
    AND project_id = {projectId:String}
    AND environment_id = {environmentId:String}
    AND created_at >= today() - {days:Int64}
    AND _is_deleted = 0
GROUP BY task_identifier, status, day
ORDER BY task_identifier ASC, day ASC, status ASC
```

返回后由 `fillInDailyTaskActivity()` 填充缺失日期为 0，按 `TaskRunStatus` 分组。

#### 运行统计查询

**ClickHouse SQL**（[taskRuns.ts L470-L496](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/internal-packages/clickhouse/src/taskRuns.ts#L470-L496)）：

```sql
SELECT task_identifier, status, count() as count
FROM trigger_dev.task_runs_v2 FINAL
WHERE
    organization_id = ... AND project_id = ... AND environment_id = ...
    AND status IN ('PENDING','WAITING_FOR_DEPLOY','WAITING_TO_RESUME','QUEUED','EXECUTING','DELAYED')
    AND _is_deleted = 0
    AND created_at >= now() - INTERVAL {days:Int64} DAY
GROUP BY task_identifier, status
```

前端显示为每个 task 的 queued / running 数量。

#### 平均耗时查询

**ClickHouse SQL**（[taskRuns.ts L512-L536](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/internal-packages/clickhouse/src/taskRuns.ts#L512-L536)）：

```sql
SELECT task_identifier,
       avg(toUnixTimestamp(completed_at) - toUnixTimestamp(started_at)) as duration
FROM trigger_dev.task_runs_v2 FINAL
WHERE
    organization_id = ... AND project_id = ... AND environment_id = ...
    AND created_at >= today() - {days:Int64}
    AND status IN ('COMPLETED_SUCCESSFULLY','COMPLETED_WITH_ERRORS')
    AND started_at IS NOT NULL AND completed_at IS NOT NULL
    AND _is_deleted = 0
GROUP BY task_identifier
```

### 9.4 鉴权与隔离

| 层级 | 机制 | 代码位置 |
|------|------|----------|
| 路由入口 | `requireUserId` + `findProjectBySlug(organizationSlug, projectParam, userId)` | [route.tsx L110-L127](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam._index/route.tsx#L110-L127) |
| 数据隔离 | 查询强制带 `organizationId` + `projectId` + `environmentId` | Repository 层三字段 WHERE |
| 软删除过滤 | `AND _is_deleted = 0` | ClickHouse SQL 内 |
| 数据源 | 仅 ClickHouse（`task_runs_v2 FINAL`） | 无 PostgreSQL fallback |

---

## 10. 日志页访问开关与 Debug 日志可见性

### 10.1 日志页面访问控制

**路由**：[logs/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.logs/route.tsx#L63-L95)

日志页面有 **独立的访问开关**，不走常规 RBAC，而是检查组织级 Feature Flag：

```typescript
async function hasLogsPageAccess(
  userId: string,
  isAdmin: boolean,
  isImpersonating: boolean,
  organizationSlug: string
): Promise<boolean> {
  if (isAdmin || isImpersonating) {
    return true; // 管理员/冒充者直接放行
  }

  const organization = await prisma.organization.findFirst({
    where: { slug: organizationSlug, members: { some: { userId } } },
    select: { featureFlags: true },
  });

  const flags = organization?.featureFlags as Record<string, unknown>;
  return validateFeatureFlagValue(FEATURE_FLAG.hasLogsPageAccess, flags?.hasLogsPageAccess)
    .success && result.data === true;
}
```

**Feature Flag 定义**：[featureFlags.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/featureFlags.ts#L7)

| 访问者类型 | 是否可访问日志页 | 条件 |
|------------|-----------------|------|
| 平台 Admin（`user.admin=true`） | ✅ | 直接放行 |
| 冒充会话（Impersonation） | ✅ | 直接放行 |
| 普通组织成员 | ⚠️ | 需要 `hasLogsPageAccess` Feature Flag = true |
| 非成员 | ❌ | 组织查询不命中 |

**无权限时**：直接重定向到首页 (`redirect("/")`)。

### 10.2 日志页保留期限制

Loader 中读取用户订阅计划确定保留期：

```typescript
const plan = await getCurrentPlan(project.organizationId);
const retentionLimitDays = plan?.v3Subscription?.plan?.limits.logRetentionDays.number ?? 30;
```

此值传入 `LogsListPresenter.call()` 用于裁剪查询时间范围。

### 10.3 Debug 日志可见性 — 三层控制

#### 层级 1：写入时全局开关

**环境变量**：`EVENT_REPOSITORY_DEBUG_LOGS_DISABLED`

在 [env.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/env.server.ts#L1487) 中定义。设为 `true` 时，`recordRunDebugLog()` 静默返回——debug 事件 **不被写入**。

#### 层级 2：Run 详情页 Debug 开关

**路由**：[runs.$runParam/route.tsx L248-L265](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.runs.$runParam/route.tsx#L248-L265)

```typescript
// 服务端 loader
const showDebug = url.searchParams.get("showDebug") === "true";
presenter.call({ userId, showDeletedLogs: !!impersonationId, showDebug, ... });
```

**UI 端**（[L714-L786](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.runs.$runParam/route.tsx#L714-L786)）：

```typescript
const isAdmin = useHasAdminAccess(); // user.admin || isImpersonating

{isAdmin && (
  <Switch
    variant="small"
    label="Debug"
    checked={showDebug}
    onCheckedChange={(checked) => {
      replace({ showDebug: checked ? "true" : "false" });
    }}
  />
)}
```

**效果**：
- 非 Admin 用户：**看不到 Debug 开关**，`showDebug` 默认为 `false`
- Admin / 冒充用户：可见并操作 Debug 开关

`showDebug=true` → `includeDebugLogs: true` → 查询不再过滤 `kind=LOG`（PG）或 `kind=DEBUG_EVENT`（CH）

#### 层级 3：查询层过滤

**PostgreSQL 路径**（[taskEventStore.server.ts L126-L132](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/taskEventStore.server.ts#L126-L132)）：

```typescript
const filterDebug = options?.includeDebugLogs === false || options?.includeDebugLogs === undefined;
// filterDebug = true → WHERE kind != 'LOG'
```

**ClickHouse 路径**（[clickhouseEventRepository.server.ts L1292-L1294](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/eventRepository/clickhouseEventRepository.server.ts#L1292-L1294)）：

```typescript
if (options?.includeDebugLogs === false) {
  queryBuilder.where("kind != {kind: String}", { kind: "DEBUG_EVENT" });
}
```

**注意**：`includeDebugLogs` 默认为 `undefined`，等价于 `false`——debug 日志 **默认不返回**。

### 10.4 已删除日志可见性

**Presenter**：[RunPresenter.server.ts L115-L146](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/presenters/v3/RunPresenter.server.ts#L115-L146)

```typescript
const showLogs = showDeletedLogs || !run.logsDeletedAt;
```

| 条件 | `showDeletedLogs` 来源 | `run.logsDeletedAt` | 结果 |
|------|----------------------|---------------------|------|
| 正常用户查看 | `false`（无 impersonationId） | `null`（未删除） | ✅ 显示日志 |
| 正常用户查看已删除 | `false` | 有值 | ❌ trace=undefined |
| Admin 冒充会话 | `true`（有 impersonationId） | 有值 | ✅ 显示已删除日志 |

`showDeletedLogs=true` 时还会将 `logsDeletedAt` 设为 `null` 返回前端（隐藏"日志已删除"提示）。

---

## 11. 运行详情 vs 导出 API 的鉴权差异

### 11.1 鉴权模型对比

| 维度 | Dashboard 运行详情 | 导出 API |
|------|-------------------|----------|
| **认证方式** | Session Cookie（`requireUserId`） | API Key / JWT / PAT（Bearer Token） |
| **路由构建器** | `dashboardBuilder` → `authenticateAndAuthorize` | `apiBuilder` → `authenticateRequestForApiBuilder` |
| **身份来源** | `userId` → Session | `environment` → API Key 绑定的环境 |
| **授权粒度** | 组织成员关系 + RBAC ability | JWT scope / PAT cap-and-floor ability |
| **资源定位** | `findFirstOrThrow({ userId, projectSlug })` | `findFirst({ friendlyId, runtimeEnvironmentId })` |
| **隔离层级** | 组织 + 项目 | 环境 |

### 11.2 Dashboard 运行详情鉴权链

```
浏览器请求
  → requireUserId(request) — Session 认证
  → getImpersonationId(request) — 检测冒充会话
  → findProjectBySlug(orgSlug, projectSlug, userId) — 组织成员验证
  → RunPresenter.call({ userId, ... }) — Run 查询（含组织成员关系 JOIN）
  → Run 数据返回
```

**权限结果**：
- ✅ 组织成员 → 可看该组织下所有项目的 Run
- ❌ 非成员 → `findFirstOrThrow` 抛 404
- ⚠️ Admin 冒充 → 额外看到 `showDeletedLogs` + Debug 开关

### 11.3 API 导出鉴权链

#### `/api/v1/runs/$runId/events`（事件列表）

**代码**：[api.v1.runs.$runId.events.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/routes/api.v1.runs.$runId.events.ts)

```
API 请求
  → createLoaderApiRoute({ allowJWT: true })
    → authenticateRequestForApiBuilder(request, { allowJWT })
      → rbac.authenticateBearer(request, { allowJWT }) — 插件认证
        → 返回 { authentication, ability }
    → findResource: ApiRetrieveRunPresenter.findRun(runId, auth.environment)
      → 按 environment.id 定位 Run
    → authorization: ability.can("read", anyResource([
        { type: "runs", id: run.friendlyId },
        { type: "tasks", id: run.taskIdentifier },
        ...run.runTags.map(tag => ({ type: "tags", id: tag })),
        run.batch ? { type: "batch", id: batch.friendlyId } : ...
      ]))
    → eventRepository.getRunEvents() — 无 includeDebugLogs（默认不过滤）
```

#### `/api/v1/runs/$runId/trace`（Trace 详情）

**代码**：[api.v1.runs.$runId.trace.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/routes/api.v1.runs.$runId.trace.ts)

```
同上认证流程
  → eventRepository.getTraceDetailedSummary() — 无 includeDebugLogs
```

### 11.4 关键鉴权差异

| 差异点 | Dashboard | API |
|--------|-----------|-----|
| **身份绑定** | User → Organization → Project | API Key → Environment |
| **可见范围** | 同组织所有环境 | 仅 API Key 所属环境 |
| **Debug 日志** | 非Admin默认不返回 | **始终返回**（未传 includeDebugLogs） |
| **已删除日志** | 非冒充用户不可见 | **始终可见**（无 logsDeletedAt 检查） |
| **JWT 细粒度** | 不适用 | 支持 `read:runs:run_xxx` 限定单 Run |
| **跨环境** | 同组织可跨环境切换 | 不可，API Key 绑定单一环境 |

### 11.5 日志下载鉴权

**代码**：[resources.runs.$runParam.logs.download.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/routes/resources.runs.$runParam.logs.download.ts)

```
Session 认证（requireUser）
  → prisma.taskRun.findFirst({
      where: {
        friendlyId: runParam,
        project: { organization: { members: { some: { userId: user.id } } } }
      }
    })
  → eventRepository.getRunEvents()
  → 遍历事件时:
    if (!user.admin && event.kind === TaskEventKind.LOG) {
      return; // 非 Admin 跳过 LOG 类型事件
    }
  → Gzip 流式输出
```

**角色差异**：

| 角色 | Debug/LOG 日志 | 其他事件 |
|------|---------------|----------|
| Admin（`user.admin=true`） | ✅ 包含 | ✅ 包含 |
| 普通组织成员 | ❌ 跳过 | ✅ 包含 |

---

## 12. 不同角色能看到哪些事件或日志 — 完整矩阵

### 12.1 角色定义

| 角色 | 判定方式 | 说明 |
|------|---------|------|
| **平台 Admin** | `user.admin === true` | 平台超级管理员 |
| **冒充会话** | `getImpersonationId(request)` 有值 | Admin 以他人身份操作 |
| **组织成员** | `OrganizationMember` 记录存在 | 通过 `findProjectBySlug` 验证 |
| **API 调用方** | API Key / JWT / PAT | 绑定到 RuntimeEnvironment |
| **非成员** | 无 OrganizationMember 记录 | 被拒之门外 |

### 12.2 各页面/接口的角色可见性矩阵

| 页面/接口 | 平台 Admin | 冒充会话 | 普通组织成员 | API 调用方 | 非成员 |
|-----------|-----------|----------|-------------|-----------|--------|
| **首页 Activity 图** | ✅ | ✅ | ✅ | ❌ 无入口 | ❌ |
| **Run 详情页** | ✅ | ✅ | ✅ | ❌ 无入口 | ❌ |
| ┣ Debug 开关 | ✅ 可见/可操作 | ✅ 可见/可操作 | ❌ 不可见 | — | — |
| ┣ Debug 日志内容 | ✅ (开关开时) | ✅ (开关开时) | ❌ 默认不返回 | — | — |
| ┣ 已删除日志 | ✅ (showDeletedLogs) | ✅ (showDeletedLogs) | ❌ trace=undefined | — | — |
| ┣ AdminDebugTooltip | ✅ | ✅ | ❌ 不渲染 | — | — |
| ┗ AdminDebugRun | ✅ | ✅ | ❌ 不渲染 | — | — |
| **Span 详情页** | ✅ | ✅ | ✅ | ❌ 无入口 | ❌ |
| **日志页面** | ✅ 直接放行 | ✅ 直接放行 | ⚠️ 需 Feature Flag | ❌ 无入口 | ❌ |
| **日志下载** | ✅ 含LOG | ✅ 含LOG | ⚠️ 需Flag 且不含LOG | ❌ 无入口 | ❌ |
| **API events** | — | — | — | ✅ 含全部Kind | ❌ 401 |
| **API trace** | — | — | — | ✅ 含全部Kind | ❌ 401 |
| **Impersonation审计** | ✅ (数据库直接查) | — | ❌ | ❌ | ❌ |

### 12.3 `useHasAdminAccess` 客户端判定

**代码**：[useUser.ts L33-L38](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/hooks/useUser.ts#L33-L38)

```typescript
export function useHasAdminAccess(matches?: UIMatch[]): boolean {
  const user = useOptionalUser(matches);
  const isImpersonating = useIsImpersonating(matches);
  return Boolean(user?.admin) || isImpersonating;
}
```

此 hook 控制以下 UI 元素的可见性：
- **Debug 开关**（[L774](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.runs.$runParam/route.tsx#L774)）：`isAdmin && <Switch label="Debug">`
- **AdminDebugTooltip**（[debugTooltip.tsx L18](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/components/admin/debugTooltip.tsx#L18)）：`!hasAdminAccess && !isImpersonating → return null`
- **AdminDebugRun**（[debugRun.tsx L18](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/components/admin/debugRun.tsx#L18)）：`!hasAdminAccess && !isImpersonating → return null`

### 12.4 RBAC Ability 体系

**Ability 构建**（[ability.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/internal-packages/rbac/src/ability.ts)）：

| Ability 类型 | `can()` | `canSuper()` | 适用场景 |
|-------------|---------|--------------|----------|
| `permissiveAbility` | 始终 true | false | 普通认证用户（OSS fallback） |
| `superAbility` | 始终 true | true | 平台 Admin |
| `denyAbility` | 始终 false | false | 废弃的 PUBLIC token / 未认证 |
| `buildJwtAbility(scopes)` | 按 scope 匹配 | false | JWT 限定授权 |

**JWT scope 匹配规则**：
- `admin` → 全通配（`canSuper` 仍为 false）
- `read:all` → 读任意资源
- `read:runs` → 读所有 runs
- `read:runs:run_abc` → 仅读 `run_abc`
- `*:all` → 任意操作任意资源

**Dashboard 路由**的 `isAuthorized` 检查（[dashboardBuilder.server.ts L27-L32](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/services/routeBuilders/dashboardBuilder.server.ts#L27-L32)）：

```typescript
function isAuthorized(ability: RbacAbility, authorization: AuthorizationOption): boolean {
  if ("requireSuper" in authorization) {
    return ability.canSuper(); // 仅 Admin 可通过
  }
  return ability.can(authorization.action, authorization.resource);
}
```

### 12.5 API 路由的授权检查

API 路由通过 `anyResource()` / `everyResource()` 表达多资源授权语义：

- **`anyResource([...])`**：任一资源通过即授权（如 Run 可通过 friendlyId / task / tag / batch 任一匹配）
- **`everyResource([...])`**：所有资源都必须通过（如批量操作的每个元素）

**events/trace API 授权声明**：

```typescript
authorization: {
  action: "read",
  resource: (run) => anyResource([
    { type: "runs", id: run.friendlyId },
    { type: "tasks", id: run.taskIdentifier },
    ...run.runTags.map(tag => ({ type: "tags", id: tag })),
    run.batch ? { type: "batch", id: batch.friendlyId } : undefined,
  ]),
}
```

这意味着 JWT scope 为 `read:runs:run_abc`、`read:tasks:my-task`、`read:tags:my-tag` 或 `read:all` 中的任一个均可通过授权。

---

## 13. 补充代码文件索引

| 文件 | 职责 |
|------|------|
| [TaskListPresenter.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/presenters/v3/TaskListPresenter.server.ts) | 首页 Activity 图 Presenter |
| [environmentMetricsRepository.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/services/environmentMetricsRepository.server.ts) | 环境指标 Repository（ClickHouse） |
| [taskRuns.ts L424-L536](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/internal-packages/clickhouse/src/taskRuns.ts#L424-L536) | ClickHouse Activity/Stats/Duration SQL |
| [logs/route.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.logs/route.tsx) | 日志页面路由（含访问开关） |
| [featureFlags.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/v3/featureFlags.ts) | Feature Flag 定义（含 hasLogsPageAccess） |
| [useUser.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/hooks/useUser.ts) | useHasAdminAccess 客户端 hook |
| [debugTooltip.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/components/admin/debugTooltip.tsx) | Admin 调试信息 Tooltip |
| [debugRun.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/components/admin/debugRun.tsx) | Admin Run 调试按钮 |
| [apiBuilder.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/services/routeBuilders/apiBuilder.server.ts) | API 路由构建器（含 anyResource/everyResource） |
| [ability.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/internal-packages/rbac/src/ability.ts) | RBAC Ability 构建（permissive/super/deny/JWT） |
| [impersonation.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/services/impersonation.server.ts) | 冒充会话 ID 管理 |
| [api.v1.runs.$runId.events.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/routes/api.v1.runs.$runId.events.ts) | API 事件列表端点 |
| [api.v1.runs.$runId.trace.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/193-trigger.dev/apps/webapp/app/routes/api.v1.runs.$runId.trace.ts) | API Trace 详情端点 |
