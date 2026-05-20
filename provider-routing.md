# Provider 路由与调度机制详解

## 概述

Trigger.dev v3 采用了多执行 Provider 的架构设计，支持 Docker 和 Kubernetes 两种主要的任务执行环境。任务在不同 Provider 之间的分派逻辑涉及 **Provider 能力声明**、**调度判定** 和 **回退路径** 三个核心环节。

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

### 1.5 Provider 注册与连接

Provider 通过 WebSocket 连接到平台，连接时声明自身类型：

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

平台端在 `apps/webapp/app/v3/handleSocketIo.server.ts:383-418` 中创建 Provider namespace 处理连接。

---

## 二、任务调度判定流程

### 2.1 整体架构

```
┌─────────────────┐     ┌────────────────────┐     ┌─────────────────┐
│   Run Engine    │────▶│   Shared Queue     │────▶│  Provider Pool  │
│  (任务生成器)   │     │   (消息队列)      │     │  (执行器池)    │
└─────────────────┘     └────────────────────┘     └─────────────────┘
          │                        │                         │
          ▼                        ▼                         ▼
  任务入队与优先级        消费与分发                 实际执行
```

### 2.2 SharedQueueConsumer 调度流程

调度逻辑主要在 `apps/webapp/app/v3/marqs/sharedQueueConsumer.server.ts` 中实现。

#### 步骤 1：消息出队

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

#### 步骤 2：任务状态校验

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

#### 步骤 3：部署匹配

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

#### 步骤 4：任务存在性校验

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

#### 步骤 5：锁定任务

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

#### 步骤 6：发送到 Provider 执行

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

### 2.3 Provider 端执行流程

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

## 三、回退路径与容错机制

### 3.1 Provider 连接失败回退

当 Provider 连接不可用时，任务会被 nack 并在延迟后重试：

```typescript
// sharedQueueConsumer.server.ts:947-956
if (await this._providerSender.validateCanSendMessage()) {
  // 发送到 Provider
} else {
  return {
    action: "nack_and_do_more_work",
    reason: "provider_not_connected",
    interval: this._options.nextTickInterval,
    retryInMs: 5_000,  // 5秒后重试
  };
}
```

### 3.2 部署缺失回退

当找不到匹配的部署时，任务被标记为 `WAITING_FOR_DEPLOY`：

```typescript
// sharedQueueConsumer.server.ts:632-651
if (!deployment || !worker) {
  await this.#markRunAsWaitingForDeploy(existingTaskRun.id);
  return {
    action: "ack_and_do_more_work",
    reason: "no_matching_deployment",
  };
}
```

### 3.3 任务未部署回退

当任务在当前部署中不存在时，同样标记为 `WAITING_FOR_DEPLOY`：

```typescript
// sharedQueueConsumer.server.ts:673-718
if (!backgroundTask) {
  await this.#markRunAsWaitingForDeploy(existingTaskRun.id);
  return {
    action: "ack_and_do_more_work",
    reason: "task_not_deployed",
  };
}
```

### 3.4 锁定失败回退

当任务锁定失败时（可能被其他消费者抢先锁定），任务被 ack 并跳过：

```typescript
// sharedQueueConsumer.server.ts:763-783
if (!lockedTaskRun) {
  return {
    action: "ack_and_do_more_work",
    reason: "failed_to_lock_task_run",
  };
}
```

### 3.5 调度异常回退

当发送到 Provider 过程中发生异常时，解锁任务并 nack 重试：

