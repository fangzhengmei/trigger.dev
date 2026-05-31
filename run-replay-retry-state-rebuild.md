# Run Replay 与自动 Retry 状态机重建分析

## 一、核心概念差异

### 1.1 Replay（手工重放）

**定义**：对已完成（成功/失败）的历史 run 进行手工触发的重新执行，创建一个全新的 run 实例。

**核心代码入口**：`apps/webapp/app/v3/services/replayTaskRun.server.ts:24-144`

**关键特性**：
- 创建全新的 `TaskRun` 数据库记录，拥有独立的 `id` 和 `friendlyId`
- 通过 `replayedFromTaskRunFriendlyId` 字段记录源 run 的关联关系
- 输入参数可选择性覆盖（payload、metadata、tags 等）
- 支持跨环境重放（如从生产环境重放到开发环境）
- 追踪链路上通过 `parentAsLinkType: "replay"` 建立关联而非父子关系

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
| **输入参数** | 可覆盖修改 | 完全复用 |
| **执行上下文** | 重建 | 重建（但复用部分状态） |
| **子任务/副作用** | 全部重新执行 | 从 checkpoint 恢复或全部重试 |
| **计费** | 新 run 独立计费 | 累计到同一 run |
| **最大次数限制** | 无限制（手工操作） | 受 `maxAttempts` 限制 |
| **追踪链路** | Link 关联 | 同一 trace 下的不同 span |

---

## 二、状态机从断点/起点重建过程

### 2.1 核心数据结构：ExecutionSnapshot（执行快照）

状态机的重建完全基于 `TaskRunExecutionSnapshot` 表，每个快照代表 run 在某个时间点的执行状态。

**核心表结构**（`internal-packages/database/prisma/schema.prisma`）：
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

**状态枚举**（`internal-packages/run-engine/src/engine/statuses.ts:1-61`）：
- `RUN_CREATED` - run 已创建
- `DELAYED` - 延迟执行中
- `QUEUED` - 队列中等待
- `PENDING_EXECUTING` - 出队后等待执行
- `EXECUTING` - 执行中
- `EXECUTING_WITH_WAITPOINTS` - 执行中但等待子任务
- `SUSPENDED` - 已暂停（checkpoint 后）
- `PENDING_CANCEL` - 待取消
- `FINISHED` - 已完成

### 2.2 自动 Retry 状态重建流程

**代码位置**：`internal-packages/run-engine/src/engine/systems/runAttemptSystem.ts:861-1142`

```
失败发生 → retryOutcomeFromCompletion() → 判断重试策略
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
从 latestSnapshot 重建状态：
- completedWaitpoints（已完成子任务）
- checkpoint（进程内存快照）
- 已累积的使用时长和费用
```

### 2.3 Replay 状态重建流程

**代码位置**：`apps/webapp/app/v3/services/replayTaskRun.server.ts:24-144`

