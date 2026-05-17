# 队列层在多租户与多环境前提下的并发上限与优先级排队协同机制

## 1. 架构总览

Trigger.dev 的队列系统采用分层设计，从 API 入口到任务执行，经过多层控制机制协同工作，确保在多租户、多环境场景下实现公平调度、资源隔离和优先级保障。

```
API 入口 (令牌桶限流)
    ↓
运行队列 (RunQueue)
    ↓
公平队列 (FairQueue)
    ├─ 两级租户分发索引 (TenantDispatch)
    ├─ 并发管理器 (ConcurrencyManager)
    ├─ 调度器 (DRR/Weighted/RoundRobin)
    ├─ 可见性管理器 (VisibilityManager)
    └─ Worker 队列管理器 (WorkerQueueManager)
    ↓
Worker 拉取与执行
```

## 2. 核心组件详解

### 2.1 API 层：令牌桶（Token Bucket）限流

**位置**：`apps/webapp/app/services/apiRateLimit.server.ts`

#### 工作原理
令牌桶是 API 入口的第一道防线，用于控制租户的 API 请求频率：

- **核心参数**：
  - `refillRate`：令牌补充速率（每秒/每分补充多少令牌）
  - `interval`：补充间隔
  - `maxTokens`：桶的最大容量（突发流量上限）

- **按租户差异化配置**：
  - 默认配置适用于所有租户
  - 每个组织可通过 `apiRateLimiterConfig` 字段自定义限流策略
  - PUBLIC_JWT 类型使用固定窗口（fixedWindow）策略而非令牌桶

- **白名单机制**：
  - 内部回调、Webhook、Packet 上传下载等路径不受限流
  - 确保关键基础设施通信不受影响

#### 与队列层的关系
令牌桶是**前置保护机制**，防止单个租户通过高频 API 调用压垮系统。它限制的是"入队请求频率"，而非"任务执行并发"。

### 2.2 租户额度：并发控制（ConcurrencyManager）

**位置**：`packages/redis-worker/src/fair-queue/concurrency.ts`

#### 核心能力
ConcurrencyManager 实现了**多级并发组**机制，支持同时对多个维度进行并发限制：

```typescript
// 并发组配置示例
concurrencyGroups: [
  {
    name: "tenant",
    extractGroupId: (queue) => queue.tenantId,
    defaultLimit: 10,
    getLimit: async (envId) => getEnvConcurrency(envId),
  },
  // 可扩展：org、project、queue 等其他维度
]
```

#### 关键机制

1. **原子性操作**（Lua 脚本保证）：
   ```lua
   -- reserveConcurrency 脚本逻辑
   1. 检查所有并发组的当前使用量
   2. 如任一组达到上限，返回 0（失败）
   3. 所有组有剩余容量时，原子性添加 messageId 到所有组
   4. 返回 1（成功）
   ```

2. **容量查询**：
   - `getAvailableCapacity()`：跨所有组取最小可用容量
   - 确保不会因某一组耗尽而超发

3. **批量释放**：
   - 支持批量释放多个消息的并发槽位
   - 提高完成大量任务时的效率

#### 租户额度来源
额度配置来自多个层级，优先级从高到低：
1. **环境级覆盖**：`runtimeEnvironment.maximumConcurrencyLimit`
2. **突发因子**：`concurrencyLimitBurstFactor`（允许短时超出基础限制的倍数）
3. **组织级配置**：`organization.maximumConcurrencyLimit`
4. **计划级限制**：订阅计划中的并发配额
5. **系统默认值**：如未配置则使用默认值（通常为 10）

### 2.3 两级租户分发索引（TenantDispatch）

**位置**：`packages/redis-worker/src/fair-queue/tenantDispatch.ts`

#### 设计目标
解决大规模租户下的调度效率问题，避免在单一主队列中遍历所有队列。

#### 两级索引结构

**Level 1 - 分发索引（Dispatch Index）**：
- Key: `{prefix}:dispatch:{shardId}`
- ZSET 结构，存储 tenantId，score 为该租户所有队列中最旧消息的时间戳
- 仅包含当前有待处理消息的租户

