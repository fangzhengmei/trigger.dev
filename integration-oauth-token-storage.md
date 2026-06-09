# Trigger.dev 第三方 OAuth 凭据存储安全分析

> 本文档面向金融客户安全部门，基于代码逐层剖析 trigger.dev 如何处理第三方 OAuth 凭据的授权、加密存储、运行时取用、过期与撤销，以及多用户多环境下的权限隔离。

---

## 1. 端到端 OAuth 授权链路概览

trigger.dev 当前支持两类第三方集成 OAuth：**Slack** 和 **Vercel**。整体链路如下：

```
Slack 安装流程：
  Dashboard 点击"连接 Slack"
  → 后端生成 OAuth URL（state = 裸 organizationId）
  → 用户在 Slack 授权
  → Slack 回调带 code + state
  → 后端验证用户登录 + 组织成员身份
  → 用 code 换取 access_token（服务端到服务端）
  → access_token 经 AES-256-GCM 加密后存入 SecretStore
  → 数据库记录 SecretReference → OrganizationIntegration 关联

Vercel 安装流程（两条入口，state 生成时机不同）：

  入口 A — Dashboard：
    Dashboard 点击"连接 Vercel"
    → 后端生成 OAuth URL（state = JWT，含 orgId/projectId/envSlug）
    → 用户在 Vercel 授权
    → Vercel 回调带 code + state + configurationId?
    → 后端验证 state JWT 签名 + 用户权限
    → 用 code 换取 access_token → 加密存入 SecretStore

  入口 B — Marketplace：
    用户从 Vercel Marketplace 安装
    → Vercel 回调带 code + configurationId（无 state）
    → 跳转 Onboarding 页 → 用户选组织/项目
    → 选择项目后，后端生成 state JWT
    → 带 state + code + configurationId 重定向到 /vercel/connect
    → 后端验证 state JWT 签名 + 用户权限
    → 用 code 换取 access_token → 加密存入 SecretStore

运行时取用：
  任务运行时按需从 SecretStore 取出解密 → 构造第三方 SDK 客户端
```

### 1.1 授权发起——各入口的 state 机制差异

