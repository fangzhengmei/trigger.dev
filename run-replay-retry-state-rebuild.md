# Run Replay 与自动 Retry 状态机重建分析（口径统一版）

> **⚠️ 版本历史**：
> - v1.0：初始分析（存在部分不准确结论）
> - v2.0：代码验证版（修正 5 个关键错误结论）
> - v3.0：最终验证版（阐述 checkpoint 双分支差异、补充 waitpoint 完整代码证据）
> - **v4.0：口径统一版（✅ 本次更新）**：消除自动 retry 触发状态的前后口径冲突，明确 4 种状态的触发边界，补充完整代码证据链

> **核心修正点（v4.0）**：
> 1. **口径冲突修正**：之前「仅执行中失败后触发」的表述不准确，实际是 **4 种状态可触发自动 retry**
> 2. **边界明确**：PENDING_EXECUTING / EXECUTING / EXECUTING_WITH_WAITPOINTS / QUEUED_EXECUTING 四种状态的具体失败场景
> 3. **代码证据链**：补充 3 个核心入口的代码验证（attemptFailed 检查、startRunAttempt 前置检查、重试决策逻辑）
> 4. **重试方式差异**：明确 RETRY_QUEUED 与 RETRY_IMMEDIATELY 两种方式的触发条件与适用场景

---

## 一、核心概念差异

### 1.1 Replay（手工重放）

**定义**：对任意状态的 run 进行手工触发的重新执行，创建一个全新的 run 实例。

**核心代码入口**：`apps/webapp/app/v3/services/replayTaskRun.server.ts:24-144`

**✅ 正确结论**：
- 创建全新的 `TaskRun` 数据库记录，拥有独立的 `id` 和 `friendlyId`
- 通过 `replayedFromTaskRunFriendlyId` 字段记录源 run 的关联关系
- 输入参数可选择性覆盖（payload、metadata、tags 等）
- 支持跨环境重放（如从生产环境重放到开发环境）
- **对源 run 状态无限制**：不检查源 run 是否完成/失败/取消，只要环境未归档即可 replay

### 1.2 自动 Retry（自动重试）

**定义**：同一个 run 实例在执行失败后，根据重试配置自动进行的再次尝试，不创建新的 run。

**核心代码入口**：`internal-packages/run-engine/src/engine/retrying.ts:46-178`

**关键特性**：
- 复用同一个 `TaskRun` 数据库记录
- 通过递增 `attemptNumber` 字段区分不同尝试
- 每次尝试都会创建新的 `TaskRunExecutionSnapshot`
- 重试策略由任务的 `retry` 配置控制（最大尝试次数、退避算法等）
- 支持 OOM（内存不足）时自动升级机器规格后重试

### 1.3 核心差异对比表

| 维度 | Replay（手工重放） | 自动 Retry |
|------|-------------------|------------|
| **Run 实体** | 创建全新 run | 复用原有 run |
| **Run ID** | 新的 friendlyId | 相同 friendlyId |
| **关联字段** | `replayedFromTaskRunFriendlyId` | `attemptNumber` 递增 |
| **触发时机** | 手工操作（API/UI） | 执行失败后自动触发 |
| **源 run 状态检查** | ❌ 无检查（仅检查环境未归档） | 仅在 `PENDING_EXECUTING`/`EXECUTING` 状态下触发 |
| **输入参数** | 可覆盖修改 | 完全复用 |
| **执行上下文** | 重建 | 重建（但复用部分状态） |
| **子任务/副作用** | 全部重新执行 | 从 checkpoint 恢复或全部重试 |
| **计费** | 新 run 独立计费 | 累计到同一 run |
| **最大次数限制** | 无限制（手工操作） | 受 `maxAttempts` 限制 |
| **追踪链路** | Link 关联 | 同一 trace 下的不同 span |

---

## 二、状态适用矩阵（经过源码验证）

### 2.1 Checkpoint 可创建状态矩阵

**✅ 验证代码**：`internal-packages/run-engine/src/engine/statuses.ts:21-32`

```typescript
export function isCheckpointable(status: TaskRunExecutionStatus): boolean {
  const checkpointableStatuses: TaskRunExecutionStatus[] = [
    "RUN_CREATED",
    "QUEUED",
    "EXECUTING",
    "EXECUTING_WITH_WAITPOINTS",
    "QUEUED_EXECUTING",
  ];
  return checkpointableStatuses.includes(status);
}
```

| ExecutionStatus | 是否可创建 Checkpoint | 说明 |
|----------------|---------------------|------|
| `RUN_CREATED` | ✅ 是 | run 刚创建时 |
| `DELAYED` | ❌ 否 | 延迟执行中不可 checkpoint |
| `QUEUED` | ✅ 是 | 队列中等待时 |
| `PENDING_EXECUTING` | ❌ 否 | 出队后等待执行中不可 checkpoint |
| `EXECUTING` | ✅ 是 | 执行中 |
| `EXECUTING_WITH_WAITPOINTS` | ✅ 是 | 执行中但等待子任务 |
| `QUEUED_EXECUTING` | ✅ 是 | 队列中但同时在执行（checkpoint 后特殊状态） |
| `SUSPENDED` | ❌ 否 | **已暂停后不可再 checkpoint**（已验证：`checkpoints.test.ts:759`） |
| `PENDING_CANCEL` | ❌ 否 | 待取消 |
| `FINISHED` | ❌ 否 | 已完成 |

> **❌ 之前的错误结论**：认为 SUSPENDED 状态可以创建 checkpoint
> 
> **✅ 正确结论**：SUSPENDED 状态后无法创建新的 checkpoint，测试用例明确验证了 "Status SUSPENDED is not checkpointable"

### 2.2 自动 Retry 可触发状态矩阵（✅ 完整代码验证 · 口径统一版）

> **⚠️ 口径统一说明**：
> - 之前的表述「仅执行中失败后触发」是不完整的，实际覆盖 4 种状态
> - 正确表述：**在 4 种执行相关状态下发生失败时可触发自动 retry**
> - 唯一明确禁止的状态：`FINISHED`（代码有显式检查）

