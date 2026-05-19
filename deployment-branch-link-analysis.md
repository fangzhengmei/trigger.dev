# 分支化部署代码链路分析

本文档分析 Trigger.dev 中分支化部署的实现机制，重点修正和阐述以下三个核心环节：
1. 运行时根据请求头分支信息及鉴权结果映射到目标分支环境
2. 环境变量合并的优先级及预览分支标识变量的注入时机
3. 预览分支从创建、归档到任务队列处理的完整生命周期

---

## 一、运行时请求头分支信息到目标环境的映射

### 1.1 请求头分支信息提取

**文件**: `apps/webapp/app/services/apiAuth.server.ts:278-290`

```typescript
export function branchNameFromRequest(request: Request): string | undefined {
  return request.headers.get("x-trigger-branch") ?? undefined;
}

function getApiKeyFromRequest(request: Request): {
  apiKey: string | undefined;
  branchName: string | undefined;
} {
  const apiKey = getApiKeyFromHeader(request.headers.get("Authorization"));
  const branchName = branchNameFromRequest(request);

  return { apiKey, branchName };
}
```

**关键机制**：
- 从 HTTP 请求头 `x-trigger-branch` 中提取分支名
- 与 `Authorization` 头中的 API Key 一起传递给认证流程

### 1.2 API Key 认证与分支环境查找

**文件**: `apps/webapp/app/services/apiAuth.server.ts:88-103, 109-170`

```typescript
export async function authenticateApiRequestWithFailure(
  request: Request,
  options: { allowPublicKey?: boolean; allowJWT?: boolean } = {}
): Promise<ApiAuthenticationResult> {
  const { apiKey, branchName } = getApiKeyFromRequest(request);
  // ...
  const authentication = await authenticateApiKeyWithFailure(apiKey, { ...options, branchName });
  return authentication;
}

// 在 authenticateApiKey 中根据 key 类型调用不同的查找函数
switch (result.type) {
  case "PUBLIC":
  case "PRIVATE": {
    const environment = await findEnvironmentByApiKey(result.apiKey, options.branchName);
    // ...
  }
}
```

### 1.3 根据 API Key + BranchName 查找环境

**文件**: `apps/webapp/app/models/runtimeEnvironment.server.ts:94-166`

```typescript
export async function findEnvironmentByApiKey(
  apiKey: string,
  branchName: string | undefined
): Promise<AuthenticatedEnvironment | null> {
  const include = {
    ...authIncludeBase,
    childEnvironments: branchName
      ? {
          where: {
            branchName: sanitizeBranchName(branchName),
            archivedAt: null,  // 只查找未归档的分支环境
          },
        }
      : undefined,
  };

  // 1. 先用 API Key 查找主环境（父环境）
  let environment = await $replica.runtimeEnvironment.findFirst({
    where: { apiKey },
    include,
  });

  // 2. 如果 API Key 属于预览环境（PREVIEW 类型）且提供了 branchName
  if (environment.type === "PREVIEW") {
    if (!branchName) {
      return null;  // 预览环境必须提供分支名
    }

    const childEnvironment = environment.childEnvironments.at(0);
    if (childEnvironment) {
      // 返回子环境，但使用父环境的 apiKey 用于后续认证
      return toAuthenticated({
        ...childEnvironment,
        apiKey: environment.apiKey,  // 重要：继承父环境的 API Key
        orgMember: environment.orgMember,
        organization: environment.organization,
        project: environment.project,
      });
    }
    return null;
  }

  return toAuthenticated(environment);
}
```

### 1.4 个人/组织访问令牌的分支环境解析

**文件**: `apps/webapp/app/services/apiAuth.server.ts:440-614`