```
手工触发 replay → ReplayTaskRunService.call()
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

### 2.4 Checkpoint（检查点）恢复机制

**代码位置**：`internal-packages/run-engine/src/engine/systems/checkpointSystem.ts:21-366`

Checkpoint 是一种特殊的状态保存机制，用于在长时间运行任务中保存进程内存状态，支持从断点恢复而非从头开始。

```
执行中触发 checkpoint → createCheckpoint()
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
│ worker 拉取 → 检测到 checkpointId            │
│ → 从 imageRef 恢复进程内存                    │
│ → continueRunExecution() → 状态变为 EXECUTING │
│ → 从断点继续执行，已完成的 waitpoints 保留    │
└──────────────────────────────────────────────┘
```

---

## 三、输入参数、上下文、子任务、副作用重新装配

### 3.1 输入参数（Payload）

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

### 3.2 执行上下文（Context）

**代码位置**：`internal-packages/run-engine/src/engine/systems/runAttemptSystem.ts:177-299`

```typescript
public async resolveTaskRunContext(runId: string): Promise<TaskRunContext> {
  // 从数据库读取 run 完整信息
  const run = await this.$.readOnlyPrisma.taskRun.findFirst(...);

  // 并行解析关联实体
  const [task, queue, organization, project, machinePreset, deployment] = 
    await Promise.all([
      this.#resolveTaskRunExecutionTask(...),  // 任务元数据
      this.#resolveTaskRunExecutionQueue(...), // 队列配置
      this.#resolveTaskRunExecutionOrganization(...), // 组织信息
      this.#resolveTaskRunExecutionProjectByRuntimeEnvironmentId(...), // 项目信息
      this.#resolveTaskRunExecutionMachinePreset(...), // 机器规格
      this.#resolveTaskRunExecutionDeployment(...), // 部署版本
    ]);

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
    task, queue, organization, project, machine, deployment,
    environment: { /* 环境信息 */ },
    batch: run.batchId ? { /* 批处理信息 */ } : undefined,
  };
}
```

**上下文装配差异**：
| 场景 | isReplay 标记 | attempt.number | startedAt |
|------|--------------|----------------|-----------|
| 首次执行 | false | 1 | 当前时间 |
| 自动 Retry | false | 递增（2,3...） | 首次执行时间 |
| Replay | true | 1 | 当前时间（新 run） |

### 3.3 子任务（Waitpoints）装配

**代码位置**：`internal-packages/run-engine/src/engine/systems/executionSnapshotSystem.ts:44-116`

Waitpoint 用于管理子任务依赖关系，每个 ExecutionSnapshot 会关联已完成的 waitpoints。

```typescript
function enhanceExecutionSnapshotWithWaitpoints(
  snapshot: ExecutionSnapshotWithCheckpoint,
  waitpoints: Waitpoint[],
  completedWaitpointOrder: string[]
): EnhancedExecutionSnapshot {
  return {
    ...snapshot,
    friendlyId: SnapshotId.toFriendlyId(snapshot.id),
    runFriendlyId: RunId.toFriendlyId(snapshot.runId),
    completedWaitpoints: waitpoints.flatMap((w) => {
      // 同一个 waitpoint 可能在 batch 中出现多次
      let indexes: (number | undefined)[] = [];
      for (let i = 0; i < completedWaitpointOrder.length; i++) {
        if (completedWaitpointOrder[i] === w.id) {
          indexes.push(i);
        }
      }
      // ...
      return indexes.map((index) => ({
        id: w.id,
        index: index === -1 ? undefined : index,
        friendlyId: w.friendlyId,
        type: w.type,
        completedAt: w.completedAt ?? new Date(),
        idempotencyKey: w.userProvidedIdempotencyKey ? w.idempotencyKey : undefined,
        completedByTaskRun: w.completedByTaskRunId ? { /* 子任务信息 */ } : undefined,
        output: w.output ?? undefined,    // 子任务输出
        outputType: w.outputType,
        outputIsError: w.outputIsError,  // 子任务是否失败
      }));
    }),
  };
}
```

**装配策略**：

| 场景 | completedWaitpoints | 子任务执行行为 |
|------|---------------------|---------------|
| 首次执行 | 空数组 | 所有子任务全部执行 |
| 自动 Retry（无 checkpoint） | 空数组 | 所有子任务重新执行 |
| 自动 Retry（有 checkpoint） | 保留 checkpoint 时已完成的 | 未完成的子任务继续执行 |
| Replay | 空数组（新 run） | 所有子任务重新执行 |

**重试时清理阻塞 waitpoints**（`runAttemptSystem.ts:897-901`）：
```typescript
// remove waitpoints blocking the run
const deletedCount = await this.waitpointSystem.clearBlockingWaitpoints({ runId, tx });
if (deletedCount > 0) {
  this.$.logger.debug("Cleared blocking waitpoints", { runId, deletedCount });
}
```

### 3.4 外部副作用重新执行

#### Session Streams（会话流）管理
**代码位置**：`packages/core/src/v3/sessionStreams/manager.ts:43-577`

Session Stream 管理实时输入流（如 chat.agent 的用户输入），在重试时需要避免重复处理已处理的消息。

```typescript
// 关键状态字段
private lastDispatchedSeqNums = new Map<string, number>(); // 已消费的消息序号
private minTimestamps = new Map<string, number>(); // 最小时间戳过滤

