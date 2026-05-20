# API 凭证签发、作用域校验与项目隔离协作链路

## 一、核心概念分层

### 1.1 凭证类型体系

| 前缀 | 类型 | 作用域 | 说明 |
|------|------|--------|------|
| `tr_` | 私有 API Key | 环境级 | 服务端使用，完整权限 |
| `pk_` | 公开 API Key | 环境级 | 已废弃，旧版客户端使用 |
| `tr_pat_` | 个人访问令牌 | 用户级 | 跨项目，以用户身份访问 |
| JWT | 公开访问令牌 | 资源级 | 带 scopes 声明，细粒度控制 |

### 1.2 环境与项目的层级关系

```
Organization (组织)
  └── Project (项目)  ← 外部凭证通过 projectId/externalRef 绑定
        └── RuntimeEnvironment (运行环境)
              ├── DEVELOPMENT (dev)  - 个人开发环境，绑定 orgMemberId
              ├── STAGING (stg)      - 预发布
              ├── PRODUCTION (prod)  - 生产
              └── PREVIEW (preview)  - 预览分支，通过 branchName 关联
```

**关键关联字段**（`AuthenticatedEnvironment` 类型）：
- `projectId` / `project.externalRef`：项目唯一标识
- `organizationId`：组织归属
- `apiKey` / `pkApiKey`：环境级凭证
- `orgMemberId`：开发环境绑定的成员（个人隔离）
- `parentEnvironmentId`：预览环境的父环境（继承凭证）

### 1.3 认证与授权的关键差异

| 维度 | 认证 (Authentication) | 授权 (Authorization) |
|------|----------------------|----------------------|
| 核心问题 | "你是谁？" | "你能做什么？" |
| 验证目标 | 凭证有效性 + 身份归属 | 权限范围 + 资源访问许可 |
| 典型操作 | API Key 匹配、JWT 签名校验、PAT 哈希比对 | Scope 匹配、RBAC 角色检查、资源归属校验 |
| 失败状态码 | 401 Unauthorized | 403 Forbidden |
| 执行时机 | 请求入口，最先执行 | 认证通过后，资源访问前 |
| 数据来源 | 请求头 (Authorization) + 数据库 | 认证结果 + JWT claims + 路由配置 |
| 项目隔离方式 | Environment → Project 外键绑定 | projectRef 参数校验 + 作用域限制 |

**关键区分**：
- 认证只验证"凭证是否有效"和"属于哪个环境/用户"
- 授权验证"该凭证是否有权访问特定资源"
- API Key 认证通过 ≠ 可以访问所有资源（JWT scopes 会进一步限制）

## 二、凭证签发机制

### 2.1 API Key 生成规则

**文件**：`apps/webapp/app/models/api-key.server.ts:93-118`

```typescript
export function createApiKeyForEnv(envType: RuntimeEnvironment["type"]) {
  return `tr_${envSlug(envType)}_${apiKeyId(20)}`;  // tr_dev_xxx, tr_prod_xxx
}

export function createPkApiKeyForEnv(envType: RuntimeEnvironment["type"]) {
  return `pk_${envSlug(envType)}_${apiKeyId(20)}`;  // pk_dev_xxx, pk_prod_xxx
}
```

- 前缀 `tr_` = 私钥，`pk_` = 公钥（已废弃）
- 中间段 `envSlug` = dev/stg/prod/preview，标识环境类型
- 后缀 20 位随机字符（大小写字母+数字）
- **每环境独立签发**，凭证天然绑定到特定环境 → 间接绑定到项目

### 2.2 重新签发与吊销窗口

**文件**：`apps/webapp/app/models/api-key.server.ts:18-91`

```typescript
const REVOKED_API_KEY_GRACE_PERIOD_MS = 24 * 60 * 60 * 1000; // 24小时

export async function regenerateApiKey({ userId, environmentId }: RegenerateAPIKeyInput) {
  // 1. 权限检查：用户必须属于该组织
  // 2. 开发环境额外检查：必须是该环境的绑定成员
  // 3. 生成新密钥对
  const newApiKey = createApiKeyForEnv(environment.type);
  const newPkApiKey = createPkApiKeyForEnv(environment.type);
  // 4. 旧密钥写入 revokedApiKey 表，设置 24 小时宽限期
  // 5. 原子更新 runtimeEnvironment.apiKey
}
```

**项目隔离点**：
- 开发环境（DEVELOPMENT）通过 `orgMemberId` 绑定到具体用户，实现用户级隔离
- 非开发环境（STAGING/PRODUCTION）组织内所有成员可见
- 重新签发前必须校验用户在该组织中的成员身份

## 三、认证流程与环境解析

### 3.1 认证入口：API Key → 环境映射

**文件**：`apps/webapp/app/models/runtimeEnvironment.server.ts:94-166`