---

#### 2.2.1 状态矩阵与代码证据

| ExecutionStatus | 是否可自动 Retry | 触发场景 | 代码证据 |
|----------------|-----------------|---------|----------|
| `RUN_CREATED` | ❌ 否 | 尚未执行，无 attempt 可失败 | - |
| `DELAYED` | ❌ 否 | 延迟中，无 attempt 可失败 | - |
| `QUEUED` | ❌ 否 | 队列中，尚未分配 worker | - |
| `PENDING_EXECUTING` | ✅ 是 | 出队后 worker 初始化阶段失败（如 OOM、代码加载失败） | `runAttemptSystem.ts:861-893` |
| `EXECUTING` | ✅ 是 | 业务代码执行中抛出异常 | `runAttemptSystem.ts:861-893` |
| `EXECUTING_WITH_WAITPOINTS` | ✅ 是 | 等待子任务时 worker 崩溃或超时 | `runAttemptSystem.ts:861-893` |
| `QUEUED_EXECUTING` | ✅ 是 | checkpoint 后重新入队，执行阶段失败 | `runAttemptSystem.ts:861-893` + `checkpointSystem.ts:175-208` |
| `SUSPENDED` | ❌ 否 | 已暂停，需通过 resume 流程恢复而非 retry | - |
| `PENDING_CANCEL` | ❌ 否 | 待取消状态，由取消流程处理 | `runAttemptSystem.ts:368` |
| `FINISHED` | ❌ 否 | **显式检查禁止**，已完成后不可自动 retry | `runAttemptSystem.ts:891-893` |

---

#### 2.2.2 ✅ 核心代码验证证据

**入口 1：attemptFailed() 入口检查**
```typescript
// runAttemptSystem.ts:891-893
if (latestSnapshot.executionStatus === "FINISHED") {
  throw new ServiceValidationError("Run is already finished", 400);
}
// ✅ 证明：唯一显式禁止的状态是 FINISHED
```

**入口 2：startRunAttempt() 前置检查**
```typescript
// runAttemptSystem.ts:368-370
if (isFinishedOrPendingFinished(latestSnapshot.executionStatus)) {
  throw new ServiceValidationError("Task run is already finished", 400);
}

// statuses.ts:34-37
export function isFinishedOrPendingFinished(status: TaskRunExecutionStatus): boolean {
  const finishedStatuses: TaskRunExecutionStatus[] = ["FINISHED", "PENDING_CANCEL"];
  return finishedStatuses.includes(status);
}
// ✅ 证明：startRunAttempt 时会排除 FINISHED 和 PENDING_CANCEL
```

**入口 3：重试决策（不限制当前状态）**
```typescript
// retrying.ts:46-178
export async function retryOutcomeFromCompletion(
  prisma: PrismaClientOrTransaction,
  { runId, attemptNumber, error, retryUsingQueue, retrySettings }: Params
): Promise<RetryOutcome> {
  // 只检查：
  // 1. 是否是取消错误 → cancel_run
  // 2. 是否是 OOM 错误 → 特殊升级机器逻辑
  // 3. 错误是否可重试
  // 4. attempt 次数是否超限
  // 5. 是否有重试配置
  // ✅ 证明：重试决策本身不对当前 executionStatus 做限制
}
```

---

#### 2.2.3 ✅ 四种可重试状态的失败路径详解

| 状态 | 失败发生阶段 | 失败类型 | 触发方式 |
|------|-------------|---------|---------|
| **PENDING_EXECUTING** | Worker 出队后，执行用户代码前 | - OOM（内存不足）<br>- 代码加载/导入失败<br>- 初始化异常<br>- Worker 进程崩溃 | 正常 attemptFailed() 调用 |
| **EXECUTING** | 执行用户业务代码中 | - 业务代码抛出异常<br>- 运行时错误<br>- Worker 超时/崩溃 | 正常 attemptFailed() 调用 |
| **EXECUTING_WITH_WAITPOINTS** | 等待子任务完成时 | - Worker 崩溃<br>- 等待超时<br>- 检测到子任务失败 | 正常 attemptFailed() 调用 |
| **QUEUED_EXECUTING** | checkpoint 后重新入队的执行阶段 | - 恢复 checkpoint 失败<br>- 断点续跑时代码异常<br>- Worker 崩溃 | forceRequeue=true 调用 attemptFailed() |

---

#### 2.2.4 ✅ 重试后的两种执行方式

```typescript
// runAttemptSystem.ts:1073-1133
if (forceRequeue || retryResult.method === "queue" || 长延迟) {
  // 方式 1：RETRY_QUEUED（重新入队）
  // → NACK 消息重新入队
  // → 创建状态为 QUEUED 的 snapshot
  // → 等待 worker 重新拉取
  // 适用场景：
  //   - forceRequeue=true（QUEUED_EXECUTING 状态失败）
  //   - OOM 错误需要升级机器
  //   - 重试延迟超过阈值
} else {
  // 方式 2：RETRY_IMMEDIATELY（立即重试）
  // → 直接创建状态为 EXECUTING 的新 snapshot
  // → 通知当前 worker 继续执行
  // → 无需重新入队
  // 适用场景：
  //   - 重试延迟短（< retryWarmStartThresholdMs）
}
```

> **关键提示**：重试方式与当前失败时的状态无关，只与失败类型、延迟配置有关

### 2.3 Replay 可触发状态矩阵

**✅ 验证代码**：`apps/webapp/app/v3/services/replayTaskRun.server.ts:33-35`

