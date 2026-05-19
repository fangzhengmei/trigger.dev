# 定时调度环境隔离机制分析

## 概述

本文分析Trigger.dev平台中定时调度服务、环境路由和运行器选择三者如何协作，确保调度信号在staging和production环境之间正确隔离。

---

## 1. 核心数据模型

### 1.1 环境模型 (RuntimeEnvironment)

**文件**: `internal-packages/database/prisma/schema.prisma:314-400`

```prisma
model RuntimeEnvironment {
  id          String                    @id @default(cuid())
  slug        String
  apiKey      String                    @unique
  type        RuntimeEnvironmentType    @default(DEVELOPMENT)
  branchName  String?
  parentEnvironmentId String?
  organizationId String
  projectId    String
  archivedAt   DateTime?
  // ...
}

enum RuntimeEnvironmentType {
  PRODUCTION
  STAGING
  DEVELOPMENT
  PREVIEW
}
```

**关键隔离属性**:
- `type`: 环境类型枚举，明确区分PRODUCTION/STAGING/DEVELOPMENT/PREVIEW
- `apiKey`: 每个环境独立的API密钥，作为认证边界
- `organizationId` + `projectId`: 环境属于特定组织和项目
- `archivedAt`: 软删除标记，确保已归档环境不接收调度

### 1.2 调度实例模型 (TaskScheduleInstance)

**文件**: `internal-packages/database/prisma/schema.prisma:2256-2282`

```prisma
model TaskScheduleInstance {
  id             String             @id @default(cuid())
  taskSchedule   TaskSchedule       @relation(fields: [taskScheduleId], references: [id])
  taskScheduleId String
  environment    RuntimeEnvironment @relation(fields: [environmentId], references: [id])
  environmentId  String
  projectId      String
  active         Boolean            @default(true)
  // ...

  @@unique([taskScheduleId, environmentId])
}
```

**隔离机制**:
- 每个调度（`TaskSchedule`）在每个环境中都有独立的实例（`TaskScheduleInstance`）
- `@@unique([taskScheduleId, environmentId])` 确保一个调度在同一环境中只能有一个实例
- `active` 字段支持按环境单独启用/禁用调度

---

## 2. 调度服务 (Schedule Engine)

### 2.1 核心架构

**文件**: `internal-packages/schedule-engine/src/engine/index.ts`

```typescript
export class ScheduleEngine {
  // Redis worker用于调度触发
  private worker: Worker<typeof scheduleWorkerCatalog>;
  
  // 数据库访问
  prisma: PrismaClient;
  
  // 调度触发回调
  private onTriggerScheduledTask: TriggerScheduledTaskCallback;
  
  // 开发环境连接检查
  isDevEnvironmentConnectedHandler: (environmentId: string) => Promise<boolean>;
}
```

### 2.2 调度注册流程

**方法**: `registerNextTaskScheduleInstance` (line 125-269)

```
调度实例加载 → 计算下次执行时间 → 环境有效性检查 → 入队Redis
```

**环境隔离检查点**:
1. 加载实例时包含 `environment` 关联 (line 155)
2. 从实例获取 `environment.type` 用于指标标记 (line 170)
3. 根据环境类型应用不同的调度策略

### 2.3 调度触发流程

**方法**: `triggerScheduledTask` (line 285-636)

触发前的多层环境校验:

```typescript
// 1. 组织有效性检查
if (instance.environment.organization.deletedAt) {
  skipReason = "organization_deleted";
}

// 2. 项目有效性检查  
if (instance.environment.project.deletedAt) {
  skipReason = "project_deleted";
}

// 3. 环境归档检查
if (instance.environment.archivedAt) {
  skipReason = "environment_archived";
}

// 4. 调度实例和调度本身的active状态
if (!instance.active || !instance.taskSchedule.active) {
  skipReason = "schedule_inactive";
}

// 5. 开发环境特殊处理 - 检查是否有活跃的dev会话
if (instance.environment.type === "DEVELOPMENT") {
  const isConnected = await this.options.isDevEnvironmentConnectedHandler(
    instance.environment.id
  );
  if (!isConnected) {
    skipReason = "dev_disconnected";
  }
}
```

