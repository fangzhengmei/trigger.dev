# by-worker keyspace 过期回填路径深度分析

## 1. 回填触发点：`resolveLockedTaskMetadata`

### 1.1 单 slug 回填的设计

```typescript
// apps/webapp/app/runEngine/concerns/queues.server.ts:262-294
private async resolveLockedTaskMetadata(
  workerId: string,
  environmentId: string,
  slug: string
): Promise<TaskMetadataEntry | null> {
  // 1. 先查缓存
  const cached = await this.taskMetaCache.getByWorker(workerId, slug);
  if (cached) return cached;  // ✅ 命中，直接返回

  // 2. 未命中，查 PG (只读副本)
  const row = await this.replicaPrisma.backgroundWorkerTask.findFirst({
    where: { workerId, runtimeEnvironmentId: environmentId, slug },
    select: { ttl: true, triggerSource: true, queue: { select: { id: true, name: true } } },
  });

  if (!row) return null;

  const entry: TaskMetadataEntry = {
    slug,
    ttl: row.ttl,
    triggerSource: row.triggerSource,
    queueId: row.queue?.id ?? null,
    queueName: row.queue?.name ?? "",
  };

  // 3. Fire-and-forget 回填：只写这一个 slug
  void this.taskMetaCache.setByWorker(workerId, entry);

  return entry;
}
```

**为什么只回填单个 slug？**

这是**需求驱动**的设计，而非 bug：

1. **每次触发只需要一个 slug**：`resolveQueueProperties` 是 per-trigger 调用的，每个触发请求只关心一个 task 的元数据（`request.taskId`）。

2. **预取全量是浪费**：如果一个 Worker 有 100 个 task，但当前只触发 3 个，查询并缓存 97 个未使用的 task 是浪费的。

3. **PG 开销可控**：`findFirst` 按主键级索引（`workerId + runtimeEnvironmentId + slug`）查询，非常快。

4. **TTL 侧效果**：每个 slug 的回填都会重置整个 by-worker Hash 的 TTL 为 30 天（见下文）。

---

## 2. `setByWorker` 对 TTL 的影响

### 2.1 单字段写入，全 Hash 续期

```typescript
// apps/webapp/app/services/taskMetadataCache.server.ts:374-389
async setByWorker(workerId: string, entry: TaskMetadataEntry): Promise<void> {
  await this.redis.taskMetaSetFieldRefreshTtl(
    byWorkerKey(workerId),
    String(this.byWorkerTtlSeconds),  // 传入完整的 30 天 (2592000s)
    entry.slug,
    encode(entry)
  );
}
```

调用 `SET_FIELD_REFRESH_TTL_LUA` 脚本：

```lua
-- apps/webapp/app/services/taskMetadataCache.server.ts:176-182
redis.call("HSET", KEYS[1], ARGV[2], ARGV[3])  -- 只写入这一个 field
local ttl = tonumber(ARGV[1])
if ttl and ttl > 0 then
  redis.call("EXPIRE", KEYS[1], ttl)            -- 但！重置整个 Hash 的 TTL
end
```

**关键洞察**：
- `HSET` 只影响**单个 field**（即当前 slug）
- `EXPIRE` 影响**整个 Hash**（即整个 by-worker keyspace）
- **任何一个 slug 的回填都会让整个 Worker 的缓存再活 30 天**

### 2.2 图示

```
初始状态（Hash 已过期被删除）：
  task-meta:by-worker:W1  ← 不存在

触发 task-A 发生 cache miss：
  1. getByWorker(W1, "task-A") → null
  2. PG 查询 task-A
  3. setByWorker(W1, task-A)
     → HSET "task-A" → {...}
     → EXPIRE 2592000
  结果：
    task-meta:by-worker:W1
      ├── "task-A" → {...}
      └── TTL = 30d ← 从这一刻开始倒计时

10 天后，触发 task-B 发生 cache miss：
  1. getByWorker(W1, "task-B") → null (Hash 存在，但 task-B 字段不存在)
  2. PG 查询 task-B
  3. setByWorker(W1, task-B)
     → HSET "task-B" → {...}
     → EXPIRE 2592000  ← 重置！
  结果：
    task-meta:by-worker:W1
      ├── "task-A" → {...}
      ├── "task-B" → {...}
      └── TTL = 30d ← 从这一刻重新开始倒计时！

注意：TTL 被重置，不是从剩余的 20 天继续。
```

---

## 3. 批量触发场景：连续命中序列

