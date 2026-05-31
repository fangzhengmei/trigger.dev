# Run Replay 与自动 Retry 状态机重建分析（代码验证版）

> **⚠️ 重要提示**：本文档经过源码验证，修正了部分之前的错误结论。标记为 ❌ 的是之前的错误理解，✅ 是验证后的正确结论。

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

### 2.2 自动 Retry 可触发状态矩阵

**✅ 验证代码**：`internal-packages/run-engine/src/engine/systems/runAttemptSystem.ts:891-893`

| ExecutionStatus | 是否可自动 Retry | 说明 |
|----------------|-----------------|------|
| `RUN_CREATED` | ❌ 否 | 尚未执行 |
| `DELAYED` | ❌ 否 | 延迟中 |
| `QUEUED` | ❌ 否 | 队列中 |
| `PENDING_EXECUTING` | ✅ 是 | 出队后执行失败 |
| `EXECUTING` | ✅ 是 | 执行中失败 |
| `EXECUTING_WITH_WAITPOINTS` | ✅ 是 | 等待子任务时失败 |
| `QUEUED_EXECUTING` | ✅ 是 | checkpoint 后重试失败 |
| `SUSPENDED` | ❌ 否 | 已暂停（需 resume 而非 retry） |
| `PENDING_CANCEL` | ❌ 否 | 待取消 |
| `FINISHED` | ❌ 否 | 已完成后不可自动 retry |

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

### 3.3 ✅ Waitpoint 重试处理机制（重要修正）

**代码位置**：
- `runAttemptSystem.ts:897-901` - 失败时清除 blocking waitpoints
- `waitpointSystem.ts:53-68` - `clearBlockingWaitpoints` 实现
- `dequeueSystem.ts:457-458` - 新 snapshot 保留 completedWaitpoints

**两种 Waitpoint 的区别**：

| 类型 | 存储位置 | 重试时处理 | 说明 |
|------|---------|-----------|------|
| **Blocking Waitpoints** | `taskRunWaitpoint` 表 | ❌ 全部删除 | 当前正在等待的子任务关联 |
| **Completed Waitpoints** | `snapshot.completedWaitpoints` | ✅ 保留并传递 | 历史已完成的子任务引用 |

```typescript
// 失败时清除阻塞 waitpoints
const deletedCount = await this.waitpointSystem.clearBlockingWaitpoints({ runId, tx });
// 清除 taskRunWaitpoint 表中该 run 的所有记录

// 出队时保留 completedWaitpoints
completedWaitpoints: {
  connect: snapshot.completedWaitpoints.map((w) => ({ id: w.id })),
}
// 通过 connect 关联已有的 waitpoint 记录
```

> **❌ 之前的错误结论**：认为重试时所有 waitpoints 都会被保留
> 
> **✅ 正确结论**：
> - `taskRunWaitpoint`（阻塞关联）：失败后被清除，新 attempt 不会等待之前的未完成子任务
> - `snapshot.completedWaitpoints`（历史记录）：通过 snapshot 保留，新 attempt 知道哪些子任务已完成

### 3.4 Checkpoint（检查点）恢复机制

**代码位置**：`internal-packages/run-engine/src/engine/systems/checkpointSystem.ts:21-366`

Checkpoint 是一种特殊的状态保存机制，用于在长时间运行任务中保存进程内存状态，支持从断点恢复而非从头开始。

```
执行中触发 checkpoint（仅在可 checkpoint 状态）
    ↓
创建 TaskRunCheckpoint 记录：
- type: checkpoint 类型
- location: 快照存储位置
- imageRef: 虚拟机镜像引用
- reason: 触发原因（如 OOM 预警）
    ↓
更新 run 状态为 WAITING_TO_RESUME
    ↓
创建 SUSPENDED 状态的 ExecutionSnapshot
    ↓
释放并发资源
    ↓
┌──────────────────────────────────────────────┐
│ 恢复流程：                                    │
│ worker 拉取 → 检测到 snapshot.checkpointId   │
│ → 从 imageRef 恢复进程内存                    │
│ → continueRunExecution() → 状态变为 EXECUTING │
│ → completedWaitpoints 保留，继续执行          │
└──────────────────────────────────────────────┘
```

**✅ Checkpoint 状态边界验证**：
```typescript
// checkpointSystem.ts:91-114
if (!isCheckpointable(snapshot.executionStatus)) {
  return {
    ok: false as const,
    error: `Status ${snapshot.executionStatus} is not checkpointable`,
  };
}
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

## 七、结论修正汇总

### 7.1 ❌ 错误结论 vs ✅ 正确结论

| # | 之前的错误结论 | 验证后的正确结论 | 证据 |
|---|--------------|----------------|------|
| 1 | SUSPENDED 状态可以创建 checkpoint | SUSPENDED 状态**不能**创建 checkpoint | `checkpoints.test.ts:759` |
| 2 | PENDING_EXECUTING 状态可以创建 checkpoint | PENDING_EXECUTING 状态**不能**创建 checkpoint | `statuses.ts:21-32` |
| 3 | Replay 仅适用于已完成/失败的 run | Replay 适用于**任何状态**的 run，仅检查环境未归档 | `replayTaskRun.server.ts:33-35` |
| 4 | 重试时所有 waitpoints 都保留 | 重试时 `taskRunWaitpoint`（阻塞）被清除，只有 `snapshot.completedWaitpoints` 保留 | `runAttemptSystem.ts:898` + `dequeueSystem.ts:457` |
| 5 | FINISHED 状态的 run 会被 replay 拒绝 | FINISHED 状态的 run **可以**被 replay | Replay 服务无状态检查 |

### 7.2 关键边界条件总结

1. **Checkpoint 创建边界**：仅 5 种状态可创建
2. **自动 Retry 触发边界**：仅执行中失败后触发
3. **Replay 触发边界**：**无状态限制**，仅检查环境未归档
4. **Waitpoint 清理边界**：阻塞关联清除，历史记录保留
5. **最终状态边界**：8 种 final 状态不再自动 retry，但仍可 replay

---

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
