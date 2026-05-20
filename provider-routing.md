# Provider 路由与调度机制详解

## 概述

Trigger.dev v3 采用了多执行 Provider 的架构设计，支持 Docker 和 Kubernetes 两种主要的任务执行环境。任务在不同 Provider 之间的分派逻辑涉及 **Provider 能力声明**、**调度判定** 和 **回退路径** 三个核心环节。

> **重要修正**：经过深入代码分析，发现之前对 Provider 路由机制的理解存在多处错误。本文档将准确描述实际的架构设计和潜在问题。

---

## 一、Provider 类型与能力声明

### 1.1 Provider 类型

系统目前支持两种主要的 Provider 实现：

| Provider 类型 | 实现位置 | 适用场景 |
|--------------|----------|----------|
| Docker | `apps/docker-provider/src/index.ts` | 本地开发、单节点部署 |
| Kubernetes | `apps/kubernetes-provider/src/index.ts` | 生产环境、多节点集群 |

### 1.2 Provider Shell 抽象层

所有 Provider 都通过 `ProviderShell` 类进行统一封装，定义在 `packages/core/src/v3/apps/provider.ts`。

```typescript
// ProviderShell 构造函数签名
class ProviderShell {
  constructor(options: {
    tasks: TaskOperations;    // 任务操作实现
    type: "docker" | "kubernetes";  // Provider 类型
    host?: string;
    port?: number;
  });
}
```

### 1.3 TaskOperations 接口

每个 Provider 必须实现 `TaskOperations` 接口来声明其能力：

```typescript
interface TaskOperations {
  init: () => Promise<any>;                          // 初始化
  index: (opts: TaskOperationsIndexOptions) => Promise<any>;     // 索引任务
  create: (opts: TaskOperationsCreateOptions) => Promise<any>;   // 创建运行
  restore: (opts: TaskOperationsRestoreOptions) => Promise<any>; // 从检查点恢复
  delete?: (...args: any[]) => Promise<any>;         // 删除（可选）
  get?: (...args: any[]) => Promise<any>;            // 查询（可选）
  prePullDeployment?: (opts: TaskOperationsPrePullDeploymentOptions) => Promise<any>;  // 预拉取镜像（可选）
}
```

### 1.4 能力声明机制

#### Docker Provider 的能力探测

Docker Provider 在初始化时会自动探测 checkpoint 能力：

```typescript
// apps/docker-provider/src/index.ts:40-55
async init(): Promise<TaskOperationsInitReturn> {
  const testCheckpoint = await testDockerCheckpoint();
  
  if (testCheckpoint.ok) {
    return this.#getInitReturn(true);  // 支持完整 checkpoint
  }
  
  return this.#getInitReturn(false);   // 降级为模拟模式
}
```

能力声明结果：
- `canCheckpoint: boolean` - 是否支持真实 checkpoint
- `willSimulate: boolean` - 是否处于模拟模式（暂停/恢复容器）

#### Kubernetes Provider 的能力

Kubernetes Provider 提供更丰富的能力：
- 支持 `prePullDeployment` - 预拉取镜像到所有节点
- 支持 Pod 资源限制（CPU、内存、临时存储）
- 支持自定义标签和亲和性调度
- 内置 Pod 清理器和任务监控器

### 1.5 Provider 连接与类型声明

Provider 会建立 **两个独立的 WebSocket 连接** 到平台：

#### 连接 1：/provider namespace（控制通道）

```typescript
// packages/core/src/v3/apps/provider.ts:183-194
#createPlatformSocket() {
  const platformConnection = new ZodSocketConnection({
    namespace: "provider",
    host: PLATFORM_HOST,
    port: Number(PLATFORM_WS_PORT),
    extraHeaders: {
      "x-trigger-provider-type": this.options.type,  // 声明 Provider 类型
    },
    // ...
  });
}
```

**用途**：
- 接收 `INDEX` 消息进行部署索引
- 上报 `WORKER_CRASHED`、`INDEXING_FAILED` 等状态
- **x-trigger-provider-type 仅在此连接中发送**

#### 连接 2：/shared-queue namespace（数据通道）

```typescript
// packages/core/src/v3/apps/provider.ts:125-181
#createSharedQueueSocket() {
  const sharedQueueConnection = new ZodSocketConnection({
    namespace: "shared-queue",
    host: PLATFORM_HOST,
    port: Number(PLATFORM_WS_PORT),
    // 注意：这里没有发送 x-trigger-provider-type header！
    handlers: {
      SERVER_READY: async (message) => {
        await sender.send("READY_FOR_TASKS", {
          backgroundWorkerId: "placeholder",
        });
      },
      BACKGROUND_WORKER_MESSAGE: async (message) => {
        if (message.data.type === "SCHEDULE_ATTEMPT") {
          try {
            await this.tasks.create({...});  // 执行任务
          } catch (error) {
            logger.error("create failed", error);
          }
        }
      },
    },
  });
}
```

