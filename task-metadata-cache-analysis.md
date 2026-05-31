# TaskMetadataCache 深度代码分析

## 1. 设计动机与问题背景

Trigger.dev 的任务触发（trigger）热路径需要解析每个 task 的队列名、TTL、触发源（triggerSource）等元数据，否则就要查询 PostgreSQL 的 `BackgroundWorkerTask` 表（JOIN `TaskQueue`）。在高并发触发场景下，这会成为瓶颈。

`TaskMetadataCache` 就是为解决此问题引入的 Redis 缓存层。但缓存的难点在于：

1. **两个独立的读取场景**：非锁定触发（走当前环境最新版本）和锁定版本触发（走特定 Worker），它们的查询键完全不同。
2. **写入时序的竞态**：部署 Promotion 时会原子替换当前环境的所有元数据，而并发进行的回填写（back-fill）可能用过时数据覆盖新 Promotion。
3. **生命周期差异**：env keyspace 应跟随 Promotion 走（换版本即替换），by-worker keyspace 只在被锁定引用时才有用（滑动过期即可）。

下面逐一拆解。

---

## 2. Redis 数据结构详解

### 2.1 两个 Keyspace

| Keyspace | Redis Key | 数据类型 | 用途 |
|----------|-----------|---------|------|
| env keyspace | `task-meta:env:{envId}` | Hash | 存储某环境下"当前版本"的所有任务元数据 |
| by-worker keyspace | `task-meta:by-worker:{workerId}` | Hash | 存储某 Worker（可能非当前版本）的所有任务元数据 |

> 注：实际 key 带有 `tr:` 前缀（`keyPrefix: "tr:"`），即 `tr:task-meta:env:{envId}` 和 `tr:task-meta:by-worker:{workerId}`。

### 2.2 为什么是 Hash + slug 作为 field

**核心选择**：使用 Redis Hash（而非为每个 slug 单独设 key）。

```
task-meta:env:env_abc123
  ├── "my-task"           → '{"t":"5m","k":"STANDARD","q":"queue_id_1","n":"task/my-task"}'
  ├── "send-email"        → '{"t":null,"k":"SCHEDULED","q":"queue_id_2","n":"emails"}'
  ├── "__owner_worker_id" → 'worker_xyz789'     ← 内部管理字段
  └── ...

task-meta:by-worker:worker_xyz789
  ├── "my-task"           → '{"t":"5m","k":"STANDARD","q":"queue_id_1","n":"task/my-task"}'
  ├── "send-email"        → '{"t":null,"k":"SCHEDULED","q":"queue_id_2","n":"emails"}'
  └── ...
```

**slug 作为 field 的原因**：

1. **O(1) 单任务查询**：`HGET task-meta:env:env_abc123 "my-task"` 直接返回一个 task 的元数据，无需扫描。触发路径只需查一个 task，这是最常见操作。
2. **原子批量替换**：Promotion 时通过 Lua 脚本 `DEL` + `HSET` 整个 Hash，确保 env keyspace 的全部字段要么全部是旧版本、要么全部是新版本，不会出现同一 env 下部分 task 指向旧 Worker、部分指向新 Worker 的不一致状态。
3. **空间效率**：一个 Hash 共享一次 key 开销和一次 TTL 管控，比每个 slug 一个独立 key 更紧凑。
4. **CAS 守卫**：`__owner_worker_id` field 存在同一个 Hash 中，可以在 Lua 脚本中原子读取并与预期值比较，不需要额外 key。

### 2.3 编码格式

每个 field 的 value 是 JSON 编码的 `EncodedEntry`：

```typescript
type EncodedEntry = {
  t: string | null;          // ttl
  k: TaskTriggerSource;      // triggerSource (STANDARD | SCHEDULED | AGENT)
  q: string | null;          // queueId
  n: string;                 // queueName
};
```

解码后还原为 `TaskMetadataEntry`：

```typescript
type TaskMetadataEntry = {
  slug: string;              // task 标识符 (从 field 名还原)
  ttl: string | null;        // 任务 TTL
  triggerSource: TaskTriggerSource;
  queueId: string | null;    // 队列 ID
  queueName: string;         // 队列名称
};
```

