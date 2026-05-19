# 环境变量生效路径分析

本文档分析 Trigger.dev 平台中环境变量从面板编辑、保存到后端再注入到任务执行进程的完整路径，包括各层的覆盖与合并规则。

## 一、面板写入（前端）

### 1.1 UI 组件
**文件**: `apps/webapp/app/routes/_app.orgs.$organizationSlug.projects.$projectParam.env.$envParam.environment-variables/route.tsx`

前端页面提供以下功能：
- 显示环境变量列表，支持搜索过滤
- "Reveal values" 开关控制是否显示明文值
- 支持内联编辑（`EditEnvironmentVariablePanel` 组件）
- 支持删除操作（`DeleteEnvironmentVariableButton` 组件）
- 支持 Vercel 同步开关（如果启用了 Vercel 集成）

### 1.2 数据提交流程
```
用户编辑表单
  ↓
Remix action 处理 POST 请求
  ↓
调用 EnvironmentVariablesRepository.editValue() / deleteValue()
  ↓
返回结果，刷新页面数据
```

**关键代码** (`route.tsx:141-258`):
- action 函数处理三种操作：`edit`、`delete`、`update-vercel-sync`
- 使用 Zod schema 验证提交数据
- 操作成功后重定向或返回 JSON 结果

## 二、服务端校验与加密存储

### 2.1 核心 Repository
**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts`

`EnvironmentVariablesRepository` 类提供完整的 CRUD 操作。

#### 2.1.1 创建/编辑流程
```
输入校验
  ↓
移除黑名单变量 (removeBlacklistedVariables)
  ↓
过滤空值
  ↓
检查重复 (非 override 模式)
  ↓
事务处理:
  ├─ upsert EnvironmentVariable (项目级元数据)
  ├─ 检测是否继承父环境的 isSecret 属性
  ├─ 创建/更新 SecretReference
  ├─ 创建/更新 EnvironmentVariableValue
  └─ 加密存储值到 SecretStore
```

**密钥格式**:
```
environmentvariable:{projectId}:{environmentId}:{key}
```

#### 2.1.2 黑名单变量
**文件**: `apps/webapp/app/v3/environmentVariableRules.server.ts`

系统保留变量，用户无法设置：
- `TRIGGER_SECRET_KEY` - 精确匹配
- `TRIGGER_API_URL` - 精确匹配

### 2.2 加密存储
**文件**: `apps/webapp/app/services/secrets/secretStore.server.ts`

#### 2.2.1 加密方案
- **算法**: AES-256-GCM（认证加密）
- **密钥来源**: `env.ENCRYPTION_KEY`（16字节十六进制字符串）
- **存储结构**:
  ```json
  {
    "nonce": "12字节随机数(hex)",
    "ciphertext": "密文(hex)",
    "tag": "认证标签(hex)"
  }
  ```

#### 2.2.2 SecretStore 接口
```typescript
interface SecretStoreProvider {
  getSecret<T>(schema: z.Schema<T>, key: string): Promise<T | undefined>;
  getSecrets<T>(schema: z.Schema<T>, keyPrefix: string): Promise<{ key: string; value: T }[]>;
  setSecret<T extends object>(key: string, value: T): Promise<void>;
  deleteSecret(key: string): Promise<void>;
}
```

当前仅支持 `DATABASE` 提供者，存储在 Postgres 的 `SecretStore` 表中。

### 2.3 数据模型
```
EnvironmentVariable (项目级)
  ├─ id
  ├─ key (变量名)
  ├─ projectId
  └─ friendlyId

EnvironmentVariableValue (环境级)
  ├─ variableId
  ├─ environmentId
  ├─ valueReferenceId → SecretReference
  ├─ isSecret (是否为密钥)
  ├─ version (版本号，每次更新+1)
  └─ lastUpdatedBy (更新者信息)

SecretReference
  ├─ key (secretKey)
  └─ provider ("DATABASE")