**文件**: `apps/webapp/app/v3/scheduleEngine.server.ts:18-50`

开发环境连接检查实现:
```typescript
async function isDevEnvironmentConnectedHandler(environmentId: string) {
  const environment = await prisma.runtimeEnvironment.findFirst({
    where: { id: environmentId },
    select: {
      currentSession: { select: { disconnectedAt: true } },
      project: { select: { engine: true } },
    },
  });
  
  // V3引擎检查devPresence
  if (environment.project.engine === "V3") {
    return devPresence.isConnected(environmentId);
  }
  
  // V1引擎检查currentSession
  return !environment.currentSession || !environment.currentSession.disconnectedAt;
}
```

### 2.4 调度触发回调

**文件**: `apps/webapp/app/v3/scheduleEngine.server.ts:78-134`

```typescript
onTriggerScheduledTask: async ({
  taskIdentifier,
  environment,  // 完整的环境信息，包含org/project/type
  payload,
  scheduleInstanceId,
  scheduleId,
  exactScheduleTime,
}) => {
  const triggerService = new TriggerTaskService();
  
  // 将环境信息传递给触发服务
  return triggerService.call(
    taskIdentifier,
    environment,  // 环境信息贯穿整个调用链
    { payload: payloadPacket.data, options: { payloadType: payloadPacket.dataType } },
    { /* 调度元数据 */ }
  );
}
```

---

## 3. 环境路由 (Environment Routing)

### 3.1 认证环境对象

**文件**: `packages/core/src/v3/auth/environment.ts`

```typescript
export type AuthenticatedEnvironment = {
  id: string;
  slug: string;
  type: RuntimeEnvironmentType;  // "PRODUCTION" | "STAGING" | "DEVELOPMENT" | "PREVIEW"
  apiKey: string;
  organizationId: string;
  projectId: string;
  orgMemberId: string | null;
  parentEnvironmentId: string | null;
  branchName: string | null;
  archivedAt: Date | null;
  paused: boolean;
  // ...
  
  project: {
    id: string;
    engine: RunEngineVersion;  // "V1" | "V2"
    // ...
  };
  
  organization: {
    id: string;
    // ... 组织级配置和限制
  };
};
```

**设计意图**:
- 结构化的环境对象在认证边界创建后贯穿整个调用链
- 包含环境类型、组织、项目等所有路由决策所需信息
- 避免后续服务重复查询数据库

### 3.2 触发服务的环境路由

**文件**: `apps/webapp/app/v3/services/triggerTask.server.ts:53-121`

```typescript
export class TriggerTaskService extends WithRunEngine {
  public async call(
    taskId: string,
    environment: AuthenticatedEnvironment,  // 认证环境对象
    body: TriggerTaskRequestBody,
    options: TriggerTaskServiceOptions = {},
    version?: RunEngineVersion
  ): Promise<TriggerTaskServiceResult | undefined> {
    // 根据项目引擎版本选择执行路径
    const v = await determineEngineVersion({
      environment,
      workerVersion: body.options?.lockToVersion,
      engineVersion: version,
    });

    switch (v) {
      case "V1":
        return this.callV1(taskId, environment, body, options);
      case "V2":
        return this.callV2(taskId, environment, body, options);
    }
  }
}
```

### 3.3 V2引擎触发流程

**文件**: `apps/webapp/app/runEngine/services/triggerTask.server.ts:87-450`

