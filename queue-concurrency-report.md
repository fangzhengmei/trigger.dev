# 队列层并发与优先级排队机制分析报告

## 1. 优先级时间偏移的唯一计算公式

### 1.1 公式定义

基于 `internal-packages/run-engine/src/engine/systems/enqueueSystem.ts:89` 的代码实现：

```typescript
const timestamp = (run.queueTimestamp ?? run.createdAt).getTime() - run.priorityMs;
```

**唯一计算公式**：
```
effectiveScore = baseTimestamp - priorityMs
```

其中：
- `baseTimestamp` = `run.queueTimestamp`（如果显式设置）否则为 `run.createdAt`
- `priorityMs` = 用户传入的优先级值（秒）× 1000（转换为毫秒）
- ZSET 按 score 升序排列，score 越小的消息越早出队

### 1.2 时序示例验证

**示例 1：优先级插队场景**

场景描述：
- T0（1000ms）：任务 A 入队，priority = 0
- T0 + 8000ms（9000ms）：任务 B 入队，priority = 10（即 priorityMs = 10000）

计算过程：
```
任务 A: effectiveScore = 1000 - 0 = 1000
任务 B: effectiveScore = 9000 - 10000 = -1000
```

队列排序结果（按 score 升序）：
| 任务 | effectiveScore | 出队顺序 |
|------|---------------|---------|
| B    | -1000         | 1st     |
| A    | 1000          | 2nd     |

**结论**：任务 B 虽然晚 8 秒入队，但由于优先级为 10 秒，最终排在任务 A 前面出队。

---

**示例 2：多优先级混合排序**

场景描述（来自 `priority.test.ts` 的测试用例）：
- 同一时刻触发 5 个任务，优先级分别为：
  - 任务 0：priority = undefined（即 0）
  - 任务 1：priority = 500 秒
  - 任务 2：priority = -1200 秒（延迟执行）
  - 任务 3：priority = 1000 秒
  - 任务 4：priority = 4000 秒

计算过程（假设 baseTimestamp = 100000ms）：
```
任务 0: 100000 - 0      = 100000
任务 1: 100000 - 500000 = -400000
任务 2: 100000 - (-1200000) = 1300000
任务 3: 100000 - 1000000 = -900000
任务 4: 100000 - 4000000 = -3900000
```

队列排序结果（按 score 升序）：
| 任务 | effectiveScore | 出队顺序 |
|------|---------------|---------|
| 4    | -3900000      | 1st     |
| 3    | -900000       | 2nd     |
| 1    | -400000       | 3rd     |
| 0    | 100000        | 4th     |
| 2    | 1300000       | 5th     |

**结论**：与测试用例预期的出队顺序 `[4, 3, 1, 0, 2]` 完全一致，验证了公式的正确性。

### 1.3 优先级特性边界

- **作用范围**：仅在同一租户（环境）内生效，不跨租户抢占资源
- **负数含义**：`priority < 0` 表示延迟执行，effectiveScore 大于当前时间的消息不会被调度
- **相对值**：优先级的具体数值不重要，重要的是相对大小关系
- **不绕过并发**：高优先级任务在并发满时仍需等待，仅在排序时优先

---

## 2. 租户额度来源与生效顺序

### 2.1 四层关系的最终结论（无歧义版）