| TaskRunStatus | 是否可 Replay | 说明 |
|--------------|--------------|------|
| `CREATED` | ✅ 是 | 刚创建的 run 也可 replay |
| `PENDING` | ✅ 是 | 队列中可 replay |
| `DELAYED` | ✅ 是 | 延迟中可 replay |
| `DEQUEUED` | ✅ 是 | 出队中可 replay |
| `EXECUTING` | ✅ 是 | **执行中也可 replay** |
| `WAITING_TO_RESUME` | ✅ 是 | checkpoint 后可 replay |
| `CANCELED` | ✅ 是 | 已取消可 replay |
| `INTERRUPTED` | ✅ 是 | 已中断可 replay |
| `COMPLETED_SUCCESSFULLY` | ✅ 是 | 成功完成可 replay |
| `COMPLETED_WITH_ERRORS` | ✅ 是 | 完成但有错误可 replay |
| `SYSTEM_FAILURE` | ✅ 是 | 系统失败可 replay |
| `CRASHED` | ✅ 是 | 崩溃可 replay |
| `EXPIRED` | ✅ 是 | 已过期可 replay |
| `TIMED_OUT` | ✅ 是 | 超时可 replay |

> **❌ 之前的错误结论**：认为 replay 仅适用于已完成/失败的 run
> 
> **✅ 正确结论**：Replay 服务**完全不检查源 run 状态**！仅检查 `environment.archivedAt`。理论上，**任何状态的 run 都可以被 replay**。

### 2.4 最终状态矩阵

**✅ 验证代码**：`internal-packages/run-engine/src/engine/statuses.ts:44-57`

```typescript
const finalStatuses: TaskRunStatus[] = [
  "CANCELED",
  "INTERRUPTED",
  "COMPLETED_SUCCESSFULLY",
  "COMPLETED_WITH_ERRORS",
  "SYSTEM_FAILURE",
  "CRASHED",
  "EXPIRED",
  "TIMED_OUT",
];
```

这些状态表示 run 的生命周期已结束，**不会再自动 retry**，但**仍然可以被 replay**。

---

## 三、状态机从断点/起点重建过程

### 3.1 核心数据结构：ExecutionSnapshot（执行快照）

状态机的重建完全基于 `TaskRunExecutionSnapshot` 表，每个快照代表 run 在某个时间点的执行状态。

**核心表结构**：
```prisma
model TaskRunExecutionSnapshot {
  id                  String   @id
  runId               String
  executionStatus     TaskRunExecutionStatus
  description         String
  previousSnapshotId  String?
  attemptNumber       Int?
  checkpointId        String?
  completedWaitpoints Waitpoint[]
  checkpoint          TaskRunCheckpoint?
  // ...
}
```

### 3.2 自动 Retry 状态重建流程

**代码位置**：`internal-packages/run-engine/src/engine/systems/runAttemptSystem.ts:861-1142`

```
失败发生 → retryOutcomeFromCompletion() → 判断重试策略
    ↓
┌──────────────────────────────────────────────────────────┐
│ ✅ 关键步骤 1：清理阻塞 waitpoints                        │
│    await this.waitpointSystem.clearBlockingWaitpoints()   │
│    → 删除 taskRunWaitpoint 表中该 run 的所有阻塞记录       │
│    → 但 snapshot.completedWaitpoints 保留                 │
└──────────────────────────────────────────────────────────┘
    ↓
创建新 ExecutionSnapshot（executionStatus: "EXECUTING"）
    ↓
递增 attemptNumber
    ↓
累积 usageDurationMs 和 costInCents
    ↓
┌─────────────────────────────────────┐
│ 两种重试方式：                        │
│ 1. RETRY_IMMEDIATELY（短延迟）       │
│    → 通知 worker 直接创建新 attempt │
│ 2. RETRY_QUEUED（长延迟/队列重试）   │
│    → NACK 消息重新入队              │
│    → 状态重置为 QUEUED              │
└─────────────────────────────────────┘
    ↓
worker 拉取 run → startRunAttempt()
    ↓
┌──────────────────────────────────────────────────────────┐
│ ✅ 关键步骤 2：从 latestSnapshot 重建状态                 │
│    - completedWaitpoints：通过 connect 保留已完成的        │
│      （dequeueSystem.ts:457-458）                         │
│    - checkpointId：保留（如果有）                         │
│      （dequeueSystem.ts:455）                             │
│    - 已累积的使用时长和费用：继续累加                      │
└──────────────────────────────────────────────────────────┘
```

### 3.3 ✅ Waitpoint 重试处理机制（重要修正 · 完整代码证据）

**代码位置**：
| 操作 | 文件 | 行号 |
|------|------|------|
| 失败时清除 blocking waitpoints | `runAttemptSystem.ts` | 897-901 |
| clearBlockingWaitpoints 实现 | `waitpointSystem.ts` | 53-68 |
| 新 snapshot 保留 completedWaitpoints（dequeue） | `dequeueSystem.ts` | 457-458 |
| 新 snapshot 保留 completedWaitpoints（requeue） | `runAttemptSystem.ts` | 1276-1277 |

---

#### 3.3.1 两种 Waitpoint 的本质区别

| 维度 | Blocking Waitpoints | Completed Waitpoints |
|------|--------------------|---------------------|
| **存储位置** | `taskRunWaitpoint` 关联表 | `snapshot.completedWaitpoints`（通过 join 关联） |
| **数据结构** | 多对多关联表（run ↔ waitpoint） | Waitpoint 实体数组，通过 snapshot 关联 |
| **表示含义** | run **当前正在等待**的子任务 | run **历史上已完成**的子任务 |
| **生命周期** | 子任务完成后删除 | 永久保留在 snapshot 链上 |
| **重试时处理** | ❌ 全部删除 | ✅ 完整保留并传递 |
| **代码证据** | `waitpointSystem.ts:53-68` | `dequeueSystem.ts:457-458` |

---

#### 3.3.2 失败重试时的处理流程（✅ 代码验证）

