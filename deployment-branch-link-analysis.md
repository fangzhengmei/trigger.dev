# 分支化部署代码链路分析

本文档分析 Trigger.dev 中分支化部署的实现机制，包括三个核心环节：部署创建时与 Git 分支绑定、运行时选中对应分支代码版本、不同分支的环境变量与设置切换。

---

## 一、部署创建时与 Git 分支绑定

### 1.1 Git 元数据模型

**文件**: `packages/core/src/v3/schemas/common.ts:247-263`

```typescript
export const GitMeta = z.object({
  provider: z.string().optional(),
  source: z.enum(["trigger_github_app", "github_actions", "local"]).optional(),
  ghUsername: z.string().optional(),
  ghUserAvatarUrl: z.string().optional(),
  commitAuthorName: z.string().optional(),
  commitMessage: z.string().optional(),
  commitRef: z.string().optional(),      // 分支名
  commitSha: z.string().optional(),      // 提交哈希
  dirty: z.boolean().optional(),
  remoteUrl: z.string().optional(),
  pullRequestNumber: z.number().optional(),
  pullRequestTitle: z.string().optional(),
  pullRequestState: z.enum(["open", "closed", "merged"]).optional(),
});
```

### 1.2 Git 元数据收集（CLI 端）

**文件**: `packages/cli-v3/src/utilities/gitMeta.ts`

`createGitMeta()` 函数负责从不同来源收集 Git 信息：

1. **Trigger GitHub App 环境**：从环境变量读取（`TRIGGER_GITHUB_APP === "true"`）
   - `GITHUB_REPOSITORY_URL`、`GITHUB_HEAD_COMMIT_SHA`、`GITHUB_REF` 等

2. **GitHub Actions 环境**：从 CI 环境变量读取
   - `GITHUB_SHA`、`GITHUB_REF`、`GITHUB_EVENT_PATH` 等
   - 支持 push 和 pull_request 事件类型

3. **本地开发环境**：通过 git 命令获取
   - 使用 `git-last-commit` 库读取最后一次提交
   - `git status -s` 检查工作区是否 dirty

### 1.3 部署初始化与分支绑定

**文件**: `apps/webapp/app/v3/services/initializeDeployment.server.ts:196-261`

在 `InitializeDeploymentService.call()` 中，通过 `createDeploymentWithNextVersion()` 创建部署：

```typescript
const deployment = await createDeploymentWithNextVersion(
  this._prisma,
  environment.id,
  async (nextVersion) => {
    // ...
    return {
      friendlyId: generateFriendlyId("deployment"),
      contentHash: payload.contentHash,
      shortCode: deploymentShortCode,
      status: initialStatus,
      projectId: environment.projectId,
      git: payload.gitMeta ?? undefined,           // 存储完整 GitMeta
      commitSHA: payload.gitMeta?.commitSha ?? undefined,  // 单独存储 commitSHA 用于索引
      runtime: payload.runtime ?? undefined,
      // ...
    };
  }
);
```

### 1.4 数据库存储模型

**文件**: `internal-packages/database/prisma/schema.prisma:2106-2165`

```prisma
model WorkerDeployment {
  id          String  @id @default(cuid())
  contentHash String
  friendlyId  String  @unique
  shortCode   String
  version     String
  commitSHA   String?  // 单独字段，便于索引查询
  git         Json?    // 完整 GitMeta 对象，包含 commitRef（分支名）
  // ...
  @@index([commitSHA])
}
```

**关键设计点**：
- `git` 字段存储完整的 `GitMeta` JSON，包含分支名 (`commitRef`)
- `commitSHA` 单独字段用于数据库索引，支持按提交哈希快速查询
- `@@unique([environmentId, version])` 确保同一环境下版本号唯一

---

## 二、运行时选中对应分支代码版本

### 2.1 部署晋升（Promotion）机制

**文件**: `apps/webapp/app/v3/services/changeCurrentDeployment.server.ts`

`ChangeCurrentDeploymentService` 负责切换当前活跃部署：

```typescript
// 将部署标记为当前环境的活跃版本
await this._prisma.workerDeploymentPromotion.upsert({
  where: {
    environmentId_label: {
      environmentId: deployment.environmentId,
      label: CURRENT_DEPLOYMENT_LABEL,  // "current"
    },
  },
  create: {
    deploymentId: deployment.id,
    environmentId: deployment.environmentId,
    label: CURRENT_DEPLOYMENT_LABEL,
  },
  update: {
    deploymentId: deployment.id,
  },
});
```