基于完整代码证据分析，并发额度的四层关系、触发条件、生效顺序严格如下：

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                      并发额度四层关系（最终版）                                │
├─────────────┬─────────────────────────────────┬───────────────────────────────┤
│ 优先级（高→低）│ 配置层级                       │ 触发条件与边界                 │
├─────────────┼─────────────────────────────────┼───────────────────────────────┤
│ 1           │ 环境级显式覆盖                   │ 始终生效（数据库有值即生效）    │
│             │ RuntimeEnvironment              │ 存储：PostgreSQL + Redis 缓存 │
│             │ .maximumConcurrencyLimit        │ 同步：updateEnvConcurrencyLimits() │
│             │ 数据库 DEFAULT：5               │ 边界：单环境粒度               │
├─────────────┼─────────────────────────────────┼───────────────────────────────┤
│ 2           │ 订阅计划级默认值（仅云端）       │ billing client 已配置           │
│             │ getDefaultEnvironmentLimit-     │ 且 currentPlan() 调用成功       │
│             │ FromPlan()                      │ 按环境类型区分配额             │
├─────────────┼─────────────────────────────────┼───────────────────────────────┤
│ 3           │ 组织级兜底值（仅无计费）         │ billing client 未配置时         │
│             │ Organization                    │ 主动查询组织表作为默认值       │
│             │ .maximumConcurrencyLimit        │ 边界：自托管场景默认配额       │
├─────────────┼─────────────────────────────────┼───────────────────────────────┤
│ 4           │ 系统默认值（最后兜底）           │ Redis 中无环境限制记录时        │
│             │ DEFAULT_ENV_EXECUTION_          │ 环境变量默认值：100            │
│             │ CONCURRENCY_LIMIT              │ 边界：运行时最后防线           │
└─────────────┴─────────────────────────────────┴───────────────────────────────┘
```

### 2.2 云端模式的失败语义边界

云端模式下，`getDefaultEnvironmentConcurrencyLimit()` 函数（`platform.v3.server.ts:286-310`）存在两种截然不同的失败场景，其行为边界严格区分如下：

---

#### 场景 A：billing client 未配置

**代码证据**：`platform.v3.server.ts:290-300`
```typescript
if (!client) {
  const org = await $replica.organization.findFirst({
    where: { id: organizationId },
    select: { maximumConcurrencyLimit: true },
  });
  if (!org) throw new Error("Organization not found");
  return org.maximumConcurrencyLimit;
}
```

| 项目 | 行为 |
|------|------|
| **触发条件** | `!client` 为 true（billing 服务未初始化，如自托管、本地开发） |
| **返回值** | `Organization.maximumConcurrencyLimit`（组织表中的配置值） |
| **报错行为** | 仅当组织不存在时抛出 `Error("Organization not found")` |
| **是否降级到组织级** | ✅ 是，主动降级，这是设计行为 |
| **是否影响环境创建** | ❌ 不影响，可正常返回额度值 |

---

#### 场景 B：billing client 已配置但 `currentPlan()` 调用失败

**代码证据**：`platform.v3.server.ts:303-307`
```typescript
const result = await client.currentPlan(organizationId);
if (!result.success) throw new Error("Error getting current plan");