**关键发现**：
- `/shared-queue` 连接 **不发送** `x-trigger-provider-type` header
- 平台端在处理 `/shared-queue` 连接时 **完全不使用** Provider 类型信息
- **结论：Provider 类型声明不参与平台侧的调度判定**

#### 平台端验证

在 `apps/webapp/app/v3/handleSocketIo.server.ts` 中：

```typescript
// /provider namespace 的 postAuth 只处理 x-supports-dynamic-config
postAuth: async (socket, next, logger) => {
  // 只读取 x-supports-dynamic-config，不读取 x-trigger-provider-type
  setSocketDataFromHeader("supportsDynamicConfig", "x-supports-dynamic-config", false);
  // ...
}

// /shared-queue namespace 没有 postAuth，完全不读取任何 header
function createSharedQueueConsumerNamespace(io: Server) {
  const sharedQueue = new ZodNamespace({
    name: "shared-queue",
    onConnection: async (socket, handler, sender, logger) => {
      // 直接创建 SharedSocketConnection，不检查 provider 类型
      const sharedSocketConnection = new SharedSocketConnection({...});
    },
  });
}
```

**最终结论**：`x-trigger-provider-type` header 仅用于标识，**不参与任何调度决策**。

---

## 二、任务调度判定流程

### 2.1 整体架构（修正版）

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Platform (webapp)                               │
│                                                                         │
│  ┌─────────────────────────────┐  ┌─────────────────────────────┐       │
│  │   SharedSocketConnection    │  │   SharedSocketConnection    │       │
│  │ (Provider A 连接建立时创建) │  │ (Provider B 连接建立时创建) │       │
│  │                             │  │                             │       │
│  │  _sender: namespace.emit()  │  │  _sender: namespace.emit()  │       │
│  │  ConsumerPool (10 个消费者) │  │  ConsumerPool (10 个消费者) │       │
│  └──────────────┬──────────────┘  └──────────────┬──────────────┘       │
│                 │                                │                       │
│                 └────────────────┬───────────────┘                       │
│                                  │                                       │
│                                  ▼                                       │
│                    Redis 共享队列 (LPOP 原子操作)                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
                         消息被某个消费者获取
                                   │
                                   ▼
                         通过 namespace.emit() 广播
                                   │
              ┌────────────────────┴────────────────────┐
              ▼                                         ▼
     Provider A 执行任务                        Provider B 执行任务
```

### 2.2 关键架构特征

#### 特征 1：每个 Provider 连接创建独立的消费者池

```typescript
// apps/webapp/app/v3/sharedSocketConnection.ts:67-100
constructor(opts: SharedSocketConnectionOptions) {
  this._sender = new ZodMessageSender({
    schema: serverWebsocketMessages,
    sender: async (message) => {
      const { type, ...payload } = message;
      opts.namespace.emit(type, payload as any);  // 广播到整个 namespace
    },
    canSendMessage() {
      // 只要 namespace 有至少一个连接就返回 true
      return opts.namespace.sockets.size > 0;
    },
  });

  // 每个连接创建独立的消费者池
  this._sharedQueueConsumerPool = new SharedQueueConsumerPool({
    poolSize: opts.poolSize ?? this._defaultPoolSize,  // 默认 10
    sender: this._sender,
  });
}
```

#### 特征 2：所有消费者池共享同一个 Redis 队列

```typescript
// apps/webapp/app/v3/marqs/index.server.ts:654-746
public async dequeueMessageFromSharedWorkerQueue(consumerId: string) {
  // 从同一个 Redis list 中 LPOP（原子操作）
  const messageId = await this.redis.popMessageFromWorkerQueue(workerQueueKey);
  // ...
}
```

Redis 的 `LPOP` 是原子操作，确保每条消息只会被一个消费者获取。

#### 特征 3：消息通过广播发送到所有 Provider

```typescript
// apps/webapp/app/v3/sharedSocketConnection.ts:72-81
sender: async (message) => {
  return new Promise((resolve, reject) => {
    try {
      const { type, ...payload } = message;
      opts.namespace.emit(type, payload as any);  // 广播到所有连接的客户端
      resolve();
    } catch (err) {
      reject(err);
    }
  });
},
```

**潜在问题**：如果有 N 个 Provider 连接到 `/shared-queue` namespace，同一条 `SCHEDULE_ATTEMPT` 消息会被广播到所有 N 个 Provider，导致 **同一个任务被执行 N 次**！

### 2.3 SharedQueueConsumer 调度流程

调度逻辑主要在 `apps/webapp/app/v3/marqs/sharedQueueConsumer.server.ts` 中实现。

#### 阶段 1：消息出队与基本校验

```typescript
// sharedQueueConsumer.server.ts:427-449
async #doWorkInternal(): Promise<DoWorkInternalResult> {
  // 从共享队列尝试出队消息
  const message = await marqs?.dequeueMessageFromSharedWorkerQueue(this._id);
  
  if (!message) {
    return {
      reason: "no_message_dequeued",
      outcome: "noop",
      interval: this._options.nextTickInterval,
    };
  }
  // ...
}
```

#### 阶段 2：任务状态校验

```typescript
// sharedQueueConsumer.server.ts:572-616
const EXECUTABLE_RUN_STATUSES = {
  fromCheckpoint: ["WAITING_TO_RESUME"] satisfies TaskRunStatus[],
  withoutCheckpoint: ["PENDING", "RETRYING_AFTER_FAILURE"] satisfies TaskRunStatus[],
};

