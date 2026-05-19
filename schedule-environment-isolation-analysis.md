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

## 2. 分支与环境选择决策链

### 2.1 分支跟踪配置 (BranchTrackingConfig)

项目级别的分支映射配置存储在 `ConnectedGithubRepository` 表的 `branchTracking` 字段中，定义了哪些Git分支对应到哪些环境。

**文件**: `apps/webapp/app/v3/github.ts:3-12`

```typescript
export const BranchTrackingConfigSchema = z.object({
  prod: z.object({
    branch: z.string().optional(),
  }),
  staging: z.object({
      branch: z.string().optional(),
    }),
});

export function getTrackedBranchForEnvironment(
  branchTracking: BranchTrackingConfig | undefined,
  previewDeploymentsEnabled: boolean,
  environment: {
    type: "PRODUCTION" | "STAGING" | "DEVELOPMENT" | "PREVIEW";
    branchName?: string;
  }
): string | undefined {
  switch (environment.type) {
    case "PRODUCTION":
      return branchTracking?.prod?.branch;
    case "STAGING":
      return branchTracking?.staging?.branch;
    case "PREVIEW":
      return previewDeploymentsEnabled ? environment.branchName : undefined;
    case "DEVELOPMENT":
      return undefined;
  }
}
```

**配置示例**:
```javascript
// 连接GitHub仓库时的默认配置
branchTracking: {
  prod: { branch: "main" },     // main分支 → production环境
  staging: { branch: "develop" },   // develop分支 → staging环境
}
```

**文件**: `apps/webapp/app/services/projectSettings.server.ts:91-94`

```typescript
// 连接GitHub仓库时自动创建分支跟踪配置
branchTracking: {
  prod: { branch: defaultBranch },
  staging: {},
}
```

### 2.2 环境类型与分支的关联

**文件**: `apps/webapp/app/models/runtimeEnvironment.server.ts:94-166`

```typescript
// 环境通过两个关键字段建立分支关联:
type RuntimeEnvironment = {
  id: string;
  type: RuntimeEnvironmentType;  // PRODUCTION | STAGING | DEVELOPMENT | PREVIEW
  branchName: string | null;     // 关联的Git分支名（仅PREVIEW环境有值
  parentEnvironmentId: string | null;  // 父环境ID（PREVIEW环境指向PREVIEW根环境
  shortcode: string;           // 环境短标识
  // ...
}
```

**环境层级关系**:

```
┌──────────────────────────────────────────────────────────┐
│              RuntimeEnvironment 层级关系                 │
├──────────────────────────────────────────────────────────┤
│                                                  │
│  PRODUCTION (type=PRODUCTION)                      │
│    └─ shortcode: "prod"                          │
│    └─ branchName: null                            │
│    └─ parentEnvironmentId: null                  │
│                                                  │
│  STAGING (type=STAGING)                      │
│    └─ shortcode: "staging"                       │
│    └─ branchName: null                            │
│    └─ parentEnvironmentId: null                  │
│                                                  │
│  PREVIEW (type=PREVIEW, isBranchable=true)   │
│    └─ shortcode: "preview"                       │
│    └─ branchName: null                            │
│    └─ parentEnvironmentId: null                  │
│    │                                              │
│    ├─ PREVIEW (type=PREVIEW, branchName="feature-x")  │
│    │    └─ shortcode: "preview-feature-x" │
│    │    └─ branchName: "feature-x"           │
│    │    └─ parentEnvironmentId: <preview-root-id │
│    │                                              │
│    └─ PREVIEW (type=PREVIEW, branchName="feature-y")  │
│         └─ shortcode: "preview-feature-y" │
│         └─ branchName: "feature-y"           │
│         └─ parentEnvironmentId: <preview-root-id │
│                                                  │
│  DEVELOPMENT (type=DEVELOPMENT)                │
│    └─ shortcode: "dev"                             │
│    └─ branchName: null                            │
│    └─ parentEnvironmentId: null                  │
│                                                  │
└──────────────────────────────────────────────────────────┘
```

**关键设计**:
1. **PRODUCTION/STAGING/DEVELOPMENT 是**环境类型，每个项目各有一个根环境
2. **PREVIEW** 环境可以有多个子环境，每个子环境对应一个Git分支
3. `branchName` 字段仅在 PREVIEW 子环境上有值
4. `parentEnvironmentId` 建立父子环境的层级关系

