# TTL 语义补全：Fast Path 下的过期行为差异

## 摘要

本文档补全 TTL 语义中剩余的关键边界：
1. 生产环境走 fast path 时 TTL sorted set 被跳过的机制与影响
2. 开发环境由 legacy `expireRun` 作业兜底的设计原因
3. 这一差异对过期行为与调度公平性的具体影响
4. 串联整个 TTL 系统在不同入队路径、不同环境下的完整语义

---

## 一、Fast Path 与 Slow Path 的 TTL 处理差异

### 1.1 Lua 脚本中的分支逻辑

在 `RunQueue.#callEnqueueMessage()` 的 Lua 脚本中，fast path 和 slow path 对 TTL 的处理存在本质差异：

```lua
-- Fast path: 直接推送到 worker queue
if enableFastPath == '1' then
  -- 检查队列是否为空、环境并发是否可用、队列并发是否可用
  if #available == 0 and envCurrent < envLimitWithBurst and queueCurrent < queueLimit then
    redis.call('SET', messageKey, messageData)
    redis.call('SADD', queueCurrentConcurrencyKey, messageId)
    redis.call('SADD', envCurrentConcurrencyKey, messageId)
    redis.call('RPUSH', workerQueueKey, messageKeyValue)
    -- ⚠️  关键：跳过 TTL sorted set
    -- Skip TTL sorted set: the expireRun worker job handles TTL expiry independently
    return 1  -- fastPathTaken = true
  end
end

-- Slow path: 写入 queue sorted set，等待调度
redis.call('SET', messageKey, messageData)
redis.call('ZADD', queueKey, messageScore, messageId)
redis.call('ZADD', envQueueKey, messageScore, messageId)
-- ✅ 加入 TTL sorted set
redis.call('ZADD', ttlQueueKey, ttlScore, ttlMember)
-- ... 更新父队列权重 ...
return 0  -- fastPathTaken = false
```

**核心差异表**：

| 路径 | 是否加入 TTL sorted set | 过期机制 | 过期检查时机 |
|------|------------------------|----------|--------------|
| **Slow Path** | ✅ 是 | TTL 消费者轮询 TTL sorted set | 入队后立即开始，每 1000ms 检查 |
| **Fast Path** | ❌ 否 | 依赖 legacy `expireRun` 作业（仅 DEV 环境） | 仅 DEV 环境有兜底，PROD 环境无过期检查 |

### 1.2 为什么 Fast Path 跳过 TTL Sorted Set？

#### 设计考量 1：Fast Path 的本质是"立即执行"

Fast Path 的前提条件是：
- 队列为空（没有待执行的任务）
- 环境级并发有可用槽位
- 队列级并发有可用槽位

满足这些条件意味着任务**理论上应该立即被执行**，不应该在队列中等待。既然不等待，"排队超时"（TTL 的核心语义）就不适用。

#### 设计考量 2：TTL 消费者的架构限制

TTL 消费者的工作原理是：
```
1. ZRANGEBYSCORE ttlQueueKey -inf <now> → 获取过期任务
2. 从正常 queue sorted set 中 ack（ZREM）这些任务
3. 推送到 TTL worker 队列进行 DB 更新
```

但 Fast Path 的任务直接进入了 worker queue（Redis List），而不是 queue sorted set（Redis Sorted Set）。TTL 消费者无法高效地扫描和确认 worker queue 中的任务。

#### 设计考量 3：性能与复杂度权衡

在 Fast Path 中加入 TTL 追踪需要：
- 额外维护一个"worker queue 中的任务"的 TTL 集合
- 增加 Lua 脚本的复杂度
- 可能引入竞态条件（任务被 Worker 拉取的同时被 TTL 消费者过期）

权衡之后，设计选择是：**Fast Path 任务不加入 TTL sorted set，依赖其他机制兜底**。

---

## 二、开发环境的 Legacy ExpireRun 兜底