```typescript
// 1. 环境权限验证
if (!options.skipChecks) {
  const entitlementValidation = await this.validator.validateEntitlement({
    environment,
  });
  if (!entitlementValidation.ok) {
    throw entitlementValidation.error;
  }
}

// 2. 队列属性解析（包含环境信息）
const { queueName, lockedQueueId, taskTtl, taskKind } =
  await this.queueConcern.resolveQueueProperties(
    triggerRequest,  // 包含environment
    lockedToBackgroundWorker
  );

// 3. 队列大小限制检查（按环境隔离）
if (!options.skipChecks) {
  const queueSizeGuard = await this.queueConcern.validateQueueLimits(
    environment,
    queueName
  );
  if (!queueSizeGuard.ok) {
    throw new QueueSizeLimitExceededError(...);
  }
}

// 4. Worker队列选择（关键：运行器路由）
const workerQueueResult = await this.queueConcern.getWorkerQueue(
  environment,
  body.options?.region
);
const workerQueue = workerQueueResult?.masterQueue;

// 5. 最终入队运行引擎
const taskRun = await this.engine.trigger({
  friendlyId: runFriendlyId,
  environment: environment,  // 环境信息传递给运行引擎
  workerQueue,               // 选定的worker队列
  // ... 其他参数
}, this.prisma);
```

---

## 4. 运行器选择 (Runner Selection)

### 4.1 Worker队列解析器

**文件**: `internal-packages/run-engine/src/run-queue/workerQueueResolver.ts`

```typescript
export class WorkerQueueResolver {
  private overrides: WorkerQueueOverrides | null;
  
  // 支持通过环境变量配置队列覆盖
  // RUN_ENGINE_WORKER_QUEUE_OVERRIDES = {
  //   "environmentId": { "env_xxx": "worker-queue-1" },
  //   "projectId": { "proj_xxx": "worker-queue-2" },
  //   "orgId": { "org_xxx": "worker-queue-3" },
  //   "workerQueue": { "default": "override" }
  // }

  public getWorkerQueueFromMessage(message: OutputPayload): string {
    if (message.version === "2") {
      // 检查覆盖配置，优先级: environmentId > projectId > orgId > workerQueue
      const override = this.#getOverride(message);
      if (override) return override;
      return message.workerQueue;
    }
    
    // 开发环境: 使用environmentId作为队列名，确保dev环境隔离
    if (message.environmentType === "DEVELOPMENT") {
      return message.environmentId;
    }
    
    return message.masterQueues[0];
  }
  
  #getOverride(message: OutputPayloadV2): string | null {
    if (!this.overrides) return null;
    
    // 优先级顺序确保更具体的配置优先
    if (this.overrides.environmentId?.[message.environmentId]) {
      return this.overrides.environmentId[message.environmentId];
    }
    if (this.overrides.projectId?.[message.projectId]) {
      return this.overrides.projectId[message.projectId];
    }
    if (this.overrides.orgId?.[message.orgId]) {
      return this.overrides.orgId[message.orgId];
    }
    if (this.overrides.workerQueue?.[message.workerQueue]) {
      return this.overrides.workerQueue[message.workerQueue];
    }
    return null;
  }
}
```

### 4.2 默认Worker队列选择逻辑

**文件**: `apps/webapp/app/runEngine/concerns/queues.server.ts:376-410`

```typescript
async getWorkerQueue(
  environment: AuthenticatedEnvironment,
  regionOverride?: string
): Promise<{ masterQueue: string; enableFastPath: boolean } | undefined> {
  // 开发环境: 直接使用environmentId作为队列名
  // 这确保每个开发环境有独立的队列，完全隔离
  if (environment.type === "DEVELOPMENT") {
    return { masterQueue: environment.id, enableFastPath: true };
  }
  
  // 非开发环境: 通过WorkerGroupService获取项目的默认worker组
  const workerGroupService = new WorkerGroupService({
    prisma: this.prisma,
    engine: this.engine,
  });
  
  const workerGroup = await workerGroupService.getDefaultWorkerGroupForProject({
    projectId: environment.projectId,
    regionOverride,
  });
  
  // 返回worker组对应的队列
  return {
    masterQueue: workerGroup.queueName,
    enableFastPath: workerGroup.supportsFastPath,
  };
}
```

### 4.3 运行队列 (RunQueue)