```typescript
// ============================================
// 步骤 1：清除 blocking waitpoints
// ============================================
// runAttemptSystem.ts:897-901
// remove waitpoints blocking the run
const deletedCount = await this.waitpointSystem.clearBlockingWaitpoints({ runId, tx });
if (deletedCount > 0) {
  this.$.logger.debug("Cleared blocking waitpoints", { runId, deletedCount });
}

// waitpointSystem.ts:53-68
public async clearBlockingWaitpoints({
  runId,
  tx,
}: {
  runId: string;
  tx?: PrismaClientOrTransaction;
}) {
  const prisma = tx ?? this.$.prisma;
  // 直接删除 taskRunWaitpoint 表中该 run 的所有记录
  const deleted = await prisma.taskRunWaitpoint.deleteMany({
    where: {
      taskRunId: runId,
    },
  });
  return deleted.count;
}

// ============================================
// 步骤 2：保留 completedWaitpoints（通过 snapshot 传递）
// ============================================
// dequeueSystem.ts:457-458
completedWaitpoints: {
  connect: snapshot.completedWaitpoints.map((w) => ({ id: w.id })),
}
// 通过 connect 关联已有的 waitpoint 记录
// 新 snapshot 与这些 waitpoint 建立关联关系

// runAttemptSystem.ts:1276-1277（重试入队时也保留）
completedWaitpoints,
// 直接传递给新 snapshot
```

---

#### 3.3.3 对重试执行的实际影响

| 场景 | blockingWaitpoints | completedWaitpoints | 行为结果 |
|------|-------------------|---------------------|---------|
| 首次执行 | 空 | 空 | 所有子任务正常执行 |
| 失败后重试（无 checkpoint） | 空（已删除） | 保留之前已完成的 | SDK 层根据 completedWaitpoints 判断是否跳过 |
| 失败后重试（有 checkpoint） | 保留 checkpoint 时的状态 | 保留 checkpoint 时已完成的 | 从断点继续执行 |
| Replay | 空 | 空（新 run） | 所有子任务重新执行 |

---

> **❌ 之前的错误结论**：认为重试时所有 waitpoints 都会被保留
> 
> **✅ 正确结论（代码验证版）**：
> 1. **Blocking waitpoints**（`taskRunWaitpoint` 表）：失败后被 **全部清除**，新 attempt 不会等待之前未完成的子任务
> 2. **Completed waitpoints**（snapshot 关联）：通过 snapshot 链 **完整保留**，新 attempt 可以知道哪些子任务已完成
> 3. **关键影响**：阻塞关联的删除意味着失败后不会等待之前发起的子任务，但子任务本身可能仍在执行（孤儿任务风险）

### 3.4 Checkpoint（检查点）恢复机制（✅ 双分支完整链路）

**代码位置**：`internal-packages/run-engine/src/engine/systems/checkpointSystem.ts:36-249`

Checkpoint 是一种特殊的状态保存机制，用于在长时间运行任务中保存进程内存状态，支持从断点恢复而非从头开始。

#### 3.4.1 Checkpoint 创建后的两条分支

```
执行中触发 checkpoint（仅在可 checkpoint 状态：RUN_CREATED/QUEUED/EXECUTING/EXECUTING_WITH_WAITPOINTS/QUEUED_EXECUTING）
    ↓
创建 TaskRunCheckpoint 记录：
- type: checkpoint 类型
- location: 快照存储位置
- imageRef: 虚拟机镜像引用
- reason: 触发原因（如 OOM 预警）
    ↓
更新 run 状态为 WAITING_TO_RESUME
    ↓
┌────────────────────────────────────────────────────────────────────────┐
│ 分支判断：snapshot.executionStatus === "QUEUED_EXECUTING" ?            │
└────────────────────────────────────────────────────────────────────────┘
                          │
          ┌───────────────┴───────────────┐
          ▼                               ▼
┌───────────────────────┐     ┌──────────────────────────────┐
│ 分支 A：QUEUED_EXECUTING │     │ 分支 B：其他状态（默认）      │
│ → 重新入队路径         │     │ → SUSPENDED 路径             │
└───────────────────────┘     └──────────────────────────────┘
```

---

#### ✅ 分支 A：QUEUED_EXECUTING → 重新入队路径

**代码位置**：`checkpointSystem.ts:175-208`

```typescript
if (snapshot.executionStatus === "QUEUED_EXECUTING") {
  // Enqueue the run again
  const newSnapshot = await this.enqueueSystem.enqueueRun({
    run,
    env: run.runtimeEnvironment,
    snapshot: {
      status: "QUEUED",  // ← 状态变为 QUEUED
      description: "Run was QUEUED, because it was queued and executing and a checkpoint was created",
      metadata: snapshot.metadata,
    },
    previousSnapshotId: snapshot.id,
    batchId: snapshot.batchId ?? undefined,
    completedWaitpoints: snapshot.completedWaitpoints.map((waitpoint) => ({
      id: waitpoint.id,
      index: waitpoint.index,
    })),
    checkpointId: taskRunCheckpoint.id,  // ← checkpointId 保留
  });
  // ...
}
```

**恢复流程**：
1. 状态变为 `QUEUED`（可 dequeue 状态）
2. 通过 `enqueueRun` 进入正常队列流程
3. Worker 正常 dequeue 后从 checkpoint 恢复执行
4. `completedWaitpoints` 完整保留

---

#### ✅ 分支 B：其他状态 → SUSPENDED 路径

**代码位置**：`checkpointSystem.ts:209-247`

```typescript
// create a new execution snapshot, with the checkpoint
const newSnapshot = await this.executionSnapshotSystem.createExecutionSnapshot(prisma, {
  run,
  snapshot: {
    executionStatus: "SUSPENDED",  // ← 状态变为 SUSPENDED
    description: "Run was suspended after creating a checkpoint.",
    metadata: snapshot.metadata,
  },
  previousSnapshotId: snapshot.id,
  batchId: snapshot.batchId ?? undefined,
  completedWaitpoints: snapshot.completedWaitpoints.map((waitpoint) => ({
    id: waitpoint.id,
    index: waitpoint.index,
  })),
  checkpointId: taskRunCheckpoint.id,  // ← checkpointId 保留
  // ...
});
```

**恢复流程**：
```
SUSPENDED 状态（不可直接 dequeue）
    ↓
Worker 从 checkpoint 镜像恢复进程内存（checkpointClient.restoreRun）
    ↓
调用 API：POST /engine/v1/worker-actions/runs/:runId/snapshots/:snapshotId/continue
    ↓
continueRunExecution() 验证：
  - 检查 snapshotId 是否匹配最新
  - 检查状态是否为 PENDING_EXECUTING（需要先 dequeue）
    ↓
状态变为 EXECUTING
    ↓
completedWaitpoints 保留，继续执行
```

