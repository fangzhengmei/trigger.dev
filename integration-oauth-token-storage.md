# Trigger.dev 第三方 OAuth 凭据存储安全分析

> 本文档面向金融客户安全部门，基于代码逐层剖析 trigger.dev 如何处理第三方 OAuth 凭据的授权、加密存储、运行时取用、过期与撤销，以及多用户多环境下的权限隔离。

---

## 1. 端到端 OAuth 授权链路概览

trigger.dev 当前支持两类第三方集成 OAuth：**Slack** 和 **Vercel**。整体链路如下：

```
前端点击"连接 Slack/Vercel"
  → 后端生成 OAuth URL + state（JWT，15分钟过期）
  → 用户在第三方页面授权
  → 第三方回调 callback URL（带 code + state）
  → 后端用 code 换取 access_token（服务端到服务端）
  → access_token 经 AES-256-GCM 加密后存入 SecretStore
  → 数据库记录 SecretReference → OrganizationIntegration 关联
  → 任务运行时按需取出解密使用
```

### 1.1 授权发起

| 集成 | 授权 URL 生成 | state 参数 |
|------|-------------|-----------|
| Slack | [OrgIntegrationRepository.slackAuthorizationUrl](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/orgIntegration.server.ts#L116-L133) | 普通字符串，存入 Session cookie |
| Vercel | [OrgIntegrationRepository.vercelInstallUrl](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/orgIntegration.server.ts#L108-L114) | JWT，含 orgId/projectId/envSlug，15分钟过期，签名密钥为 `ENCRYPTION_KEY` |

Vercel 的 state 使用 JWT 生成（[vercelOAuthState.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/v3/vercel/vercelOAuthState.server.ts#L15-L22)），在回调时验证签名和过期时间，防止 CSRF 和重放攻击。Slack 的 state 通过 Session cookie 传递。

### 1.2 回调处理与 Token 交换

**Slack 回调**：[integrations.$serviceName.callback.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/integrations.$serviceName.callback.ts#L1-L66)

1. 验证用户登录状态（`requireUserId`）
2. 解析 URL 参数中的 `code` 和 `state`
3. 调用 `CreateOrgIntegrationService.call()` → `OrgIntegrationRepository.createOrgIntegration()`
4. 内部使用 Slack SDK `client.oauth.v2.access()` 用 code 换取 token（服务端到服务端，client_secret 不暴露给前端）

**Vercel 回调**：[vercel.callback.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.callback.ts#L21-L78) → [vercel.connect.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.connect.tsx#L66-L170)

1. 验证用户登录状态
2. 验证 state JWT 签名与过期
3. 验证用户对 project/org 有权限
4. 调用 `VercelIntegrationRepository.exchangeCodeForToken()` 用 code 换取 access_token
5. 调用 `createOrFindVercelIntegration()` 创建或更新集成记录

### 1.3 凭据存储路径

Token 交换完成后，凭据的存储路径如下：

```
access_token + metadata
  → JSON.stringify
  → AES-256-GCM 加密（ENCRYPTION_KEY + 随机 nonce）
  → { nonce, ciphertext, tag } 结构
  → SecretStore 表（Postgres，key = integrationFriendlyId）
  → SecretReference 记录指向 SecretStore 的 key
  → OrganizationIntegration.tokenReferenceId → SecretReference.id
```

---

## 2. 加密体系分析

### 2.1 加密算法：AES-256-GCM

核心加密实现在 [secretStore.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/secrets/secretStore.server.ts#L219-L254)：

```typescript
export async function encryptSecret(encryptionKey: string, value: string): Promise<EncryptedSecretValue> {
  const nonce = nodeCrypto.randomBytes(12);
  const cipher = nodeCrypto.createCipheriv("aes-256-gcm", encryptionKey, nonce);
  let encrypted = cipher.update(value, "utf8", "hex");
  encrypted += cipher.final("hex");
  const tag = cipher.getAuthTag().toString("hex");
  return { nonce: nonce.toString("hex"), ciphertext: encrypted, tag };
}
```

| 安全属性 | 实现方式 |
|---------|---------|
| 算法 | AES-256-GCM（认证加密） |
| 密钥长度 | 256 位（32 字节） |
| Nonce | 每次加密随机生成 12 字节 |
| 认证标签 | GCM 内置 128-bit tag，防篡改 |
| 密钥来源 | 环境变量 `ENCRYPTION_KEY` |

GCM 模式同时提供**机密性**和**完整性**保护。解密时如果 tag 校验失败，说明密文被篡改，直接报错。

### 2.2 密钥粒度

**当前设计：全局单一 `ENCRYPTION_KEY`**

- 密钥在 [env.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/env.server.ts#L114-L119) 中验证，要求严格 32 字节
- 所有租户（organization）、所有类型的凭据共享同一加密密钥
- 数据隔离完全依赖数据库层面的行级权限和逻辑隔离

**影响范围**：同一 `ENCRYPTION_KEY` 加密的凭据包括：

| 凭据类型 | 存储位置 | Schema |
|---------|---------|--------|
| Slack OAuth token | SecretStore 表 | `SlackSecretSchema`（含 botAccessToken, userAccessToken, refreshToken 等） |
| Vercel OAuth token | SecretStore 表 | `VercelSecretSchema`（含 accessToken, tokenType, teamId 等） |
| 环境变量（标记为 secret） | SecretStore 表 | `SecretValue`（含 secret 字符串） |
| MFA 密钥 | SecretStore 表 | `SecretSchema`（含 secret 字符串） |
| Personal Access Token | PersonalAccessToken.encryptedToken 列 | 直接 JSON 列存储 `{ nonce, ciphertext, tag }` |

### 2.3 加密边界

```
┌─────────────────────────────────────────────────────────┐
│ 应用内存（明文）                                          │
│   - token 交换后短暂存在                                   │
│   - 运行时取出解密后短暂存在                                │
├─────────────────────────────────────────────────────────┤
│ SecretStore 表 / encryptedToken 列（密文）                 │
│   - AES-256-GCM 加密                                      │
│   - 存储 { nonce, ciphertext, tag } 结构                   │
├─────────────────────────────────────────────────────────┤
│ SecretReference 表（索引层，不含密文）                       │
│   - key: 指向 SecretStore 的唯一键                         │
│   - provider: "DATABASE" | "AWS_PARAM_STORE"              │
├─────────────────────────────────────────────────────────┤
│ OrganizationIntegration 表（业务层，不含密文）               │
│   - tokenReferenceId → SecretReference                    │
│   - integrationData: 非敏感元数据                          │
│   - organizationId: 租户隔离                               │
└─────────────────────────────────────────────────────────┘
```

**关键设计原则**：
- 密文与业务数据分离：`SecretStore` 表只存加密后的值，业务表通过 `SecretReference` 间接引用
- 数据库中不存在明文凭据
- `PersonalAccessToken` 的 `obfuscatedToken` 列仅显示脱敏格式（如 `tr_pat_bhbd•••••••••••••••••••fd4a`）

### 2.4 SecretStore 版本演进

[secretStore.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/secrets/secretStore.server.ts#L86-L88) 中存在版本兼容逻辑：

- **version "1"**：明文 JSON 存储（历史遗留）
- **version "2"**：AES-256-GCM 加密存储（当前版本）

新写入始终使用 version "2"。读取时自动判断版本，version "1" 直接解析 JSON，version "2" 先解密再解析。

### 2.5 备选存储后端

[SecretStoreProvider](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/secrets/secretStore.server.ts#L200-L217) 枚举支持：

| Provider | 状态 | 说明 |
|----------|------|------|
| `DATABASE` | ✅ 已实现 | Postgres + AES-256-GCM |
| `AWS_PARAM_STORE` | ⚠️ 枚举已定义，未实现 | 可扩展为 AWS SSM Parameter Store |

---

## 3. Token 过期与刷新路径

### 3.1 Slack Token

[SlackSecretSchema](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/orgIntegration.server.ts#L16-L24) 中保存了以下与过期相关的字段：

```typescript
const SlackSecretSchema = z.object({
  botAccessToken: z.string(),
  userAccessToken: z.string().optional(),
  expiresIn: z.number().optional(),      // access_token 过期时间（秒）
  refreshToken: z.string().optional(),    // 用于刷新 access_token
  botScopes: z.array(z.string()).optional(),
  userScopes: z.array(z.string()).optional(),
  raw: z.record(z.any()).optional(),
});
```

**当前状态**：[OrgIntegrationRepository.getAuthenticatedClientForIntegration](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/orgIntegration.server.ts#L58-L94) 中有明确的 `// TODO refresh access token here` 注释，说明 **Slack token 的自动刷新机制尚未实现**。当前行为是直接使用存储的 access_token，如果 token 过期，API 调用将失败。

Slack 的 bot token（xoxb-）在应用未重新安装的情况下通常不会过期，但 user token 会过期。refreshToken 已被存储，但刷新逻辑尚未编码。

### 3.2 Vercel Token

[VercelSecretSchema](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/vercelIntegration.server.ts#L147-L154)：

```typescript
const VercelSecretSchema = z.object({
  accessToken: z.string(),
  tokenType: z.string().optional(),
  teamId: z.string().nullable().optional(),
  userId: z.string().optional(),
  installationId: z.string().optional(),
  raw: z.record(z.any()).optional(),
});
```

Vercel 集成的 access_token 是长期有效的（不会过期），因此不需要 refresh_token 和刷新机制。

Vercel 提供了 token 有效性验证方法：[validateVercelToken](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/vercelIntegration.server.ts#L340-L357)，通过调用 Vercel API 检查 token 是否仍然有效。

### 3.3 Personal Access Token

[PAT 撤销](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/personalAccessToken.server.ts#L95-L109) 通过设置 `revokedAt` 时间戳实现：

```typescript
export async function revokePersonalAccessToken(tokenId: string, userId: string) {
  const result = await prisma.personalAccessToken.updateMany({
    where: { id: tokenId, userId },
    data: { revokedAt: new Date() },
  });
}
```

认证时检查 `revokedAt: null`（[personalAccessToken.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/personalAccessToken.server.ts#L246-L251)），已撤销的 token 无法通过认证。

### 3.4 Organization Access Token

[OrganizationAccessToken](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/organizationAccessToken.server.ts#L92-L136) 支持**过期**和**撤销**两种失效机制：

- `expiresAt`：可选的过期时间，认证时 `WHERE expiresAt IS NULL OR expiresAt >= NOW()`
- `revokedAt`：撤销时间戳，认证时 `WHERE revokedAt IS NULL`

**注意**：OrganizationAccessToken **不加密存储**，仅存储 `hashedToken`（SHA-256 哈希），类似密码的存储方式。原始 token 只在创建时返回一次。

### 3.5 过期/撤销路径汇总

| 凭据类型 | 过期机制 | 撤销机制 | 刷新机制 |
|---------|---------|---------|---------|
| Slack OAuth Token | Schema 中有 expiresIn 字段但未实现检查 | 通过删除 OrganizationIntegration（软删除 deletedAt） | refreshToken 已存储但未实现（TODO） |
| Vercel OAuth Token | 无（长期有效） | 通过删除 OrganizationIntegration（软删除 deletedAt） | 不需要 |
| Personal Access Token | 无 | revokedAt 时间戳 | 不适用 |
| Organization Access Token | expiresAt 可选 | revokedAt 时间戳 | 不适用 |
| 环境变量 Secret | 无 | 删除 EnvironmentVariableValue 时级联删除 SecretReference + SecretStore | 不适用 |

---

## 4. 多用户多环境权限隔离

### 4.1 组织（Organization）级别隔离

trigger.dev 采用**组织作为顶层隔离单元**的设计：

```
Organization
  ├── OrgMember (用户-组织关联，含角色)
  ├── RuntimeEnvironment (dev/staging/prod)
  │     └── apiKey (环境级别 API Key)
  ├── Project
  │     ├── EnvironmentVariable → SecretReference → SecretStore
  │     └── OrganizationProjectIntegration
  └── OrganizationIntegration
        └── tokenReference → SecretReference → SecretStore
```

**数据库层**：
- `OrganizationIntegration.organizationId` 外键确保每个集成属于一个组织
- 查询时始终带 `organizationId` 条件，防止跨组织访问

### 4.2 集成安装权限控制

**Slack 集成安装**（[CreateOrgIntegrationService](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/v3/services/createOrgIntegration.server.ts#L6-L29)）：

1. 验证 `userId` 是 `organization` 的成员（`members: { some: { userId } }`）
2. 只有组织成员才能为该组织安装集成

**Vercel 集成安装**（[vercel.connect.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.connect.tsx#L66-L170)）：

1. `requireUserId` 验证用户已登录
2. 验证 state JWT 签名和过期（防篡改 + 防重放）
3. 验证用户对 project 有访问权限（`organization: { members: { some: { userId } } }`）
4. 验证 environment 属于该 project

### 4.3 项目级集成关联

`OrganizationProjectIntegration` 是组织级集成与项目的关联表，提供更细粒度的隔离：

```prisma
model OrganizationProjectIntegration {
  organizationIntegrationId  String   // 指向组织级集成
  projectId                  String   // 指向具体项目
  externalEntityId           String   // 外部实体 ID（如 Vercel projectId）
  installedBy                String?  // 安装者 userId
}
```

一个组织可以有一个 Vercel 集成（组织级别 token），但可以选择性地关联到不同项目。

### 4.4 运行时凭据取用

任务运行时获取集成客户端的流程（以 Slack 为例）：

```
OrgIntegrationRepository.getAuthenticatedClientForIntegration()
  → getSecretStore(integration.tokenReference.provider)  // 按provider获取存储
  → secretStore.getSecret(SlackSecretSchema, integration.tokenReference.key)
  → 解密后构造 WebClient
```

**关键隔离保障**：
1. 集成记录通过 `organizationId` 隔离，不同组织的集成互不可见
2. `SecretReference.key` 使用 `friendlyId`（如 `org_integration_xxxx`），全局唯一且不可猜测
3. 凭据在内存中仅短暂存在（函数调用周期内），用后即弃

### 4.5 环境变量隔离

环境变量的加密存储通过 key 命名规则实现环境级隔离：

```typescript
// [environmentVariablesRepository.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts#L26-L36)
function secretKey(projectId: string, environmentId: string, key: string) {
  return `environmentvariable:${projectId}:${environmentId}:${key}`;
}
```

每个环境的变量值独立加密存储，同一个变量名在不同环境（dev/staging/prod）有不同的加密值。

### 4.6 API 认证与权限层级

API 请求的认证链路（[apiAuth.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/apiAuth.server.ts)）支持多种认证方式，每种方式绑定的隔离粒度不同：

| 认证方式 | 格式前缀 | 隔离粒度 | 存储方式 |
|---------|---------|---------|---------|
| API Key | `tr_dev_` / `tr_prod_` | 环境（Environment） | 明文存储，哈希查找 |
| Personal Access Token | `tr_pat_` | 用户（User） | AES-256-GCM 加密 + SHA-256 哈希 |
| Organization Access Token | `tr_oat_` | 组织（Organization） | SHA-256 哈希（无加密） |
| Public JWT | - | 环境（Environment） | JWT 签名验证 |

### 4.7 RBAC 权限控制

[rbac/fallback.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/internal-packages/rbac/src/fallback.ts) 实现了基于角色的访问控制：

- 用户认证后构建 `RbacSubject`，包含 `userId`、`organizationId`、`projectId`
- 通过 `ability.can(action, resource)` 检查权限
- PAT 认证时，使用 `permissiveAbility`（OSS 版本无角色限制）
- 支持插件化的 RBAC 实现（云版本）

---

## 5. 安全风险与建议

### 5.1 当前架构风险点

| 风险 | 严重度 | 说明 |
|------|-------|------|
| 全局单一加密密钥 | 中 | 所有租户共享 `ENCRYPTION_KEY`，密钥泄露影响所有凭据。数据库泄露 + 密钥泄露 = 所有凭据明文暴露 |
| Slack Token 未实现自动刷新 | 中 | `refreshToken` 已存储但未使用，user token 过期后集成静默失败 |
| 集成删除未清理 SecretStore | 低 | 软删除（`deletedAt`）后加密凭据仍在 SecretStore 中，虽然无法通过正常路径访问 |
| OAuth state 验证不一致 | 低 | Vercel 使用 JWT 验证 state，Slack 使用 Session cookie，安全性不同 |
| OrganizationAccessToken 未加密 | 低 | 仅存哈希，但原始 token 只在创建时返回一次，类似密码存储模式 |

### 5.2 金融客户加固建议

1. **密钥层级化**：为每个 Organization 生成独立的 KEK（Key Encryption Key），用 KEK 加密该组织的凭据。主密钥（ENCRYPTION_KEY）只加密 KEK，实现密钥隔离
2. **HSM/KMS 集成**：将 `ENCRYPTION_KEY` 托管至 AWS KMS / HashiCorp Vault，实现密钥轮换和审计
3. **实现 Slack Token 自动刷新**：在 `getAuthenticatedClientForIntegration` 中实现基于 `refreshToken` 的自动刷新逻辑
4. **集成删除时清理凭据**：在软删除 `OrganizationIntegration` 时，同步删除 `SecretStore` 和 `SecretReference` 中的记录
5. **审计日志**：对凭据的读取、解密操作记录审计日志，包含操作者、时间、目标集成
6. **传输加密**：确保数据库连接使用 SSL/TLS（PostgreSQL `sslmode=require`）
7. **密钥轮换**：实现 `ENCRYPTION_KEY` 轮换机制，新数据用新密钥加密，旧数据保留旧密钥版本标记（当前 version 字段已预留此能力）

---

## 6. 代码索引

| 组件 | 文件路径 |
|------|---------|
| 加密/解密核心 | [secretStore.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/secrets/secretStore.server.ts) |
| Token 工具函数 | [tokens.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/utils/tokens.server.ts) |
| Slack 集成 | [orgIntegration.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/orgIntegration.server.ts) |
| Vercel 集成 | [vercelIntegration.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/vercelIntegration.server.ts) |
| Vercel OAuth State | [vercelOAuthState.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/v3/vercel/vercelOAuthState.server.ts) |
| PAT 服务 | [personalAccessToken.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/personalAccessToken.server.ts) |
| OAT 服务 | [organizationAccessToken.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/organizationAccessToken.server.ts) |
| API 认证 | [apiAuth.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/apiAuth.server.ts) |
| RBAC | [fallback.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/internal-packages/rbac/src/fallback.ts) |
| 环境变量加密存储 | [environmentVariablesRepository.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts) |
| 集成回调路由 | [integrations.$serviceName.callback.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/integrations.$serviceName.callback.ts) |
| Vercel 回调路由 | [vercel.callback.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.callback.ts) |
| Vercel 连接路由 | [vercel.connect.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.connect.tsx) |
| 环境变量定义 | [env.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/env.server.ts) |
| 数据库 Schema | [schema.prisma](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/internal-packages/database/prisma/schema.prisma) |