### 3.1 单 Worker 多 task 批量触发

**场景**：
- Worker W1 有 task-A、task-B、task-C、task-D
- by-worker Hash 已过期被 Redis 删除
- 批量触发这 4 个 task（使用 `batchTrigger`）

#### 步骤详解

```
初始状态：
  task-meta:by-worker:W1  ← 不存在（TTL 已过期）

───────────────────────────────────────────────

第 1 个 item：task-A
  ├── resolveLockedTaskMetadata(W1, "task-A")
  │   ├── getByWorker → null  ❌ miss
  │   ├── PG findFirst → 得到 task-A 元数据
  │   └── setByWorker(W1, task-A)
  │       → HSET "task-A" → {...}
  │       → EXPIRE 2592000 (30d)
  └── 返回

当前 Redis 状态：
  task-meta:by-worker:W1
    ├── "task-A" → {...}
    └── TTL = 30d

───────────────────────────────────────────────

第 2 个 item：task-B
  ├── resolveLockedTaskMetadata(W1, "task-B")
  │   ├── getByWorker → HGET "task-B" → null  ❌ miss (Hash 存在，但字段不存在)
  │   ├── PG findFirst → 得到 task-B 元数据
  │   └── setByWorker(W1, task-B)
  │       → HSET "task-B" → {...}
  │       → EXPIRE 2592000  ← 重置为 30d！
  └── 返回

当前 Redis 状态：
  task-meta:by-worker:W1
    ├── "task-A" → {...}
    ├── "task-B" → {...}
    └── TTL = 30d  ← 重新开始倒计时

───────────────────────────────────────────────

第 3 个 item：task-C
  ├── resolveLockedTaskMetadata(W1, "task-C")
  │   ├── getByWorker → HGET "task-C" → null  ❌ miss
  │   ├── PG findFirst → 得到 task-C 元数据
  │   └── setByWorker(W1, task-C)
  │       → HSET "task-C" → {...}
  │       → EXPIRE 2592000  ← 再次重置！
  └── 返回

当前 Redis 状态：
  task-meta:by-worker:W1
    ├── "task-A" → {...}
    ├── "task-B" → {...}
    ├── "task-C" → {...}
    └── TTL = 30d  ← 又重新开始

───────────────────────────────────────────────

第 4 个 item：task-D
  ├── resolveLockedTaskMetadata(W1, "task-D")
  │   ├── getByWorker → HGET "task-D" → null  ❌ miss
  │   ├── PG findFirst → 得到 task-D 元数据
  │   └── setByWorker(W1, task-D)
  │       → HSET "task-D" → {...}
  │       → EXPIRE 2592000  ← 第 4 次重置！
  └── 返回

最终 Redis 状态：
  task-meta:by-worker:W1
    ├── "task-A" → {...}
    ├── "task-B" → {...}
    ├── "task-C" → {...}
    ├── "task-D" → {...}
    └── TTL = 30d  ← 从第 4 个 item 的 setByWorker 时间开始
```

### 3.2 Miss 序列模式

对于一个过期的 by-worker Hash，连续触发 N 个不同 slug 会产生：

| 触发顺序 | 操作 | Cache 结果 | PG 查询 | TTL 变化 |
|---------|------|-----------|---------|---------|
| 1st (task-A) | getByWorker + PG + setByWorker | ❌ miss | ✅ 1 次 | 无 → 30d |
| 2nd (task-B) | getByWorker + PG + setByWorker | ❌ miss | ✅ 1 次 | 剩余 30d → 重置为 30d |
| 3rd (task-C) | getByWorker + PG + setByWorker | ❌ miss | ✅ 1 次 | 剩余 30d → 重置为 30d |
| 4th (task-D) | getByWorker + PG + setByWorker | ❌ miss | ✅ 1 次 | 剩余 30d → 重置为 30d |
| ... (重复 task-A) | getByWorker | ✅ hit | ❌ 0 次 | 不变 |

**关键点**：
- 每个**新 slug** 第一次触发都会 miss 并触发一次 PG 查询
- 同一 slug 第二次触发就会 hit
- 每个新 slug 的回填都会**重置整个 Hash 的 TTL**
- N 个不同 slug → N 次 PG 查询 + N 次 TTL 重置

### 3.3 批量处理的实际路径

