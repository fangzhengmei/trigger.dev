# Trigger.dev 版本演进与可回滚机制：代码级深度分析报告

## 1. 概述

Trigger.dev 平台通过一套精密的版本化构建、原子部署和版本锁定机制，确保每次构建产物与版本号、运行环境、依赖快照之间建立不可变的关联关系，同时保证旧版本任务仍可被正确触发和执行。本报告基于代码层面，详细解析整个流程。

---

## 2. 核心数据模型与关联关系

### 2.1 关键实体关系

```
WorkerDeployment (部署记录)
    ├── version: "20250313.2"           ← 版本标识符
    ├── contentHash: "a1b2c3d4..."      ← 构建产物指纹
    ├── imageReference: "registry/img@sha256:..." ← 容器镜像引用
    ├── imagePlatform: "linux/amd64"
    ├── runtime: "node"                 ← 运行时类型
    ├── runtimeVersion: "20.11.0"       ← 运行时版本
    ├── commitSHA: "abc1234"            ← Git 提交哈希
    ├── git: { branch, ref, ... }       ← Git 元数据 (Json)
    ├── status: DEPLOYED                ← 部署状态
    ├── externalBuildData: Json?        ← 外部构建数据
    ├── buildServerMetadata: Json?      ← 构建服务器元数据
    │
    ├── workerId → BackgroundWorker     ← 1:1 关联
    │       ├── version: "20250313.2"           (与 Deployment 相同)
    │       ├── contentHash: "a1b2c3d4..."      (与 Deployment 相同)
    │       ├── sdkVersion: "3.3.0"
    │       ├── cliVersion: "3.3.0"
    │       ├── runtime: "node"
    │       ├── runtimeVersion: "20.11.0"
    │       ├── metadata: Json                  (任务调度信息)
    │       └── tasks → BackgroundWorkerTask[]  ← 包含的任务列表
    │               ├── slug: "my-task"         (taskIdentifier)
    │               ├── filePath: "src/tasks.ts"
    │               ├── queueConfig: Json?
    │               ├── retryConfig: Json?
    │               └── machineConfig: Json?
    │
    ├── environmentId → RuntimeEnvironment  ← 运行环境
    │       ├── slug: "prod"
    │       ├── type: "PRODUCTION" | "DEVELOPMENT" | "STAGING"
    │       └── git: Json?
    │
    └── promotions → WorkerDeploymentPromotion[]
            └── label: "current"           ← 标识当前活跃部署

TaskRun (任务运行实例)
    ├── taskIdentifier: "my-task"
    ├── lockedToVersionId → BackgroundWorker   ← 版本锁定
    ├── lockedById → BackgroundWorkerTask      ← 任务级锁定
    ├── taskVersion: "20250313.2"              ← 冗余存储的版本号
    ├── sdkVersion: "3.3.0"
    ├── cliVersion: "3.3.0"
    └── runtimeEnvironmentId → RuntimeEnvironment

TaskRunAttempt (执行尝试)
    ├── backgroundWorkerId → BackgroundWorker  ← 执行此尝试的 Worker
    ├── backgroundWorkerTaskId → BackgroundWorkerTask
    └── runtimeEnvironmentId → RuntimeEnvironment

Checkpoint (检查点)
    ├── imageRef: string                       ← 容器镜像引用
    ├── type: CheckpointType
    └── location: string

TaskIdentifier (任务标识注册表)
    ├── slug: "my-task"
    ├── currentWorkerId → BackgroundWorker?    ← 当前版本关联的 Worker
    └── isInLatestDeployment: Boolean          ← 是否在最新部署中
```

**源码位置**：
- `internal-packages/database/prisma/schema.prisma:2106` — WorkerDeployment
- `internal-packages/database/prisma/schema.prisma:533` — BackgroundWorker
- `internal-packages/database/prisma/schema.prisma:669` — BackgroundWorkerTask
- `internal-packages/database/prisma/schema.prisma:864` — TaskRun
- `internal-packages/database/prisma/schema.prisma:1617` — TaskRunAttempt
- `internal-packages/database/prisma/schema.prisma:3128` — TaskIdentifier

---

## 3. 构建产物的版本号关联机制

