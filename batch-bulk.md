# 批量触发 (Batch Trigger) 与 Bulk Action 协同机制详解

## 概述

Trigger.dev 中有两种核心的批量处理机制：

1. **Batch Trigger** - 批量触发任务（主动创建新任务）
2. **Bulk Action** - 对现有任务执行批量操作（取消、重播等）

两者共享相似的设计哲学：**批次拆分 + 并发控制 + 结果回收**，但在具体实现上有所不同。

---

## 一、Batch Trigger 核心架构

### 1.1 三个版本的演进

| 版本 | 标识 | 实现位置 | 特点 |
|------|------|----------|------|
| v1 | `batchVersion: "runengine:v1"` | `batchTrigger.server.ts` | 同步处理，小批量（≤20 个）直接处理，较大批量使用 worker 队列 |
| v2 | `batchVersion: "runengine:v2"` | `batch-queue/index.ts` | 基于 FairQueue + DRR 调度的分布式批次队列 |
| v3 | `batchVersion: "v3"` | `batchTriggerV3.server.ts` | 旧版 v3 API，兼容用 |

### 1.2 v1 版本实现（RunEngineBatchTriggerService）

**核心文件**: `apps/webapp/app/runEngine/services/batchTrigger.server.ts`

#### 批次拆分策略

```typescript
const PROCESSING_BATCH_SIZE = 50;      // 每个批次最多处理 50 个项
const ASYNC_BATCH_PROCESS_SIZE_THRESHOLD = 20;  // 超过 20 个使用异步处理
```

**处理流程**:

```
用户提交批量触发请求
    │
    ├─→ items ≤ 20: 同步内联处理 (#processBatchTaskRunItems)
    │     └─→ 全部完成直接返回
    │
    └─→ items > 20: 分批次异步处理
          ├─→ 创建 BatchTaskRun 记录
          ├─→ 按 50/批次拆分
          └─→ 入队 batchTriggerWorker 处理
```

#### 并发控制

```typescript
// batchTrigger.server.ts:68-72
// 注意：强制使用 sequential 策略，避免并行时对父运行锁的高竞争
this._batchProcessingStrategy = "sequential";
```

**设计考量**: 并行处理会导致大量任务同时尝试锁定父运行（parent run），造成数据库锁竞争。顺序处理确保每次只有一个任务锁定父运行。

#### 结果回收接力

1. **批次内处理**: `#processBatchTaskRunItems()` 循环处理当前批次的每个项
2. **进度追踪**: 每处理一项，`workingIndex++`，并将 runId 追加到 `batch.runIds`
3. **状态判断**:
   ```typescript
   if (updatedBatch.processingJobsCount >= updatedBatch.runCount) {
     // 全部运行已创建，尝试完成批次
     await this._engine.tryCompleteBatch({ batchId: batch.id });
   }
   ```
4. **接力处理**: 如果未完成，返回 `INCOMPLETE`，由 `processBatchTaskRun()` 重新入队下一批次

#### 容错机制

```typescript
// 单个项触发失败时，创建一个 pre-failed 运行
const failedRunId = await triggerFailedTaskService.call({...});
```

这样即使某个项触发失败，批次也能继续推进，不会卡住整个批次。

### 1.3 v2 版本实现（BatchQueue）

**核心文件**: `internal-packages/run-engine/src/batch-queue/index.ts`

这是最新的批次处理架构，基于 **FairQueue + DRR (Deficit Round Robin) 调度算法**。

#### 核心组件

```
BatchQueue
    ├─→ FairQueue: 基于 Redis 的公平队列，DRR 调度
    ├─→ WorkerQueueManager: 工作队列管理器
    ├─→ BatchCompletionTracker: 完成度追踪（Redis 原子计数）
    └─→ 消费者池: 多个并发消费循环
```

#### 两阶段提交（2-Phase API）

**Phase 1 - 初始化批次**:
```typescript
// createBatch.server.ts
await this._engine.initializeBatch({
  batchId: id,
  friendlyId,
  environmentId: environment.id,
  runCount: body.runCount,  // 预期总数
  processingConcurrency: config.processingConcurrency,
  // ...
});
```

