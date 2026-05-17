# 等待点恢复链路三个关键边界场景分析

## 边界一：外部信号完成 Token 时的鉴权与幂等短路

### 1.1 场景描述

当外部系统通过 HTTP API 调用 `POST /api/v1/waitpoints/tokens/:waitpointFriendlyId/complete` 完成一个 MANUAL 类型的等待点时，系统需要同时保证：
- 只有授权的调用者才能完成该等待点
- 重复调用不会产生副作用（幂等性）

### 1.2 鉴权机制

**API 路由层鉴权**（`apps/webapp/app/routes/api.v1.waitpoints.tokens.$waitpointFriendlyId.complete.ts`）：

```typescript
const { action, loader } = createActionApiRoute(
  {
    // ...
    allowJWT: true,
    authorization: {
      action: "write",
      resource: (params) => ({ type: "waitpoints", id: params.waitpointFriendlyId }),
    },
    corsStrategy: "all",
  },
  async ({ authentication, body, params }) => {
    // 1. 将友好ID转换为内部ID
    const waitpointId = WaitpointId.toId(params.waitpointFriendlyId);

    // 2. 二次校验：确保等待点属于当前认证环境
    const waitpoint = await $replica.waitpoint.findFirst({
      where: {
        id: waitpointId,
        environmentId: authentication.environment.id,  // 关键边界
      },
    });

    if (!waitpoint) {
      throw json({ error: "Waitpoint not found" }, { status: 404 });
    }
    // ...
  }
);
```

**两层鉴权边界**：

| 层级 | 校验点 | 作用 |
|------|--------|------|
| 框架层 | `authorization: { action: "write", resource: "waitpoints" }` | 基于 RBAC 的粗粒度权限控制 |
| 业务层 | `environmentId: authentication.environment.id` | 确保只能操作当前环境下的等待点，防止跨环境越权 |

### 1.3 幂等短路机制

**短路点 1：API 层前置检查**

```typescript
if (waitpoint.status === "COMPLETED") {
  return json<CompleteWaitpointTokenResponseBody>({ success: true });
}
```

- **作用**：对已完成的等待点直接返回成功，避免后续不必要的处理
- **时机**：在执行任何数据库写入操作之前
- **边界**：这是一个"尽力而为"的优化，不保证绝对幂等（并发场景下可能漏过）

**短路点 2：数据库层原子条件更新**

```typescript
// waitpointSystem.ts:84-94
const [updateError, updateResult] = await tryCatch(
  this.$.prisma.waitpoint.updateMany({
    where: { id, status: "PENDING" },  // 只更新 PENDING 状态的记录
    data: {
      status: "COMPLETED",
      completedAt: new Date(),
      // ...
    },
  })
);
```

- **关键设计**：使用 `updateMany` + `status: "PENDING"` 条件
- **原子性保证**：PostgreSQL 单条语句的原子性确保并发调用只会有一个成功
- **副作用控制**：
  - `updateResult.count === 0` 表示没有记录被更新（已被其他调用完成）
  - 此时记录 info 日志但不抛出错误，对外表现为成功

**短路点 3：Redis 作业去重**

```typescript
const jobId = `continueRunIfUnblocked:${run.taskRunId}`;
await this.$.worker.enqueue({
  id: jobId,  // 唯一键，自动去重
  job: "continueRunIfUnblocked",
  payload: { runId: run.taskRunId },
  availableAt: new Date(Date.now() + 50),
});
```

- 即使 `completeWaitpoint` 被多次调用，同一 runId 的恢复作业只会执行一次

### 1.4 幂等性层级总结

```
┌─────────────────────────────────────────────────────────────────┐
│  Level 1: API 层前置检查                                        │
│  - 检查 waitpoint.status === "COMPLETED"                        │
│  - 快速路径，无副作用                                           │
└───────────────────────────────────┬─────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Level 2: 数据库原子更新                                        │
│  - updateMany + status = 'PENDING' 条件                        │
│  - 数据库级原子性保证，并发安全                                 │
└───────────────────────────────────┬─────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Level 3: Redis 作业去重                                        │
│  - jobId = continueRunIfUnblocked:${runId}                      │
│  - 防止重复触发恢复流程                                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 边界二：continueRunIfUnblocked 在已排队/执行中/已结束状态下的分支语义

### 2.1 场景描述

`continueRunIfUnblocked` 函数在所有阻塞等待点完成后被调用，但此时任务可能已经处于各种非典型状态（如已被重新入队、正在执行、甚至已结束）。函数需要针对每种状态做出正确的语义决策。

### 2.2 状态分支全景图

```
continueRunIfUnblocked(runId)
        │
        ▼
    检查阻塞等待点 → 仍有未完成 → 返回 blocked
        │
        ▼ 全部完成
    获取最新快照 → switch (snapshot.executionStatus)
        │
        ├─ RUN_CREATED          → skipped ("run is already executing")
        ├─ DELAYED              → skipped ("run is delayed")
        ├─ QUEUED               → skipped ("run is already queued")
        ├─ PENDING_EXECUTING    → skipped ("run is already pending executing")
        ├─ QUEUED_EXECUTING     → skipped ("run is already queued executing")
        ├─ EXECUTING            → skipped ("run is already executing")
        ├─ PENDING_CANCEL       → skipped ("run is finished")
        ├─ FINISHED             → skipped ("run is finished")
        ├─ EXECUTING_WITH_WAITPOINTS → 转换为 EXECUTING + 发送通知
        └─ SUSPENDED            → 重新入队 QUEUED
