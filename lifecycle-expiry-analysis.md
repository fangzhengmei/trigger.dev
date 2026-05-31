# by-worker keyspace 缓存生命周期与过期机制深度分析

## 1. 核心机制的代码确认

### 1.1 只读路径：`getByWorker` —— 纯 HGET，不触碰 TTL

```typescript
// apps/webapp/app/services/taskMetadataCache.server.ts:295-297
async getByWorker(workerId: string, slug: string): Promise<TaskMetadataEntry | null> {
  return this.#get(byWorkerKey(workerId), slug);
}

// apps/webapp/app/services/taskMetadataCache.server.ts:391-400
async #get(key: string, slug: string): Promise<TaskMetadataEntry | null> {
  try {
    const raw = await this.redis.hget(key, slug);  // 只有 HGET！
    if (!raw) return null;
    return decode(slug, raw);
  } catch (error) {
    logger.error("Failed to read task metadata from cache", { key, slug, error });
    return null;
  }
}
```

**铁证**：`#get` 方法内部只有 `hget`，**没有任何 EXPIRE 调用**。

> ✅ 结论：缓存命中时，TTL 倒计时**完全不受影响**，继续减少。

---

### 1.2 写入路径：才会重置 TTL

所有会重置 TTL 的方法：

| 方法 | Lua 脚本 | TTL 操作 |
|------|---------|---------|
| `populateByWorker()` | `REPLACE_HASH_LUA` | DEL + HSET + EXPIRE 重置为 30d |
| `populateByCurrentWorker()` | `REPLACE_TWO_HASHES_LUA` | DEL + HSET + EXPIRE 重置为 30d |
| `setByWorker()` | `SET_FIELD_REFRESH_TTL_LUA` | HSET 单 field + EXPIRE 重置为 30d |
| `setByCurrentWorker()` | `SET_TWO_FIELDS_LUA` | by-worker 侧 HSET + EXPIRE 重置为 30d |

**共同点**：这些方法都执行了 `EXPIRE key 2592000`，将 TTL **重置为完整的 30 天**。

---

## 2. 边界条件的精确描述

### 2.1 TTL 续期的触发边界

**只有以下事件能让 by-worker Hash 的 TTL "续命"：**

| 场景 | 触发方法 | TTL 变化 |
|------|---------|---------|
| 1. Worker 创建时预填充 | `populateByWorker()` 或 `populateByCurrentWorker()` | 从无到有，设置为 30d |
| 2. Promotion 时重建 | `populateByCurrentWorker()` | 重置为 30d |
| 3. Cache miss 回填（锁定路径） | `setByWorker()` | 重置为 30d |
| 4. Cache miss 回填（非锁定路径） | `setByCurrentWorker()` | by-worker 侧重置为 30d |

### 2.2 TTL 不续期的边界

**以下事件**不影响**TTL 倒计时：**

| 场景 | 触发方法 | TTL 变化 |
|------|---------|---------|
| 1. 锁定触发 cache hit | `getByWorker()` 命中 | 倒计时继续减少 |
| 2. 非锁定触发 cache hit | `getCurrent()` 命中 | by-worker key 根本不被访问 |
| 3. Redis 内部清理 | — | TTL 归零时删除 |

---

## 3. 长期只命中场景：为什么会过期？

### 3.1 问题核心

```
问题：如果一个 Worker 版本被频繁触发，但每次都是 cache hit（没有任何 miss），
      30 天后缓存会过期吗？

答案：会。
```

**原因**：每次命中只调用 `getByWorker` → 只有 `HGET`，没有 `EXPIRE` 刷新 TTL。

TTL 是从最后一次**写入**开始倒计时的，不是从最后一次**读取**。

### 3.2 完整的时间推进序列

**前置条件**：
- Worker W1 在 Day 0 创建，包含 task-A
- 从 Day 0 开始，每天触发 task-A 100 次
- 每次触发都锁定到 W1 版本
- 期间没有任何 cache miss（因为第一次触发后就回填了）

**按时间推进的详细序列**：