**Phase 2 - 流式入队项**:
```typescript
// 每个项单独入队
await batchQueue.enqueueBatchItem(batchId, envId, itemIndex, item);
```

**优势**:
- 支持大规模批次（数万个项）
- 内存友好，无需一次性加载所有项
- 支持流式 NDJSON 上传

#### DRR 公平调度

```typescript
// 队列 ID 格式: env:{envId}:batch:{batchId}
const queueId = `env:${envId}:batch:${batchId}`;

// DRR 配置
const scheduler = new DRRScheduler({
  quantum: options.drr.quantum,           // 每轮分配的 credits
  maxDeficit: options.drr.maxDeficit,     // 最大累积赤字（防止饥饿）
  masterQueueLimit: options.drr.masterQueueLimit,
});
```

**工作原理**:
- 每个环境（tenant）获得公平的处理时间片
- 避免某个环境的大批次独占所有处理资源
- 通过 `quantum` 控制每轮处理的消息数

#### 并发控制层级

| 层级 | 实现 | 作用 |
|------|------|------|
| 全局速率限制 | `createBatchGlobalRateLimiter()` | 所有环境合计的 items/second 限制 |
| 环境级并发 | `getEnvConcurrency(envId)` | 单个环境同时处理的批次数 |
| 队列级并发 | `concurrencyGroups` in FairQueue | 单环境内并发处理的 item 数 |
| 消费者池 | `consumerCount` | 总消费者线程数 |

#### 结果回收与完成检测

**原子计数追踪**（Redis）:
```typescript
// 成功时原子递增计数
processedCount = await this.completionTracker.recordSuccess(batchId, runId, itemIndex);

// 失败时原子递增计数
processedCount = await this.completionTracker.recordFailure(batchId, failure);

// 完成检测: 原子计数 === 预期总数
if (processedCount === meta.runCount) {
  await this.#finalizeBatch(batchId, meta);
}
```

**完成回调**:
```typescript
if (this.completionCallback) {
  await this.completionCallback!(result);  // 通知 RunEngine 批次完成
}
```

---

## 二、Bulk Action 核心架构

**核心文件**: `apps/webapp/app/v3/services/bulk/BulkActionV2.server.ts`

Bulk Action 用于对**已存在**的任务执行批量操作，主要支持：
- `CANCEL`: 批量取消运行
- `REPLAY`: 批量重播运行

### 2.1 批次拆分策略

```typescript
// 从环境变量读取配置
const BULK_ACTION_BATCH_SIZE = env.BULK_ACTION_BATCH_SIZE;  // 每批处理数量
const BULK_ACTION_BATCH_DELAY_MS = env.BULK_ACTION_BATCH_DELAY_MS;  // 批次间延迟
const BULK_ACTION_SUBBATCH_CONCURRENCY = env.BULK_ACTION_SUBBATCH_CONCURRENCY;  // 批次内并发
```

**处理流程**:
```
用户创建 Bulk Action（筛选条件）
    │
    ↓
查询符合条件的运行总数 → 创建 BulkActionGroup
    │
    ↓
入队 processBulkAction job
    │
    ┌──────────────────────────────────┐
    │  process() 循环执行:              │
    │  1. 按 cursor 拉取一批 runIds     │
    │  2. pMap 并发处理（默认并发 8）   │
    │  3. 更新 successCount/failureCount │
    │  4. 还有更多 → 延迟后重新入队     │
    │  5. 全部完成 → 发通知邮件         │
    └──────────────────────────────────┘
```

### 2.2 并发控制

使用 `p-map` 库控制单批次内的并发度：

```typescript
await pMap(
  runs,
  async (run) => { /* 处理单个运行 */ },
  { concurrency: env.BULK_ACTION_SUBBATCH_CONCURRENCY }
);
```

**批次间延迟**: 处理完一批后，延迟 `BULK_ACTION_BATCH_DELAY_MS` 再处理下一批，避免对系统造成瞬时压力。

### 2.3 结果回收