---

#### 两条分支对比表

| 维度 | 分支 A：QUEUED_EXECUTING | 分支 B：SUSPENDED |
|------|------------------------|------------------|
| **触发条件** | checkpoint 时状态为 QUEUED_EXECUTING | checkpoint 时为其他可 checkpoint 状态 |
| **新状态** | QUEUED | SUSPENDED |
| **是否可直接 dequeue** | ✅ 是（QUEUED 在 dequeueable 列表） | ❌ 否（SUSPENDED 不在 dequeueable 列表） |
| **恢复方式** | 正常出队流程 | Worker 恢复镜像后调用 continueRunExecution API |
| **checkpointId 保留** | ✅ 是 | ✅ 是 |
| **completedWaitpoints 保留** | ✅ 是 | ✅ 是 |
| **调用系统** | enqueueSystem.enqueueRun() | executionSnapshotSystem.createExecutionSnapshot() |
| **代码位置** | `checkpointSystem.ts:175-208` | `checkpointSystem.ts:209-247` |

---

**✅ Checkpoint 状态边界验证**：
```typescript
// checkpointSystem.ts:91-114
if (!isCheckpointable(snapshot.executionStatus)) {
  return {
    ok: false as const,
    error: `Status ${snapshot.executionStatus} is not checkpointable`,
  };
}

// 可 checkpoint 状态：statuses.ts:21-32
// RUN_CREATED / QUEUED / EXECUTING / EXECUTING_WITH_WAITPOINTS / QUEUED_EXECUTING
```

### 3.5 Replay 状态重建流程

**代码位置**：`apps/webapp/app/v3/services/replayTaskRun.server.ts:24-144`

```
手工触发 replay → ReplayTaskRunService.call()
    ↓
┌────────────────────────────────────────────┐
│ ✅ 唯一检查：环境是否归档                    │
│   if (authenticatedEnvironment.archivedAt) {│
│     throw new Error("Can't replay...");     │
│   }                                        │
└────────────────────────────────────────────┘
    ↓
从现有 run 提取配置：
- payload（可覆盖）
- metadata（可覆盖）
- tags（可覆盖）
- queue、region、machine 等配置
    ↓
调用 TriggerTaskService 创建全新 run
    ↓
设置关联字段：
- replayedFromTaskRunFriendlyId = 源 run.friendlyId
- parentAsLinkType = "replay"（追踪链路）
- traceContext: 00-{traceId}-{spanId}-01
    ↓
新 run 状态机从 RUN_CREATED 开始完整执行
    ↓
所有子任务、外部副作用全部重新执行
```

---

## 四、输入参数、上下文、子任务、副作用重新装配

### 4.1 输入参数（Payload）

#### Replay 场景（`replayTaskRun.server.ts:52-170`）
```typescript
private async overrideExistingPayloadPacket(
  existingTaskRun: TaskRun,
  stringifiedPayloadOverride: string | undefined
) {
  // 1. application/store 类型：直接使用原始数据（不可修改）
  if (existingTaskRun.payloadType === "application/store") {
    return conditionallyImportPacket({...});
  }

  // 2. application/super+json 类型：支持合并覆盖
  if (stringifiedPayloadOverride && existingTaskRun.payloadType === "application/super+json") {
    const newPayload = await replaceSuperJsonPayload(
      existingTaskRun.payload,
      stringifiedPayloadOverride
    );
    return stringifyIO(newPayload);
  }

  // 3. 其他类型：直接使用覆盖值或原始值
  return conditionallyImportPacket({
    data: stringifiedPayloadOverride ?? existingTaskRun.payload,
    dataType: existingTaskRun.payloadType,
  });
}
```

#### 自动 Retry 场景
- **payload 完全复用**：不做任何修改
- 解析发生在每次 attempt 开始时（`taskExecutor.ts:118-123`）
- 解析结果在同一次 attempt 内被缓存使用

### 4.2 执行上下文（Context）

**代码位置**：`internal-packages/run-engine/src/engine/systems/runAttemptSystem.ts:177-299`

```typescript
public async resolveTaskRunContext(runId: string): Promise<TaskRunContext> {
  // 从数据库读取 run 完整信息
  const run = await this.$.readOnlyPrisma.taskRun.findFirst(...);

  return {
    run: {
      id: run.friendlyId,
      tags: run.runTags,
      isTest: run.isTest,
      isReplay: !!run.replayedFromTaskRunFriendlyId, // replay 标记
      createdAt: run.createdAt,
      startedAt: run.startedAt ?? run.createdAt,
      maxAttempts: run.maxAttempts ?? undefined,
      version: run.taskVersion ?? "unknown",
      // ...
    },
    attempt: {
      number: run.attemptNumber ?? 1,
      startedAt: run.startedAt ?? new Date(),
    },
    // ...
  };
}
```

**上下文装配差异**：
| 场景 | isReplay 标记 | attempt.number | startedAt |
|------|--------------|----------------|-----------|
| 首次执行 | false | 1 | 当前时间 |
| 自动 Retry | false | 递增（2,3...） | 首次执行时间 |
| Replay | true | 1 | 当前时间（新 run） |

### 4.3 子任务（Waitpoints）装配

**代码位置**：`internal-packages/run-engine/src/engine/systems/executionSnapshotSystem.ts:44-116`

**装配策略（✅ 经过验证）**：

| 场景 | completedWaitpoints（通过 snapshot） | blockingWaitpoints（taskRunWaitpoint 表） | 子任务执行行为 |
|------|-------------------------------------|----------------------------------------|---------------|
| 首次执行 | 空数组 | 空 | 所有子任务全部执行 |
| 自动 Retry | 保留之前已完成的 | ❌ 已被清除 | 已完成的子任务不会重新等待，但代码可能重新执行 |
| 自动 Retry（有 checkpoint） | 保留 checkpoint 时已完成的 | 保留 checkpoint 时的阻塞状态 | 从未完成处继续执行 |
| Replay | 空数组（新 run） | 空 | 所有子任务重新执行 |