const limit = getDefaultEnvironmentLimitFromPlan(environmentType, result);
if (!limit) throw new Error("No plan found");
```

| 失败类型 | 触发条件 | 报错行为 | 是否降级到组织级 | 对环境创建的影响 |
|---------|---------|---------|----------------|----------------|
| **B1：API 调用失败** | `result.success === false`（网络错误、billing 服务宕机等） | 抛出 `Error("Error getting current plan")` | ❌ 不降级，直接报错 | ⚠️ 环境创建流程会失败（`organization.server.ts:108` 无 try-catch） |
| **B2：计划数据缺失** | `getDefaultEnvironmentLimitFromPlan()` 返回 `undefined`（计划中无对应环境类型的配额） | 抛出 `Error("No plan found")` | ❌ 不降级，直接报错 | ⚠️ 环境创建流程会失败 |

**关键结论**：云端模式下，billing 服务故障是**致命错误**，不会自动降级到组织级兜底。这是设计使然——云端模式依赖计费系统确定配额，降级可能导致配额计算错误。

---

### 2.3 两层默认值的严格区分

| 概念 | 位置 | 默认值 | 触发时机 | 代码证据 | 作用域 |
|------|------|-------|---------|---------|-------|
| 数据库层默认值 | `RuntimeEnvironment.maximumConcurrencyLimit` 字段 `DEFAULT` 约束 | **5** | 新环境 `INSERT` 数据库时 | 迁移脚本 `20240402105424` | 数据持久化层 |
| 系统运行时兜底值 | `DEFAULT_ENV_EXECUTION_CONCURRENCY_LIMIT` 环境变量 | **100** | Redis 中无环境限制记录时（极罕见） | `env.server.ts:334`、`docs/self-hosting/env/webapp.mdx:60` | 运行时调度层 |

**常见误解澄清**：
- ❌ 错误："defaultEnvConcurrency 默认为 5"
- ✅ 正确：5 是数据库字段的 DEFAULT 约束，100 是运行时 Redis 兜底的环境变量默认值
- ✅ 正确：两者处于完全不同的系统层级，触发时机无重叠

### 2.4 云端 vs 无计费部署生效路径对照表（精简版）

| 维度 | 云端模式（有 billing） | 无计费模式（自托管） |
|------|----------------------|---------------------|
| **第 1 层 环境级覆盖** | ✅ 始终生效，数据库值同步到 Redis | ✅ 始终生效，数据库值同步到 Redis |
| **第 2 层 订阅计划级** | ✅ billing 正常时生效 | ❌ 无 billing 不触发 |
| **第 3 层 组织级兜底** | ❌ 仅 billing 未配置时降级（非失败场景） | ✅ 主动使用，无 billing 时直接查组织表 |
| **第 4 层 系统默认值** | ✅ Redis 无记录时兜底（100） | ✅ Redis 无记录时兜底（100） |
| **billing 未配置** | 降级到组织级（场景 A） | 正常路径，直接用组织级 |
| **billing 调用失败** | ❌ 抛出异常，不降级 | N/A（无 billing 服务） |
| **典型生效路径** | 环境级 → 订阅计划级 → 系统默认值 | 环境级 → 组织级兜底 → 系统默认值 |

### 2.5 突发因子（Burst Factor）

**位置**：`RuntimeEnvironment.concurrencyLimitBurstFactor`

- **作用**：允许环境短时超出基础并发限制
- **计算公式**：`实际并发上限 = 基础并发上限 × burstFactor`
- **默认值**：`DEFAULT_ENV_EXECUTION_CONCURRENCY_BURST_FACTOR` 默认 1.0（不允许突发）
- **边界**：独立于并发限制层级，在基础并发上限确定后应用
- **触发时机**：调度前计算可用容量时生效

### 2.6 队列大小限制

基于 `apps/webapp/app/v3/utils/queueLimits.server.ts:24-33`：

```typescript
if (environmentType === "DEVELOPMENT") {
  return organization.maximumDevQueueSize ?? env.MAXIMUM_DEV_QUEUE_SIZE ?? null;
}
return organization.maximumDeployedQueueSize ?? env.MAXIMUM_DEPLOYED_QUEUE_SIZE ?? null;
```

**生效顺序（高→低）**：
1. 组织级 `maximumDevQueueSize` / `maximumDeployedQueueSize`（数据库字段）
2. 环境变量 `MAXIMUM_DEV_QUEUE_SIZE` / `MAXIMUM_DEPLOYED_QUEUE_SIZE`
3. `null`（无限制）

---

## 3. 令牌桶、并发额度、Worker 拉取三段协同链路

### 3.1 系统架构总览

```
                    ┌─────────────────┐
                    │   API 请求入口   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  令牌桶限流检查  │  ◄── 按组织配置差异化限流
                    └────────┬────────┘
                             │
                      ┌──────▼──────┐
                      │  RunQueue   │
                      │  消息入队    │  ◄── 计算 effectiveScore
                      └──────┬──────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
    ┌─────▼─────┐      ┌─────▼─────┐      ┌─────▼─────┐
    │  快路径   │      │  慢路径   │      │  延迟任务  │
    │(并发有剩余)│      │(等待调度)  │      │(未来可见)  │
    └─────┬─────┘      └─────┬─────┘      └───────────┘
          │                  │
          └──────────────────┘
                             │
                    ┌────────▼────────┐
                    │ Master 消费者   │
                    │ DRR 公平调度    │  ◄── 两级分发索引
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ 并发额度预留检查 │  ◄── ConcurrencyManager
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ 推入 Worker 队列 │  ◄── 背压机制限制深度
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Worker BLPOP   │  ◄── 低延迟拉取
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   任务执行完成   │
                    │ 释放并发额度     │  ◄── 触发下一轮调度
                    └─────────────────┘
