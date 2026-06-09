# CLI 部署流程深度解析：项目发现、凭据解析与增量发布

本文档面向需要搭建 CI 自动化部署的团队，系统梳理 trigger.dev CLI 从「扫描项目元信息」到「触发版本上线」的完整代码路径，重点覆盖三个易混淆领域：多项目仓库下项目根的判定、环境与凭据的解析顺序、增量与全量发布的差异。所有源码引用格式为 `仓库相对路径:行号`，方便在 IDE 中直接跳转复核。

---

## 一、总体部署流水线概览

CLI `deploy` 命令主入口 `packages/cli-v3/src/commands/deploy.ts:256` 的 `_deployCommand()`。整个流水线在认证后于 `packages/cli-v3/src/commands/deploy.ts:364` 分叉为两条路径：

```
                     ┌─────────────┐   ┌──────────────┐
                     │ 1. 项目发现  │──▶│ 2. 用户认证   │
                     │ (loadConfig) │   │ (login)      │
                     └──────┬──────┘   └──────┬───────┘
                            │                 │
                            ▼                 ▼
              ┌─────────────────────────────────────────────┐
              │        getProjectClient() (packages/cli-v3/src/commands/deploy.ts:351-358)        │
              └─────────────────┬───────────────────────────┘
                                │
                    nativeBuildServer?
                     ╱              ╲
                   Yes               No
                   ╱                  ╲
    ┌──────────────────────┐   ┌──────────────────────┐
    │ Native Build Server  │   │ 3. 本地打包           │
    │ 归档+上传 workspace  │   │ (buildWorker)         │
    │ contentHash: "-"     │   │ contentHash: 真实 MD5 │
    └──────────┬───────────┘   └──────────┬───────────┘
               │                          │
               ▼                          ▼
    ┌──────────────────────┐   ┌──────────────────────┐
    │ initializeDeployment │   │ 4. 镜像构建           │
    │ (服务端构建+推送)     │   │ (buildImage)          │
    │ progress 更新 Hash   │   │ Docker / Depot        │
    └──────────┬───────────┘   └──────────┬───────────┘
               │                          │
               ▼                          ▼
    ┌──────────────────────┐   ┌──────────────────────┐
    │ 5. finalize + 晋升   │◀──│ 5. 环境变量同步       │
    │ (promotion)          │   │ 6. finalize + 晋升    │
    └──────────────────────┘   └──────────────────────┘
```

详细分叉点分析见第四章，contentHash 三层语义见第五章。

---

## 二、多项目仓库下项目根的判定

### 2.1 入口：路径参数 → `projectPath`

`packages/cli-v3/src/commands/deploy.ts:265-266`：

```ts
const cwd = process.cwd();
const projectPath = resolve(cwd, dir);  // dir 是命令行 [path] 参数，默认 "."
```

CLI 接受一个可选的位置参数 `[path]`（默认 `.`），将其与 `process.cwd()` 拼接得到绝对路径 `projectPath`。

### 2.2 配置文件发现：`loadConfig()` 的三层搜索

