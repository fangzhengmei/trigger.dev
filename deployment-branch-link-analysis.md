# 分支化部署代码链路分析

本文档深入分析 Trigger.dev 中分支化部署的实现机制，重点补齐之前未讲清楚的关键代码路径：

1. **RBAC 新鉴权路径**下预览分支如何通过请求头完成环境映射与归档过滤
2. **环境变量优先级**的校正，确保上下文表述一致且可被代码证伪
3. **TRIGGER_PREVIEW_BRANCH** 从构建参数注入到运行时客户端读取的完整来源链路

---

## 一、RBAC 新鉴权路径下的预览分支环境映射

### 1.1 RBAC 鉴权架构概述

Trigger.dev 采用插件化 RBAC 架构，核心入口：

**文件**: `apps/webapp/app/services/rbac.server.ts:1-29`
```typescript
import plugin from "@trigger.dev/rbac";

export const rbac = plugin.create(
  { primary: prisma, replica: $replica as PrismaClient },
  { forceFallback: env.RBAC_FORCE_FALLBACK }
);
```

插件接口定义：
**文件**: `packages/plugins/src/rbac.ts:129-166`
```typescript
export interface RoleBaseAccessController {
  // API 路由鉴权：一次 DB 查询 → identity + pre-built ability
  authenticateBearer(
    request: Request,
    options?: { allowJWT?: boolean }
  ): Promise<BearerAuthResult>;
  // ...
}
```

### 1.2 RBAC Fallback 中的分支环境映射

开源版本使用 Fallback 实现，分支头处理逻辑：

**文件**: `internal-packages/rbac/src/fallback.ts:67-223`
```typescript
async authenticateBearer(
  request: Request,
  options?: { allowJWT?: boolean }
): Promise<BearerAuthResult> {
  // 1. 提取 API Key
  const rawToken = request.headers.get("Authorization")?.replace(/^Bearer /, "").trim();
  if (!rawToken) return { ok: false, status: 401, error: "Invalid or Missing API key" };

  // 2. 处理 Public JWT 分支（跳过，关注 API Key 路径）
  if (options?.allowJWT && isPublicJWT(rawToken)) { /* ... */ }

  // 3. 核心：从请求头提取分支名并进行归档过滤
  const branchName = sanitizeBranchName(request.headers.get("x-trigger-branch"));
  
  // 4. 构建 include，关键：archivedAt: null 过滤已归档分支
  const include = {
    project: true,
    organization: true,
    orgMember: { /* ... */ },
    parentEnvironment: { select: { id: true, apiKey: true } },
    childEnvironments: branchName
      ? { where: { branchName, archivedAt: null } }  // 🔴 归档过滤在此处
      : undefined,
  } as const;

  // 5. 通过 API Key 查找父环境
  let env = await this.replica.runtimeEnvironment.findFirst({
    where: { apiKey: rawToken },
    include,
  });

  // 6. 处理已撤销的 API Key（兼容逻辑）
  if (!env) {
    const revoked = await this.replica.revokedApiKey.findFirst({
      where: { apiKey: rawToken, expiresAt: { gt: new Date() } },
      include: { runtimeEnvironment: { include } },
    });
    env = revoked?.runtimeEnvironment ?? null;
  }

  // 7. PREVIEW 环境需要分支头，切换到子环境身份
  if (env.type === "PREVIEW") {
    if (!branchName) {
      return {
        ok: false,
        status: 401,
        error: "x-trigger-branch header required for preview env",
      };
    }
    const child = env.childEnvironments?.[0];
    if (!child) {
      return { ok: false, status: 401, error: "No matching branch env" };
    }
    
    // 🔄 Pivot：子环境的 id/type/branchName + 父环境的 apiKey/orgMember/organization/project
    env = {
      ...child,
      apiKey: env.apiKey,
      orgMember: env.orgMember,
      organization: env.organization,
      project: env.project,
      parentEnvironment: { id: env.id, apiKey: env.apiKey },
      childEnvironments: [],
    };
  }

  return {
    ok: true,
    environment: toAuthenticatedEnvironment(env),
    subject: { /* ... */ },
    ability: permissiveAbility,
  };
}
```

