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

RunQueue 采用**两层队列架构**：
1. **Master Queue**：按环境 + 队列维度的公平调度（Fair Queue Selection Strategy）
2. **Worker Queue**：具体工作节点的消费队列

### 3.2 公平调度策略

`FairQueueSelectionStrategy` 实现了加权轮询算法：
- 支持环境级并发限制（`defaultEnvConcurrency`）
- 支持队列级优先级和权重配置
- 支持突发因子（`concurrencyBurstFactor`）
- 避免单个环境/队列饿死

### 3.3 出队详细流程

```
1. Worker 发起拉取请求
   └─ dequeueFromWorkerQueue(consumerId, workerQueue)

2. 队列拉取
   ├─ Redis XREAD 阻塞式拉取（支持 BLOOM 过滤器）
   └─ 多消费者竞争，确保每条消息只被消费一次

3. 运行时检查
   ├─ 分布式锁 (RunLocker)
   ├─ 快照状态验证（必须是 QUEUED / QUEUED_EXECUTING）
   ├─ Worker/Task 存在性检查
   ├─ Deployment 有效性检查
   └─ 环境归档检查

4. 任务锁定
   ├─ 更新 TaskRun.status = DEQUEUED
   ├─ 设置 lockedAt / lockedToVersionId / lockedQueueId
   ├─ 创建 PENDING_EXECUTING 快照
   └─ 发布 runLocked 事件

5. 结果返回
   └─ DequeuedMessage 包含执行所需的全部上下文
```

### 3.4 关键代码位置
- 出队系统：`internal-packages/run-engine/src/engine/systems/dequeueSystem.ts`
- 公平调度：`internal-packages/run-engine/src/run-queue/fairQueueSelectionStrategy.ts`

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
