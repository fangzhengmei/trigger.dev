# 任务生命周期与事件流透出机制 - 概念级文档

## 1. 系统架构总览

Trigger.dev 的任务处理系统采用分布式、事件驱动的架构设计，核心由以下模块组成：

### 1.1 核心模块

| 模块 | 职责 | 所在位置 |
|------|------|----------|
| **RunEngine** | 任务生命周期的核心编排器，协调各个子系统 | `internal-packages/run-engine/src/engine/index.ts` |
| **RunQueue** | 分布式队列管理，基于 Redis 实现公平调度 | `internal-packages/run-engine/src/run-queue/index.ts` |
| **ExecutionSnapshotSystem** | 执行快照管理，记录任务状态变迁 | `internal-packages/run-engine/src/engine/systems/executionSnapshotSystem.ts` |
| **RunAttemptSystem** | 任务尝试管理，负责启动、完成、重试 | `internal-packages/run-engine/src/engine/systems/runAttemptSystem.ts` |
| **EventBus** | 进程内事件总线，广播生命周期事件 | `internal-packages/run-engine/src/engine/eventBus.ts` |
| **Socket.IO Server** | 实时通信层，向客户端推送事件 | `apps/webapp/app/v3/handleSocketIo.server.ts` |
| **RedisRealtimeStreams** | 基于 Redis Stream 的实时数据流 | `apps/webapp/app/services/realtime/redisRealtimeStreams.server.ts` |

### 1.2 状态机定义

任务执行状态（`TaskRunExecutionStatus`）：
```
RUN_CREATED → QUEUED → PENDING_EXECUTING → EXECUTING → [FINAL_STATE]
                    ↓
                QUEUED_EXECUTING (继续执行中)
                    ↓
            EXECUTING_WITH_WAITPOINTS (等待点执行中)
                    ↓
                SUSPENDED (挂起)
```

任务最终状态（`TaskRunStatus`）：
```
COMPLETED_SUCCESSFULLY / COMPLETED_WITH_ERRORS
CANCELED / INTERRUPTED / CRASHED
SYSTEM_FAILURE / EXPIRED / TIMED_OUT
```

---

## 2. 任务入队流程 (Enqueue)

### 2.1 触发入口

任务通过 `RunEngine.trigger()` 方法触发，支持多种触发方式：
- API 调用（`triggerTask.server.ts`）
- 批量触发（`batchTriggerV3.server.ts`）
- 定时调度（`schedule-engine`）
- 重放（`replayTaskRun.server.ts`）

### 2.2 入队详细流程

```
1. 前置处理
   ├─ 防抖处理 (DebounceSystem)
   │  ├─ Leading 模式：首次触发立即执行，后续防抖
   │  └─ Trailing 模式：延迟执行，新触发重置计时器
   ├─ 幂等性检查 (Idempotency Key)
   └─ 延迟任务处理 (DelayedRunSystem)

2. 数据库写入
   ├─ 创建 TaskRun 记录
   ├─ 创建初始 ExecutionSnapshot (RUN_CREATED / DELAYED)
   └─ 关联 Waitpoint（如果是 triggerAndWait 模式）

3. 入队操作 (EnqueueSystem.enqueueRun)
   ├─ 分布式锁保护 (RunLocker)
   ├─ 创建 QUEUED 状态快照
   ├─ 计算队列优先级 (priorityMs)
   └─ 调用 RunQueue.enqueueMessage()

4. 队列写入
   └─ Redis ZADD 写入有序集合
      ├─ Score = queueTimestamp - priorityMs
      └─ 支持 TTL 过期自动清理
```

### 2.3 关键代码位置
- 触发入口：`internal-packages/run-engine/src/engine/index.ts:447` (`trigger` 方法)
- 入队系统：`internal-packages/run-engine/src/engine/systems/enqueueSystem.ts`

---

## 3. 任务排程流程 (Dequeue & Schedule)

### 3.1 队列架构

RunQueue 采用**严格的两层队列架构**，数据结构和职责完全分离：

| 层级 | 数据结构 | 职责 | Key 示例 |
|------|----------|------|----------|
| **Master Queue** | Redis ZSET（有序集合） | 全局调度、公平排序、并发控制 | `rq:master:shard:0` |
| **Message Queue** | Redis ZSET（有序集合） | 按环境+队列维度的待执行任务池 | `rq:queue:{orgId}:{projId}:{envId}:{queueName}` |
| **Worker Queue** | Redis LIST（列表） | 单个 Worker 的消费队列 | `rq:worker:queue:{workerQueueId}` |