### 3.1 版本号生成算法

版本号格式为 `YYYYMMDD.N`，其中日期部分为构建当天的日期，序号部分为当天同环境下的递增编号。

```typescript
// apps/webapp/app/v3/utils/calculateNextBuildVersion.ts
export function calculateNextBuildVersion(latestVersion?: string | null): string {
  const today = new Date();
  const todayFormatted = `${year}${month < 10 ? "0" : ""}${month}${day < 10 ? "0" : ""}${day}`;

  if (!latestVersion) {
    return `${todayFormatted}.1`;
  }

  const [date, buildNumber] = latestVersion.split(".");
  if (date === todayFormatted) {
    return `${date}.${parseInt(buildNumber, 10) + 1}`;
  }
  return `${todayFormatted}.1`;
}
```

**关键特性**：
- 版本号按环境独立计算（dev/prod 各自拥有独立的版本序列）
- 唯一性约束：`@@unique([environmentId, version])` 确保同一环境内版本号不重复
- 并发安全：`createDeploymentWithNextVersion` 使用乐观重试（最多 5 次 + 随机抖动）解决版本号碰撞

**源码位置**：
- `apps/webapp/app/v3/utils/calculateNextBuildVersion.ts:1`
- `apps/webapp/app/v3/services/initializeDeployment/createDeploymentWithNextVersion.server.ts:45`

### 3.2 contentHash —— 构建产物指纹

contentHash 是整个构建输出的密码学指纹，用于去重判定和变更检测。

**CLI 端计算**（构建时）：

```typescript
// packages/cli-v3/src/build/bundle.ts:244-248
const hasher = createHash("md5");
for (const outputFile of result.outputFiles) {
  hasher.update(outputFile.hash);  // 将每个 esbuild 输出文件的 hash 汇总
  outputHashes[outputFile.path] = outputFile.hash;
}
// 最终的 contentHash = hasher.digest("hex")
```

**Webapp 端去重判定**：

```typescript
// apps/webapp/app/v3/services/createBackgroundWorker.server.ts:102-106
if (latestBackgroundWorker?.contentHash === body.metadata.contentHash) {
  return latestBackgroundWorker;  // 内容未变，跳过创建新版本
}
```

**依赖快照关联**：

`BackgroundWorkerMetadata` 的完整结构定义了每个构建的依赖快照：

```typescript
// packages/core/src/v3/schemas/resources.ts:56-66
export const BackgroundWorkerMetadata = z.object({
  packageVersion: z.string(),         // SDK 版本
  contentHash: z.string(),            // 构建产物指纹
  cliPackageVersion: z.string(),      // CLI 版本
  tasks: z.array(TaskResource),       // 所有任务定义
  prompts: z.array(PromptResource),   // Prompt 资源
  queues: z.array(QueueManifest),     // 队列声明
  sourceFiles: z.array(BackgroundWorkerSourceFileMetadata), // 源文件快照
  runtime: z.string(),                // 运行时类型 (node/bun)
  runtimeVersion: z.string(),         // 运行时版本
});
```

其中 `sourceFiles` 包含每个源文件的内容和哈希：

```typescript
export const BackgroundWorkerSourceFileMetadata = z.object({
  filePath: z.string(),
  contents: z.string(),       // 源文件完整内容
  contentHash: z.string(),    // 单文件 SHA-256 摘要
  taskIds: z.array(z.string()),
});
```

**源码位置**：
- `packages/cli-v3/src/build/bundle.ts:244`
- `packages/core/src/v3/schemas/resources.ts:56`
- `apps/webapp/app/v3/services/createBackgroundWorker.server.ts:102`

### 3.3 部署状态机

```
PENDING → INSTALLING → BUILDING → DEPLOYING → DEPLOYED
                                          ├── FAILED
                                          ├── TIMED_OUT
                                          └── CANCELED
```

每个状态转换都记录了时间戳（`startedAt`, `builtAt`, `deployedAt`, `failedAt` 等），提供完整的审计追踪。

**源码位置**：`internal-packages/database/prisma/schema.prisma:2167`

---

## 4. 运行环境与依赖快照关联

### 4.1 环境隔离