// 校验任务状态是否可执行
if ((retryingFromCheckpoint && !EXECUTABLE_RUN_STATUSES.fromCheckpoint.includes(status)) ||
    (!retryingFromCheckpoint && !EXECUTABLE_RUN_STATUSES.withoutCheckpoint.includes(status))) {
  return { action: "ack_and_do_more_work", reason: "invalid_run_status" };
}
```

#### 阶段 3：部署匹配

```typescript
// sharedQueueConsumer.server.ts:618-651
const deployment = existingTaskRun.lockedById
  ? await getWorkerDeploymentFromWorkerTask(existingTaskRun.lockedById)
  : existingTaskRun.lockedToVersionId
  ? await getWorkerDeploymentFromWorker(existingTaskRun.lockedToVersionId)
  : await findCurrentWorkerDeployment({
      environmentId: existingTaskRun.runtimeEnvironmentId,
      type: "V1",
    });

if (!deployment || !worker) {
  await this.#markRunAsWaitingForDeploy(existingTaskRun.id);
  return { action: "ack_and_do_more_work", reason: "no_matching_deployment" };
}
```

#### 阶段 4：任务存在性校验

```typescript
// sharedQueueConsumer.server.ts:673-718
const backgroundTask = worker.tasks.find(
  (task) => task.slug === existingTaskRun.taskIdentifier
);

if (!backgroundTask) {
  await this.#markRunAsWaitingForDeploy(existingTaskRun.id);
  return { action: "ack_and_do_more_work", reason: "task_not_deployed" };
}
```

#### 阶段 5：锁定任务

```typescript
// sharedQueueConsumer.server.ts:720-783
const lockedTaskRun = await prisma.taskRun.update({
  where: { id: message.messageId },
  data: {
    lockedAt: new Date(),
    lockedById: backgroundTask.id,
    lockedToVersionId: worker.id,
    taskVersion: worker.version,
    sdkVersion: worker.sdkVersion,
    cliVersion: worker.cliVersion,
    startedAt: startedAt,
    machinePreset: machinePreset,
    maxDurationInSeconds,
  },
  // ...
});
```

#### 阶段 6：发送到 Provider 执行

```typescript
// sharedQueueConsumer.server.ts:915-958
return await this.#startActiveSpan("scheduleAttemptOnProvider", async (span) => {
  if (await this._providerSender.validateCanSendMessage()) {
    await this._providerSender.send("BACKGROUND_WORKER_MESSAGE", {
      backgroundWorkerId: worker.friendlyId,
      data: {
        type: "SCHEDULE_ATTEMPT",
        image: imageReference,
        version: deployment.version,
        machine,
        nextAttemptNumber,
        envId: lockedTaskRun.runtimeEnvironment.id,
        envType: lockedTaskRun.runtimeEnvironment.type,
        orgId: lockedTaskRun.runtimeEnvironment.organizationId,
        projectId: lockedTaskRun.runtimeEnvironment.projectId,
        runId: lockedTaskRun.id,
        dequeuedAt: dequeuedAt.getTime(),
      },
    });

    return { action: "noop", reason: "scheduled_attempt" };
  } else {
    return {
      action: "nack_and_do_more_work",
      reason: "provider_not_connected",
      interval: this._options.nextTickInterval,
      retryInMs: 5_000,
    };
  }
});
```

### 2.4 Provider 端执行流程

Provider 收到 `SCHEDULE_ATTEMPT` 消息后，根据自身类型执行任务：

#### Docker Provider 执行

```typescript
// apps/docker-provider/src/index.ts:111-155
async create(opts: TaskOperationsCreateOptions) {
  const runArgs = [
    "run",
    `--network=${DOCKER_NETWORK}`,
    "--detach",
    `--env=TRIGGER_ENV_ID=${opts.envId}`,
    `--env=TRIGGER_RUN_ID=${opts.runId}`,
    // ... 其他环境变量
    `--name=${containerName}`,
    `${opts.image}`,
  ];

  // 机器规格限制（可选）
  if (process.env.ENFORCE_MACHINE_PRESETS) {
    runArgs.push(`--cpus=${opts.machine.cpu}`, `--memory=${opts.machine.memory}G`);
  }

  await execa("docker", runArgs);
}
```

#### Kubernetes Provider 执行

```typescript
// apps/kubernetes-provider/src/index.ts:162-227
async create(opts: TaskOperationsCreateOptions) {
  await this.#createPod({
    metadata: {
      name: containerName,
      labels: {
        app: "task-run",
        run: opts.runId,
        env: opts.envId,
        // ... 其他标签
      },
    },
    spec: {
      containers: [{
        name: containerName,
        image: getImageRef("deployment", opts.image),
        resources: this.#getResourcesForMachine(opts.machine),
        // ... 其他配置
      }],
    },
  });
}
```

---

## 三、回退路径与容错机制（按阶段梳理，三段式说明）

### 3.0 核心架构：Ack/Nack 统一在外层执行

#### 事实结论
- `#handleExecuteMessage`、`#handleResumeMessage` 等处理函数 **只返回 `{ action, reason, ... }` 结果**
- 真正的 `ack`/`nack` 操作 **统一在外层 `#doWorkInternal()` 的 switch 语句中执行**
- 各分支内部不会直接调用 `marqs.acknowledgeMessage()` 或 `marqs.nackMessage()`