```
─────────────────────────────────────────────────────────────────────
Day 0, 10:00 AM
─────────────────────────────────────────────────────────────────────
  事件：Worker W1 创建 + populateByWorker()
  Redis 操作：
    DEL task-meta:by-worker:W1
    HSET "task-A" → {...}
    EXPIRE 2592000  ← TTL 设置为 30 天
  状态：
    task-meta:by-worker:W1
      ├── "task-A" → {...}
      └── TTL = 30d (2592000s) ← 开始倒计时

  当天后续触发 task-A 100 次：
    每次 → getByWorker(W1, "task-A") → HGET → 命中 ✅
    但！TTL 继续减少：30d → 29d 23h → ... → 29d
  当天结束：TTL ≈ 29 天

─────────────────────────────────────────────────────────────────────
Day 1, 全天
─────────────────────────────────────────────────────────────────────
  触发 task-A 100 次
    每次 → getByWorker(W1, "task-A") → 命中 ✅
    TTL 继续减少：29d → 28d
  当天结束：TTL ≈ 28 天

─────────────────────────────────────────────────────────────────────
Day 2 - Day 28, 每天
─────────────────────────────────────────────────────────────────────
  每天触发 task-A 100 次
    每次 → getByWorker(W1, "task-A") → 命中 ✅
    但 TTL 持续减少：28d → 27d → ... → 2d
  Day 28 结束：TTL ≈ 2 天

─────────────────────────────────────────────────────────────────────
Day 29, 全天
─────────────────────────────────────────────────────────────────────
  触发 task-A 100 次
    每次 → getByWorker(W1, "task-A") → 命中 ✅
    TTL 继续减少：2d → 1d
  当天结束：TTL ≈ 1 天

─────────────────────────────────────────────────────────────────────
Day 30, 09:59 AM
─────────────────────────────────────────────────────────────────────
  触发 task-A → getByWorker → 命中 ✅
  此时 TTL ≈ 60 秒

─────────────────────────────────────────────────────────────────────
Day 30, 10:00 AM （创建满 30 天的那一刻）
─────────────────────────────────────────────────────────────────────
  事件：Redis 检测到 TTL 归零，自动删除整个 Hash
  状态：task-meta:by-worker:W1 ← 不存在了

─────────────────────────────────────────────────────────────────────
Day 30, 10:01 AM
─────────────────────────────────────────────────────────────────────
  触发 task-A
    ├── getByWorker(W1, "task-A")
    │   └── HGET → null  ❌ miss（Hash 不存在了）
    │
    ├── PG 查询 backgroundWorkerTask → 得到 task-A 元数据
    │
    ├── setByWorker(W1, task-A)  ← fire-and-forget 回填
    │   ├── HSET "task-A" → {...}
    │   └── EXPIRE 2592000  ← TTL 重新设置为 30 天
    │
    └── 返回结果

  状态：
    task-meta:by-worker:W1
      ├── "task-A" → {...}
      └── TTL = 30d ← 从这一刻重新开始倒计时

  ⚠️ 关键：这一次触发产生了 1 次 PG 查询，性能退化。
```

### 3.3 序列总结表

| 时间点 | 事件 | 方法调用 | Cache 结果 | PG 查询 | TTL 状态 |
|--------|------|---------|-----------|---------|---------|
| Day 0, 10:00 | Worker 创建 | populateByWorker() | N/A | 0 次 | 设置为 30d |
| Day 0, 10:01 | 第 1 次触发 task-A | getByWorker() | ✅ hit | 0 次 | 30d → 29d 23:59 |
| Day 1-29, 每天 | 每天 100 次触发 | getByWorker() | ✅ hit | 0 次 | 持续减少 |
| Day 30, 09:59 | 触发 | getByWorker() | ✅ hit | 0 次 | ≈ 60s |
| Day 30, 10:00 | Redis 过期 | — | — | — | 归零，Hash 被删除 |
| Day 30, 10:01 | 触发 | getByWorker() + setByWorker() | ❌ miss | 1 次 | 重置为 30d |
| Day 30, 10:02 | 触发 | getByWorker() | ✅ hit | 0 次 | 30d → 29d 23:59 |

---

## 4. 影响评估

### 4.1 性能影响

**影响极低**：
- 每 Worker 每 30 天最多产生 1 次额外的 PG 查询
- PG 查询是主键级的 `findFirst`，通常 < 1ms
- 回填是 fire-and-forget，不阻塞请求返回
- 回填后立即恢复 0 PG 查询状态