```typescript
// 每批处理后更新计数
await this._prisma.bulkActionGroup.update({
  where: { id: bulkActionId },
  data: {
    cursor: runIdsToProcess.at(-1),  // 记录处理位置
    successCount: { increment: successCount },
    failureCount: { increment: failureCount },
    status: isFinished ? "COMPLETED" : undefined,
    completedAt: isFinished ? new Date() : undefined,
  },
});
```

**游标分页**: 使用最后处理的 runId 作为 cursor，避免 offset 分页在数据变化时的重复/遗漏问题。

---

## 三、两者的协同与差异

### 3.1 核心差异对比

| 维度 | Batch Trigger | Bulk Action |
|------|--------------|-------------|
| **目标** | 创建新任务 | 操作现有任务 |
| **触发源** | API / SDK 调用 | 用户在 Dashboard 操作 |
| **数据来源** | 提交的 items 列表 | ClickHouse 查询结果 |
| **调度器** | FairQueue + DRR | commonWorker 简单队列 |
| **批次拆分** | 按固定大小（50）拆分 | 按 cursor 分页拉取 |
| **并发模型** | 顺序处理（v1）/ 公平调度（v2） | 批次内 p-map 并发 |
| **错误处理** | 创建 pre-failed run | 记录 failureCount，继续推进 |
| **结果存储** | BatchTaskRun.runIds | BulkActionGroup.successCount/failureCount |

### 3.2 共享的设计模式

#### 模式 1: 接力式批次处理

```
[批次 1] 处理 → 返回 INCOMPLETE → 入队 [批次 2]
     ↑                                    ↓
     └────────────────────────────────────┘
```

两种机制都采用这种**自我调度**模式：当前批次处理完后，如果还有未处理项，就自己入队下一批次。

**优势**:
- 避免长时间阻塞单个 worker
- 支持中断恢复（进程重启后可从 cursor 继续）
- 便于流量控制（批次间延迟）

#### 模式 2: 幂等性保证

Batch Trigger v2 使用 Redis 位图标记已处理项：
```typescript
const isNewItem = await this.completionTracker.markItemEnqueued(batchId, itemIndex);
if (!isNewItem) return { enqueued: false };  // 重复项跳过
```

#### 模式 3: 进度可观测

- **Batch Trigger**: Redis 原子计数器 + `getBatchProgress()` API
- **Bulk Action**: Postgres 实时更新 `successCount`/`failureCount`

---

## 四、完整链路示例：batchTriggerAndWait

这是 Batch Trigger 最复杂的使用场景，展示了完整的协同流程：

```
用户代码: await batch.triggerAndWait(items)
    │
    ▼
1. SDK 发送 batch trigger 请求
    │
    ▼
2. CreateBatchService 创建 BatchTaskRun（status=PENDING）
    │
    ▼
3. 阻塞父运行: blockRunWithCreatedBatch()
    ├─→ 创建 Waitpoint（等待批次完成）
    └─→ 父运行进入 BLOCKED 状态
    │
    ▼
4. 流式入队所有 items 到 BatchQueue
    │
    ▼
5. BatchQueue 消费者按 DRR 调度处理 items
    ├─→ 每个 item 触发一个任务
    └─→ 成功/失败都原子更新计数
    │
    ▼
6. 最后一个 item 处理完成
    ├─→ processedCount === runCount
    └─→ 调用 completionCallback
    │
    ▼
7. BatchSystem.performCompleteBatch()
    ├─→ 检查所有子运行是否都到终态
    ├─→ 更新 BatchTaskRun.status = COMPLETED
    └─→ 完成 Waitpoint
    │
    ▼
8. WaitpointSystem.completeWaitpoint()
    ├─→ 标记 waitpoint 为 COMPLETED
    └─→ 入队 continueRunIfUnblocked
    │
    ▼
9. 父运行被唤醒，继续执行
    └─→ 返回批次结果给用户代码
```

---