**文件**: `packages/core/src/v3/isomorphic/consts.ts`
```typescript
export const CURRENT_DEPLOYMENT_LABEL = "current";
```

### 2.2 部署晋升数据模型

**文件**: `internal-packages/database/prisma/schema.prisma:2182-2196`

```prisma
model WorkerDeploymentPromotion {
  id             String           @id @default(cuid())
  label          String           // "current"
  deployment     WorkerDeployment @relation(fields: [deploymentId], references: [id])
  deploymentId   String
  environment    RuntimeEnvironment @relation(fields: [environmentId], references: [id])
  environmentId  String

  @@unique([environmentId, label])  // 每个环境只有一个当前部署
}
```

### 2.3 运行时任务执行与版本选择

**任务执行流程**：

1. 任务入队时，关联到特定的 `RuntimeEnvironment`
2. 执行时通过 `environmentId` 查找 `WorkerDeploymentPromotion` 中 `label = "current"` 的记录
3. 获取对应的 `WorkerDeployment`，包含该版本的代码镜像 (`imageReference`) 和运行时信息

**关键关联**：
- `TaskRun` → `RuntimeEnvironment` → `WorkerDeploymentPromotion` (label="current") → `WorkerDeployment`
- 每个部署版本有独立的 Docker 镜像 (`imageReference`)，包含对应分支的代码

### 2.4 版本比较与方向控制

**文件**: `apps/webapp/app/v3/services/changeCurrentDeployment.server.ts:69-94`

```typescript
if (!disableVersionCheck) {
  switch (direction) {
    case "promote": {
      // 只能晋升比当前版本更新的部署
      if (compareDeploymentVersions(currentPromotion.deployment.version, deployment.version) >= 0) {
        throw new ServiceValidationError("Cannot promote a deployment that is older than the current deployment.");
      }
      break;
    }
    case "rollback": {
      // 只能回滚到比当前版本更旧的部署
      if (compareDeploymentVersions(currentPromotion.deployment.version, deployment.version) <= 0) {
        throw new ServiceValidationError("Cannot rollback to a deployment that is newer than the current deployment.");
      }
      break;
    }
  }
}
```

---

## 三、不同分支的环境变量与设置切换

### 3.1 分支环境（Preview Branches）模型

**文件**: `internal-packages/database/prisma/schema.prisma:324-330`

```prisma
model RuntimeEnvironment {
  // ...
  isBranchableEnvironment Boolean              @default(false)
  branchName              String?
  parentEnvironment       RuntimeEnvironment?  @relation("parentEnvironment", fields: [parentEnvironmentId], references: [id])
  parentEnvironmentId     String?
  childEnvironments       RuntimeEnvironment[] @relation("parentEnvironment")
  git                     Json?                // 环境级别的 GitMeta
  // ...
}
```

**设计要点**：
- `isBranchableEnvironment`: 标记该环境是否支持分支部署
- `branchName`: 关联的 Git 分支名
- `parentEnvironmentId`: 支持父子环境继承关系
- 子环境（预览分支）继承父环境（如 staging）的配置，但可以覆盖

### 3.2 环境变量存储结构

**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts`

环境变量按 `projectId + environmentId + key` 三级存储：

```typescript
function secretKey(projectId: string, environmentId: string, key: string) {
  return `environmentvariable:${projectId}:${environmentId}:${key}`;
}
```

### 3.3 环境变量解析与继承

**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:899-933`

`resolveVariablesForEnvironment()` 函数负责合并环境变量：

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

  // 2. 可覆盖的 Trigger 内置变量
  const overridableTriggerVariables = await resolveOverridableTriggerVariables(runtimeEnvironment);

  // 3. 内置环境变量（DEV 或 PROD）
  const builtInVariables = runtimeEnvironment.type === "DEVELOPMENT"
    ? await resolveBuiltInDevVariables(runtimeEnvironment)
    : await resolveBuiltInProdVariables(runtimeEnvironment, parentEnvironment);

  // 4. 合并去重（优先级：用户变量 > 内置变量）
  const result = deduplicateVariableArray([
    ...overridableTriggerVariables,
    ...overridableOtelVariables,
    ...projectSecrets,
    ...builtInVariables,
  ]);

  return result;
}
```

### 3.4 父子环境变量继承逻辑

**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:645-685`