### 4.2 什么时候影响变大？

如果系统中有 **N 个不活跃的旧版本 Worker**，每个版本每 30 天会产生至少 1 次 PG 查询（如果还有触发的话）。但考虑到：
1. 部署周期通常短于 30 天（旧版本会被新版本替代，不再被触发）
2. 回滚场景罕见
3. `findFirst` 开销极低

实际影响可以忽略。

---

## 5. 与 "真正滑动过期" 的对比

### 5.1 当前实现（Fixed TTL on Write）

```
写操作 → 重置 TTL
读操作 → 不碰 TTL
结果：从最后一次写开始倒计时 30 天
```

**优点**：
- 纯读，Redis 开销最小
- 实现简单

**缺点**：
- 即使一直被读，30 天后还是会过期

### 5.2 理想滑动过期（Sliding TTL on Read）

```
写操作 → 重置 TTL
读操作 → 重置 TTL
结果：从最后一次访问（读或写）开始倒计时 30 天
只有连续 30 天没人访问才过期
```

**优点**：
- 真正的"热数据永不过期"
- 活跃数据不会产生周期性 miss

**缺点**：
- 读变成读写混合（HGET + EXPIRE）
- 每次触发都产生 EXPIRE 写命令
- Redis 主从复制 + AOF 持久化压力增加

### 5.3 为什么 Trigger.dev 选择当前实现

```
权衡决策：
  接受「每 30 天 1 次 PG 查询」的微小代价
  换取「每次触发都是纯读」的高性能
```

这是典型的"性能优先、退化极小"的工程权衡。

---

## 6. 代码注释与实际行为的偏差

### 6.1 注释写的是 "Idle TTL"

```typescript
// apps/webapp/app/services/taskMetadataCache.server.ts:51-52
/** Idle TTL on `task-meta:by-worker:{workerId}`. Default 30d. Use 0 for no expiry. */
byWorkerTtlSeconds?: number;
```

"Ideally TTL" 的标准定义是：**超过 TTL 时间没有任何访问（读或写）才过期**。

### 6.2 实际行为不是 Idle TTL

实际行为是：**超过 TTL 时间没有任何写操作就过期**，不管有没有读。

### 6.3 这是注释不准确，不是 bug

代码行为是有意的设计选择，注释应该被修正为更准确的描述。

---

## 7. 完整生命周期图示

```
                              Day 0        Day 15       Day 30      Day 30+1min
                                │            │            │            │
                                ▼            ▼            ▼            ▼
TTL 倒计时：  30d ◄───────────► 15d ◄───────► 0d ◄───────► 30d
                                │            │            │            │
                                │            │            │            │
Cache 状态：  [创建]         [持续命中]   [过期删除]   [回填重建]
                                │            │            │            │
                                │            │            │            │
写入事件：    populateByWorker  │            │            │  setByWorker
                              无写入        无写入        无写入
                                │            │            │            │
                                ▼            ▼            ▼            ▼
读命中次数：   ───────── 每天 100 次 HIT ────────►  MISS  ──► 继续 HIT
```

**关键点标注**：
- Day 0 到 Day 30 之间：共 3000 次触发，全部命中，0 PG 查询
- 但 TTL 一直在减少，因为没有写入
- Day 30 TTL 归零：缓存被删除
- Day 30 + 1min：第 3001 次触发 miss，产生 1 次 PG 查询，然后回填，TTL 重置
- 之后的 30 天：再次进入"持续命中"循环

---

## 8. 结论

1. **长期只命中仍会过期是设计行为**：`getByWorker` 只做纯读，不刷新 TTL；TTL 只从最后一次写入开始倒计时。

2. **续期边界清晰**：
   - ✅ 续期：`populateByWorker`、`populateByCurrentWorker`、`setByWorker`、`setByCurrentWorker`（写操作）
   - ❌ 不续期：`getByWorker`、`getCurrent`（读操作）

3. **周期性性能微降**：每 Worker 每 30 天会出现一次 cache miss（1 次 PG 查询），然后立即回填恢复正常。

4. **这是工程权衡**：接受极低概率的性能退化，换取纯读的高性能。

5. **注释不准确**："Idle TTL" 的描述与实际行为不符，但代码行为是正确的。