| 入口 | 授权 URL 生成 | 回调携带参数 | state 参数 | 回跳地址保存 |
|------|-------------|-------------|-----------|-------------|
| Slack（Dashboard） | [slackAuthorizationUrl](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/orgIntegration.server.ts#L116-L133) | `code + state` | **裸字符串** = `organizationId`，授权前生成 | Session cookie（`REDIRECT_AFTER_AUTH_KEY`） |
| Vercel（Dashboard） | [vercelInstallUrl](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/orgIntegration.server.ts#L108-L114) | `code + state + configurationId?` | **JWT**，含 orgId/projectId/envSlug，15分钟过期，授权前生成 | 无需 Session，state JWT 内自包含 |
| Vercel（Marketplace） | 用户从 Vercel Marketplace 发起 | `code + configurationId`（**无 state**） | **无**——回调时不携带 state；用户在 [onboarding](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.onboarding.tsx#L249-L258) 选择项目后才生成 state JWT | 无需 Session，state JWT 在 onboarding 中生成后传递 |

**关键区分**：三种入口的 state 生成时机和安全保障完全不同。

- **Slack**：`state` 是一个**裸 organizationId 字符串**，直接拼进 OAuth URL。它**不做签名校验**，Slack 回调时将其原样带回，后端仅依赖两点保证安全：
  1. `requireUserId` 确认当前用户已登录
  2. [CreateOrgIntegrationService.call()](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/v3/services/createOrgIntegration.server.ts#L6-L29) 查库时验证 `org.members: { some: { userId } }`，即用户必须是该组织成员

  回跳地址通过 [Session cookie](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/orgIntegration.server.ts#L141-L142) 中的 `REDIRECT_AFTER_AUTH_KEY` 保存，回调后由 [redirectAfterAuth](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/orgIntegration.server.ts#L165-L187) 读取并跳转，随后从 Session 中清除。

- **Vercel（Dashboard 入口）**：`state` 在授权前由 [generateVercelOAuthState](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/v3/vercel/vercelOAuthState.server.ts#L15-L22) 生成，使用 `ENCRYPTION_KEY` 签名的 JWT，15 分钟过期，payload 包含 `organizationId`、`projectId`、`environmentSlug`、`organizationSlug`、`projectSlug`。回调时由 [validateVercelOAuthState](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/v3/vercel/vercelOAuthState.server.ts#L25-L39) 验证签名和过期时间，然后根据 JWT 内的 `projectId` 再次查库验证用户权限。

- **Vercel（Marketplace 入口）**：用户从 Vercel Marketplace 安装时，回调**不携带 state**，只有 `code + configurationId`。[vercel.callback.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.callback.ts#L61-L74) 将其重定向到 [onboarding 页面](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.onboarding.tsx)，用户选择组织→项目（每步验证成员身份）后，[onboarding action](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.onboarding.tsx#L249-L258) 才生成 state JWT，然后带 `state + code + configurationId` 重定向到 `/vercel/connect`。这意味着 Marketplace 入口的 state **在回调之后、用户选择项目之后才生成**，而非授权前。

  两种 Vercel 入口最终都汇聚到 `/vercel/connect`，由相同的 state JWT 验证逻辑保障安全，都不依赖 Session。

### 1.2 回调处理与 Token 交换

**Slack 回调**：[integrations.$serviceName.callback.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/integrations.$serviceName.callback.ts#L21-L66)

1. 验证用户登录状态（`requireUserId`）
2. 解析 URL 参数中的 `code` 和 `state`（`state` = `organizationId`）
3. 调用 `CreateOrgIntegrationService.call(userId, state, serviceName, code)`
4. 服务内部验证用户是 `state` 所指组织的成员
5. 使用 Slack SDK `client.oauth.v2.access()` 用 code 换取 token（服务端到服务端，client_secret 不暴露给前端）
6. 创建成功后，从 Session 读取 `REDIRECT_AFTER_AUTH_KEY` 回跳

**Vercel 回调**：[vercel.callback.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.callback.ts#L21-L78) → [vercel.connect.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.connect.tsx#L66-L170)

Vercel 回调有两种入口，根据是否携带 `state` 参数分叉：

1. **Dashboard 入口（有 state）**：回调带 `state + code + configurationId?`，直接跳转 `/vercel/connect`
2. **Marketplace 入口（无 state，有 configurationId）**：回调带 `code + configurationId`，先跳转 `/vercel/onboarding` 让用户选择组织/项目，选择后生成 state JWT，再重定向到 `/vercel/connect`

在 `/vercel/connect` 中：
1. 验证用户登录状态
2. 验证 state JWT 签名与过期
3. 从 state JWT 中提取 `projectId`、`organizationId`，查库验证用户对 project 有权限
4. 调用 `VercelIntegrationRepository.exchangeCodeForToken(code)` 用 code 换取 access_token
5. 调用 `createOrFindVercelIntegration()` 创建或更新集成记录，`configurationId` 作为 `installationId` 存入密文

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

## 2. 第三方 OAuth 凭据 vs 内部 Token：代码层面的区分

trigger.dev 的凭据体系包含两类性质完全不同的 token，在代码中通过**存储模型、认证入口、前缀匹配**三个维度严格区分。

### 2.1 分类总览

| 维度 | 第三方 OAuth 凭据（Slack / Vercel） | 内部 Token（PAT / OAT / API Key） |
|------|-------------------------------------|-----------------------------------|
| **用途** | 代表组织访问第三方 API | 代表用户/组织/环境访问 trigger.dev 自身 API |
| **存储模型** | `OrganizationIntegration` → `SecretReference` → `SecretStore` | PAT: `PersonalAccessToken.encryptedToken` 列；OAT: `OrganizationAccessToken.hashedToken` 列；API Key: `RuntimeEnvironment.apiKey` 列 |
| **加密方式** | AES-256-GCM（通过 SecretStore） | PAT: AES-256-GCM（通过 `encryptToken` 工具函数）；OAT: 仅 SHA-256 哈希；API Key: 明文 |
| **认证入口** | 运行时由 `OrgIntegrationRepository.getAuthenticatedClientForIntegration()` 取用 | API 请求时由 [authenticateRequest](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/apiAuth.server.ts#L376-L438) 统一分派 |
| **前缀匹配** | 无前缀（不经过 Bearer 认证链路） | `tr_pat_` → PAT；`tr_oat_` → OAT；`tr_dev_`/`tr_prod_` → API Key |
| **撤销方式** | 软删除 `OrganizationIntegration.deletedAt` | PAT/OAT: `revokedAt` 时间戳；API Key: 归档环境 |

### 2.2 认证分派机制

[authenticateRequest](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/apiAuth.server.ts#L376-L438) 通过 token 前缀进行路由：

```typescript
// 简化后的分派逻辑
if (isPersonalAccessToken(apiKey)) {       // 前缀 "tr_pat_"
  → authenticateApiRequestWithPersonalAccessToken()
}
if (isOrganizationAccessToken(apiKey)) {   // 前缀 "tr_oat_"
  → authenticateApiRequestWithOrganizationAccessToken()
}
// 否则
→ authenticateApiKey()                     // 前缀 "tr_dev_" / "tr_prod_" 等
```

第三方 OAuth 凭据**从不进入此分派链路**——它们只在运行时通过 `OrgIntegrationRepository` 或 `VercelIntegrationRepository` 按需取出，构造第三方 SDK 客户端。

### 2.3 存储架构差异

```
┌─────────────────────────────────────────────────────────────────────┐
│ 第三方 OAuth 凭据                                                     │
│                                                                     │
│  OrganizationIntegration.tokenReferenceId                           │
│    → SecretReference.id (key = friendlyId, provider = "DATABASE")   │
│      → SecretStore.key (密文: { nonce, ciphertext, tag })           │
│                                                                     │
│  特征: 三层间接引用，密文与业务数据分离                                  │
├─────────────────────────────────────────────────────────────────────┤
│ 内部 Token                                                          │
│                                                                     │
│  PAT: PersonalAccessToken.encryptedToken (JSON 列, 直接存密文)       │
│       PersonalAccessToken.hashedToken  (SHA-256, 用于查找)           │
│       PersonalAccessToken.obfuscatedToken (UI 脱敏显示)              │
│                                                                     │
│  OAT: OrganizationAccessToken.hashedToken (SHA-256, 无加密)          │
│                                                                     │
│  API Key: RuntimeEnvironment.apiKey (明文, 无加密)                   │
│                                                                     │
│  特征: 各自有独立表和列，不经过 SecretStore                           │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.4 安全边界说明

- **PAT 使用与 OAuth 凭据相同的 AES-256-GCM 算法**，但密文直接存在 `PersonalAccessToken.encryptedToken` 列中，而非通过 SecretStore。这是因为 PAT 的认证流程（[authenticatePersonalAccessToken](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/personalAccessToken.server.ts#L236-L272)）需要同时读取 `hashedToken`（用于查找）和 `encryptedToken`（用于解密比对），将两者放在同一行更高效。

- **OAT 不加密**：`OrganizationAccessToken` 仅存储 `hashedToken`（SHA-256），不存原始 token。认证时只需比对哈希值，无需解密。这种设计与密码存储类似——原始 token 只在创建时返回一次。

- **API Key 明文存储**：`RuntimeEnvironment.apiKey` 在数据库中是明文。这是因为 API Key 需要频繁查询且需在 Dashboard 中展示，其安全边界依赖于数据库访问控制和传输加密。

---

## 3. 加密体系分析

### 3.1 加密算法：AES-256-GCM

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

### 3.2 密钥粒度

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

### 3.3 加密边界

```
┌─────────────────────────────────────────────────────────┐
│ 应用内存（明文）                                          │
│   - token 交换后短暂存在                                   │
│   - 运行时取出解密后短暂存在                                │
├─────────────────────────────────────────────────────────┤
│ SecretStore 表 / encryptedToken 列（密文）                 │
│   - AES-256-GCM 加密                                      │
│   - 存储 { nonce, ciphertext, tag } 结构                   │
│   - 覆盖：OAuth 凭据、环境变量 Secret、MFA 密钥、PAT       │
├─────────────────────────────────────────────────────────┤
│ RuntimeEnvironment.apiKey / RevokedApiKey.apiKey（明文）   │
│   - ⚠️ API Key 以明文存储，无加密无哈希                    │
│   - RuntimeEnvironment.apiKey 有 @@unique 约束            │
│   - RevokedApiKey.apiKey 有 @@index 索引，含 24h 过期      │
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
- 数据库中**第三方 OAuth 凭据和 PAT 不存在明文**，但 API Key（`RuntimeEnvironment.apiKey`）和已撤销 API Key（`RevokedApiKey.apiKey`）是**明文存储**的例外
- `PersonalAccessToken` 的 `obfuscatedToken` 列仅显示脱敏格式（如 `tr_pat_bhbd•••••••••••••••••••fd4a`）

### 3.4 SecretStore 版本演进

[secretStore.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/secrets/secretStore.server.ts#L86-L88) 中存在版本兼容逻辑：

- **version "1"**：明文 JSON 存储（历史遗留）
- **version "2"**：AES-256-GCM 加密存储（当前版本）

新写入始终使用 version "2"。读取时自动判断版本，version "1" 直接解析 JSON，version "2" 先解密再解析。

### 3.5 备选存储后端

[SecretStoreProvider](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/secrets/secretStore.server.ts#L200-L217) 枚举支持：

| Provider | 状态 | 说明 |
|----------|------|------|
| `DATABASE` | ✅ 已实现 | Postgres + AES-256-GCM |
| `AWS_PARAM_STORE` | ⚠️ 枚举已定义，未实现 | 可扩展为 AWS SSM Parameter Store |

---

## 4. Token 过期与刷新路径

### 4.1 Slack Token

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

### 4.2 Vercel Token

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

### 4.3 Personal Access Token

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

### 4.4 Organization Access Token

[OrganizationAccessToken](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/organizationAccessToken.server.ts#L92-L136) 支持**过期**和**撤销**两种失效机制：

- `expiresAt`：可选的过期时间，认证时 `WHERE expiresAt IS NULL OR expiresAt >= NOW()`
- `revokedAt`：撤销时间戳，认证时 `WHERE revokedAt IS NULL`

**注意**：OrganizationAccessToken **不加密存储**，仅存储 `hashedToken`（SHA-256 哈希），类似密码的存储方式。原始 token 只在创建时返回一次。

---

## 5. 集成撤销路径详解

### 5.1 Vercel：外部卸载 → 本地软删除

Vercel 集成的撤销是**双向操作**，分两步执行：

**步骤 1：调用 Vercel API 删除外部配置**

[uninstallVercelIntegration](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/vercelIntegration.server.ts#L1777-L1818)：

1. 从 SecretStore 解密取出 `VercelSecret`，获取 `installationId`
2. 调用 `client.integrations.deleteConfiguration({ id: installationId })` 删除 Vercel 侧配置
3. 若 Vercel 返回 401/403（token 已失效），标记 `authInvalid: true` 但**不中断流程**，仍继续本地清理

**步骤 2：本地数据库软删除**

[settings.integrations.vercel.tsx action](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.integrations.vercel.tsx#L163-L178)：

```typescript
await $transaction(prisma, async (tx) => {
  // 1. 软删除所有关联的项目集成
  await tx.organizationProjectIntegration.updateMany({
    where: { organizationIntegrationId: vercelIntegration.id, deletedAt: null },
    data: { deletedAt: new Date() },
  });
  // 2. 软删除组织级集成
  await tx.organizationIntegration.update({
    where: { id: vercelIntegration.id },
    data: { deletedAt: new Date() },
  });
});
```

**⚠️ 重要：SecretStore 中的加密凭据未被清理**。软删除后：
- `OrganizationIntegration` 记录仍存在（`deletedAt` 非空）
- `SecretReference` 和 `SecretStore` 中的密文仍存在
- 但正常查询路径（`where: { deletedAt: null }`）无法再获取到该集成，因此密文无法被解密取用

此外，在项目级别还有一个独立的断开操作：[disconnectVercelProject](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/vercelIntegration.server.ts#L717-L731)，仅软删除 `OrganizationProjectIntegration`（项目-集成关联），不影响组织级集成和 token。

### 5.2 Slack：仅本地软删除

[settings.integrations.slack.tsx action](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.integrations.slack.tsx#L108-L182)：

Slack 的撤销**不调用 Slack API**（不会在 Slack 侧卸载应用），仅在本地执行软删除：

```typescript
await $transaction(prisma, async (tx) => {
  // 1. 禁用所有关联的 Slack 告警通道
  await tx.projectAlertChannel.updateMany({
    where: { type: "SLACK", OR: [...] },
    data: { enabled: false, integrationId: null },
  });
  // 2. 软删除组织级集成
  await tx.organizationIntegration.update({
    where: { id: slackIntegration.id },
    data: { deletedAt: new Date() },
  });
});
```

**与 Vercel 的关键差异**：
- Slack 不调用第三方 API 撤销 token——token 在 Slack 侧仍然有效
- Slack 的 `authInvalid` 检测机制不存在（不像 Vercel 有 `validateVercelToken`）
- SecretStore 中的加密凭据同样未被清理

### 5.3 撤销路径汇总

| 凭据类型 | 外部撤销 | 本地撤销 | SecretStore 清理 |
|---------|---------|---------|-----------------|
| Slack OAuth Token | ❌ 不调用 Slack API | ✅ 软删除 `OrganizationIntegration` + 禁用告警通道 | ❌ 未清理 |
| Vercel OAuth Token | ✅ 调用 `deleteConfiguration` API | ✅ 软删除 `OrganizationIntegration` + `OrganizationProjectIntegration` | ❌ 未清理 |
| Personal Access Token | 不适用 | ✅ `revokedAt` 时间戳 | 不适用（密文在同一行） |
| Organization Access Token | 不适用 | ✅ `revokedAt` 时间戳 | 不适用（仅存哈希） |
| 环境变量 Secret | 不适用 | ✅ 硬删除 `EnvironmentVariableValue` + `SecretReference` + `SecretStore` | ✅ 已清理 |

**安全边界说明**：环境变量是唯一在删除时**完整清理 SecretStore** 的凭据类型（见 [environmentVariablesRepository.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts#L346-L356)）。OAuth 集成凭据在软删除后残留于 SecretStore 中，构成潜在的合规风险。

---

## 6. 多用户多环境权限隔离

### 6.1 组织（Organization）级别隔离

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

### 6.2 集成安装权限控制

**Slack 集成安装**（[CreateOrgIntegrationService](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/v3/services/createOrgIntegration.server.ts#L6-L29)）：

1. 验证 `userId` 是 `organization` 的成员（`members: { some: { userId } }`）
2. 只有组织成员才能为该组织安装集成

**Vercel 集成安装**（[vercel.connect.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.connect.tsx#L66-L170)）：

1. `requireUserId` 验证用户已登录
2. 验证 state JWT 签名和过期（防篡改 + 防重放）
3. 验证用户对 project 有访问权限（`organization: { members: { some: { userId } } }`）
4. 验证 environment 属于该 project

### 6.3 项目级集成关联

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

### 6.4 运行时凭据取用

任务运行时获取集成客户端的流程（以 Slack 为例）：

```
OrgIntegrationRepository.getAuthenticatedClientForIntegration()
  → getSecretStore(integration.tokenReference.provider)  // 按 provider 获取存储
  → secretStore.getSecret(SlackSecretSchema, integration.tokenReference.key)
  → 解密后构造 WebClient
```

**关键隔离保障**：
1. 集成记录通过 `organizationId` 隔离，不同组织的集成互不可见
2. `SecretReference.key` 使用 `friendlyId`（如 `org_integration_xxxx`），全局唯一且不可猜测
3. 凭据在内存中仅短暂存在（函数调用周期内），用后即弃

### 6.5 环境变量隔离

环境变量的加密存储通过 key 命名规则实现环境级隔离：

```typescript
// [environmentVariablesRepository.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts#L26-L36)
function secretKey(projectId: string, environmentId: string, key: string) {
  return `environmentvariable:${projectId}:${environmentId}:${key}`;
}
```

每个环境的变量值独立加密存储，同一个变量名在不同环境（dev/staging/prod）有不同的加密值。

### 6.6 API 认证与权限层级

API 请求的认证链路（[apiAuth.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/apiAuth.server.ts)）支持多种认证方式，每种方式绑定的隔离粒度不同：

| 认证方式 | 格式前缀 | 隔离粒度 | 存储方式 |
|---------|---------|---------|---------|
| API Key | `tr_dev_` / `tr_prod_` | 环境（Environment） | 明文存储，明文查找 |
| Personal Access Token | `tr_pat_` | 用户（User） | AES-256-GCM 加密 + SHA-256 哈希 |
| Organization Access Token | `tr_oat_` | 组织（Organization） | SHA-256 哈希（无加密） |
| Public JWT | - | 环境（Environment） | JWT 签名验证 |

### 6.7 RBAC 权限控制

[rbac/fallback.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/internal-packages/rbac/src/fallback.ts) 实现了基于角色的访问控制：

- 用户认证后构建 `RbacSubject`，包含 `userId`、`organizationId`、`projectId`
- 通过 `ability.can(action, resource)` 检查权限
- PAT 认证时，使用 `permissiveAbility`（OSS 版本无角色限制）
- 支持插件化的 RBAC 实现（云版本）

---

## 7. 安全边界深度分析

### 7.1 Vercel configurationId 进入 Onboarding 的完整流程

Vercel 集成的安装有两种入口，对应不同的 OAuth 回调路径：

**入口 A：Dashboard 发起（有 state）**

用户从 trigger.dev Dashboard 点击"连接 Vercel"，后端已生成 state JWT，Vercel 回调时携带 `state + code`：

```
Dashboard → vercelInstallUrl(state=JWT) → Vercel 授权
  → callback 带 state + code + configurationId?
  → redirect → /vercel/connect（直接创建集成）
```

**入口 B：Vercel Marketplace 发起（无 state，有 configurationId）**

用户从 Vercel Marketplace 安装，没有 trigger.dev 侧的 state JWT，Vercel 回调时只携带 `code + configurationId`：

```
Vercel Marketplace → callback 无 state, 有 code + configurationId
  → redirect → /vercel/onboarding（引导选组织/项目）
```

[vercel.callback.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.callback.ts#L61-L74) 的路由逻辑：

```typescript
// 有 state：dashboard 流程，直接跳转 /vercel/connect
if (state) {
  const params = new URLSearchParams({ state, code, origin: "dashboard" });
  if (configurationId) params.set("configurationId", configurationId);
  return redirect(`/vercel/connect?${params.toString()}`);
}

// 无 state 但有 configurationId：marketplace 流程，跳转 /vercel/onboarding
if (configurationId) {
  const params = new URLSearchParams({ code, configurationId, origin: "marketplace" });
  return redirect(`/vercel/onboarding?${params.toString()}`);
}
```

**Marketplace 流程的 onboarding 步骤**（[vercel.onboarding.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.onboarding.tsx)）：

1. 用户选择组织 → 选择项目（每步都验证用户是该组织成员）
2. 选择项目后，后端**生成 state JWT**（[vercel.onboarding.tsx#L249-L258](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.onboarding.tsx#L249-L258)）：

```typescript
const stateResult = await generateVercelOAuthState({
  organizationId: project.organizationId,
  projectId: project.id,
  environmentSlug: environment.slug,    // 固定为 "prod"
  organizationSlug: project.organization.slug,
  projectSlug: project.slug,
});
```

3. 带 `state + code + configurationId + origin=marketplace` 重定向到 `/vercel/connect`
4. `/vercel/connect` 验证 state JWT 签名、过期、用户权限，然后交换 token 并创建集成

**安全边界**：Marketplace 流程中，`configurationId` 从 Vercel 回调一路传递到 `createOrFindVercelIntegration`，最终存入 `VercelSecret.installationId`。`configurationId` 本身不是 secret（它是 Vercel 侧的集成配置 ID），但它在 onboarding 各步骤间通过隐藏表单字段传递。

**⚠️ configurationId 与 code 在本地并未绑定校验**：从 [vercel.callback.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.callback.ts#L61-L74) 到 [vercel.onboarding.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.onboarding.tsx) 再到 [vercel.connect.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.connect.tsx#L66-L170)，`code` 和 `configurationId` 始终作为**独立参数**传递，本地代码从未校验两者是否属于同一个 Vercel 安装。具体来说：

1. `exchangeCodeForToken(code)` 只使用 `code`，不涉及 `configurationId`
2. `configurationId` 仅在 `createOrFindVercelIntegration` 中作为 `installationId` 存入 `VercelSecret`
3. 没有任何代码检查"该 `code` 换来的 token 是否确实属于 `configurationId` 所指的配置"

实际安全保障依赖 Vercel 侧的 OAuth 协议正确性：Vercel 在用户授权时将 `code` 和 `configurationId` 绑定到同一个安装会话中，回传时它们自然对应。但如果攻击者能同时获得一个有效的 `code`（来自安装 A）和一个不同的 `configurationId`（来自安装 B），本地代码无法检测这种不匹配——安装 B 的 `configurationId` 会被存入密文，而 token 来自安装 A。

### 7.2 Slack State 未与 Session 绑定的风险分析

Slack 的 `state` 参数是裸 `organizationId`，**不与 Session 绑定**。这意味着：

**攻击场景：CSRF 式集成安装**

```
攻击者构造 URL:
  https://slack.com/oauth/v2/authorize?...&state=VICTIM_ORG_ID

攻击者诱导受害者（VICTIM_ORG 的成员）点击该 URL
  → 受害者在 Slack 授权
  → Slack 回调 /integrations/slack/callback?code=xxx&state=VICTIM_ORG_ID
  → CreateOrgIntegrationService.call(受害者userId, VICTIM_ORG_ID, "slack", code)
  → 成员校验通过 → 集成创建成功
```

**此攻击的效果**：攻击者可以诱导受害者把某个 Slack workspace 安装到受害者的组织中。不过：

1. **攻击者无法获取凭据**：凭据存储在 SecretStore 中，攻击者无法读取
2. **攻击者无法直接控制安装哪个 Slack workspace**：OAuth 授权页面由 Slack 展示给受害者，受害者看到并授权的是自己有权限的 workspace
3. **实际威胁有限**：攻击者最多能让受害者组织连接一个"不想要的" Slack workspace，但不会导致凭据泄露

**更严重但无法实现的攻击场景（理论上）**：

如果攻击者能获取一个有效的 Slack `code`，理论上可以构造：

```
攻击者自己的 Slack workspace 授权 → 获得 code
构造回调 URL → /integrations/slack/callback?code=ATTACKER_CODE&state=VICTIM_ORG_ID
发送给受害者浏览器访问
```

但此攻击**不可行**，因为 Slack 的 `code` 只能使用一次且绑定 OAuth 应用与 `redirect_uri`；攻击者即使在自己的浏览器触发授权，也无法在不消耗该 code 的情况下把同一个 code 交给受害者会话复用。

**对比 Vercel 的安全性**：Vercel 的 state JWT 包含 `projectId`、`organizationSlug` 等信息并用 `ENCRYPTION_KEY` 签名，攻击者无法篡改 state 中的目标组织/项目，也无法伪造有效的 state。

**加固建议**：将 Slack 的 state 也改为 JWT 签名方式，并在 state 中绑定 `userId`（或至少绑定 Session ID），使回调时可以验证 state 的发出者和当前用户一致。

### 7.3 API Key 与 RevokedApiKey 的明文查找机制

[findEnvironmentByApiKey](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/runtimeEnvironment.server.ts#L94-L130) 的认证流程：

```typescript
// 1. 先在 RuntimeEnvironment 表按明文 apiKey 查找
let environment = await $replica.runtimeEnvironment.findFirst({
  where: { apiKey },
  include,
});

// 2. 找不到 → 在 RevokedApiKey 表按明文 apiKey 查找（宽限期内仍可用）
if (!environment) {
  const revokedApiKey = await $replica.revokedApiKey.findFirst({
    where: {
      apiKey,
      expiresAt: { gt: new Date() },
    },
    include: { runtimeEnvironment: { include } },
  });
  environment = revokedApiKey?.runtimeEnvironment ?? null;
}
```

**确认：API Key 认证完全基于明文查找，无哈希。**

这意味着：

| 方面 | 说明 |
|------|------|
| 查找方式 | `WHERE apiKey = 'tr_prod_xxxxx'`，直接明文匹配 |
| RevokedApiKey | 同样明文存储，额外有 `expiresAt` 过滤（24小时宽限期） |
| 数据库索引 | `RuntimeEnvironment.apiKey` 有 `@@unique` 约束，`RevokedApiKey.apiKey` 有 `@@index` |
| 风险 | 数据库泄露 → 所有 API Key 明文暴露，可立即用于 API 认证 |

**与 PAT/OAT 的对比**：

| Token 类型 | 存储方式 | 数据库泄露影响 |
|-----------|---------|-------------|
| API Key | 明文 | ⚠️ 可直接使用 |
| RevokedApiKey | 明文（24h TTL） | ⚠️ 宽限期内可直接使用 |
| PAT | AES-256-GCM 加密 + SHA-256 哈希 | ✅ 需 ENCRYPTION_KEY 才能解密原始 token |
| OAT | 仅 SHA-256 哈希 | ✅ 不可逆，无法还原原始 token |
| OAuth 凭据 | AES-256-GCM 加密 | ✅ 需 ENCRYPTION_KEY 才能解密 |

**RevokedApiKey 的宽限机制**：

当用户重新生成 API Key 时（[api-key.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/api-key.server.ts#L64-L88)）：

1. 旧 apiKey 明文写入 `RevokedApiKey` 表，`expiresAt = now + 24h`
2. `RuntimeEnvironment.apiKey` 更新为新 key
3. 旧 key 在 24 小时宽限期内仍可通过 `RevokedApiKey` 表认证成功

**安全边界**：宽限期的设计目的是让正在运行的 worker 在 key 轮换后仍能短暂工作，避免中断。但这意味着数据库泄露后，即使管理员立即轮换了 API Key，旧 key 在 24 小时内仍然有效。

### 7.4 Slack 重装 vs Vercel 按 teamId 更新 Token

两种集成在"重新安装"场景下的行为完全不同：

**Slack：每次都创建新集成记录**

[createOrgIntegration](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/orgIntegration.server.ts#L189-L267) 中，Slack 流程**没有"查找已有"逻辑**：

```
用户点击 reinstall=true
  → 跳过 "已有集成则重定向" 的检查
  → 重新走完整 OAuth 流程
  → createOrgIntegration() → 总是创建新的 OrganizationIntegration
     → 新的 friendlyId
     → 新的 SecretReference
     → 新的 SecretStore 记录
     → 新的 OrganizationIntegration 行
```

结果：同一组织可以有**多条** `service: "SLACK"` 的 `OrganizationIntegration` 记录（旧记录的 `deletedAt` 可能为 null 或有值）。代码中查询 Slack 集成使用 `findFirst` 且没有显式排序，实际会拿到某一条活跃记录，不能保证一定是最近安装的那条。

**旧凭据未清理**：重装后，旧的 SecretStore 和 SecretReference 记录仍然存在，旧的加密凭据未被删除。

**Vercel：应用层按 teamId 查找并更新（非数据库约束）**

[createOrFindVercelIntegration](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.connect.tsx#L21-L64) 中，Vercel 流程**先查找已有集成**：

```typescript
let orgIntegration = await VercelIntegrationRepository.findVercelOrgIntegrationByTeamId(
  organizationId,
  tokenResponse.teamId ?? null
);

if (orgIntegration) {
  // 已有 → 更新 token
  await VercelIntegrationRepository.updateVercelOrgIntegrationToken({ ... });
} else {
  // 没有 → 创建新集成
  await VercelIntegrationRepository.createVercelOrgIntegration({ ... });
}
```

[findVercelOrgIntegrationByTeamId](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/vercelIntegration.server.ts#L859-L874) 按 `externalOrganizationId = teamId` 查找：

```typescript
return prisma.organizationIntegration.findFirst({
  where: {
    organizationId,
    service: "VERCEL",
    externalOrganizationId: teamId,
    deletedAt: null,
  },
});
```

⚠️ **这是应用层面的 `findFirst` 查询，不是数据库唯一约束**。`OrganizationIntegration` 表上没有 `(organizationId, service, externalOrganizationId)` 的 `@@unique` 约束——只有 `friendlyId` 有 `@unique`。这意味着：

- 正常情况下 `findFirst` 会找到已有记录并更新，行为看起来是幂等的
- 但如果存在并发安装、或数据库中已有脏数据（如同一 teamId 有多条未删除记录），`findFirst` 只会更新其中一条，其余成为孤立记录
- 这与 Slack 重装的问题本质相同——区别仅在于 Vercel **尝试**查找已有记录，而 Slack **完全不查找**

[updateVercelOrgIntegrationToken](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/vercelIntegration.server.ts#L753-L799) 更新时：

1. 通过现有的 `SecretReference.key` 找到 SecretStore 中的旧凭据
2. 用新 token **覆盖**旧凭据（`secretStore.setSecret`）
3. 更新 `integrationData` 中的元数据

**关键区别**：

| 维度 | Slack 重装 | Vercel 重装 |
|------|-----------|------------|
| 策略 | 总是创建新记录 | 应用层按 teamId `findFirst` 查找并更新 |
| 查找机制 | 无查找逻辑 | `findFirst({ organizationId, service: "VERCEL", externalOrganizationId: teamId, deletedAt: null })` |
| 数据库约束保障 | 无 | 无——`OrganizationIntegration` 表无 `(organizationId, service, externalOrganizationId)` 唯一约束 |
| 旧凭据 | 保留在 SecretStore 中（孤立） | 被覆盖（同一 SecretStore key 重写），但并发场景下仍可能产生孤立记录 |
| 幂等性 | ❌ 不幂等 | ⚠️ 条件幂等——依赖 `findFirst` 找到唯一匹配记录 |
| 凭据残留 | ⚠️ 有——旧凭据成为孤立数据 | ✅ 正常情况下无，但无约束保障 |

**安全边界**：`findVercelOrgIntegrationByTeamId` 的查询条件包含 `organizationId`，因此不同 trigger.dev 组织之间的 Vercel 集成记录是隔离的——即使两个 trigger.dev 组织连接了同一个 Vercel team，也不会在数据库层面互相覆盖，因为 `findFirst` 限定在各自 `organizationId` 范围内查找。但这只是**应用层查询逻辑**的隔离，并非数据库唯一约束强制；如果绕过应用层直接操作数据库，同一 `(organizationId, teamId)` 仍可插入多条记录。

---

## 8. 安全风险与建议

### 8.1 当前架构风险点

| 风险 | 严重度 | 说明 |
|------|-------|------|
| 全局单一加密密钥 | 中 | 所有租户共享 `ENCRYPTION_KEY`，密钥泄露影响所有凭据。数据库泄露 + 密钥泄露 = 所有凭据明文暴露 |
| API Key 明文存储且按明文查找 | 中 | RuntimeEnvironment.apiKey 和 RevokedApiKey.apiKey 均为明文，数据库泄露即可直接使用所有 API Key |
| Slack Token 未实现自动刷新 | 中 | `refreshToken` 已存储但未使用，user token 过期后集成静默失败 |
| 集成软删除未清理 SecretStore | 中 | `OrganizationIntegration` 软删除后，`SecretStore` 和 `SecretReference` 中的密文仍残留。数据库泄露时这些密文可被解密（若有 ENCRYPTION_KEY） |
| Slack 重装产生孤立凭据 | 中 | 每次 Slack 重装都创建新记录，旧 SecretStore 凭据未清理，形成不可控的凭据残留 |
| Slack 撤销不调用第三方 API | 中 | Slack 集成删除后，token 在 Slack 侧仍有效，存在被滥用的窗口期 |
| Vercel configurationId 与 code 未绑定校验 | 低 | Marketplace 流程中 `code` 和 `configurationId` 作为独立参数传递，本地不校验两者是否属于同一个 Vercel 安装。实际安全依赖 Vercel 侧协议正确性 |
| Slack state 无签名校验 | 低 | Slack 的 `state` 参数是裸 `organizationId`，不与 Session 绑定，依赖后续成员校验保证安全。CSRF 式安装攻击理论上可行但实际威胁有限 |
| OAuth state 验证机制不一致 | 低 | Vercel 使用签名 JWT 验证 state，Slack 依赖 Session + 成员校验，两种模式安全基线不同 |
| Vercel 重装无数据库唯一约束 | 低 | `OrganizationIntegration` 表无 `(organizationId, service, externalOrganizationId)` 唯一约束，`findFirst` 查找在并发或脏数据场景下可能产生多条记录 |
| RevokedApiKey 宽限期过长 | 低 | API Key 轮换后旧 key 仍有 24 小时有效期，数据库泄露场景下缩小了轮换的即时防护效果 |
| OrganizationAccessToken 未加密 | 低 | 仅存哈希，但原始 token 只在创建时返回一次，类似密码存储模式 |

### 8.2 金融客户加固建议

1. **密钥层级化**：为每个 Organization 生成独立的 KEK（Key Encryption Key），用 KEK 加密该组织的凭据。主密钥（ENCRYPTION_KEY）只加密 KEK，实现密钥隔离
2. **HSM/KMS 集成**：将 `ENCRYPTION_KEY` 托管至 AWS KMS / HashiCorp Vault，实现密钥轮换和审计
3. **API Key 改为哈希存储**：参考 PAT 的 `hashedToken` 模式，将 API Key 改为 SHA-256 哈希存储，仅在创建时返回明文
4. **缩短 RevokedApiKey 宽限期**：将 24 小时宽限期缩短至 5-10 分钟，或实现优雅滚动轮换（先创建新 key，再废弃旧 key）
5. **实现 Slack Token 自动刷新**：在 `getAuthenticatedClientForIntegration` 中实现基于 `refreshToken` 的自动刷新逻辑
6. **集成删除时清理凭据**：在软删除 `OrganizationIntegration` 时，同步删除 `SecretStore` 和 `SecretReference` 中的记录
7. **Slack 重装时查找并更新**：参考 Vercel 的 `createOrFindVercelIntegration` 模式，按 Slack team_id 查找已有集成并更新 token，而非创建新记录
8. **添加数据库唯一约束**：为 `OrganizationIntegration` 表添加 `(organizationId, service, externalOrganizationId, deletedAt)` 的部分唯一索引，从数据库层面保证同一组织同一 teamId 只有一条活跃集成
9. **Slack 撤销时调用 Slack API**：参考 Vercel 的 `uninstallVercelIntegration` 模式，在删除 Slack 集成时调用 Slack 的 `auth.revoke` API 使 token 失效
10. **统一 state 验证机制**：将 Slack 的 state 也改为 JWT 签名方式，并在 payload 中绑定 userId，与 Vercel 对齐
11. **Vercel configurationId 与 code 绑定校验**：在 `createOrFindVercelIntegration` 中，将 `configurationId` 存入密文后，可通过 Vercel API 调用 `getConfiguration({ id: configurationId })` 验证该配置的安装状态与当前 token 一致
12. **审计日志**：对凭据的读取、解密操作记录审计日志，包含操作者、时间、目标集成
13. **传输加密**：确保数据库连接使用 SSL/TLS（PostgreSQL `sslmode=require`）
14. **密钥轮换**：实现 `ENCRYPTION_KEY` 轮换机制，新数据用新密钥加密，旧数据保留旧密钥版本标记（当前 version 字段已预留此能力）

---

## 9. 代码索引

| 组件 | 文件路径 |
|------|---------|
| 加密/解密核心 | [secretStore.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/secrets/secretStore.server.ts) |
| Token 工具函数 | [tokens.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/utils/tokens.server.ts) |
| Slack 集成 | [orgIntegration.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/orgIntegration.server.ts) |
| Vercel 集成 | [vercelIntegration.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/vercelIntegration.server.ts) |
| Vercel OAuth State | [vercelOAuthState.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/v3/vercel/vercelOAuthState.server.ts) |
| PAT 服务 | [personalAccessToken.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/personalAccessToken.server.ts) |
| OAT 服务 | [organizationAccessToken.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/organizationAccessToken.server.ts) |
| Vercel 集成服务 | [vercelIntegration.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/vercelIntegration.server.ts) |
| API 认证 | [apiAuth.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/services/apiAuth.server.ts) |
| RBAC | [fallback.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/internal-packages/rbac/src/fallback.ts) |
| 环境变量加密存储 | [environmentVariablesRepository.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts) |
| 集成回调路由 | [integrations.$serviceName.callback.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/integrations.$serviceName.callback.ts) |
| Vercel 回调路由 | [vercel.callback.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.callback.ts) |
| Vercel 连接路由 | [vercel.connect.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.connect.tsx) |
| Slack 设置/撤销页 | [settings.integrations.slack.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.integrations.slack.tsx) |
| Vercel 设置/撤销页 | [settings.integrations.vercel.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/_app.orgs.$organizationSlug.settings.integrations.vercel.tsx) |
| Vercel Onboarding | [vercel.onboarding.tsx](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/routes/vercel.onboarding.tsx) |
| API Key 管理 | [api-key.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/api-key.server.ts) |
| 环境查找（含 RevokedApiKey） | [runtimeEnvironment.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/models/runtimeEnvironment.server.ts) |
| 环境变量定义 | [env.server.ts](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/apps/webapp/app/env.server.ts) |
| 数据库 Schema | [schema.prisma](file:///d:/fz/0508-3/solo-dogfeeding/code/189-trigger.dev/internal-packages/database/prisma/schema.prisma) |