```

### 2.3 各分支语义详解

#### 分支组 1："活跃进行中"状态（QUEUED / PENDING_EXECUTING / QUEUED_EXECUTING / EXECUTING）

**处理策略**：直接 `skipped`，不做任何操作

**设计考量**：
- 这些状态表明任务已经在正常流程中，等待点完成通知只是"迟到"的信号
- 任务会自然地从当前状态继续执行，不需要额外干预
- 如果强行操作（如再次入队）会导致重复执行或状态混乱

**典型时序场景**：
```
1. 任务阻塞于 Waitpoint A，状态 EXECUTING_WITH_WAITPOINTS
2. Waitpoint A 完成 → 调度 continueRunIfUnblocked (50ms 延迟)
3. 任务又遇到 Waitpoint B → 仍保持 EXECUTING_WITH_WAITPOINTS
4. Waitpoint B 完成 → 又调度一个 continueRunIfUnblocked
5. 第一个 continueRunIfUnblocked 执行，发现状态已变为 EXECUTING → skipped
6. 第二个 continueRunIfUnblocked 执行，同样 skipped
```

#### 分支组 2："已结束"状态（PENDING_CANCEL / FINISHED）

**处理策略**：直接 `skipped`，不做任何操作

**设计考量**：
- 任务已经到达终态，恢复没有意义
- 等待点完成是正常的生命周期事件，但任务已经不需要继续了
- `PENDING_CANCEL` 虽然不是严格的终态，但正在向终态过渡，应该让取消流程完成

#### 分支组 3："运行中遇到等待点"状态（EXECUTING_WITH_WAITPOINTS）

**处理策略**：
```typescript
case "EXECUTING_WITH_WAITPOINTS": {
  // 1. 创建新的 EXECUTING 状态快照
  const newSnapshot = await this.executionSnapshotSystem.createExecutionSnapshot(
    this.$.prisma,
    {
      run: { ... },
      snapshot: {
        executionStatus: "EXECUTING",
        description: "Run was continued, whilst still executing.",
      },
      // ...
      completedWaitpoints: blockingWaitpoints.map(...),
    }
  );

  // 2. 直接通知 Worker 继续执行
  await sendNotificationToWorker({
    runId,
    snapshot: newSnapshot,
    eventBus: this.$.eventBus,
  });

  break;
}
```

**关键语义**：
- **不重新入队**：任务仍在 Worker 上运行，只是暂时被等待点阻塞
- **保留执行上下文**：不需要从 Checkpoint 恢复
- **直接通知**：通过 `workerNotification` 事件让 Worker 立即继续执行
- **completedWaitpoints 传递**：将完成的等待点信息写入快照，供 Worker 端解析

#### 分支组 4："已挂起"状态（SUSPENDED）

**处理策略**：
```typescript
case "SUSPENDED": {
  // 安全检查：必须有 checkpoint
  if (!snapshot.checkpointId) {
    if (snapshot.runStatus === "CANCELED") {
      // 任务在挂起过程中被取消，正常跳过
      return { status: "skipped", reason: "run was canceled while suspended" };
    }
    throw new Error(`run is suspended, but has no checkpoint: ${runId}`);
  }

  // 通过 EnqueueSystem 重新入队
  const newSnapshot = await this.enqueueSystem.enqueueRun({
    run,
    env: run.runtimeEnvironment,
    snapshot: {
      status: "QUEUED",
      description: "Run was QUEUED, because all waitpoints are completed",
    },
    checkpointId: snapshot.checkpointId,  // 传递 checkpoint 供恢复使用
    completedWaitpoints: blockingWaitpoints.map(...),
  });

  break;
}
```

**关键语义**：
- **必须重新入队**：任务已经从 Worker 上卸载，需要重新调度分配执行资源
- **Checkpoint 依赖**：必须有 checkpoint 才能从挂起点恢复
- **优先级保留**：使用原始 `queueTimestamp`，挂起恢复的任务比新任务有更高优先级
- **completedWaitpoints 传递**：与 EXECUTING_WITH_WAITPOINTS 分支相同

### 2.4 分支设计的核心原则

1. **无操作优先**：只要任务在正常流程中，就不做额外操作
2. **状态机单向性**：状态只能向前推进，不能回退
3. **边界安全**：每个分支都有明确的前置条件检查
4. **幂等友好**：所有操作都是可重入的，重复调用不会产生副作用

---

## 边界三：批量等待点在运行时解析阶段被忽略却仍可恢复的原因

### 3.1 现象描述

在 `SharedRuntimeManager.resolveWaitpoint()` 中：

```typescript
private resolveWaitpoint(waitpoint: CompletedWaitpoint, resolverId?: ResolverId | null): void {
  if (waitpoint.type === "BATCH") {
    // We currently ignore these, they're not required to resume after a batch completes
    this.debugLog("ignoring BATCH waitpoint", { ... });
    return;  // 直接返回，不做任何解析
  }
  // ... 其他类型正常解析
}
```

BATCH 类型的等待点在运行时解析阶段被完全忽略，但批量任务仍然可以正常恢复执行。这看似矛盾的设计背后有清晰的架构考量。

### 3.2 批量等待的两层架构

批量等待实际上采用**"聚合标记 + 个体等待"**的两层设计：

```
┌─────────────────────────────────────────────────────────────────┐
│  任务代码调用 batchTriggerAndWait(tasks)                         │
└───────────────────────────────────┬─────────────────────────────┘
                                    │
            ┌───────────────────────┴───────────────────────┐
            │                                               │
            ▼                                               ▼