```typescript
export async function findEnvironmentByApiKey(
  apiKey: string,
  branchName: string | undefined
): Promise<AuthenticatedEnvironment | null> {
  // 1. 主查询：apiKey 精确匹配
  let environment = await $replica.runtimeEnvironment.findFirst({
    where: { apiKey },
    include: { project: true, organization: true, ... }
  });

  // 2. 降级查询：吊销表中仍在宽限期的密钥
  if (!environment) {
    const revokedApiKey = await $replica.revokedApiKey.findFirst({
      where: { apiKey, expiresAt: { gt: new Date() } },
      include: { runtimeEnvironment: { include } },
    });
    environment = revokedApiKey?.runtimeEnvironment ?? null;
  }

  // 3. 项目有效性检查：软删除的项目拒绝访问
  if (environment.project.deletedAt !== null) return null;

  // 4. 预览环境分支路由：通过 x-trigger-branch 头定位子环境
  if (environment.type === "PREVIEW") {
    const childEnvironment = environment.childEnvironments.at(0);
    if (childEnvironment) {
      // 子环境继承父环境的 apiKey、orgMember、organization、project
      return toAuthenticated({
        ...childEnvironment,
        apiKey: environment.apiKey,
        orgMember: environment.orgMember,
        organization: environment.organization,
        project: environment.project,
      });
    }
  }

  return toAuthenticated(environment);
}
```

**项目隔离核心**：
- API Key 是 `runtimeEnvironment` 表的唯一键，通过环境 → 项目外键关联实现项目绑定
- 已删除的项目（`project.deletedAt !== null`）直接返回 null，即使密钥有效也无法访问
- 预览环境通过 `branchName` 实现同一父密钥下的分支级隔离

### 3.2 认证服务层：多凭证类型分发

**文件**：`apps/webapp/app/services/apiAuth.server.ts:301-313`

```typescript
function getApiKeyResult(apiKey: string): {
  apiKey: string;
  type: "PUBLIC" | "PRIVATE" | "PUBLIC_JWT";
} {
  const type = isPublicApiKey(apiKey)    // startsWith("pk_")
    ? "PUBLIC"
    : isSecretApiKey(apiKey)             // startsWith("tr_")
    ? "PRIVATE"
    : isPublicJWT(apiKey)                // JWT 格式 + payload.pub === true
    ? "PUBLIC_JWT"
    : "PRIVATE";                         // 兜底
  return { apiKey, type };
}
```

### 3.3 项目维度的二次校验：Legacy vs apiBuilder

#### 3.3.1 Legacy 链路：显式二次校验

**文件**：`apps/webapp/app/services/apiAuth.server.ts:440-614`

```typescript
export async function authenticatedEnvironmentForAuthentication(
  auth: AuthenticationResult,
  projectRef: string,
  slug: string,
  branch?: string
): Promise<AuthenticatedEnvironment> {
  if (slug === "staging") {
    slug = "stg";
  }

  switch (auth.type) {
    case "apiKey": {
      if (!auth.result.ok) {
        throw json({ error: auth.result.error }, { status: 401 });
      }

      // Legacy 强制校验：projectRef 必须完全匹配
      if (auth.result.environment.project.externalRef !== projectRef) {
        throw json(
          {
            error:
              "Invalid project ref for this API key. Make sure you are using an API key associated with that project.",
          },
          { status: 400 }
        );
      }

      // Legacy 强制校验：slug 或 branchName 必须匹配
      if (auth.result.environment.slug !== slug && auth.result.environment.branchName !== branch) {
        throw json(
          {
            error:
              "Invalid environment slug for this API key. Make sure you are using an API key associated with that environment.",
          },
          { status: 400 }
        );
      }

      return auth.result.environment;
    }
    // ... PAT 和 organizationAccessToken 类似逻辑
  }
}
```

**Legacy 调用示例**：`apps/webapp/app/routes/api.v1.projects.$projectRef.envvars.$slug.ts:29-34`

```typescript
const environment = await authenticatedEnvironmentForAuthentication(
  authenticationResult,
  parsedParams.data.projectRef,  // URL 路径参数
  parsedParams.data.slug,        // URL 路径参数
  branchNameFromRequest(request)
);
```

#### 3.3.2 apiBuilder 新链路：apiKey 路由 vs PAT 路由

apiBuilder 有两类路由，它们在 projectRef 的暴露和校验方式上完全不同：

##### 3.3.2.1 apiKey 路由（`createLoaderApiRoute` / `createActionApiRoute`）

**文件**：`apps/webapp/app/services/routeBuilders/apiBuilder.server.ts:51-79, 232-390`

```typescript
async function authenticateRequestForApiBuilder(
  request: Request,
  { allowJWT }: { allowJWT: boolean }
): Promise<
  | { ok: false; status: 401 | 403; error: string }
  | { ok: true; authentication: ApiAuthenticationResultSuccess; ability: RbacAbility }
> {
  const result = await rbac.authenticateBearer(request, { allowJWT });
  if (!result.ok) {
    return { ok: false, status: result.status, error: result.error };
  }

  // apiKey 路由不做显式 projectRef 校验
  // 项目隔离通过以下方式保证：
  // 1. authenticateBearer 返回的 environment 已绑定到具体 project
  // 2. findResource 使用 environment.projectId 过滤查询
  // 3. authorization 检查时 resource 已归属到该项目

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

**调用示例**：`apps/webapp/app/routes/api.v1.runs.ts:12-51`
```typescript
// URL: /api/v1/runs （路径中无 projectRef）
export const loader = createLoaderApiRoute(
  {
    searchParams: ApiRunListSearchParams,
    allowJWT: true,
    findResource: async () => 1, // 无资源查找，直接通过认证
    // handler 中通过 authentication.environment.project 隐式获取项目
  },
  async ({ searchParams, authentication, apiVersion }) => {
    const presenter = new ApiRunListPresenter();
    const result = await presenter.call(
      authentication.environment.project,  // 项目来自认证结果
      searchParams,
      apiVersion,
      authentication.environment
    );
    return json(result);
  }
);
```

##### 3.3.2.2 PAT 路由（`createLoaderPATApiRoute` / `createPATActionApiRoute`）

**文件**：`apps/webapp/app/services/routeBuilders/apiBuilder.server.ts:392-590`

```typescript
type PATRouteBuilderOptions<...> = {
  // ...
  // Resolves the target org/project for the request. Fed to
  // `rbac.authenticatePat` so the plugin can compute the user's role
  // floor (their authority in that org) for the cap intersection.
  // When omitted, the PAT runs in identity-only mode — no role floor,
  // no per-route ability gating beyond what authorization (if any)
  // declares against a permissive baseline.
  context?: (
    params: ...,
    request: Request
  ) =>
    | { organizationId?: string; projectId?: string }
    | Promise<{ organizationId?: string; projectId?: string }>;
  // ...
};