**文件**: `internal-packages/run-engine/src/run-queue/index.ts`

```typescript
export class RunQueue {
  private workerQueueResolver: WorkerQueueResolver;
  public keys: RunQueueKeyProducer;
  
  // 入队消息
  async enqueueMessage(
    env: MinimalAuthenticatedEnvironment,
    message: Omit<InputPayload, "orgId" | "projectId" | "environmentId" | "environmentType">
  ) {
    const inputPayload: InputPayload = {
      ...message,
      orgId: env.organization.id,
      projectId: env.project.id,
      environmentId: env.id,
      environmentType: env.type,
    };
    
    // 根据环境生成唯一的队列key
    const queueKey = this.keys.queueKey(env, message.queue, message.concurrencyKey);
    
    // 消息存储在环境专属的队列中
    await this.redis.zadd(queueKey, message.timestamp, messageId);
  }
  
  // 出队时根据workerQueue分发
  async dequeueMessageFromWorkerQueue(
    workerQueue: string,
    consumerId: string,
    visibilityTimeoutMs: number
  ) {
    // 从指定的worker队列获取消息
    // 不同环境的消息进入不同的worker队列
  }
}
```

---

## 5. 端到端协作流程

### 5.1 调度触发到运行的完整链路

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Schedule Engine (调度服务)                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 1. 加载TaskScheduleInstance                                  │  │
│  │    - 关联environment信息                                      │  │
│  │    - 检查organization/project/environment是否有效            │  │
│  └───────────────────────────────┬───────────────────────────────┘  │
│                                  │                                  │
│  ┌───────────────────────────────▼───────────────────────────────┐  │
│  │ 2. 开发环境特殊检查                                          │  │
│  │    - DEVELOPMENT: 检查devPresence连接状态                    │  │
│  │    - STAGING/PRODUCTION: 无额外检查                          │  │
│  └───────────────────────────────┬───────────────────────────────┘  │
│                                  │                                  │
│  ┌───────────────────────────────▼───────────────────────────────┐  │
│  │ 3. 调用onTriggerScheduledTask回调                            │  │
│  │    - 传递完整environment对象                                  │  │
│  │    - 传递调度元数据(scheduleId, exactScheduleTime等)         │  │
│  └───────────────────────────────┬───────────────────────────────┘  │
└──────────────────────────────────┼──────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────┐
│                    TriggerTaskService (触发服务)                    │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 4. 引擎版本路由                                              │  │
│  │    - 根据environment.project.engine选择V1/V2路径            │  │
│  └───────────────────────────────┬───────────────────────────────┘  │
│                                  │                                  │
│  ┌───────────────────────────────▼───────────────────────────────┐  │
│  │ 5. V2引擎 - RunEngineTriggerTaskService                      │  │
│  │    - 验证环境权限(entitlement)                                │  │
│  │    - 解析队列属性                                             │  │
│  │    - 检查环境队列大小限制                                     │  │
│  └───────────────────────────────┬───────────────────────────────┘  │
└──────────────────────────────────┼──────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────┐
│                  DefaultQueueManager (队列管理器)                   │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 6. Worker队列选择                                            │  │
│  │    - DEVELOPMENT: masterQueue = environment.id              │  │
│  │    - STAGING/PRODUCTION: 查询项目默认worker组               │  │
│  │    - 支持RUN_ENGINE_WORKER_QUEUE_OVERRIDES覆盖               │  │
│  └───────────────────────────────┬───────────────────────────────┘  │
└──────────────────────────────────┼──────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────┐
│                       RunEngine (运行引擎)                          │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 7. 创建TaskRun并写入数据库                                   │  │
│  │    - runtimeEnvironmentId = environment.id                   │  │
│  │    - workerQueue = 选定的队列名                               │  │
│  └───────────────────────────────┬───────────────────────────────┘  │
│                                  │                                  │
│  ┌───────────────────────────────▼───────────────────────────────┐  │
│  │ 8. 入队RunQueue                                              │  │
│  │    - queueKey包含orgId/projectId/envId                       │  │
│  │    - 消息payload包含environmentId和environmentType           │  │
│  │    - 不同环境的消息物理隔离在不同的Redis队列中                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Staging与Production隔离关键点