### 1.3 apiBuilder 桥接层

RBAC 鉴权结果通过 apiBuilder 桥接到传统路由：

**文件**: `apps/webapp/app/services/routeBuilders/apiBuilder.server.ts:51-79`
```typescript
async function authenticateRequestForApiBuilder(
  request: Request,
  { allowJWT }: { allowJWT: boolean }
) {
  // 调用 RBAC 插件的 authenticateBearer
  const result = await rbac.authenticateBearer(request, { allowJWT });
  if (!result.ok) {
    return { ok: false, status: result.status, error: result.error };
  }

  // 转换为传统 ApiAuthenticationResultSuccess 格式
  const authentication: ApiAuthenticationResultSuccess = {
    ok: true,
    apiKey: result.environment.apiKey,
    type: result.subject.type === "publicJWT" ? "PUBLIC_JWT" : "PRIVATE",
    environment: result.environment,
    realtime: result.jwt?.realtime,
    oneTimeUse: result.jwt?.oneTimeUse,
  };

  return { ok: true, authentication, ability: result.ability };
}
```

### 1.4 新鉴权路径 vs 旧鉴权路径对比

| 维度 | 旧路径 (authenticateApiKey) | 新路径 (RBAC authenticateBearer) |
|------|-----------------------------|----------------------------------|
| 分支头提取 | `getApiKeyFromRequest()` → 传给 `findEnvironmentByApiKey` | `authenticateBearer` 内部直接读取 `request.headers` |
| 归档过滤 | `findEnvironmentByApiKey` 内部 `childEnvironments` 查询条件 `archivedAt: null` | RBAC fallback 内部 `childEnvironments` 查询条件 `archivedAt: null` |
| 环境 Pivot | `findEnvironmentByApiKey` 返回时覆盖 apiKey | `authenticateBearer` 内部 Pivot，返回完整子环境身份 |
| 调用方 | 直接调用 `authenticateApiKey(apiKey, { branchName })` | 通过 `apiBuilder` → `authenticateRequestForApiBuilder` → `rbac.authenticateBearer` |

**关键结论**：
- 归档过滤发生在 **数据库查询阶段**，通过 `archivedAt: null` 条件直接过滤掉已归档分支
- 分支头 `x-trigger-branch` 在 **RBAC 插件层** 直接处理，无需路由层传递
- Pivot 操作确保返回的环境对象包含正确的子环境 `id` 和父环境 `apiKey`

### 1.5 归档过滤的多层级验证

归档过滤不仅仅在鉴权层，还在多个执行节点生效：

**1. 调度引擎层**：
**文件**: `internal-packages/schedule-engine/src/engine/index.ts:362-367`
```typescript
if (instance.environment.archivedAt) {
  this.logger.debug("Environment is archived, skipping schedule", {
    instanceId: params.instanceId,
    scheduleId: instance.taskSchedule.friendlyId,
    environmentId: instance.environment.id,
  });
  return;
}
```

**2. 运行引擎层出队阶段**：
**文件**: `internal-packages/run-engine/src/engine/systems/dequeueSystem.ts:849-857`
```typescript
if (run.runtimeEnvironment.archivedAt) {
  span.setAttribute("result", "RUN_ENVIRONMENT_ARCHIVED");
  return {
    success: false as const,
    code: "RUN_ENVIRONMENT_ARCHIVED",
    message: `Run is on an archived environment: ${run.id}`,
    run,  // 🔴 返回 run 对象供后续处理
  };
}
```