`BackgroundWorker` 和 `WorkerDeployment` 均通过 `runtimeEnvironmentId` 关联到具体的 `RuntimeEnvironment`。版本号在环境级别独立递增，且唯一约束为 `@@unique([projectId, runtimeEnvironmentId, version])`，确保不同环境之间互不干扰。

### 4.2 运行时与镜像引用

每个 `WorkerDeployment` 记录了：

| 字段 | 含义 | 示例 |
|------|------|------|
| `runtime` | 运行时类型 | `"node"`, `"bun"` |
| `runtimeVersion` | 运行时版本 | `"20.11.0"` |
| `imageReference` | 容器镜像引用 | `"registry.example.com/my-app:v1@sha256:abc..."` |
| `imagePlatform` | 目标平台 | `"linux/amd64"` |
| `commitSHA` | Git 提交哈希 | `"a1b2c3d4"` |
| `git` | Git 元数据 | `{ branch, ref, commitMessage }` |
| `contentHash` | 构建产物指纹 | MD5 of all esbuild output hashes |

### 4.3 源文件与依赖快照

`BackgroundWorkerFile` 表存储了构建中所有源文件的去重快照：

```prisma
model BackgroundWorkerFile {
  filePath    String
  contentHash String
  contents    Bytes           // 源文件完整内容
  projectId   String
  @@unique([projectId, contentHash])  // 按项目+内容去重
}
```

同一个 `contentHash` 的文件在项目级别只存储一份（通过 upsert 实现），不同版本的 Worker 通过关联表引用同一份文件，既节省存储又保证快照不可变性。

**源码位置**：
- `internal-packages/database/prisma/schema.prisma:581`
- `apps/webapp/app/v3/services/createBackgroundWorker.server.ts:785`

---

## 5. 旧版本任务的触发与执行

### 5.1 版本锁定机制

当一个 TaskRun 开始执行时，它会被锁定到当前最新版本的 BackgroundWorker：

```typescript
// apps/webapp/app/v3/marqs/devQueueConsumer.server.ts:446-465
const lockedTaskRun = await prisma.taskRun.update({
  where: { id: message.messageId },
  data: {
    lockedAt,
    lockedById: backgroundTask.id,           // 锁定到具体任务
    lockedToVersionId: backgroundWorker.id,  // 锁定到具体 Worker 版本
    taskVersion: backgroundWorker.version,   // 冗余记录版本号
    sdkVersion: backgroundWorker.sdkVersion,
    cliVersion: backgroundWorker.cliVersion,
    status: "EXECUTING",
  },
});
```

**锁定粒度**：
- `lockedToVersionId` → BackgroundWorker：版本级锁定
- `lockedById` → BackgroundWorkerTask：任务级锁定

一旦锁定，即使后续部署了新版本，正在运行的任务也不会受影响。

**源码位置**：`apps/webapp/app/v3/marqs/devQueueConsumer.server.ts:446`

### 5.2 旧版本任务的调度路径

#### 5.2.1 V1 引擎（共享队列消费者）

```typescript
// apps/webapp/app/v3/marqs/sharedQueueConsumer.server.ts:612-637
const deployment = existingTaskRun.lockedById
  ? await getWorkerDeploymentFromWorkerTask(existingTaskRun.lockedById)
  : existingTaskRun.lockedToVersionId
  ? await getWorkerDeploymentFromWorker(existingTaskRun.lockedToVersionId)
  : await findCurrentWorkerDeployment({
      environmentId: existingTaskRun.runtimeEnvironmentId,
      type: "V1",
    });
```

调度优先级：
1. 如果 TaskRun 已锁定到具体的 `BackgroundWorkerTask` → 从该 Task 的 Worker 找到 Deployment
2. 如果 TaskRun 只锁定了 Worker 版本 → 从该 Worker 找到 Deployment
3. 未锁定 → 使用当前被 promote 的 Deployment

#### 5.2.2 V2 引擎（Run Engine 的 DequeueSystem）