#### 证据位置
```typescript
// sharedQueueConsumer.server.ts:427-535
async #doWorkInternal(): Promise<DoWorkInternalResult> {
  // ... 消息出队 ...
  
  const messageResult = await this.#handleMessage(message, messageBody.data, dequeuedAt);
  
  // ✅ 所有 ack/nack 统一在这里执行
  switch (messageResult.action) {
    case "noop": { /* ... */ }
    case "ack_and_do_more_work": {
      await this.#ack(message.messageId);  // 统一 ack
      return { ... };
    }
    case "nack_and_do_more_work": {
      await this.#nack(message.messageId, messageResult.retryInMs);  // 统一 nack
      return { ... };
    }
    case "nack": {
      await marqs?.nackMessage(message.messageId);
      return { ... };
    }
  }
}
```

#### 未证实风险说明
无 - 此为明确的代码事实。

---

### 3.1 部署匹配阶段回退

此阶段发生在消息出队后，发送到 Provider 之前。

#### 回退 1.1：无效任务状态

**事实结论**：返回 `ack_and_do_more_work`，外层统一执行 ack，消息从队列永久移除。

**证据位置**：
```typescript
// sharedQueueConsumer.server.ts:597-616
if (
  (retryingFromCheckpoint &&
    !EXECUTABLE_RUN_STATUSES.fromCheckpoint.includes(existingTaskRun.status)) ||
  (!retryingFromCheckpoint &&
    !EXECUTABLE_RUN_STATUSES.withoutCheckpoint.includes(existingTaskRun.status))
) {
  return {
    action: "ack_and_do_more_work",
    reason: "invalid_run_status",
    attrs: { status: existingTaskRun.status, retryingFromCheckpoint },
  };
}
```

**未证实风险说明**：无 - 任务状态无效直接丢弃是预期行为。

---

#### 回退 1.2：无匹配部署

**事实结论**：标记任务为 `WAITING_FOR_DEPLOY` + 返回 `ack_and_do_more_work`，外层统一执行 ack。

**证据位置**：
```typescript
// sharedQueueConsumer.server.ts:632-651
if (!deployment || !worker) {
  await this.#markRunAsWaitingForDeploy(existingTaskRun.id);
  return {
    action: "ack_and_do_more_work",
    reason: "no_matching_deployment",
    attrs: {
      run_id: existingTaskRun.id,
      locked_by_id: existingTaskRun.lockedById ?? undefined,
      locked_to_version_id: existingTaskRun.lockedToVersionId ?? undefined,
      environment_id: existingTaskRun.runtimeEnvironmentId,
    },
  };
}
```

**未证实风险说明**：
- 代码注释说明 "This happens when a run is 'WAITING_FOR_DEPLOY' and is expected"，表明这是预期行为
- 部署完成后任务是否会被重新入队，需要查看部署索引完成后的代码逻辑，此处未证实

---

#### 回退 1.3：任务未部署

**事实结论**：标记任务为 `WAITING_FOR_DEPLOY` + 返回 `ack_and_do_more_work`，外层统一执行 ack。