// PAT 路由认证流程
const ctx = contextFn ? await contextFn(parsedParams, request) : {};
const patAuth = await rbac.authenticatePat(request, ctx);  // ctx 传入 authenticatePat
```

**PAT 路由特点**：
- PAT 是**用户级令牌**，本身不绑定到任何项目/环境
- 必须通过 `context` 回调显式解析目标 `organizationId` / `projectId`
- context 传给 `rbac.authenticatePat`，用于在有 RBAC 插件时计算用户在该项目中的角色权限
- 无 context 时运行在 **identity-only 模式**，返回 `permissiveAbility`，依赖 authorization 做额外检查

#### 3.3.3 新旧链路对比

| 对比项 | Legacy 链路 | apiBuilder apiKey 路由 | apiBuilder PAT 路由 |
|--------|------------|---------------------|-------------------|
| 认证方式 | `authenticateRequest()` | `rbac.authenticateBearer()` | `rbac.authenticatePat(request, ctx)` |
| projectRef 位置 | URL 路径参数 | 不暴露，来自认证结果 | 通过 `context` 回调显式解析 |
| 校验方式 | 显式比较 `project.externalRef === projectRef` | 通过 `environment.projectId` 隐式过滤 | 无集中校验，依赖 context + RBAC |
| 失败状态码 | 400 Bad Request | 404 Not Found (资源不存在) | 403 Forbidden (无权限) |
| slug/branch 校验 | 显式检查 `environment.slug === slug` | 认证时已通过 branchName 路由 | 无（PAT 无环境概念） |
| 错误信息 | 明确提示 "Invalid project ref" | 不暴露项目存在性信息 | 权限不足提示 |
| 适用场景 | 旧版 API（`/api/v1/projects/:projectRef/...`） | 新版环境级 API | 用户级 API（CLI、跨项目操作） |

**设计意图**：
- Legacy：URL 路径中包含 `projectRef`，需要显式校验防止跨项目调用
- apiKey 路由：URL 不直接暴露 `projectRef`，通过 API Key → Environment → Project 链隐式绑定，更安全
- PAT 路由：PAT 是跨项目用户令牌，必须显式指定目标项目上下文才能计算权限

### 3.4 JWT 签名密钥轮换与宽限期机制

JWT 使用环境的 `apiKey` 作为签名密钥。当 API Key 轮换时，存在两种不同的宽限期回退机制：

#### 3.4.1 两种宽限期机制的区别

| 机制 | 适用场景 | 触发位置 | 回退目标 |
|------|---------|---------|---------|
| **API Key 宽限期** | API Key 认证 | `authenticateBearer`（fallback.ts:168-174） | 使用已吊销的 API Key 直接查询环境 |
| **JWT 签名密钥宽限期** | JWT 签名验证 | `validatePublicJwtKey`（jwtAuth.server.ts:47-53） | 使用已吊销的 API Key 作为 JWT 签名密钥重试 |

> **重要说明**：`authenticateBearer` 中的 JWT 路径（fallback.ts:86-132）**没有**签名密钥回退逻辑，签名验证失败直接返回 401。回退逻辑仅存在于 `validatePublicJwtKey` 函数（用于 realtime 认证）。

### 3.4.2 JWT 签名验证的三阶段流程（validatePublicJwtKey）

**文件**：`apps/webapp/app/services/realtime/jwtAuth.server.ts:38-78`

```
JWT 验证请求
   │
   ▼
┌──────────────────────────────────────────────────┐
│ 阶段 1：主密钥验证                                │
│ 使用 environment.apiKey（或父环境 apiKey）        │
│ 调用 validateJWT(token, signingKey) 验证签名       │
└───────────────────────┬──────────────────────────┘
                        │ 验证成功
                        ├────────────────────────► 返回验证通过
                        │ 验证失败（签名不匹配）
                        ▼
┌──────────────────────────────────────────────────┐
│ 阶段 2：宽限期回退（仅签名失败时触发）            │
│ 查询 revokedApiKey 表：                           │
│   WHERE runtimeEnvironmentId = 目标环境           │
│     AND expiresAt > NOW()                        │
│ 对每个旧密钥调用 validateJWT 重试                 │
└───────────────────────┬──────────────────────────┘
                        │ 任一重试成功
                        ├────────────────────────► 返回验证通过
                        │ 全部重试失败
                        ▼