```typescript
// internal-packages/run-engine/src/engine/systems/dequeueSystem.ts:853-870
const workerId = run.lockedToVersionId ?? backgroundWorkerId;

if (run.runtimeEnvironment.type === "DEVELOPMENT") {
  workerWithTasks = workerId
    ? await this.#getWorkerById(prisma, workerId)
    : await this.#getMostRecentWorker(prisma, run.runtimeEnvironmentId);
} else {
  workerWithTasks = workerId
    ? await this.#getWorkerDeploymentFromWorker(prisma, workerId)
    : await this.#getManagedWorkerFromCurrentlyPromotedDeployment(
        prisma, run.runtimeEnvironmentId
      );
}
```

对于已部署环境（非 DEV），旧版本的任务会：
1. 通过 `lockedToVersionId` 找到原始的 BackgroundWorker
2. 从该 Worker 找到关联的 WorkerDeployment
3. 使用 Deployment 的 `imageReference` 拉取正确的容器镜像执行

**源码位置**：
- `apps/webapp/app/v3/marqs/sharedQueueConsumer.server.ts:612`
- `internal-packages/run-engine/src/engine/systems/dequeueSystem.ts:853`

### 5.3 显式版本锁定触发

SDK 支持在触发任务时指定版本号：

```typescript
// 通过 API 触发时指定版本
await myTask.trigger({ foo: "bar" }, { version: "20250228.1" });

// 或通过环境变量全局指定
TRIGGER_VERSION=20250228.1
```

在 `triggerTask.server.ts` 中的实现：

```typescript
// apps/webapp/app/runEngine/services/triggerTask.server.ts:259-270
const lockedToBackgroundWorker = body.options?.lockToVersion
  ? await this.prisma.backgroundWorker.findFirst({
      where: {
        projectId: environment.projectId,
        runtimeEnvironmentId: environment.id,
        version: body.options?.lockToVersion,
      },
      select: {
        id: true,
        version: true,
        sdkVersion: true,
        cliVersion: true,
      },
    })
  : undefined;
```

### 5.4 子任务版本继承

`triggerAndWait()` 和 `batchTriggerAndWait()` 会将子任务锁定到与父任务相同的版本：

| 触发方式 | 父任务版本 | 子任务版本 | 是否锁定 |
|----------|-----------|-----------|---------|
| `trigger()` | 20240313.2 | 当前最新 | 否 |
| `batchTrigger()` | 20240313.2 | 当前最新 | 否 |
| `triggerAndWait()` | 20240313.2 | 20240313.2 | 是 |
| `batchTriggerAndWait()` | 20240313.2 | 20240313.2 | 是 |

### 5.5 PENDING_VERSION 状态

V2 引擎引入了 `PENDING_VERSION` 状态，当任务被触发但对应 taskIdentifier 尚无可用 Worker 时：

```typescript
// internal-packages/run-engine/src/engine/systems/pendingVersionSystem.ts:53-69
const pendingRuns = await this.$.readOnlyPrisma.taskRun.findMany({
  where: {
    runtimeEnvironmentId: backgroundWorker.runtimeEnvironmentId,
    projectId: backgroundWorker.projectId,
    status: "PENDING_VERSION",
    taskIdentifier: {
      in: backgroundWorker.tasks.map((task) => task.slug),
    },
  },
});
```

当新的 BackgroundWorker 注册时，系统会自动将匹配的 PENDING_VERSION 运行转为 PENDING 并入队执行。

**源码位置**：`internal-packages/database/prisma/schema.prisma:1131`

### 5.6 WAITING_FOR_DEPLOY 状态

V1 引擎的类似机制，当任务触发时没有匹配的 Deployment：

```typescript
// apps/webapp/app/v3/services/executeTasksWaitingForDeploy.ts:31-38
const runsWaitingForDeploy = await this._replica.taskRun.findMany({
  where: {
    runtimeEnvironmentId: backgroundWorker.runtimeEnvironmentId,
    status: "WAITING_FOR_DEPLOY",
    taskIdentifier: {
      in: backgroundWorker.tasks.map((task) => task.slug),
    },
  },
});
```

**源码位置**：`apps/webapp/app/v3/services/executeTasksWaitingForDeploy.ts:8`

### 5.7 重试与重放

- **重试**：锁定到原始版本（`lockedToVersionId` 不变），确保重试行为一致
- **重放（Replay）**：创建一个新运行，锁定到**当前最新版本**，而非原始版本

