# TaskMetadataCache TTL 续期边界与时效问题分析

## 1. TTL 配置参数

```typescript
// apps/webapp/app/services/taskMetadataCache.server.ts:270-271
this.currentEnvTtlSeconds = options.currentEnvTtlSeconds ?? 86400;       // 24 小时
this.byWorkerTtlSeconds = options.byWorkerTtlSeconds ?? 30 * 24 * 60 * 60;  // 30 天
```

| Keyspace | 默认 TTL | 设计意图 |
|----------|---------|---------|
| `task-meta:env:{envId}` | 24 小时 | 兜底安全网，跟随 Promotion 走 |
| `task-meta:by-worker:{workerId}` | 30 天 | "滑动过期"，活跃 Worker 应常驻 |

> ⚠️ 关键注意：注释写着 "Idle TTL on by-worker"，但实际实现**并不是**真正的"空闲过期"。

---

## 2. 各方法对 TTL 的实际影响

### 2.1 `getByWorker()` —— 只读，不续期

```typescript
// apps/webapp/app/services/taskMetadataCache.server.ts:295-297
async getByWorker(workerId: string, slug: string): Promise<TaskMetadataEntry | null> {
  return this.#get(byWorkerKey(workerId), slug);
}

async #get(key: string, slug: string): Promise<TaskMetadataEntry | null> {
  try {
    const raw = await this.redis.hget(key, slug);  // 纯读取
    if (!raw) return null;
    return decode(slug, raw);
  } catch (error) {
    return null;
  }
}
```

**TTL 影响**：`getByWorker` 只执行 `HGET`，**完全不触碰 TTL**。

### 2.2 `setByWorker()` —— 强制重置 TTL（每次写入都续）

```typescript
// apps/webapp/app/services/taskMetadataCache.server.ts:374-389
async setByWorker(workerId: string, entry: TaskMetadataEntry): Promise<void> {
  await this.redis.taskMetaSetFieldRefreshTtl(
    byWorkerKey(workerId),
    String(this.byWorkerTtlSeconds),  // 传入完整的 30 天
    entry.slug,
    encode(entry)
  );
}
```

调用 `SET_FIELD_REFRESH_TTL_LUA` 脚本：

```lua
-- apps/webapp/app/services/taskMetadataCache.server.ts:176-182
redis.call("HSET", KEYS[1], ARGV[2], ARGV[3])
local ttl = tonumber(ARGV[1])
if ttl and ttl > 0 then
  redis.call("EXPIRE", KEYS[1], ttl)  -- 无条件重置 TTL
end
```

**TTL 影响**：每次 `setByWorker` 都会把 by-worker Hash 的 TTL **重置为完整的 30 天**。

### 2.3 `setByCurrentWorker()` —— by-worker 侧重置，env 侧有条件重置

```typescript
// apps/webapp/app/services/taskMetadataCache.server.ts:349-372
async setByCurrentWorker(envId, workerId, entry): Promise<void> {
  await this.redis.taskMetaSetTwoFields(
    currentEnvKey(envId),
    byWorkerKey(workerId),
    String(this.currentEnvTtlSeconds),
    String(this.byWorkerTtlSeconds),
    workerId,
    entry.slug,
    encode(entry)
  );
}
```

调用 `SET_TWO_FIELDS_LUA` 脚本：

```lua
-- apps/webapp/app/services/taskMetadataCache.server.ts:207-226
-- 1. by-worker 侧：无条件写入 + 重置 TTL
redis.call("HSET", KEYS[2], ARGV[4], ARGV[5])
local workerTtl = tonumber(ARGV[2])
if workerTtl and workerTtl > 0 then
  redis.call("EXPIRE", KEYS[2], workerTtl)  -- 重置为完整 30 天
end

-- 2. env 侧：CAS 后有条件写入 + 有条件设置 TTL
local owner = redis.call("HGET", KEYS[1], "__owner_worker_id")
if owner == false or owner == ARGV[3] then
  redis.call("HSET", KEYS[1], ARGV[4], ARGV[5])
  ...
  local envTtl = tonumber(ARGV[1])
  -- ⚠️ 只有当 Hash 无过期时才设置 TTL（-1 = no expiry）
  if envTtl and envTtl > 0 and redis.call("TTL", KEYS[1]) == -1 then
    redis.call("EXPIRE", KEYS[1], envTtl)
  end
end
```