┌──────────────────────────────────────────────────┐
│ 阶段 3：验证失败，返回具体错误                    │
│ - ERR_JWT_EXPIRED：JWT 自身 exp claim 过期        │
│ - ERR_JWT_CLAIM_INVALID：声明格式错误             │
│ - 其他：签名验证完全失败（密钥不匹配）             │
└──────────────────────────────────────────────────┘
```

**各阶段触发条件与行为**：

| 阶段 | 触发条件 | 操作 | 退出路径 |
|------|---------|------|---------|
| **阶段 1：主密钥验证** | 所有 JWT 请求必经 | 用当前环境 `apiKey` 验证签名 | 成功 → 返回通过；签名失败 → 进入阶段 2 |
| **阶段 2：宽限期回退** | 阶段 1 签名验证失败 | 查询 `revokedApiKey` 表中 `expiresAt > NOW()` 的所有旧密钥，依次重试 | 任一成功 → 返回通过；全部失败 → 进入阶段 3 |
| **阶段 3：失效** | 阶段 2 全部重试失败 | 按错误类型返回 401 | 直接返回错误 |

**核心代码**：
```typescript
// 阶段 1：主密钥验证
let result = await validateJWT(
  token,
  environment.parentEnvironment?.apiKey ?? environment.apiKey
);

// 阶段 2：宽限期回退（仅在阶段 1 失败时触发）
if (!result.ok) {
  result = await validateAgainstRevokedApiKeys(
    token,
    environment.parentEnvironment?.id ?? environment.id,
    result
  );
}

// 阶段 3：返回具体错误
if (!result.ok) {
  switch (result.code) {
    case "ERR_JWT_EXPIRED": { /* JWT 自身已过期 */ }
    case "ERR_JWT_CLAIM_INVALID": { /* 声明无效 */ }
    default: { /* 签名完全失败，密钥不匹配 */ }
  }
}
```

**回退验证实现**：
```typescript
async function validateAgainstRevokedApiKeys(
  token: string,
  signingEnvironmentId: string,
  primaryResult: ValidationResult
): Promise<ValidationResult> {
  // 仅查询当前环境下仍在宽限期内的密钥
  const revokedApiKeys = await $replica.revokedApiKey.findMany({
    where: {
      runtimeEnvironmentId: signingEnvironmentId,
      expiresAt: { gt: new Date() },  // 宽限期未过期
    },
    select: { apiKey: true },
  });

  // 依次尝试每个已吊销密钥作为签名密钥
  for (const { apiKey } of revokedApiKeys) {
    const fallbackResult = await validateJWT(token, apiKey);
    if (fallbackResult.ok) {
      return fallbackResult;  // 任一成功即返回
    }
  }

  return primaryResult;  // 全部失败，返回原始错误
}
```

**宽限期关键约束**：
1. **触发条件严格**：仅当 JWT 签名验证失败时触发，JWT 自身过期（`exp` claim）不触发回退
2. **范围限制**：仅查询当前 `environmentId` 下的吊销记录，防止跨环境尝试
3. **顺序保证**：严格按 主密钥 → 宽限期密钥 → 错误返回 的顺序执行
4. **失效时机**：宽限期到期后（默认 24 小时），`revokedApiKey` 表中该密钥的 `expiresAt` 字段已过，查询不到 → JWT 彻底失效
5. **设计权衡**：密钥轮换时已签发的 JWT 可继续使用到宽限期结束，避免业务中断

### 3.5 PAT 在无 RBAC 插件 Fallback 下的默认授权

**文件**：`internal-packages/rbac/src/fallback.ts:275-317`

```typescript
async authenticatePat(
  request: Request,
  context: { organizationId?: string; projectId?: string }
): Promise<PatAuthResult> {
  const rawToken = request.headers
    .get("Authorization")
    ?.replace(/^Bearer /, "")
    .trim();
  if (!rawToken || !rawToken.startsWith("tr_pat_")) {
    return { ok: false, status: 401, error: "Invalid or Missing PAT" };
  }

  const hashedToken = createHash("sha256").update(rawToken).digest("hex");
  const pat = await this.replica.personalAccessToken.findFirst({
    where: { hashedToken, revokedAt: null },
    select: { id: true, userId: true, lastAccessedAt: true },
  });
  if (!pat) {
    return { ok: false, status: 401, error: "Invalid PAT" };
  }

  return {
    ok: true,
    tokenId: pat.id,
    userId: pat.userId,
    lastAccessedAt: pat.lastAccessedAt,
    subject: {
      type: "personalAccessToken",
      tokenId: pat.id,
      organizationId: context.organizationId ?? "",
      projectId: context.projectId,
    },
    // 无 RBAC 插件时返回 permissiveAbility
    ability: permissiveAbility,  // can() = true, canSuper() = false
  };
}
```

**PAT Fallback 授权行为**：

| 场景 | 授权结果 | 说明 |
|------|---------|------|
| OSS 版本（无插件） | `permissiveAbility` | PAT 认证通过后，环境内无任何权限限制 |
| Cloud 版本（有插件） | 基于角色的能力 | PAT 关联的角色决定具体权限 |
| `lastAccessedAt` 更新 | JS 节流 + SQL 条件更新 | 5 分钟内重复访问不触发 DB 写入 |
| 跨项目访问 | 依赖用户成员身份 | PAT 是用户令牌，需目标项目中存在该用户的成员关系 |

**关键代码**：`apps/webapp/app/services/personalAccessToken.server.ts:129-150`

```typescript
export const PAT_LAST_ACCESSED_THROTTLE_MS = 5 * 60 * 1000; // 5分钟