**Level 2 - 租户队列索引（Per-Tenant Queue Index）**：
- Key: `{prefix}:tenantq:{tenantId}`
- ZSET 结构，存储 queueId，score 为该队列中最旧消息的时间戳

#### 分片策略
使用 **Jump Consistent Hash** 算法将租户映射到分片：
- 相同租户始终映射到同一分片
- 分片数可配置，支持水平扩展
- 避免热点问题

### 2.4 调度器（Schedulers）

系统提供三种调度策略，默认为 DRR（赤字轮询）。

#### 2.4.1 DRR 调度器（Deficit Round Robin）

**位置**：`packages/redis-worker/src/fair-queue/schedulers/drr.ts`

DRR 是核心的公平调度算法，确保租户间的资源公平分配：

**核心概念**：
- **量子（Quantum）**：每轮分配给每个租户的消息处理配额
- **赤字（Deficit）**：未使用的量子累积到下一轮
- **最大赤字（Max Deficit）**：防止单个租户长期不活跃后累积过多赤字

**调度流程**：
```
每轮调度:
  1. 从 Level 1 分发索引获取所有活跃租户
  2. 为每个租户的赤字 += 量子（原子 Lua 操作）
  3. 按赤字从高到低排序租户
  4. 选择第一个有可用并发容量的租户
  5. 从该租户的 Level 2 索引获取队列（数量取 deficit 向上取整）
  6. 处理完成后，租户赤字 -= 实际处理的消息数
```

**优点**：
- 公平性：每个租户按比例获得处理机会
- 效率：O(1) 选择下一个租户（跳过无容量的）
- 防饥饿：通过赤字累积保证即使小租户也能获得服务

#### 2.4.2 加权调度器（Weighted Scheduler）

**位置**：`packages/redis-worker/src/fair-queue/schedulers/weighted.ts`

适用于需要差异化服务质量的场景，支持三种权重偏置：

1. **并发限制偏置**（concurrencyLimitBias）：
   - 并发限制越高的租户获得更多权重
   - 适用于付费等级高的客户应获得更多资源

2. **可用容量偏置**（availableCapacityBias）：
   - 当前可用容量越多的租户获得更多权重
   - 防止某租户占满并发后仍被优先调度

3. **队列年龄随机化**（queueAgeRandomization）：
   - 队列越旧，权重越高（基础 FIFO）
   - 添加随机化防止特定模式下的饥饿

#### 2.4.3 轮询调度器（Round Robin）

简单的轮询策略，适用于租户间无差异化需求的场景。

### 2.5 优先级排队机制

**位置**：`internal-packages/run-engine/src/run-queue/index.ts`、`docs/runs/priority.mdx`

#### 优先级模型
优先级以**时间偏移量（秒）**表示，而非传统的整数优先级：

```typescript
// 示例：优先级为 10 秒
await myTask.trigger({ foo: "bar" }, { priority: 10 });
```

#### 工作原理
- 优先级值会加到消息的入队时间戳上
- 排序时：`effectiveScore = enqueueTimestamp + priorityMs`
- 高优先级的消息在排序时表现为"更旧"，因此更早出队

**示例**：
```
T0: Run A 入队，priority=0  → score = T0
T0+8s: Run B 入队，priority=10 → score = T0+8+10 = T0+18

排序结果: Run A (T0) → Run B (T0+18)
但实际 B 会排在 A 前面，因为 B 的优先级让它"看起来"是 T0+18 入队的
不，等一下：实际是 score 越小越先出队，所以应该是 priority 为负值？

实际上是: effectiveQueueTime = timestamp - priorityMs
所以 priority 10 意味着：effectiveQueueTime = (T0+8s) - 10s = T0-2s
这样 Run B 的有效入队时间是 T0-2s，比 Run A 的 T0 更早，所以 B 先出队
```

#### 重要特性
- **租户内生效**：优先级只影响同一租户内的任务排序，不会跨租户抢占资源
- **负数优先级**：可用于延迟任务执行（相当于在未来时间才可见）
- **相对值**：优先级是相对的，具体数值不重要，重要的是相对大小

### 2.6 Worker 队列与拉取机制

**位置**：`packages/redis-worker/src/fair-queue/workerQueue.ts`