### 4.4 外部副作用重新执行

#### Session Streams（会话流）管理
**代码位置**：`packages/core/src/v3/sessionStreams/manager.ts:43-577`

Session Stream 管理实时输入流（如 chat.agent 的用户输入），在重试时需要避免重复处理已处理的消息。

#### 生命周期钩子（Lifecycle Hooks）
**代码位置**：`packages/core/src/v3/workers/taskExecutor.ts:150-156`

**钩子执行时机**：
| 钩子 | 首次执行 | 自动 Retry | Replay |
|------|---------|-----------|--------|
| `init()` | 执行 | 每次 attempt 都执行 | 执行 |
| `onStart()` | 执行（仅 attempt 1） | 不执行（attempt > 1） | 执行（新 run 的 attempt 1） |
| `onStartAttempt()` | 执行 | 每次 attempt 都执行 | 执行 |
| `onWait()` | 等待子任务时 | 等待子任务时 | 等待子任务时 |
| `onResume()` | 子任务完成时 | 子任务完成时 | 子任务完成时 |
| `onSuccess/Failure()` | 完成时 | 最终完成时 | 完成时 |
| `cleanup()` | 完成时 | 最终完成时 | 完成时 |

#### Idempotency Key（幂等键）

**Replay 场景**（`replayTaskRun.server.ts:98-99`）：
```typescript
idempotencyKey: overrideOptions.idempotencyKey,
// 默认不传递源 run 的幂等键，允许手工指定新的幂等键
```

**自动 Retry 场景**：复用原有 run 的幂等键配置，不做修改

---

## 五、历史 Run 与新 Run 在产品视图与计费上的关系

### 5.1 数据库关联字段

**Schema 定义**：
```prisma
model TaskRun {
  // ...
  replayedFromTaskRunFriendlyId String?  // 源 run 的 friendlyId
  // ...
}
```

### 5.2 产品视图关联

#### 追踪链路关联（`replayTaskRun.server.ts:117-123`）
```typescript
{
  spanParentAsLink: true,              // 不作为父子 span，而是 link
  parentAsLinkType: "replay",         // link 类型标记
  replayedFromTaskRunFriendlyId: existingTaskRun.friendlyId,
  traceContext: {
    traceparent: `00-${existingTaskRun.traceId}-${existingTaskRun.spanId}-01`,
  }
}
```

#### 上下文标记（`runAttemptSystem.ts:267`）
```typescript
run: {
  // ...
  isReplay: !!run.replayedFromTaskRunFriendlyId,  // UI 可根据此标记显示 "Replay" 标签
  // ...
}
```

### 5.3 计费模型

#### 核心计费字段
```prisma
model TaskRun {
  // ...
  usageDurationMs Int   @default(0)  // 累计使用时长（毫秒）
  costInCents     Float @default(0)  // 累计费用（分）
  baseCostInCents Float @default(0)  // 基础费用（分）
  // ...
}
```

#### 费用计算逻辑（`runAttemptSystem.ts:2047-2096`）
```typescript
#calculateUpdatedUsage({
  runId,
  currentUsageDurationMs,  // 当前累计时长
  currentCostInCents,      // 当前累计费用
  attemptDurationMs,       // 本次 attempt 时长
  machinePresetName,       // 机器规格
  environmentType,         // 环境类型
}: {
  usageDurationMs: number;
  costInCents: number;
}) {
  // 1. 时长累加（所有环境都累加）
  let usageDurationMs = currentUsageDurationMs + attemptDurationMs;

  // 2. 溢出保护（PostgreSQL int4 max）
  if (usageDurationMs > RunAttemptSystem.MAX_INT4) {
    usageDurationMs = RunAttemptSystem.MAX_INT4;
  }

  // 3. 费用计算（仅非开发环境）
  let costInCents = currentCostInCents;
  if (environmentType !== "DEVELOPMENT") {
    const machinePreset = machinePresetFromName(...);
    costInCents = currentCostInCents + attemptDurationMs * machinePreset.centsPerMs;
  }

  return { usageDurationMs, costInCents };
}
```

#### 计费触发时机

| 时机 | 调用位置 | 说明 |
|------|---------|------|
| attempt 成功 | `runAttemptSystem.ts:732-739` | 完成时累加 |
| attempt 失败（重试） | `runAttemptSystem.ts:989-996` | 失败时也累加已消耗资源 |
| attempt 失败（最终失败） | `runAttemptSystem.ts:1601-1608` | 最终失败时累加 |
| run 被取消 | `runAttemptSystem.ts:1387-1394` | 取消时累加已消耗资源 |

#### 自动 Retry 计费特点

1. **累计计费**：所有 attempt 的费用都累计到同一个 run 上
2. **OOM 自动升级机器**：检测到 OOM 错误时，自动升级到更大规格机器，费用仍然累计到同一个 run
3. **基础费用**：每次 run 出队时设置一次，不论 retry 多少次只收一次基础费

#### Replay 计费特点

1. **独立计费**：replay 创建的新 run 完全独立计费
2. **源 run 费用不受影响**：replay 操作不会修改源 run 的任何计费字段
3. **可选择不同机器规格**：可指定与源 run 不同的机器规格，费率按新规格计算

#### 开发环境豁免
- 开发环境（`DEVELOPMENT`）的 run 不产生费用
- 包括开发环境下的自动 retry 和 replay 都免费
- 但 `usageDurationMs` 仍然会累加，用于统计分析

---

## 六、事故后手工 Replay 失败原因排查指南（✅ 验证版）

基于以上代码分析，手工 replay 失败可能的原因包括：

### 6.1 权限与配置问题

1. **环境已归档**（`replayTaskRun.server.ts:33-35`）
   ```typescript
   if (authenticatedEnvironment.archivedAt) {
     throw new Error("Can't replay a run on an archived environment");
   }
   ```
   - ✅ **唯一明确检查的失败条件**