export async function updateLastAccessedAtIfStale(
  tokenId: string,
  lastAccessedAt: Date | null
): Promise<void> {
  if (
    lastAccessedAt &&
    Date.now() - lastAccessedAt.getTime() <= PAT_LAST_ACCESSED_THROTTLE_MS
  ) {
    return; // 5 分钟内已更新过，跳过
  }
  await prisma.personalAccessToken.updateMany({
    where: {
      id: tokenId,
      revokedAt: null,
      OR: [
        { lastAccessedAt: null },
        { lastAccessedAt: { lt: new Date(Date.now() - PAT_LAST_ACCESSED_THROTTLE_MS) } },
      ],
    },
    data: { lastAccessedAt: new Date() },
  });
}
```

## 四、RBAC 作用域校验机制

### 4.1 JWT 作用域（Scopes）格式

```
scope = <action>:<resource_type>[:<resource_id>]
```

**示例**：
- `read:runs` - 读取所有运行记录
- `read:runs:run_abc123` - 仅读取特定运行记录
- `write:tasks` - 写入所有任务
- `admin` - 超级管理员（无资源限制）
- `read:all` - 读取所有资源

### 4.2 JWT Ability 构建

**文件**：`internal-packages/rbac/src/ability.ts:26-62`

```typescript
export function buildJwtAbility(scopes: string[]): RbacAbility {
  const matches = (action: string, r: RbacResource): boolean =>
    scopes.some((scope) => {
      const parts = scope.split(":");
      const scopeAction = parts[0];
      const scopeType = parts[1];
      const scopeId = parts.length > 2 ? parts.slice(2).join(":") : undefined;

      if (scopeAction === "admin" && !scopeType) return true;      // admin 通配
      if (scopeAction !== action && scopeAction !== "*") return false;
      if (scopeType === "all") return true;                        // all 通配
      if (scopeType !== r.type) return false;
      if (!scopeId) return true;                                   // 类型级匹配
      return scopeId === r.id;                                     // 实例级匹配
    });

  return {
    can(action: string, resource: RbacResource | RbacResource[]): boolean {
      return Array.isArray(resource)
        ? resource.some((r) => matches(action, r))  // 多资源任一通过即授权
        : matches(action, resource);
    },
    canSuper(): boolean { return false; },
  };
}
```

**作用域校验规则**：
1. 通配符 `admin` 授予全部权限
2. 通配符 `*` 匹配任意 action，`all` 匹配任意 resource_type
3. 无 resource_id 的 scope 是类型级授权（如 `read:runs` 覆盖所有 runs）
4. 有 resource_id 的 scope 是实例级授权（精确到单条记录）
5. 多资源数组（`RbacResource[]`）采用 OR 语义

### 4.3 动作别名机制

**文件**：`internal-packages/rbac/src/index.ts:32-46`

```typescript
const ACTION_ALIASES: Record<string, readonly string[]> = {
  trigger: ["write"],
  batchTrigger: ["write"],
  update: ["write"],
};

export function withActionAliases(underlying: RbacAbility): RbacAbility {
  return {
    can(action: string, resource: RbacResource | RbacResource[]): boolean {
      if (underlying.can(action, resource)) return true;
      const aliases = ACTION_ALIASES[action] ?? [];
      return aliases.some((a) => underlying.can(a, resource));
    },
    canSuper: () => underlying.canSuper(),
  };
}
```

设计意图：向后兼容。例如 `trigger` 动作可被 `write:tasks` scope 满足，无需为每个新动作单独签发 scope。

### 4.4 Fallback 能力（无 RBAC 插件时）

**文件**：`internal-packages/rbac/src/ability.ts:1-23`

| 能力类型 | 规则 | 适用场景 |
|---------|------|---------|
| `permissiveAbility` | `can()=true, canSuper()=false` | 普通认证用户，OSS 默认 |
| `superAbility` | `can()=true, canSuper()=true` | 平台管理员（`user.admin=true`） |
| `denyAbility` | `can()=false, canSuper()=false` | 未认证或已废弃 token |

**关键**：OSS 版本无 RBAC 插件时，API Key 认证通过后获得 `permissiveAbility`，即**环境内无额外限制**。项目隔离仍通过上一层的 Environment-Project 绑定保证。

## 五、API Builder 中的授权链路

### 5.1 认证与授权流水线

**文件**：`apps/webapp/app/services/routeBuilders/apiBuilder.server.ts:241-389`

```
请求到达
   ↓
authenticateRequestForApiBuilder()  → 调用 rbac.authenticateBearer()
   ↓ 失败返回 401/403
参数解析 (params/searchParams/headers)
   ↓
findResource()  → 加载目标资源（可跳过）
   ↓ 不存在返回 404
authorization?  → 可选的作用域校验
   ↓ 不通过返回 403
handler()  → 执行业务逻辑
```

### 5.2 多资源授权策略

**文件**：`apps/webapp/app/services/routeBuilders/apiBuilder.server.ts:104-165`

```typescript
// 任一资源匹配即授权（如一条记录有多个标识）
export function anyResource(resources: RbacResource[]): AnyResourceAuth