**TTL 影响**：
- by-worker 侧：**每次都重置为 30 天**（同 `setByWorker`）
- env 侧：只有当 `TTL == -1`（即当前无过期）时才设置 24h TTL；已存在 TTL 时**不覆盖、不刷新**

> 为什么 env 侧不刷新？Promotion 时已经设置了 TTL，回填路径不应该重置这个倒计时。否则频繁回填的 env 可能永远不会过期，失去了兜底过期的意义。

### 2.4 `populateByWorker()` —— 删除重建后设置 TTL

```typescript
// apps/webapp/app/services/taskMetadataCache.server.ts:329-347
async populateByWorker(workerId: string, entries: TaskMetadataEntry[]): Promise<void> {
  await this.redis.taskMetaReplaceHash(
    byWorkerKey(workerId),
    String(this.byWorkerTtlSeconds),
    ...fieldValues
  );
}
```

调用 `REPLACE_HASH_LUA` 脚本：

```lua
-- apps/webapp/app/services/taskMetadataCache.server.ts:103-117
redis.call("DEL", KEYS[1])               -- 先删除
if #ARGV > 1 then
  redis.call("HSET", KEYS[1], unpack(fv))  -- 重建
end
local ttl = tonumber(ARGV[1])
if ttl and ttl > 0 then
  redis.call("EXPIRE", KEYS[1], ttl)      -- 设置完整 TTL
end
```

**TTL 影响**：by-worker Hash TTL 重置为完整的 30 天。

### 2.5 `populateByCurrentWorker()` —— 双 Hash 都重置 TTL

```typescript
// apps/webapp/app/services/taskMetadataCache.server.ts:299-327
async populateByCurrentWorker(envId, workerId, entries): Promise<void> {
  await this.redis.taskMetaReplaceTwoHashes(
    currentEnvKey(envId),
    byWorkerKey(workerId),
    String(this.currentEnvTtlSeconds),
    String(this.byWorkerTtlSeconds),
    workerId,
    ...fieldValues
  );
}
```

调用 `REPLACE_TWO_HASHES_LUA` 脚本：

```lua
-- apps/webapp/app/services/taskMetadataCache.server.ts:144-165
redis.call("DEL", KEYS[1])
redis.call("DEL", KEYS[2])
-- ... HSET ...
local envTtl = tonumber(ARGV[1])
if envTtl and envTtl > 0 then
  redis.call("EXPIRE", KEYS[1], envTtl)    -- env 重置为 24h
end
local workerTtl = tonumber(ARGV[2])
if workerTtl and workerTtl > 0 then
  redis.call("EXPIRE", KEYS[2], workerTtl)  -- by-worker 重置为 30d
end
```

**TTL 影响**：env Hash 重置为 24 小时，by-worker Hash 重置为 30 天。

---

## 3. TTL 影响汇总表

| 方法 | env keyspace TTL 变化 | by-worker keyspace TTL 变化 |
|------|---------------------|---------------------------|
| `getCurrent()` | 无变化（只读） | N/A |
| `getByWorker()` | N/A | 无变化（只读）⚠️ |
| `populateByCurrentWorker()` | **重置为完整 24h** | **重置为完整 30d** |
| `populateByWorker()` | N/A | **重置为完整 30d** |
| `setByCurrentWorker()` | 仅当 TTL==-1 时设置 24h（不覆盖） | **重置为完整 30d** |
| `setByWorker()` | N/A | **重置为完整 30d** |

---

## 4. 核心问题：命中缓存却不续期的时效问题

### 4.1 问题根因

```
注释描述（Idling TTL）：每次访问都应该刷新 TTL，空闲才过期
实际实现（Fixed TTL）：只有写操作才刷新 TTL，读操作不刷新
```