> **关键澄清**：不存在 XREAD + BLOOM 过滤机制。Master Queue 和 Message Queue 使用 ZSET + Lua 脚本原子操作，Worker Queue 使用 LIST + BLPOP 阻塞弹出。

### 3.2 入队写入路径回顾

入队时有两条路径（由 Lua 脚本原子判定）：

```
Fast Path（直接入 Worker Queue）
  条件：enableFastPath=true 且 并发可用
  操作：直接 RPUSH 到 Worker Queue

Slow Path（标准流程）
  操作：ZADD 到 Message Queue
        ZADD 到 Master Queue（标记该队列有消息）
        调度 processQueueForWorkerQueue Job（500ms 去抖）
```

### 3.3 公平调度策略

`FairQueueSelectionStrategy` 实现加权轮询算法，运行在 **Master Queue Consumer** 中：

```
Master Queue Consumer（定时 500ms 轮询）
    │
    ├─ ZRANGE 从 Master Queue 获取候选队列列表
    │
    ├─ 按环境分组，应用公平调度权重
    │  ├─ 环境级并发限制（defaultEnvConcurrency）
    │  ├─ 突发因子（concurrencyBurstFactor）
    │  └─ 队列优先级排序
    │
    └─ 对选中的队列调用 dequeueMessagesFromQueue
```

**调度特性**：
- 支持 Cooloff 机制：连续空队列自动冷却（默认 10s）
- 支持 CK（Concurrency Key）队列的通配符匹配
- 避免单个环境/队列饿死

### 3.4 出队完整流程（两步架构）

#### 第一步：Master Queue → Worker Queue（后台调度）

由 `#processMasterQueueShard` 定时任务执行（默认 500ms 间隔）：

```
1. 公平调度选队
   ├─ 调用 distributeFairQueuesFromParentQueue()
   └─ 获得候选队列列表（按环境 + 权重排序）

2. 对每个队列执行 Lua 脚本出队
   └─ redis.dequeueMessagesFromQueue()
      ├─ ZRANGEBYSCORE 从 Message Queue 取消息（按时间戳）
      ├─ 原子检查并发限制（环境级 + 队列级）
      ├─ SADD 到 currentConcurrency 集合（占用并发槽）
      ├─ ZREM 从 Message Queue 移除
      └─ 返回消息列表（最多 10 条）

3. 推入 Worker Queue
   └─ RPUSH 到对应 Worker Queue（List）
      └─ 每个消息存的是 messageKey 引用，不是完整 payload
```

#### 第二步：Worker Queue → Worker（Worker 拉取）

由 Worker 主动调用 `dequeueMessageFromWorkerQueue()`：

```
1. Worker 发起阻塞拉取
   └─ BLPOP workerQueueKey timeout（默认 10s 阻塞超时）
      └─ 返回 messageKey（如 "rq:message:{orgId}:{runId}"）

2. 读取完整消息
   └─ GET messageKey 获取完整 payload

3. 运行时检查（dequeueSystem.ts）
   ├─ 分布式锁 (RunLocker)
   ├─ 快照状态验证（必须是 QUEUED / QUEUED_EXECUTING）
   ├─ Worker/Task 存在性检查
   ├─ Deployment 有效性检查
   └─ 环境归档检查

4. 任务锁定
   ├─ 更新 TaskRun.status = DEQUEUED
   ├─ 设置 lockedAt / lockedToVersionId / lockedQueueId
   ├─ SADD 到 currentDequeued 集合
   ├─ 创建 PENDING_EXECUTING 快照
   └─ 发布 runLocked 事件

5. 结果返回
   └─ DequeuedMessage 包含执行所需的全部上下文
```

### 3.5 并发控制机制（Lua 脚本原子性保证）

并发控制完全在 Lua 脚本中原子执行，避免竞态条件：

```
检查顺序：
1. 队列级并发限制：SCARD queueCurrentConcurrency < queueConcurrencyLimit
2. 环境级并发限制：SCARD envCurrentConcurrency < envConcurrencyLimit * burstFactor
3. 全部满足则：
   ├─ SADD queueCurrentConcurrency messageId
   ├─ SADD envCurrentConcurrency messageId
   └─ 返回成功
```