### 2.3 部署时的环境选择决策链

**CLI部署命令: `packages/cli-v3/src/commands/deploy.ts:63-111`

```typescript
const DeployCommandOptions = CommonCommandOptions.extend({
  env: z.enum(["prod", "staging", "preview", "production"],
  branch: z.string().optional(),
  // ...
});

// 命令行选项说明
.option(
  "-e, --env <env>",
  "Deploy to a specific environment",
  "prod"
)
.option(
  "-b, --branch <branch>",
  "The preview branch to deploy to when passing --env preview"
)
```

**环境选择流程:

```
CLI部署 → 环境类型解析 → 环境ID查找 → 认证环境对象创建
```

**步骤1: CLI环境参数解析**
- `--env prod` → 查找 type=PRODUCTION
- `--env staging` → 查找 type=STAGING
- `--env preview` → 需要 `--branch` 参数

**步骤2: 预览环境分支匹配**

**文件**: `apps/webapp/app/models/runtimeEnvironment.server.ts:94-166

```typescript
export async function findEnvironmentByApiKey(
  apiKey: string,
  branchName: string | undefined
): Promise<AuthenticatedEnvironment | null> {
  // 1. 根据apiKey查找环境
  let environment = await $replica.runtimeEnvironment.findFirst({
    where: { apiKey },
    include: {
      // 如果提供了branchName，加载匹配的子环境
      childEnvironments: branchName
        ? {
            where: {
              branchName: sanitizeBranchName(branchName),
              archivedAt: null,
            },
          }
        : undefined,
    },
  });

  // 2. 如果是PREVIEW类型且提供了branchName
  if (environment.type === "PREVIEW") {
    if (!branchName) {
      logger.warn("Preview env with no branch name provided");
      return null;
    }

    // 返回匹配的子环境（如果存在）
    const childEnvironment = environment.childEnvironments.at(0);
    if (childEnvironment) {
      return toAuthenticated({
        ...childEnvironment,
        apiKey: environment.apiKey,  // 继承父环境的apiKey
        orgMember: environment.orgMember,
        organization: environment.organization,
        project: environment.project,
      });
    }

    // 分支不存在则返回null
    return null;
  }

  // 3. 非PREVIEW环境直接返回
  return toAuthenticated(environment);
}
```

**步骤3: 预览分支环境创建（如果不存在）**

**文件**: `apps/webapp/app/services/upsertBranch.server.ts:17-158

```typescript
export class UpsertBranchService {
  public async call(
    orgFilter: { type: "userMembership"; userId: string } | { type: "orgId"; organizationId: string },
    { parentEnvironmentId, branchName, git }: CreateBranchOptions
  ) {
    const sanitizedBranchName = sanitizeBranchName(branchName);

    // 1. 查找父环境（PREVIEW根环境）
    const parentEnvironment = await this.#prismaClient.runtimeEnvironment.findFirst({
      where: { id: parentEnvironmentId },
      include: { organization: true, project: true },
    });

    // 2. 检查父环境必须支持分支
    if (!parentEnvironment.isBranchableEnvironment) {
      return { success: false, error: "Your preview environment is not branchable" };
    }

    // 3. 创建或更新分支环境
    const branchSlug = `${slug(`${parentEnvironment.slug}-${sanitizedBranchName}`;
    const shortcode = branchSlug;

    const branch = await this.#prismaClient.runtimeEnvironment.upsert({
      where: {
        projectId_shortcode: {
        projectId: parentEnvironment.project.id,
        shortcode: shortcode,
      },
      create: {
        slug: branchSlug,
        apiKey: createApiKeyForEnv(parentEnvironment.type),
        shortcode,
        branchName: sanitizedBranchName,
        type: parentEnvironment.type,  // 继承父环境类型（PREVIEW）
        parentEnvironment: { connect: { id: parentEnvironment.id },
        organization: { connect: { id: parentEnvironment.organization.id } },
        project: { connect: { id: parentEnvironment.project.id } },
        git: git ?? undefined,
      },
      update: {
        git: git ?? undefined,
      },
    });

    return { success: true, branch, alreadyExisted: branch.createdAt < now };
  }
}
```

**分支环境创建后的完整流程:

```
部署 --env preview --branch feature-x
        │
        ▼
