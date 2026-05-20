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

### 3.3 项目维度的二次校验

**文件**：`apps/webapp/app/services/apiAuth.server.ts:456-464`

```typescript
if (auth.result.environment.project.externalRef !== projectRef) {
  throw json(
    {
      error: "Invalid project ref for this API key. Make sure you are using an API key associated with that project.",
    },
    { status: 400 }
  );
}
```

在 `authenticatedEnvironmentForAuthentication` 中，即使 API Key 认证通过，仍需校验：
1. `project.externalRef` 必须与请求参数中的 `projectRef` 一致
2. `environment.slug` 必须与请求的 `slug` 匹配（或分支名匹配）
3. 这是**项目维度的强制隔离边界**，防止跨项目密钥滥用

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

## 六、完整协作链路图

```
┌─────────────────────────────────────────────────────────────────┐
│                     外部凭证（请求头）                           │
│  Authorization: Bearer tr_dev_xxx / tr_pat_xxx / JWT           │
│  x-trigger-branch: feature-branch  (可选)                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                第一层：凭证类型识别 + 环境解析                    │
│  apps/webapp/app/services/apiAuth.server.ts                     │
│  internal-packages/rbac/src/fallback.ts:authenticateBearer()   │
│                                                                 │
│  tr_xxx → findEnvironmentByApiKey(apiKey)                       │
│          → runtimeEnvironment.projectId → 确定项目归属          │
│          → 检查 project.deletedAt → 软删除项目拒绝              │
│          → 预览环境 branchName 路由 → 子环境隔离                │
│                                                                 │
│  JWT → extractJWTSub() → environmentId                          │
│      → validateJWT(token, env.apiKey) → 签名校验                │
│      → payload.scopes → 构建细粒度能力                          │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                第二层：项目维度强制校验                           │
│  apps/webapp/app/services/apiAuth.server.ts:456-474             │
│                                                                 │
│  ✓ environment.project.externalRef === request.projectRef       │
│  ✓ environment.slug === request.slug                            │
│  ✗ 不匹配 → 400 "Invalid project ref for this API key"          │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                第三层：RBAC 作用域校验（可选）                    │
│  apps/webapp/app/services/routeBuilders/apiBuilder.server.ts    │
│  internal-packages/rbac/src/ability.ts:buildJwtAbility()        │
│                                                                 │
│  PRIVATE API Key → permissiveAbility (环境内无限制)              │
│  PUBLIC JWT → buildJwtAbility(scopes) → can(action, resource)   │
│  PAT → 用户级能力（基于角色）                                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                第四层：资源级授权检查                             │
│  apiBuilder authorization 配置                                   │
│                                                                 │
│  anyResource([...]) → 任一匹配即通过                              │
│  everyResource([...]) → 全部匹配才通过                            │
└─────────────────────────────────────────────────────────────────┘
```

## 七、关键设计要点

### 7.1 项目隔离的四层保障

1. **凭证绑定层**：API Key 唯一绑定到 RuntimeEnvironment → 天然绑定 Project
2. **软删除防护**：`project.deletedAt` 检查，已删除项目即使密钥有效也无法访问
3. **参数校验层**：`project.externalRef` 与请求参数二次匹配，防止跨项目调用
4. **作用域限制层**：JWT scopes 可进一步限制到项目内特定资源类型/实例

### 7.2 预览环境的特殊设计

- 预览环境共享父环境的 apiKey，通过 `x-trigger-branch` 头区分
- 认证时自动 pivot 到子环境，但继承父环境的组织/项目/成员关系
- 实现"同一密钥，分支隔离"的部署体验

### 7.3 向后兼容策略

- 旧版 `pk_*` 公钥在新 RBAC 路径（apiBuilder）直接返回 401
- 动作别名（`trigger`→`write`）保证旧 scope 继续有效
- 吊销宽限期（24小时）避免密钥轮换导致业务中断

### 7.4 性能优化

- `AuthenticatedEnvironment` 是精简结构，只包含认证链路上必需的字段
- 预览环境查询时通过 include 一次性加载子环境，避免 N+1
- PAT 的 `lastAccessedAt` 更新采用 JS 层节流 + SQL 条件更新双层优化

## 八、常见疑问

**Q: 为什么 API Key 不直接存储 projectId？**
A: 通过 RuntimeEnvironment 间接关联。一个项目有多个环境（dev/stg/prod），每个环境独立密钥是更精细的安全模型。

**Q: 如何实现"只读 API Key"？**
A: 普通 `tr_*` API Key 是环境全权限。如需只读，应签发带 `read:all` scope 的 JWT，而非使用原生 API Key。

**Q: 跨项目的密钥如何实现？**
A: 使用 PAT（`tr_pat_*`）而非环境级 API Key。PAT 是用户身份令牌，通过用户在各项目的成员身份实现跨项目访问。

**Q: 为什么 JWT 用 apiKey 作为签名密钥？**
A: 密钥轮换自动使所有该环境签发的 JWT 失效，无需额外维护 JWT 黑名单。