## 五、关键代码位置速查

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| v1 批次触发 | `batchTrigger.server.ts` | `call()`, `processBatchTaskRun()`, `#processBatchTaskRunItems()` |
| v2 批次队列 | `batch-queue/index.ts` | `initializeBatch()`, `enqueueBatchItem()`, `#handleMessage()`, `#finalizeBatch()` |
| 批次完成检测 | `batchSystem.ts` | `performCompleteBatch()`, `#tryCompleteBatch()` |
| 等待点系统 | `waitpointSystem.ts` | `completeWaitpoint()` |
| 批次限流 | `batchLimits.server.ts` | `getBatchLimits()`, `BatchRateLimitExceededError` |
| 全局速率限制 | `batchGlobalRateLimiter.server.ts` | `createBatchGlobalRateLimiter()` |
| Bulk Action | `BulkActionV2.server.ts` | `create()`, `process()`, `abort()` |
| 批次 Worker | `batchTriggerWorker.server.ts` | `initializeWorker()` |

---

## 七、Bulk Action 作用于 Batch Trigger 运行的协同链路

当 Bulk Action（取消/重播）作用于由 Batch Trigger 生成的运行时，会触发一条复杂的协同链路，涉及**状态回写**、**批次完成判定**和**父任务等待恢复**三个核心环节。

### 7.1 场景说明

考虑以下典型场景：

```
用户代码中执行 batch.triggerAndWait([...100个任务...])
    │
    ├─→ 父任务被阻塞（等待 Waitpoint）
    ├─→ 创建 BatchTaskRun（100个预期运行）
    └─→ 100个子任务开始执行
          ├─→ 50个已完成（SUCCEEDED/FAILED）
          ├─→ 30个正在执行（RUNNING）
          └─→ 20个排队中（PENDING）
```

此时用户在 Dashboard 中：
- **场景 A**: 选择 "取消所有运行" → Bulk CANCEL Action
- **场景 B**: 选择 "重播所有失败运行" → Bulk REPLAY Action

### 7.2 链路 1：Bulk Cancel 对 Batch Trigger 的影响

#### 阶段 1：Bulk Action 发起取消

```
用户在 Dashboard 选择批量取消 → 创建 BulkActionGroup(type=CANCEL)
    │
    ▼
BulkActionService.process() 拉取一批 runIds（带 batchId）
    │
    ▼
对每个 runId 调用 CancelTaskRunService.call(run, { bulkActionId })
    │
    ├─→ V2 运行 → engine.cancelRun({ runId, bulkActionId })
    │     ├─→ 取消正在执行的运行
    │     └─→ 运行状态更新为 CANCELED
    │           ↓
    │           关键：#finalizeRun({ id, batchId })
    │                 ↓
    │                 batchSystem.scheduleCompleteBatch({ batchId })
    │
    └─→ bulkActionId 追加到 taskRun.bulkActionGroupIds 数组
```

**关键代码** (`runAttemptSystem.ts:1723-1725`):
```typescript
async #finalizeRun({ id, batchId }: { id: string; batchId: string | null }) {
  if (batchId) {
    await this.batchSystem.scheduleCompleteBatch({ batchId });
  }
  // ...
}
```

每个被取消的运行如果属于某个批次（`batchId != null`），都会触发批次完成检测。

#### 阶段 2：批次完成判定（去抖动 + 幂等）

`scheduleCompleteBatch` 会延迟 200ms 后执行，目的是**去抖动**：当批量取消 100 个运行时，不会触发 100 次批次完成检测，而是合并为一次。

```typescript
// batchSystem.ts:20-29
public async scheduleCompleteBatch({ batchId }: { batchId: string }): Promise<void> {
  await this.$.worker.enqueue({
    id: `tryCompleteBatch:${batchId}`,  // 相同 ID 会自动去重
    job: "tryCompleteBatch",
    payload: { batchId: batchId },
    availableAt: new Date(Date.now() + 200),  // 200ms 延迟
  });
}
```

#### 阶段 3：批次完成检测逻辑

