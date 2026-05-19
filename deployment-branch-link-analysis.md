# 分支化部署代码链路分析

本文档分析 Trigger.dev 中分支化部署的实现机制，重点阐述三个核心环节的关键代码路径：
1. 运行时根据请求头分支信息及鉴权结果映射到目标分支环境
2. 环境变量合并的优先级及预览分支标识变量的注入时机
3. 预览分支从创建、归档到任务队列处理的完整生命周期

---

## 一、运行时分支环境映射：请求头 + 鉴权 → 目标环境

### 1.1 请求头分支信息提取

**文件**: `apps/webapp/app/services/apiAuth.server.ts:278-290`

请求通过 `x-trigger-branch` HTTP 头携带目标分支名：

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

### 1.2 API Key 鉴权时的分支环境解析

**文件**: `apps/webapp/app/services/apiAuth.server.ts:88-104`

```typescript
export async function authenticateApiRequestWithFailure(
  request: Request,
  options: { allowPublicKey?: boolean; allowJWT?: boolean } = {}
): Promise<ApiAuthenticationResult> {
  const { apiKey, branchName } = getApiKeyFromRequest(request);

  if (!apiKey) {
    return { ok: false, error: "Invalid API Key" };
  }

  // branchName 传递给 authenticateApiKeyWithFailure
  const authentication = await authenticateApiKeyWithFailure(
    apiKey,
    { ...options, branchName }
  );

  return authentication;
}
```

### 1.3 根据 API Key + BranchName 查找目标环境

**文件**: `apps/webapp/app/models/runtimeEnvironment.server.ts:94-166`