```typescript
// sharedQueueConsumer.server.ts:959-987
catch (e) {
  // 解锁任务
  await prisma.$transaction([
    prisma.taskRun.update({
      where: { id: lockedTaskRun.id },
      data: {
        lockedAt: null,
        lockedById: null,
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

### 3.6 Checkpoint 恢复失败回退

当从 checkpoint 恢复失败时，任务被 ack 并标记：

```typescript
// sharedQueueConsumer.server.ts:832-857
if (data.checkpointEventId) {
  const restoreService = new RestoreCheckpointService();
  const checkpoint = await restoreService.call({
    eventId: data.checkpointEventId,
    isRetry,
  });

  if (!checkpoint) {
    return {
      action: "ack_and_do_more_work",
      reason: "failed_to_restore_checkpoint",
    };
  }
}
```

### 3.7 Docker Checkpoint 能力降级

当 Docker 不支持 checkpoint 时，自动降级为模拟模式：

```typescript
// apps/docker-provider/src/index.ts:157-184
async restore(opts: TaskOperationsRestoreOptions) {
  if (!this.#canCheckpoint || this.opts.forceSimulate) {
    logger.log("Simulating restore");
    // 使用 pause/unpause 模拟 checkpoint
    await $`docker unpause ${containerName}`;
    await this.#sendPostStart(containerName);
    return;
  }

  // 真实 checkpoint 恢复
  await $`docker start --checkpoint=${opts.checkpointRef} ${containerName}`;
}
```

### 3.8 恢复消息回退（RESUME 路径）

当 coordinator 无法恢复任务时，尝试从 checkpoint 恢复：

```typescript
// sharedQueueConsumer.server.ts:1221-1298
if (resumableRun.status === "WAITING_TO_RESUME") {
  const restoreService = new RestoreCheckpointService();
  const checkpointEvent = await restoreService.getLastCheckpointEventIfUnrestored(
    resumableRun.id
  );

  if (checkpointEvent) {
    const checkpoint = await restoreService.call({
      eventId: checkpointEvent.id,
    });
    // ...
  }
}
```

---

## 四、关键设计取舍分析

### 4.1 消息确认机制（Ack/Nack）

| 操作 | 含义 | 使用场景 |
|------|------|----------|
| `ack` | 确认消息，从队列移除 | 任务已成功分派或状态无效 |
| `nack` | 拒绝消息，重新入队 | Provider 不可用、调度异常 |
| `nack + retryInMs` | 延迟重试 | 临时故障，需要等待恢复 |

### 4.2 任务锁定策略

- **乐观锁定**：通过数据库更新实现，失败则放弃
- **锁定信息**：`lockedAt`、`lockedById`、`lockedToVersionId`
- **解锁时机**：调度失败时立即解锁，避免任务永久卡住

### 4.3 Provider 选择策略

当前实现采用 **"隐式路由"** 策略：
1. 所有 Provider 连接到同一个 `shared-queue` namespace
2. `SharedSocketConnection` 的 sender 会广播到所有连接的 Provider
3. 实际由哪个 Provider 执行取决于 WebSocket 连接的广播机制
4. 没有显式的 Provider 负载均衡或能力路由（目前设计）

> **注意**：当前架构中，任务分派不区分 Docker 和 Kubernetes Provider。如果同时部署了多个 Provider，任务可能被发送到任意一个连接的 Provider。

### 4.4 错误分类处理

| 错误类型 | 处理策略 | 回退路径 |
|----------|----------|----------|
| 临时故障（Provider 断开） | Nack + 延迟重试 | 等待 Provider 重连 |
| 配置缺失（无部署/任务） | Ack + WAITING_FOR_DEPLOY | 等待新部署 |
| 状态冲突（已锁定/状态无效） | Ack + 丢弃 | 避免重复处理 |
| 数据损坏（Checkpoint 失效） | Ack + 标记 | 需要人工介入 |

---

## 五、代码位置索引

| 模块 | 文件路径 | 关键函数/类 |
|------|----------|------------|
| Provider 抽象 | `packages/core/src/v3/apps/provider.ts` | `ProviderShell`, `TaskOperations` |
| Docker Provider | `apps/docker-provider/src/index.ts` | `DockerTaskOperations` |
| Kubernetes Provider | `apps/kubernetes-provider/src/index.ts` | `KubernetesTaskOperations` |
| 调度消费者 | `apps/webapp/app/v3/marqs/sharedQueueConsumer.server.ts` | `SharedQueueConsumer`, `#handleExecuteMessage` |
| Socket 连接管理 | `apps/webapp/app/v3/handleSocketIo.server.ts` | `createProviderNamespace`, `createSharedQueueConsumerNamespace` |
| 共享连接 | `apps/webapp/app/v3/sharedSocketConnection.ts` | `SharedSocketConnection`, `SharedQueueConsumerPool` |
| 消息定义 | `packages/core/src/v3/schemas/messages.ts` | `BackgroundWorkerServerMessages`, `SCHEDULE_ATTEMPT` |

---

## 六、改进建议

### 当前架构潜在问题

1. **缺少显式 Provider 路由**：任务可能被发送到不具备相应能力的 Provider（如需要 checkpoint 但 Provider 不支持）

2. **多 Provider 负载均衡**：当前广播模式可能导致任务重复或负载不均

3. **能力感知调度**：调度时未考虑 Provider 的实际负载和能力

### 可能的优化方向

1. 引入 Provider 注册中心，跟踪每个 Provider 的能力和负载
2. 在调度时根据任务需求（如 checkpoint、GPU 等）选择合适的 Provider
3. 实现 Provider 级别限流和熔断机制
4. 增加 Provider 健康检查，及时移除不健康的 Provider
