# 任务暂停/恢复机制概念说明

## 1. 核心概念

### 1.1 Waitpoint（等待点）

Waitpoint 是任务暂停与恢复的核心抽象，代表一个需要等待的条件。任务在执行过程中遇到 Waitpoint 时会暂停，直到该 Waitpoint 被标记为完成。

**四种类型的 Waitpoint：**

| 类型 | 触发方式 | 完成时机 | 典型场景 |
|------|----------|----------|----------|
| `DATETIME` | 时间驱动 | 到达指定时间自动完成 | `wait.for({ minutes: 30 })` |
| `MANUAL` | 外部信号 | 通过 API 调用 `completeToken` 完成 | 人工审批、Webhook 回调 |
| `RUN` | 子任务驱动 | 关联的子任务完成时自动完成 | `triggerAndWait()` |
| `BATCH` | 批量驱动 | 批量中所有子任务完成时自动完成 | `batchTriggerAndWait()` |

### 1.2 Execution Snapshot（执行快照）

Execution Snapshot 是任务运行状态的不可变记录，每次状态变更都会创建新的快照。它记录了：
- `executionStatus` - 执行状态（运行状态机的核心）
- `runStatus` - 任务运行状态
- `checkpointId` - 关联的检查点 ID（如果已挂起）
- `completedWaitpoints` - 已完成的等待点列表

**关键执行状态：**

```
RUN_CREATED → QUEUED → EXECUTING → EXECUTING_WITH_WAITPOINTS → SUSPENDED
                                                          ↓
                                                     （等待点完成）
                                                          ↓
EXECUTING ← QUEUED ← SUSPENDED
```

## 2. 核心组件与边界

### 2.1 组件架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Task Code (SDK)                                │
│  ┌────────────┐     ┌────────────┐     ┌──────────────────────────────┐    │
│  │ wait.for() │────▶│ runtime    │────▶│ SharedRuntimeManager         │    │
│  │ wait.until()│    │ API        │     │  - 等待点解析器注册          │    │
│  │ wait.forToken() │            │     │  - 暂停/恢复生命周期钩子      │    │
│  └────────────┘     └────────────┘     └──────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Worker / Supervisor                               │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ Socket.IO Client                                                    │   │
│  │  - 订阅 run:notify 事件                                             │   │
│  │  - 收到通知后调用 getRunExecutionData()                             │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Platform (Run Engine)                             │
│                                                                             │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐  │
│  │ WaitpointSystem  │  │ ExecutionSnapshot │  │ EnqueueSystem           │  │
│  │  - createWaitpoint│  │ System           │  │  - 入队调度             │  │
│  │  - completeWaitpoint │  - 创建快照     │  │  - 优先级排序           │  │
│  │  - blockRunWithWaitpoint │  - 状态追溯 │  │  - 并发控制             │  │
│  │  - continueRunIfUnblocked │           │  │                          │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────────────┘  │
│         │                       │                       │                   │
│         ▼                       ▼                       ▼                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐  │
│  │ CheckpointSystem │  │ EventBus         │  │ RunQueue                 │  │
│  │  - 创建检查点     │  │  - workerNotification │  - 公平队列调度       │  │
│  │  - 恢复执行       │  │  - runStatusChanged  │  - 并发槽管理         │  │
│  │  - 状态校验       │  │  - runSucceeded/Failed │  - TTL 过期处理     │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────────────┘  │
│         │                       │                                           │
│         ▼                       ▼                                           │
│  ┌──────────────────┐  ┌──────────────────┐                                │
│  │ Database         │  │ Socket.IO Server │                                │
│  │  - Waitpoint 表  │  │  - run:notify 事件广播                           │
│  │  - TaskRunWaitpoint 关联表 │                                           │
│  │  - ExecutionSnapshot 表 │                                               │
│  └──────────────────┘  └──────────────────┘                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 3. 暂停流程详解

### 3.1 暂停触发（SDK 层）

当任务代码调用 `wait.*` API 时：

```typescript
// 示例：等待人工审批
const token = await wait.createToken({ timeout: "24h" });
// ... 将 token.url 发送给审批系统 ...
const result = await wait.forToken(token);
```

**关键步骤：**
1. `runtime.waitUntil(waitpointId)` 被调用
2. `SharedRuntimeManager.waitForWaitpoint()` 创建 Promise 并注册解析器
3. 调用 `setSuspendable(true)` 通知 Worker 可以安全挂起
4. Promise 进入等待状态，任务代码暂停

### 3.2 状态转换（Engine 层）

Worker 检测到可挂起状态后，通知 Platform 创建检查点：

```
EXECUTING → EXECUTING_WITH_WAITPOINTS → SUSPENDED
```

**关键边界：**
- `CheckpointSystem.createCheckpoint()` 验证快照有效性（必须是最新快照）
- 只有 `isCheckpointable()` 状态允许创建检查点
- 检查点创建后，释放并发槽（`runQueue.releaseAllConcurrency()`）
- 创建 `SUSPENDED` 状态的执行快照