```typescript
// batchSystem.ts:39-136
async #tryCompleteBatch({ batchId }: { batchId: string }) {
  const batch = await this.$.prisma.batchTaskRun.findFirst({...});
  
  // v2 批次使用 successfulRunCount + failedRunCount
  const processedRunCount = batch.successfulRunCount + batch.failedRunCount;
  
  // 关键判断 1：所有运行都已被处理（创建或标记失败）
  if (processedRunCount < batch.runCount) {
    return;  // 还有运行未创建，不完成批次
  }
  
  // 关键判断 2：所有运行都到达终态
  const runs = await this.$.prisma.taskRun.findMany({
    where: { batchId, runtimeEnvironmentId: batch.runtimeEnvironmentId },
    select: { id: true, status: true },
  });
  
  if (runs.every((r) => isFinalRunStatus(r.status))) {
    // 所有运行都完成了（包括 CANCELED）
    await this.$.prisma.batchTaskRun.update({
      where: { id: batchId },
      data: { status: "COMPLETED" },
    });
    
    // 检查是否有等待点（triggerAndWait 场景）
    const waitpoint = await this.$.prisma.waitpoint.findFirst({
      where: { completedByBatchId: batchId },
    });
    
    if (waitpoint) {
      await this.waitpointSystem.completeWaitpoint({
        id: waitpoint.id,
        output: { value: "Batch waitpoint completed", isError: false },
      });
    }
  }
}
```

**重要结论**:
- `CANCELED` 状态被视为终态（`isFinalRunStatus` 返回 true）
- 即使所有运行都被取消，批次仍然可以正常完成
- 批次完成后会自动触发 Waitpoint 完成

#### 阶段 4：父任务恢复

当 Waitpoint 完成后，父任务被唤醒：

```typescript
// waitpointSystem.ts:72-150
async completeWaitpoint({ id, output }) {
  // 1. 更新 waitpoint 状态为 COMPLETED
  await this.$.prisma.waitpoint.updateMany({...});
  
  // 2. 查找被阻塞的父运行
  const affectedTaskRuns = await this.$.prisma.taskRunWaitpoint.findMany({
    where: { waitpointId: id },
    select: { taskRunId: true, ... },
  });
  
  // 3. 入队继续运行的任务
  for (const run of affectedTaskRuns) {
    await this.$.worker.enqueue({
      id: `continueRunIfUnblocked:${run.taskRunId}`,
      job: "continueRunIfUnblocked",
      payload: { runId: run.taskRunId },
      availableAt: new Date(Date.now() + 50),
    });
  }
}
```

父任务恢复后，`batch.triggerAndWait()` 会返回结果，其中包含所有子运行的 ID（包括被取消的）。

### 7.3 链路 2：Bulk Replay 对 Batch Trigger 的影响

Bulk Replay 的影响更为复杂，因为它**创建新运行**而不是修改现有运行状态。

#### 阶段 1：Bulk Action 发起重播

```
用户选择批量重播 → 创建 BulkActionGroup(type=REPLAY)
    │
    ▼
BulkActionService.process() 拉取一批 runIds（带 batchId）
    │
    ▼
对每个 runId 调用 ReplayTaskRunService.call(run, { bulkActionId })
    │
    ├─→ 读取原运行的 payload 和配置
    ├─→ 调用 TriggerTaskService 创建新运行
    │     └─→ 新运行的 bulkActionGroupIds 包含该 bulkActionId
    └─→ 注意：新运行**没有** batchId！
```

**关键点**: 重播创建的新运行**不属于原来的批次**。原来的批次只跟踪它最初创建的那些运行。

#### 阶段 2：对原批次的影响

重播操作本身不会直接影响原批次的完成判定，因为：
1. 原运行仍然存在（状态可能是 FAILED）
2. 新运行有自己的 ID，不关联原 batchId
3. 原批次的 `runCount` 和 `runIds` 不会变化

但如果原运行在重播前处于非终态，重播操作可能会：
- 先取消原运行（取决于实现）
- 取消会触发 `#finalizeRun` → `scheduleCompleteBatch`
- 从而加速原批次的完成检测

#### 阶段 3：批次完成判定不受影响

原批次的完成判定仍然只看它自己创建的那些运行：