核心函数 `packages/cli-v3/src/config.ts:34-47` 使用 [c12](https://github.com/unjs/c12) 库加载名为 `trigger` 的配置：

```ts
const result = await c12.loadConfig<TriggerConfig>({
  name: "trigger",    // 搜索 trigger.config.ts / .js / .mjs
  cwd,                // 即上面的 projectPath
  configFile,         // 用户通过 --config 显式指定
});
```

c12 的搜索顺序（由近及远）：
1. `--config` 参数指定的文件（绝对路径或相对于 `cwd`）
2. `cwd` 下的 `trigger.config.ts` / `trigger.config.js` / `trigger.config.mjs`
3. 向上递归父目录查找同名配置文件（c12 内置行为）

**关键判断**：如果找不到配置文件，CLI 直接抛出 `OutroCommandError`（`packages/cli-v3/src/config.ts:169-178`），部署终止。**配置文件是项目根的锚定物。**

### 2.3 `workingDir` 的推导

`packages/cli-v3/src/config.ts:160-164`，`workingDir` 按以下优先级确定：

```ts
const workingDir = result.configFile
  ? dirname(result.configFile)     // ① 配置文件所在目录
  : packageJsonPath
    ? dirname(packageJsonPath)     // ② package.json 所在目录
    : cwd;                         // ③ 退回 cwd
```

**workingDir 就是「项目根」，是后续一切相对路径计算的基准。**

### 2.4 `workspaceDir`：monorepo 的「仓库根」

`packages/cli-v3/src/config.ts:158`：

```ts
const workspaceDir = await findWorkspaceDir(cwd);
```

使用 `pkg-types` 的 `findWorkspaceDir()` 向上查找包含 `pnpm-workspace.yaml`、`lerna.json`、`nx.json` 或根 `package.json`（含 `workspaces` 字段）的目录。`workspaceDir` 在以下场景中被使用：
- Git 元信息采集（`.git/config` 在仓库根下，见 `packages/cli-v3/src/utilities/gitMeta.ts:18`）
- Native Build Server 的归档范围（整个 workspace 打成 tar.gz，见 `packages/cli-v3/src/commands/deploy.ts:1009`）
- Preview 分支的 context 归档

### 2.5 Monorepo 中多 Trigger 项目的场景

在 monorepo 中，每个 Trigger 项目在自己的子目录下拥有独立的 `trigger.config.ts`。部署时：

- **CLI 必须在对应子目录执行**，或通过 `[path]` 参数指向该子目录
- `workingDir` = 配置文件所在子目录 → 任务文件搜索范围限定在该子项目内
- `workspaceDir` = monorepo 根 → Git 信息和归档覆盖整个仓库
- 项目引用（`projectRef`）来自配置文件中的 `project` 字段，或 `--project-ref` 参数，或 `TRIGGER_PROJECT_REF` 环境变量

### 2.6 任务目录（`dirs`）的自动发现

如果配置文件未显式声明 `dirs`，CLI 会调用 `packages/cli-v3/src/config.ts:257-281` 的 `autoDetectDirs()`：

```ts
async function autoDetectDirs(workingDir: string): Promise<string[]> {
  // 递归扫描 workingDir 下所有子目录
  // 跳过: node_modules, .git, dist, out, build, 隐藏目录 (packages/cli-v3/src/config.ts:263)
  // 跳过: 以 app/api/trigger 结尾的路径（Next.js API route）(packages/cli-v3/src/config.ts:269)
  // 匹配: 名为 "trigger" 的目录 (packages/cli-v3/src/config.ts:273)
  // 递归进入所有其他目录继续查找 (packages/cli-v3/src/config.ts:277)
}
```

即使没有配置 `dirs`，只要在 `workingDir` 下存在 `trigger/` 子目录，CLI 就能自动发现任务定义文件。

---

## 三、环境与凭据的解析顺序

### 3.1 认证凭据的三层来源

`packages/cli-v3/src/commands/login.ts:101-398` 的 `login()` 函数按以下优先级解析凭据：

```
优先级    来源                              适用场景
────────────────────────────────────────────────────────
 1       TRIGGER_ACCESS_TOKEN 环境变量       CI/CD（必需）
 2       本地配置文件 (config.json)          本地开发
 3       交互式浏览器登录 (OAuth flow)       首次使用
```

**第一优先级：环境变量 `TRIGGER_ACCESS_TOKEN`**

`packages/cli-v3/src/commands/login.ts:120-155`：

```ts
const accessTokenFromEnv = env.TRIGGER_ACCESS_TOKEN;
if (accessTokenFromEnv) {
  const validationResult = validateAccessToken(accessTokenFromEnv);
  // 必须以 tr_pat_（个人令牌）或 tr_oat_（组织令牌）开头
  // 令牌前缀定义见 packages/cli-v3/src/utilities/accessTokens.ts:1-2
  // tr_oat_ 在当前版本仅内部使用，对用户不可见
}
```

API URL 的解析（`packages/cli-v3/src/commands/login.ts:135`）：
```ts
const apiUrl = env.TRIGGER_API_URL ?? opts.defaultApiUrl ?? CLOUD_API_URL;
// TRIGGER_API_URL → 命令行 --api-url → 默认 https://api.trigger.dev
// CLOUD_API_URL 定义见 packages/cli-v3/src/consts.ts:3
```

**第二优先级：本地配置文件**

文件位置由 XDG 规范确定（`packages/cli-v3/src/utilities/configFiles.ts:8-11`）：
```
~/.config/trigger/config.json     (Linux/macOS)
%APPDATA%\trigger\config.json     (Windows)
```

配置文件存储多 profile，每个 profile 含 `accessToken` 和 `apiUrl`（`packages/cli-v3/src/utilities/configFiles.ts:19-22`）。`--profile` 参数切换 profile，默认 `default`。

**第三优先级：交互式登录**

`packages/cli-v3/src/commands/login.ts:273-313`：
1. 调用 API 生成授权码（`packages/cli-v3/src/commands/login.ts:276`）
2. 打开浏览器让用户授权（`packages/cli-v3/src/commands/login.ts:288`）
3. 轮询 API 等待令牌（最多 60 秒，1 秒间隔，`packages/cli-v3/src/commands/login.ts:295-302`）
4. 将令牌写入本地配置文件（`packages/cli-v3/src/commands/login.ts:307-313`）

### 3.2 CI 环境的特殊行为

`packages/cli-v3/src/commands/login.ts:251-267`：

```ts
if (isCI) {
  // 必须设置 TRIGGER_ACCESS_TOKEN，否则直接报错
  throw new Error("Authentication required in CI environment. ...");
}
```

**CI 中没有环境变量 → 直接失败，不会尝试交互式登录。**

### 3.3 项目引用（`projectRef`）的解析链

`packages/cli-v3/src/commands/deploy.ts:294-304`：

```ts
const envVars = resolveLocalEnvVars(options.envFile);
const resolvedConfig = await loadConfig({
  cwd: projectPath,
  overrides: { project: options.projectRef ?? envVars.TRIGGER_PROJECT_REF },
  configFile: options.config,
});
```

`projectRef` 最终值由 `packages/cli-v3/src/config.ts:196-224` 中 `defu()` 合并决定：

```
defu() 合并优先级（后者覆盖前者）：
  默认值 → config 文件中的 project → overrides（--project-ref 或 TRIGGER_PROJECT_REF）
```

### 3.4 环境变量加载的完整链路

部署命令中涉及两层环境变量加载：

**第一层：CLI 进程自身的环境变量**（用于凭据和项目发现）

`packages/cli-v3/src/utilities/localEnvVars.ts:4-15` 合并顺序：
```ts
{
  ...sanitizeEnvVars(processEnv),          // ① 系统环境变量 (packages/cli-v3/src/utilities/localEnvVars.ts:8)
  ...sanitizeEnvVars(additionalVariables), // ② 额外变量（如有）(packages/cli-v3/src/utilities/localEnvVars.ts:13)
  ...sanitizeEnvVars(dotEnvVars),          // ③ .env 文件变量（后加载覆盖前者）(packages/cli-v3/src/utilities/localEnvVars.ts:14)
}
```

`packages/cli-v3/src/utilities/dotEnv.ts:13-37` 的 `resolveDotEnvVars()` 加载：
- `.env`、`.env.development`、`.env.local`、`.env.development.local`、`dev.vars`（`packages/cli-v3/src/utilities/dotEnv.ts:5-11`）
- 或 `--env-file` 指定的路径
- **显式删除** `TRIGGER_API_URL`、`TRIGGER_SECRET_KEY`、`OTEL_EXPORTER_OTLP_ENDPOINT`（`packages/cli-v3/src/utilities/dotEnv.ts:28-30`）（这些应该来自 worker，不应从本地 .env 泄露）

**第二层：构建时的服务端环境变量**（用于代码打包时的 inline 替换）

`packages/cli-v3/src/commands/deploy.ts:376-395`：

```ts
const serverEnvVars = await projectClient.client.getEnvironmentVariables(resolvedConfig.project);
// serverEnvVars 传入 buildWorker()，在 esbuild 打包时注入
```

### 3.5 环境类型的解析

`packages/cli-v3/src/commands/deploy.ts:66`：

```ts
env: z.enum(["prod", "staging", "preview", "production"]),
// production 会被强制转换为 prod (packages/cli-v3/src/commands/deploy.ts:290-291)
```

- `prod` / `production` → 生产环境
- `staging` → 预发布环境
- `preview` → 预览分支（需要 `--branch` 参数或自动检测 Git 分支）

### 3.6 项目客户端的获取

`packages/cli-v3/src/commands/deploy.ts:351-358` 调用 `getProjectClient()`，该函数在 `packages/cli-v3/src/utilities/session.ts:84-118` 中：
- 用个人令牌创建 API 客户端，调用 `getProjectEnv()` 获取项目环境的 API Key
- 用该 API Key 创建新的客户端实例（`packages/cli-v3/src/utilities/session.ts:111`），后续操作使用这个环境级客户端

---

## 四、Native Build Server 与普通部署的分叉点

### 4.1 分叉位置：`options.nativeBuildServer` 判断

两条路径在 `packages/cli-v3/src/commands/deploy.ts:364-374` 处分叉：

```ts
if (options.nativeBuildServer) {
  await handleNativeBuildServerDeploy({ ... });
  return;  // 提前退出，不再执行后续本地构建流程
}
// ↓ 以下代码仅在普通路径执行
const serverEnvVars = await projectClient.client.getEnvironmentVariables(resolvedConfig.project);
// ...
const [error, buildManifest] = await tryCatch(buildWorker({ ... }));
// ...
const deployment = await initializeOrAttachDeployment(projectClient.client, {
  contentHash: buildManifest.contentHash,
  // ...
  isNativeBuild: false,
}, envVars.TRIGGER_EXISTING_DEPLOYMENT_ID);
```

**关键区别**：
- 普通路径：先 `buildWorker()` 在本地打包，得到真实 `contentHash`，再 `initializeDeployment()` 创建部署记录
- Native 路径：先 `initializeDeployment(contentHash: "-")` 创建部署记录（占位值），归档上传整个 workspace，由服务端完成打包和构建

两条路径的公共前置步骤是项目发现、认证、`getProjectClient()`（`packages/cli-v3/src/commands/deploy.ts:351-358`），之后立即分叉。

### 4.2 两条路径的执行序列对比

```
普通路径                                           Native Build Server 路径
──────────                                         ──────────────────────
getProjectClient()                                 getProjectClient()
  ↓                                                  ↓
serverEnvVars = getEnvironmentVariables()           (跳过——服务端构建时获取)
  ↓                                                  ↓
buildWorker()  ← 本地 esbuild 打包                   createContextArchive()  ← 归档整个 workspace
  ↓ contentHash = 真实 MD5                            ↓
  ↓                                                  createArtifact() + 上传 tar.gz
  ↓                                                  ↓
initializeDeployment(contentHash: 真实值)            initializeDeployment(contentHash: "-")
  isNativeBuild: false                               isNativeBuild: true
  ↓                                                  artifactKey: S3 key
buildImage()  ← 本地 Docker / Depot 构建              configFilePath: 相对路径
  ↓                                                  ↓
syncEnvVarsWithServer()  (可选)                      (服务端构建时处理)
  ↓                                                  ↓
finalizeDeployment()                                 finalizeDeployment()
  ↓                                                  ↓
完成                                                完成（或 --detach 提前返回）
```

---

## 五、contentHash 的语义：占位值、真实值与元数据

### 5.1 contentHash 的计算：仅存在于本地打包路径

`packages/cli-v3/src/build/bundle.ts:244-323` 的哈希计算逻辑：

```ts
const hasher = createHash("md5");
for (const outputFile of result.outputFiles) {
  hasher.update(outputFile.hash);   // 累加每个输出文件的 esbuild 内部哈希 (packages/cli-v3/src/build/bundle.ts:248)
  outputHashes[outputFile.path] = outputFile.hash;  // packages/cli-v3/src/build/bundle.ts:250
}
// ...
contentHash: hasher.digest("hex"),  // packages/cli-v3/src/build/bundle.ts:323 最终 MD5 摘要
```

**contentHash 是所有 esbuild 输出文件哈希的 MD5 聚合**。任何一个源文件变动都会改变 contentHash。这个值仅在本地打包路径中计算——Native Build Server 路径中 CLI 不执行 esbuild，因此无法计算。

### 5.2 contentHash 在三个层面的不同角色

contentHash 在系统中以三种身份出现，它们的语义和生命周期各不相同：

**① 部署记录占位值（WorkerDeployment.contentHash）**

- 普通路径：`initializeDeployment()` 时传入真实 MD5 值（`packages/cli-v3/src/commands/deploy.ts:425`），服务端写入 `WorkerDeployment` 记录（`apps/webapp/app/v3/services/initializeDeployment.server.ts:244`）
- Native 路径：`initializeDeployment()` 时传入 `"-"` 占位（`packages/cli-v3/src/commands/deploy.ts:1082`），服务端同样写入 `WorkerDeployment` 记录，但此时 `contentHash` 字段为无意义值

Native 路径的占位值通过 **progressDeployment** API 更新为真实值。构建服务器在完成 esbuild 打包后，调用 `POST /api/v1/deployments/:id/progress`（`apps/webapp/app/routes/api.v1.deployments.$deploymentId.progress.ts:47-51`），传入 `contentHash` 和 `runtime`：

```ts
// apps/webapp/app/routes/api.v1.deployments.$deploymentId.progress.ts:47-51
deploymentService.progressDeployment(authenticatedEnv, deploymentId, {
  contentHash: body.data.contentHash,
  git: body.data.gitMeta,
  runtime: body.data.runtime,
  buildServerMetadata: body.data.buildServerMetadata,
});
```

`progressDeployment()` 仅在 `PENDING` 或 `INSTALLING` 状态时可调用（`apps/webapp/app/v3/services/deployment.server.ts:60`），将占位值 `"-"` 更新为真实哈希。

**② Worker 元数据真实值（BackgroundWorker.contentHash）**

部署终结时，`finalizeDeployment()` 要求 `WorkerDeployment` 上已关联一个 `BackgroundWorker`（`apps/webapp/app/v3/services/finalizeDeployment.server.ts:41-52`）。BackgroundWorker 的 `contentHash` 来自构建产物元数据，而非部署记录：

- V4 路径：`apps/webapp/app/v3/services/createDeploymentBackgroundWorkerV4.server.ts:83`
  ```ts
  contentHash: body.metadata.contentHash,  // 来自 CLI 或构建服务器传入的 metadata
  ```
- V3 路径（已废弃）：`apps/webapp/app/v3/services/createDeploymentBackgroundWorkerV3.server.ts:70`

**BackgroundWorker.contentHash 是实际运行时的指纹**。它用于判断两个 worker 是否代码相同——在 `apps/webapp/app/v3/services/createBackgroundWorker.server.ts:107` 中，如果最新 worker 的 contentHash 与新部署相同，则直接复用现有 worker 而不创建新的：

```ts
// apps/webapp/app/v3/services/createBackgroundWorker.server.ts:107-109
if (latestBackgroundWorker?.contentHash === body.metadata.contentHash) {
  return latestBackgroundWorker;  // 代码未变，复用已有 worker
}
```

**③ Docker 构建参数（TRIGGER_CONTENT_HASH）**

contentHash 还作为 Docker build-arg 传入镜像构建，最终成为容器内的环境变量：

- 本地构建：`packages/cli-v3/src/deploy/buildImage.ts:229`
  ```ts
  "--build-arg", `TRIGGER_CONTENT_HASH=${options.contentHash}`,
  ```
- 远程 Depot 构建：`packages/cli-v3/src/deploy/buildImage.ts:580`
  ```ts
  "--build-arg", `TRIGGER_CONTENT_HASH=${options.contentHash}`,
  ```

**这意味着 contentHash 被烧录进了 Docker 镜像**，worker 运行时可通过 `process.env.TRIGGER_CONTENT_HASH` 读取自己的内容指纹。

### 5.3 contentHash 三层值的关系总结

```
┌─────────────────────────────────────────────────────────────────┐
│                     WorkerDeployment 记录                        │
│  contentHash: 普通路径=真实MD5 / Native路径=初始"-",后由progress更新 │
│  用途: Dashboard 展示、版本记录                                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │ finalize 时要求已关联
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     BackgroundWorker 记录                        │
│  contentHash: 始终为真实值 (来自 metadata)                        │
│  用途: worker 去重判断 (同 contentHash 复用已有 worker)             │
└──────────────────────────┬──────────────────────────────────────┘
                           │ 镜像内可读取
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Docker 镜像 (运行时)                         │
│  TRIGGER_CONTENT_HASH: 作为 build-arg 烧录                       │
│  用途: worker 进程可读自己的指纹                                    │
└─────────────────────────────────────────────────────────────────┘
```

**注意**：WorkerDeployment.contentHash 和 BackgroundWorker.contentHash 可能不同步——Native 路径中，如果 `progressDeployment` 因故障未执行，部署记录的 contentHash 可能停留在 `"-"`，但 BackgroundWorker 的 contentHash 始终是真实值。

---

## 六、Docker 缓存与增量构建

### 6.1 Docker 层缓存

虽然每次部署都创建新记录，Docker 镜像构建可以利用缓存：

```ts
// packages/cli-v3/src/commands/deploy.ts:74
cache: z.boolean().default(true),    // --no-cache 可禁用
// packages/cli-v3/src/commands/deploy.ts:79
useRegistryCache: z.boolean().default(false),  // --use-registry-cache 启用远端缓存
```

- `cache: true`（默认）：Docker BuildKit 使用本地构建缓存，**未改变的层会被复用**
- `--no-cache`：强制所有层重新构建（全量构建）
- `--use-registry-cache`：将 Docker 缓存推送到远端 registry，跨 CI 运行复用

### 6.2 contentHash 与 Docker 缓存的关系

contentHash 和 Docker 缓存是**两个独立的优化维度**：

- contentHash 是**源码级指纹**，反映 esbuild 产物的内容是否变化
- Docker 缓存是**镜像层级复用**，反映 Dockerfile 指令对应层是否可复用

它们的交互方式：

1. contentHash 变化 → 新部署记录 → 新镜像构建 → Docker 可能复用未改变的层
2. contentHash 不变 → dev 模式下跳过重建（`packages/cli-v3/src/dev/devSupervisor.ts:310-311`），生产部署仍会创建新记录
3. `--no-cache` 跳过 Docker 层缓存 → 即使 contentHash 未变也全量构建
4. contentHash 作为 `TRIGGER_CONTENT_HASH` build-arg → **如果 contentHash 变化，对应 Docker 层必然失效**（因为 ARG 值变了）

这意味着 contentHash 实际上充当了 Docker 缓存的一个「失效信号」：代码改动 → contentHash 改变 → `TRIGGER_CONTENT_HASH` build-arg 改变 → 至少一层缓存失效。

### 6.3 版本号分配：乐观并发 + 重试

`apps/webapp/app/v3/services/initializeDeployment/createDeploymentWithNextVersion.server.ts:46-99` 处理并发部署的版本冲突：

```
1. 查询当前环境最新部署的 version (apps/webapp/app/v3/services/initializeDeployment/createDeploymentWithNextVersion.server.ts:59-63)
2. 调用 calculateNextBuildVersion(latest?.version) 计算下一个版本号 (apps/webapp/app/v3/services/initializeDeployment/createDeploymentWithNextVersion.server.ts:65)
3. 尝试写入数据库（version + environmentId 联合唯一约束）(apps/webapp/app/v3/services/initializeDeployment/createDeploymentWithNextVersion.server.ts:70-71)
4. 如果唯一约束冲突（并发竞争），加随机 jitter 后重试，最多 5 次 (apps/webapp/app/v3/services/initializeDeployment/createDeploymentWithNextVersion.server.ts:73-91)
```

版本号格式类似 `20250208.1`（日期.序号）。

### 6.4 两种构建路径的对比

| 维度 | 本地构建（默认路径） | Native Build Server（`--native-build-server`） |
|------|---------------------|-----------------------------------------------|
| 代码打包 | CLI 本地 esbuild | 服务端 esbuild |
| 镜像构建 | CLI 调用 Docker / Depot | 服务端 Build Server |
| contentHash | 由本地 esbuild 输出计算 | 传 `"-"` 占位（`packages/cli-v3/src/commands/deploy.ts:1082`） |
| 上下文传输 | 仅传输构建产物到远端 | 整个 workspace 打成 tar.gz 上传（`packages/cli-v3/src/commands/deploy.ts:1009`） |
| 构建日志 | Docker CLI 输出 | S2 流式日志 |
| 环境变量同步 | buildManifest.deploy.sync | 服务端处理 |
| 交互模式 | 等待构建+部署完成 | 可 `--detach` 立即返回 |

### 6.5 本地构建 vs 自托管构建

`packages/cli-v3/src/commands/deploy.ts:440-442`：

```ts
const isLocalBuild = options.localBuild || !deployment.externalBuildData;
const authenticateToTriggerRegistry = options.localBuild;
const skipServerSideRegistryPush = options.localBuild;
```

- `externalBuildData` 为空（自托管场景）→ 隐式进入本地构建路径
- 本地构建需要本地安装 Docker BuildKit（验证在 `packages/cli-v3/src/commands/deploy.ts:446-453`）
- 自托管场景：镜像构建后不推送到 Trigger 云端 registry

---

## 七、完整部署流程追踪

### 阶段 1：项目发现

```
deploy [path]                                                         # packages/cli-v3/src/commands/deploy.ts:102
  → projectPath = resolve(cwd, dir)                                   # packages/cli-v3/src/commands/deploy.ts:266
  → loadConfig({ cwd: projectPath, ... })                             # packages/cli-v3/src/commands/deploy.ts:300-304
     → c12.loadConfig({ name: "trigger", cwd })                       # packages/cli-v3/src/config.ts:40-45
        → 搜索 trigger.config.ts 等
     → resolveConfig()                                                # packages/cli-v3/src/config.ts:149-232
        → workingDir = dirname(configFile) / dirname(packageJson) / cwd  # packages/cli-v3/src/config.ts:160-164
        → workspaceDir = findWorkspaceDir(cwd)                        # packages/cli-v3/src/config.ts:158
        → dirs = config.dirs ?? autoDetectDirs(workingDir)            # packages/cli-v3/src/config.ts:186
        → project = defu(overrides, config, defaults).project         # packages/cli-v3/src/config.ts:196-224
```

### 阶段 2：用户认证

```
login({ embedded: true, ... })                                        # packages/cli-v3/src/commands/deploy.ts:270-275
  → TRIGGER_ACCESS_TOKEN 存在？                                        # packages/cli-v3/src/commands/login.ts:120
    → validateAccessToken() 验证前缀                                    # packages/cli-v3/src/utilities/accessTokens.ts:12-23
    → whoAmI() 验证令牌有效性                                           # packages/cli-v3/src/commands/login.ts:138
    → apiUrl = TRIGGER_API_URL ?? defaultApiUrl ?? CLOUD_API_URL       # packages/cli-v3/src/commands/login.ts:135
  → 本地 profile 有 token？                                            # packages/cli-v3/src/commands/login.ts:158-160
    → whoAmI() 验证存储令牌                                            # packages/cli-v3/src/commands/login.ts:161
  → CI 环境？                                                          # packages/cli-v3/src/commands/login.ts:251
    → 报错，要求设置 TRIGGER_ACCESS_TOKEN                               # packages/cli-v3/src/commands/login.ts:260-266
  → 交互式环境？                                                       # packages/cli-v3/src/commands/login.ts:273
    → 浏览器 OAuth 登录 + 轮询                                         # packages/cli-v3/src/commands/login.ts:276-302
```

### 阶段 3：任务收集 + 代码打包

```
buildWorker({ target: "deploy", ... })                                # packages/cli-v3/src/build/buildWorker.ts:44
  → createEntryPointManager(dirs, config, "deploy", false)            # packages/cli-v3/src/build/buildWorker.ts:74 (间接)
     → glob(dirs 下的 *.{ts,tsx,mts,cts,js,jsx,mjs,cjs})             # packages/cli-v3/src/build/entryPoints.ts:63-67
     → 添加 managedEntryPoints（runController, indexController 等）    # packages/cli-v3/src/build/entryPoints.ts:84
     → 添加 config.configFile                                         # packages/cli-v3/src/build/entryPoints.ts:75
  → bundleWorker({ entryPoints, ... })                                # packages/cli-v3/src/build/buildWorker.ts:79-89
     → esbuild.build() → 所有输出文件的 MD5 聚合 → contentHash         # packages/cli-v3/src/build/bundle.ts:244-323
  → createBuildManifestFromBundle()                                   # packages/cli-v3/src/build/buildWorker.ts:93-101
     → 提取任务文件列表、入口点、外部依赖、sync 配置
  → bundleSkills() (如果存在 AI skill 定义)                            # packages/cli-v3/src/build/buildWorker.ts:111-122
  → notifyExtensionOnBuildComplete() (build extensions 钩子)           # packages/cli-v3/src/build/buildWorker.ts:128
  → writeDeployFiles()                                                # packages/cli-v3/src/build/buildWorker.ts:135-141
     → package.json（仅含外部依赖）                                     # packages/cli-v3/src/build/buildWorker.ts:222-237
     → build.json（BuildManifest）                                     # packages/cli-v3/src/build/buildWorker.ts:239
     → Containerfile（Dockerfile）                                     # packages/cli-v3/src/build/buildWorker.ts:240
```

### 阶段 4：部署初始化 + 镜像构建

**标准路径（非 Native Build）：**

```
initializeOrAttachDeployment()                                        # packages/cli-v3/src/commands/deploy.ts:422-435
  → TRIGGER_EXISTING_DEPLOYMENT_ID 存在？                              # packages/cli-v3/src/commands/deploy.ts:903
    → attach 到已有部署                                                # packages/cli-v3/src/commands/deploy.ts:911
  → 否则 → apiClient.initializeDeployment({ contentHash, ... })       # packages/cli-v3/src/commands/deploy.ts:941-943
           → POST /api/v1/deployments                                  # packages/cli-v3/src/apiClient.ts:414
           → 服务端创建新 WorkerDeployment 记录                         # apps/webapp/app/v3/services/initializeDeployment.server.ts:196-261
           → 返回 deployment.id, version, imageTag, externalBuildData

buildImage()                                                          # packages/cli-v3/src/commands/deploy.ts:530
  → isLocalBuild ? localBuildImage() : remoteBuildImage()
  → localBuildImage(): 调用 docker buildx build
  → remoteBuildImage(): 调用 Depot 远程构建
```

**Native Build Server 路径：**

```
handleNativeBuildServerDeploy()                                       # packages/cli-v3/src/commands/deploy.ts:986
  → createContextArchive(workspaceDir, archivePath)                   # packages/cli-v3/src/commands/deploy.ts:1009
     → 过滤: .git, node_modules, dist, .env, .trigger, ...           # packages/cli-v3/src/deploy/archiveContext.ts:5-49
     → 合并 .gitignore 规则                                           # packages/cli-v3/src/deploy/archiveContext.ts:96-101
  → apiClient.createArtifact()                                        # packages/cli-v3/src/commands/deploy.ts:1015
     → 获取 S3 预签名上传 URL
  → POST 上传 tar.gz                                                  # packages/cli-v3/src/commands/deploy.ts:1050-1054
  → apiClient.initializeDeployment({                                  # packages/cli-v3/src/commands/deploy.ts:1081-1092
      contentHash: "-",                               // 占位，服务端计算
      isNativeBuild: true,
      artifactKey,
      configFilePath,                                 // 配置文件在 workspace 中的相对路径 # packages/cli-v3/src/commands/deploy.ts:1076-1079
      skipPromotion,
    })
  → 服务端入队构建任务                                                 # apps/webapp/app/v3/services/initializeDeployment.server.ts:274-306
  → 返回 S2 event stream 用于实时日志                                 # apps/webapp/app/v3/services/initializeDeployment.server.ts:144-152
```

### 阶段 5：环境变量同步

`packages/cli-v3/src/commands/deploy.ts:456-499`：

```ts
if (hasVarsToSync) {
  syncEnvVarsWithServer(client, projectRef, env, childVars, parentVars);
  // 仅当 BuildManifest.deploy.sync 存在时触发 (packages/cli-v3/src/commands/deploy.ts:456-459)
  // preview 分支额外同步 parentEnv（从父环境继承变量）(packages/cli-v3/src/commands/deploy.ts:459)
}
```

### 阶段 6：终结部署 + 晋升

`packages/cli-v3/src/commands/deploy.ts:667-688`：

```
apiClient.finalizeDeployment(deploymentId, {
  imageDigest,                          // 镜像摘要 (packages/cli-v3/src/commands/deploy.ts:670)
  skipPromotion,                        // --skip-promotion 跳过晋升 (packages/cli-v3/src/commands/deploy.ts:671)
  skipPushToRegistry,                   // 本地构建跳过推送 (packages/cli-v3/src/commands/deploy.ts:672)
})
  → 服务端验证镜像、创建 Worker 记录
  → 若非 skipPromotion → 自动晋升为当前部署
  → SSE 返回进度日志
```

晋升（promotion）是将某次部署设为环境「当前活跃版本」的操作。可通过 `--skip-promotion` 延迟晋升，之后用 `trigger.dev promote <version>` 手动执行。

---

## 八、CI 自动化部署的实践要点

### 8.1 必要的环境变量

```yaml
env:
  TRIGGER_ACCESS_TOKEN: tr_pat_xxxx    # 个人访问令牌（必需）
  # TRIGGER_API_URL: https://xxx       # 自托管时设置（可选）
  # TRIGGER_PROJECT_REF: proj_xxxx     # 覆盖配置文件中的 project（可选）
```

### 8.2 推荐的 CLI 命令

```bash
# 基础部署（生产环境）
npx trigger.dev@latest deploy . --env prod

# 自托管实例
npx trigger.dev@latest deploy . --env prod --api-url https://your-instance.example.com

# Native Build Server（推荐 CI 使用，无需本地 Docker）
npx trigger.dev@latest deploy . --env prod --native-build-server

# 非阻塞模式（部署入队后立即返回）
npx trigger.dev@latest deploy . --env prod --native-build-server --detach

# 跳过自动晋升（蓝绿发布场景）
npx trigger.dev@latest deploy . --env prod --skip-promotion
# 后续手动晋升
npx trigger.dev@latest promote <version>
```

### 8.3 项目根的确定逻辑（CI 注意事项）

| 场景 | 命令 | projectPath | workingDir |
|------|------|-------------|------------|
| 单项目仓库根目录 | `deploy .` | `repo/` | `repo/` |
| Monorepo 子项目 | `deploy packages/trigger` | `repo/packages/trigger` | `repo/packages/trigger` |
| 显式指定配置 | `deploy . --config apps/worker/trigger.config.ts` | `repo/` | `repo/apps/worker/` |

**CI 中务必确保工作目录与 `trigger.config.ts` 的位置关系正确。**

### 8.4 凭据错误的常见排查

| 错误信息 | 原因 | 源码位置 | 解决 |
|----------|------|----------|------|
| `You must login first` | 无 TRIGGER_ACCESS_TOKEN 且本地无 profile | `packages/cli-v3/src/commands/login.ts:283-285` | CI 中设置 TRIGGER_ACCESS_TOKEN |
| `not a Personal Access Token` | TRIGGER_ACCESS_TOKEN 格式错误 | `packages/cli-v3/src/utilities/accessTokens.ts:15-23` | 确保以 `tr_pat_` 开头 |
| `Project not found` | projectRef 与 profile 指向不同实例 | `packages/cli-v3/src/utilities/session.ts:98-101` | 检查 `--profile` 和 `--api-url` |
| `Failed to connect to ...` | API URL 不可达 | `packages/cli-v3/src/commands/deploy.ts:279-281` | 检查 `TRIGGER_API_URL` 或 `--api-url` |

---

## 九、关键源码索引

| 关注点 | 仓库相对路径 |
|--------|-------------|
| 部署命令入口 | `packages/cli-v3/src/commands/deploy.ts` |
| 配置加载 + 项目发现 | `packages/cli-v3/src/config.ts` |
| 用户认证 | `packages/cli-v3/src/commands/login.ts` |
| Profile 配置存储 | `packages/cli-v3/src/utilities/configFiles.ts` |
| 环境变量解析 | `packages/cli-v3/src/utilities/localEnvVars.ts` |
| .env 加载 | `packages/cli-v3/src/utilities/dotEnv.ts` |
| 令牌验证 | `packages/cli-v3/src/utilities/accessTokens.ts` |
| 项目客户端 + 环境 | `packages/cli-v3/src/utilities/session.ts` |
| 代码打包 | `packages/cli-v3/src/build/buildWorker.ts` |
| esbuild 打包 | `packages/cli-v3/src/build/bundle.ts` |
| 入口点发现 | `packages/cli-v3/src/build/entryPoints.ts` |
| contentHash + 产物去重 | `packages/cli-v3/src/build/manifests.ts` |
| Docker 镜像构建 | `packages/cli-v3/src/deploy/buildImage.ts` |
| 归档打包 (Native) | `packages/cli-v3/src/deploy/archiveContext.ts` |
| Git 元信息 | `packages/cli-v3/src/utilities/gitMeta.ts` |
| API 客户端 | `packages/cli-v3/src/apiClient.ts` |
| 常量定义 (CLOUD_API_URL, CONFIG_FILES) | `packages/cli-v3/src/consts.ts` |
| 服务端部署初始化 | `apps/webapp/app/v3/services/initializeDeployment.server.ts` |
| 版本号分配 + 并发重试 | `apps/webapp/app/v3/services/initializeDeployment/createDeploymentWithNextVersion.server.ts` |
| 部署初始化 API 路由 | `apps/webapp/app/routes/api.v1.deployments.ts` |
| 部署 progress API（Native 构建更新 contentHash） | `apps/webapp/app/routes/api.v1.deployments.$deploymentId.progress.ts` |
| 部署 progress 服务端逻辑 | `apps/webapp/app/v3/services/deployment.server.ts` |
| 部署终结服务 | `apps/webapp/app/v3/services/finalizeDeployment.server.ts` |
| 部署终结 V2（含镜像推送） | `apps/webapp/app/v3/services/finalizeDeploymentV2.server.ts` |
| BackgroundWorker 创建 V4 | `apps/webapp/app/v3/services/createDeploymentBackgroundWorkerV4.server.ts` |
| BackgroundWorker 创建 V3（已废弃） | `apps/webapp/app/v3/services/createDeploymentBackgroundWorkerV3.server.ts` |
| Worker 去重与资源创建 | `apps/webapp/app/v3/services/createBackgroundWorker.server.ts` |
| 部署 API schema | `packages/core/src/v3/schemas/api.ts` |
| Dev 模式增量检测 | `packages/cli-v3/src/dev/devSupervisor.ts` |