#### 三层队列结构
```
1. 消息队列（Message Queue）：
   ZSET，存储所有待处理消息，按优先级+时间排序
   
2. 主队列（Master Queue / Dispatch Index）：
   ZSET，存储有待处理消息的队列/租户，按最旧消息时间排序
   
3. Worker 队列（Worker Queue）：
   LIST，已认领但未被 Worker 拉取的消息
   使用 BLPOP 实现低延迟拉取
```

#### 消息流转过程

**入队**：
```
API 请求 → 令牌桶检查 → RunQueue.enqueueMessage()
  → 决定快路径/慢路径
    快路径：并发有剩余 → 直接入 Worker 队列
    慢路径：入消息队列 → 等待 Master 消费者调度
```

**调度认领**：
```
Master 消费者（每分片一个）:
  1. 从分发索引（Level 1）获取租户
  2. DRR 调度器选择租户
  3. 从租户索引（Level 2）获取队列
  4. 检查并发容量
  5. 批量认领消息（claimBatch）
  6. 预留并发槽位（reserve）
  7. 推入 Worker 队列
```

**Worker 拉取**：
```
Worker 进程:
  1. BLPOP 阻塞等待 Worker 队列
  2. 收到消息后，从 in-flight 存储获取完整消息数据
  3. 执行任务
  4. 完成后调用 completeMessage()
    → 释放并发槽位
    → 从 in-flight 移除
    → 如队列已空，从分发索引移除
```

#### 背压机制
Worker 队列深度限制（workerQueueMaxDepth）：
- 防止已认领的消息无限堆积
- 保护 Visibility Timeout（堆积过深可能导致消息在 Worker 拉取前就超时）
- 达到上限时，Master 消费者暂停认领

## 3. 多环境隔离机制

### 3.1 环境作为租户
在 Trigger.dev 中，**环境（Environment）是实际的调度租户单位**，而非组织（Organization）：

```typescript
// 队列 ID 格式
queueKey = "{orgId}:{projId}:{envId}:{queueName}"

// 提取租户 ID（即环境 ID）
extractTenantId = (queueId) => {
  const parts = queueId.split(":");
  return parts[2]; // envId
}
```

这意味着：
- 同一组织的不同环境（如 production、staging、dev）被视为独立租户
- 它们各自拥有独立的并发限制和调度配额
- 生产环境不会被开发环境的任务阻塞

### 3.2 环境级并发配置

每个环境可独立配置：
- `maximumConcurrencyLimit`：基础并发上限
- `concurrencyLimitBurstFactor`：突发因子（如 1.5 表示允许短时 150% 负载）
- `queueSizeLimit`：队列最大长度（按环境类型区分 dev/prod）

### 3.3 Worker 队列路由

**位置**：`internal-packages/run-engine/src/run-queue/workerQueueResolver.ts`

支持通过配置将特定环境/项目/组织的任务路由到专用 Worker 队列：

```typescript
// 优先级：environmentId > projectId > orgId > workerQueue
overrideConfig = {
  environmentId: { "env_123": "high-priority-queue" },
  projectId: { "proj_456": "project-dedicated-queue" },
  orgId: { "org_789": "org-shared-queue" },
}
```

这为重要客户或关键环境提供了物理隔离能力。

## 4. 完整协同流程

### 4.1 消息入队到出队的全链路