### 2.4 TTL 策略

| Keyspace | 默认 TTL | 策略 |
|----------|---------|------|
| `task-meta:env:{envId}` | 24 小时 (`86400s`) | 固定过期，每次 Promotion 重新设置 |
| `task-meta:by-worker:{workerId}` | 30 天 (`2592000s`) | 滑动过期，每次读写刷新 |

**设计意图**：
- env keyspace 由 Promotion 拥有，TTL 是兜底安全网（防止 Promotion 失败后永久驻留旧数据）
- by-worker keyspace 不跟随 Promotion，只在被锁定版本触发时才被访问，因此用滑动过期保持"活跃则常驻"

---

## 3. 四个 Lua 脚本深度解析

### 3.1 REPLACE_HASH_LUA — 单 Hash 原子替换

```
使用场景：populateByWorker()
操作对象：task-meta:by-worker:{workerId}
```

```lua
redis.call("DEL", KEYS[1])                    -- 先删除整个旧 Hash
if #ARGV > 1 then
  local fv = {}
  for i = 2, #ARGV do fv[#fv + 1] = ARGV[i] end
  redis.call("HSET", KEYS[1], unpack(fv))     -- 批量写入新 field-value 对
end
local ttl = tonumber(ARGV[1])
if ttl and ttl > 0 then
  redis.call("EXPIRE", KEYS[1], ttl)          -- 重设 TTL
end
```

**关键特性**：`DEL + HSET` 在同一个 Lua 脚本中原子执行，保证 Hash 要么全是旧数据，要么全是新数据。对于空 entries（Worker 没有任何 task），只执行 `DEL`，清除残留。

### 3.2 REPLACE_TWO_HASHES_LUA — 双 Hash 原子替换

```
使用场景：populateByCurrentWorker()
操作对象：task-meta:env:{envId} + task-meta:by-worker:{workerId}
```

```lua
redis.call("DEL", KEYS[1])                    -- 清空 env Hash
redis.call("DEL", KEYS[2])                    -- 清空 by-worker Hash
if #ARGV > 3 then
  local fv = {}
  for i = 4, #ARGV do fv[#fv + 1] = ARGV[i] end
  redis.call("HSET", KEYS[1], unpack(fv))     -- 写入 env Hash
  redis.call("HSET", KEYS[2], unpack(fv))     -- 写入 by-worker Hash（相同内容）
end
redis.call("HSET", KEYS[1], "__owner_worker_id", ARGV[3])  -- 标记 env Hash 的属主
-- 设置各自的 TTL
```

**关键设计**：
- 同一组 field-value 对同时写入两个 Hash，保证数据一致
- env Hash 额外写入 `__owner_worker_id` 字段，记录当前"拥有"此 env keyspace 的 Worker ID
- 这个 owner 标记是后续 `SET_TWO_FIELDS_LUA` 做 CAS 守卫的基础

### 3.3 SET_FIELD_REFRESH_TTL_LUA — 单字段 upsert + 刷新 TTL

```
使用场景：setByWorker()
操作对象：task-meta:by-worker:{workerId}
```

```lua
redis.call("HSET", KEYS[1], ARGV[2], ARGV[3])  -- upsert 单个 field
local ttl = tonumber(ARGV[1])
if ttl and ttl > 0 then
  redis.call("EXPIRE", KEYS[1], ttl)            -- 刷新整个 Hash 的 TTL
end
```

**用途**：回填路径（cache miss → 查 PG → 写回缓存），只更新 by-worker keyspace。

### 3.4 SET_TWO_FIELDS_LUA — 双 keyspace 单字段 upsert（带 CAS）

```
使用场景：setByCurrentWorker()
操作对象：task-meta:env:{envId} + task-meta:by-worker:{workerId}
```