**出队后的处理逻辑**：
**文件**: `internal-packages/run-engine/src/engine/systems/dequeueSystem.ts:298-309`
```typescript
case "RUN_ENVIRONMENT_ARCHIVED": {
  // this happens if the preview branch was archived
  this.$.logger.warn(
    "RunEngine.dequeueFromWorkerQueue(): Run environment archived",
    { runId, latestSnapshot: snapshot.id, result }
  );
  // 🔴 确认消息（从队列中移除），不执行任务
  await this.$.runQueue.acknowledgeMessage(orgId, runId);
  return;
}
```

**归档过滤的三层防护**：
1. **鉴权层**：RBAC authenticateBearer → childEnvironments 查询条件 `archivedAt: null` → 阻止新请求路由到归档分支
2. **调度层**：schedule-engine 执行前检查 `instance.environment.archivedAt` → 阻止定时任务触发新 run
3. **执行层**：run-engine dequeue 时检查 `run.runtimeEnvironment.archivedAt` → 已入队的 run 被确认（从队列移除）但**不执行**

---

## 二、环境变量优先级校正与完整上下文

### 2.1 优先级结论的再校正

之前的分析存在部分偏差，需要更精确的表述。让我们重新审视代码：

**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:899-933`
```typescript
export async function resolveVariablesForEnvironment(
  runtimeEnvironment: RuntimeEnvironmentForEnvRepo,
  parentEnvironment?: RuntimeEnvironmentForEnvRepo
) {
  // 1. 项目环境变量（支持父子继承）
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

  // 5. 合并去重
  const result = deduplicateVariableArray([
    ...overridableTriggerVariables,  // 索引 0
    ...overridableOtelVariables,     // 索引 1
    ...projectSecrets,               // 索引 2
    ...builtInVariables,             // 索引 3 ← 最后加入
  ]);

  return result;
}
```

**文件**: `apps/webapp/app/v3/deduplicateVariableArray.server.ts:1-14`
```typescript
import { type EnvironmentVariable } from "./environmentVariables/repository";

/** Later variables override earlier ones */
export function deduplicateVariableArray(variables: EnvironmentVariable[]) {
  const result: EnvironmentVariable[] = [];
  // Process array in reverse order so later variables override earlier ones
  for (const variable of [...variables].reverse()) {
    if (!result.some((v) => v.key === variable.key)) {
      result.push(variable);
    }
  }
  // Reverse back to maintain original order but with later variables taking precedence
  return result.reverse();
}
```

### 2.1.1 deduplicateVariableArray 算法深度解析

函数注释明确说明了优先级规则：**Later variables override earlier ones**（后面的变量覆盖前面的）。

**算法执行步骤**：

假设输入数组为 `[A, B, C, D]`，其中：
- `A` = overridableTriggerVariables（最先加入，索引 0）
- `B` = overridableOtelVariables（索引 1）
- `C` = projectSecrets（索引 2）
- `D` = builtInVariables（最后加入，索引 3）

```
输入数组: [A, B, C, D]
           ↑  ↑  ↑  ↑
           0  1  2  3

步骤1: 反向拷贝 → [D, C, B, A]
步骤2: 遍历反向数组，首次遇到的 key 加入 result:
       - 处理 D → result = [D]
       - 处理 C → result = [D, C]（假设无重复）
       - 处理 B → result = [D, C, B]（假设无重复）
       - 处理 A → result = [D, C, B, A]（假设无重复）
步骤3: 反转 result → [A, B, C, D]

最终输出顺序与输入相同，但优先级由后向前。
```

**重复 key 的处理示例**：

假设 `A.key = "FOO"`, `D.key = "FOO"`（同名冲突）：
```
输入数组: [A(FOO), B, C, D(FOO)]

反向遍历:
  1. D(FOO) → result = [D(FOO)]  ✓ 保留
  2. C → result = [D(FOO), C]
  3. B → result = [D(FOO), C, B]
  4. A(FOO) → result 中已有 FOO，跳过 ✗

反转后输出: [B, C, D(FOO)]