这意味着：**by-worker keyspace 的 TTL 不是真正的滑动过期（sliding expiry），而是固定过期（fixed expiry）**。

### 4.2 问题场景复现

**场景：一个 Worker 版本被锁定版本的任务持续命中缓存，持续触发 31 天**

```
Day 0:
  Worker W1 创建
  populateByWorker() → TTL = 30 天
  task-meta:by-worker:W1 倒计时开始：30d → 29d → ...

Day 1 to Day 29:
  每次锁定版本触发 W1 的某个 task
  getByWorker(W1, "my-task") → HGET 命中 ✅
  ⚠️ 但！TTL 倒计时继续减少：29d → 28d → ... → 1d
  没有写操作，TTL 不会刷新

Day 30:
  TTL 倒计时归零
  Redis 自动删除 task-meta:by-worker:W1 整个 Hash

Day 30 + 1 minute:
  同样的锁定版本触发同一个 task
  getByWorker(W1, "my-task") → HGET 未命中 ❌
  → PG 查询回填
  → setByWorker() → TTL 重置为 30 天
  (性能退化：从 0 DB 变为 1 DB 查询)
```

### 4.3 问题严重程度评估

| 维度 | 评估 |
|------|------|
| 正确性影响 | **无** — 缓存缺失只是回退到 PG 查询，不会出错 |
| 性能影响 | **低** — 每个 Worker 每 30 天最多 miss 一次，且 miss 后立即回填 |
| 存储影响 | **可能有** — 不活跃的旧 Worker 版本会占用内存 30 天 |
| 实际触发条件 | 需要连续 30 天每天都有任务触发同一个 Worker 版本（跨版本部署周期通常短于 30 天） |

### 4.4 更可能出现的边界场景

**真实场景：跨月回滚**

```
Day 0:  部署版本 W1（生产环境）
        populateByCurrentWorker → env Hash + by-worker Hash 都设 TTL

Day 15: 部署版本 W2
        Promotion → env Hash 切到 W2（TTL 重置 24h）
                  → W2 by-worker Hash TTL 30d
        W1 by-worker Hash 倒计时剩余 15 天

Day 30: W1 by-worker Hash TTL 归零 → Redis 删除

Day 35: 发现 W2 有问题，需要回滚到 W1
        Promotion W1 → populateByCurrentWorker
        → env Hash 重建（没问题，PG 有数据）
        → W1 by-worker Hash 重建（没问题，PG 有数据）
        ✅ 不影响回滚
```

**虽然 by-worker Hash 已被删除，但 Promotion 时会重新 populate，不影响正确性**。

### 4.5 另一个边角场景

```
Day 0:  部署 W1

Day 29: W1 by-worker Hash TTL 还剩 1 天
        某个锁定到 W1 的 task cache miss（比如之前没被触发过）
        setByWorker() → TTL 重置为 30 天

Day 30: Promotion W2，env Hash 切走
        W1 by-worker Hash 现在有 29 天 TTL

Day 59: W1 by-worker Hash 终于过期
```

被回填的那个操作"意外"延长了整个 by-worker Hash 的寿命 29 天。但这不是 bug，只是内存占用时间更长一点而已。

---

## 5. 为什么这么设计？不做 read refresh 的权衡

### 5.1 如果每次 getByWorker 都刷新 TTL

```lua
-- 假设的实现：HGET + EXPIRE
local val = redis.call("HGET", KEYS[1], ARGV[1])
if val then
  redis.call("EXPIRE", KEYS[1], 2592000)  -- 每次读都写 TTL
end
return val
```

**开销**：每次 `getByWorker` 从纯读变成读 + 写（EXPIRE 是写命令），在高并发下：
1. Redis 写放大（所有触发请求都产生写操作）
2. 主从复制流量增加
3. AOF 持久化压力

### 5.2 当前设计的权衡

| 方案 | 每次触发的 Redis 开销 | 30 天节点的 cache miss 率 |
|------|---------------------|------------------------|
| 读后刷新 TTL | HGET + EXPIRE (读写混合) | 0% (理论上) |
| 当前实现 | HGET (纯读) | 每个 Worker 每 30 天最多 1 次 miss |