// 所有资源都必须匹配（如批量操作）
export function everyResource(resources: RbacResource[]): EveryResourceAuth

function checkAuth(ability: RbacAbility, action: string, resource: AuthResource): boolean {
  if (isEveryResource(resource)) {
    return resource.resources.every((r) => ability.can(action, r));
  }
  if (isAnyResource(resource)) {
    return ability.can(action, [...resource.resources]);
  }
  return ability.can(action, resource);
}
```

**使用场景示例**：
```typescript
// 单次触发：run 可通过 friendlyId、batchId、tags 等多种方式定位
authorization: {
  action: "read",
  resource: (run) => anyResource([
    { type: "runs", id: run.friendlyId },
    { type: "batches", id: run.batchId },
    { type: "tasks", id: run.taskIdentifier },
  ]),
}

// 批量触发：每条记录必须单独授权
authorization: {
  action: "trigger",
  resource: (_, __, ___, body) => everyResource(
    body.items.map(item => ({ type: "tasks", id: item.taskId }))
  ),
}
```

### 5.3 JWT 认证流程（RBAC Fallback）

**文件**：`internal-packages/rbac/src/fallback.ts:86-132`

```typescript
async authenticateBearer(request: Request, options?: { allowJWT?: boolean }) {
  if (options?.allowJWT && isPublicJWT(rawToken)) {
    const envId = extractJWTSub(rawToken);  // payload.sub = environmentId
    const env = await this.replica.runtimeEnvironment.findFirst({ where: { id: envId } });
    
    // JWT 签名密钥 = 环境的 apiKey（或父环境 apiKey）
    const signingKey = env.parentEnvironment?.apiKey ?? env.apiKey;
    const result = await validateJWT(rawToken, signingKey);
    
    const scopes = Array.isArray(result.payload.scopes) 
      ? result.payload.scopes as string[] 
      : [];
    
    return {
      ok: true,
      environment: toAuthenticatedEnvironment(env),
      subject: { type: "publicJWT", environmentId: env.id, ... },
      ability: buildJwtAbility(scopes),  // 基于 scopes 构建细粒度能力
      jwt: { realtime, oneTimeUse },
    };
  }
  // ... 普通 API Key 流程
}
```

**JWT 项目隔离**：
- JWT 的 `sub` 声明直接绑定 `environmentId`，通过环境 → 项目关联实现隔离
- 签名密钥使用环境自身的 apiKey，确保无法跨环境伪造
- `scopes` 声明进一步限制可访问的资源范围

## 六、Preview 分支匹配与边界条件

### 6.1 Preview 环境认证流程

**文件**：`internal-packages/rbac/src/fallback.ts:139-208`

```typescript
async authenticateBearer(request: Request, options?: { allowJWT?: boolean }) {
  // ... JWT 流程省略

  // PREVIEW 环境分支路由
  const branchName = sanitizeBranchName(request.headers.get("x-trigger-branch"));
  const include = {
    project: true,
    organization: true,
    orgMember: { select: { userId: true, user: { select: { ... } } },
    parentEnvironment: { select: { id: true, apiKey: true } },
    childEnvironments: branchName
      ? { where: { branchName, archivedAt: null } }
      : undefined,
  } as const;

  let env = await this.replica.runtimeEnvironment.findFirst({
    where: { apiKey: rawToken },
    include,
  });

  // ... 吊销密钥回退

  // PREVIEW 环境边界条件处理
  if (env.type === "PREVIEW") {
    // 边界 1：PREVIEW 环境必须提供 branchName
    if (!branchName) {
      return {
        ok: false,
        status: 401,
        error: "x-trigger-branch header required for preview env",
      };
    }
    const child = env.childEnvironments?.[0];
    // 边界 2：branchName 对应的子环境必须存在
    if (!child) {
      return { ok: false, status: 401, error: "No matching branch env" };
    }
    // Pivot：子环境继承父环境的安全上下文
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
  // ...
}
```

### 6.2 slug/branch 判断边界条件

| 场景 | 条件 | 行为 |
|------|------|------|
| PREVIEW 环境无 `x-trigger-branch` 头 | `env.type === "PREVIEW" && !branchName` | 返回 401，明确要求提供 branch 头 |
| PREVIEW 环境有 branch 但子环境不存在 | `childEnvironments.length === 0` | 返回 401，提示无匹配分支 |
| 子环境已归档 | `child.archivedAt !== null` | 不返回（查询条件已过滤） |
| 非 PREVIEW 环境传了 branch 头 | `env.type !== "PREVIEW" && branchName` | branchName 被忽略，使用主环境 |
| slug 参数为 "staging" | `slug === "staging"` | 自动转换为 "stg" 匹配 |
| Legacy 链路 slug 不匹配 | `env.slug !== slug && env.branchName !== branch` | 返回 400 |

### 6.3 sanitizeBranchName 的实际行为

**文件**：`packages/core/src/v3/utils/gitBranch.ts:20-31`

```typescript
export function sanitizeBranchName(ref: string | null | undefined): string | null {
  if (!ref) return null;
  if (ref.startsWith("refs/heads/")) return ref.substring("refs/heads/".length);
  if (ref.startsWith("refs/remotes/")) return ref.substring("refs/remotes/".length);
  if (ref.startsWith("refs/tags/")) return ref.substring("refs/tags/".length);
  if (ref.startsWith("refs/pull/")) return ref.substring("refs/pull/".length);
  if (ref.startsWith("refs/merge/")) return ref.substring("refs/merge/".length);
  if (ref.startsWith("refs/release/")) return ref.substring("refs/release/".length);
  if (ref.startsWith("refs/")) return null;  // 未知 refs 前缀返回 null

  return ref;  // 普通分支名原样返回
}
```

**核心功能与限制**：

| 输入示例 | 输出 | 说明 |
|---------|------|------|
| `null` / `undefined` / `""` | `null` | 空值直接返回 null |
| `"refs/heads/feature/login"` | `"feature/login"` | 剥离本地分支前缀 |
| `"refs/remotes/origin/feature/login"` | `"origin/feature/login"` | 剥离远程分支前缀，保留 origin |
| `"refs/tags/v1.0.0"` | `"v1.0.0"` | 剥离标签前缀 |
| `"refs/pull/123/head"` | `"123/head"` | 剥离 PR 前缀 |
| `"refs/unknown/xxx"` | `null` | 未知 refs 前缀拒绝 |
| `"feature/login"` | `"feature/login"` | 普通分支名原样返回 |

**重要澄清**：
- ❌ **不是字符清洗器**：不做任何特殊字符过滤、转义或截断
- ❌ **不验证分支名合法性**：合法性检查由独立的 `isValidGitBranchName()` 函数负责
- ✅ **Git ref 前缀剥离器**：唯一职责是将完整 Git ref 路径简化为短名
- ✅ **返回 `string | null`**：不是 `string | undefined`，空值返回 null 而非 undefined

## 七、完整协作链路图

```
┌─────────────────────────────────────────────────────────────────┐
│                     外部凭证（请求头）                           │
│  Authorization: Bearer tr_dev_xxx / tr_pat_xxx / JWT           │
│  x-trigger-branch: feature-branch  (可选)                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                第一层：认证 (Authentication)                     │
│  目标：验证"你是谁"                                              │
│                                                                 │
│  ┌─ apiKey 路由 ─────────────────────────────────────────────┐  │
│  │ tr_xxx → authenticateBearer()                              │  │
│  │         → runtimeEnvironment.projectId → 确定项目归属      │  │
│  │         → 检查 project.deletedAt → 软删除项目拒绝          │  │
│  │         → PREVIEW 环境检查 branchName → 子环境 pivot       │  │
│  │         → 已吊销密钥回退校验（24小时宽限期）                │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌─ JWT 认证 ────────────────────────────────────────────────┐  │
│  │ JWT → extractJWTSub() → environmentId                     │  │
│  │     → 阶段 1: 主密钥 validateJWT(env.apiKey)               │  │
│  │     → 阶段 2: 签名失败时（仅 validatePublicJwtKey）        │  │
│  │             宽限期 revokedApiKeys 回退重试                 │  │
│  │     → 阶段 3: 全部失败 → 返回具体错误                       │  │
│  │     → payload.scopes → buildJwtAbility() 细粒度能力        │  │
│  │  注意：authenticateBearer 中 JWT 路径无宽限期回退          │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌─ PAT 路由 ────────────────────────────────────────────────┐  │
│  │ PAT → authenticatePat(request, ctx)                        │  │
│  │     → 哈希比对 → 验证 token 有效性                          │  │
│  │     → ctx 解析 organizationId/projectId（可选）             │  │
│  │     → 无 RBAC 插件时返回 permissiveAbility                  │  │
│  │     → lastAccessedAt 5 分钟节流更新                         │  │
│  └────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                第二层：项目隔离校验                               │
│  Legacy 链路：显式校验                                           │
│    ✓ environment.project.externalRef === request.projectRef     │
│    ✓ environment.slug === request.slug                           │
│    ✗ 不匹配 → 400 "Invalid project ref"                          │
│                                                                 │
│  apiBuilder apiKey 路由：隐式校验                                │
│    ✓ findResource 使用 environment.projectId 过滤查询            │
│    ✓ 资源不存在 → 404（不暴露项目存在性）                        │
│                                                                 │
│  apiBuilder PAT 路由：上下文绑定                                 │
│    ✓ 通过 context 回调显式解析目标 projectId                     │
│    ✓ 无 context 时运行在 identity-only 模式                      │
│    ✓ RBAC 插件基于 ctx 计算用户在该项目的角色权限                │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                第三层：授权 (Authorization)                      │
│  目标：验证"你能做什么"                                          │
│                                                                 │
│  PRIVATE API Key → permissiveAbility (环境内无限制)              │
│  PUBLIC JWT → buildJwtAbility(scopes) → can(action, resource)   │
│  PAT → 无插件时 permissiveAbility，有插件时基于角色              │
│                                                                 │
│  动作别名：trigger/batchTrigger/update → write                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                第四层：资源级授权检查                             │
│  apiBuilder authorization 配置                                   │
│                                                                 │
│  anyResource([...]) → 任一匹配即通过（多标识资源）                │
│  everyResource([...]) → 全部匹配才通过（批量操作）                │
└─────────────────────────────────────────────────────────────────┘
```

## 八、关键设计要点

### 8.1 项目隔离的四层保障

1. **凭证绑定层**：API Key 唯一绑定到 RuntimeEnvironment → 天然绑定 Project
2. **软删除防护**：`project.deletedAt` 检查，已删除项目即使密钥有效也无法访问
3. **参数校验层**（Legacy）：`project.externalRef` 与请求参数二次匹配，防止跨项目调用
4. **作用域限制层**：JWT scopes 可进一步限制到项目内特定资源类型/实例

### 8.2 预览环境的特殊设计

- 预览环境共享父环境的 apiKey，通过 `x-trigger-branch` 头区分
- 认证时自动 pivot 到子环境，但继承父环境的组织/项目/成员关系
- 实现"同一密钥，分支隔离"的部署体验
- 无 branch 头时明确返回 401 提示，避免歧义

### 8.3 向后兼容策略

- 旧版 `pk_*` 公钥在新 RBAC 路径（apiBuilder）直接返回 401
- 动作别名（`trigger`→`write`）保证旧 scope 继续有效
- API Key 吊销宽限期（24小时）：旧密钥在宽限期内仍可查询环境
- JWT 签名密钥宽限期（仅 `validatePublicJwtKey`）：签名失败时回退到已吊销密钥重试
- `authenticateBearer` 中 JWT 路径无签名密钥回退，签名失败直接返回 401

### 8.4 性能优化

- `AuthenticatedEnvironment` 是精简结构，只包含认证链路上必需的字段
- 预览环境查询时通过 include 一次性加载子环境，避免 N+1
- PAT 的 `lastAccessedAt` 更新采用 JS 层节流 + SQL 条件更新双层优化
- JWT 签名密钥回退验证（仅 `validatePublicJwtKey`）仅在主验证失败时触发，不影响正常路径性能

## 九、常见疑问

**Q: 为什么 API Key 不直接存储 projectId？**
A: 通过 RuntimeEnvironment 间接关联。一个项目有多个环境（dev/stg/prod），每个环境独立密钥是更精细的安全模型。

**Q: 如何实现"只读 API Key"？**
A: 普通 `tr_*` API Key 是环境全权限。如需只读，应签发带 `read:all` scope 的 JWT，而非使用原生 API Key。

**Q: 跨项目的密钥如何实现？**
A: 使用 PAT（`tr_pat_*`）而非环境级 API Key。PAT 是用户身份令牌，通过用户在各项目的成员身份实现跨项目访问。

**Q: 为什么 JWT 用 apiKey 作为签名密钥？**
A: 密钥轮换时，新签发的 JWT 使用新密钥签名，旧密钥签发的 JWT 在 24 小时宽限期内仍可通过回退机制验证。宽限期结束后，旧密钥从 `revokedApiKey` 表中过期，所有旧 JWT 自动失效，无需额外维护 JWT 黑名单。

**Q: Legacy 和 apiBuilder apiKey 路由的 projectRef 校验为什么不一样？**
A: Legacy URL 包含 `projectRef` 路径参数，必须显式校验防止跨项目调用；apiBuilder apiKey 路由不暴露 `projectRef`，通过 API Key → Environment → Project 链隐式绑定，更安全且减少冗余。

**Q: apiBuilder 中 PAT 路由为什么需要 context 回调？**
A: PAT 是用户级令牌，本身不绑定到任何项目/环境。必须通过 context 回调显式解析目标 `organizationId` / `projectId`，才能在有 RBAC 插件时计算用户在该项目中的角色权限。无 context 时 PAT 运行在 identity-only 模式。

**Q: JWT 签名验证的三个阶段顺序是怎样的？**
A: 严格按 阶段 1（主密钥验证）→ 阶段 2（宽限期回退）→ 阶段 3（失效返回错误）的顺序执行。只有前一阶段失败才会进入后一阶段。宽限期回退**仅在签名验证失败时触发**，不处理 JWT 自身过期（`exp` claim）。注意：该三阶段流程仅适用于 `validatePublicJwtKey`（realtime 认证），`authenticateBearer` 中的 JWT 路径无宽限期回退。

**Q: 所有 JWT 认证路径都有宽限期回退吗？**
A: 不是。只有 `validatePublicJwtKey` 函数（用于 realtime 认证）有签名密钥宽限期回退逻辑。`authenticateBearer` 中的 JWT 路径（apiBuilder 路由）签名验证失败直接返回 401，无回退。但 API Key 自身的吊销宽限期（使用旧 API Key 直接查询环境）在两条路径中都存在。

**Q: JWT 宽限期结束后会怎样？**
A: 宽限期到期后（默认 24 小时），`revokedApiKey` 表中旧密钥的 `expiresAt` 字段已过，查询不到该密钥。此时用旧密钥签名的 JWT 在阶段 2 回退时找不到可重试的密钥，进入阶段 3 返回错误，JWT 彻底失效。

**Q: sanitizeBranchName 做字符清洗吗？**
A: 不做。sanitizeBranchName 的唯一职责是剥离 Git ref 前缀（`refs/heads/`、`refs/remotes/` 等），不做任何特殊字符过滤或转义。分支名合法性检查由独立的 `isValidGitBranchName()` 函数负责。

**Q: PREVIEW 环境为什么必须传 x-trigger-branch 头？**
A: PREVIEW 是父环境，本身不直接承载运行。必须通过 branch 头定位到具体子环境，确保操作的是正确的分支环境。无 branch 头时明确返回 401。