┌──────────────────────────────┐            ┌──────────────────────────────┐
│ BATCH 类型等待点             │            │ 每个子任务的 RUN 类型等待点   │
│  - id: batch_123             │            │  - id: run_001 (index 0)     │
│  - 作用：标记批量完成状态     │            │  - id: run_002 (index 1)     │
│  - 不参与运行时解析           │            │  - id: run_003 (index 2)     │
│                              │            │  - 参与运行时解析            │
└──────────────────────────────┘            └──────────────────────────────┘
```

### 3.3 SDK 端等待机制

当调用 `waitForBatch()` 时：

```typescript
async waitForBatch(params: { id: string; runCount: number; ctx: TaskRunContext }) {
  return this._preventMultipleWaits(async () => {
    if (!params.runCount) {
      return Promise.resolve({ id: params.id, items: [] });
    }

    // 关键：为每个子任务创建独立的 resolver
    const promises = Array.from({ length: params.runCount }, (_, index) => {
      const resolverId = `${params.id}_${index}` as ResolverId;
      return new Promise<CompletedWaitpoint>((resolve) => {
        this.resolversById.set(resolverId, resolve);
      });
    });

    // ...
    const waitpoints = await this.suspendable(Promise.all(promises));
    // ...
  });
}
```

**关键点**：
- 不创建 `batchId` 对应的 resolver
- 而是创建 `${batchId}_0`、`${batchId}_1`、...、`${batchId}_${runCount-1}` 共 `runCount` 个 resolver
- 批量等待的完成等价于所有子任务 resolver 全部完成

### 3.4 子任务完成时的解析机制

当某个子任务完成时，其对应的 RUN 类型等待点会被解析：

```typescript
private resolverIdFromWaitpoint(waitpoint: CompletedWaitpoint): ResolverId | null {
  switch (waitpoint.type) {
    case "RUN": {
      if (waitpoint.completedByTaskRun.batch) {
        // 属于批量的子任务：使用 batchId + index 构造 resolverId
        id = `${waitpoint.completedByTaskRun.batch.friendlyId}_${waitpoint.index}`;
      } else {
        // 独立子任务：使用 runId 作为 resolverId
        id = waitpoint.completedByTaskRun.friendlyId;
      }
      break;
    }
    // ...
  }
  return id as ResolverId;
}
```

**匹配过程**：
1. 子任务 `run_002` 完成 → 生成 RUN 类型等待点
2. 等待点携带 `batch.friendlyId = "batch_123"` 和 `index = 1`
3. `resolverIdFromWaitpoint()` 构造出 `"batch_123_1"`
4. 与 `waitForBatch()` 中注册的 resolver 完美匹配
5. Promise 被 resolve，对应位置的子任务结果可用

### 3.5 BATCH 等待点的真实作用

既然 BATCH 等待点不参与运行时解析，那它的作用是什么？

**作用 1：数据库层阻塞标记**

```sql
-- 批量任务创建时，插入一条 BATCH 等待点
INSERT INTO "Waitpoint" (id, type, idempotencyKey, completedByBatchId, ...)
VALUES ('wp_xxx', 'BATCH', 'batch_123', 'batch_123', ...);