```
原批次 (batch_123) 有 100 个运行：
  ├─→ run_001 ~ run_100（batchId = batch_123）
  │     ├─→ 50 个 SUCCEEDED
  │     ├─→ 30 个 FAILED（用户选择重播）
  │     └─→ 20 个 CANCELED
  │
  └─→ 重播创建的新运行：run_101 ~ run_130（batchId = null）
```

当 `run_001` ~ `run_100` 全部到达终态后，批次 `batch_123` 就完成了，父任务恢复。新运行 `run_101` ~ `run_130` 的生命周期与原批次无关。

### 7.4 状态回写设计：bulkActionGroupIds 字段

每个运行都有一个 `bulkActionGroupIds` 数组字段，记录对其执行过的所有 Bulk Action：

```typescript
// runAttemptSystem.ts:1403-1407
data: {
  status: "CANCELED",
  // ...
  bulkActionGroupIds: bulkActionId
    ? { push: bulkActionId }
    : undefined,
}
```

**设计意图**:
1. **可追溯性**: 可以查询某个运行被哪些 Bulk Action 影响过
2. **审计**: 支持 "这个运行为什么被取消了？" 这类问题的回答
3. **幂等性**: 避免重复执行相同的 Bulk Action（虽然当前实现没有检查）

**注意**: 这个字段是**纯标记**，不影响批次完成判定逻辑。批次完成只看 `status` 是否为终态。

### 7.5 完整时序图（以 Bulk Cancel 为例）

```
  Bulk Action 发起者          Run Engine         Batch System      Waitpoint System
       │                        │                    │                   │
       │ 批量取消 100 个运行     │                    │                   │
       ├───────────────────────▶│                    │                   │
       │                        │                    │                   │
       │                    取消 run_001             │                   │
       │                        ├───┐                │                   │
       │                        │   │ 状态→CANCELED   │                   │
       │                        │   │                │                   │
       │                        │◀──┘                │                   │
       │                        │                    │                   │
       │                        #finalizeRun         │                   │
       │                        │                    │                   │
       │                        ├───────────────────▶│                   │
       │                        │ scheduleCompleteBatch                   │
       │                        │                    │                   │
       │                    取消 run_002             │                   │
       │                        ├───┐                │                   │
       │                        │   │ 状态→CANCELED   │                   │
       │                        │   │                │                   │
       │                        │◀──┘                │                   │
       │                        │                    │                   │
       │                        #finalizeRun         │                   │
       │                        │                    │                   │
       │                        ├───────────────────▶│ (去重，只保留一个)  │
       │                        │                    │                   │
       │                        ... (重复 98 次)     │                   │
       │                        │                    │                   │
       │                        │                    │ 200ms 后执行        │
       │                        │                    │ tryCompleteBatch   │
       │                        │                    │                    │
       │                        │                    ├───┐               │
       │                        │                    │   │ 检查所有运行    │
       │                        │                    │   │ 是否为终态      │
       │                        │                    │   │                │
       │                        │                    │◀──┘               │
       │                        │                    │                    │
       │                        │                    │ Batch.status       │
       │                        │                    │ = COMPLETED        │
       │                        │                    │                    │
       │                        │                    ├───────────────────▶│
       │                        │                    │ completeWaitpoint  │
       │                        │                    │                    │
       │                        │                    │                    ├───┐
       │                        │                    │                    │   │ 唤醒父任务
       │                        │                    │                    │   │
       │                        │                    │                    │◀──┘
       │◀─────────────────────────────────────────────────────────────────┤
       │                     batch.triggerAndWait() 返回结果              │
```

### 7.6 关键设计决策分析

#### 决策 1：为什么每个运行取消时都触发批次检测，而不是批量检测后统一触发？

```typescript
// 每个运行取消后都调用
if (batchId) {
  await this.batchSystem.scheduleCompleteBatch({ batchId });
}
```