```lua
-- 1. by-worker Hash 无条件写入
redis.call("HSET", KEYS[2], ARGV[4], ARGV[5])
-- 刷新 by-worker TTL
local workerTtl = tonumber(ARGV[2])
if workerTtl and workerTtl > 0 then
  redis.call("EXPIRE", KEYS[2], workerTtl)
end

-- 2. env Hash 有条件写入 (CAS 守卫)
local owner = redis.call("HGET", KEYS[1], "__owner_worker_id")
if owner == false or owner == ARGV[3] then
  -- 只有当 env Hash 还没有 owner，或者 owner 等于当前 workerId 时才写入
  redis.call("HSET", KEYS[1], ARGV[4], ARGV[5])
  if owner == false then
    redis.call("HSET", KEYS[1], "__owner_worker_id", ARGV[3])
  end
  -- 只在 Hash 没有 TTL 时设置（不覆盖 Promotion 设置的 TTL）
  local envTtl = tonumber(ARGV[1])
  if envTtl and envTtl > 0 and redis.call("TTL", KEYS[1]) == -1 then
    redis.call("EXPIRE", KEYS[1], envTtl)
  end
end
```

**CAS 守卫详解**：

```
时序竞态场景：
  t1: 回填者 A 读取 PG，发现 env_X 的当前 worker 是 W1
  t2: Promotion 发生，W2 替换 W1 成为 current，REPLACE_TWO_HASHES 原子清空并重建 env Hash
  t3: 回填者 A 拿着 W1 的旧数据尝试写回 env Hash

  没有 CAS：A 会用 W1 的旧数据覆盖 W2 的新数据 → 数据错乱
  有 CAS：A 检查 __owner_worker_id != W1 → 跳过 env 侧写入 → 数据安全
```

条件判断逻辑：
- `owner == false`：env Hash 不存在或没有 owner（可能是首次创建），允许写入
- `owner == ARGV[3]`（即当前 workerId）：env Hash 的 owner 仍是自己查询时对应的 Worker，安全写入
- 否则：owner 已被 Promotion 替换为另一个 Worker，**跳过 env 侧写入**

by-worker 侧无条件写入是因为：by-worker key 包含 workerId，写入"错误"的 Worker 的 keyspace 不会被错误读取——只有锁定到该 Worker 版本的请求才会查到它。

---

## 4. 读写命中链路完整追踪

### 4.1 读取链路一：非锁定触发 → `getCurrent`

```
触发入口：triggerTask / batchTrigger (无 lockToVersion)
    │
    ▼
DefaultQueueManager.resolveQueueProperties()
    │  lockedBackgroundWorker === undefined
    ▼
DefaultQueueManager.getTaskQueueInfo()
    │
    ▼
DefaultQueueManager.resolveCurrentTaskMetadata(environment, slug)
    │
    ├── 1. this.taskMetaCache.getCurrent(envId, slug)
    │       → Redis: HGET task-meta:env:{envId} "{slug}"
    │       ├── 命中 → 直接返回 TaskMetadataEntry ✅ (0 PG 查询)
    │       └── 未命中 ↓
    │
    ├── 2. findCurrentWorkerFromEnvironment(environment)
    │       → PG: 查询当前 promote 的 BackgroundWorker
    │
    ├── 3. backgroundWorkerTask.findFirst({ workerId, slug })
    │       → PG: 查询该 Worker 下的任务元数据
    │       ├── 找到 → 构建 TaskMetadataEntry
    │       │   ↓
    │       │   void this.taskMetaCache.setByCurrentWorker(envId, workerId, entry)
    │       │       → Lua: SET_TWO_FIELDS_LUA
    │       │       → 同时回填 env + by-worker 两个 keyspace
    │       │       → env 侧带 CAS 守卫
    │       │   返回 entry ✅ (2 PG 查询)
    │       │
    │       └── 未找到 → 返回 null
```

**源码位置**：`apps/webapp/app/runEngine/concerns/queues.server.ts:304-344`

### 4.2 读取链路二：锁定版本触发 → `getByWorker`