### 2.1 触发条件

在三个关键入队路径中，只有 `DEVELOPMENT` 环境会调度 legacy `expireRun` 作业：

```typescript
// engine/index.ts:800-808
// The new batch TTL path only expires runs still in the queue
// sorted set (waiting on a concurrency slot). For DEV
// environments where the dev CLI may not be running, fast-pathed
// runs can sit on the worker queue indefinitely and never get
// claimed for expiration. Keep the legacy per-run expireRun job
// armed for DEV so those runs still expire.
if (taskRun.ttl && environment.type === "DEVELOPMENT") {
  await this.ttlSystem.scheduleExpireRun({ runId: taskRun.id, ttl: taskRun.ttl });
}
```

同样的逻辑也出现在 `delayedRunSystem.ts:147-159`（Delayed Run 入队时）。

**哪些路径会触发 legacy expireRun？**

| 入队路径 | DEV 环境 | PROD 环境 |
|---------|----------|-----------|
| 触发首入队 | ✅ 调度 `expireRun` 作业 | ❌ 不调度 |
| Delayed Run 入队 | ✅ 调度 `expireRun` 作业 | ❌ 不调度 |
| Pending Version 入队 | ❌ 不调度（代码中无此逻辑） | ❌ 不调度 |
| 等待点恢复入队 | ❌ 不调度 | ❌ 不调度 |
| Checkpoint 重新入队 | ❌ 不调度 | ❌ 不调度 |

### 2.2 Legacy ExpireRun 的工作原理

```typescript
// ttlSystem.ts:136-147
async scheduleExpireRun({ runId, ttl }: { runId: string; ttl: string }) {
  const expireAt = parseNaturalLanguageDuration(ttl);
  if (expireAt) {
    await this.$.worker.enqueue({
      id: `expireRun:${runId}`,  // 每个任务独立的作业 ID
      job: "expireRun",
      payload: { runId },
      availableAt: expireAt,      // 在 TTL 到期时执行
    });
  }
}
```

**与 TTL 消费者的区别**：

| 维度 | TTL 消费者（批量） | Legacy ExpireRun（独立作业） |
|------|-------------------|-----------------------------|
| 触发方式 | 定时轮询（每 1000ms） | 基于时间的独立作业调度 |
| 处理粒度 | 批量处理（每次最多 100 个） | 每个任务独立作业 |
| 资源消耗 | 低（O(log N) 扫描） | 高（每个 TTL 任务一个 Redis 作业） |
| 过期精度 | ~1000ms | 取决于 Redis Worker 的调度精度（通常 < 100ms） |
| 适用场景 | Slow Path 大量任务 | Fast Path DEV 环境少量任务 |

### 2.3 expireRun 执行时的安全检查

```typescript
// ttlSystem.ts:25-57
async expireRun({ runId, tx }: { runId: string; tx?: PrismaClientOrTransaction }) {
  await this.$.runLock.lock("expireRun", [runId], async () => {
    const snapshot = await getLatestExecutionSnapshot(prisma, runId);

    // 安全检查 1: 如果正在执行，不过期
    if (isExecuting(snapshot.executionStatus)) {
      return;
    }

    // 安全检查 2: 只过期 PENDING 状态的任务
    const run = await prisma.taskRun.findFirst({ where: { id: runId } });
    if (!run || run.status !== "PENDING") {
      return;
    }

    // 安全检查 3: 如果已被锁定（即将执行），不过期
    if (run.lockedAt) {
      return;
    }

    // ... 执行过期逻辑 ...
  });
}
```

**三重安全检查**确保了：
1. 不会过期正在执行的任务
2. 不会过期已经结束的任务
3. 不会过期即将被 Worker 拉取的任务

---

## 三、生产环境 Fast Path 任务的过期行为

### 3.1 PROD 环境 Fast Path 的"无过期"状态

在生产环境中，如果任务走 Fast Path：