### 3.3 数据持久化

暂停时写入的数据：

1. **Waitpoint 表**：记录等待点状态为 `PENDING`
2. **TaskRunWaitpoint 关联表**：建立 TaskRun 与 Waitpoint 的多对多阻塞关系
3. **ExecutionSnapshot 表**：记录 `SUSPENDED` 状态和 `checkpointId`
4. **TaskRunCheckpoint 表**：存储检查点位置和镜像引用

## 4. 恢复流程详解

### 4.1 恢复触发

恢复可以由多种方式触发：

| 触发源 | 触发方式 | 目标 Waitpoint 类型 |
|--------|----------|---------------------|
| 定时调度 | `finishWaitpoint` 作业（Redis Worker） | `DATETIME` |
| 外部 API | `completeToken()` API 调用 | `MANUAL` |
| 子任务完成 | 子任务完成时自动完成关联 Waitpoint | `RUN` |
| 批量完成 | 批量中所有子任务完成 | `BATCH` |

### 4.2 恢复核心流程

以 `completeWaitpoint()` 为例，恢复流程如下：

```
┌───────────────────────────────────────────────────────────────────┐
│ 1. completeWaitpoint(id, output)                                   │
│    - 更新 Waitpoint.status = COMPLETED                             │
│    - 写入 output/outputType/outputIsError                          │
│    - 查找所有被此 Waitpoint 阻塞的 TaskRun                         │
└───────────────────────────────────┬───────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────────┐
│ 2. 为每个阻塞的 TaskRun 调度 continueRunIfUnblocked 作业           │
│    - 去重：jobId = continueRunIfUnblocked:${runId}                │
│    - 延迟 50ms 执行（允许批量等待点同时完成）                       │
└───────────────────────────────────┬───────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────────┐
│ 3. continueRunIfUnblocked(runId)                                  │
│    ├─ 获取所有阻塞的 Waitpoint                                     │
│    ├─ 检查是否所有 Waitpoint 都已 COMPLETED                        │
│    ├─ 如有未完成的，返回 blocked 状态                              │
│    └─ 全部完成时，根据当前快照状态决定恢复策略                      │
└───────────────────────────────────┬───────────────────────────────┘
                                    │
            ┌───────────────────────┴───────────────────────┐
            │                                               │
            ▼                                               ▼
┌──────────────────────────────┐            ┌──────────────────────────────┐
│ 快照状态: EXECUTING_WITH_WAITPOINTS │      │ 快照状态: SUSPENDED         │
│ （任务仍在运行中，只是遇到等待点） │      │ （任务已完全挂起）           │
│                              │            │                              │
│ 直接创建 EXECUTING 快照      │            │ 通过 enqueueSystem 重新入队  │
│ 发送 workerNotification      │            │ 创建 QUEUED 快照             │
│ Worker 继续执行              │            │ 等待 RunQueue 调度           │
└──────────────────────────────┘            └───────────────┬──────────────┘
                                                             │
                                                             ▼
┌───────────────────────────────────────────────────────────────────┐
│ 4. workerNotification 事件                                        │
│    ├─ EventBus 广播 workerNotification 事件                        │
│    ├─ Socket.IO 向对应 run 的 room 发送 run:notify 消息           │
│    └─ Worker/DevWorker 命名空间同时广播                            │
└───────────────────────────────────┬───────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────────┐
│ 5. Worker 端恢复                                                  │
│    ├─ 收到 run:notify 事件                                        │
│    ├─ 调用 getRunExecutionData() 获取最新快照和完成的 Waitpoint    │
│    ├─ SharedRuntimeManager.resolveWaitpoints() 解析等待点          │
│    ├─ 匹配 resolver，resolve 等待的 Promise                        │
│    └─ 任务代码从暂停处继续执行                                     │
└───────────────────────────────────────────────────────────────────┘
```

### 4.3 关键边界与设计决策

#### 边界 1：并发安全的等待点完成检查

**问题**：如果在 `blockRunWithWaitpoint` 执行期间，另一个线程同时完成了 Waitpoint，会出现竞态条件。

**解决方案**（`waitpointSystem.ts:386-526`）：

```typescript
async blockRunWithWaitpoint(...) {
  // 步骤 1: 插入 TaskRunWaitpoint 关联（CTE 原子操作）
  await prisma.$queryRaw`WITH inserted AS (...) ...`;

  // 步骤 2: 单独的 SELECT 检查 PENDING 状态
  // 关键：使用独立快照，确保能看到并发提交的 COMPLETED 状态
  const pendingCheck = await prisma.$queryRaw`
    SELECT COUNT(*) as pending_count
    FROM "Waitpoint"
    WHERE id IN (...) AND status = 'PENDING'
  `;
}
```

**设计考量**：PostgreSQL READ COMMITTED 隔离级别下，每条语句有独立快照。如果合并为单条 CTE，并发提交的 COMPLETED 状态对 CTE 不可见。

#### 边界 2：去重调度与防抖