| 层面 | 隔离机制 | 代码位置 |
|------|---------|----------|
| **数据层** | 独立的RuntimeEnvironment记录，独立的apiKey | schema.prisma:314 |
| **调度层** | 每个环境独立的TaskScheduleInstance | schema.prisma:2256 |
| **调度触发** | 触发时携带完整environment对象，校验有效性 | schedule-engine/index.ts:306-415 |
| **队列路由** | DEVELOPMENT使用environment.id作为队列名 | queues.server.ts:380-382 |
| **队列覆盖** | 支持按environmentId精确指定worker队列 | workerQueueResolver.ts:82-84 |
| **运行时** | 消息payload包含environmentId和environmentType | run-queue/index.ts:InputPayload |
| **权限控制** | 按环境验证entitlement和队列大小限制 | triggerTask.server.ts:135-142 |

---

## 6. 关键设计决策分析

### 6.1 为什么开发环境使用environment.id作为队列名？

```typescript
// queues.server.ts:380-382
if (environment.type === "DEVELOPMENT") {
  return { masterQueue: environment.id, enableFastPath: true };
}
```

**设计意图**:
1. **强隔离**: 每个开发环境有独立的worker队列，避免不同开发者的任务相互干扰
2. **本地开发**: dev环境通常运行在开发者本地，需要精确路由到特定的CLI实例
3. **快速路径**: enableFastPath=true支持更直接的消息分发，提升开发体验

### 6.2 为什么需要TaskScheduleInstance中间表？

```prisma
model TaskScheduleInstance {
  @@unique([taskScheduleId, environmentId])
}
```

**设计意图**:
1. **环境级控制**: 允许同一调度在不同环境有独立的active状态
2. **多环境部署**: 一次部署可以创建/更新多个环境的调度实例
3. **状态追踪**: 每个环境的调度执行历史独立记录

### 6.3 WorkerQueueResolver的覆盖优先级设计

```
environmentId > projectId > orgId > workerQueue
```

**设计意图**:
1. **精确优先**: 更具体的配置（environmentId）优先于更宽泛的配置（orgId）
2. **灵活路由**: 支持运维场景下临时将特定环境/项目的流量路由到指定的worker池
3. **灰度发布**: 可以按环境维度灰度新的worker版本

---

## 7. 潜在问题与优化建议

### 7.1 潜在问题

1. **环境类型硬编码**: 目前在多处检查 `environment.type === "DEVELOPMENT"`，如果未来新增环境类型需要修改多处代码

2. **队列覆盖配置复杂**: `RUN_ENGINE_WORKER_QUEUE_OVERRIDES` 是JSON格式，配置错误可能导致意外路由

3. **开发环境连接检查**: 调度触发时才检查dev连接状态，可能导致调度延迟或跳过

### 7.2 优化建议

1. **环境策略模式**: 将不同环境类型的行为封装为策略对象，避免多处if-else

2. **队列覆盖配置验证**: 启动时验证覆盖配置的合法性，避免运行时错误

3. **预检查优化**: 在调度注册阶段预检查dev环境状态，提前发现问题

---

## 8. 总结

整个系统通过**多层隔离机制**确保staging和production环境的调度信号正确隔离:

1. **数据隔离**: 独立的环境记录和调度实例
2. **认证隔离**: 每个环境独立的API密钥
3. **路由隔离**: 按环境类型选择不同的worker队列策略
4. **运行隔离**: 消息携带环境标识，物理存储在不同的Redis队列

这种设计既保证了环境间的强隔离，又通过统一的 `AuthenticatedEnvironment` 对象简化了跨服务的环境信息传递，是一个典型的"边界清晰、上下文贯穿"的架构设计。