**证据位置**：
```typescript
// sharedQueueConsumer.server.ts:707-717
await this.#markRunAsWaitingForDeploy(existingTaskRun.id);
return {
  action: "ack_and_do_more_work",
  reason: "task_not_deployed",
  attrs: {
    run_id: existingTaskRun.id,
    task_identifier: existingTaskRun.taskIdentifier,
  },
};
```

**未证实风险说明**：
- 代码注释说明 "If this task is ever deployed, a new message will be enqueued after successful indexing"，表明预期会重新入队
- 但部署索引完成后的重新入队逻辑此处未验证

---

#### 回退 1.4：锁定失败

**事实结论**：返回 `ack_and_do_more_work`，外层统一执行 ack。

**证据位置**：
```typescript
// sharedQueueConsumer.server.ts:763-783
if (!lockedTaskRun) {
  return {
    action: "ack_and_do_more_work",
    reason: "failed_to_lock_task_run",
    attrs: {
      run_id: existingTaskRun.id,
      task_identifier: existingTaskRun.taskIdentifier,
      deployment_id: deployment.id,
      background_worker_id: worker.id,
      message_id: message.messageId,
    },
  };
}
```

**未证实风险说明**：
- 锁定失败可能是因为任务被其他消费者抢先锁定，这是正常的并发竞争
- 但也可能是因为任务之前被锁定后未正确解锁（如 provider_not_connected 场景）

---

#### 回退 1.5：Checkpoint 恢复失败（EXECUTE 路径）

**事实结论**：返回 `ack_and_do_more_work`，外层统一执行 ack。

**证据位置**：
```typescript
// sharedQueueConsumer.server.ts:840-858
if (!checkpoint) {
  return {
    action: "ack_and_do_more_work",
    reason: "failed_to_restore_checkpoint",
    attrs: {
      checkpoint_event_id: data.checkpointEventId,
      run_status: lockedTaskRun.status,
      is_retry: isRetry,
    },
  };
}
```

**未证实风险说明**：
- Checkpoint 恢复失败后任务被 ack 丢弃，没有重试机制
- 任务可能因此永久卡住，需要人工介入

---

### 3.2 发送执行阶段回退

此阶段发生在任务锁定后，发送到 Provider 时。

#### 回退 2.1：Provider 未连接

**事实结论**：
1. 任务已被锁定（`lockedAt`、`lockedById`、`lockedToVersionId` 已设置）
2. 仅返回 `nack_and_do_more_work`，**不解锁任务**
3. 外层统一执行 nack，消息 5 秒后重新入队
4. 任务状态 **保持不变**（仍是 PENDING 或 RETRYING_AFTER_FAILURE）

**证据位置**：
```typescript
// sharedQueueConsumer.server.ts:947-956
} else {
  // ⚠️  这里没有解锁操作！
  return {
    action: "nack_and_do_more_work",
    reason: "provider_not_connected",
    attrs: { run_id: lockedTaskRun.id },
    interval: this._options.nextTickInterval,
    retryInMs: 5_000,
  };
}
```

**未证实风险说明**：
- ❌ **之前的推测错误**："下次重试时因锁定失败而被丢弃" 是推测，不是代码事实
- ✅ **实际可证实的流程**：
  1. 5 秒后消息重新出队
  2. `prisma.taskRun.update()` 尝试锁定（WHERE id = ?）
  3. 由于 `lockedAt` 不为 NULL，但 update **不检查** `lockedAt IS NULL`，锁定会成功
  4. 任务会被重新处理
- 唯一可证实的风险：任务保持锁定状态，如果 visibility timeout 到期会自动解锁

---

#### 回退 2.2：发送异常（唯一显式解锁路径）

**事实结论**：
1. 显式解锁任务（`lockedAt`、`lockedById` 设为 NULL）
2. 恢复任务状态和 startedAt 到锁定前的值
3. 返回 `nack_and_do_more_work`，外层统一执行 nack，消息 5 秒后重新入队

**证据位置**：
```typescript
// sharedQueueConsumer.server.ts:959-987
catch (e) {
  // ✅ 唯一的显式解锁路径
  await prisma.$transaction([
    prisma.taskRun.update({
      where: { id: lockedTaskRun.id },
      data: {
        lockedAt: null,       // 解锁
        lockedById: null,     // 解锁
        status: lockedTaskRun.status,
        startedAt: existingTaskRun.startedAt,
      },
    }),
  ]);

  return {
    action: "nack_and_do_more_work",
    reason: "failed_to_schedule_attempt",
    error: e instanceof Error ? e : String(e),
    interval: this._options.nextTickInterval,
    retryInMs: 5_000,
  };
}
```