---

## 6. 控制面与执行端之间的元数据共享

### 6.1 架构概览

```
┌─────────────────────┐         ┌─────────────────────┐         ┌─────────────────────┐
│    Webapp (控制面)    │◄──────►│   Supervisor (调度)   │◄──────►│   Runner (执行端)     │
│                     │  HTTP/  │                     │  HTTP/  │                     │
│  - API Server       │  Socket │  - WorkloadManager  │  Docker │  - 任务执行进程       │
│  - RunEngine        │   .io   │  - SupervisorSession│   K8s   │  - SDK Runtime      │
│  - MarQS Consumer   │         │  - WorkloadServer   │         │  - WorkloadClient    │
│  - TaskMetadataCache│         │  - ResourceMonitor  │         │                     │
└─────────────────────┘         └─────────────────────┘         └─────────────────────┘
```

### 6.2 控制面 → Supervisor：任务分配

控制面通过两种方式将任务分配给 Supervisor：

#### 方式一：Socket.io 事件推送（V1）

Webapp 通过 Socket.io 的 `coordinatorNamespace` 推送任务信息。

#### 方式二：Supervisor 主动 Dequeue（V2/当前主流）

Supervisor 通过 HTTP API 主动从 Webapp 拉取任务：

```
Supervisor → POST /api/v2/runs/dequeue → Webapp
Webapp 返回 DequeuedMessage，包含：
  - runId, snapshotId, imageReference
  - deploymentVersion, deploymentFriendlyId
  - machine preset, traceContext
```

### 6.3 Supervisor → Runner：容器创建与元数据传递

Supervisor 在创建 Runner 容器时，将关键元数据以环境变量形式注入：

```typescript
// apps/supervisor/src/workloadManager/docker.ts:66-96
const envVars: string[] = [
  `OTEL_EXPORTER_OTLP_ENDPOINT=${env.OTEL_EXPORTER_OTLP_ENDPOINT}`,
  `TRIGGER_DEQUEUED_AT_MS=${opts.dequeuedAt.getTime()}`,
  `TRIGGER_POD_SCHEDULED_AT_MS=${Date.now()}`,
  `TRIGGER_ENV_ID=${opts.envId}`,
  `TRIGGER_DEPLOYMENT_ID=${opts.deploymentFriendlyId}`,
  `TRIGGER_DEPLOYMENT_VERSION=${opts.deploymentVersion}`,
  `TRIGGER_RUN_ID=${opts.runFriendlyId}`,
  `TRIGGER_SNAPSHOT_ID=${opts.snapshotFriendlyId}`,
  `TRIGGER_SUPERVISOR_API_PROTOCOL=${this.opts.workloadApiProtocol}`,
  `TRIGGER_SUPERVISOR_API_PORT=${this.opts.workloadApiPort}`,
  `TRIGGER_SUPERVISOR_API_DOMAIN=${this.opts.workloadApiDomain}`,
  `TRIGGER_WORKER_INSTANCE_NAME=${env.TRIGGER_WORKER_INSTANCE_NAME}`,
  `TRIGGER_RUNNER_ID=${runnerId}`,
  `TRIGGER_MACHINE_CPU=${opts.machine.cpu}`,
  `TRIGGER_MACHINE_MEMORY=${opts.machine.memory}`,
  `PRETTY_LOGS=${env.RUNNER_PRETTY_LOGS}`,
];
```

Kubernetes 模式下类似，还增加了 `status.hostIP` 等字段引用。

**WorkloadManagerCreateOptions 完整定义**：

```typescript
// apps/supervisor/src/workloadManager/types.ts:20-47
export interface WorkloadManagerCreateOptions {
  image: string;                    // 容器镜像 (来自 WorkerDeployment.imageReference)
  machine: MachinePreset;           // 机器配置
  version: string;                  // 部署版本号
  nextAttemptNumber?: number;       // 重试次数
  dequeuedAt: Date;                 // 出队时间
  envId: string;                    // 环境ID
  envType: EnvironmentType;         // 环境类型
  orgId: string;                    // 组织ID
  projectId: string;                // 项目ID
  deploymentFriendlyId: string;     // 部署友好ID
  deploymentVersion: string;        // 部署版本
  runId: string;                    // 运行ID
  runFriendlyId: string;            // 运行友好ID
  snapshotId: string;               // 快照ID
  snapshotFriendlyId: string;       // 快照友好ID
  traceContext?: Record<string, unknown>; // OTel 追踪上下文
  annotations?: RunAnnotations;     // 运行注解
}
```

