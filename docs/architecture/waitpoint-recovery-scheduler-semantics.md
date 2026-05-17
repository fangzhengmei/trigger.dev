# 等待点恢复链路：调度器语义与边界全景

## 摘要

本文档系统性阐述等待点恢复链路中的三个关键调度器语义边界：
1. Checkpoint 创建后并发槽的释放时机
2. SUSPENDED 恢复入队时 queueTimestamp/priorityMs 与 TTL 的继承规则
3. 恢复路径默认不开 fast path 对公平调度的影响

并将这三点与已有等待点边界（外部信号鉴权幂等、continueRunIfUnblocked 状态分支、批量等待点两层架构）串联，形成完整的恢复链路概念模型。

---

## 一、Checkpoint 创建后并发槽释放时机

### 1.1 并发槽生命周期

并发槽是 RunQueue 中控制环境级和队列级并发的核心机制。每个执行中的任务会占用一个或多个并发槽，任务结束或挂起时必须释放，否则会导致并发泄漏。

```
任务触发 → 入队 → 抢占并发槽 → 执行 → [遇到等待点] → 创建 Checkpoint → 释放并发槽
                                                                 ↓
                                                          任务进入 SUSPENDED
```

### 1.2 释放时机的精确边界

在 `CheckpointSystem.createCheckpoint()` 中，并发槽释放发生在**两个关键操作之后**：

```typescript
// checkpointSystem.ts:163-247
async createCheckpoint(...) {
  // 1. 状态前置校验（必须是可 checkpoint 的状态）
  if (!isCheckpointable(snapshot.executionStatus)) { ... }

  // 2. 更新 TaskRun 状态为 WAITING_TO_RESUME
  const run = await this.$.prisma.taskRun.update({
    data: { status: "WAITING_TO_RESUME" },
    ...
  });

  // 3. 创建 Checkpoint 记录（持久化执行镜像位置）
  const taskRunCheckpoint = await prisma.taskRunCheckpoint.create({ ... });

  // 4. 根据当前状态分支处理
  if (snapshot.executionStatus === "QUEUED_EXECUTING") {
    // 分支 A: 重新入队 QUEUED
    const newSnapshot = await this.enqueueSystem.enqueueRun({ ... });

    // 关键边界: 新快照创建完成后才释放并发
    if (run.organizationId) {
      await this.$.runQueue.releaseAllConcurrency(run.organizationId, run.id);
    }
  } else {
    // 分支 B: 创建 SUSPENDED 快照
    const newSnapshot = await this.executionSnapshotSystem.createExecutionSnapshot({
      snapshot: { executionStatus: "SUSPENDED", ... },
      checkpointId: taskRunCheckpoint.id,
      ...
    });

    // 关键边界: SUSPENDED 快照创建完成后才释放并发
    if (run.organizationId) {
      await this.$.runQueue.releaseAllConcurrency(run.organizationId, run.id);
    }
  }
}
```

### 1.3 设计考量：为什么在这个时机释放？

| 操作顺序 | 目的 | 风险 |
|---------|------|------|
| 创建 Checkpoint → 创建新快照 → 释放并发 | 保证状态机原子性：只有当新状态（QUEUED/SUSPENDED）被持久化后，才允许其他任务抢占并发槽 | 如果先释放并发再创建快照，极端情况下并发槽被抢占后快照创建失败，会出现"并发已释放但任务仍标记为执行中"的不一致 |

### 1.4 与等待点边界的关联

并发槽释放是等待点恢复链路的**资源交接点**：
- 暂停时释放的并发槽可能被新任务抢占
- 恢复时需要重新竞争并发槽
- 这是"已挂起任务必须重新入队"的根本原因之一

---

## 二、SUSPENDED 恢复入队时的属性继承规则

当 `continueRunIfUnblocked` 检测到 SUSPENDED 状态且所有等待点完成时，会调用 `enqueueSystem.enqueueRun()` 重新入队。入队时三个关键属性的继承规则如下：

### 2.1 queueTimestamp 与 priorityMs 继承

```typescript
// enqueueSystem.ts:89
const timestamp = (run.queueTimestamp ?? run.createdAt).getTime() - run.priorityMs;
```

**继承规则**：
- `queueTimestamp`：如果存在则直接继承，否则回退到 `createdAt`
- `priorityMs`：完全继承原始值
- 最终排队 score = `queueTimestamp - priorityMs`

**设计意图**：
- 挂起恢复的任务**不获得优先级加成**，与新任务公平竞争
- 保留原始排队时间，确保"先触发先执行"的语义
- priorityMs 可以为负值（提高优先级）或正值（降低优先级），继承保证语义一致