**未证实风险说明**：无 - 此为明确的代码事实。

---

### 3.3 Checkpoint 恢复阶段回退

此阶段发生在任务执行过程中或恢复时。

#### 回退 3.1：Docker Checkpoint 能力降级

**事实结论**：当 Docker 不支持 checkpoint 时，自动降级为使用 pause/unpause 模拟。

**证据位置**：
```typescript
// apps/docker-provider/src/index.ts:157-184
async restore(opts: TaskOperationsRestoreOptions) {
  if (!this.#canCheckpoint || this.opts.forceSimulate) {
    logger.log("Simulating restore");
    await $`docker unpause ${containerName}`;
    await this.#sendPostStart(containerName);
    return;
  }

  await $`docker start --checkpoint=${opts.checkpointRef} ${containerName}`;
}
```

**未证实风险说明**：无 - 此为明确的代码事实。

---

#### 回退 3.2：RESUME 消息恢复失败

**事实结论**：
- 恢复成功 → 返回 `noop`
- 恢复失败 → 返回 `ack_and_do_more_work`，外层统一执行 ack
- 发生异常 → 返回 `nack_and_do_more_work`，外层统一执行 nack

**证据位置**：
```typescript
// sharedQueueConsumer.server.ts:995-1031
if (data.checkpointEventId) {
  try {
    const checkpoint = await restoreService.call({ eventId: data.checkpointEventId });
    if (!checkpoint) {
      return {
        action: "ack_and_do_more_work",
        reason: "failed_to_restore_checkpoint",
      };
    }
    return { action: "noop", reason: "restored_checkpoint" };
  } catch (e) {
    return {
      action: "nack_and_do_more_work",
      reason: "failed_to_restore_checkpoint",
      error: e instanceof Error ? e : String(e),
    };
  }
}
```

**未证实风险说明**：
- 恢复失败后任务被 ack 丢弃，没有重试机制
- 但任务状态仍为 `WAITING_TO_RESUME`，可能通过其他路径重试

---

### 3.4 回退路径汇总表（事实版）

| 阶段 | 回退场景 | 返回 action | 解锁操作 | 外层执行 | 重试机制 | 事实/推测 |
|------|----------|------------|----------|----------|----------|----------|
| **部署匹配阶段** | 无效任务状态 | `ack_and_do_more_work` | 无 | ack | 无 | 事实 |
| | 无匹配部署 | `ack_and_do_more_work` | 无 | ack | 等待新部署（注释说明） | 部分推测 |
| | 任务未部署 | `ack_and_do_more_work` | 无 | ack | 等待新部署（注释说明） | 部分推测 |
| | 锁定失败 | `ack_and_do_more_work` | 无 | ack | 无 | 事实 |
| | Checkpoint 恢复失败 | `ack_and_do_more_work` | 无 | ack | 无 | 事实 |
| **发送执行阶段** | Provider 未连接 | `nack_and_do_more_work` | ❌ 无 | nack | 5秒后重试 | 事实 |
| | 发送异常（catch） | `nack_and_do_more_work` | ✅ 显式解锁 | nack | 5秒后重试 | 事实 |
| **Checkpoint 恢复阶段** | Docker 不支持 checkpoint | 无 | 无 | 无 | 自动降级 | 事实 |
| | RESUME 恢复失败 | `ack_and_do_more_work` | 无 | ack | 无 | 事实 |
| | RESUME 恢复异常 | `nack_and_do_more_work` | 无 | nack | 立即重试 | 事实 |

---

### 3.5 关键发现：锁定逻辑的不一致性

#### 事实结论
- 发送异常（catch 路径）：✅ 显式解锁任务
- Provider 未连接：❌ 不解锁任务

#### 证据位置
- 解锁：`sharedQueueConsumer.server.ts:961-973`
- 不解锁：`sharedQueueConsumer.server.ts:947-956`

#### 未证实风险说明
- 两种场景都是"发送失败"，但处理策略不一致
- provider_not_connected 时任务保持锁定，依赖 visibility timeout 自动解锁（默认 5 分钟）
- 这可能导致：
  1. 任务在 5 分钟内无法被其他消费者处理
  2. 如果 visibility timeout 机制失效，任务可能永久锁定
  3. 但 **不会** 导致"下次重试因锁定失败被丢弃"（之前的推测错误）

---

### 3.6 更正：provider_not_connected 后重试不会因锁定失败被丢弃

#### 之前的错误推测
```
provider_not_connected → 不解锁 → 5秒后重试 → 锁定失败 → 丢弃
```