**源码位置**：
- `apps/supervisor/src/workloadManager/docker.ts:64`
- `apps/supervisor/src/workloadManager/kubernetes.ts:102`
- `apps/supervisor/src/workloadManager/types.ts:20`

### 6.4 Runner → 控制面：状态回报与心跳

Runner 通过 Workload API (HTTP) 与 Supervisor 通信：

- **心跳**：Runner 定期向 Supervisor 发送心跳，报告存活状态
- **快照轮询**：Runner 轮询 Supervisor 获取执行快照更新
- **I/O 操作**：Logger、Metadata 更新等通过 Supervisor 转发到 Webapp

### 6.5 控制面内部：TaskMetadataCache

Webapp 使用 Redis 缓存加速任务元数据查询：

```
task-meta:by-env:{environmentId}:{taskSlug} → {
  workerId, queueName, queueId, triggerSource, ttl
}
task-meta:by-worker:{workerId}:{taskSlug} → {
  workerId, queueName, queueId, triggerSource, ttl
}
```

- `by-env` keyspace：指向当前 promote 的 Deployment 中的任务
- `by-worker` keyspace：指向特定 Worker 版本中的任务（用于版本锁定查询）

新 Worker 注册时：

```typescript
// apps/webapp/app/v3/services/createDeploymentBackgroundWorkerV4.server.ts:141-143
if (workerTaskEntries) {
  await this._taskMetaCache.populateByWorker(backgroundWorker.id, workerTaskEntries);
}
```

Deployment 被 promote 时，`by-env` keyspace 更新指向新 Deployment。

**源码位置**：`apps/webapp/app/v3/services/createDeploymentBackgroundWorkerV4.server.ts:141`

### 6.6 环境变量在版本间的隔离

每个 BackgroundWorker 的元数据中记录了 `sdkVersion` 和 `cliVersion`，TaskRun 也在执行时冗余记录这两个值：

```typescript
// apps/webapp/app/v3/marqs/devQueueConsumer.server.ts:458-460
taskVersion: backgroundWorker.version,
sdkVersion: backgroundWorker.sdkVersion,
cliVersion: backgroundWorker.cliVersion,
```

用户定义的环境变量通过 `populateEnv` 工具在 Runner 进程中设置：

```typescript
// packages/core/src/v3/workers/populateEnv.ts:30-64
export function populateEnv(
  envObject: Record<string, string>,
  options: PopulateEnvOptions = {}
): void {
  for (const key of Object.keys(envObject)) {
    if (Object.prototype.hasOwnProperty.call(process.env, key)) {
      if (override) process.env[key] = envObject[key];
    } else {
      process.env[key] = envObject[key];
    }
  }
  // 处理被移除的变量
  if (previousEnv) {
    for (const key of Object.keys(previousEnv)) {
      if (!Object.prototype.hasOwnProperty.call(envObject, key)) {
        delete process.env[key];
      }
    }
  }
}
```

在本地开发模式下，每个 TaskRun 在独立进程中执行（`populateEnv` 在进程启动时调用），确保版本间环境变量完全隔离。

---

## 7. 部署升级与回滚流程

### 7.1 部署提升（Promotion）