```

### 3.2 第一段：令牌桶（API 入口限流）

**位置**：`apps/webapp/app/services/apiRateLimit.server.ts`

**核心机制**：
- **算法**：令牌桶（Token Bucket）
- **按租户差异化**：每个组织可通过 `apiRateLimiterConfig` 自定义 `refillRate`（补充速率）、`interval`（间隔）、`maxTokens`（最大容量）
- **白名单**：内部回调、Webhook、Packet 上传下载等路径不受限流
- **降级策略**：PUBLIC_JWT 类型使用固定窗口（fixedWindow）而非令牌桶

**与后续链路的关系**：
- 令牌桶限制的是**入队请求频率**，而非任务执行并发
- 通过令牌桶的请求才能进入 RunQueue 入队流程
- 防止单个租户通过高频 API 调用压垮系统

### 3.3 第二段：并发额度（调度前检查）

**位置**：`packages/redis-worker/src/fair-queue/concurrency.ts`、`apps/webapp/app/v3/marqs/fairDequeuingStrategy.server.ts`

**核心机制**：

**ConcurrencyManager 多级并发组**：
```typescript
concurrencyGroups: [
  {
    name: "environment",
    extractGroupId: (queue) => queue.environmentId,
    defaultLimit: 10,
    getLimit: async (envId) => getEnvConcurrency(envId),
  },
  // 可扩展：org、project、queue 等维度
]
```

**原子预留流程**（Lua 脚本保证）：
```lua
-- reserveConcurrency 脚本逻辑
1. 遍历所有并发组，查询当前使用量
2. 如任一并发组已达到上限，返回 0（失败）
3. 所有组有剩余容量时，原子性将 messageId 添加到所有组的 SET 中
4. 返回 1（成功）
```

**容量计算**：
- `getAvailableCapacity()`：跨所有并发组取最小可用容量
- 确保不会因某一组耗尽而超发
- 批量释放：支持批量释放多个消息的并发槽位

**两级分发索引加速调度**：
- **Level 1 - 分发索引**：ZSET `{prefix}:dispatch:{shardId}`，存储 tenantId（即 envId），score 为该租户最旧消息时间戳
- **Level 2 - 租户队列索引**：ZSET `{prefix}:tenantq:{tenantId}`，存储 queueId，score 为该队列最旧消息时间戳
- 使用 Jump Consistent Hash 分片，相同租户始终映射到同一分片

**DRR 调度器**：
- 量子（Quantum）：每轮分配给每个租户的消息处理配额
- 赤字（Deficit）：未使用的量子累积到下一轮
- 最大赤字（Max Deficit）：防止单个租户长期不活跃后累积过多赤字
- 每轮按赤字从高到低排序租户，选择有可用并发容量的租户调度

### 3.4 第三段：Worker 拉取（执行层）

**位置**：`packages/redis-worker/src/fair-queue/workerQueue.ts`

**三层队列结构**：
```
1. 消息队列（Message Queue）
   ZSET 结构
   Key: {prefix}:queue:{orgId}:{envId}:{queueName}
   Score: effectiveScore = baseTimestamp - priorityMs
   存储所有待处理消息

2. 主队列（Master Queue / 分发索引）
   ZSET 结构
   Key: {prefix}:dispatch:{shardId}
   存储有待处理消息的租户 ID，按最旧消息时间排序

3. Worker 队列（Worker Queue）
   LIST 结构
   Key: {prefix}:workerq:{workerQueueName}
   已认领但未被 Worker 拉取的消息
   使用 BLPOP 实现低延迟拉取
```

**消息认领与拉取流程**：

Master 消费者（每分片一个）：
```
1. 从 Level 1 分发索引获取所有活跃租户
2. DRR 调度器为每个租户增加 quantum 到 deficit
3. 按 deficit 排序，选择有可用并发容量的租户
4. 从 Level 2 索引获取该租户的队列（数量取 deficit 向上取整）
5. 批量认领消息（claimBatch）：
   - 设置 Visibility Timeout
   - 原子预留并发槽位（ConcurrencyManager.reserve）
   - 推入 Worker 队列