```
触发入口：triggerTask (有 lockToVersion) / triggerAndWait 子任务
    │
    ▼
DefaultQueueManager.resolveQueueProperties(lockedBackgroundWorker)
    │  lockedBackgroundWorker !== undefined
    ▼
DefaultQueueManager.resolveLockedTaskMetadata(workerId, envId, slug)
    │
    ├── 1. this.taskMetaCache.getByWorker(workerId, slug)
    │       → Redis: HGET task-meta:by-worker:{workerId} "{slug}"
    │       ├── 命中 → 直接返回 TaskMetadataEntry ✅ (0 PG 查询)
    │       └── 未命中 ↓
    │
    ├── 2. backgroundWorkerTask.findFirst({ workerId, slug })
    │       → PG: 查询该 Worker 下的任务元数据
    │       ├── 找到 → 构建 TaskMetadataEntry
    │       │   ↓
    │       │   void this.taskMetaCache.setByWorker(workerId, entry)
    │       │       → Lua: SET_FIELD_REFRESH_TTL_LUA
    │       │       → 只回填 by-worker keyspace
    │       │       → 不触碰 env keyspace
    │       │   返回 entry ✅ (1 PG 查询)
    │       │
    │       └── 未找到 → 返回 null
```

**源码位置**：`apps/webapp/app/runEngine/concerns/queues.server.ts:262-294`

### 4.3 写入链路一：Worker 创建（尚未 Promotion）→ `populateByWorker`

```
触发场景：V4 构建路径 — Worker 创建但尚未 promote
    │
    ▼
CreateDeploymentBackgroundWorkerServiceV4.call()
    │
    ├── 创建 BackgroundWorker + BackgroundWorkerTask
    │
    └── this._taskMetaCache.populateByWorker(workerId, workerTaskEntries)
            → Lua: REPLACE_HASH_LUA
            → 只写入 task-meta:by-worker:{workerId}
            → 不触碰 env keyspace（因为还未 promote）
```

**为什么只写 by-worker**：V4 构建路径中，Worker 创建和 Promotion 是解耦的两步：
1. `CreateDeploymentBackgroundWorkerServiceV4` → 创建 Worker，写入 by-worker
2. `FinalizeDeploymentService` → 推送镜像
3. `ChangeCurrentDeploymentService` → promote，此时才写入 env keyspace

这种解耦的好处是：即使 Worker 尚未 promote，锁定到该 Worker 版本的触发请求也能命中缓存。

**源码位置**：`apps/webapp/app/v3/services/createDeploymentBackgroundWorkerV4.server.ts:156-158`

### 4.4 写入链路二：本地开发 Worker 创建 → `populateByCurrentWorker`

```
触发场景：本地开发 (npx trigger.dev dev) — Worker 立即成为当前版本
    │
    ▼
CreateBackgroundWorkerService.call()
    │
    ├── 创建 BackgroundWorker + BackgroundWorkerTask
    │
    └── if (environment.type === "DEVELOPMENT")
            this._taskMetaCache.populateByCurrentWorker(envId, workerId, entries)
            → Lua: REPLACE_TWO_HASHES_LUA
            → 同时写入 env + by-worker 两个 keyspace
            → 在 env Hash 中设置 __owner_worker_id
        else
            this._taskMetaCache.populateByWorker(workerId, entries)
            → 仅写入 by-worker keyspace
```

**DEV 环境特殊处理**：本地开发没有 Promotion 流程，每次代码变更就直接创建新 Worker 并成为当前版本（`findCurrentWorkerFromEnvironment` 对 DEV 环境取最新 Worker），所以可以立即写入 env keyspace。

**源码位置**：`apps/webapp/app/v3/services/createBackgroundWorker.server.ts:239-249`

### 4.5 写入链路三：V3 部署路径 → `populateByCurrentWorker`

```
触发场景：V3 构建路径（已废弃但仍在代码中）— Worker 创建后立即 promote
    │
    ▼
CreateDeploymentBackgroundWorkerServiceV3.call()
    │
    ├── 创建 BackgroundWorker
    ├── 直接 upsert WorkerDeploymentPromotion (立即 promote)
    │
    └── this._taskMetaCache.populateByCurrentWorker(envId, workerId, entries)
            → Lua: REPLACE_TWO_HASHES_LUA
```

**源码位置**：`apps/webapp/app/v3/services/createDeploymentBackgroundWorkerV3.server.ts:173-177`

### 4.6 写入链路四：部署 Promotion → `populateByCurrentWorker`