### 3.6 关键代码位置
- Master Queue 调度：`run-queue/index.ts:1598` (`#processMasterQueueShard`)
- 消息出队 Lua：`run-queue/index.ts:2024` (`#callDequeueMessagesFromQueue`)
- Worker 取任务：`run-queue/index.ts:773` (`dequeueMessageFromWorkerQueue`)
- 运行时检查：`engine/systems/dequeueSystem.ts`
- 公平调度：`run-queue/fairQueueSelectionStrategy.ts`

---

## 3.7 关于 XREAD + BLOOM 过滤的误判说明

### 错误描述回顾
之前的文档错误地将出队机制描述为 "Redis XREAD 阻塞式拉取（支持 BLOOM 过滤器）"，这与实际实现完全不符。

### 真实机制 vs 错误描述

| 维度 | 真实机制 | 错误描述 | 差异影响 |
|------|----------|----------|----------|
| **数据结构** | ZSET (Master/Message Queue) + LIST (Worker Queue) | Redis Stream + Bloom Filter | 排障时会查错 Redis 数据类型 |
| **消费模式** | 两步：后台调度推入 + Worker BLPOP 拉出 | 单步：XREAD 阻塞拉取 | 无法理解消息延迟来源（500ms 调度间隔） |
| **原子性保证** | Lua 脚本原子操作 | （假设）Stream 消费组 | 排查并发问题时方向错误 |
| **消息引用** | Worker Queue 存 messageKey 引用 | （假设）存完整消息 | 理解内存占用和大消息处理时出错 |

### 该误判可能造成的排障误导

1. **消息延迟排查**：
   - 错误方向：怀疑 XREAD 阻塞超时、Stream 消费者组 lag
   - 正确方向：检查 Master Queue Consumer 调度间隔（500ms）、Lua 脚本执行耗时、Cooloff 状态

2. **并发问题排查**：
   - 错误方向：怀疑 Bloom Filter 误判导致重复消费
   - 正确方向：检查 Lua 脚本中的 SADD/ZREM 原子性、currentConcurrency 集合状态

3. **性能瓶颈分析**：
   - 错误方向：怀疑 Stream 写入放大、Bloom Filter 内存占用
   - 正确方向：检查 ZSET 大小、Lua 脚本复杂度、Worker Queue LIST 长度

4. **数据恢复场景**：
   - 错误方向：尝试从 Stream 恢复消息
   - 正确方向：检查 Message Queue ZSET、Message Key 是否存在

---

## 4. 任务执行流程 (Execution)

### 4.1 执行架构

执行层采用 **Platform → Coordinator → Worker** 三层架构：

```
[Platform (webapp)]
       ↓ Socket.IO
[Coordinator] 任务分发与状态协调
       ↓ gRPC / HTTP
[Worker (supervisor / docker-provider / k8s-provider)]
       ↓
[实际执行容器 (Docker / K8s Pod / 本地进程)]
```

### 4.2 执行详细流程

```
1. 启动尝试 (startRunAttempt)
   ├─ 验证快照有效性
   ├─ 递增 attemptNumber
   ├─ 更新 TaskRun 执行信息
   ├─ 创建 EXECUTING 快照
   ├─ 发布 runAttemptStarted 事件
   └─ 向 Worker 发送 READY_FOR_EXECUTION

2. 执行阶段
   ├─ Worker 拉取执行 payload
   ├─ 启动执行容器（Docker / K8s / 子进程）
   ├─ 心跳上报 (heartbeatRun)
   │  └─ 超时检测：PENDING_EXECUTING (60s) / EXECUTING (60s)
   ├─ 进度流式推送 (Realtime Streams)
   ├─ 检查点创建 (createCheckpoint)
   └─ 等待点处理 (WaitpointSystem)

3. 完成尝试 (completeRunAttempt)
   ├─ 解析执行结果 (TaskRunExecutionResult)
   ├─ 判定重试逻辑 (retryOutcomeFromCompletion)
   │  ├─ 成功：进入完成流程
   │  ├─ 可重试：计算退避延迟，重新入队
   │  └─ 不可重试：标记最终失败
   ├─ 更新执行快照
   └─ 发布相应事件 (runSucceeded / runFailed / runRetryScheduled)
```

### 4.3 重试机制

