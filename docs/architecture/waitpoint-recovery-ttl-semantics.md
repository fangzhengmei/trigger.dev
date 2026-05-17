# 等待点恢复链路 TTL 语义澄清

## 摘要

本文档系统性澄清恢复链路中的 TTL（Time-To-Live）语义：
1. 明确四种入队路径中哪些会携带 `includeTtl: true`
2. 解释等待点恢复入队为何不携带 TTL
3. 分析这一差异对过期时间计算与调度公平性的影响
4. 串联 TTL 系统与等待点恢复的完整交互边界

---

## 一、TTL 系统核心机制

### 1.1 TTL 的两层含义

在 Trigger.dev 中，TTL 有两层互补的含义：

| 层面 | 语义 | 作用范围 | 执行时机 |
|------|------|----------|----------|
| **排队超时** | 任务在队列中等待执行的最长时间 | 仅排队中状态 | RunQueue 中的 TTL 消费者轮询 |
| **总生命周期超时** | 任务从触发到最终完成的最长时间 | 整个任务生命周期 | （当前未独立实现，依赖排队超时 + 执行超时） |

> **重要澄清**：当前 TTL 系统主要实现的是**排队超时**，即只对"在队列中等待并发槽"的任务进行过期检查。任务一旦开始执行（进入 EXECUTING 状态），TTL 系统不再监控。执行超时由 Worker 端的 `maxDuration` 机制控制。

### 1.2 TTL 系统工作原理

```
入队时（includeTtl = true）:
  ├─ 计算 ttlExpiresAt = 当前时间 + parseNaturalLanguageDuration(run.ttl)
  ├─ 写入队列消息的 ttlExpiresAt 字段
  └─ Lua 脚本原子地将任务加入 TTL 排序集（score = ttlExpiresAt）

TTL 消费者（每个分片，默认 1000ms 轮询）:
  ├─ ZRANGEBYSCORE ttlQueueKey -inf <当前时间>  获取所有过期任务
  ├─ 原子操作：
  │   ├─ 从 TTL 排序集中移除
  │   ├─ 从正常队列 sorted set 中确认（ack）
  │   └─ 推送到 TTL worker 队列进行 DB 状态更新
  └─ TTL worker 执行：更新 TaskRun 状态为 TIMED_OUT，记录错误

任务开始执行（从 worker queue 弹出）:
  └─ TTL 系统不再追踪该任务（即使 ttlExpiresAt 已过）
```

**关键数据结构**：
- **TTL 排序集**：`runqueue:v3:ttl:queue:<shard>`，score = `ttlExpiresAt`（unix ms）
- **TTL worker 队列**：`runqueue:v3:ttl:worker:queue`，存储待处理的过期任务
- **TTL worker 哈希表**：`runqueue:v3:ttl:worker:items`，存储任务详情供 worker 处理

---

## 二、入队路径与 TTL 携带规则

### 2.1 四种入队路径全景

| 入队路径 | 调用位置 | `includeTtl` | 设计意图 |
|---------|----------|--------------|----------|
| **触发首入队** | `engine/index.ts:817` | `true` | 任务首次进入系统，开始计算排队超时 |
| **Delayed Run 入队** | `delayedRunSystem.ts:174` | `true` | 延迟任务"到期"才是真正的入队起点 |
| **Pending Version 入队** | `pendingVersionSystem.ts:105` | `true` | 版本就绪后才是真正的入队起点 |
| **等待点恢复入队** | `waitpointSystem.ts:865` | `false`（默认） | 已执行过的任务不应再受排队超时限制 |
| **Checkpoint 重新入队** | `checkpointSystem.ts:177` | `false`（默认） | 同上，已执行过的任务不重置排队超时 |

### 2.2 各路径详细说明

#### 路径 1：触发首入队（Trigger First Enqueue）

```typescript
// engine/index.ts:810-819
await this.enqueueSystem.enqueueRun({
  run: taskRun,
  env: environment,
  workerId,
  runnerId,
  tx: prisma,
  skipRunLock: true,
  includeTtl: true,      // ✅ 携带 TTL
  enableFastPath,
});
```

**语义**：这是任务的"出生点"，TTL 从此时开始计算。任务在触发后可能需要等待并发槽，排队超时从这里计时。