#### 实际可证实的代码事实
```typescript
// sharedQueueConsumer.server.ts:731-746
const lockedTaskRun = await prisma.taskRun.update({
  where: {
    id: message.messageId,
    // ⚠️  注意：这里没有 AND lockedAt IS NULL 条件！
  },
  data: {
    lockedAt,       // 直接覆盖，不管之前是否锁定
    lockedById,     // 直接覆盖
    lockedToVersionId,
    // ...
  },
  // ...
});
```

**关键事实**：`prisma.taskRun.update()` 的 where 条件 **只有 `id`**，没有 `lockedAt IS NULL`。
- 即使任务已被锁定，update 仍然会成功
- 新的 `lockedAt` 会直接覆盖旧值
- 任务会被重新处理，不会因锁定失败被丢弃

**修正后的结论**：provider_not_connected 后重试不会因锁定失败被丢弃，任务会被正常重新处理。唯一的问题是任务会保持锁定状态直到 visibility timeout 或被重新锁定。

---

## 四、关键设计取舍分析

### 4.1 消息确认机制（Ack/Nack）

#### 核心设计：统一外层执行

```
#handleExecuteMessage() / #handleResumeMessage()
    ↓
只返回 { action, reason, ... }
    ↓
#doWorkInternal() switch 语句
    ↓
统一执行 ack / nack
```

| 操作 | 含义 | 使用场景 |
|------|------|----------|
| `ack` | 确认消息，从队列永久移除 | 状态无效、配置缺失、锁定失败 |
| `nack` | 拒绝消息，立即重新入队 | 消费者未启用 |
| `nack + retryInMs` | 延迟指定时间后重试 | Provider 不可用、调度异常 |

### 4.2 任务锁定策略（修正版）

**核心事实**：
- 不是"乐观锁定"，而是"覆盖式锁定"
- `prisma.taskRun.update()` 的 where 条件 **只有 `id`**，没有 `lockedAt IS NULL`
- 即使任务已被锁定，update 仍然会成功，新的 `lockedAt` 会直接覆盖旧值

**锁定信息**：`lockedAt`（锁定时间）、`lockedById`（锁定的任务 ID）、`lockedToVersionId`（锁定的部署版本）

**解锁时机**：
- ✅ 发送异常（catch 路径）：显式解锁
- ❌ Provider 未连接：不解锁
- ⏰ 超时：通过 visibility timeout 机制自动解锁（默认 5 分钟）

**关键发现**：
- 锁定策略不存在"锁定失败被丢弃"的问题
- 但解锁策略存在不一致性：同样是发送失败，catch 路径解锁，provider_not_connected 不解锁

### 4.3 Provider 选择策略（修正版）

当前实现 **没有任何 Provider 选择逻辑**：

1. 所有 Provider 连接到同一个 `shared-queue` namespace
2. 每个 Provider 连接创建独立的消费者池，共享同一个 Redis 队列
3. 消息通过 `namespace.emit()` 广播到 **所有** 连接的 Provider
4. **没有** 负载均衡、能力路由或任何类型的 Provider 选择

> **重要警告**：如果同时部署了多个 Provider（如 Docker + Kubernetes），同一个任务会被 **所有 Provider 同时执行**！这是当前架构的一个严重设计问题。

### 4.4 错误分类处理（修正版）

| 错误类型 | 处理策略 | 解锁 | 消息操作 | 回退路径 |
|----------|----------|------|----------|----------|
| Provider 未连接 | 返回 nack_and_do_more_work | ❌ 不解锁 | nack（5秒后重试） | 等待 Provider 重连 |
| 发送异常（catch） | 解锁 + 返回 nack_and_do_more_work | ✅ 解锁 | nack（5秒后重试） | 等待问题解决 |
| 配置缺失（无部署/任务） | 标记 WAITING_FOR_DEPLOY | 无 | ack | 等待新部署 |
| 状态冲突（状态无效） | 丢弃 | 无 | ack | 避免重复处理 |
| 数据损坏（Checkpoint 失效） | 丢弃 | 无 | ack | 需要人工介入 |
| 能力不足（Docker 不支持 checkpoint） | 自动降级 | 无 | 无 | 使用模拟模式继续执行 |

> **注意**：不存在"已锁定"导致的状态冲突，因为锁定是覆盖式的，不会失败。

---

## 五、代码位置索引