```typescript
async #getSecretEnvironmentVariables(
  projectId: string,
  environmentId: string,
  parentEnvironmentId?: string
) {
  const parentSecrets = parentEnvironmentId
    ? await secretStore.getSecrets(SecretValue, secretKeyEnvironmentPrefix(projectId, parentEnvironmentId))
    : [];

  const childSecrets = await secretStore.getSecrets(SecretValue, secretKeyEnvironmentPrefix(projectId, environmentId));

  // 子环境变量覆盖父环境
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

### 3.5 预览分支特殊环境变量

**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:1123-1130`

当环境关联到分支时，自动注入 `TRIGGER_PREVIEW_BRANCH`：

```typescript
if (runtimeEnvironment.branchName) {
  result = result.concat([
    {
      key: "TRIGGER_PREVIEW_BRANCH",
      value: runtimeEnvironment.branchName,
    },
  ]);
}
```

### 3.6 内置环境变量覆盖

**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:1336-1360`

支持通过 `builtInEnvironmentVariableOverrides` 字段覆盖内置变量：

```typescript
function resolveBuiltInEnvironmentVariableOverrides(
  key: string,
  runtimeEnvironment: RuntimeEnvironmentForEnvRepo,
  defaultValue: string
) {
  const overrides = runtimeEnvironment.builtInEnvironmentVariableOverrides;
  if (!overrides) return defaultValue;
  if (typeof overrides === "object" && key in overrides) {
    const value = (overrides as Record<string, unknown>)[key];
    if (typeof value === "string") return value;
  }
  return defaultValue;
}
```

---

## 四、完整链路串联

### 4.1 分支化部署全流程

```
Git 分支代码
    ↓
[CLI] createGitMeta() 收集分支信息
    ↓
[API] POST /api/v1/deployments
    ↓
[InitializeDeploymentService] 创建 WorkerDeployment
    │  ├─ 存储 git (GitMeta JSON) 包含 commitRef
    │  ├─ 存储 commitSHA 单独字段
    │  └─ 关联到 RuntimeEnvironment
    ↓
[构建] 基于该版本代码构建 Docker 镜像
    ↓
[ChangeCurrentDeploymentService] 晋升为 CURRENT
    │  └─ 创建 WorkerDeploymentPromotion (label="current")
    ↓
[运行时] 任务执行
    │  ├─ 通过 environmentId 查找 current promotion
    │  ├─ 获取对应 WorkerDeployment
    │  └─ 使用该版本的 imageReference 启动容器
    ↓
[环境变量注入]
    ├─ 从 RuntimeEnvironment 读取 branchName
    ├─ 解析父子环境继承的变量
    ├─ 注入 TRIGGER_PREVIEW_BRANCH
    └─ 合并内置变量与用户变量
```

### 4.2 关键数据流转表

| 阶段 | 数据项 | 存储位置 | 说明 |
|------|--------|----------|------|
| 部署创建 | commitRef (分支名) | WorkerDeployment.git.commitRef | 完整 GitMeta 的一部分 |
| 部署创建 | commitSHA | WorkerDeployment.commitSHA | 单独字段，支持索引 |
| 部署创建 | 分支名 (环境级) | RuntimeEnvironment.branchName | 预览分支环境的分支名 |
| 版本选择 | 当前部署标记 | WorkerDeploymentPromotion.label="current" | 每个环境唯一 |
| 环境变量 | 用户自定义变量 | SecretStore (key=projectId:envId:varName) | 支持父子继承 |
| 环境变量 | 分支标识 | TRIGGER_PREVIEW_BRANCH 环境变量 | 运行时自动注入 |

### 4.3 核心设计模式

1. **标签晋升模式**：通过 `WorkerDeploymentPromotion` 的 label 机制实现版本切换，无需修改部署记录本身
2. **父子环境继承**：预览分支环境继承父环境配置，减少重复配置
3. **可变与不可变分离**：部署元数据（Git 信息、镜像引用）不可变，活跃版本通过关联表动态指向
4. **三级密钥存储**：项目 → 环境 → 变量名的层级结构，支持细粒度权限和隔离