```typescript
// apps/webapp/app/v3/services/changeCurrentDeployment.server.ts:30-100
export class ChangeCurrentDeploymentService extends BaseService {
  public async call(
    deployment: WorkerDeployment,
    direction: "promote" | "rollback",
    disableVersionCheck?: boolean
  ) {
    // 验证方向性：promote 只能向前，rollback 只能向后
    if (!disableVersionCheck) {
      switch (direction) {
        case "promote":
          if (compareDeploymentVersions(current.version, deployment.version) >= 0)
            throw new ServiceValidationError("Cannot promote a deployment older than current");
          break;
        case "rollback":
          if (compareDeploymentVersions(current.version, deployment.version) <= 0)
            throw new ServiceValidationError("Cannot rollback to a deployment newer than current");
          break;
      }
    }

    // 原子更新 Promotion 记录
    await this._prisma.workerDeploymentPromotion.upsert({
      where: { environmentId_label: { environmentId, label: CURRENT_DEPLOYMENT_LABEL } },
      create: { deploymentId: deployment.id, environmentId, label: CURRENT_DEPLOYMENT_LABEL },
      update: { deploymentId: deployment.id },
    });

    // 同步 TaskIdentifier 注册表
    await syncTaskIdentifiers(/* ... */);

    // 更新 Redis 缓存
    await this._taskMetaCache.populateByEnv(deployment.environmentId, tasks);

    // 处理等待部署的任务
    await new ExecuteTasksWaitingForDeployService().call(deployment.workerId!);
  }
}
```

**源码位置**：`apps/webapp/app/v3/services/changeCurrentDeployment.server.ts:30`

### 7.2 部署完成（Finalize）

```typescript
// apps/webapp/app/v3/services/finalizeDeployment.server.ts:76-87
const finalizedDeployment = await this._prisma.workerDeployment.update({
  where: { id: deployment.id },
  data: {
    status: "DEPLOYED",
    deployedAt: new Date(),
    imageReference: imageDigest
      ? `${deployment.imageReference}@${imageDigest}`  // 追加镜像 digest
      : undefined,
  },
});

// 自动 promote
if (!body.skipPromotion) {
  await new ChangeCurrentDeploymentService().call(finalizedDeployment, "promote");
}
```

### 7.3 回滚操作

回滚是将一个较旧版本的 Deployment 重新设为 "current"：

```typescript
// 前端判断逻辑 (apps/webapp/app/routes/...deployments/route.tsx:427-437)
const canBeRolledBack =
  canBeMadeCurrent &&
  currentDeployment?.version &&
  compareDeploymentVersions(deployment.version, currentDeployment.version) === -1;
```

回滚后：
- 新触发的不带版本锁定的任务将使用回滚版本执行
- 已经锁定到更新版本的任务仍用原版本执行
- 镜像引用不变，旧版本的 Docker image 仍可拉取

---

## 8. 可追溯性与审计

### 8.1 时间线追踪

每个 WorkerDeployment 记录了完整的生命周期时间戳：

| 字段 | 含义 |
|------|------|
| `createdAt` | 创建时间 |
| `startedAt` | 开始构建时间 |
| `builtAt` | 构建完成时间 |
| `deployedAt` | 部署完成时间 |
| `failedAt` | 失败时间 |
| `canceledAt` | 取消时间 |

### 8.2 Git 关联

WorkerDeployment 和 RuntimeEnvironment 均可关联 Git 元数据：

```typescript
git: {
  branch: string;
  ref: string;
  commitMessage: string;
  commitSha: string;
  dirty: boolean;
}  // GitMeta type
```

此外还有专用的 `commitSHA` 字段用于直接查询（`@@index([commitSHA])`）。

### 8.3 构建元数据

`buildServerMetadata` 和 `externalBuildData` 存储了外部构建系统的信息（如 Depot 构建ID、原生构建标志等），用于追溯构建过程。

### 8.4 TaskIdentifier 注册表

`TaskIdentifier` 表追踪每个任务标识符的出现历史：

```prisma
model TaskIdentifier {
  slug                   String
  currentWorkerId        String?        ← 当前关联的 Worker
  isInLatestDeployment   Boolean        ← 是否在最新部署中
  firstSeenAt            DateTime       ← 首次出现时间
  lastSeenAt             DateTime       ← 最近出现时间
}
```

这使得系统能够知道某个任务是否已从最新版本中移除，以及它首次出现在哪个版本。

### 8.5 Checkpoint 机制

对于需要暂停/恢复的长运行任务，Checkpoint 记录了容器镜像引用：

```prisma
model Checkpoint {
  type     CheckpointType
  location String
  imageRef String    ← 用于恢复时拉取正确的镜像
  reason   String?
  metadata String?
}
```

---

## 9. 完整流程总结