-- 同时插入 TaskRunWaitpoint 关联，标记父任务被阻塞
INSERT INTO "TaskRunWaitpoint" (taskRunId, waitpointId, ...)
VALUES ('parent_run_id', 'wp_xxx', ...);
```

- 这是 `continueRunIfUnblocked` 判断"是否还有未完成等待点"的依据
- 所有子任务完成前，BATCH 等待点保持 PENDING 状态
- 因此 `continueRunIfUnblocked` 会一直返回 `blocked`，父任务不会被提前恢复

**作用 2：批量完成的聚合触发点**

在 `BatchSystem.#tryCompleteBatch()` 中：

```typescript
if (runs.every((r) => isFinalRunStatus(r.status))) {
  // 所有子任务完成后，完成 BATCH 等待点
  await this.waitpointSystem.completeWaitpoint({
    id: waitpoint.id,
    output: { value: "Batch waitpoint completed", isError: false },
  });
}
```

- BATCH 等待点的完成是"批量全部完成"的原子标记
- 完成后 `continueRunIfUnblocked` 才会认为所有等待点都已完成
- 此时才会触发父任务的恢复流程

**作用 3：审计与可观测性**

- 记录批量操作的元数据（创建时间、完成时间等）
- 在 UI 上展示批量等待状态
- 便于问题排查和历史追溯

### 3.6 完整恢复时序

```
1. 父任务调用 batchTriggerAndWait([task1, task2, task3])
   ├─ 创建 BATCH 等待点 (PENDING)
   ├─ 创建 3 个子任务 RUN 等待点 (PENDING)
   └─ 父任务状态变为 EXECUTING_WITH_WAITPOINTS 或 SUSPENDED

2. 子任务陆续完成
   ├─ task1 完成 → RUN 等待点 1 变为 COMPLETED
   ├─ task2 完成 → RUN 等待点 2 变为 COMPLETED
   │
   └─ 此时 continueRunIfUnblocked 检查发现 BATCH 等待点仍为 PENDING
      → 返回 blocked，不恢复父任务

3. task3 完成 → RUN 等待点 3 变为 COMPLETED
   ├─ BatchSystem 检测到所有子任务完成
   └─ 将 BATCH 等待点标记为 COMPLETED

4. BATCH 等待点完成触发 continueRunIfUnblocked
   ├─ 检查所有等待点：RUN1=COMPLETED, RUN2=COMPLETED, RUN3=COMPLETED, BATCH=COMPLETED
   ├─ 全部完成，进入恢复分支
   ├─ 创建新快照，携带 3 个 completedWaitpoints
   └─ 发送 workerNotification

5. Worker 端收到通知
   ├─ getRunExecutionData() 获取最新快照
   ├─ resolveWaitpoints() 逐个解析完成的等待点
   │  ├─ RUN1 → 匹配 resolver "batch_123_0" → resolve
   │  ├─ RUN2 → 匹配 resolver "batch_123_1" → resolve
   │  ├─ RUN3 → 匹配 resolver "batch_123_2" → resolve
   │  └─ BATCH → 忽略，直接返回
   ├─ Promise.all() 中的所有 Promise 都被 resolve
   └─ 父任务从 batchTriggerAndWait 处继续执行
```

### 3.7 设计权衡

| 设计决策 | 优点 | 缺点 |
|---------|------|------|
| BATCH 等待点不参与运行时解析 | 简化解析逻辑，避免重复等待；解耦批量标记与个体等待 | 看似"冗余"的设计，增加理解成本 |
| 每个子任务独立等待 | 子任务结果可以按序就绪，支持流式处理（如果需要） | resolver 数量与批量大小成正比 |
| BATCH 等待点作为阻塞标记 | 原子判断批量完成，避免并发问题 | 需要额外的数据库记录 |

这种设计的核心洞见是：**批量的"控制流"（何时恢复）和"数据流"（子任务结果）是分离的**。BATCH 等待点负责控制流，RUN 等待点负责数据流，两者各司其职。