```
PROD 环境 + enableFastPath = true + 并发可用 → Fast Path
                                                          ↓
                                          不加入 TTL sorted set
                                                          ↓
                                      不调度 legacy expireRun 作业
                                                          ↓
                        ❌ 任务在 worker queue 中不会被 TTL 系统过期
```

**这是一个已知的设计权衡**，而非 bug。理由如下：

1. **Fast Path 应该立即执行**：如果并发可用，任务应该在几秒内被 Worker 拉取执行
2. **Worker 端有 maxDuration 控制**：任务开始执行后，由 Worker 端的 `maxDuration` 机制控制执行超时
3. **极端情况罕见**：在 PROD 环境中，Worker 通常持续运行，worker queue 中的任务不会长时间堆积
4. **监控告警兜底**：生产环境通常有额外的监控系统，可以发现"卡在 worker queue 中"的任务

### 3.2 可能的边缘场景

**场景 1：Worker 突然下线**
```
T0: 任务 A 触发，走 Fast Path 进入 worker queue
T1: Worker 进程崩溃（或部署重启）
T2: 任务 A 留在 worker queue 中
T3: 没有新的 Worker 连接（罕见，但可能发生）
结果：任务 A 永远不会被执行，也不会被 TTL 过期
```

**场景 2：Worker 被大量任务压垮**
```
T0-T10: 1000 个任务同时触发，都走 Fast Path（并发限制 = 100）
T0: 前 100 个任务被执行
T10: 剩余 900 个任务在 worker queue 中等待
结果：这 900 个任务不会被 TTL 过期，即使它们等待了远超 TTL 的时间
```

> **注意**：场景 2 实际上不会发生，因为 Fast Path 的前提条件是"队列当前为空"。如果已经有 900 个任务在等待，后续任务会走 Slow Path 并加入 TTL sorted set。

### 3.3 与等待点恢复的关联

等待点恢复入队默认走 Slow Path（`enableFastPath = false`），因此：
- ✅ 恢复任务会加入 TTL sorted set
- ✅ TTL 消费者会正常监控和过期
- ✅ 不存在"卡在 worker queue 中无法过期"的问题

这是恢复路径不开 Fast Path 的另一个好处：**保证 TTL 语义的一致性**。

---

## 四、对过期行为的具体影响

### 4.1 不同环境、不同路径的过期行为矩阵

| 环境 | 入队路径 | Fast Path 触发 | TTL sorted set | Legacy ExpireRun | 过期行为 |
|------|---------|----------------|----------------|------------------|----------|
| **PROD** | 触发首入队 | 是（并发可用） | ❌ 未加入 | ❌ 未调度 | ⚠️ 仅依赖 Worker 拉取，无过期 |
| **PROD** | 触发首入队 | 否（并发不足） | ✅ 已加入 | ❌ 未调度 | ✅ TTL 消费者轮询过期 |
| **PROD** | Delayed Run 入队 | 是（并发可用） | ❌ 未加入 | ❌ 未调度 | ⚠️ 仅依赖 Worker 拉取，无过期 |
| **PROD** | Delayed Run 入队 | 否（并发不足） | ✅ 已加入 | ❌ 未调度 | ✅ TTL 消费者轮询过期 |
| **PROD** | 等待点恢复入队 | 否（默认关闭） | ✅ 已加入 | ❌ 未调度 | ✅ TTL 消费者轮询过期 |
| **DEV** | 触发首入队 | 是（DEV 默认开启） | ❌ 未加入 | ✅ 已调度 | ✅ Legacy ExpireRun 作业兜底 |
| **DEV** | 触发首入队 | 否（并发不足） | ✅ 已加入 | ✅ 已调度 | ✅ 双重保障（先触发的为准） |
| **DEV** | Delayed Run 入队 | 是（DEV 默认开启） | ❌ 未加入 | ✅ 已调度 | ✅ Legacy ExpireRun 作业兜底 |
| **DEV** | 等待点恢复入队 | 否（默认关闭） | ✅ 已加入 | ❌ 未调度 | ✅ TTL 消费者轮询过期 |