1. 使用PREVIEW环境的apiKey认证
        │
        ▼
2. findEnvironmentByApiKey(apiKey, "feature-x")
        │
        ├─► 查找子环境 branchName="feature-x"
        │
        ├─► 存在 → 返回子环境
        │
        └─► 不存在 →
              │
              ▼
3. UpsertBranchService 创建分支环境
        │
        ▼
4. 返回新创建的分支环境
```

### 2.4 调度实例归属与分支的关联

**文件**: `apps/webapp/app/v3/services/createBackgroundWorker.server.ts:626-784`

```typescript
export async function syncDeclarativeSchedules(
  tasks: TaskResource[],
  worker: BackgroundWorker,
  environment: AuthenticatedEnvironment,
  prisma: PrismaClientOrTransaction
) {
  // 1. 过滤出带schedule定义的task
  const tasksWithDeclarativeSchedules = tasks.filter((task) => task.schedule);

  // 2. 加载该项目已有的DECLARATIVE类型的调度
  const existingDeclarativeSchedules = await prisma.taskSchedule.findMany({
    where: {
      type: "DECLARATIVE",
      projectId: environment.projectId,
    },
    include: { instances: true },
  });

  // 3. 为每个带调度的task创建/更新调度
  for (const task of tasksWithDeclarativeSchedules) {
    // 关键: 检查环境过滤
    if (task.schedule.environments && task.schedule.environments.length > 0) {
      // 如果调度定义了environments数组，检查当前环境是否在列表中
      if (!task.schedule.environments.includes(environment.type)) {
        logger.debug("Skipping schedule creation due to environment filter", {
          taskId: task.id,
          environmentType: environment.type,
          allowedEnvironments: task.schedule.environments,
        });
        continue;  // 当前环境不在允许列表中，跳过
      }
    }

    // 查找该task在当前环境是否已有调度实例
    const existingSchedule = existingDeclarativeSchedules.find(
      (schedule) =>
        schedule.taskIdentifier === task.id &&
        schedule.instances.some(
          (instance) => instance.environmentId === environment.id
        )
    );

    if (existingSchedule) {
      // 更新现有调度
      const schedule = await prisma.taskSchedule.update({
        where: { id: existingSchedule.id },
        data: {
          generatorExpression: task.schedule.cron,
          generatorDescription: cronstrue.toString(task.schedule.cron),
          timezone: task.schedule.timezone,
        },
        include: { instances: true },
      });

      // 重新注册调度实例
      const instance = schedule.instances.at(0);
      if (instance) {
        await scheduleEngine.registerNextTaskScheduleInstance({
          instanceId: instance.id,
        });
      }
    } else {
      // 创建新调度及其实例
      const newSchedule = await prisma.taskSchedule.create({
        data: {
          friendlyId: generateFriendlyId("sched"),
          projectId: environment.projectId,
          taskIdentifier: task.id,
          generatorExpression: task.schedule.cron,
          generatorDescription: cronstrue.toString(task.schedule.cron),
          timezone: task.schedule.timezone,
          type: "DECLARATIVE",
          instances: {
            create: [
              {
                environmentId: environment.id,  // 关键: 关联到当前环境
                projectId: environment.projectId,
              },
            ],
          },
        },
        include: { instances: true },
      });

      const instance = newSchedule.instances.at(0);
      if (instance) {
        await scheduleEngine.registerNextTaskScheduleInstance({
          instanceId: instance.id,
        });
      }
    }
  }

  // 4. 删除不再需要的调度实例
  // 只删除当前环境的实例，不影响其他环境
  for (const schedule of potentiallyDeletableSchedules) {
    const canDeleteSchedule =
      schedule.instances.length === 0 ||
      schedule.instances.every(
      (instance) => instance.environmentId === environment.id
    );

    if (canDeleteSchedule) {
      // 所有实例都属于当前环境，可以删除整个调度
      await prisma.taskSchedule.delete({ where: { id: schedule.id });
    } else {
      // 只删除当前环境的实例，保留其他环境的
      await prisma.taskScheduleInstance.deleteMany({
        where: {
          taskScheduleId: schedule.id,
          environmentId: environment.id,
        },
      });
    }
  }
}
```

**调度实例归属决策链:

```
部署到环境 E (environmentId=env_123, type=STAGING, branchName=null)
        │
        ▼
syncDeclarativeSchedules(tasks, worker, E)
        │
        ▼
遍历每个带schedule的task
        │
        ├─► task.schedule.environments = ["PRODUCTION"]
        │       │
        │       ├─► E.type = STAGING
        │       │
        │       └─► 不在列表中 → 跳过 ❌
        │
        └─► task.schedule.environments = ["STAGING", "PRODUCTION"]
                │
                ├─► E.type = STAGING
                │
                └─► 在列表中 → 继续 ✅
                        │
                        ▼
                查找该task在E中是否已有实例
                        │
                        ├─► 有 → 更新调度cron/timezone
                        │       重新注册实例
                        │
                        └─► 无 → 创建TaskSchedule
                                │
                                └─► 创建TaskScheduleInstance
                                        │
                                        └─► environmentId = E.id
                                        └─► projectId = E.projectId
```

**关键点**:
1. **调度定义 (TaskSchedule) 是**项目级**的，跨环境共享
2. **调度实例 (TaskScheduleInstance) 是**环境级**的，每个环境独立
3. 一个调度可以在多个环境有实例，也可以只在特定环境有实例
4. 部署时只处理**当前环境**的调度实例，不影响其他环境

### 2.5 分支对运行器路由的影响

**文件**: `apps/webapp/app/runEngine/concerns/queues.server.ts:376-410`

```typescript
async getWorkerQueue(
  environment: AuthenticatedEnvironment,
  regionOverride?: string
): Promise<{ masterQueue: string; enableFastPath: boolean } | undefined> {
  // 开发环境: 使用environment.id作为队列名
  if (environment.type === "DEVELOPMENT") {
    return { masterQueue: environment.id, enableFastPath: true };
  }

  // 非开发环境: 通过WorkerGroupService获取项目默认worker组
  // 关键: 这里没有区分STAGING和PRODUCTION
  // 它们使用相同的默认worker组策略
  const workerGroupService = new WorkerGroupService({
    prisma: this.prisma,
    engine: this.engine,
  });

  const workerGroup = await workerGroupService.getDefaultWorkerGroupForProject({
    projectId: environment.projectId,
    regionOverride,
  });

  return {
    masterQueue: workerGroup.queueName,
    enableFastPath: workerGroup.supportsFastPath,
  };
}
```

**文件**: `internal-packages/run-engine/src/run-queue/workerQueueResolver.ts:30-64`

```typescript
export class WorkerQueueResolver {
  // 支持通过环境变量精确指定环境到队列的映射
  // 优先级: environmentId > projectId > orgId > workerQueue

  #getOverride(message: OutputPayloadV2): string | null {
    if (!this.overrides) return null;

    // 1. 按环境ID精确匹配（优先级最高）
    if (this.overrides.environmentId?.[message.environmentId]) {
      return this.overrides.environmentId[message.environmentId];
    }

    // 2. 按项目ID匹配
    if (this.overrides.projectId?.[message.projectId]) {
      return this.overrides.projectId[message.projectId];
    }

    // 3. 按组织ID匹配
    if (this.overrides.orgId?.[message.orgId]) {
      return this.overrides.orgId[message.orgId];
    }

    // 4. 按workerQueue匹配
    if (this.overrides.workerQueue?.[message.workerQueue]) {
      return this.overrides.workerQueue[message.workerQueue];
    }

    return null;
  }
}
```

**运行器路由决策链:

```
调度触发 → 环境E
        │
        ▼