```typescript
export async function authenticatedEnvironmentForAuthentication(
  auth: AuthenticationResult,
  projectRef: string,
  slug: string,
  branch?: string
): Promise<AuthenticatedEnvironment> {
  // ...
  const sanitizedBranch = sanitizeBranchName(branch);

  if (!sanitizedBranch) {
    // 没有分支名，直接按 slug 查找主环境
    const environment = await $replica.runtimeEnvironment.findFirst({
      where: { projectId: project.id, slug: slug },
      include: authIncludeBase,
    });
    return toAuthenticated(environment);
  }

  // 有分支名，查找对应的 PREVIEW 类型子环境
  const environment = await $replica.runtimeEnvironment.findFirst({
    where: {
      projectId: project.id,
      type: "PREVIEW",
      branchName: sanitizedBranch,
      archivedAt: null,  // 排除已归档的分支
    },
    include: authIncludeWithParent,
  });

  // PREVIEW 环境重用父环境的 apiKey
  return toAuthenticated({
    ...environment,
    apiKey: environment.parentEnvironment.apiKey,
  });
}
```

### 1.5 分支环境映射总结

```
请求到达
    ↓
提取 x-trigger-branch 头 + Authorization 头
    ↓
API Key 认证 → 查找主环境（父环境）
    ↓
如果提供了 branchName：
  ├─ 主环境是 PREVIEW 类型 → 查找匹配 branchName 的子环境
  └─ 认证使用 PAT/OAT → 按 branchName 直接查找 PREVIEW 环境
    ↓
返回 AuthenticatedEnvironment（子环境继承父环境 apiKey）
```

---

## 二、环境变量合并优先级与注入时机

### 2.1 环境变量合并优先级

**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:899-933`

```typescript
export async function resolveVariablesForEnvironment(
  runtimeEnvironment: RuntimeEnvironmentForEnvRepo,
  parentEnvironment?: RuntimeEnvironmentForEnvRepo
) {
  // 1. 用户可覆盖的 Trigger 内置变量（优先级最低）
  const overridableTriggerVariables = await resolveOverridableTriggerVariables(runtimeEnvironment);

  // 2. 用户可覆盖的 OTEL 变量
  const overridableOtelVariables = await resolveOverridableOtelVariables(runtimeEnvironment);

  // 3. 用户自定义环境变量（支持父子环境继承，子覆盖父）
  let projectSecrets = await environmentVariablesRepository.getEnvironmentVariables(
    runtimeEnvironment.projectId,
    runtimeEnvironment.id,
    parentEnvironment?.id
  );

  // 4. 内置环境变量（DEV 或 PROD，优先级最高）
  const builtInVariables = runtimeEnvironment.type === "DEVELOPMENT"
    ? await resolveBuiltInDevVariables(runtimeEnvironment)
    : await resolveBuiltInProdVariables(runtimeEnvironment, parentEnvironment);

  // deduplicateVariableArray: 后出现的覆盖先出现的
  const result = deduplicateVariableArray([
    ...overridableTriggerVariables,  // 优先级 4（最低）
    ...overridableOtelVariables,     // 优先级 3
    ...projectSecrets,               // 优先级 2
    ...builtInVariables,             // 优先级 1（最高，最后合并）
  ]);

  return result;
}
```

**优先级（从高到低）**：
1. **内置环境变量** (`builtInVariables`) - 最后合并，优先级最高
2. **用户自定义变量** (`projectSecrets`) - 子环境变量覆盖父环境
3. **可覆盖 OTEL 变量** (`overridableOtelVariables`)
4. **可覆盖 Trigger 变量** (`overridableTriggerVariables`) - 优先级最低

### 2.2 父子环境变量继承逻辑

**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:645-685`