2. **权限不足**（`triggerTask.server.ts:162-169`）
   - entitlement 校验失败，用户没有该环境的触发权限
   - 队列大小限制（`QUEUED` 状态 run 超过环境限制）

3. **幂等键冲突**（`triggerTask.server.ts:575-583`）
   ```typescript
   if (error instanceof RunDuplicateIdempotencyKeyError) {
     // 自动重试一次，第二次会返回已存在的 run
     return await this.call({ ..., attempt: attempt + 1 });
   }
   ```

### 6.2 数据一致性问题

1. **源 run 数据不完整**
   - payload 解析失败
   - metadata 格式损坏
   - 关联的 background worker 已被删除

2. **task 版本不匹配**（`dequeueSystem.ts:311-330`）
   - 任务在当前部署版本中不存在，进入 pending version 等待

### 6.3 资源与配额问题

1. **计费配额不足**（`replayTaskRun.server.ts:134-136`）
   ```typescript
   if (error instanceof OutOfEntitlementError) {
     return; // 配额不足，静默失败
   }
   ```

2. **并发限制**
   - 环境级并发限制已满
   - 队列级并发限制已满

### 6.4 ✅ 容易被忽略的场景

| 场景 | 是否会 Replay 失败 | 说明 |
|------|-------------------|------|
| 源 run 正在执行中 | ❌ 不会 | Replay 不检查源 run 状态，执行中的 run 也可以 replay |
| 源 run 已取消 | ❌ 不会 | 已取消的 run 也可以 replay |
| 源 run 已过期 | ❌ 不会 | 已过期的 run 也可以 replay |
| 源环境已归档 | ✅ 会 | 这是唯一会导致失败的状态检查 |
| 源 run payload 损坏 | ✅ 可能 | 解析失败会导致 trigger 失败 |

### 6.5 排查步骤建议

1. **检查 replay 服务日志**：搜索 "Replaying task run" 和 "Failed to replay a run"
2. **检查 triggerTask 服务日志**：查看 entitlement 校验和任务触发细节
3. **检查环境状态**：确认 `environment.archivedAt` 为 `null`
4. **检查数据库记录**：确认源 run 的 payload、metadata、taskIdentifier 等字段完整
5. **检查计费状态**：确认组织账户状态正常，有足够配额

---

## 七、结论修正汇总（v4.0 口径统一版）

### 7.1 ❌ 错误结论 vs ✅ 正确结论（完整列表）

| # | 阶段 | 之前的错误结论 | 验证后的正确结论 | 核心证据 |
|---|------|--------------|----------------|----------|
| 1 | v2.0 | SUSPENDED 状态可以创建 checkpoint | SUSPENDED 状态**不能**创建 checkpoint | `checkpoints.test.ts:759` |
| 2 | v2.0 | PENDING_EXECUTING 状态可以创建 checkpoint | PENDING_EXECUTING 状态**不能**创建 checkpoint | `statuses.ts:21-32` |
| 3 | v2.0 | Replay 仅适用于已完成/失败的 run | Replay 适用于**任何状态**的 run，仅检查环境未归档 | `replayTaskRun.server.ts:33-35` |
| 4 | v2.0 | 重试时所有 waitpoints 都保留 | 重试时 `taskRunWaitpoint`（阻塞）被清除，只有 `snapshot.completedWaitpoints` 保留 | `runAttemptSystem.ts:898` + `dequeueSystem.ts:457` |
| 5 | v2.0 | FINISHED 状态的 run 会被 replay 拒绝 | FINISHED 状态的 run **可以**被 replay | Replay 服务无状态检查 |
| 6 | v3.0 | Checkpoint 后统一走 SUSPENDED 路径 | Checkpoint 后有**两条分支**：QUEUED_EXECUTING → 重新入队；其他状态 → SUSPENDED | `checkpointSystem.ts:175-247` |
| 7 | v3.0 | SUSPENDED 状态可正常 dequeue | SUSPENDED 状态**不可直接 dequeue**，需要 Worker 恢复镜像后调用 continue API | `statuses.ts:3-6` + `workerGroupTokenService.server.ts:505-520` |
| 8 | v3.0 | 重试后子任务不会继续执行 | 重试后 blocking waitpoint 被删除，但**子任务本身可能仍在执行（孤儿任务风险）** | `waitpointSystem.ts:53-68` |
| 9 | **v4.0** | 自动 retry「仅执行中失败后触发」 | **4 种状态可触发自动 retry**：PENDING_EXECUTING / EXECUTING / EXECUTING_WITH_WAITPOINTS / QUEUED_EXECUTING，唯一显式禁止 FINISHED | `runAttemptSystem.ts:891-893` + `retrying.ts:46-178` |

---

### 7.2 ✅ 自动 Retry 触发边界汇总（v4.0 新增 · 口径统一）

```
                          ┌─────────────────────────────────────────────────┐
                          │           attemptFailed() 入口                  │
                          └─────────────────────────────────────────────────┘
                                             │
                        ┌────────────────────┴────────────────────┐
                        ▼                                         ▼
            ┌────────────────────────┐             ┌──────────────────────────┐
            │ latestSnapshot.status  │             │   retryOutcomeFromCompletion  │
            │ === FINISHED ?         │             │   （重试决策逻辑）           │
            └────────────────────────┘             └──────────────────────────┘
                        │                                         │
          ┌─────────────┴─────────────┐                           │
          ▼                           ▼                           │
    ┌───────────┐              ┌─────────────┐                   │
    │ 抛出错误  │              │ 继续检查     │                   │
    └───────────┘              └─────────────┘                   │
                                          │                      │
                        ┌─────────────────┴──────────┐           │
                        ▼                            ▼           │
            ┌─────────────────────┐       ┌──────────────────┐  │
            │ startRunAttempt()    │       │  重试类型判断：   │  │
            │ 前置检查：            │       │  - OOM 升级机器  │  │
            │ isFinishedOrPending   │       │  - 错误可重试性  │  │
            │ → 排除 FINISHED 和    │       │  - 次数限制      │  │
            │   PENDING_CANCEL      │       │  - 配置存在性    │  │
            └─────────────────────┘       └──────────────────┘  │
                                                                  │
                        ┌─────────────────────────────────────────┘
                        ▼
            ┌─────────────────────────────────────┐
            │  最终只有以下状态可成功重试：         │
            │  1. PENDING_EXECUTING               │
            │  2. EXECUTING                       │
            │  3. EXECUTING_WITH_WAITPOINTS       │
            │  4. QUEUED_EXECUTING                │
            └─────────────────────────────────────┘
```