```
触发场景：V4 构建路径的最终步骤 — 部署被 promote 为当前版本
    │
    ▼
ChangeCurrentDeploymentService.call(deployment, "promote"|"rollback")
    │
    ├── 验证方向性（promote 向前 / rollback 向后）
    ├── upsert WorkerDeploymentPromotion
    ├── 查询该 Worker 的所有 BackgroundWorkerTask
    │
    └── this._taskMetaCache.populateByCurrentWorker(envId, workerId, metadataEntries)
            → Lua: REPLACE_TWO_HASHES_LUA
            → 原子替换 env + by-worker 两个 keyspace
            → 新的 __owner_worker_id 标记
```

**源码位置**：`apps/webapp/app/v3/services/changeCurrentDeployment.server.ts:163-167`

---

## 5. 全链路总结图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          写入侧 (Write Path)                            │
│                                                                          │
│  ┌───────────────────────┐                                               │
│  │ Worker 创建 (V4 构建)  │                                              │
│  │ 尚未 promote           │──populateByWorker()──► by-worker Hash only   │
│  └───────────────────────┘                                               │
│                                                                          │
│  ┌───────────────────────┐                                               │
│  │ Worker 创建 (DEV 模式) │                                              │
│  │ 立即成为当前版本        │──populateByCurrentWorker()──► env + by-worker│
│  └───────────────────────┘                                               │
│                                                                          │
│  ┌───────────────────────┐                                               │
│  │ Worker 创建 (V3 路径)  │                                              │
│  │ 立即 promote           │──populateByCurrentWorker()──► env + by-worker│
│  └───────────────────────┘                                               │
│                                                                          │
│  ┌───────────────────────┐                                               │
│  │ Promotion / Rollback  │                                              │
│  │ 切换当前版本            │──populateByCurrentWorker()──► env + by-worker│
│  └───────────────────────┘                                               │
│                                                                          │
│  ┌───────────────────────┐                                               │
│  │ Cache Miss 回填        │                                              │
│  │ (非锁定路径)           │──setByCurrentWorker()──► env(CAS) + by-worker│
│  │ (锁定路径)             │──setByWorker()────────► by-worker only       │
│  └───────────────────────┘                                               │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                          读取侧 (Read Path)                              │
│                                                                          │
│  非锁定触发：trigger() / batchTrigger()                                   │
│    getCurrent(envId, slug)                                                │
│    → HGET task-meta:env:{envId} "{slug}"                                 │
│    → 命中 → 返回 (0 PG)                                                   │
│    → 未命中 → PG 回填 → setByCurrentWorker()                             │
│                                                                          │
│  锁定版本触发：trigger({ lockToVersion }) / triggerAndWait()              │
│    getByWorker(workerId, slug)                                            │
│    → HGET task-meta:by-worker:{workerId} "{slug}"                        │
│    → 命中 → 返回 (0 PG)                                                   │
│    → 未命中 → PG 回填 → setByWorker()                                    │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 6. `__owner_worker_id` 的 CAS 机制深入

### 6.1 为什么需要 CAS

回填路径的时序问题：

```
Thread A (回填者)                   Thread B (Promotion)
─────────────────                   ────────────────────
1. HGET env → miss
2. 查 PG: 当前 worker = W1
                                   3. promote W2
                                   4. REPLACE_TWO_HASHES:
                                      - DEL env Hash
                                      - 写入 W2 的所有 task
                                      - __owner_worker_id = W2
5. 尝试回填 W1 的数据到 env
```

如果步骤 5 无条件写入，W1 的旧数据会覆盖 W2 的新数据。

### 6.2 CAS 如何工作

```lua
-- SET_TWO_FIELDS_LUA 中的关键代码
local owner = redis.call("HGET", KEYS[1], "__owner_worker_id")

if owner == false or owner == ARGV[3] then
  -- owner 不存在 或 owner 等于当前回填者的 workerId → 安全写入
  redis.call("HSET", KEYS[1], ARGV[4], ARGV[5])
  ...
end
-- owner 存在且不等于当前 workerId → 说明已被 Promotion 替换 → 跳过写入
```

回到上面的时序：