```typescript
async #getSecretEnvironmentVariables(
  projectId: string,
  environmentId: string,
  parentEnvironmentId?: string
) {
  // 先加载父环境变量
  const parentSecrets = parentEnvironmentId
    ? await secretStore.getSecrets(SecretValue, secretKeyEnvironmentPrefix(projectId, parentEnvironmentId))
    : [];

  // 再加载子环境变量
  const childSecrets = await secretStore.getSecrets(SecretValue, secretKeyEnvironmentPrefix(projectId, environmentId));

  // 合并：子环境变量覆盖父环境
  const mergedSecrets = new Map<string, string>();
  for (const secret of parentSecrets) {
    const { key: parsedKey } = parseSecretKey(secret.key);
    mergedSecrets.set(parsedKey, secret.value.secret);
  }
  for (const secret of childSecrets) {
    const { key: parsedKey } = parseSecretKey(secret.key);
    mergedSecrets.set(parsedKey, secret.value.secret);  // 子覆盖父
  }
  // ...
}
```

### 2.3 预览分支标识变量的注入时机

**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:1123-1130`

```typescript
// 在 resolveBuiltInProdVariables 内部注入
if (runtimeEnvironment.branchName) {
  result = result.concat([
    {
      key: "TRIGGER_PREVIEW_BRANCH",
      value: runtimeEnvironment.branchName,
    },
  ]);
}
```

**注入时机与使用场景**：

| 阶段 | 注入位置 | 用途 |
|------|----------|------|
| **构建阶段** | `packages/cli-v3/src/entryPoints/managed-index-controller.ts:33` | CLI 从环境变量读取 `TRIGGER_PREVIEW_BRANCH`，用于创建 API 客户端时指定分支 |
| **任务启动时** | `apps/webapp/app/v3/services/worker/workerGroupTokenService.server.ts:545-584` | `getEnvVars()` 调用 `resolveVariablesForEnvironment()`，注入到任务运行环境 |
| **运行时访问** | 任务代码中通过 `process.env.TRIGGER_PREVIEW_BRANCH` 访问 | 用于业务逻辑识别当前运行的分支 |

**关键流程**：
```
Worker 启动任务
    ↓
getEnvVars(environment, runId, machinePreset, parentEnvironment)
    ↓
resolveVariablesForEnvironment(environment, parentEnvironment)
    ↓
resolveBuiltInProdVariables() → 检测到 branchName → 注入 TRIGGER_PREVIEW_BRANCH
    ↓
合并所有变量 → 转换为 Record<string, string>
    ↓
传递给 Docker 容器作为环境变量
    ↓
任务代码中通过 process.env.TRIGGER_PREVIEW_BRANCH 访问
```

### 2.4 运行时额外注入的变量

**文件**: `apps/webapp/app/v3/services/worker/workerGroupTokenService.server.ts:559-578`

在 `resolveVariablesForEnvironment` 返回的基础上，还会额外注入：

```typescript
variables.push(
  ...[
    { key: "TRIGGER_JWT", value: jwt },                    // 任务认证 JWT
    { key: "TRIGGER_RUN_ID", value: runId },               // 当前运行 ID
    { key: "TRIGGER_MACHINE_PRESET", value: machinePreset.name },  // 机器配置
  ]
);