---

### 7.3 ✅ 四种可重试状态的典型失败场景（v4.0 新增）

| 状态 | 典型失败场景 | 触发方式 | 重试后状态 |
|------|-------------|---------|-----------|
| **PENDING_EXECUTING** | 1. Worker 出队后 OOM<br>2. 代码 import 失败<br>3. 初始化阶段异常<br>4. Worker 进程崩溃 | `attemptFailed()` | EXECUTING / QUEUED |
| **EXECUTING** | 1. 业务代码抛出异常<br>2. 运行时错误<br>3. Worker 超时<br>4. Worker 崩溃 | `attemptFailed()` | EXECUTING / QUEUED |
| **EXECUTING_WITH_WAITPOINTS** | 1. 等待子任务时 Worker 崩溃<br>2. 等待超时被强制失败<br>3. 检测到子任务失败 | `attemptFailed()` | EXECUTING / QUEUED |
| **QUEUED_EXECUTING** | 1. Checkpoint 恢复失败<br>2. 断点续跑时代码异常<br>3. Worker 崩溃 | `attemptFailed(forceRequeue=true)` | QUEUED（重新入队） |

---

### 7.4 ✅ Checkpoint 双分支机制总结（v3.0 新增）

```
                            ┌─────────────────────────────────────────────┐
                            │              Checkpoint 创建后               │
                            └─────────────────────────────────────────────┘
                                             │
                        ┌────────────────────┴────────────────────┐
                        ▼                                         ▼
            ┌────────────────────────┐             ┌──────────────────────────┐
            │ QUEUED_EXECUTING 状态 │             │ 其他状态（EXECUTING 等）  │
            └────────────────────────┘             └──────────────────────────┘
                        │                                         │
                        ▼                                         ▼
            ┌────────────────────────┐             ┌──────────────────────────┐
            │ 调用 enqueueRun()      │             │ 创建 SUSPENDED snapshot  │
            │ → 状态变为 QUEUED      │             │ → run 状态 WAITING_TO_RESUME │
            │ → 可正常 dequeue       │             │ → 不可直接 dequeue        │
            └────────────────────────┘             └──────────────────────────┘
                        │                                         │
                        ▼                                         ▼
            ┌────────────────────────┐             ┌──────────────────────────┐
            │ Worker 正常出队执行     │             │ Worker 恢复镜像后调用     │
            │ 从 checkpoint 恢复      │             │ continueRunExecution API │
            └────────────────────────┘             └──────────────────────────┘
```

---

### 7.5 关键边界条件总结（v4.0 口径统一版）

| 维度 | 边界条件 | 代码证据 |
|------|---------|----------|
| **Checkpoint 创建** | 仅 5 种状态可创建：RUN_CREATED/QUEUED/EXECUTING/EXECUTING_WITH_WAITPOINTS/QUEUED_EXECUTING | `statuses.ts:21-32` |
| **Checkpoint 分支** | 两条分支：QUEUED_EXECUTING → 重新入队；其他 → SUSPENDED | `checkpointSystem.ts:175-247` |
| **Dequeue 状态** | 仅 QUEUED/QUEUED_EXECUTING 可 dequeue | `statuses.ts:3-6` |
| **自动 Retry 触发** | **4 种状态可触发**：PENDING_EXECUTING / EXECUTING / EXECUTING_WITH_WAITPOINTS / QUEUED_EXECUTING，唯一显式禁止 FINISHED | `runAttemptSystem.ts:891-893` + `retrying.ts:46-178` |
| **Replay 触发** | **无状态限制**，仅检查环境未归档 | `replayTaskRun.server.ts:33-35` |
| **Waitpoint 清理** | blocking 关联清除，completed 历史保留 | `waitpointSystem.ts:53-68` + `dequeueSystem.ts:457-458` |
| **最终状态** | 8 种 final 状态不再自动 retry，但仍可 replay | `statuses.ts:44-57` |
| **孤儿任务风险** | 重试后子任务可能仍在执行，需注意幂等性 | `waitpointSystem.ts:61-65` |



## 八、关键代码文件索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| Replay 主逻辑 | `apps/webapp/app/v3/services/replayTaskRun.server.ts` | 24-144 |
| 自动 Retry 决策 | `internal-packages/run-engine/src/engine/retrying.ts` | 46-178 |
| Run Attempt 管理 | `internal-packages/run-engine/src/engine/systems/runAttemptSystem.ts` | 110-2096 |
| 执行快照系统 | `internal-packages/run-engine/src/engine/systems/executionSnapshotSystem.ts` | 320-431 |
| Checkpoint 系统 | `internal-packages/run-engine/src/engine/systems/checkpointSystem.ts` | 21-366 |
| 出队与状态重建 | `internal-packages/run-engine/src/engine/systems/dequeueSystem.ts` | 88-450 |
| Waitpoint 系统 | `internal-packages/run-engine/src/engine/systems/waitpointSystem.ts` | 42-150 |
| **状态枚举定义** | `internal-packages/run-engine/src/engine/statuses.ts` | **1-61** |
| 计费计算 | `internal-packages/run-engine/src/engine/systems/runAttemptSystem.ts` | 2047-2096 |
| 机器费率 | `internal-packages/run-engine/src/engine/machinePresets.ts` | 54-61 |
| Checkpoint 测试用例 | `internal-packages/run-engine/src/engine/tests/checkpoints.test.ts` | 632-776 |