**原因**:
1. **解耦**: Bulk Action 不需要知道它取消的运行属于哪个批次，也不需要关心批次逻辑
2. **实时性**: 最后一个运行取消后 200ms 内就能完成批次检测
3. **鲁棒性**: 即使 Bulk Action 处理过程中断，已取消的运行仍然会触发批次检测
4. **去抖动**: 通过 `scheduleCompleteBatch` 的延迟 + 唯一 job ID 实现合并

#### 决策 2：为什么 CANCELED 被视为终态？

```typescript
// isFinalRunStatus 包括 CANCELED
if (runs.every((r) => isFinalRunStatus(r.status))) {
  // 批次完成
}
```

**原因**:
1. **用户期望**: 用户取消批量运行后，期望父任务能立即恢复，而不是永远等待
2. **语义正确**: 被取消的运行不会再执行，确实是"最终"状态
3. **结果可用**: 即使部分/全部运行被取消，批次结果仍然有意义（哪些成功了，哪些被取消了）

#### 决策 3：为什么重播的新运行不继承原 batchId？

**原因**:
1. **批次语义**: 批次代表"一次批量触发操作"，重播是另一次独立操作
2. **计数准确**: 原批次的 `runCount` 是固定的，添加新运行会导致计数混乱
3. **责任清晰**: 原批次只对它创建的运行负责，重播的运行由 Bulk Action 负责
4. **实现简单**: 不需要修改 BatchTaskRun 的不可变字段

---

## 八、设计权衡与思考

### 8.1 为什么 v1 批次强制顺序处理？

```typescript
// batchTrigger.server.ts:68-72
// Eric note: We need to force sequential processing because when doing parallel,
// we end up with high-contention on the parent run lock...
this._batchProcessingStrategy = "sequential";
```

**权衡**: 牺牲并行速度，避免数据库锁竞争导致的死锁和性能下降。对于批次触发来说，创建任务的速度通常不是瓶颈，任务本身的执行才是。

### 8.2 为什么 v2 批次使用 DRR 调度？

**问题**: 单个大客户的 10 万项批次可能阻塞所有其他用户的批次处理。

**解决方案**: DRR 确保每个环境（tenant）公平分配处理资源，通过 `quantum` 控制每轮处理的消息数。

### 8.3 为什么 Bulk Action 不用 BatchQueue？

- Bulk Action 的操作（取消/重播）通常很快，不需要复杂的调度
- Bulk Action 是用户交互式操作，响应速度比公平调度更重要
- Bulk Action 的数据来源是查询结果，不是预定义的 items 列表

---

## 总结

### 核心设计模式

Batch Trigger 和 Bulk Action 虽然目标不同，但共享了**"批次拆分 → 并发控制 → 接力处理 → 结果回收"**的核心设计模式：

1. **批次拆分** 确保系统不会被大任务压垮
2. **多层并发控制** 保障系统稳定性和公平性
3. **接力式处理** 支持中断恢复和流量平滑
4. **原子计数 + 回调** 实现可靠的结果回收

### 跨系统协同的关键洞察

当 Bulk Action 作用于 Batch Trigger 生成的运行时，两者通过**事件驱动 + 去抖动**的方式协同：

1. **状态回写**: `bulkActionGroupIds` 字段实现可追溯性，不影响业务逻辑
2. **完成检测**: 每个运行终态化时触发 `scheduleCompleteBatch`，通过 200ms 延迟 + 唯一 Job ID 实现去抖动
3. **等待恢复**: `CANCELED` 被视为终态，确保用户取消后父任务能及时恢复
4. **边界清晰**: 重播创建的新运行不继承原 batchId，保持批次语义纯净

### 设计哲学

整个系统体现了以下设计原则：
- **解耦优于协调**: Bulk Action 不需要知道批次逻辑，通过事件隐式协同
- **最终一致性**: 不追求强一致，通过去抖动和重试达到最终一致
- **用户期望优先**: `CANCELED` 作为终态、父任务及时恢复都是为了符合用户直觉
- **鲁棒性**: 任何环节中断都不会导致系统死锁或状态不一致

理解这些机制有助于在使用 Trigger.dev 时更好地规划批量任务，以及在遇到问题时快速定位。