`retryOutcomeFromCompletion` 支持：
- 最大尝试次数（`maxAttempts`）
- 指数退避（`factor`）
- 抖动（`randomize`）
- 最大延迟（`maxTimeoutInMs`）
- OOM 自动升级机器规格

### 4.4 关键代码位置
- 执行尝试系统：`internal-packages/run-engine/src/engine/systems/runAttemptSystem.ts`
- 心跳机制：`internal-packages/run-engine/src/engine/systems/executionSnapshotSystem.ts`

---

## 5. 进度推送与实时订阅链路

### 5.1 事件流分层架构

事件流采用**三层透出**架构：

```
┌─────────────────────────────────────────────────┐
│  层1: 进程内事件总线 (EventBus)                 │
│  - 内存级 EventEmitter                          │
│  - 16+ 种生命周期事件                           │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│  层2: 后端事件分发 (Socket.IO / Redis Streams)   │
│  - Socket.IO Room 广播                          │
│  - Redis Stream 持久化流                        │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│  层3: 前端订阅 (EventSource / WebSocket)        │
│  - useEventSource Hook                          │
│  - Socket.IO 客户端                             │
└─────────────────────────────────────────────────┘
```

### 5.2 层1：进程内事件总线 (EventBus)

**核心事件类型**（`EventBusEvents`）：

| 事件 | 触发时机 | 携带数据 |
|------|----------|----------|
| `runCreated` | 任务创建完成 | runId, time |
| `runEnqueuedAfterDelay` | 延迟任务入队 | run, org, project, env |
| `runLocked` | 任务被锁定准备执行 | 完整 run 信息 + 锁定信息 |
| `runStatusChanged` | 任务状态变更 | run, org, project, env |
| `runAttemptStarted` | 尝试启动 | run, attemptNumber |
| `runAttemptFailed` | 尝试失败 | run, error, attemptNumber |
| `runSucceeded` | 任务成功 | run, output, usage |
| `runFailed` | 任务失败 | run, error, usage |
| `runCancelled` | 任务取消 | run, error |
| `runExpired` | TTL 过期 | run, ttl |
| `runRetryScheduled` | 重试已排期 | run, retryAt |
| `runMetadataUpdated` | 元数据更新 | run, metadata |
| `executionSnapshotCreated` | 快照创建 | run, snapshot |
| `workerNotification` | Worker 通知 | run, snapshot |

**代码位置**：`internal-packages/run-engine/src/engine/eventBus.ts`

### 5.3 层2：后端事件分发

#### 5.3.1 Socket.IO 实时通道

**Room 机制**：
- 每个任务有独立的 Room：`room:${friendlyRunId}`
- Worker 通过 `run:subscribe` 加入房间
- 平台通过 `socketIo.workerNamespace.to(room).emit()` 广播

**关键事件**：
```typescript
// 平台 → Worker
"run:notify"          // 通知 Worker 拉取最新状态

// Worker → 平台  
"TASK_RUN_COMPLETED"  // 任务完成上报
"READY_FOR_EXECUTION" // 请求执行 payload
```

**代码位置**：`apps/webapp/app/v3/handleSocketIo.server.ts`

#### 5.3.2 Redis Stream 持久化流

用于日志、trace 等大数据量实时数据：
- Stream Key 格式：`stream:${runId}:${streamId}`
- 支持断点续传（`lastEventId`）
- 支持活跃超时自动关闭（默认 15s）
- 内置 Ping 保活机制（10s）

**代码位置**：`apps/webapp/app/services/realtime/redisRealtimeStreams.server.ts`

### 5.4 层3：前端订阅

#### 5.4.1 SSE (Server-Sent Events)

前端通过 `useEventSource` Hook 订阅：
```typescript
const data = useEventSource("/api/v1/runs/${runId}/events", {
  event: "message"
});
```

**后端实现**：`apps/webapp/app/utils/sse.server.ts`

#### 5.4.2 Socket.IO 客户端

前端实时接收任务状态变更：
- 自动重连机制
- 消息去重
- 离线消息补发

### 5.5 事件流透出时序图