SecretStore
  ├─ key (secretKey)
  ├─ value (加密的 JSON)
  └─ version ("2")
```

## 三、运行时解析与合并

### 3.1 核心解析函数
**文件**: `apps/webapp/app/v3/environmentVariables/environmentVariablesRepository.server.ts:899-933`

`resolveVariablesForEnvironment()` 是环境变量合并的核心入口。

### 3.2 合并优先级（从低到高）

```
1. overridableTriggerVariables
   └─ TRIGGER_REALTIME_STREAM_VERSION
   
2. overridableOtelVariables (仅开发环境)
   └─ OTEL_EXPORTER_OTLP_ENDPOINT

3. projectSecrets (用户设置的环境变量)
   ├─ 父环境变量 (如果存在)
   └─ 当前环境变量 (覆盖父环境相同key)
   * 注意: 会重命名 OTEL_RESOURCE_ATTRIBUTES → CUSTOM_OTEL_RESOURCE_ATTRIBUTES

4. builtInVariables (内置系统变量)
   ├─ 开发环境: TRIGGER_OTEL_*, TRIGGER_API_URL, TRIGGER_STREAM_URL 等
   └─ 生产环境: TRIGGER_SECRET_KEY, TRIGGER_API_URL, TRIGGER_ORG_ID 等
```

### 3.3 去重规则
**文件**: `apps/webapp/app/v3/deduplicateVariableArray.server.ts`

```typescript
/** 后出现的变量覆盖先出现的 */
function deduplicateVariableArray(variables: EnvironmentVariable[]) {
  // 反向遍历，保留后面的（优先级高的）
  for (const variable of [...variables].reverse()) {
    if (!result.some((v) => v.key === variable.key)) {
      result.push(variable);
    }
  }
  return result.reverse();
}
```

**关键特性**:
- 数组中后面的元素优先级更高
- 保留第一个出现的位置，但使用最后出现的值

### 3.4 父子环境继承
**文件**: `environmentVariablesRepository.server.ts:645-685`

```typescript
async #getSecretEnvironmentVariables(
  projectId: string,
  environmentId: string,
  parentEnvironmentId?: string
) {
  // 先加载父环境变量
  const parentSecrets = parentEnvironmentId ? await getSecrets(parentPrefix) : [];
  
  // 再加载子环境变量
  const childSecrets = await getSecrets(childPrefix);
  
  // 合并：子环境覆盖父环境
  const mergedSecrets = new Map<string, string>();
  for (const secret of parentSecrets) {
    mergedSecrets.set(parsedKey, secret.value.secret);
  }
  for (const secret of childSecrets) {
    mergedSecrets.set(parsedKey, secret.value.secret); // 覆盖
  }
}
```

### 3.5 内置变量覆盖机制
**文件**: `environmentVariablesRepository.server.ts:1336-1360`

部分内置变量支持通过 `RuntimeEnvironment.builtInEnvironmentVariableOverrides` 进行环境级覆盖：
- `TRIGGER_OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT`
- `TRIGGER_OTEL_LOG_ATTRIBUTE_COUNT_LIMIT`
- `TRIGGER_WAIT_UNTIL_TIMEOUT_MS`
- 等多个 OTEL 相关配置

## 四、API 暴露

### 4.1 外部 API
**文件**: `apps/webapp/app/routes/api.v1.projects.$projectRef.envvars.ts`

```
GET /api/v1/projects/:projectRef/envvars
  ↓
认证 (API Key)
  ↓
resolveVariablesForEnvironment()
  ↓