```
Thread A                           Thread B
─────────                          ─────────
1. HGET env → miss
2. 查 PG: 当前 worker = W1
                                   3. promote W2 → __owner_worker_id = W2
5. CAS 检查: owner = "W2" ≠ "W1"
   → 跳过 env 侧写入 ✅
   → by-worker 侧仍写入 W1 的 keyspace（不影响，只有锁定到 W1 才会读）
```

### 6.3 `owner == false` 分支

当 env Hash 完全不存在（可能是首次部署或 Hash 已过期被 Redis 淘汰），`HGET` 返回 `false`，此时允许写入并设置 owner。这处理了以下边界场景：

- 部署后的首次触发，缓存尚未被 populate
- Redis 重启后缓存丢失
- env Hash 的 24 小时 TTL 到期后

---

## 7. 方法到 Lua 脚本的完整映射

| 方法 | Lua 脚本 | 操作的 Keyspace | 写入模式 | CAS 守卫 |
|------|---------|----------------|---------|---------|
| `populateByCurrentWorker()` | `REPLACE_TWO_HASHES_LUA` | env + by-worker | 全量替换 | N/A（替换语义，不需要 CAS） |
| `populateByWorker()` | `REPLACE_HASH_LUA` | by-worker only | 全量替换 | N/A |
| `setByCurrentWorker()` | `SET_TWO_FIELDS_LUA` | env + by-worker | 单字段 upsert | ✅ env 侧有 CAS |
| `setByWorker()` | `SET_FIELD_REFRESH_TTL_LUA` | by-worker only | 单字段 upsert | N/A |
| `getCurrent()` | 直接 `HGET` | env | 只读 | N/A |
| `getByWorker()` | 直接 `HGET` | by-worker | 只读 | N/A |

---

## 8. 容错与降级设计

### 8.1 写入容错

所有缓存写入方法都包裹在 `try/catch` 中，错误只记日志不抛出：

```typescript
async populateByCurrentWorker(envId, workerId, entries) {
  try {
    await this.redis.taskMetaReplaceTwoHashes(/* ... */);
  } catch (error) {
    logger.error("Failed to populate task metadata cache (current worker)", { envId, workerId, error });
  }
}
```

这意味着：
- Redis 不可用时，部署流程不会被阻塞
- 回填失败只是性能退化（下次触发走 PG 查询）
- Promotion 仍然会在 PG 层面成功（`WorkerDeploymentPromotion` 表已更新）

### 8.2 读取容错

```typescript
async #get(key: string, slug: string): Promise<TaskMetadataEntry | null> {
  try {
    const raw = await this.redis.hget(key, slug);
    if (!raw) return null;
    return decode(slug, raw);
  } catch (error) {
    logger.error("Failed to read task metadata from cache", { key, slug, error });
    return null;  // 降级到 PG 查询
  }
}
```

Redis 读取失败时返回 `null`，调用方会走 PG 回填路径。

### 8.3 NoopTaskMetadataCache

当 `TASK_META_CACHE_REDIS_HOST` 未配置时，使用 `NoopTaskMetadataCache`，所有读返回 `null`，所有写静默忽略。系统完全回退到 PG 查询。

---

## 9. 缓存一致性保证总结

| 场景 | 保证机制 |
|------|---------|
| Promotion 原子性 | `REPLACE_TWO_HASHES_LUA` 在一个 Lua 脚本中 DEL + HSET 两个 Hash |
| Promotion 与回填并发 | `__owner_worker_id` CAS 守卫阻止旧数据覆盖新数据 |
| by-worker 侧无竞态 | key 包含 workerId，不同版本写入不同 key，互不干扰 |
| 空 Worker 部署 | `populateByWorker([])` / `populateByCurrentWorker([])` 执行 DEL，清除旧残留 |
| env Hash 过期 | 24h TTL 兜底，回填路径会重建 |
| by-worker Hash 过期 | 30d 滑动 TTL，活跃 Worker 常驻 |
| env Hash TTL 不被回填覆盖 | `SET_TWO_FIELDS_LUA` 中只在 `TTL == -1`（无过期）时才设置 TTL |