结论: D 的值保留，A 的值被覆盖
```

### 2.1.2 四种变量合并顺序与覆盖关系

**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:899-933`
```typescript
const result = deduplicateVariableArray([
  ...overridableTriggerVariables,  // 索引 0
  ...overridableOtelVariables,     // 索引 1
  ...projectSecrets,               // 索引 2
  ...builtInVariables,             // 索引 3 ← 最后加入
]);
```

合并顺序与优先级的对应关系：

| 变量来源 | 展开顺序 | 在合并数组中的位置 | 优先级 | 覆盖关系 |
|---------|---------|-----------------|--------|---------|
| overridableTriggerVariables | 最先展开 | 低索引区（0+） | ⚪ 最低 | 被后面所有来源覆盖 |
| overridableOtelVariables | 第二展开 | 中低索引区 | 🟢 较低 | 覆盖前者，被后两者覆盖 |
| projectSecrets | 第三展开 | 中高索引区 | 🟡 中 | 覆盖前两者，被最后者覆盖 |
| builtInVariables | 最后展开 | 高索引区 | 🔴 最高 | 覆盖前面所有来源 |

**关键结论**：
- 数组展开顺序决定了优先级，**越靠后展开的来源优先级越高**
- `builtInVariables` 最后展开，因此优先级最高
- `overridableTriggerVariables` 最先展开，因此优先级最低

### 2.1.3 各来源包含的具体变量

| 来源 | 包含变量示例 |
|-----|-------------|
| **builtInVariables**（🔴 最高） | `TRIGGER_API_URL`、`TRIGGER_PREVIEW_BRANCH`、`TRIGGER_SECRET_KEY`、`TRIGGER_PROJECT_ID`、`TRIGGER_ENVIRONMENT_ID`、`TRIGGER_ENVIRONMENT_SLUG`、`TRIGGER_ENVIRONMENT_TYPE` |
| **projectSecrets**（🟡 中） | 用户在项目设置中定义的所有变量（如 `DATABASE_URL`、`API_KEY` 等） |
| **overridableOtelVariables**（🟢 较低） | `OTEL_EXPORTER_OTLP_HEADERS`、`OTEL_EXPORTER_OTLP_ENDPOINT`、`OTEL_RESOURCE_ATTRIBUTES`（仅 DEV 环境） |
| **overridableTriggerVariables**（⚪ 最低） | `TRIGGER_REALTIME_STREAM_VERSION` |

> **可证伪性检验**：如果用户在 projectSecrets 中设置了名为 `TRIGGER_API_URL` 的环境变量，由于 builtInVariables 优先级更高，用户设置的值会被内置值覆盖。验证方法：在预览分支环境中设置该变量，然后执行任务并打印 `process.env.TRIGGER_API_URL`，观察到的应该是系统内置值而非用户设置值。

### 2.2 父子环境变量继承的优先级