返回 { variables: Record<string, string> }
```

这是 CLI 和 SDK 获取环境变量的主要接口。

### 4.2 isSecret 变量的返回行为（关键细节）

**结论**: `/api/v1/projects/:projectRef/envvars` 返回 isSecret 变量的**明文值**。

**代码证据**:
- `api.v1.projects.$projectRef.envvars.ts:54-64` 调用 `resolveVariablesForEnvironment()`
- `resolveVariablesForEnvironment()` 调用 `environmentVariablesRepository.getEnvironmentVariables()`
- `getEnvironmentVariables()` 调用 `#getSecretEnvironmentVariables()`
- `#getSecretEnvironmentVariables()` (`environmentVariablesRepository.server.ts:645-685`) 直接从 secretStore 读取所有密钥的明文值，**没有做任何脱敏处理**
- 最后仅通过 `removeBlacklistedVariables()` 过滤黑名单变量（`TRIGGER_SECRET_KEY`、`TRIGGER_API_URL`）

**对比 UI 端**:
UI 端使用 `getEnvironmentWithRedactedSecrets()` 方法，该方法会将 isSecret 标记的变量值替换为 `<redacted>`。

**安全说明**: 该 API 受 API Key 保护，只有拥有有效 API Key 的可信客户端才能访问。

## 五、构建时注入

### 5.1 CLI 构建流程
**文件**: `packages/cli-v3/src/commands/workers/build.ts:200`

```
build 命令执行
  ↓
调用 GET /api/v1/projects/:projectRef/envvars
  ↓
获取服务器端环境变量
  ↓
传递给 buildWorker / buildImage
```

### 5.2 镜像构建时索引
**文件**: `packages/cli-v3/src/entryPoints/managed-index-controller.ts:66-78`

在 Docker 镜像构建过程中：
```typescript
// 获取环境变量
const $env = await cliApiClient.getEnvironmentVariables(projectRef);

// 索引时注入环境变量
const workerManifest = await indexWorkerManifest({
  env: $env.data.variables,  // 注入到 Node.js 进程
  // ...
});
```

## 六、任务执行时注入

### 6.1 环境变量获取链路（生产环境）

```
Worker 启动
  ↓
调用 startRunAttempt API (POST /engine/v1/worker-actions/runs/.../attempts/start)
  ↓
平台端 workerGroupTokenService.server.ts:409-457
  ├─ 调用 engine.startRunAttempt()
  ├─ 查询 RuntimeEnvironment（含 parentEnvironment）
  ├─ 调用 resolveVariablesForEnvironment() 获取用户变量 + 内置变量
  ├─ 生成 TRIGGER_JWT、TRIGGER_RUN_ID、TRIGGER_MACHINE_PRESET
  └─ 返回 { ...engineResult, envVars: Record<string, string> }
  ↓
Worker 收到 envVars，通过 IPC 传递给子进程
  ↓
子进程调用 populateEnv(env, { override: true })
```

### 6.2 Supervisor 容器创建
Supervisor 在创建任务执行容器时注入系统级环境变量：

#### Docker 方式
**文件**: `apps/supervisor/src/workloadManager/docker.ts:70-111`

```typescript
const envVars: string[] = [
  `OTEL_EXPORTER_OTLP_ENDPOINT=${env.OTEL_EXPORTER_OTLP_ENDPOINT}`,
  `TRIGGER_DEQUEUED_AT_MS=${opts.dequeuedAt.getTime()}`,
  `TRIGGER_POD_SCHEDULED_AT_MS=${Date.now()}`,
  `TRIGGER_ENV_ID=${opts.envId}`,
  `TRIGGER_DEPLOYMENT_ID=${opts.deploymentFriendlyId}`,
  `TRIGGER_RUN_ID=${opts.runFriendlyId}`,
  `TRIGGER_SUPERVISOR_API_DOMAIN=...`,
  `TRIGGER_MACHINE_CPU=${opts.machine.cpu}`,
  `TRIGGER_MACHINE_MEMORY=${opts.machine.memory}`,
  // ... 更多系统变量
];

// 支持 additionalEnvVars 扩展
if (this.opts.additionalEnvVars) {
  Object.entries(this.opts.additionalEnvVars).forEach(([key, value]) => {
    envVars.push(`${key}=${value}`);
  });
}
```