**问题**：批量任务可能在短时间内有大量 Waitpoint 同时完成，导致 `continueRunIfUnblocked` 被重复调度。

**解决方案**：

```typescript
// 使用 jobId 去重，同一 runId 只会执行一次
await this.$.worker.enqueue({
  id: `continueRunIfUnblocked:${runId}`,  // 唯一键，自动去重
  job: "continueRunIfUnblocked",
  payload: { runId: run.taskRunId },
  availableAt: new Date(Date.now() + 50),  // 50ms 延迟，允许批量完成
});
```

#### 边界 3：运行中与已挂起的恢复差异

**两种暂停状态的恢复策略：**

| 快照状态 | 恢复策略 | 适用场景 |
|---------|----------|----------|
| `EXECUTING_WITH_WAITPOINTS` | 直接通知 Worker 继续，无需重新入队 | 任务仍在 Worker 上运行，只是遇到多个等待点中的一个 |
| `SUSPENDED` | 通过 EnqueueSystem 重新入队，等待调度 | 任务已完全挂起，需要重新分配执行资源 |

**关键判断**（`waitpointSystem.ts:795-887`）：

```typescript
case "EXECUTING_WITH_WAITPOINTS": {
  // 任务仍在执行中，直接发送通知让 Worker 继续
  await sendNotificationToWorker({ runId, snapshot: newSnapshot, ... });
  break;
}
case "SUSPENDED": {
  // 任务已挂起，需要重新入队
  const newSnapshot = await this.enqueueSystem.enqueueRun({
    run,
    env: run.runtimeEnvironment,
    snapshot: { status: "QUEUED", ... },
    checkpointId: snapshot.checkpointId,
  });
  break;
}
```

#### 边界 4：等待点解析的时序一致性

**问题**：恢复通知可能在 Worker 注册等待点解析器之前到达。

**解决方案**（`sharedRuntimeManager.ts:29-30, 245-264`）：

```typescript
// 双 Map 设计：
private readonly resolversById = new Map<ResolverId, Resolver>();
private readonly waitpointsByResolverId = new Map<ResolverId, CompletedWaitpoint>();

// 解析时先检查是否有待处理的等待点
private resolvePendingWaitpoints(): void {
  for (const [resolverId, waitpoint] of this.waitpointsByResolverId.entries()) {
    this.resolveWaitpoint(waitpoint, resolverId);
  }
}
```

**恢复时序**：
1. 恢复通知到达，`resolveWaitpoint()` 被调用
2. 如果 resolver 不存在，将 waitpoint 存入 `waitpointsByResolverId`
3. 稍后任务代码调用 `wait.forToken()`，注册 resolver
4. 调用 `resolvePendingWaitpoints()` 检查并解析已完成的等待点

## 5. 失败场景与容错

### 5.1 幂等性保证

- **Waitpoint 创建**：使用 `idempotencyKey` 确保重复调用返回同一 Waitpoint
- **completeWaitpoint**：`updateMany` 只更新 `status = 'PENDING'` 的记录，重复调用安全
- **continueRunIfUnblocked**：Redis Worker 的 jobId 去重机制防止重复执行

### 5.2 超时处理

- 创建 Waitpoint 时可指定 `timeout`，超时会自动以错误状态完成
- 超时任务通过 `finishWaitpoint` 作业调度执行
- 超时错误通过 `WaitpointTimeoutError` 传递给任务代码

### 5.3 检查点丢失

```typescript
// continueRunIfUnblocked 中的安全检查
if (!snapshot.checkpointId) {
  if (snapshot.runStatus === "CANCELED") {
    // 任务在挂起过程中被取消，正常跳过
    return { status: "skipped", reason: "run was canceled while suspended" };
  }
  // 检查点丢失，抛出错误
  throw new Error(`run is suspended, but has no checkpoint: ${runId}`);
}
```

## 6. 关键代码位置参考

| 组件 | 文件路径 | 核心函数 |
|------|----------|----------|
| Waitpoint 系统 | `internal-packages/run-engine/src/engine/systems/waitpointSystem.ts` | `completeWaitpoint`, `blockRunWithWaitpoint`, `continueRunIfUnblocked` |
| 执行快照系统 | `internal-packages/run-engine/src/engine/systems/executionSnapshotSystem.ts` | `createExecutionSnapshot`, `getLatestExecutionSnapshot` |
| 检查点系统 | `internal-packages/run-engine/src/engine/systems/checkpointSystem.ts` | `createCheckpoint`, `continueRunExecution` |
| 入队系统 | `internal-packages/run-engine/src/engine/systems/enqueueSystem.ts` | `enqueueRun` |
| 运行时管理器 | `packages/core/src/v3/runtime/sharedRuntimeManager.ts` | `waitForWaitpoint`, `resolveWaitpoints` |
| 事件总线 | `internal-packages/run-engine/src/engine/eventBus.ts` | `sendNotificationToWorker` |
| SDK API | `packages/trigger-sdk/src/v3/wait.ts` | `wait.for`, `wait.forToken`, `wait.createToken` |