```typescript
// apps/webapp/app/runEngine/services/batchTrigger.server.ts:537-601
for (const item of itemsToProcess) {
  // 每个 item 单独调用 TriggerTaskService.call()
  const run = await this.#processBatchTaskRunItem({
    batch, environment, item, ...
  });
}

// #processBatchTaskRunItem 内部：
async #processBatchTaskRunItem(...) {
  const triggerTaskService = new TriggerTaskService();
  const result = await triggerTaskService.call(
    item.task,     // 每个 item 的 task 可能不同
    environment,
    {...},
    {...},
    "V2"
  );
}

// TriggerTaskService.call 内部最终调用：
await defaultQueueManager.resolveQueueProperties(request, lockedBackgroundWorker);
// → 如果有 lockedBackgroundWorker，调用 resolveLockedTaskMetadata
// → 每个 item 的每个 task 单独查缓存、单独回填
```

**串行处理**：由于 `PROCESSING_BATCH_SIZE = 50` 但策略是 "sequential"，这 50 个 item 是串行处理的，每个 miss 都会产生一次单独的 PG 查询。

---

## 4. 重复触发场景：已回填 slug 的二次访问

### 4.1 同一 slug 短时间内重复触发

```
t=0: 触发 task-A (miss) → PG + setByWorker → TTL=30d
t=5min: 触发 task-A 再次 → getByWorker hit ✅ → 0 PG，TTL 不变
t=10min: 触发 task-A 再次 → hit ✅ → 0 PG，TTL 不变
```

### 4.2 交叉触发

```
t=0:    触发 task-A (miss) → Hash 建立，TTL=30d
t=5min: 触发 task-B (miss) → 回填，TTL 重置为 30d
t=10min: 触发 task-A 再次 → hit ✅，TTL 继续从 29d55min 倒计时
t=15min: 触发 task-C (miss) → 回填，TTL 重置为 30d  ← 又重置了！
```

**重要**：`getByWorker` 命中时 **不刷新 TTL**，只有 `setByWorker` 才刷新。

这意味着：如果一个 Worker 有 N 个 task，每个 task 每 29 天被触发一次，整个 Hash 永远不会过期（因为每次新 task 的触发都会重置 TTL）。

---

## 5. 与 `setByCurrentWorker` 的对比

`setByCurrentWorker` 是非锁定路径的回填方法，它也会回填 by-worker keyspace：

```typescript
// apps/webapp/app/runEngine/concerns/queues.server.ts:341
void this.taskMetaCache.setByCurrentWorker(environment.id, worker.id, entry);
```

调用 `SET_TWO_FIELDS_LUA`：

```lua
-- apps/webapp/app/services/taskMetadataCache.server.ts:207-226
-- by-worker 侧：无条件写入 + 重置 TTL
redis.call("HSET", KEYS[2], ARGV[4], ARGV[5])
local workerTtl = tonumber(ARGV[2])
if workerTtl and workerTtl > 0 then
  redis.call("EXPIRE", KEYS[2], workerTtl)  -- 重置为完整 30 天
end
```

**效果**：非锁定路径的回填同样会重置 by-worker Hash 的 TTL。

这意味着：
- 锁定触发的回填会重置 TTL
- 非锁定触发的回填也会重置 TTL
- 只要有任何触发（锁定或非锁定）导致 cache miss 并回填，TTL 就会重置

---

## 6. 极端场景分析

### 6.1 场景一：温缓存部分过期

**不可能发生**。Redis 的 TTL 是 key 级别的，不是 field 级别的。整个 Hash 要么存在（所有 field 都存在），要么不存在（所有 field 都消失）。

不能出现 "task-A 还在缓存里，但 task-B 已经过期了" 的情况。

### 6.2 场景二：Hash 存在但 field 不存在

**可能发生**。例如：
1. 触发 task-A → 回填 task-A → Hash 建立（只有 task-A）
2. 触发 task-B → Hash 存在，但 field "task-B" 不存在 → miss → 回填 task-B

这是"部分填充"的 Hash，不是"部分过期"。

### 6.3 场景三：长周期零散触发

```
Day 0:   部署 W1 → populateByWorker → 所有 task 都在缓存，TTL=30d

Day 1-29: 没有任何触发 → TTL 倒计时

Day 30:  TTL 归零，Redis 删除整个 Hash

Day 31:  触发 task-A → miss → PG + setByWorker → TTL 重置为 30d
         Hash 里现在只有 task-A

Day 61:  TTL 归零，Redis 删除

Day 62:  触发 task-B → miss → PG + setByWorker → TTL 重置为 30d
         Hash 里现在只有 task-B
```