#### Kubernetes 方式
**文件**: `apps/supervisor/src/workloadManager/kubernetes.ts:136-255`

类似 Docker，通过 Pod spec 的 `env` 字段注入。

#### Compute 方式
**文件**: `apps/supervisor/src/workloadManager/compute.ts:71-110`

通过 `env` 对象传递给计算实例。

### 6.3 运行时进程填充
**文件**: `packages/core/src/v3/workers/populateEnv.ts`

```typescript
export function populateEnv(
  envObject: Record<string, string>,
  options: PopulateEnvOptions = {}
): void {
  const { override = false, debug = false, previousEnv } = options;

  for (const key of Object.keys(envObject)) {
    if (Object.prototype.hasOwnProperty.call(process.env, key)) {
      if (override) {
        process.env[key] = envObject[key]; // 覆盖
      }
    } else {
      process.env[key] = envObject[key]; // 设置
    }
  }

  // 清理 previousEnv 中有但 envObject 中没有的变量
  if (previousEnv) {
    for (const key of Object.keys(previousEnv)) {
      if (!Object.prototype.hasOwnProperty.call(envObject, key)) {
        delete process.env[key];
      }
    }
  }
}
```

**默认行为**: 不覆盖已存在的 `process.env` 变量。

### 6.4 Supervisor 预置变量与 populateEnv 的冲突处理（关键细节）

**核心结论**: 平台返回的 envVars 优先级高于 Supervisor 预置变量，**会覆盖同名键**。

**代码证据**:

1. **开发环境调用** (`packages/cli-v3/src/entryPoints/managed-run-worker.ts:374-381`):
   ```typescript
   EXECUTE_TASK_RUN: async (
     { execution, traceContext, metadata, metrics, env, isWarmStart },
     sender
   ) => {
     if (env) {
       populateEnv(env, {
         override: true,  // 显式传入 true
         previousEnv: _lastEnv,
       });
       _lastEnv = env;
     }
     // ...
   }
   ```

2. **生产环境调用** (`packages/cli-v3/src/executions/taskRunProcess.ts:305-312`):
   ```typescript
   await this._ipc?.send("EXECUTE_TASK_RUN", {
     execution,
     traceContext,
     metadata: this.options.serverWorker,
     metrics,
     env: params.env,  // 从平台获取的 envVars
     isWarmStart: isWarmStart ?? this.options.isWarmStart,
   });
   ```

3. **envVars 来源** (`apps/webapp/app/v3/services/worker/workerGroupTokenService.server.ts:545-580`):
   ```typescript
   private async getEnvVars(
     environment: RuntimeEnvironment,
     runId: string,
     machinePreset: MachinePreset,
     parentEnvironment?: RuntimeEnvironment,
     taskEventStore?: string
   ): Promise<Record<string, string>> {
     const variables = await resolveVariablesForEnvironment(environment, parentEnvironment);
     
     // 添加运行时变量
     variables.push(
       ...[
         { key: "TRIGGER_JWT", value: jwt },
         { key: "TRIGGER_RUN_ID", value: runId },
         { key: "TRIGGER_MACHINE_PRESET", value: machinePreset.name },
       ]
     );
     // ...
     return variables.reduce((acc, v) => ({ ...acc, [v.key]: v.value }), {});
   }
   ```

**生效顺序总结**（从低到高）:

```
1. 容器基础环境变量 (OS / Docker 基础镜像)
   ↓
2. Supervisor 预置变量 (创建容器时注入)
   ├─ OTEL_EXPORTER_OTLP_ENDPOINT
   ├─ TRIGGER_DEQUEUED_AT_MS
   ├─ TRIGGER_POD_SCHEDULED_AT_MS
   ├─ TRIGGER_ENV_ID
   ├─ TRIGGER_DEPLOYMENT_ID
   ├─ TRIGGER_MACHINE_CPU
   ├─ TRIGGER_MACHINE_MEMORY
   └─ ... 其他系统变量
   ↓
3. 平台返回的 envVars (通过 populateEnv({ override: true }) 注入)
   ├─ overridableTriggerVariables (最低)
   ├─ overridableOtelVariables (dev only)
   ├─ projectSecrets (用户变量，含 isSecret 明文)
   ├─ builtInVariables (TRIGGER_SECRET_KEY, TRIGGER_API_URL 等)
   └─ 运行时追加 (TRIGGER_JWT, TRIGGER_RUN_ID, TRIGGER_MACHINE_PRESET)
```

**注意**:
- 虽然 `populateEnv` 的默认值是 `override: false`，但实际调用时**总是**传入 `override: true`
- 这意味着平台返回的 envVars 会覆盖 Supervisor 预置的同名变量
- 只有当平台返回的 envVars 中不包含某个键时，才会保留 Supervisor 预置的值

## 七、完整路径总结

```
┌─────────────────────────────────────────────────────────┐
│  面板编辑 (前端)                                         │
│  apps/webapp/app/routes/...environment-variables/route.tsx │
└─────────────────────────────┬───────────────────────────┘
                              │ POST / action
                              ▼
┌─────────────────────────────────────────────────────────┐
│  服务端处理                                              │
│  EnvironmentVariablesRepository                          │
│  - removeBlacklistedVariables()                         │
│  - 加密存储 (AES-256-GCM)                               │
│  apps/webapp/app/v3/environmentVariables/...            │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│  运行时解析                                              │
│  resolveVariablesForEnvironment()                       │
│  合并顺序 (优先级低→高):                                 │
│  1. overridableTriggerVariables                         │
│  2. overridableOtelVariables (dev only)                 │
│  3. projectSecrets (父→子环境合并)                      │
│  4. builtInVariables                                    │
│  - deduplicateVariableArray() 去重                     │
└─────────────────────────────┬───────────────────────────┘
                              │
           ┌──────────────────┼──────────────────┐
           │                  │                  │
           ▼                  ▼                  ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│  API 暴露         │ │  构建时注入       │ │  任务执行注入     │
│  GET /envvars     │ │  Dockerfile      │ │  Supervisor       │
│  返回 isSecret    │ │  镜像构建阶段     │ │  创建容器时注入   │
│  明文值           │ │  indexWorker     │ │  预置系统变量     │
│                  │ │  populateEnv()   │ │    ↓              │
│                  │ │                  │ │  平台返回 envVars  │
│                  │ │                  │ │  populateEnv({     │
│                  │ │                  │ │    override: true  │
│                  │ │                  │ │  }) 覆盖预置变量  │
└──────────────────┘ └──────────────────┘ └──────────────────┘
```

## 八、关键注意事项

1. **黑名单保护**: `TRIGGER_SECRET_KEY` 和 `TRIGGER_API_URL` 无法被用户设置，由系统自动注入

2. **环境继承**: 子环境变量会覆盖父环境的同名变量

3. **优先级**: 内置变量 > 用户变量 > 可覆盖系统变量

4. **加密**: 所有环境变量值都使用 AES-256-GCM 加密存储在数据库中

5. **版本追踪**: 每个环境变量值都有 `version` 字段，每次更新递增

6. **密钥标记**: `isSecret` 标记的变量在 UI 中显示为 `<redacted>`，但通过 API `/api/v1/projects/:projectRef/envvars` 返回明文值

7. **运行时覆盖**: `populateEnv()` 默认不覆盖 `process.env` 中已存在的变量，但在任务执行时**总是**传入 `override: true`，因此平台返回的 envVars 会覆盖 Supervisor 预置的同名变量

8. **API 安全**: `/api/v1/projects/:projectRef/envvars` 和 worker-actions API 都受认证保护（API Key / Worker Token）