getWorkerQueue(E)
        │
        ├─► E.type === DEVELOPMENT
        │       │
        │       └─► masterQueue = E.id  (特殊处理)
        │
        └─► E.type === STAGING / PRODUCTION / PREVIEW
                │
                ├─► 检查RUN_ENGINE_WORKER_QUEUE_OVERRIDES
                │       │
                │       ├─► 有environmentId覆盖 → 使用覆盖值
                │       │
                │       └─► 无 → 查询项目默认worker组
                │
                └─► 返回workerGroup.masterQueue
```

**各环境运行器隔离方式对比**:

| 场景 | 隔离方式 | 代码位置 |
|------|---------|----------|
| **DEVELOPMENT** | 每个开发环境独立队列，`masterQueue = environment.id` | queues.server.ts:380-381 |
| **STAGING** | 默认共享项目worker组，可通过覆盖配置隔离 | queues.server.ts:384-407 |
| **PRODUCTION** | 默认共享项目worker组，可通过覆盖配置隔离 | queues.server.ts:384-407 |
| **PREVIEW** | 默认共享项目worker组，可通过覆盖配置隔离 | queues.server.ts:384-407 |

**关键代码事实**:
- `queues.server.ts:380` 只对 `environment.type === "DEVELOPMENT"` 有特殊处理
- STAGING、PRODUCTION、PREVIEW 都通过 `WorkerGroupService.getDefaultWorkerGroupForProject()` 获取项目默认worker组
- 三类环境都支持 `RUN_ENGINE_WORKER_QUEUE_OVERRIDES` 按 `environmentId` 精确指定队列

---

## 3. 调度服务 (Schedule Engine)

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

### 3.4 调度触发回调

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
│  │    - STAGING/PRODUCTION/PREVIEW: 查询项目默认worker组        │  │
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

## 9. 总结

### 9.1 分支与环境选择决策链总览

"按项目和分支选定运行环境"的完整决策链如下:

```
用户部署 → CLI参数解析 → 环境类型匹配 → 分支映射 → 环境ID确定
                                                          │
                                                          ▼
                                                创建/更新调度实例
                                                          │
                                                          ▼
                                                调度触发 → 运行器选择 → 执行