**每次过期后重建时，只会回填被触发的那些 slug**。如果 Worker 有 100 个 task，但只有 5 个被触发，那么缓存里只有这 5 个。

---

## 7. 设计权衡总结

### 7.1 单 slug 回填的优点

1. **按需加载**：只缓存实际被使用的 task，节省内存
2. **实现简单**：不需要批量查询和批量写入逻辑
3. **PG 开销低**：`findFirst` 按主键查询非常快
4. **自动续期**：任何一个新 slug 的触发都会重置整个 Hash 的 TTL，活跃 Worker 不会过期

### 7.2 单 slug 回填的缺点

1. **批量触发时 N+1 查询**：N 个不同的 task → N 次 PG 查询
2. **Hash 重建时逐步填充**：过期后第一次触发 N 个 task，要经历 N 次 miss
3. **内存碎片**：Hash 可能处于"部分填充"状态（虽然 Redis 内部管理得很好）

### 7.3 改进空间（如果需要）

如果批量场景很多，可以考虑：

**方案：批量回填优化**

```typescript
// 假想的优化实现
async resolveLockedTaskMetadataBatch(
  workerId: string,
  environmentId: string,
  slugs: string[]
): Promise<Map<string, TaskMetadataEntry>> {
  // 1. HMGET 批量查缓存
  const cached = await this.taskMetaCache.getByWorkerBatch(workerId, slugs);

  // 2. 收集未命中的 slugs
  const misses = slugs.filter(s => !cached.has(s));

  if (misses.length > 0) {
    // 3. 批量查 PG
    const rows = await this.replicaPrisma.backgroundWorkerTask.findMany({
      where: { workerId, runtimeEnvironmentId: environmentId, slug: { in: misses } },
    });

    // 4. HMSET 批量回填
    await this.taskMetaCache.setByWorkerBatch(workerId, rows);

    // 5. 合并结果
    for (const row of rows) {
      cached.set(row.slug, toEntry(row));
    }
  }

  return cached;
}
```

**但当前没有这样做的原因**：
- 批量触发的情况相对较少
- 单次 `findFirst` 已经很快（主键查询 < 1ms）
- 代码复杂度与性能收益不成正比

---

## 8. 完整流程图

```
锁定版本触发路径：

  trigger(..., { lockToVersion: "20250313.1" })
      │
      ▼
  resolveQueueProperties(request, lockedBackgroundWorker)
      │
      ▼
  resolveLockedTaskMetadata(workerId, envId, slug)
      │
      ├── getByWorker(workerId, slug)
      │   └── HGET task-meta:by-worker:{workerId} "{slug}"
      │
      ├── 命中 → 直接返回 ✅ (0 PG)
      │
      └── 未命中 →
          │
          ├── PG: backgroundWorkerTask.findFirst({ workerId, slug })
          │   └── 得到单条记录
          │
          ├── setByWorker(workerId, entry)  ← fire-and-forget
          │   └── Lua: SET_FIELD_REFRESH_TTL_LUA
          │       ├── HSET "{slug}" → {...}  ← 只写这一个 field
          │       └── EXPIRE 2592000        ← 但重置整个 Hash 的 TTL
          │
          └── 返回 entry

非锁定触发路径（也会写 by-worker）：

  trigger()
      │
      ▼
  resolveCurrentTaskMetadata(env, slug)
      │
      ├── 未命中 → PG 查询
      │
      └── setByCurrentWorker(envId, workerId, entry)  ← fire-and-forget
          └── Lua: SET_TWO_FIELDS_LUA
              ├── by-worker 侧：HSET + EXPIRE  ← 同样重置 TTL
              └── env 侧：CAS + HSET
```

---

## 9. 关键结论

1. **单 slug 回填是有意设计**：每次触发只需要一个 task 的元数据，按需加载最经济。

2. **TTL 是 key 级的，不是 field 级的**：
   - 整个 Hash 要么存在，要么不存在
   - 任何一个 field 的写入都会重置整个 key 的 TTL
   - 读取命中不刷新 TTL

3. **过期后的 miss 序列**：
   - N 个不同 task 连续触发 → N 次 cache miss + N 次 PG 查询
   - 每个 miss 都会重置整个 Hash 的 TTL 为 30 天
   - 同一 task 第二次触发就会 hit

4. **活跃 Worker 永不过期**：只要 30 天内至少有一个 task 触发（并触发一次回填），TTL 就会被重置，整个 Hash 就不会过期。