### 4.2 双重保障的竞态条件

在 DEV 环境中，如果任务走 Slow Path，会同时有两种过期机制：

```
任务 TTL = 30 分钟
T0: 入队（Slow Path）
    ├─ 加入 TTL sorted set，TTL 消费者将在 T0+30min 检查
    └─ 调度 legacy expireRun 作业，将在 T0+30min 执行
T0+30min:
    ├─ TTL 消费者扫描到任务，尝试过期
    └─ expireRun 作业同时触发，尝试过期
```

**结果**：先获得 `runLock` 的那个会成功执行过期，另一个会因为状态不再是 `PENDING` 而安全跳过。这是良性竞态，不会导致重复过期。

### 4.3 过期时间的精度差异

| 过期机制 | 理论精度 | 实际精度 | 影响因素 |
|---------|----------|----------|----------|
| TTL 消费者 | 1000ms | ~1000-2000ms | 轮询间隔、批量大小、分片数量 |
| Legacy ExpireRun | Redis Worker 调度精度 | ~10-100ms | Worker 负载、作业队列长度 |

**实际影响**：对于分钟级以上的 TTL（如 `ttl: "5m"`），精度差异可以忽略。只有对于秒级 TTL（如 `ttl: "30s"`），精度差异才可能有感知。

---

## 五、对调度公平性的影响

### 5.1 Fast Path 对公平性的破坏

Fast Path 本质上是一种"插队"机制：

```
时间线（并发限制 = 1）:

T0: 任务 A（低优先级）入队 → Slow Path，score = 1000
T1: 任务 B（高优先级）入队 → 检测到并发不可用 → Slow Path，score = 500
T2: 任务 A 执行完成，释放并发
T3: processQueueForWorkerQueue 运行（500ms debounce 后）
    → 按 score 排序，任务 B（500）先执行，任务 A（1000）后执行
    ✅ 公平调度，高优先级先执行

如果任务 B 走 Fast Path:
T0: 任务 A（低优先级）执行中
T1: 任务 B（高优先级）入队 → 检测到并发不可用 → Slow Path，score = 500
T2: 任务 C（高优先级）入队 → 检测到并发可用（任务 A 刚完成）
    → Fast Path，直接进入 worker queue
    → 任务 C 先于任务 B 执行
    ❌ 不公平！任务 B 比任务 C 早触发，但后执行
```

**Fast Path 的公平性代价**：
- 新触发的任务可能"插队"到已排队任务之前
- 优先级排序只在 Slow Path 中有效
- 这是为了低延迟而做出的公平性妥协

### 5.2 TTL 差异加剧的公平性问题

Fast Path 任务不加入 TTL sorted set，进一步加剧了公平性问题：

```
场景：DEV 环境，并发限制 = 1，大量任务触发

T0-T100: 100 个任务陆续触发
    ├─ 前 10 个：Fast Path，立即执行
    ├─ 中间 80 个：Slow Path，加入 TTL sorted set（TTL = 5m）
    └─ 后 10 个：Fast Path（中间 80 个还在 debounce 等待期，队列看似为空）
        → 这 10 个任务插队执行
        → 中间 80 个任务等待更久，部分可能在 TTL 到期后被过期

结果：
  ✅ Fast Path 任务：低延迟，无过期风险
  ❌ Slow Path 任务：高延迟，有过期风险
  → 相同优先级的任务，仅因为触发时机的微小差异，命运截然不同
```

### 5.3 恢复路径的公平性保障

等待点恢复入队默认走 Slow Path，这在公平性方面是有意为之：