```
1. API 请求到达
   ├─ 令牌桶限流检查（按组织配置）
   └─ 通过 → 进入 RunQueue

2. RunQueue 入队
   ├─ 生成消息 payload（包含 priorityMs）
   ├─ 检查快路径条件：
   │  ├─ 环境并发是否有剩余
   │  ├─ 队列是否为空
   │  └─ 如满足 → 直接推入 Worker 队列
   └─ 否则 → 进入消息队列（ZSET，score = timestamp - priorityMs）

3. Master 消费者调度
   ├─ 从 Level 1 分发索引获取活跃租户（环境）
   ├─ DRR 调度器：
   │  ├─ 为每个租户增加 quantum 到 deficit
   │  ├─ 按 deficit 排序
   │  └─ 选择有可用容量的租户
   ├─ 从 Level 2 获取该租户的队列（按优先级+时间排序）
   ├─ 批量认领消息（claimBatch）
   │  ├─ 设置 Visibility Timeout
   │  ├─ 原子预留并发槽位（ConcurrencyManager.reserve）
   │  └─ 推入 Worker 队列
   └─ 租户 deficit -= 处理的消息数

4. Worker 拉取
   ├─ BLPOP 阻塞等待 Worker 队列
   ├─ 获取 messageKey → 从 in-flight 存储读取完整消息
   ├─ 执行任务
   └─ 完成/失败处理：
      ├─ 成功 → completeMessage()
      │  ├─ 释放并发槽位
      │  └─ 清理 in-flight 数据
      ├─ 失败 → failMessage()
      │  ├─ 如重试次数 < maxAttempts
      │  │  ├─ 计算下一次重试延迟（指数退避）
      │  │  └─ 重新入队（score = now + delay）
      │  └─ 否则 → 移入死信队列（DLQ）
      └─ 如队列已空 → 从分发索引移除租户
```

### 4.2 并发与优先级协同的关键场景

#### 场景 1：高优先级任务在并发满时的行为

假设：
- 租户 A 并发限制 = 10，当前已用 = 10
- 新任务到达，priority = 3600（1小时）

结果：
- 任务进入消息队列，score 被设置为 `timestamp - 3600`
- 由于并发已满，任务在队列中等待
- 当有并发槽位释放时，由于该任务的 score 更小（更"旧"），它会被优先调度
- 优先级只影响排序，不绕过并发限制

#### 场景 2：多租户公平性

假设：
- 租户 1：1000 个任务排队，并发限制 = 10
- 租户 2：1 个任务排队，并发限制 = 10
- DRR quantum = 5

调度行为：
- 每轮两个租户各获得 5 个消息的配额
- 实际每轮租户 1 处理 5 个，租户 2 处理 1 个（只有 1 个任务）
- 租户 2 的任务不会被租户 1 的大量任务饿死
- 租户 2 完成后，其赤字会被重置，不再参与调度

#### 场景 3：突发流量处理

假设：
- 租户并发限制 = 10，burstFactor = 1.5
- 短时内到达 100 个任务

处理：
- 令牌桶限制入队速度（如每分钟最多 60 个）
- 入队后，消息队列存储所有 100 个任务
- 并发管理器允许最多 15 个任务同时执行（10 × 1.5）
- 其余 85 个在队列中等待，按优先级+时间排序
- 通过 DRR 调度，确保该租户不会影响其他租户

## 5. 关键特性总结

| 机制 | 作用层面 | 核心目标 | 实现方式 |
|------|---------|---------|---------|
| 令牌桶 | API 入口 | 限制入队频率 | 按 token 速率消耗，定时补充 |
| 并发管理器 | 调度前 | 限制执行并发 | 多组并发计数器，原子预留/释放 |
| 两级分发索引 | 调度层 | 提高调度效率 | Level 1 租户索引，Level 2 队列索引 |
| DRR 调度器 | 调度层 | 租户间公平 | 量子+赤字的轮询算法 |
| 优先级 | 队列内 | 租户内差异化 | 时间偏移量调整排序 score |
| Worker 队列 | 执行层 | 低延迟拉取 | BLPOP 阻塞列表 |
| 环境隔离 | 全链路 | 环境间资源隔离 | 环境作为调度租户单位 |

## 6. 设计权衡

### 优点
1. **分层控制**：每一层解决特定问题，职责清晰
2. **可观测性**：每层都有独立的指标和日志
3. **可扩展性**：分片架构支持水平扩展
4. **灵活性**：多种调度策略可配置，支持差异化服务
5. **健壮性**：Visibility Timeout + 死信队列保证消息不丢失

### 权衡点
1. **最终一致性**：由于分片和缓存，各层状态可能短暂不一致
2. **延迟开销**：多层检查增加了单任务的端到端延迟（通常毫秒级）
3. **配置复杂度**：大量可调参数需要合理配置才能达到最佳性能
4. **优先级限制**：优先级只在租户内生效，无法跨租户抢占资源（设计使然，防止滥用）