#### 路径 2：Delayed Run 入队

```typescript
// delayedRunSystem.ts:165-175
// includeTtl: true so the run's TTL is armed from the moment it enters
// the queue (not from taskRun.createdAt). The TTL system tracks runs
// that are queued and have never started — delayed runs are first
// enqueued here, so this is the correct point to arm TTL.
await this.enqueueSystem.enqueueRun({
  run,
  env: run.runtimeEnvironment,
  batchId: run.batchId ?? undefined,
  skipRunLock: true,
  includeTtl: true,      // ✅ 携带 TTL
});
```

**语义**：延迟任务在 `delayUntil` 之前处于"休眠"状态，不应该计入排队时间。只有当延迟到期、真正进入排队系统时，TTL 才开始计时。

#### 路径 3：Pending Version 入队

```typescript
// pendingVersionSystem.ts:101-106
// PENDING_VERSION re-enqueue is the first time this run is actually
// entering the run queue (the original enqueue was held back waiting
// for a worker version). Arm TTL here so the TTL system can expire it
// if it sits queued waiting on a concurrency slot.
await this.enqueueSystem.enqueueRun({
  ...
  includeTtl: true,      // ✅ 携带 TTL
});
```

**语义**：任务被触发但所需的 Worker 版本尚未部署，处于"等待版本"状态。只有当版本就绪、真正进入排队系统时，TTL 才开始计时。

#### 路径 4：等待点恢复入队（SUSPENDED → QUEUED）

```typescript
// waitpointSystem.ts:865-878
const newSnapshot = await this.enqueueSystem.enqueueRun({
  run,
  env: run.runtimeEnvironment,
  snapshot: {
    status: "QUEUED",
    description: "Run was QUEUED, because all waitpoints are completed",
  },
  checkpointId: snapshot.checkpointId ?? undefined,
  // ❌ 不传递 includeTtl，使用默认值 false
});
```

**语义**：任务已经执行过至少一部分，只是因为等待点而挂起。恢复后重新排队，但：
1. 任务的"总生命周期"已经在流逝，不应该重置
2. 已经获得过并发槽（至少执行过一部分），不应该再受"排队超时"限制
3. 防止通过"挂起-恢复"循环无限延长任务生命周期

#### 路径 5：Checkpoint 重新入队（QUEUED_EXECUTING → QUEUED）

```typescript
// checkpointSystem.ts:177-193
const newSnapshot = await this.enqueueSystem.enqueueRun({
  run,
  env: run.runtimeEnvironment,
  snapshot: {
    status: "QUEUED",
    description:
      "Run was QUEUED, because it was queued and executing and a checkpoint was created",
    metadata: snapshot.metadata,
  },
  checkpointId: taskRunCheckpoint.id,
  // ❌ 不传递 includeTtl，使用默认值 false
});
```

**语义**：与等待点恢复类似，任务已经在执行过程中，只是因为某种原因（如抢占）创建了 Checkpoint 并需要重新排队。不应该重置 TTL。

### 2.3 携带规则的判定公式

```
includeTtl = (这是任务第一次进入排队系统) ? true : false
```

**判定标准**：
- ✅ 是：任务从未获得过并发槽，从未开始执行
- ❌ 否：任务已经开始执行（即使被挂起/抢占）

---

## 三、等待点恢复不携带 TTL 的深层原因

### 3.1 核心理念：TTL 是"排队超时"，不是"生命周期超时"

TTL 系统的设计初衷是防止任务**在队列中无限期等待**，而不是限制任务的**总执行时间**。

```
任务生命周期视图：

  触发 → [排队] → 执行 → [等待点] → 挂起 → [恢复排队] → 继续执行 → ... → 完成
          │                                                │
          └─ TTL 监控（首次入队时 arm）                   └─ TTL 不监控（已执行过）
```

**为什么不监控执行中的任务？**
- 执行中的任务由 Worker 端的 `maxDuration` 机制控制
- TTL 系统基于 Redis 排序集，无法高效追踪"执行中"状态
- 执行时间差异很大（从毫秒到小时），单一 TTL 不适用

### 3.2 避免的反模式

如果恢复入队也携带 TTL（即每次恢复都重置 TTL），会出现以下问题：