1. **统一排队**：恢复任务和新任务在同一个 sorted set 中竞争
2. **TTL 一致**：恢复任务（如果是首次入队后挂起）也受 TTL 约束
3. **优先级继承**：恢复任务使用原始的 `queueTimestamp - priorityMs`，不获得特殊待遇

这与"恢复不重置 TTL"、"恢复不继承 Fast Path"共同构成了恢复路径的**公平性三原则**。

---

## 六、完整语义串联

### 6.1 TTL 系统全景图

```
任务触发
    │
    ├─ 检查环境类型
    │   ├─ PROD: 不调度 legacy expireRun
    │   └─ DEV:  调度 legacy expireRun（仅触发首入队和 Delayed Run）
    │
    └─ 调用 enqueueRun(includeTtl = true)
            │
            ├─ 计算 ttlExpiresAt = now + parseDuration(ttl)
            │
            └─ 调用 runQueue.enqueueMessage(enableFastPath)
                    │
                    ├─ Lua 脚本决策
                    │   ├─ Fast Path 条件满足？
                    │   │   ├─ 是：RPUSH 到 worker queue，跳过 TTL sorted set
                    │   │   └─ 否：ZADD 到 queue sorted set + ZADD 到 TTL sorted set
                    │   │
                    │   └─ 返回 fastPathTaken
                    │
                    └─ fastPathTaken？
                        ├─ 是：不调度 processQueueForWorkerQueue
                        └─ 否：调度 processQueueForWorkerQueue（500ms debounce）

运行时：
    ├─ TTL 消费者（每 1000ms）：
    │   └─ 扫描 TTL sorted set，过期 Slow Path 任务
    │
    └─ Legacy expireRun 作业（仅 DEV）：
        └─ 在指定时间执行，过期 Fast Path 任务
```

### 6.2 与等待点恢复的交互边界

| 恢复路径特性 | 与 TTL 语义的协同 |
|-------------|-------------------|
| **默认关闭 Fast Path** | 保证恢复任务走 Slow Path，加入 TTL sorted set，TTL 语义一致 |
| **不重置 TTL** | 恢复任务的 TTL 从首次入队开始计算，不因为挂起而延长 |
| **继承 queueTimestamp** | 恢复任务与新任务公平竞争，不插队 |
| **状态分支 skipped** | 已排队/执行中任务不重复入队，避免 TTL 被意外重置 |

### 6.3 关键设计原则总结

1. **TTL 是排队超时，不是生命周期超时**：只对"在队列中等待并发槽"的任务有效
2. **Fast Path 是低延迟优先，公平性其次**：为了 <10ms 的延迟，牺牲部分公平性和 TTL 语义
3. **DEV 环境鲁棒性优先**：DEV 环境 Worker 可能不在线，必须有 legacy expireRun 兜底
4. **PROD 环境效率优先**：PROD 环境 Worker 持续在线，容忍极端边缘情况
5. **恢复路径公平性优先**：恢复任务不走 Fast Path、不重置 TTL、继承原始时间戳

### 6.4 代码位置索引

| 语义点 | 文件位置 | 关键代码 |
|--------|----------|----------|
| Fast Path 跳过 TTL | `run-queue/index.ts:3168-3174` | Lua 脚本中 `-- Skip TTL sorted set` |
| Slow Path 加入 TTL | `run-queue/index.ts:3190-3191` | `ZADD ttlQueueKey` |
| DEV 环境调度 expireRun | `engine/index.ts:806-807` | `environment.type === "DEVELOPMENT"` |
| Delayed Run 调度 expireRun | `delayedRunSystem.ts:151-159` | `environment.type === "DEVELOPMENT"` |
| Legacy expireRun 实现 | `systems/ttlSystem.ts:25-134` | `expireRun()` 方法 |
| TTL 消费者实现 | `run-queue/index.ts:1376-1415` | `#startTtlConsumers()` |
| 恢复路径默认关闭 Fast Path | `waitpointSystem.ts:865-878` | 无 `enableFastPath` 参数 |
