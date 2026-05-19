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
**文件**: `internal-packages/run-engine/src/engine/systems/dequeueSystem.ts:849-854`
```typescript
if (run.runtimeEnvironment.archivedAt) {
  span.setAttribute("result", "RUN_ENVIRONMENT_ARCHIVED");
  return {
    success: false as const,
    code: "RUN_ENVIRONMENT_ARCHIVED",
    message: `Run is on an archived environment: ${run.id}`,
  };
}
```

**归档过滤的三层防护**：
1. **鉴权层**：RBAC authenticateBearer → childEnvironments 查询条件 `archivedAt: null`
2. **调度层**：schedule-engine 执行前检查 `instance.environment.archivedAt`
3. **执行层**：run-engine dequeue 时检查 `run.runtimeEnvironment.archivedAt`

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

  // 2. 可