**反模式 1：无限挂起循环**
```
任务 TTL = 30 分钟
T0: 入队，TTL 到期时间 = T0 + 30min
T25: 遇到等待点，挂起
T35: 等待点完成，恢复入队 → 如果重置 TTL，新到期时间 = T35 + 30min
T60: 再次遇到等待点，挂起
T70: 再次恢复 → 再次重置 TTL
... 无限循环，任务永远不会因为排队超时而被终止
```

**反模式 2：不公平的优先级**
```
高优先级任务 A（TTL = 1h）：
  T0: 入队，TTL = T0+1h
  T59: 挂起
  T61: 恢复 → 重置 TTL = T61+1h
  → 实际上获得了近 2 小时的排队超时保护

普通任务 B（TTL = 1h）：
  T0: 入队，TTL = T0+1h
  T55: 仍在排队
  T61: 因为超时而被终止

→ 能够挂起的任务获得了不公平的超时保护
```

### 3.3 边缘场景说明

**场景：任务挂起了很长时间，恢复后立即执行失败**
```
任务 TTL = 30 分钟
T0: 入队
T5: 开始执行
T10: 遇到等待点，挂起
T60: 等待点完成（已过去 50 分钟）
T60: 恢复入队（不携带 TTL）
T61: 获得并发槽，开始执行
T62: 执行失败（比如依赖的资源已过期）
```

**当前行为**：任务不会因为 TTL 过期而被系统自动终止，而是正常执行，业务代码自行处理过期问题。

**设计考量**：
- TTL 系统不知道任务挂起的原因（可能是等待外部审批，需要几天）
- 强制终止可能会破坏业务逻辑（如审批流程）
- 业务代码应该自己处理"结果是否过期"的判断
- 如果需要总生命周期超时，应该在业务层面实现（如使用 `wait.for({ timeout: "24h" })`）

---

## 四、对过期时间与调度公平性的影响

### 4.1 对过期时间计算的影响

#### 正面影响：语义清晰，可预测

```
任务 A（ttl: "30m"）:
  T0: 触发入队 → TTL 到期 = T0 + 30min
  T5: 开始执行 → TTL 系统停止追踪
  T10: 挂起
  T40: 恢复入队 → 不重置 TTL（即使 T0+30min = T30 已过）
  T41: 继续执行 → 正常执行

结果：任务不会因为"挂起期间 TTL 到期"而被终止
```

**可预测性**：开发者知道 TTL 只影响"首次排队"时间，挂起不会导致任务意外终止。

#### 负面影响：总生命周期不可控

```
任务 B（ttl: "30m"）:
  T0: 触发入队 → TTL 到期 = T0 + 30min
  T5: 开始执行
  T10: 挂起（等待人工审批）
  T1440: （24 小时后）审批完成，恢复入队
  T1441: 继续执行

结果：任务总生命周期 24 小时+，远超 TTL 设置的 30 分钟
```

**当前限制**：TTL 不能用作总生命周期限制。如果需要严格的总生命周期控制，需要：
1. 在业务代码中记录开始时间，每次恢复时检查
2. 使用 `wait.for({ timeout: "24h" })` 为单个等待点设置超时
3. （高级）使用外部监控系统追踪总生命周期

### 4.2 对调度公平性的影响

#### 正面影响：防止"恢复插队"

```
时间线（并发限制 = 1）:

T0: 任务 A（ttl: "1h"）入队 → 开始执行
T5: 任务 B（ttl: "1h"）入队 → 排队，TTL 到期 = T5 + 1h
T10: 任务 A 遇到等待点 → 挂起，释放并发槽
T10: 任务 B 获得并发槽 → 开始执行
T30: 任务 A 等待点完成 → 恢复入队（不重置 TTL）
T65: 任务 B 执行完成，释放并发槽
T65: 任务 A 获得并发槽 → 继续执行

如果恢复入队重置 TTL:
  T30: 任务 A 恢复入队 → 重置 TTL = T30 + 1h
  T65: 任务 B 完成
  → 任务 A 正常执行（但获得了额外的 TTL 保护）
```

**公平性保证**：
- 恢复任务和新任务使用相同的排队 score（`queueTimestamp - priorityMs`）
- 恢复任务不会因为"挂起过"而获得额外的超时保护
- 所有任务在排队系统中遵循相同的规则