**设计决策**：接受极低概率的 cache miss（30 天一次），换取纯读的高性能。

### 5.3 注释与实现的偏差

注释写的是 "Idle TTL on by-worker"，但实现不是真正的空闲过期。这是一个**注释不准确**，但**实现是有意为之**的设计选择。

真正的空闲过期应该是：
```
每次访问 → 重置 TTL
超过 TTL 时间无访问 → 过期
```

当前实现是：
```
每次写操作 → 重置 TTL
写操作后 TTL 时间内不管有没有读 → 都会过期
```

---

## 6. env keyspace 的 TTL 设计意图

env keyspace 采用**固定 24h 不刷新**的设计，有明确的设计理由：

1. **Promotion 后必须有机会过期**：如果某次 Promotion 后 `populateByCurrentWorker` 成功了，但系统后来出问题没再 Promotion，env keyspace 应该在 24h 后过期（兜底机制）。

2. **回填路径不应该干扰兜底机制**：`setByCurrentWorker` 中 env 侧的 TTL 只有在 `TTL == -1` 时才设置，就是为了不重置 Promotion 时设置的 24h 倒计时。

3. **env keyspace 的 TTL 只是安全网**：正常情况下每 24h 内至少有一次 Promotion（或者缓存过期后触发的回填会重建），所以不会频繁 miss。

---

## 7. 结论与改进空间

### 7.1 当前设计的合理性

**当前设计是合理的**：
- 正确性：没有问题，cache miss 只是回退到 PG 查询
- 性能：接受极低概率（每 Worker 每 30 天一次）的性能退化
- 资源：TTL 机制保证了不活跃的旧版本最终会被清理

### 7.2 如果要实现真正的滑动过期

如果确实需要 "idle TTL" 语义，可以考虑以下两种修改：

**方案一：在 getByWorker 中增加 EXPIRE（简单直接）**

```typescript
async getByWorker(workerId: string, slug: string): Promise<TaskMetadataEntry | null> {
  try {
    const key = byWorkerKey(workerId);
    const raw = await this.redis.hget(key, slug);
    if (!raw) return null;
    // 命中后刷新 TTL
    await this.redis.expire(key, this.byWorkerTtlSeconds);
    return decode(slug, raw);
  } catch (error) {
    return null;
  }
}
```

缺点：纯读变读写混合，增加 Redis 写压力。

**方案二：用 Lua 脚本原子化 HGET + EXPIRE（性能更好）**

```lua
local val = redis.call("HGET", KEYS[1], ARGV[1])
if val then
  local ttl = tonumber(ARGV[2])
  if ttl and ttl > 0 then
    redis.call("EXPIRE", KEYS[1], ttl)
  end
end
return val
```

缺点：需要新增一个自定义命令，增加复杂度。

### 7.3 风险极低的改进

**最低风险的改进**：只改注释，不改代码。把 "Idle TTL" 的注释改为更准确的描述：

```typescript
/**
 * Fixed TTL on `task-meta:by-worker:{workerId}`. Default 30d.
 * Reset on write operations only; read hits do not refresh the TTL.
 * This is a deliberate trade-off: we accept a once-per-30-days cache
 * miss per worker to avoid EXPIRE write traffic on every read.
 */
byWorkerTtlSeconds?: number;
```

---

## 8. 完整的 TTL 生命周期图

```
Worker W1 生命周期中的 TTL 变化：

  0d          populateByWorker() ──► TTL = 30d
                (V4 构建建 Worker)
              │
              │
 15d          setByWorker() ──► TTL 重置为 30d
                (cache miss 回填)
              │
              │
 30d          Promotion W2 ──► env Hash 切走
                W1 by-worker TTL 继续倒计时
              │
              │
 45d          TTL 归零
              Redis 删除 task-meta:by-worker:W1
              │
              │
 50d          Promotion 回滚 W1 ──► populateByCurrentWorker()
                                    TTL 重置为 30d (从 PG 重建)

  注：期间每次 cache hit 不改变 TTL 倒计时
```