### 2.2 TTL 不继承规则

```typescript
// enqueueSystem.ts:91-99
let ttlExpiresAt: number | undefined;
if (includeTtl && run.ttl) {  // includeTtl 默认 false
  const expireAt = parseNaturalLanguageDuration(run.ttl);
  if (expireAt) {
    ttlExpiresAt = expireAt.getTime();
  }
}
```

**继承规则**：
- `includeTtl` 参数默认为 `false`
- SUSPENDED 恢复路径调用 `enqueueRun()` 时**不传递** `includeTtl`
- 因此 TTL 不会被重新计算和添加

**设计意图**：
- TTL 是"任务从触发到完成的最大时间"，不是"单次执行的最大时间"
- 挂起时间会计入 TTL 总时长（如果 TTL 是绝对时间）
- 避免每次恢复都重置 TTL，导致任务可以无限期挂起

### 2.3 与等待点边界的关联

属性继承规则是**公平调度**的基石：
- 继承 queueTimestamp 保证了等待时间长的恢复任务不会被新任务持续插队
- 不继承 TTL 保证了任务生命周期的可控性
- 这与 `continueRunIfUnblocked` 中"已排队/执行中任务不重复入队"的分支语义共同构成了调度一致性

---

## 三、恢复路径默认不开 fast path 对公平调度的影响

### 3.1 Fast Path 与 Slow Path 的本质区别

RunQueue 的 `enqueueMessage` 支持两种入队路径：

| 路径 | 触发条件 | 流程 | 延迟 |
|------|----------|------|------|
| **Fast Path** | `enableFastPath = true` 且有可用并发 | 直接写入 worker queue（Redis List） | < 10ms |
| **Slow Path** | 默认，或 fast path 条件不满足 | 写入 queue sorted set → 等待 500ms debounce → `processQueueForWorkerQueue` 作业调度 → 检查并发 → 写入 worker queue | ~500ms+ |

```
Fast Path:
  入队调用 → Lua 脚本检查并发 → 直接 push 到 worker queue → Worker 立即拉取

Slow Path:
  入队调用 → 写入 queue sorted set → 调度 processQueueForWorkerQueue (500ms 延迟)
                                                              ↓
                                              检查并发限制 → 按 score 排序 → 移动到 worker queue
```

### 3.2 恢复路径的默认配置

在等待点恢复链路中，`enqueueRun()` 的调用如下：

```typescript
// waitpointSystem.ts:865-878 (SUSPENDED 分支)
const newSnapshot = await this.enqueueSystem.enqueueRun({
  run,
  env: run.runtimeEnvironment,
  snapshot: {
    status: "QUEUED",
    description: "Run was QUEUED, because all waitpoints are completed",
  },
  checkpointId: snapshot.checkpointId ?? undefined,
  // 注意: 没有传递 enableFastPath，使用默认值 false
});
```

**关键事实**：
- 恢复路径**默认不开 fast path**
- 只有首次触发（从 trigger 接口入队）时才会传递 `includeTtl: true, enableFastPath: true`
- DEV 环境例外：根据 `WorkerInstanceGroup` 配置，DEV 环境可能始终开启 fast path

### 3.3 不开 Fast Path 对公平调度的影响

#### 正面影响：保证调度公平性

```
时间线示例（生产环境，并发限制=2）:

T0: 任务 A、B、C 同时触发 → A、B 抢占并发开始执行，C 排队 (score=1000)
T1: 任务 A 遇到等待点 → 创建 checkpoint → 释放并发 → 进入 SUSPENDED
T2: 任务 D 触发 → 抢占刚释放的并发开始执行
T3: 任务 A 的等待点完成 → 恢复入队（slow path，继承原始 score=1000）
T4: 500ms debounce 后，processQueueForWorkerQueue 运行
    → 比较 score: C(1000) vs A(1000) → 按 FIFO，C 先执行，A 继续排队

如果恢复走 fast path:
T3: 任务 A 恢复 → fast path 直接抢占并发 → A 先于 C 执行（不公平）
```

**公平性保证**：
- 恢复任务和新任务在同一个排队系统中竞争
- 按照 `queueTimestamp - priorityMs` 排序，先排队先执行
- 避免恢复任务"插队"，保证调度可预测性

#### 负面影响：增加恢复延迟

- 恢复任务必须等待至少 500ms 的 debounce 延迟
- 如果队列有积压，可能需要等待更久
- 对于时间敏感的任务（如 Webhook 响应），这个延迟可能不可接受