// OOM 重试时跳过已处理消息（manager.ts:450-453）
const minTs = this.minTimestamps.get(key);
if (minTs !== undefined && part.timestamp <= minTs) {
  return; // 跳过早于该时间戳的消息
}
```

#### 生命周期钩子（Lifecycle Hooks）
**代码位置**：`packages/core/src/v3/workers/taskExecutor.ts:150-156`

```typescript
// 注册钩子监听器
lifecycleHooks.registerOnWaitHookListener(async (wait) => {
  await this.#callOnWaitFunctions(wait, parsedPayload, ctx, initOutput, signal);
});

lifecycleHooks.registerOnResumeHookListener(async (wait) => {
  await this.#callOnResumeFunctions(wait, parsedPayload, ctx, initOutput, signal);
});
```

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

幂等键是防止重复执行的关键机制，在不同场景下的处理：

**Replay 场景**（`replayTaskRun.server.ts:98-99`）：
```typescript
idempotencyKey: overrideOptions.idempotencyKey,
// 默认不传递源 run 的幂等键，允许手工指定新的幂等键
```

**自动 Retry 场景**（`retrying.ts:97-109`）：
```typescript
// 复用原有 run 的幂等键配置，不做修改
const run = await prisma.taskRun.findFirst({
  select: {
    maxAttempts: true,
    lockedRetryConfig: true,
    // idempotencyKey 保留在 run 上，自动复用
  }
});
```

---

## 四、历史 Run 与新 Run 在产品视图与计费上的关系

### 4.1 数据库关联字段

**Schema 定义**（`schema.prisma:998-1001`）：
```prisma
model TaskRun {
  // ...
  replayedFromTaskRunFriendlyId String?  // 源 run 的 friendlyId
  // ...
}
```

**迁移记录**：`internal-packages/database/prisma/migrations/20250711124003_add_replayed_from_run_friendly_id/migration.sql`

### 4.2 产品视图关联

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

这种设计使得：
- 在追踪视图中，replay run 与源 run 是关联关系而非包含关系
- 可以通过 `replayedFromTaskRunFriendlyId` 反向查询所有从某个 run 重放出来的 run
- 保留完整的审计追溯能力

#### 上下文标记（`runAttemptSystem.ts:267`）
```typescript
run: {
  // ...
  isReplay: !!run.replayedFromTaskRunFriendlyId,  // UI 可根据此标记显示 "Replay" 标签
  // ...
}
```

### 4.3 计费模型

#### 核心计费字段（`schema.prisma:942-944`）
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
    // 费用 = 历史累计 + 本次时长 × 机器费率（分/毫秒）
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
   ```
   attempt 1: 1000ms × 0.0001分/ms = 0.1分
   attempt 2: 1000ms × 0.0001分/ms = 0.1分
   attempt 3: 1000ms × 0.0001分/ms = 0.1分
   ─────────────────────────────────────
   总计: 0.3分（记录在同一个 run 的 costInCents）
   ```

2. **OOM 自动升级机器**（`retrying.ts:58-81`）：
   - 检测到 OOM 错误时，自动升级到更大规格机器
   - 升级后的 attempt 使用新机器的 `centsPerMs` 费率
   - 费用仍然累计到同一个 run

3. **基础费用**（`dequeueSystem.ts:434`）：
   ```typescript
   baseCostInCents: this.options.machines.baseCostInCents,
   // 每次 run 出队时设置一次，不论 retry 多少次只收一次基础费
   ```

#### Replay 计费特点

1. **独立计费**：replay 创建的新 run 完全独立计费
   ```
   源 run: run_abc (costInCents: 0.3分)
   replay 后: run_def (costInCents: 0.1分)  // 独立计算，不累加到源 run
   ```

2. **源 run 费用不受影响**：replay 操作不会修改源 run 的任何计费字段

3. **可选择不同机器规格**（`replayTaskRun.server.ts:106-109`）：
   ```typescript
   machine:
     overrideOptions.machine ??
     (existingTaskRun.machinePreset as MachinePresetName) ??
     undefined,
   // 可指定与源 run 不同的机器规格，费率按新规格计算
   ```

#### 开发环境豁免

**代码位置**：`runAttemptSystem.ts:2076-2090`
```typescript
// Only calculate cost for non-dev environments
if (environmentType !== "DEVELOPMENT") {
  // 仅非开发环境计算费用
  costInCents = currentCostInCents + attemptDurationMs * machinePreset.centsPerMs;
}
```

这意味着：
- 开发环境（`DEVELOPMENT`）的 run 不产生费用
- 包括开发环境下的自动 retry 和 replay 都免费
- 但 `usageDurationMs` 仍然会累加，用于统计分析

### 4.4 计费汇总与报表

**事件总线通知**（`runAttemptSystem.ts:819-844`）：
```typescript
this.$.eventBus.emit("runSucceeded", {
  time: completedAt,
  run: {
    id: runId,
    status: run.status,
    usageDurationMs: run.usageDurationMs,  // 包含所有 attempt
    costInCents: run.costInCents,          // 包含所有 attempt
    // ...
  },
  // ...
});
```

这些事件被下游系统消费，用于：
- 生成账单
- 用量统计报表
- 成本分析
- 配额监控

---

## 五、事故后手工 Replay 失败原因排查指南

基于以上代码分析，手工 replay 失败可能的原因包括：

### 5.1 权限与配置问题

1. **环境已归档**（`replayTaskRun.server.ts:33-35`）
   ```typescript
   if (authenticatedEnvironment.archivedAt) {
     throw new Error("Can't replay a run on an archived environment");
   }
   ```

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

### 5.2 数据一致性问题

1. **源 run 数据不完整**
   - payload 解析失败（`taskExecutor.ts:125-146`）
   - metadata 格式损坏
   - 关联的 background worker 已被删除

2. **task 版本不匹配**（`dequeueSystem.ts:311-330`）
   ```typescript
   case "TASK_NOT_IN_LATEST":
   case "TASK_NEVER_REGISTERED":
     // 任务在当前部署版本中不存在，进入 pending version 等待
   ```

### 5.3 资源与配额问题

1. **计费配额不足**（`replayTaskRun.server.ts:134-136`）
   ```typescript
   if (error instanceof OutOfEntitlementError) {
     return; // 配额不足，静默失败
   }
   ```

2. **并发限制**
   - 环境级并发限制已满
   - 队列级并发限制已满

### 5.4 排查步骤建议

1. **检查 replay 服务日志**：搜索 "Replaying task run" 和 "Failed to replay a run"
2. **检查 triggerTask 服务日志**：查看 entitlement 校验和任务触发细节
3. **检查数据库记录**：确认源 run 的 payload、metadata、taskIdentifier 等字段完整
4. **检查环境状态**：确认目标环境未归档、有可用的 worker 部署
5. **检查计费状态**：确认组织账户状态正常，有足够配额

---

## 六、关键代码文件索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| Replay 主逻辑 | `apps/webapp/app/v3/services/replayTaskRun.server.ts` | 24-144 |
| 自动 Retry 决策 | `internal-packages/run-engine/src/engine/retrying.ts` | 46-178 |
| Run Attempt 管理 | `internal-packages/run-engine/src/engine/systems/runAttemptSystem.ts` | 110-2096 |
| 执行快照系统 | `internal-packages/run-engine/src/engine/systems/executionSnapshotSystem.ts` | 320-431 |
| Checkpoint 系统 | `internal-packages/run-engine/src/engine/systems/checkpointSystem.ts` | 21-366 |
| 出队与状态重建 | `internal-packages/run-engine/src/engine/systems/dequeueSystem.ts` | 88-450 |
| Waitpoint 系统 | `internal-packages/run-engine/src/engine/systems/waitpointSystem.ts` | 42-150 |
| 状态枚举 | `internal-packages/run-engine/src/engine/statuses.ts` | 1-61 |
| 计费计算 | `internal-packages/run-engine/src/engine/systems/runAttemptSystem.ts` | 2047-2096 |
| 机器费率 | `internal-packages/run-engine/src/engine/machinePresets.ts` | 54-61 |
| 任务执行器 | `packages/core/src/v3/workers/taskExecutor.ts` | 68-1440 |
| Session Stream 管理 | `packages/core/src/v3/sessionStreams/manager.ts` | 43-577 |
| Retry 配置 | `packages/core/src/v3/utils/retries.ts` | 29-44 |
| SDK 级 Retry | `packages/trigger-sdk/src/v3/retry.ts` | 30-115 |
| 用量管理器 | `packages/core/src/v3/usage/prodUsageManager.ts` | 11-142 |
