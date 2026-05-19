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

## 六、设计权衡与思考

### 6.1 为什么 v1 批次强制顺序处理？

```typescript
// batchTrigger.server.ts:68-72
// Eric note: We need to force sequential processing because when doing parallel,
// we end up with high-contention on the parent run lock...
this._batchProcessingStrategy = "sequential";
```

**权衡**: 牺牲并行速度，避免数据库锁竞争导致的死锁和性能下降。对于批次触发来说，创建任务的速度通常不是瓶颈，任务本身的执行才是。

### 6.2 为什么 v2 批次使用 DRR 调度？

**问题**: 单个大客户的 10 万项批次可能阻塞所有其他用户的批次处理。

**解决方案**: DRR 确保每个环境（tenant）公平分配处理资源，通过 `quantum` 控制每轮处理的消息数。

### 6.3 为什么 Bulk Action 不用 BatchQueue？

- Bulk Action 的操作（取消/重播）通常很快，不需要复杂的调度
- Bulk Action 是用户交互式操作，响应速度比公平调度更重要
- Bulk Action 的数据来源是查询结果，不是预定义的 items 列表

---

## 总结

Batch Trigger 和 Bulk Action 虽然目标不同，但共享了**"批次拆分 → 并发控制 → 接力处理 → 结果回收"**的核心设计模式：

1. **批次拆分** 确保系统不会被大任务压垮
2. **多层并发控制** 保障系统稳定性和公平性
3. **接力式处理** 支持中断恢复和流量平滑
4. **原子计数 + 回调** 实现可靠的结果回收

理解这些机制有助于在使用 Trigger.dev 时更好地规划批量任务，以及在遇到问题时快速定位。