### 9.1 构建与部署流程

```
1. CLI 执行 esbuild 构建
   └── 计算所有输出文件的 MD5 → contentHash
   └── 收集 TaskResource、SourceFile、Queue 等元数据
   └── 构建 Docker image (imageReference)

2. CLI 调用 Webapp API 初始化 Deployment
   └── createDeploymentWithNextVersion()
   └── 版本号 = calculateNextBuildVersion()
   └── status = PENDING → INSTALLING → BUILDING

3. 构建完成后，CLI 调用 finalizeDeployment API
   └── 记录 imageDigest
   └── status = DEPLOYING → DEPLOYED
   └── 自动调用 ChangeCurrentDeploymentService (promote)

4. 创建 BackgroundWorker 记录
   └── 与 WorkerDeployment 1:1 关联
   └── 创建 BackgroundWorkerTask (每个任务)
   └── 创建/关联 BackgroundWorkerFile (源文件去重)
   └── 同步 TaskIdentifier 注册表
   └── 更新 TaskMetadataCache (Redis)

5. Promotion 生效
   └── WorkerDeploymentPromotion 记录更新
   └── 新任务开始使用此版本
   └── 处理 PENDING_VERSION / WAITING_FOR_DEPLOY 的任务
```

### 9.2 任务执行流程（含版本锁定）

```
1. 任务触发
   ├── 无版本锁定 → 使用当前 promote 的 Deployment
   ├── lockToVersion → 查找指定版本的 BackgroundWorker
   └── triggerAndWait → 继承父任务的版本锁定

2. Dequeue (Run Engine V2)
   ├── 检查 lockedToVersionId
   │   ├── 有 → 从该 Worker 找 Deployment → 获取 imageReference
   │   └── 无 → 从当前 promote 的 Deployment → 获取 imageReference
   └── 创建 TaskRunAttempt，关联 BackgroundWorker + BackgroundWorkerTask

3. Supervisor 创建 Runner 容器
   └── 使用 imageReference 拉取正确版本的镜像
   └── 注入环境变量 (DEPLOYMENT_VERSION, RUN_ID, SNAPSHOT_ID 等)
   └── Runner 进程执行任务代码

4. 执行完成
   └── Runner 通过 Workload API 报告结果
   └── Supervisor 转发到 Webapp
   └── Webapp 更新 TaskRun 状态
```

### 9.3 回滚流程

```
1. 用户选择较旧版本的 Deployment
2. 调用 ChangeCurrentDeploymentService (direction="rollback")
3. 版本方向性验证（只允许回滚到更旧版本）
4. 更新 WorkerDeploymentPromotion → 旧版本变为 "current"
5. 更新 TaskMetadataCache (by-env keyspace)
6. 新触发的任务使用回滚版本执行
7. 已锁定到新版本的任务不受影响
```

---

## 10. 关键保障机制汇总

| 保障项 | 实现机制 | 源码位置 |
|--------|---------|---------|
| 版本号唯一性 | `@@unique([environmentId, version])` + 乐观重试 | schema.prisma:2162 |
| 构建去重 | contentHash 比对 | createBackgroundWorker.server.ts:102 |
| 版本锁定 | TaskRun.lockedToVersionId + lockedById | devQueueConsumer.server.ts:446 |
| 环境隔离 | 按 runtimeEnvironmentId 独立管理 | schema.prisma:575 |
| 镜像不可变 | imageReference 含 SHA256 digest | finalizeDeployment.server.ts:82 |
| 回滚安全 | 方向性验证 + Promotion 原子更新 | changeCurrentDeployment.server.ts:63 |
| 任务发现 | TaskIdentifier 注册表 + isInLatestDeployment | schema.prisma:3128 |
| 未部署等待 | PENDING_VERSION / WAITING_FOR_DEPLOY | pendingVersionSystem.ts:53 |
| 子任务一致性 | triggerAndWait 版本继承 | docs/versioning.mdx |
| 审计追踪 | 完整时间戳 + Git 元数据 + buildServerMetadata | schema.prisma:2138-2143 |
| 文件去重 | BackgroundWorkerFile contentHash upsert | createBackgroundWorker.server.ts:785 |