```
任务执行中                     EventBus          Socket.IO          前端
    │                             │                   │                │
    ├─ 状态变更 (EXECUTING) ────►│                   │                │
    │                             ├─ executionSnapshotCreated          │
    │                             │                   │                │
    │                             ├─ workerNotification ────────────►│
    │                             │                   │                ├─ run:notify
    │                             │                   │                │
    ├─ 日志输出 ─────────────────────────────────────────────────────►│
    │                             │                   │                ├─ stream chunk
    │                             │                   │                │
    ├─ 完成 (SUCCESS) ──────────►│                   │                │
    │                             ├─ runSucceeded ───────────────────►│
    │                             │                   │                ├─ 状态更新
    │                             │                   │                │
```

---

## 6. 任务最终态归档流程

### 6.1 最终状态判定

任务进入最终态的触发点：
1. 执行成功 (`runSucceeded`)
2. 执行失败且无重试 (`runFailed`)
3. 用户取消 (`runCancelled`)
4. TTL 过期 (`runExpired`)
5. 超时 (`TIMED_OUT`)
6. 系统错误 (`SYSTEM_FAILURE`)

### 6.2 归档详细流程

```
1. FinalizeTaskRunService 调用
   ├─ 确认队列消息 (marqs.acknowledgeMessage)
   ├─ 更新 TaskRun 最终状态
   ├─ 记录错误信息（如果失败）
   └─ 更新元数据

2. 关联处理
   ├─ 批量任务处理 (completeBatchTaskRunItemV3)
   │  └─ 更新批次进度，触发批次完成检查
   ├─ 恢复依赖父任务 (ResumeDependentParentsService)
   │  └─ 如果所有子任务完成，恢复父任务执行
   ├─ 告警触发 (PerformTaskRunAlertsService)
   │  └─ 失败任务根据配置发送告警
   └─ TTL 清理 (ExpireEnqueuedRunService.ack)

3. 事件归档
   ├─ Event Repository 写入
   │  ├─ ClickHouse (大数据量分析)
   │  └─ PostgreSQL (实时查询)
   ├─ 使用量统计 (reportInvocationUsage)
   └─ 审计日志记录

4. 前端通知
   └─ Socket.IO 广播最终状态
      └─ 前端关闭 SSE 连接
```

### 6.3 数据保留策略

- **热数据**：PostgreSQL 存储，用于实时查询和展示
- **温数据**：ClickHouse 存储，用于历史分析和查询
- **冷数据**：可配置归档到对象存储（如 S3）

### 6.4 关键代码位置
- 任务终结服务：`apps/webapp/app/v3/services/finalizeTaskRun.server.ts`
- 事件存储：`apps/webapp/app/v3/eventRepository/index.server.ts`

---

## 7. 关键设计模式与特性

### 7.1 分布式锁 (RunLocker)
- 基于 Redis Redlock 算法
- 自动续期（`automaticExtensionThreshold`）
- 重试策略（指数退避 + 抖动）

### 7.2 快照机制 (ExecutionSnapshot)
- 不可变的状态变更记录
- 支持幂等恢复
- 完整的审计轨迹

### 7.3 等待点 (Waitpoint)
- 支持任务间依赖
- 支持批量等待
- 支持超时和手动完成

### 7.4 检查点 (Checkpoint)
- 支持任务执行中断恢复
- 基于执行快照的状态序列化
- 可配置重试 warm start

---

## 8. 核心链路代码索引

| 流程阶段 | 入口文件 | 核心方法 |
|----------|----------|----------|
| 任务触发 | `engine/index.ts` | `trigger()` |
| 入队 | `engine/systems/enqueueSystem.ts` | `enqueueRun()` |
| 出队 | `engine/systems/dequeueSystem.ts` | `dequeueFromWorkerQueue()` |
| 启动尝试 | `engine/systems/runAttemptSystem.ts` | `startRunAttempt()` |
| 完成尝试 | `engine/systems/runAttemptSystem.ts` | `completeRunAttempt()` |
| 心跳 | `engine/systems/executionSnapshotSystem.ts` | `heartbeatRun()` |
| 事件总线 | `engine/eventBus.ts` | `emit()` / `on()` |
| Socket.IO | `v3/handleSocketIo.server.ts` | 各类 Namespace 处理器 |
| 任务终结 | `v3/services/finalizeTaskRun.server.ts` | `call()` |

---

## 9. 扩展阅读

- `AGENTS.md` - 开发指南
- `internal-packages/run-engine/README.md` - Run Engine 设计文档
- `apps/webapp/test/` - 各模块测试用例