6. 租户 deficit -= 实际处理的消息数
```

Worker 进程：
```
1. BLPOP 阻塞等待 Worker 队列
2. 收到 messageKey → 从 in-flight 存储获取完整消息数据
3. 执行任务
4. 完成后调用 completeMessage()：
   - 释放并发槽位
   - 从 in-flight 存储移除
   - 如队列已空，从分发索引移除租户
5. 失败调用 failMessage()：
   - 如重试次数 < maxAttempts：计算退避延迟，重新入队（score = now + delay）
   - 否则：移入死信队列（DLQ）
```

**背压机制**：
- Worker 队列深度限制（workerQueueMaxDepth）
- 防止已认领的消息无限堆积
- 保护 Visibility Timeout（堆积过深可能导致消息在 Worker 拉取前就超时）
- 达到上限时，Master 消费者暂停认领

### 3.5 三段协同的关键交互点

**场景：高并发下的优先级任务**

假设环境 A 并发限制 = 10，当前已用 = 10，此时到达一个 priority = 3600 秒的任务：

1. **令牌桶阶段**：API 请求通过，进入 RunQueue
2. **入队阶段**：`effectiveScore = now - 3600000`，由于并发已满，走慢路径进入消息队列
3. **调度阶段**：该任务的 score 很小（很"旧"），在环境 A 的队列中排最前
4. **额度预留**：当有并发槽位释放时，该任务被优先调度，预留并发额度
5. **Worker 拉取**：推入 Worker 队列，被 Worker 拉取执行

**结论**：优先级只影响排序，不绕过并发限制。并发限制是硬上限，优先级是软排序。

---

**场景：多租户公平性**

租户 1 有 1000 个任务排队，并发限制 = 10；租户 2 有 1 个任务排队，并发限制 = 10；DRR quantum = 5：

1. 每轮调度：两个租户各获得 5 个消息的配额
2. 实际处理：租户 1 处理 5 个，租户 2 处理 1 个
3. 赤字累积：租户 1 的 deficit = 0（用完配额），租户 2 的 deficit = 5 - 1 = 4
4. 下一轮：租户 1 deficit += 5 = 5，租户 2 deficit += 5 = 9
5. 排序结果：租户 2（deficit 9）优先于租户 1（deficit 5）

**结论**：DRR 通过赤字累积保证公平性，小租户不会被大租户饿死。

---

## 4. 总结

### 4.1 核心机制对照表

| 机制 | 作用层面 | 核心目标 | 实现方式 | 代码位置 |
|------|---------|---------|---------|---------|
| 令牌桶 | API 入口 | 限制入队频率 | Token Bucket 算法，按租户配置 | `apiRateLimit.server.ts` |
| 优先级 | 队列内排序 | 租户内任务差异化 | `effectiveScore = baseTimestamp - priorityMs` | `enqueueSystem.ts:89` |
| 并发管理器 | 调度前 | 执行并发硬上限 | 多组并发计数器，Lua 原子预留 | `concurrency.ts` |
| 两级分发索引 | 调度层 | 大规模租户调度效率 | Level 1 租户索引 + Level 2 队列索引 | `tenantDispatch.ts` |
| DRR 调度器 | 调度层 | 租户间公平性 | 量子+赤字的轮询算法 | `schedulers/drr.ts` |
| Worker 队列 | 执行层 | 低延迟拉取 | BLPOP 阻塞列表，背压机制 | `workerQueue.ts` |
| 环境隔离 | 全链路 | 环境间资源隔离 | 环境 ID 作为调度租户单位 | 多处 |

### 4.2 设计权衡

**优点**：
1. **分层控制**：每一层解决特定问题，职责清晰
2. **可观测性**：每层都有独立的指标和日志
3. **可扩展性**：分片架构支持水平扩展
4. **灵活性**：多种调度策略可配置，支持差异化服务
5. **健壮性**：Visibility Timeout + 死信队列保证消息不丢失

**权衡点**：
1. **最终一致性**：由于分片和缓存，各层状态可能短暂不一致
2. **延迟开销**：多层检查增加了单任务的端到端延迟（通常毫秒级）
3. **配置复杂度**：大量可调参数需要合理配置才能达到最佳性能
4. **优先级限制**：优先级只在租户内生效，无法跨租户抢占资源（设计使然，防止滥用）