`findEnvironmentByApiKey()` 是核心的分支环境路由函数：

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
            archivedAt: null,
          },
        }
      : undefined,
  };

  // 1. 先通过 API Key 找到父环境
  let environment = await $replica.runtimeEnvironment.findFirst({
    where: { apiKey },
    include,
  });

  // 2. 如果是 PREVIEW 类型环境且提供了 branchName
  if (environment.type === "PREVIEW") {
    if (!branchName) {
      logger.warn("Preview env with no branch name provided");
      return null;
    }

    // 3. 从 childEnvironments 中找到匹配分支名的子环境
    const childEnvironment = environment.childEnvironments.at(0);

    if (childEnvironment) {
      // 4. 返回子环境，但复用父环境的 apiKey（用于下游 JWT 签名等）
      return toAuthenticated({
        ...childEnvironment,
        apiKey: environment.apiKey,
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

### 1.4 PAT/OAT 鉴权时的分支环境解析

**文件**: `apps/webapp/app/services/apiAuth.server.ts:440-614`

对于 Personal Access Token (PAT) 和 Organization Access Token (OAT)，分支环境解析逻辑类似：

```typescript
export async function authenticatedEnvironmentForAuthentication(
  auth: AuthenticationResult,
  projectRef: string,
  slug: string,
  branch?: string
): Promise<AuthenticatedEnvironment> {
  switch (auth.type) {
    case "personalAccessToken":
    case "organizationAccessToken": {
      const sanitizedBranch = sanitizeBranchName(branch);

      if (!sanitizedBranch) {
        // 无分支名，返回普通环境
        const environment = await $replica.runtimeEnvironment.findFirst({
          where: { projectId: project.id, slug: slug },
          include: authIncludeBase,
        });
        return toAuthenticated(environment);
      }

      // 有分支名，查找 PREVIEW 类型的子环境
      const environment = await $replica.runtimeEnvironment.findFirst({
        where: {
          projectId: project.id,
          type: "PREVIEW",
          branchName: sanitizedBranch,
          archivedAt: null,
        },
        include: authIncludeWithParent,
      });

      // 复用父环境的 apiKey
      return toAuthenticated({
        ...environment,
        apiKey: environment.parentEnvironment.apiKey,
      });
    }
  }
}
```

### 1.5 实际 API 路由中的使用示例

**文件**: `apps/webapp/app/routes/api.v1.projects.$projectRef.$env.workers.$tagName.ts:19-51`

```typescript
const HeadersSchema = z.object({
  "x-trigger-branch": z.string().optional(),
});

export async function loader({ request, params }: LoaderFunctionArgs) {
  const authenticationResult = await authenticateRequest(request, {
    personalAccessToken: true,
    organizationAccessToken: true,
    apiKey: false,
  });

  const parsedHeaders = HeadersSchema.safeParse(Object.fromEntries(request.headers));
  const triggerBranch = parsedHeaders.success
    ? parsedHeaders.data["x-trigger-branch"]
    : undefined;

  // 鉴权结果 + branch 一起传递，映射到目标环境
  const runtimeEnv = await authenticatedEnvironmentForAuthentication(
    authenticationResult,
    projectRef,
    env,
    triggerBranch
  );

  // 使用 runtimeEnv 查找对应环境的当前部署
  const currentWorker = await findCurrentWorkerFromEnvironment(
    { id: runtimeEnv.id, type: runtimeEnv.type },
    $replica,
    params.tagName
  );
}
```

---

## 二、环境变量合并优先级与预览分支标识注入

### 2.1 环境变量合并的核心函数

**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:899-933`

`resolveVariablesForEnvironment()` 是环境变量合并的入口：

```typescript
export async function resolveVariablesForEnvironment(
  runtimeEnvironment: RuntimeEnvironmentForEnvRepo,
  parentEnvironment?: RuntimeEnvironmentForEnvRepo
) {
  // 1. 获取项目环境变量（支持父子环境继承）
  let projectSecrets = await environmentVariablesRepository.getEnvironmentVariables(
    runtimeEnvironment.projectId,
    runtimeEnvironment.id,
    parentEnvironment?.id
  );

  projectSecrets = renameVariables(projectSecrets, {
    OTEL_RESOURCE_ATTRIBUTES: "CUSTOM_OTEL_RESOURCE_ATTRIBUTES",
  });

  // 2. 可覆盖的 Trigger 内置变量
  const overridableTriggerVariables = await resolveOverridableTriggerVariables(
    runtimeEnvironment
  );

  // 3. 内置环境变量（DEV 或 PROD）
  const builtInVariables =
    runtimeEnvironment.type === "DEVELOPMENT"
      ? await resolveBuiltInDevVariables(runtimeEnvironment)
      : await resolveBuiltInProdVariables(runtimeEnvironment, parentEnvironment);

  // 4. 可覆盖的 OTEL 变量（仅 DEV）
  const overridableOtelVariables =
    runtimeEnvironment.type === "DEVELOPMENT"
      ? await resolveOverridableOtelDevVariables(runtimeEnvironment)
      : [];

  // 5. 合并去重 —— 关键：数组顺序决定优先级
  const result = deduplicateVariableArray([
    ...overridableTriggerVariables,  // 优先级最低
    ...overridableOtelVariables,
    ...projectSecrets,               // 用户自定义变量
    ...builtInVariables,             // 优先级最高（内置变量）
  ]);

  return result;
}
```

### 2.2 去重算法与优先级规则

**文件**: `apps/webapp/app/v3/deduplicateVariableArray.server.ts:4-14`

```typescript
export function deduplicateVariableArray(variables: EnvironmentVariable[]) {
  const result: EnvironmentVariable[] = [];
  // 反向遍历，后面的变量会覆盖前面的
  for (const variable of [...variables].reverse()) {
    if (!result.some((v) => v.key === variable.key)) {
      result.push(variable);
    }
  }
  // 反转回来保持原顺序，但优先级已确定
  return result.reverse();
}
```

**优先级（从高到低）**：
1. `builtInVariables`（内置变量，如 TRIGGER_API_URL、TRIGGER_PREVIEW_BRANCH）
2. `projectSecrets`（用户在项目/环境中设置的变量）
3. `overridableOtelVariables`（OTEL 相关可覆盖变量）
4. `overridableTriggerVariables`（Trigger 可覆盖变量）

> **重要修正**：之前的理解错误，实际是**内置变量优先级最高**，可以覆盖用户变量。

### 2.3 预览分支标识变量的注入时机

**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:1123-1130`

`TRIGGER_PREVIEW_BRANCH` 在 `resolveBuiltInProdVariables()` 中作为内置变量注入：

```typescript
// 在 resolveBuiltInProdVariables 函数内部
if (runtimeEnvironment.branchName) {
  result = result.concat([
    {
      key: "TRIGGER_PREVIEW_BRANCH",
      value: runtimeEnvironment.branchName,
    },
  ]);
}
```

**注入时机说明**：
- 属于 `builtInVariables` 的一部分，因此优先级最高
- 在 `resolveVariablesForEnvironment()` 调用时动态注入
- 仅当 `runtimeEnvironment.branchName` 非空时才注入（即预览分支环境）

### 2.4 SDK 端对 TRIGGER_PREVIEW_BRANCH 的使用

**文件**: `packages/core/src/v3/apiClientManager/index.ts:47-55`

SDK 运行时读取该环境变量用于自动识别分支：

```typescript
get branchName(): string | undefined {
  const config = this.#getConfig();
  const value =
    config?.previewBranch ??
    getEnvVar("TRIGGER_PREVIEW_BRANCH") ??
    getEnvVar("VERCEL_GIT_COMMIT_REF") ??
    undefined;
  return value ? value : undefined;
}
```

### 2.5 父子环境变量继承逻辑

**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:645-685`

```typescript
async #getSecretEnvironmentVariables(
  projectId: string,
  environmentId: string,
  parentEnvironmentId?: string
) {
  const parentSecrets = parentEnvironmentId
    ? await secretStore.getSecrets(
        SecretValue,
        secretKeyEnvironmentPrefix(projectId, parentEnvironmentId)
      )
    : [];

  const childSecrets = await secretStore.getSecrets(
    SecretValue,
    secretKeyEnvironmentPrefix(projectId, environmentId)
  );

  // 子环境变量覆盖父环境
  const mergedSecrets = new Map<string, string>();
  for (const secret of parentSecrets) {
    const { key: parsedKey } = parseSecretKey(secret.key);
    mergedSecrets.set(parsedKey, secret.value.secret);
  }
  for (const secret of childSecrets) {
    const { key: parsedKey } = parseSecretKey(secret.key);
    mergedSecrets.set(parsedKey, secret.value.secret); // 子覆盖父
  }
  // ...
}
```

**父子继承优先级**：子环境变量 > 父环境变量

---

## 三、预览分支完整生命周期

### 3.1 分支创建

**文件**: `apps/webapp/app/services/upsertBranch.server.ts:10-159`

`UpsertBranchService` 负责创建或更新预览分支环境：

```typescript
public async call(
  orgFilter: { type: "userMembership"; userId: string } | { type: "orgId"; organizationId: string },
  { parentEnvironmentId, branchName, git }: CreateBranchOptions
) {
  const sanitizedBranchName = sanitizeBranchName(branchName);

  // 1. 查找父环境（必须是 isBranchableEnvironment = true）
  const parentEnvironment = await this.#prismaClient.runtimeEnvironment.findFirst({
    where: { id: parentEnvironmentId, /* 权限检查 */ },
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
  const apiKey = createApiKeyForEnv(parentEnvironment.type);
  const pkApiKey = createPkApiKeyForEnv(parentEnvironment.type);
  const shortcode = branchSlug;

  const branch = await this.#prismaClient.runtimeEnvironment.upsert({
    where: {
      projectId_shortcode: {
        projectId: parentEnvironment.project.id,
        shortcode: shortcode,
      },
    },
    create: {
      slug: branchSlug,
      apiKey,
      pkApiKey,
      shortcode,
      branchName: sanitizedBranchName,
      type: parentEnvironment.type,
      parentEnvironment: { connect: { id: parentEnvironment.id } },
      git: git ?? undefined,
    },
    update: {
      git: git ?? undefined,
    },
  });

  return { success: true, branch, /* ... */ };
}
```

### 3.2 分支部署

部署流程与普通环境相同，但部署的 `environmentId` 是预览分支环境的 ID：

**文件**: `apps/webapp/app/v3/services/initializeDeployment.server.ts`

```typescript
// 部署时 environment.id 是预览分支环境的 ID
const deployment = await createDeploymentWithNextVersion(
  this._prisma,
  environment.id,  // 预览分支环境 ID
  async (nextVersion) => {
    return {
      // ...
      git: payload.gitMeta ?? undefined,
      commitSHA: payload.gitMeta?.commitSha ?? undefined,
      // ...
    };
  }
);
```

### 3.3 运行时请求路由

运行时通过 `x-trigger-branch` 请求头将请求路由到正确的分支环境：

1. 请求携带 `x-trigger-branch: feature/new-ui` 和 API Key
2. `authenticateApiRequestWithFailure()` 提取 branchName
3. `findEnvironmentByApiKey(apiKey, branchName)` 找到对应的预览分支环境
4. 使用该环境的 `id` 查找 `WorkerDeploymentPromotion` 获取当前部署
5. 执行对应版本的代码

### 3.4 任务队列处理

**任务关联环境**：任务入队时关联到 `RuntimeEnvironment.id`（预览分支环境 ID）

**执行时版本选择**：
```
TaskRun.environmentId
    ↓
WorkerDeploymentPromotion (environmentId + label="current")
    ↓
WorkerDeployment (对应分支的代码版本)
    ↓
imageReference (Docker 镜像)
```

### 3.5 分支归档

**文件**: `apps/webapp/app/services/archiveBranch.server.ts:6-88`

`ArchiveBranchService` 负责归档分支：

```typescript
public async call(
  orgFilter: { type: "userMembership"; userId: string } | { type: "orgId"; organizationId: string },
  { environmentId }: { environmentId: string }
) {
  const environment = await this.#prismaClient.runtimeEnvironment.findFirstOrThrow({
    where: { id: environmentId, /* 权限检查 */ },
  });

  if (!environment.parentEnvironmentId) {
    return { success: false, error: "This isn't a branch, and cannot be archived." };
  }

  // 归档操作：设置 archivedAt，修改 slug 和 shortcode 避免冲突
  const slug = `${environment.slug}-${nanoid(6)}`;
  const shortcode = slug;

  const updatedBranch = await this.#prismaClient.runtimeEnvironment.update({
    where: { id: environmentId },
    data: { archivedAt: new Date(), slug, shortcode },
  });

  return { success: true, branch: updatedBranch };
}
```

**归档后的影响**：
- `archivedAt` 非空，`findEnvironmentByApiKey()` 中的 `archivedAt: null` 条件不再匹配
- 修改 `slug` 和 `shortcode` 释放原名称供新分支使用
- 已在队列中的任务仍可执行（因为 `environmentId` 不变），但新请求无法再路由到该环境

### 3.6 分支数量限制检查

**文件**: `apps/webapp/app/services/upsertBranch.server.ts:161-190`

```typescript
export async function checkBranchLimit(
  prisma: PrismaClientOrTransaction,
  organizationId: string,
  projectId: string,
  newBranchName?: string
) {
  const usedEnvs = await prisma.runtimeEnvironment.findMany({
    where: {
      projectId,
      branchName: { not: null },
      archivedAt: null, // 只统计未归档的分支
    },
  });

  const count = newBranchName
    ? usedEnvs.filter((env) => env.branchName !== newBranchName).length
    : usedEnvs.length;

  const baseLimit = await getLimit(organizationId, "branches", 100_000_000);
  const currentPlan = await getCurrentPlan(organizationId);
  const purchasedBranches = currentPlan?.v3Subscription?.addOns?.branches?.purchased ?? 0;
  const limit = baseLimit + purchasedBranches;

  return { used: count, limit, isAtLimit: count >= limit };
}
```

---

## 四、完整链路串联

### 4.1 分支化部署全流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                     分支创建阶段                                  │
├─────────────────────────────────────────────────────────────────┤
│  Git Push / PR 创建                                              │
│         ↓                                                        │
│  UpsertBranchService.call()                                      │
│    ├─ 验证父环境 isBranchableEnvironment = true                  │
│    ├─ 检查分支数量限制                                            │
│    └─ 创建 RuntimeEnvironment (type=PREVIEW, branchName=xxx)     │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│                     部署构建阶段                                  │
├─────────────────────────────────────────────────────────────────┤
│  CLI deploy 命令                                                │
│    ├─ createGitMeta() 收集分支信息                               │
│    └─ 上传构建产物 + GitMeta                                    │
│         ↓                                                        │
│  InitializeDeploymentService                                    │
│    └─ 创建 WorkerDeployment (environmentId=分支环境ID)           │
│         ↓                                                        │
│  构建 Docker 镜像 (包含 TRIGGER_PREVIEW_BRANCH build-arg)       │
│         ↓                                                        │
│  ChangeCurrentDeploymentService                                 │
│    └─ 设置 WorkerDeploymentPromotion (label=current)             │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│                     运行时请求阶段                                │
├─────────────────────────────────────────────────────────────────┤
│  API 请求 (Authorization + x-trigger-branch: feature/new-ui)   │
│         ↓                                                        │
│  authenticateApiRequestWithFailure()                            │
│    ├─ 提取 apiKey 和 branchName                                  │
│    └─ findEnvironmentByApiKey(apiKey, branchName)               │
│         ├─ 通过 apiKey 找到父环境                                │
│         └─ 从 childEnvironments 找到匹配的分支环境               │
│         ↓                                                        │
│  业务逻辑处理                                                    │
│    ├─ 使用分支环境的 id 查找当前部署                             │
│    ├─ 注入 TRIGGER_PREVIEW_BRANCH 环境变量                       │
│    └─ 执行对应版本的代码                                         │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│                     分支归档阶段                                  │
├─────────────────────────────────────────────────────────────────┤
│  PR 合并 / 分支删除                                              │
│         ↓                                                        │
│  ArchiveBranchService.call()                                    │
│    ├─ 验证是分支环境 (parentEnvironmentId 非空)                  │
│    ├─ 设置 archivedAt = NOW()                                   │
│    └─ 修改 slug/shortcode 释放名称                               │
│         ↓                                                        │
│  后续请求无法再路由到该环境 (archivedAt != null)                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 关键数据流转表

| 阶段 | 数据项 | 存储位置 | 说明 |
|------|--------|----------|------|
| 分支创建 | branchName | RuntimeEnvironment.branchName | 预览分支环境的核心标识 |
| 分支创建 | 父子关系 | RuntimeEnvironment.parentEnvironmentId | 继承关系和 API Key 复用 |
| 部署创建 | commitRef | WorkerDeployment.git.commitRef | 完整 GitMeta 的一部分 |
| 部署创建 | commitSHA | WorkerDeployment.commitSHA | 单独索引字段 |
| 版本选择 | 当前部署标记 | WorkerDeploymentPromotion.label="current" | 每个环境唯一 |
| 运行时路由 | 分支名来源 | HTTP Header x-trigger-branch | 请求时指定目标分支 |
| 环境变量 | TRIGGER_PREVIEW_BRANCH | 内置变量（builtInVariables） | 优先级最高，动态注入 |
| 环境变量 | 用户自定义变量 | SecretStore (projectId:envId:varName) | 支持父子继承，子覆盖父 |
| 分支归档 | 归档标记 | RuntimeEnvironment.archivedAt | 非空则不再参与路由 |

### 4.3 核心设计模式与之前的理解修正

| 项 | 之前理解 | 修正后 |
|----|---------|--------|
| 环境变量优先级 | 用户变量 > 内置变量 | **内置变量 > 用户变量**（builtInVariables 最后加入，反向遍历优先级最高） |
| TRIGGER_PREVIEW_BRANCH 注入 | 独立注入步骤 | **作为 builtInVariables 的一部分**，在 resolveBuiltInProdVariables 中注入 |
| 分支环境 API Key | 每个分支有独立 Key | **复用父环境 API Key**，返回时覆盖 apiKey 字段 |
| 分支环境查找 | 直接查找 | **先找父环境，再查 childEnvironments**，通过 include 关联查询 |

### 4.4 关键代码路径速查

**分支环境映射**：
- 请求头提取: `apps/webapp/app/services/apiAuth.server.ts:278-280`
- API Key 分支路由: `apps/webapp/app/models/runtimeEnvironment.server.ts:94-166`
- PAT/OAT 分支路由: `apps/webapp/app/services/apiAuth.server.ts:440-614`

**环境变量合并**：
- 合并入口: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:899-933`
- 去重算法: `apps/webapp/app/v3/deduplicateVariableArray.server.ts:4-14`
- TRIGGER_PREVIEW_BRANCH 注入: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:1123-1130`

**分支生命周期**：
- 创建: `apps/webapp/app/services/upsertBranch.server.ts:10-159`
- 归档: `apps/webapp/app/services/archiveBranch.server.ts:6-88`
- 限制检查: `apps/webapp/app/services/upsertBranch.server.ts:161-190`