#### 负面影响："古老"任务可能堆积

```
场景：系统故障，大量任务挂起
T0-T100: 1000 个任务陆续触发并执行，然后都遇到等待点挂起
T200: 系统恢复，等待点批量完成
T200: 1000 个任务同时恢复入队 → 都不重置 TTL
T200: 这些任务的 TTL 可能早在 T130 之前就已到期
T200: 但它们不会被 TTL 系统终止，全部进入排队
T200-T300: 系统花费 100 分钟处理这些"过期"任务
T300+: 新触发的任务被延迟
```

**权衡**：
- 系统选择了"让已执行的任务完成"，而不是"为新任务让路"
- 这是一个保守的设计决策，避免数据丢失（已执行的部分可能有副作用）
- 如果需要"丢弃过期任务优先处理新任务"，需要业务层面实现

### 4.3 与其他调度语义的协同

TTL 携带规则与其他调度语义形成完整的设计体系：

| 调度语义 | 设计原则 | 与 TTL 规则的协同 |
|---------|----------|-------------------|
| **queueTimestamp 继承** | 挂起任务不获得优先级加成 | 共同保证恢复任务与新任务公平竞争 |
| **Fast Path 默认关闭** | 宁可增加延迟也要保证公平 | 共同防止恢复任务插队 |
| **状态分支 skipped** | 已排队/执行中任务不重复入队 | 避免 TTL 被意外重置（如果重复入队可能携带 TTL） |

---

## 五、完整交互边界总结

### 5.1 TTL 与等待点恢复的交互时序

```
外部信号完成 Waitpoint
      │
      ▼
completeWaitpoint()
      │
      ▼
continueRunIfUnblocked()
      │
      ├─ 检查等待点：全部完成？
      ├─ 检查快照状态：SUSPENDED？
      └─ 是 → 调用 enqueueRun()
           │
           ├─ includeTtl: false（默认）
           ├─ 计算 timestamp = queueTimestamp - priorityMs（继承）
           ├─ enableFastPath: false（默认）
           └─ 写入 RunQueue
                │
                ├─ 消息不包含 ttlExpiresAt
                ├─ 不加入 TTL 排序集
                └─ 走 Slow Path（500ms debounce + 按 score 排序）
                     │
                     ▼
              与新任务公平竞争并发槽
                     │
                     ▼
              获得并发槽 → 从 Checkpoint 恢复执行
```

### 5.2 关键设计原则

1. **一次排队，一次 TTL**：任务只有在**第一次进入排队系统**时才设置 TTL
2. **执行过就不再超时**：任务一旦开始执行，TTL 系统不再追踪
3. **挂起是执行的延续**：挂起-恢复不是重新触发，不应该重置任何生命周期计时
4. **公平优先于效率**：宁可让"过期"的恢复任务执行，也不让它们抢占新任务的资源

### 5.3 代码位置索引

| 语义点 | 文件位置 | 关键代码 |
|--------|----------|----------|
| 触发首入队携带 TTL | `internal-packages/run-engine/src/engine/index.ts:817` | `includeTtl: true` |
| Delayed Run 携带 TTL | `internal-packages/run-engine/src/engine/systems/delayedRunSystem.ts:174` | `includeTtl: true` |
| Pending Version 携带 TTL | `internal-packages/run-engine/src/engine/systems/pendingVersionSystem.ts:105` | `includeTtl: true` |
| 等待点恢复不携带 | `internal-packages/run-engine/src/engine/systems/waitpointSystem.ts:865` | 无 includeTtl 参数 |
| Checkpoint 重入不携带 | `internal-packages/run-engine/src/engine/systems/checkpointSystem.ts:177` | 无 includeTtl 参数 |
| TTL 计算逻辑 | `internal-packages/run-engine/src/engine/systems/enqueueSystem.ts:91-99` | `ttlExpiresAt` 条件计算 |
| TTL 过期消费者 | `internal-packages/run-engine/src/run-queue/index.ts:1376-1415` | `#startTtlConsumers()` |
| TTL 过期原子操作 | `internal-packages/run-engine/src/run-queue/index.ts:1421-1448` | `#expireTtlRuns()` |