if (taskEventStore) {
  variables.push(
    ...[
      { key: "OTEL_RESOURCE_ATTRIBUTES", value: resourceAttributes },
      { key: "TRIGGER_OTEL_RESOURCE_ATTRIBUTES", value: resourceAttributes },
    ]
  );
}
```

---

## 三、预览分支完整生命周期

### 3.1 预览分支创建

**文件**: `apps/webapp/app/services/upsertBranch.server.ts:17-158`

```typescript
public async call(
  orgFilter: { type: "userMembership"; userId: string } | { type: "orgId"; organizationId: string },
  { parentEnvironmentId, branchName, git }: CreateBranchOptions
) {
  const sanitizedBranchName = sanitizeBranchName(branchName);

  // 1. 验证父环境支持分支
  const parentEnvironment = await this.#prismaClient.runtimeEnvironment.findFirst({
    where: { id: parentEnvironmentId, /* ... */ },
  });
  if (!parentEnvironment.isBranchableEnvironment) {
    return { success: false, error: "Your preview environment is not branchable" };
  }

  // 2. 检查分支数量限制
  const limits = await checkBranchLimit(
    this.#prismaClient,
    parentEnvironment.organization.id,
    parentEnvironment.project.id,
    sanitizedBranchName
  );
  if (limits.isAtLimit) {
    return { success: false, error: `You've used all ${limits.used} of ${limits.limit} branches...` };
  }

  // 3. 创建或更新分支环境
  const branchSlug = `${slug(`${parentEnvironment.slug}-${sanitizedBranchName}`)}`;
  const branch = await this.#prismaClient.runtimeEnvironment.upsert({
    where: {
      projectId_shortcode: {
        projectId: parentEnvironment.project.id,
        shortcode: branchSlug,
      },
    },
    create: {
      slug: branchSlug,
      apiKey: createApiKeyForEnv(parentEnvironment.type),
      pkApiKey: createPkApiKeyForEnv(parentEnvironment.type),
      shortcode: branchSlug,
      branchName: sanitizedBranchName,
      type: parentEnvironment.type,
      parentEnvironment: { connect: { id: parentEnvironment.id } },
      git: git ?? undefined,
      // ...
    },
    update: {
      git: git ?? undefined,  // 更新 Git 元数据
    },
  });

  return { success: true, alreadyExisted: branch.createdAt < now, branch, /* ... */ };
}
```

**创建入口**：
- Web 界面：`apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.branches/route.tsx:139-197`
- API 调用：通过 `UpsertBranchService` 直接调用

### 3.2 分支部署与版本选择

**部署创建流程**（参见第一部分）：
1. CLI 收集 Git 元数据（`createGitMeta()`）
2. 调用部署 API，携带 `gitMeta` 和 `branchName`
3. 创建 `WorkerDeployment` 关联到分支环境
4. 构建 Docker 镜像
5. 晋升为当前版本（`WorkerDeploymentPromotion`）

**版本选择机制**：
- 每个环境（包括分支环境）有独立的 `WorkerDeploymentPromotion`
- `label = "current"` 标记当前活跃版本
- 任务执行时通过 `environmentId` 查找对应环境的当前部署

### 3.3 分支归档

**文件**: `apps/webapp/app/services/archiveBranch.server.ts:13-87`

```typescript
public async call(
  orgFilter: { type: "userMembership"; userId: string } | { type: "orgId"; organizationId: string },
  { environmentId }: { environmentId: string }
) {
  const environment = await this.#prismaClient.runtimeEnvironment.findFirstOrThrow({
    where: { id: environmentId, /* ... */ },
  });

  if (!environment.parentEnvironmentId) {
    return { success: false, error: "This isn't a branch, and cannot be archived." };
  }

  // 归档操作：设置 archivedAt，修改 slug 和 shortcode 释放名称
  const slug = `${environment.slug}-${nanoid(6)}`;
  const shortcode = slug;

  const updatedBranch = await this.#prismaClient.runtimeEnvironment.update({
    where: { id: environmentId },
    data: { archivedAt: new Date(), slug, shortcode },
  });

  return { success: true, branch: updatedBranch, /* ... */ };
}
```

**归档效果**：
- 设置 `archivedAt` 时间戳
- 修改 `slug` 和 `shortcode`（添加随机后缀），释放原名称供新分支使用
- 变为只读，无法触发新任务

### 3.4 归档分支的队列处理

**文件**: `internal-packages/run-engine/src/engine/systems/dequeueSystem.ts:849-857, 298-308`

```typescript
// 在 dequeue 时检查环境是否已归档
if (run.runtimeEnvironment.archivedAt) {
  span.setAttribute("result", "RUN_ENVIRONMENT_ARCHIVED");
  return {
    success: false as const,
    code: "RUN_ENVIRONMENT_ARCHIVED",
    message: `Run is on an archived environment: ${run.id}`,
    run,
  };
}