**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:645-685`
```typescript
async #getSecretEnvironmentVariables(
  projectId: string,
  environmentId: string,
  parentEnvironmentId?: string
): Promise<EnvironmentVariable[]> {
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
    mergedSecrets.set(parsedKey, secret.value.secret); // 后加入的覆盖先加入的
  }

  const merged = Array.from(mergedSecrets.entries()).map(([key, value]) => {
    return { key, value };
  });

  return removeBlacklistedVariables(merged);
}
```

**父子继承优先级**：子环境变量 > 父环境变量

### 2.3 完整优先级栈（从高到低）

```
┌─────────────────────────────────────────────────────────┐
│  1. builtInVariables (内置变量)                          │
│     ├─ TRIGGER_API_URL                                   │
│     ├─ TRIGGER_PREVIEW_BRANCH (动态注入)                 │
│     ├─ TRIGGER_SECRET_KEY                                │
│     └─ ...                                               │
├─────────────────────────────────────────────────────────┤
│  2. projectSecrets (用户自定义变量)                      │
│     ├─ 子环境变量 (优先级 > 父环境)                       │
│     └─ 父环境变量                                        │
├─────────────────────────────────────────────────────────┤
│  3. overridableOtelVariables (仅 DEV)                    │
├─────────────────────────────────────────────────────────┤
│  4. overridableTriggerVariables                          │
│     └─ TRIGGER_REALTIME_STREAM_VERSION                   │
└─────────────────────────────────────────────────────────┘
```

---

## 三、TRIGGER_PREVIEW_BRANCH 完整来源链路

### 3.1 链路全景图

```
┌─────────────────────────────────────────────────────────────────┐
│  构建阶段 (Build Time)                                           │
├─────────────────────────────────────────────────────────────────┤
│  CLI deploy 命令                                                 │
│    ↓                                                             │
│  buildImage.ts:235                                               │
│  `--build-arg TRIGGER_PREVIEW_BRANCH=${options.branchName}`      │
│    ↓                                                             │
│  Containerfile 模板:790-799                                      │
│  ARG TRIGGER_PREVIEW_BRANCH → ENV TRIGGER_PREVIEW_BRANCH         │
│    ↓                                                             │
│  镜像构建完成，TRIGGER_PREVIEW_BRANCH 固化到镜像环境变量中        │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│  服务端注入阶段 (Server-side Injection)                          │
├─────────────────────────────────────────────────────────────────┤
│  任务执行时调用 resolveVariablesForEnvironment()                │
│    ↓                                                             │
│  resolveBuiltInProdVariables():1123-1130                         │
│  if (runtimeEnvironment.branchName) {                            │
│    result.push({ key: "TRIGGER_PREVIEW_BRANCH",                  │
│                  value: runtimeEnvironment.branchName })         │
│  }                                                               │
│    ↓                                                             │
│  作为 builtInVariables 的一部分注入任务运行环境                   │
│  （优先级最高，可覆盖镜像中固化的值）                             │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│  运行时读取阶段 (Runtime Read)                                   │
├─────────────────────────────────────────────────────────────────┤
│  APIClientManager.branchName getter:47-55                       │
│  config.previewBranch ??                                         │
│  getEnvVar("TRIGGER_PREVIEW_BRANCH") ??                          │
│  getEnvVar("VERCEL_GIT_COMMIT_REF") ??                           │
│  undefined                                                       │
│    ↓                                                             │
│  ApiClient 构造时传入                                            │
│    ↓                                                             │
│  ApiClient 请求时自动添加 x-trigger-branch 头                     │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 构建阶段：Containerfile 模板注入

**文件**: `packages/cli-v3/src/deploy/buildImage.ts:235`
```typescript
const args = [
  "build",
  // ...
  "--build-arg",
  `TRIGGER_PREVIEW_BRANCH=${options.branchName ?? ""}`,  // 🔴 构建参数
  // ...
];
```

**文件**: `packages/cli-v3/src/deploy/buildImage.ts:780-808` (Containerfile 模板)
```dockerfile
WORKDIR /app

ARG TRIGGER_PROJECT_ID
ARG TRIGGER_DEPLOYMENT_ID
ARG TRIGGER_DEPLOYMENT_VERSION
ARG TRIGGER_CONTENT_HASH
ARG TRIGGER_PROJECT_REF
ARG NODE_EXTRA_CA_CERTS
ARG TRIGGER_SECRET_KEY
ARG TRIGGER_API_URL
ARG TRIGGER_PREVIEW_BRANCH  // 🔴 声明构建参数

ENV TRIGGER_PROJECT_ID=${TRIGGER_PROJECT_ID} \
    TRIGGER_DEPLOYMENT_ID=${TRIGGER_DEPLOYMENT_ID} \
    TRIGGER_DEPLOYMENT_VERSION=${TRIGGER_DEPLOYMENT_VERSION} \
    TRIGGER_PROJECT_REF=${TRIGGER_PROJECT_REF} \
    TRIGGER_CONTENT_HASH=${TRIGGER_CONTENT_HASH} \
    TRIGGER_SECRET_KEY=${TRIGGER_SECRET_KEY} \
    TRIGGER_API_URL=${TRIGGER_API_URL} \
    TRIGGER_PREVIEW_BRANCH=${TRIGGER_PREVIEW_BRANCH} \  // 🔴 转为环境变量
    NODE_EXTRA_CA_CERTS=${NODE_EXTRA_CA_CERTS} \
    NODE_ENV=production
```