| 模块 | 文件路径 | 关键函数/类 |
|------|----------|------------|
| Provider 抽象 | `packages/core/src/v3/apps/provider.ts` | `ProviderShell`, `#createPlatformSocket`, `#createSharedQueueSocket` |
| Docker Provider | `apps/docker-provider/src/index.ts` | `DockerTaskOperations`, `testDockerCheckpoint` |
| Kubernetes Provider | `apps/kubernetes-provider/src/index.ts` | `KubernetesTaskOperations`, `#createPod` |
| 调度消费者 | `apps/webapp/app/v3/marqs/sharedQueueConsumer.server.ts` | `SharedQueueConsumer`, `#handleExecuteMessage`, `#doWorkInternal` |
| Socket 连接管理 | `apps/webapp/app/v3/handleSocketIo.server.ts` | `createProviderNamespace`, `createSharedQueueConsumerNamespace` |
| 共享连接 | `apps/webapp/app/v3/sharedSocketConnection.ts` | `SharedSocketConnection`, `SharedQueueConsumerPool` |
| 队列系统 | `apps/webapp/app/v3/marqs/index.server.ts` | `MarQS`, `dequeueMessageFromSharedWorkerQueue` |
| 消息定义 | `packages/core/src/v3/schemas/messages.ts` | `BackgroundWorkerServerMessages`, `SCHEDULE_ATTEMPT` |

---

## 六、架构问题与改进建议

### 6.1 当前架构的可证实问题

#### 问题 1：多 Provider 重复执行（已证实）

**事实结论**：
- 每个 Provider 连接创建独立的消费者池
- 消息通过 `namespace.emit()` 广播到所有连接的 Provider
- 如果有 N 个 Provider，同一个任务会被执行 N 次

**证据位置**：
- 广播发送：`sharedSocketConnection.ts:72-81` (`opts.namespace.emit(type, payload as any)`)
- Provider 接收执行：`provider.ts:141-160`

**影响**：
- 任务重复执行，产生副作用
- 资源浪费
- 数据不一致风险

---

#### 问题 2：解锁策略不一致（已证实）

**事实结论**：
- 发送异常（catch 路径）：✅ 显式解锁任务
- Provider 未连接：❌ 不解锁任务

**证据位置**：
- 解锁：`sharedQueueConsumer.server.ts:961-973`
- 不解锁：`sharedQueueConsumer.server.ts:947-956`

**影响**：
- provider_not_connected 时任务保持锁定，依赖 visibility timeout 自动解锁（默认 5 分钟）
- 任务在 5 分钟内无法被其他消费者处理
- 但不会导致任务丢失（因为锁定是覆盖式的，重试时会重新锁定成功）

---

#### 问题 3：缺少 Provider 能力感知调度（已证实）

**事实结论**：
- 调度时完全不考虑 Provider 的能力（如 checkpoint 支持、GPU 资源等）
- 任务通过广播发送到所有 Provider

**证据位置**：
- 发送逻辑：`sharedQueueConsumer.server.ts:920-938`（无 Provider 选择逻辑）
- 广播机制：`sharedSocketConnection.ts:72-81`

**影响**：
- 需要 checkpoint 的任务可能在不支持的 Provider 上失败
- 资源利用率低

---

#### 问题 4：缺少负载均衡（已证实）

**事实结论**：
- 没有任何负载均衡机制
- 所有 Provider 接收相同的任务广播

**证据位置**：
- 广播机制：`sharedSocketConnection.ts:72-81`
- 无 Provider 选择逻辑：`sharedQueueConsumer.server.ts:915-958`

**影响**：
- 某些 Provider 可能过载，而其他 Provider 空闲
- 系统整体吞吐量无法线性扩展

---

### 6.2 改进建议

#### 建议 1：修复解锁策略不一致性（高优先级）

```typescript
// 建议：在 provider_not_connected 分支也解锁任务
} else {
  // 解锁任务（与 catch 路径保持一致）
  await prisma.taskRun.update({
    where: { id: lockedTaskRun.id },
    data: {
      lockedAt: null,
      lockedById: null,
      status: lockedTaskRun.status,
      startedAt: existingTaskRun.startedAt,
    },
  });
  
  return {
    action: "nack_and_do_more_work",
    reason: "provider_not_connected",
    retryInMs: 5_000,
  };
}
```

**理由**：两种场景都是"发送失败"，应该采用一致的解锁策略。

---

#### 建议 2：引入 Provider 注册中心

- 跟踪每个 Provider 的类型、能力、负载和健康状态
- 在调度时根据任务需求选择合适的 Provider

---

#### 建议 3：实现点对点消息发送

- 替换 `namespace.emit()` 广播为定向发送
- 只将任务发送到选定的 Provider
- 避免多 Provider 重复执行问题

---

#### 建议 4：实现能力感知调度

- 在任务定义中声明所需能力（checkpoint、GPU、内存等）
- 调度时匹配 Provider 能力

---

#### 建议 5：增加负载均衡策略

- 实现轮询、最少连接、能力加权等负载均衡策略
- 支持 Provider 级别限流和熔断

---

#### 建议 6：增加 Provider 健康检查

- 定期检查 Provider 健康状态
- 及时从调度池中移除不健康的 Provider