// 处理归档环境的任务：确认消息，不执行
case "RUN_ENVIRONMENT_ARCHIVED": {
  this.$.logger.warn("RunEngine.dequeueFromWorkerQueue(): Run environment archived", { /* ... */ });
  await this.$.runQueue.acknowledgeMessage(orgId, runId);  // 直接确认，不执行
  return;
}
```

**调度引擎中的检查**：
**文件**: `internal-packages/schedule-engine/src/engine/index.ts:362-370`

```typescript
if (instance.environment.archivedAt) {
  this.logger.debug("Environment is archived, skipping schedule", { /* ... */ });
  span.setAttribute("skip_reason", "environment_archived");
  return;  // 跳过已归档环境的定时任务
}
```

### 3.5 完整生命周期图

```
┌─────────────────┐
│  创建预览分支   │
│  UpsertBranchService
│  - 验证父环境支持分支
│  - 检查分支数量限制
│  - 创建 RuntimeEnvironment
│    (type=PREVIEW, branchName=xxx)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  部署代码版本   │
│  InitializeDeploymentService
│  - 收集 GitMeta
│  - 创建 WorkerDeployment
│  - 构建 Docker 镜像
│  - 晋升为 CURRENT 版本
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  正常运行阶段   │
│  - API 请求通过 x-trigger-branch 映射
│  - 任务入队关联到分支环境
│  - 执行时使用该环境的 CURRENT 部署
│  - 注入 TRIGGER_PREVIEW_BRANCH 变量
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  归档分支       │
│  ArchiveBranchService
│  - 设置 archivedAt
│  - 修改 slug/shortcode 释放名称
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  归档后处理     │
│  - 已有任务：dequeue 时检测到 archivedAt
│    → 直接 ack，不执行，返回 RUN_ENVIRONMENT_ARCHIVED
│  - 定时任务：schedule 引擎跳过
│  - API 请求：findEnvironmentByApiKey 排除
└─────────────────┘
```

### 3.6 生命周期关键检查点

| 阶段 | 检查点 | 代码位置 | 行为 |
|------|--------|----------|------|
| **创建** | `isBranchableEnvironment` | `upsertBranch.server.ts:81` | 父环境必须支持分支 |
| **创建** | 分支数量限制 | `upsertBranch.server.ts:88-100` | 检查计划配额 |
| **API 路由** | `archivedAt: null` | `runtimeEnvironment.server.ts:104` | 查找环境时排除归档 |
| **任务出队** | `runtimeEnvironment.archivedAt` | `dequeueSystem.ts:849` | 归档环境任务直接丢弃 |
| **定时调度** | `instance.environment.archivedAt` | `schedule-engine/index.ts:362` | 跳过归档环境的调度 |

---

## 四、核心设计模式总结

### 4.1 环境分层模型

```
RuntimeEnvironment
├─ 主环境（如 staging, production）
│   ├─ isBranchableEnvironment: true/false
│   └─ apiKey: 用于认证
└─ 分支环境（PREVIEW 类型）
    ├─ branchName: Git 分支名
    ├─ parentEnvironmentId: 指向父环境
    ├─ apiKey: 独立生成，但认证时使用父环境的 key
    └─ archivedAt: 归档时间戳（null 表示活跃）
```

### 4.2 关键设计决策

1. **API Key 继承**：分支环境认证时使用父环境的 API Key，简化密钥管理
2. **动态环境解析**：通过 `x-trigger-branch` 头动态解析目标环境，无需为每个分支创建独立密钥
3. **软删除机制**：通过 `archivedAt` 实现软删除，保留历史数据同时释放名称
4. **变量分层合并**：四级优先级合并机制，确保用户变量可以覆盖系统默认值
5. **多系统一致性检查**：运行引擎、调度引擎、API 层都检查 `archivedAt`，确保归档分支完全隔离