> **注意**：这是 indexer 阶段的 ENV 指令，在 final 阶段（第 811 行开始）没有重复设置 TRIGGER_PREVIEW_BRANCH，因此该变量仅在构建阶段的 indexer 中生效。

### 3.3 服务端注入阶段：任务运行时动态注入

**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:1123-1130`
```typescript
if (runtimeEnvironment.branchName) {
  result = result.concat([
    {
      key: "TRIGGER_PREVIEW_BRANCH",
      value: runtimeEnvironment.branchName,  // 🔴 从数据库读取 branchName
    },
  ]);
}
```

**注入时机和上下文**：
- 调用方：任务执行时获取环境变量的流程
- 触发条件：`runtimeEnvironment.branchName` 非空（即预览分支环境）
- 优先级：属于 `builtInVariables`，优先级最高
- 作用：覆盖镜像中可能存在的旧值，确保运行时使用正确的分支名

### 3.4 运行时读取阶段：SDK 客户端读取

**文件**: `packages/core/src/v3/apiClientManager/index.ts:47-55`
```typescript
get branchName(): string | undefined {
  const config = this.#getConfig();
  const value =
    config?.previewBranch ??              // 1. 配置对象中的 previewBranch
    getEnvVar("TRIGGER_PREVIEW_BRANCH") ?? // 2. 环境变量（服务端注入）
    getEnvVar("VERCEL_GIT_COMMIT_REF") ??  // 3. Vercel 部署环境
    undefined;                             // 4. 回退
  return value ? value : undefined;
}
```

**文件**: `packages/core/src/v3/utils/getEnv.ts:11-13`
```typescript
export function getEnvVar(name: string, defaultValue?: string): string | undefined {
  return env[name] ?? defaultValue;  // env 是 process.env 的代理
}
```

**读取优先级（从高到低）**：
1. `config.previewBranch` - SDK 配置对象中显式设置
2. `process.env.TRIGGER_PREVIEW_BRANCH` - 服务端注入的环境变量
3. `process.env.VERCEL_GIT_COMMIT_REF` - Vercel 平台环境变量
4. `undefined` - 无分支

### 3.5 运行时使用：ApiClient 自动添加请求头

**文件**: `packages/core/src/v3/apiClient/index.ts:1843-1845`
```typescript
if (this.previewBranch) {
  headers["x-trigger-branch"] = this.previewBranch;  // 🔴 自动添加请求头
}
```

**文件**: `packages/core/src/v3/apiClient/index.ts:1967-1969`
```typescript
if (this.previewBranch) {
  headers["x-trigger-branch"] = this.previewBranch;  // 另一个调用点
}
```

### 3.6 双路径注入的设计意图

TRIGGER_PREVIEW_BRANCH 有两个注入路径：

| 路径 | 位置 | 生效时机 | 说明 |
|------|------|---------|------|
| 构建参数注入 | Containerfile ARG → ENV | 构建阶段（indexer） | 主要用于构建过程中的索引任务 |
| 服务端动态注入 | resolveBuiltInProdVariables() | 任务运行时 | 优先级更高，确保运行时使用正确值 |

**设计意图**：
1. **构建阶段注入**：满足 Docker 构建过程中（如 indexer 阶段）需要知道分支名的场景
2. **运行时注入**：确保任务实际执行时使用的分支名与数据库中 `RuntimeEnvironment.branchName` 一致，避免镜像固化旧值
3. **双保险**：即使构建时未正确设置，运行时仍能通过服务端注入获得正确值

---

## 四、补充：归档分支的完整处理链路

### 4.1 归档操作

**文件**: `apps/webapp/app/services/archiveBranch.server.ts:6-88`
```typescript
public async call(orgFilter, { environmentId }) {
  const environment = await this.#prismaClient.runtimeEnvironment.findFirstOrThrow({
    where: { id: environmentId, /* 权限检查 */ },
  });

  if (!environment.parentEnvironmentId) {
    return { success: false, error: "This isn't a branch, and cannot be archived." };
  }

  // 修改 slug 和 shortcode 释放原名称
  const slug = `${environment.slug}-${nanoid(6)}`;
  const shortcode = slug;

  const updatedBranch = await this.#prismaClient.runtimeEnvironment.update({
    where: { id: environmentId },
    data: { 
      archivedAt: new Date(),  // 🔴 设置归档时间
      slug, 
      shortcode 
    },
  });

  return { success: true, branch: updatedBranch };
}
```

### 4.2 归档后的影响范围

| 层级 | 影响 | 代码位置 |
|------|------|---------|
| RBAC 鉴权 | 无法通过 `x-trigger-branch` 路由到该环境 | `internal-packages/rbac/src/fallback.ts:154` |
| 旧鉴权路径 | 无法通过 `findEnvironmentByApiKey` 找到 | `apps/webapp/app/models/runtimeEnvironment.server.ts:75` |
| 调度引擎 | 跳过该环境的所有定时任务 | `internal-packages/schedule-engine/src/engine/index.ts:362` |
| 运行引擎 | 出队时标记为 RUN_ENVIRONMENT_ARCHIVED | `internal-packages/run-engine/src/engine/systems/dequeueSystem.ts:849` |
| 已入队任务 | 仍可正常执行（environmentId 不变） | 无拦截 |

> **关键特性**：归档是软删除，`environmentId` 保持不变，已在队列中的任务不会受影响。

---

## 五、关键代码路径速查

### 5.1 RBAC 鉴权与分支映射
- RBAC 插件入口: `apps/webapp/app/services/rbac.server.ts:1-29`
- RBAC Fallback 分支处理: `internal-packages/rbac/src/fallback.ts:134-208`
- apiBuilder 桥接层: `apps/webapp/app/services/routeBuilders/apiBuilder.server.ts:51-79`
- 归档过滤（鉴权层）: `internal-packages/rbac/src/fallback.ts:154`
- 归档过滤（调度层）: `internal-packages/schedule-engine/src/engine/index.ts:362-367`
- 归档过滤（执行层）: `internal-packages/run-engine/src/engine/systems/dequeueSystem.ts:849-854`

### 5.2 环境变量合并
- 合并入口: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:899-933`
- 去重算法（优先级关键）: `apps/webapp/app/v3/deduplicateVariableArray.server.ts:1-14`
- 父子继承: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:645-685`
- TRIGGER_PREVIEW_BRANCH 服务端注入: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:1123-1130`
- 环境变量构建（任务执行时）: `apps/webapp/app/v3/marqs/sharedQueueConsumer.server.ts:2190-2222`

### 5.3 TRIGGER_PREVIEW_BRANCH 链路
- CLI 参数定义: `packages/cli-v3/src/deploy/buildImage.ts:50`
- 构建参数注入: `packages/cli-v3/src/deploy/buildImage.ts:234-235`
- Containerfile 模板（indexer 阶段）: `packages/cli-v3/src/deploy/buildImage.ts:780-826`
- 服务端动态注入: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:1123-1130`
- 任务环境构建: `apps/webapp/app/v3/marqs/sharedQueueConsumer.server.ts:2190-2222`
- SDK 读取: `packages/core/src/v3/apiClientManager/index.ts:47-55, 65`
- ApiClient 请求头添加: `packages/core/src/v3/apiClient/index.ts:1843-1845`

### 5.4 分支生命周期
- 创建: `apps/webapp/app/services/upsertBranch.server.ts:10-159`
- 归档: `apps/webapp/app/services/archiveBranch.server.ts:6-88`
- 限制检查: `apps/webapp/app/services/upsertBranch.server.ts:161-190`