```

**关键决策点**:

1. **项目级别**: 每个项目有独立的 `BranchTrackingConfig` 定义分支→环境映射
2. **环境类型**:
   - `PRODUCTION`: 关联 `branchTracking.prod.branch`（如 "main"）
   - `STAGING`: 关联 `branchTracking.staging.branch`（如 "develop"）
   - `PREVIEW`: 每个Git分支创建独立的子环境，`branchName` 字段存储分支名
   - `DEVELOPMENT`: 无分支关联

3. **调度实例归属**:
   - 部署时 `syncDeclarativeSchedules` 为当前环境创建/更新调度实例
   - 每个调度实例的 `environmentId` 精确绑定到特定环境
   - 支持按环境类型过滤（`task.schedule.environments`）

4. **运行器路由**:
   - `DEVELOPMENT` 和 `PREVIEW` 环境: 使用 `environment.id` 作为队列名，强隔离
   - `STAGING` 和 `PRODUCTION`: 默认共享项目worker组，可通过 `RUN_ENGINE_WORKER_QUEUE_OVERRIDES` 按环境ID精确指定

### 9.2 Staging与Production隔离机制

| 维度 | Staging | Production | 隔离机制 |
|------|---------|------------|----------|
| **数据模型** | 独立的RuntimeEnvironment记录，`type=STAGING` | 独立的RuntimeEnvironment记录，`type=PRODUCTION` | 独立的数据库记录，独立的apiKey |
| **分支映射** | `branchTracking.staging.branch` | `branchTracking.prod.branch` | 通过BranchTrackingConfig分别配置 |
| **调度实例** | 独立的TaskScheduleInstance，`environmentId=staging_env_id` | 独立的TaskScheduleInstance，`environmentId=prod_env_id` | 每个环境独立的调度实例 |
| **运行队列** | 默认共享项目worker组，可通过覆盖配置隔离 | 默认共享项目worker组，可通过覆盖配置隔离 | 支持RUN_ENGINE_WORKER_QUEUE_OVERRIDES按environmentId精确指定 |
| **执行环境** | 部署到staging环境的worker版本 | 部署到production环境的worker版本 | 独立部署，独立版本管理 |

### 9.3 多层隔离机制总结

整个系统通过**多层隔离机制**确保staging和production环境的调度信号正确隔离:

1. **数据隔离**: 独立的环境记录和调度实例
2. **认证隔离**: 每个环境独立的API密钥
3. **分支隔离**: 通过BranchTrackingConfig建立分支与环境的映射
4. **调度隔离**: 每个环境独立的调度实例，可按环境类型过滤
5. **路由隔离**: 按环境类型选择不同的worker队列策略
6. **运行隔离**: 消息携带环境标识，物理存储在不同的Redis队列
7. **部署隔离**: 独立的部署流程和版本管理

这种设计既保证了环境间的强隔离，又通过统一的 `AuthenticatedEnvironment` 对象简化了跨服务的环境信息传递，是一个典型的"边界清晰、上下文贯穿"的架构设计。