#### 权衡决策

| 维度 | 开 Fast Path | 不开 Fast Path |
|------|-------------|----------------|
| 恢复延迟 | < 10ms | ~500ms+ |
| 公平性 | 可能插队 | 严格按 score 排序 |
| 吞吐量 | 更高（减少一次 Redis 操作） | 稍低 |
| 适用场景 | DEV 环境、低延迟要求 | PROD 环境、公平性优先 |

### 3.4 与等待点边界的关联

不开 fast path 的决策与其他边界形成协同：
- 与 `continueRunIfUnblocked` 中"已执行中任务 skipped"分支协同：避免任务同时出现在多个执行路径
- 与批量等待点的"控制流/数据流分离"设计协同：批量恢复也走 slow path，保证批量内任务的调度顺序可预测
- 与 TTL 不继承规则协同：虽然恢复延迟增加，但 TTL 不会被重置，任务总生命周期仍可控

---

## 四、完整恢复链路的语义串联

### 4.1 端到端恢复时序

```
外部信号完成 Token
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│ 边界 1: 鉴权与幂等                                              │
│  - RBAC + environmentId 双重校验                                │
│  - updateMany(status='PENDING') 原子更新                         │
│  - jobId 去重防止重复调度                                       │
└───────────────────────────────────┬─────────────────────────────┘
                                    │
                                    ▼
                          调度 continueRunIfUnblocked
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 边界 2: 状态分支语义                                            │
│  - QUEUED/EXECUTING/FINISHED: skipped                           │
│  - EXECUTING_WITH_WAITPOINTS: 转 EXECUTING + 直接通知           │
│  - SUSPENDED: 重新入队 QUEUED                                    │
└───────────────────────────────────┬─────────────────────────────┘
                                    │ （SUSPENDED 分支）
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 边界 3: 并发槽释放（已在暂停时完成）                              │
│  - Checkpoint 创建 → 新快照持久化 → releaseAllConcurrency        │
│  - 此时并发槽可能已被新任务占用                                   │
└───────────────────────────────────┬─────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 边界 4: 属性继承                                                │
│  - queueTimestamp/priorityMs: 继承，保证公平性                   │
│  - TTL: 不继承，保证生命周期可控                                 │
└───────────────────────────────────┬─────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 边界 5: Fast Path 决策                                          │
│  - 默认关闭，走 slow path                                        │
│  - 500ms debounce + 按 score 排队                               │
│  - 与新任务公平竞争                                              │
└───────────────────────────────────┬─────────────────────────────┘
                                    │
                                    ▼
                          RunQueue 调度执行
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 边界 6: 运行时解析                                              │
│  - BATCH 等待点忽略（控制流已完成）                               │
│  - RUN 等待点按 ${batchId}_${index} 匹配 resolver                │
└───────────────────────────────────┬─────────────────────────────┘
                                    │
                                    ▼
                          任务从暂停处继续执行
```

### 4.2 语义设计的核心原则

1. **状态机原子性优先**：所有状态转换都在持久化完成后才释放资源或通知下游
2. **公平调度优先于延迟**：宁可增加 500ms 延迟，也要保证调度顺序可预测
3. **生命周期可控**：TTL 不重置，避免挂起机制被滥用
4. **幂等性贯穿全链路**：从 API 层到 Redis 作业，每层都有去重机制
5. **控制流与数据流分离**：批量等待中 BATCH 类型负责控制流，RUN 类型负责数据流

### 4.3 关键代码位置索引

| 语义边界 | 文件位置 | 核心代码 |
|---------|----------|----------|
| 并发槽释放 | `internal-packages/run-engine/src/engine/systems/checkpointSystem.ts:201, 239` | `releaseAllConcurrency()` |
| 属性继承 | `internal-packages/run-engine/src/engine/systems/enqueueSystem.ts:89, 91-99` | `timestamp` 计算、`ttlExpiresAt` 条件 |
| Fast Path 决策 | `internal-packages/run-engine/src/engine/systems/enqueueSystem.ts:39` | `enableFastPath = false` 默认值 |
| 状态分支语义 | `internal-packages/run-engine/src/engine/systems/waitpointSystem.ts:710-891` | `switch (snapshot.executionStatus)` |
| 鉴权幂等 | `apps/webapp/app/routes/api.v1.waitpoints.tokens.$waitpointFriendlyId.complete.ts` | `authorization`、`updateMany` |
| 批量解析 | `packages/core/src/v3/runtime/sharedRuntimeManager.ts:224-229` | BATCH 类型忽略逻辑 |
