# Trigger.dev 幂等键（Idempotency Key）机制深度分析

## 一、核心问题背景

线上系统出现**同一外部事件触发两次任务运行**，需要沿着代码路径逐层核对幂等链路，搞清楚每个环节的职责边界和判断逻辑，为排查提供可追溯的代码证据。

---

## 二、SDK 与 Core 的职责划分

### 2.1 包依赖关系

```
@trigger.dev/sdk/v3
  └─► idempotencyKeys.create / idempotencyKeys.reset  （纯转发层）
        │
@trigger.dev/core/v3
  └─► createIdempotencyKey / resetIdempotencyKey       （实现层）
        │
        ├─► idempotency-key-catalog                     （本地注册表）
        ├─► task-context-api                            （运行时上下文）
        └─► utils/crypto → digestSHA256                 （哈希计算）
```

### 2.2 trigger-sdk：纯转发层

**文件位置：** `packages/trigger-sdk/src/v3/idempotencyKeys.ts:1-7`

```typescript
import { createIdempotencyKey, resetIdempotencyKey, type IdempotencyKey } from "@trigger.dev/core/v3";

export const idempotencyKeys = {
  create: createIdempotencyKey,
  reset: resetIdempotencyKey,
};

export type { IdempotencyKey };
```

**职责：** 仅做 re-export，零逻辑。用户代码 `import { idempotencyKeys } from "@trigger.dev/sdk/v3"` 实际调用的是 core 中的函数。

### 2.3 core：实现层

**文件位置：** `packages/core/src/v3/idempotencyKeys.ts`

#### 2.3.1 createIdempotencyKey 的完整流程

```typescript
// 第127-142行
export async function createIdempotencyKey(
  key: string | string[],
  options?: { scope?: IdempotencyKeyScope }
): Promise<IdempotencyKey> {
  const scope = options?.scope ?? "run";                          // ① 默认 scope = "run"
  const keyArray = Array.isArray(key) ? key : [key];             // ② 标准化为数组
  const userKey = keyArray.join("-");                             // ③ 拼接用户 key

  const idempotencyKey = await generateIdempotencyKey(            // ④ 哈希计算
    keyArray.concat(injectScope(scope))
  );

  idempotencyKeyCatalog.registerKeyOptions(idempotencyKey, {      // ⑤ 本地注册
    key: userKey, scope
  });

  return idempotencyKey as IdempotencyKey;                        // ⑥ 返回 branded string
}
```

#### 2.3.2 injectScope：scope 上下文注入

```typescript
// 第144-161行
function injectScope(scope: IdempotencyKeyScope): string[] {
  switch (scope) {
    case "run": {
      if (taskContext?.ctx) {
        return [taskContext.ctx.run.id];              // 追加 parentRunId
      }
      break;                                          // 无 task context → 不追加
    }
    case "attempt": {
      if (taskContext?.ctx) {
        return [
          taskContext.ctx.run.id,                     // 追加 parentRunId
          taskContext.ctx.attempt.number.toString()   // 追加 attemptNumber
        ];
      }
      break;
    }
  }
  return [];                                          // global → 空数组
}
```

**关键排查线索：** 当在 task 外部调用 `createIdempotencyKey`（如后端 API 路由中），`taskContext?.ctx` 为 undefined，即使 scope 是 `"run"` 也不会追加 parentRunId，此时行为等同于 `"global"`。

#### 2.3.3 generateIdempotencyKey：哈希计算

```typescript
// 第163-165行
async function generateIdempotencyKey(keyMaterial: string[]) {
  return await digestSHA256(keyMaterial.join("-"));
}
```

**文件位置：** `packages/core/src/v3/utils/crypto.ts:7-15`

```typescript
export async function digestSHA256(data: string): Promise<string> {
  const { subtle } = await import("uncrypto");
  const hash = await subtle.digest("SHA-256", new TextEncoder().encode(data));
  return Array.from(new Uint8Array(hash))
    .map((byte) => byte.toString(16).padStart(2, "0"))
    .join("");                          // 返回 64 字符 hex string
}
```

**计算过程示例：**

| 输入 key | scope | task context | join("-") | SHA-256 |
|----------|-------|-------------|-----------|---------|
| `"email-123"` | `global` | 有 | `"email-123"` | `a1b2c3...` |
| `"email-123"` | `run` | runId=`run_abc` | `"email-123-run_abc"` | `d4e5f6...` |
| `"email-123"` | `attempt` | runId=`run_abc`, attempt=2 | `"email-123-run_abc-2"` | `g7h8i9...` |
| `"email-123"` | `run` | **无**（API 路由） | `"email-123"` | `a1b2c3...` ← 同 global！ |

#### 2.3.4 本地 Catalog：原始 key 的注册与提取

**文件位置：** `packages/core/src/v3/idempotency-key-catalog/catalog.ts:1-11`

```typescript
export type IdempotencyKeyScope = "run" | "attempt" | "global";

export type IdempotencyKeyOptions = {
  key: string;
  scope: IdempotencyKeyScope;
};

export interface IdempotencyKeyCatalog {
  registerKeyOptions(hash: string, options: IdempotencyKeyOptions): void;
  getKeyOptions(hash: string): IdempotencyKeyOptions | undefined;
}
```

**作用：** catalog 是进程内内存注册表，用于 `resetIdempotencyKey` 时从 64 字符哈希反查原始 key 和 scope，避免用户手动传回 scope 参数。**跨进程不可用**——在另一个进程中拿到哈希字符串，catalog 里查不到。

#### 2.3.5 服务端存储：idempotencyKeyOptions 字段

V2 引擎在创建 run 时，额外将原始 key 和 scope 存入数据库 `idempotencyKeyOptions` JSON 字段：

**文件位置：** `packages/core/src/v3/schemas/api.ts:152-156`

```typescript
export const IdempotencyKeyOptionsSchema = z.object({
  key: z.string(),
  scope: z.enum(["run", "attempt", "global"]),
});
```

**文件位置：** `internal-packages/run-engine/src/engine/index.ts:619`

```typescript
// taskRun.create data 中包含：
idempotencyKeyOptions,
```

**文件位置：** `packages/core/src/v3/serverOnly/idempotencyKeys.ts`

```typescript
export function getUserProvidedIdempotencyKey(run: {
  idempotencyKey: string | null | undefined;
  idempotencyKeyOptions: unknown;
}): string | undefined {
  const parsed = IdempotencyKeyOptionsSchema.safeParse(run.idempotencyKeyOptions);
  if (parsed.success) {
    return parsed.data.key;               // 优先返回原始 key
  }
  return run.idempotencyKey ?? undefined;  // 降级返回哈希
}
```

**排查价值：** 在数据库中直接查看 `idempotencyKeyOptions` 字段即可知道用户传入的原始 key 和 scope，无需逆向 SHA-256。

---

## 三、TriggerTaskService 的 V1/V2 分流

### 3.1 分流入口

**文件位置：** `apps/webapp/app/v3/services/triggerTask.server.ts:53-79`

```typescript
export class TriggerTaskService extends WithRunEngine {
  public async call(
    taskId: string,
    environment: AuthenticatedEnvironment,
    body: TriggerTaskRequestBody,
    options: TriggerTaskServiceOptions = {},
    version?: RunEngineVersion
  ): Promise<TriggerTaskServiceResult | undefined> {
    return await this.traceWithEnv("call()", environment, async (span) => {
      const v = await determineEngineVersion({            // ← 决定 V1 还是 V2
        environment,
        workerVersion: body.options?.lockToVersion,
        engineVersion: version,
      });

      switch (v) {
        case "V1": {
          return await this.callV1(taskId, environment, body, options);
        }
        case "V2": {
          return await this.callV2(taskId, environment, body, options);
        }
      }
    });
  }
}
```

### 3.2 determineEngineVersion 决策链

**文件位置：** `apps/webapp/app/v3/engineVersion.server.ts:17-76`

```
determineEngineVersion({ environment, workerVersion, engineVersion })
    │
    ├─► ① 显式指定 engineVersion → 直接使用
    │
    ├─► ② project.engine === V1 → 全部走 V1
    │
    ├─► ③ 指定了 workerVersion → 查该 worker 的 engine 字段
    │
    ├─► ④ DEV 环境 → 查最新 BackgroundWorker 的 engine 字段
    │
    ├─► ⑤ DEPLOYED 环境 → 查当前部署的 engine 版本
    │
    └─► ⑥ 兜底 → 使用 project.engine
```

**排查注意：** 同一个 project 可能部分 worker 是 V1、部分是 V2。V1 和 V2 的幂等逻辑有差异。

### 3.3 V1 vs V2 幂等处理的差异

| 特性 | V1 (`TriggerTaskServiceV1`) | V2 (`RunEngineTriggerTaskService`) |
|------|---------------------------|-------------------------------------|
| **文件位置** | `apps/webapp/app/v3/services/triggerTaskV1.server.ts` | `apps/webapp/app/runEngine/services/triggerTask.server.ts` |
| **幂等检查位置** | 内联在 `call()` 方法中 | 独立 `IdempotencyKeyConcern` 类 |
| **幂等键 TTL 过期** | ✅ 检查 `idempotencyKeyExpiresAt` | ✅ 检查 `idempotencyKeyExpiresAt` |
| **状态清除** | ❌ **未实现** `shouldIdempotencyKeyBeCleared` | ✅ 检查 `shouldIdempotencyKeyBeCleared` |
| **过期时清除字段** | 仅清 `idempotencyKey`（保留 `idempotencyKeyExpiresAt`） | 同时清 `idempotencyKey` 和 `idempotencyKeyExpiresAt` |
| **P2002 重试** | 手动解析 `error.meta.target` 三元组 | 抛出 `RunDuplicateIdempotencyKeyError` |
| **Run TTL 调度** | `ExpireEnqueuedRunService.enqueue()` | `ttlSystem.scheduleExpireRun()` |
| **idempotencyKeyOptions** | ❌ 不存储 | ✅ 存入数据库 |

#### V1 幂等代码（关键缺陷）

**文件位置：** `apps/webapp/app/v3/services/triggerTaskV1.server.ts:70-113`

```typescript
const idempotencyKey = options.idempotencyKey ?? body.options?.idempotencyKey;
const idempotencyKeyExpiresAt =
  options.idempotencyKeyExpiresAt ??
  resolveIdempotencyKeyTTL(body.options?.idempotencyKeyTTL) ??
  new Date(Date.now() + 24 * 60 * 60 * 1000 * 30);

const existingRun = idempotencyKey
  ? await this._prisma.taskRun.findFirst({
      where: {
        runtimeEnvironmentId: environment.id,
        idempotencyKey,
        taskIdentifier: taskId,
      },
    })
  : undefined;

if (existingRun) {
  if (
    existingRun.idempotencyKeyExpiresAt &&
    existingRun.idempotencyKeyExpiresAt < new Date()
  ) {
    // TTL 过期：清除 key
    await this._prisma.taskRun.update({
      where: { id: existingRun.id },
      data: { idempotencyKey: null },       // ← 注意：只清 key，没清 ExpiresAt
    });
  } else {
    // ← 没有 shouldIdempotencyKeyBeCleared 检查！
    return { run: existingRun, isCached: true };
  }
}
```

**V1 的关键缺陷：** 不检查 `shouldIdempotencyKeyBeCleared`。当 run 处于 FAILED 或 EXPIRED 状态时，V1 仍然会命中缓存返回旧 run，而不是清除幂等键创建新 run。这可能导致：
- 失败的 run 阻止重试（幂等键未被清除）
- EXPIRED 的 run 阻止重新触发

#### V2 幂等代码（完整逻辑）

**文件位置：** `apps/webapp/app/runEngine/concerns/idempotencyKeys.server.ts:20-137`

```typescript
async handleTriggerRequest(
  request: TriggerTaskRequest,
  parentStore: string | undefined
): Promise<IdempotencyKeyConcernResult> {
  const idempotencyKey = request.options?.idempotencyKey ?? request.body.options?.idempotencyKey;
  const idempotencyKeyExpiresAt =
    request.options?.idempotencyKeyExpiresAt ??
    resolveIdempotencyKeyTTL(request.body.options?.idempotencyKeyTTL) ??
    new Date(Date.now() + 24 * 60 * 60 * 1000 * 30);

  if (!idempotencyKey) {
    return { isCached: false, idempotencyKey, idempotencyKeyExpiresAt };
  }

  const existingRun = await this.prisma.taskRun.findFirst({
    where: {
      runtimeEnvironmentId: request.environment.id,
      idempotencyKey,
      taskIdentifier: request.taskId,
    },
    include: { associatedWaitpoint: true },
  });

  if (existingRun) {
    // ① 幂等键 TTL 过期检查
    if (existingRun.idempotencyKeyExpiresAt && existingRun.idempotencyKeyExpiresAt < new Date()) {
      await this.prisma.taskRun.updateMany({
        where: { id: existingRun.id, idempotencyKey },
        data: { idempotencyKey: null, idempotencyKeyExpiresAt: null },
      });
      return { isCached: false, idempotencyKey, idempotencyKeyExpiresAt };
    }

    // ② Run 状态检查（V1 缺少此步骤！）
    if (shouldIdempotencyKeyBeCleared(existingRun.status)) {
      await this.prisma.taskRun.updateMany({
        where: { id: existingRun.id, idempotencyKey },
        data: { idempotencyKey: null, idempotencyKeyExpiresAt: null },
      });
      return { isCached: false, idempotencyKey, idempotencyKeyExpiresAt };
    }

    // ③ 命中缓存：triggerAndWait 场景处理
    if (resumeParentOnCompletion && parentRunId) {
      let associatedWaitpoint = existingRun.associatedWaitpoint;
      if (!associatedWaitpoint) {
        associatedWaitpoint = await this.engine.getOrCreateRunWaitpoint({...});
      }
      await this.engine.blockRunWithWaitpoint({...});
    }

    return { isCached: true, run: existingRun };
  }

  return { isCached: false, idempotencyKey, idempotencyKeyExpiresAt };
}
```

---

## 四、两种 TTL 的优先级计算与排查影响

### 4.1 幂等键 TTL（idempotencyKeyExpiresAt）

**计算位置：** `apps/webapp/app/runEngine/concerns/idempotencyKeys.server.ts:25-28`（V2）
和 `apps/webapp/app/v3/services/triggerTaskV1.server.ts:71-74`（V1）

```
idempotencyKeyExpiresAt 优先级：
  ① request.options.idempotencyKeyExpiresAt   → 服务端直接传入 Date 对象
  ② resolveIdempotencyKeyTTL(body.options?.idempotencyKeyTTL)  → 客户端 TTL 字符串
  ③ new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)          → 兜底 30 天
```

**TTL 字符串解析：** `apps/webapp/app/utils/idempotencyKeys.server.ts:10-41`

```typescript
export function resolveIdempotencyKeyTTL(ttl: string | undefined | null): Date | undefined {
  if (!ttl) return undefined;
  const match = ttl.match(/^(\d+)([smhd])$/);  // 只支持 整数+s/m/h/d
  if (!match) return undefined;                  // 无效格式静默返回 undefined → 走兜底 30 天
  // ...
}
```

**排查注意：**
- 传入 `"5min"` 或 `"1.5h"` 等非标准格式 → 静默降级为 30 天
- 传入 `"0s"` → 立即过期
- 不传 `idempotencyKeyTTL` → 30 天

### 4.2 Run TTL（控制 EXPIRED 状态）

**V2 计算位置：** `apps/webapp/app/runEngine/services/triggerTask.server.ts:282-291`

```typescript
// Resolve TTL with precedence: per-trigger > task-level > dev default
let ttl: string | undefined;

if (body.options?.ttl !== undefined) {
  ttl =
    typeof body.options.ttl === "number"
      ? stringifyDuration(body.options.ttl)     // ① 客户端显式指定（数字 → 字符串）
      : body.options.ttl;                       // ① 客户端显式指定（字符串）
} else {
  ttl = taskTtl ?? (environment.type === "DEVELOPMENT" ? "10m" : undefined);
  // ② task 定义中配置的 TTL
  // ③ DEV 环境兜底 10 分钟
  // ④ PROD 环境无兜底 → ttl = undefined → 永不过期
}
```

**V1 计算位置：** `apps/webapp/app/v3/services/triggerTaskV1.server.ts:78-81`

```typescript
const ttl =
  typeof body.options?.ttl === "number"
    ? stringifyDuration(body.options?.ttl)
    : body.options?.ttl ?? (environment.type === "DEVELOPMENT" ? "10m" : undefined);
```

**V1 缺少 task-level TTL**，直接跳到环境默认值。

### 4.3 Run TTL 如何触发 EXPIRED

**V1 路径：**

```
触发 run → status = PENDING
    ↓
ExpireEnqueuedRunService.enqueue(runId, expireAt)   ← 创建延迟任务
    ↓
[等待 run.ttl 时间]
    ↓
ExpireEnqueuedRunService.call(runId)
    ↓
检查: status === "PENDING" && lockedAt === null
    ↓ 满足条件
FinalizeTaskRunService.call({ status: "EXPIRED" })
```

**V2 路径：**

```
触发 run → status = PENDING
    ↓
[DEV 环境] ttlSystem.scheduleExpireRun({ runId, ttl })   ← 单独调度
[PROD 环境] enqueueRun({ includeTtl: true })              ← 批量 TTL 由队列排序集处理
    ↓
[等待 run.ttl 时间]
    ↓
ttlSystem.expireRun({ runId })
    ↓
检查: status === "PENDING" && lockedAt === null && !isExecuting
    ↓ 满足条件
prisma.taskRun.update({ status: "EXPIRED" })
```

### 4.4 EXPIRED 触发的完整前置条件

| 条件 | 代码位置 | 说明 |
|------|---------|------|
| `status === "PENDING"` | `ttlSystem.ts:45` | 只有过期 PENDING 状态的 run |
| `lockedAt === null` | `ttlSystem.ts:52` | 已被 worker 锁定的不会被过期 |
| `!isExecuting` | `ttlSystem.ts:31` | 正在执行的不会被过期 |
| `ttl` 字段有值 | V1: `triggerTaskV1.server.ts:523`; V2: `triggerTask.server.ts:806` | 无 ttl 则不调度过期 |

**排查关键影响：**
- Run 进入 `EXECUTING` 后即使超过 run.ttl 也**不会**变成 EXPIRED
- Worker 锁定（`lockedAt !== null`）的 run 也**不会**变成 EXPIRED
- PROD 环境默认 `ttl = undefined`，run **永远不会** EXPIRED
- DEV 环境默认 `ttl = "10m"`，10 分钟内未被 worker 拉取就会 EXPIRED

### 4.5 EXPIRED → 幂等键清除 → 允许重触发的连锁反应

```
Run 在 DEV 环境队列中等待超过 10 分钟
    ↓
ttlSystem 检查: PENDING + unlocked + not executing → 全部满足
    ↓
Run status 变为 EXPIRED
    ↓
[同事件再次触发]
    ↓
IdempotencyKeyConcern.handleTriggerRequest()
    ↓
findFirst 查到 EXPIRED 状态的 existingRun
    ↓
shouldIdempotencyKeyBeCleared("EXPIRED") → true   ← 单独判断，不在 FAILED_RUN_STATUSES 中
    ↓
清除 idempotencyKey + idempotencyKeyExpiresAt
    ↓
创建新 run ✓  ← 幂等键已被清除，不会阻止新 run
```

---

## 五、shouldIdempotencyKeyBeCleared 状态分类详解

### 5.1 代码定义

**文件位置：** `apps/webapp/app/v3/taskStatus.ts:133-134`

```typescript
export function shouldIdempotencyKeyBeCleared(status: TaskRunStatus): boolean {
  return isFailedRunStatus(status) || status === "EXPIRED";
}
```

### 5.2 FAILED_RUN_STATUSES 定义

**文件位置：** `apps/webapp/app/v3/taskStatus.ts:54-60`

```typescript
export const FAILED_RUN_STATUSES = [
  "INTERRUPTED",
  "COMPLETED_WITH_ERRORS",
  "SYSTEM_FAILURE",
  "CRASHED",
  "TIMED_OUT",
] satisfies TaskRunStatus[];
```

**注意：** `EXPIRED` 不在此列表中！它是 `shouldIdempotencyKeyBeCleared` 通过 `|| status === "EXPIRED"` 单独追加的。

### 5.3 完整状态分类矩阵

| 状态 | FINAL | FAILED | FATAL | 清除幂等键 | 原因 |
|------|-------|--------|-------|-----------|------|
| `COMPLETED_SUCCESSFULLY` | ✅ | ❌ | ❌ | ❌ | 成功，保留保护 |
| `CANCELED` | ✅ | ❌ | ❌ | ❌ | 用户主动取消，保留保护 |
| `INTERRUPTED` | ✅ | ✅ | ❌ | ✅ | `isFailedRunStatus` → true |
| `COMPLETED_WITH_ERRORS` | ✅ | ✅ | ❌ | ✅ | `isFailedRunStatus` → true |
| `SYSTEM_FAILURE` | ✅ | ✅ | ✅ | ✅ | `isFailedRunStatus` → true |
| `CRASHED` | ✅ | ✅ | ✅ | ✅ | `isFailedRunStatus` → true |
| `TIMED_OUT` | ✅ | ✅ | ❌ | ✅ | `isFailedRunStatus` → true |
| **`EXPIRED`** | ✅ | **❌** | ❌ | **✅** | **`status === "EXPIRED"` 单独判断** |
| `PENDING` | ❌ | ❌ | ❌ | ❌ | 进行中 |
| `EXECUTING` | ❌ | ❌ | ❌ | ❌ | 进行中 |
| `RETRYING_AFTER_FAILURE` | ❌ | ❌ | ❌ | ❌ | 进行中 |
| `WAITING_TO_RESUME` | ❌ | ❌ | ❌ | ❌ | 等待恢复 |

### 5.4 设计意图解读

- **成功/取消 → 保留幂等键：** 防止重复执行已完成的任务
- **失败类 → 清除幂等键：** 允许在失败后重试，幂等保护不应阻碍错误恢复
- **EXPIRED → 清除幂等键：** 队列超时通常是因为外部原因（worker 未运行），应允许重新触发
- **进行中 → 保留幂等键：** 防止并发重复

---

## 六、并发触发场景分析

### 6.1 V2 并发保护的三层机制

```
并发请求 A 和 B（相同 idempotencyKey）
    │
    ├─► 第1层：IdempotencyKeyConcern.findFirst（读屏障）
    │      A 查不到 existingRun → isCached: false
    │      B 查不到 existingRun → isCached: false
    │      ↓ （两个请求都通过）
    │
    ├─► 第2层：数据库唯一约束（写屏障）
    │      A 写入 taskRun 成功
    │      B 写入 taskRun → P2002 错误
    │      ↓
    └─► 第3层：异常捕获与重试（容错）
           V2: RunDuplicateIdempotencyKeyError
           V1: 手动解析 error.meta.target 三元组
           ↓
           重试 call() → 这次第1层会命中 existingRun
```

### 6.2 V1 的 P2002 重试代码

**文件位置：** `apps/webapp/app/v3/services/triggerTaskV1.server.ts:606-659`

```typescript
catch (error) {
  if (error instanceof Prisma.PrismaClientKnownRequestError) {
    if (error.code === "P2002") {
      const target = error.meta?.target;
      if (Array.isArray(target) && target[0]?.includes("oneTimeUseToken")) {
        // oneTimeUseToken 冲突 → 抛异常
      } else if (
        Array.isArray(target) &&
        target.length == 3 &&
        target[0] == "runtimeEnvironmentId" &&
        target[1] == "taskIdentifier" &&
        target[2] == "idempotencyKey"
      ) {
        // 幂等键三元组冲突 → 重试
        return await this.call(taskId, environment, body, options, attempt + 1);
      } else {
        // 其他唯一约束冲突 → 抛异常
      }
    }
  }
  throw error;
}
```

### 6.3 V2 的 P2002 重试代码

**文件位置：** `apps/webapp/app/runEngine/services/triggerTask.server.ts:574-584`

```typescript
catch (error) {
  if (error instanceof RunDuplicateIdempotencyKeyError) {
    // 重试调用，这次会命中缓存逻辑返回已有 run
    return await this.call({
      taskId,
      environment,
      body,
      options: { ...options, runFriendlyId },
      attempt: attempt + 1,
    });
  }
  // ...
}
```

**V2 改进：** P2002 检测在 `engine.trigger()` 内完成，抛出语义明确的 `RunDuplicateIdempotencyKeyError`，上层不需要手动解析 `error.meta.target`。

---

## 七、与重试机制的交互

### 7.1 父任务重试 vs 子任务幂等

```
父任务 Attempt 1
    ├─► trigger 子任务（idempotencyKey=SHA256("email-123-run_abc")）
    │    创建子 run #child-001
    └─► 父任务失败

父任务 Attempt 2（重试）
    └─► trigger 子任务（idempotencyKey=SHA256("email-123-run_abc")）
         │  ← scope="run" 时 hash 相同，因为 parentRunId 未变
         ├─► 查询 existingRun → 找到 #child-001
         ├─► 检查状态 → 进行中/成功
         └─► 返回已有 run，不创建新 run

结果：子任务只执行一次 ✓
```

### 7.2 scope 对重试的影响

| Scope | Hash 计算 | 重试时 Hash 是否相同 | 重试时行为 |
|-------|----------|---------------------|-----------|
| `run` | `key + parentRunId` | ✅ 相同（parentRunId 不变） | 重用同一个子 run |
| `attempt` | `key + parentRunId + attemptNumber` | ❌ 不同（attemptNumber 变了） | 每次重试创建新的子 run |
| `global` | `key` | ✅ 相同 | 全局唯一，跨所有父 run |

### 7.3 子任务失败后的重试

```
子任务失败（COMPLETED_WITH_ERRORS / CRASHED 等）
    ↓
V2: shouldIdempotencyKeyBeCleared(FAILED) → true → 清除幂等键
V1: 不检查此条件 → 返回 isCached: true（Bug！）
    ↓
下次触发时：
V2: 创建新 run ✓
V1: 返回失败的旧 run ✗
```

### 7.4 子任务队列超时（EXPIRED）后的重试

```
子任务 PENDING → 超过 run.ttl → EXPIRED
    ↓
V2: shouldIdempotencyKeyBeCleared("EXPIRED") → true → 清除幂等键 → 允许新 run
V1: 不检查此条件 → 返回 EXPIRED 的旧 run（Bug！）
```

---

## 八、idempotencyKeyOptions 缺失时的 fallback 判定路径

### 8.1 什么情况下 idempotencyKeyOptions 会缺失？

`idempotencyKeyOptions` 是 V2 引擎才会写入数据库的 JSON 字段，包含原始 key 和 scope。以下场景此字段会缺失：

| 场景 | idempotencyKeyOptions | idempotencyKey（哈希） |
|------|----------------------|-----------------------|
| 经 SDK createIdempotencyKey 创建 + V2 引擎 | ✅ 有 `{key, scope}` | ✅ 64 字符 |
| 直接传 64 字符哈希 + V2 引擎 | ❌ 缺失 | ✅ 64 字符 |
| 非 SDK 客户端（curl、Python 等）+ V2 引擎 | ❌ 缺失 | ✅ 64 字符（用户自行计算） |
| V1 引擎（任何客户端） | ❌ 缺失 | ✅ 64 字符 |
| 跨进程重置（catalog 失效） | ❌ 缺失（如果是新进程） | ✅ 64 字符 |

### 8.2 fallback 判定路径

当 `idempotencyKeyOptions` 缺失时，系统按以下路径判定 scope 和去重行为：

```
查询 run 记录
    │
    ├─► idempotencyKeyOptions 存在？
    │    ├─► 是 → 从 options.key 和 options.scope 获取（确定路径）
    │    └─► 否 → 进入 fallback 路径
    │
    └─► fallback：只有 idempotencyKey 哈希值
         │
         ├─► 【Scope 判定】无法从哈希反推 scope → 视为 global scope
         │      因为 global scope 的哈希 = SHA256(userKey)，不包含 parentRunId/attemptNumber
         │
         ├─► 【去重行为判定】
         │    ├─► 数据库唯一约束：三元组 [envId, taskId, hash] → 正常去重
         │    ├─► TTL 过期检查：正常检查 idempotencyKeyExpiresAt
         │    └─► 状态清除检查：V2 正常检查 shouldIdempotencyKeyBeCleared（V1 不检查）
         │
         └─► 【reset 时判定】见 8.3 节
```

### 8.3 resetIdempotencyKey 的 fallback 逻辑

**文件位置：** `packages/core/src/v3/idempotencyKeys.ts:223-268`

```typescript
export async function resetIdempotencyKey(
  taskIdentifier: string,
  idempotencyKey: IdempotencyKey | string | string[],
  options?: ResetIdempotencyKeyOptions,
  requestOptions?: ZodFetchOptions
): Promise<{ id: string }> {
  // 路径①：64 字符哈希 → 直接使用
  if (typeof idempotencyKey === "string" && idempotencyKey.length === 64) {
    return client.resetIdempotencyKey(taskIdentifier, idempotencyKey, requestOptions);
  }

  // 路径②：尝试从 catalog 提取
  const attachedOptions = typeof idempotencyKey === "string"
    ? getIdempotencyKeyOptions(idempotencyKey)   // ← 跨进程时返回 undefined
    : undefined;

  const scope = attachedOptions?.scope ?? options?.scope ?? "run";  // ← fallback: "run"

  // 路径③：根据 scope 重建 hash
  let scopeSuffix: string[] = [];
  switch (scope) {
    case "run": {
      const parentRunId = options?.parentRunId ?? taskContext?.ctx?.run.id;
      if (!parentRunId) throw new Error("parentRunId required for 'run' scope");
      scopeSuffix = [parentRunId];
      break;
    }
    // ...
  }

  const hash = await generateIdempotencyKey(keyArray.concat(scopeSuffix));
  return client.resetIdempotencyKey(taskIdentifier, hash, requestOptions);
}
```

**重置时的 fallback 路径：**

```
调用 resetIdempotencyKey(key)
    │
    ├─► key 是 64 字符哈希？
    │    ├─► 是 → 直接调用 API，不需要 scope（路径①）
    │    └─► 否 → 继续
    │
    ├─► catalog 中查得到 key 的 options？
    │    ├─► 是 → 使用 catalog 中的 scope 重建 hash（路径②）
    │    └─► 否 → 继续
    │
    ├─► options.scope 有值？
    │    ├─► 是 → 使用 options.scope 重建 hash
    │    └─► 否 → 使用默认 scope="run"
    │
    └─► 重建 hash 后调用 API（路径③）
```

**关键风险：** 当 catalog 失效且用户传入原始 key 字符串时，`scope` fallback 为 `"run"`，如果原始创建时是 `"global"` scope，重建的 hash 会不同（多了 parentRunId 后缀），导致重置失败。

---

## 九、三种特殊场景的 scope 与去重行为分析

### 9.1 场景一：直接传入 64 位预哈希 key

**场景描述：** 用户绕过 `idempotencyKeys.create()`，直接传入预先计算好的 64 字符 SHA-256 哈希。

```typescript
// 用户代码（不推荐）
await task.trigger(payload, {
  idempotencyKey: "a1b2c3d4e5f6..."  // 直接传 64 字符哈希
});
```

**代码路径分析：**

**创建路径（`createIdempotencyKey` 没被调用）：**
- `idempotencyKeyOptions` **不会被设置**（V2 engine 创建 run 时，只有调用了 SDK create 才会设置 options）
- 实际是 trigger 时在 `engine.trigger()` 中设置 `idempotencyKeyOptions`，而非在 SDK create 时

**触发路径（`IdempotencyKeyConcern.handleTriggerRequest`）：**
```typescript
const existingRun = await this.prisma.taskRun.findFirst({
  where: {
    runtimeEnvironmentId: request.environment.id,
    idempotencyKey,           // 直接使用 64 字符哈希
    taskIdentifier: request.taskId,
  },
});
```

**判定结果：**
| 维度 | 行为 |
|------|------|
| **Scope 判定** | 无法反推 → 视为 global scope（因为 hash 不包含 scope 信息） |
| **去重行为** | ✅ 正常：三元组匹配 + TTL 检查 + 状态检查 |
| **Reset 行为** | ✅ 正常：直接传 64 字符哈希走路径①，无需 scope |
| **Dashboard 显示** | ❌ 只显示哈希，不显示原始 key |

**createIdempotencyKey 对 64 字符的处理：**
**文件位置：** `packages/core/src/v3/idempotencyKeys.ts:127-142`

```typescript
// createIdempotencyKey 没有 64 字符检测！
// 即使传入的 key 已经是 64 字符哈希，仍然会再次 SHA-256
const idempotencyKey = await generateIdempotencyKey(
  keyArray.concat(injectScope(scope))  // keyArray = ["a1b2c3...（64 字符）"]
);

// 结果：SHA256(SHA256(original) + scopeSuffix) → 双重哈希
```

**⚠️ 重要区别：**
- `resetIdempotencyKey` 有 64 字符检测（第 224 行）
- `createIdempotencyKey` **没有** 64 字符检测 → 传入已有的 64 字符哈希会被双重哈希

### 9.2 场景二：非 SDK 客户端（curl、Python、Go 等）

**场景描述：** 使用非 JavaScript SDK 的客户端直接调用 Trigger API，自行计算 SHA-256 哈希。

```bash
# curl 示例
curl -X POST https://api.trigger.dev/v3/tasks/my-task/trigger \
  -H "Authorization: Bearer ${TRIGGER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "payload": {"foo": "bar"},
    "options": {
      "idempotencyKey": "a1b2c3d4e5f6..."  # 客户端自行计算 SHA256("user-key")
    }
  }'
```

**代码路径分析：**

**触发路径（`IdempotencyKeyConcern`）：**
- 非 SDK 客户端不会传入 `idempotencyKeyOptions`（SDK 才会设置）
- 服务端只用 `idempotencyKey`（哈希）进行匹配

**V2 engine 创建 run 时：**
**文件位置：** `internal-packages/run-engine/src/engine/index.ts:619`

```typescript
// 创建 run 时的 data 中：
idempotencyKey,                // 来自 API body.options.idempotencyKey
idempotencyKeyOptions,         // 来自 API body.options.idempotencyKeyOptions
                               // 非 SDK 客户端不会传此字段
```

**判定结果：**
| 维度 | 行为 |
|------|------|
| **Scope 判定** | 客户端自行决定 scope 行为。如果客户端不追加 parentRunId/attemptNumber，行为等同于 global |
| **去重行为** | ✅ 正常：三元组匹配 + TTL 检查 + 状态检查（V2） |
| **idempotencyKeyOptions** | ❌ 缺失（客户端没传） |
| **Reset 行为** | ✅ 正常：直接传 64 字符哈希走路径① |
| **Dashboard 显示** | ❌ 只显示哈希，不显示原始 key |

**非 SDK 客户端的 scope 实现建议：**
```python
# Python 客户端正确实现 global scope
import hashlib
user_key = "email-123"
idempotency_key = hashlib.sha256(user_key.encode()).hexdigest()  # 64 字符

# Python 客户端正确实现 run scope
parent_run_id = "run_abc123"
material = f"{user_key}-{parent_run_id}"
idempotency_key = hashlib.sha256(material.encode()).hexdigest()
```

### 9.3 场景三：跨进程 catalog 失效

**场景描述：** 进程 A 中 `idempotencyKeys.create("email-123", {scope: "run"})` 得到哈希，传给进程 B，进程 B 中调用 `idempotencyKeys.reset("task-id", hash)`。

**catalog 本质：**
**文件位置：** `packages/core/src/v3/idempotency-key-catalog/catalog.ts`

```typescript
export interface IdempotencyKeyCatalog {
  registerKeyOptions(hash: string, options: IdempotencyKeyOptions): void;
  getKeyOptions(hash: string): IdempotencyKeyOptions | undefined;
}

// 实现：进程内 Map<string, IdempotencyKeyOptions>
// 跨进程不可用！
```

**代码路径分析：**

进程 A：
```typescript
const key = await idempotencyKeys.create("email-123", { scope: "run" });
// catalog["a1b2c3..."] = { key: "email-123", scope: "run" }
```

进程 B（收到 key 哈希字符串）：
```typescript
// 情况 1：直接传 64 字符哈希
await idempotencyKeys.reset("task-id", "a1b2c3...");
// → 路径①：检测到 64 字符 → 直接使用 ✅ 正确

// 情况 2：传原始 key 字符串，期望重置
await idempotencyKeys.reset("task-id", "email-123");
// → catalog 中查不到 → scope fallback = "run"
// → 需要 parentRunId，用户没传 → 抛出 Error: parentRunId required ❌

// 情况 3：传原始 key + 显式 scope 和 parentRunId
await idempotencyKeys.reset("task-id", "email-123", {
  scope: "run",
  parentRunId: "run_abc123"
});
// → scope = "run"，parentRunId = "run_abc123"
// → 重建 hash = SHA256("email-123-run_abc123") ✅ 正确
```

**判定结果：**
| 场景 | 行为 |
|------|------|
| **跨进程传哈希 + reset** | ✅ 正常：64 字符走路径① |
| **跨进程传原始 key + reset（缺 options）** | ❌ 失败：scope fallback 为 run，需 parentRunId |
| **跨进程传原始 key + reset（带 scope 和 parentRunId）** | ✅ 正常：重建正确 hash |
| **跨进程触发（trigger）** | ✅ 正常：只需要哈希值，不需要 catalog |

---

## 十、batchTriggerV3 相同 key 跨 task 的误配风险分析

### 10.1 batchTriggerV3 的幂等处理逻辑

**文件位置：** `apps/webapp/app/v3/services/batchTriggerV3.server.ts:323-418`

`BatchTriggerV3Service.#prepareRunData()` 负责为每个 batch item 检查缓存：

```typescript
async #prepareRunData(environment, body) {
  // batchTriggerAndWait 场景：不做缓存检查
  if (body?.dependentAttempt) {
    return body.items.map((item) => ({
      id: generateFriendlyId("run"),
      isCached: false,
      idempotencyKey: undefined,
      taskIdentifier: item.task,
    }));
  }

  // 步骤①：按 taskIdentifier 分组
  const itemsByTask = body.items.reduce((acc, item) => {
    if (!item.options?.idempotencyKey) return acc;
    if (!acc[item.task]) acc[item.task] = [];
    acc[item.task].push(item);
    return acc;
  }, {});

  // 步骤②：按 task 分别查询缓存
  const cachedRuns = await Promise.all(
    Object.entries(itemsByTask).map(([taskIdentifier, items]) =>
      this._prisma.taskRun.findMany({
        where: {
          runtimeEnvironmentId: environment.id,
          taskIdentifier,                  // ✅ 每个查询都带 taskIdentifier
          idempotencyKey: {
            in: items.map((i) => i.options?.idempotencyKey).filter(Boolean),
          },
        },
        select: {
          friendlyId: true,
          idempotencyKey: true,
          idempotencyKeyExpiresAt: true,
          // ❌ 没有 select taskIdentifier！
        },
      })
    )
  ).then((results) => results.flat());  // flat 后失去 task 归属信息

  // 步骤③：为每个 item 匹配缓存
  const runs = body.items.map((item) => {
    const cachedRun = cachedRuns.find(
      (r) => r.idempotencyKey === item.options?.idempotencyKey  // ⚠️ 只按 key 匹配！
    );
    // ...
  });
}
```

### 10.2 误配风险：相同 key 跨不同 task

**风险场景：** batch 中包含多个不同 task，不同 task 使用相同的 idempotencyKey。

```typescript
// 用户代码
await tasks.batchTrigger([
  { task: "send-email", payload: { to: "a@b.com" }, options: { idempotencyKey: "event-123" } },
  { task: "send-sms", payload: { to: "+12345" }, options: { idempotencyKey: "event-123" } },
]);
```

**数据库状态（触发前）：**
- run1: task=`send-email`, idempotencyKey=`SHA256("event-123")`, status=`COMPLETED_SUCCESSFULLY`
- run2: task=`send-sms`, idempotencyKey=null

**匹配过程：**
1. 分组：`{ "send-email": [item1], "send-sms": [item2] }`
2. 查询：
   - 查询 send-email → 返回 [run1] { idempotencyKey: abc }
   - 查询 send-sms → 返回 [] （无缓存）
3. flat → `cachedRuns = [ {idempotencyKey: abc} ]` （没有 taskIdentifier 信息！）
4. 匹配 item1（send-email, key=abc）→ 找到 run1 ✅ 正确
5. 匹配 item2（send-sms, key=abc）→ **也找到 run1！** ❌ **误配！**

**问题根源：**
- `select` 没有包含 `taskIdentifier`（第 363-368 行）
- `find` 只按 `idempotencyKey` 匹配，没有校验 `taskIdentifier`

**后果：**
- item2（send-sms）错误地返回了 send-email 的 run 作为缓存
- **不会创建 send-sms 的 run** → **send-sms 任务被静默跳过！**
- 数据库唯一约束是 `[envId, taskId, key]`，send-sms 其实可以创建新 run，但 batch 逻辑误判为缓存命中

### 10.3 风险验证：代码证据

**`cachedRuns` 的结构中没有 taskIdentifier：**
**文件位置：** `batchTriggerV3.server.ts:363-368`

```typescript
select: {
  friendlyId: true,
  idempotencyKey: true,
  idempotencyKeyExpiresAt: true,
  // ❌ 缺少 taskIdentifier: true
},
```

**匹配时没有校验 task：**
**文件位置：** `batchTriggerV3.server.ts:378-379`

```typescript
const cachedRun = cachedRuns.find(
  (r) => r.idempotencyKey === item.options?.idempotencyKey  // ⚠️ 没检查 taskIdentifier
);
```

### 10.4 batchTriggerV3 的其他幂等缺陷

**缺陷 1：缺少 `shouldIdempotencyKeyBeCleared` 检查**
**文件位置：** `batchTriggerV3.server.ts:381-391`

```typescript
if (cachedRun) {
  // 只检查了 TTL 过期
  if (cachedRun.idempotencyKeyExpiresAt && cachedRun.idempotencyKeyExpiresAt < new Date()) {
    expiredRunIds.add(cachedRun.friendlyId);
    return { id: generateFriendlyId("run"), isCached: false, ... };
  }
  // ⚠️ 没有检查 shouldIdempotencyKeyBeCleared(status)！
  return { id: cachedRun.friendlyId, isCached: true, ... };
}
```

这意味着：
- FAILED/EXPIRED 状态的旧 run 仍会被当作缓存命中
- 行为与 V1 引擎相同，存在相同的 Bug

**缺陷 2：幂等键过期时只清 key，没清 ExpiresAt**
**文件位置：** `batchTriggerV3.server.ts:410-414`

```typescript
if (expiredRunIds.size) {
  await this._prisma.taskRun.updateMany({
    where: { friendlyId: { in: Array.from(expiredRunIds) } },
    data: { idempotencyKey: null },  // ⚠️ 没清 idempotencyKeyExpiresAt
  });
}
```

与 V1 相同，只清 key 没清 ExpiresAt 字段。

### 10.5 误配风险速查表

| 场景 | 是否误配 | 后果 |
|------|---------|------|
| 同一 batch 内不同 task 用相同 key | ✅ 误配 | 后匹配的 task 被静默跳过 |
| 不同 batch 跨时间用相同 key（不同 task） | ❌ 不误配 | 数据库三元组约束阻止 |
| 同一 batch 内同一 task 用相同 key | ❌ 不误配 | 先匹配的命中缓存 |
| batchTriggerAndWait（有 dependentAttempt） | ❌ 不误配 | 直接跳过缓存检查 |

---

## 十一、幂等键的 Reset 机制

### 11.1 SDK 层面的 reset

**文件位置：** `packages/core/src/v3/idempotencyKeys.ts:215-269`

```typescript
export async function resetIdempotencyKey(
  taskIdentifier: string,
  idempotencyKey: IdempotencyKey | string | string[],
  options?: ResetIdempotencyKeyOptions,
  requestOptions?: ZodFetchOptions
): Promise<{ id: string }> {
  const client = apiClientManager.clientOrThrow();

  // 如果已是 64 字符哈希 → 直接使用
  if (typeof idempotencyKey === "string" && idempotencyKey.length === 64) {
    return client.resetIdempotencyKey(taskIdentifier, idempotencyKey, requestOptions);
  }

  // 尝试从 catalog 提取原始 key 和 scope
  const attachedOptions = typeof idempotencyKey === "string"
    ? getIdempotencyKeyOptions(idempotencyKey)
    : undefined;

  const scope = attachedOptions?.scope ?? options?.scope ?? "run";
  const keyArray = Array.isArray(idempotencyKey)
    ? idempotencyKey
    : [attachedOptions?.key ?? String(idempotencyKey)];

  // 重建 scope 后缀（与创建时相同的逻辑）
  let scopeSuffix: string[] = [];
  switch (scope) {
    case "run": {
      const parentRunId = options?.parentRunId ?? taskContext?.ctx?.run.id;
      if (!parentRunId) throw new Error("parentRunId required for 'run' scope");
      scopeSuffix = [parentRunId];
      break;
    }
    case "attempt": {
      const parentRunId = options?.parentRunId ?? taskContext?.ctx?.run.id;
      const attemptNumber = options?.attemptNumber ?? taskContext?.ctx?.attempt.number;
      if (!parentRunId || attemptNumber === undefined) throw new Error("...");
      scopeSuffix = [parentRunId, attemptNumber.toString()];
      break;
    }
  }

  const hash = await generateIdempotencyKey(keyArray.concat(scopeSuffix));
  return client.resetIdempotencyKey(taskIdentifier, hash, requestOptions);
}
```

### 11.2 服务端 Reset API

**文件位置：** `apps/webapp/app/v3/services/resetIdempotencyKey.server.ts`

```typescript
// 将 run 的 idempotencyKey 和 idempotencyKeyExpiresAt 设为 null
await this._prisma.taskRun.update({
  where: { id: run.id },
  data: {
    idempotencyKey: null,
    idempotencyKeyExpiresAt: null,
  },
});
```

---

## 十二、多维度排查决策树

### 12.1 问题定位决策树

```
发现重复运行
    │
    ├─► 维度1：引擎版本检查
    │    │
    │    ├─► 确认 engineVersion 是 V1 还是 V2？
    │    │    │
    │    │    ├─► V1 → 缺少 shouldIdempotencyKeyBeCleared 检查
    │    │    │    ├─► 旧 run 是否 FAILED/EXPIRED？
    │    │    │    │    ├─► 是 → 【原因】V1 Bug：失败 run 仍返回 isCached=true
    │    │    │    │    └─► 否 → 继续排查
    │    │    │    └─► 旧 run 过期时是否只清了 key 没清 ExpiresAt？
    │    │    │
    │    │    └─► V2 → 继续排查
    │    │
    │    └─► 检查数据库中 run.engine 字段确认版本
    │
    ├─► 维度2：Scope 检查
    │    │
    │    ├─► 查看 idempotencyKeyOptions.scope（V2）或推断 scope
    │    │    │
    │    │    ├─► scope="run" 且来自不同父 run？
    │    │    │    └─► 【原因】不同 parentRunId → 不同 hash
    │    │    │
    │    │    ├─► scope="run" 且在 task 外部调用？
    │    │    │    └─► 【注意】taskContext 为空，scope 行为等同 global
    │    │    │
    │    │    ├─► scope="attempt" 且父任务重试？
    │    │    │    └─► 【原因】attemptNumber 变了 → 不同 hash
    │    │    │
    │    │    └─► scope="global" → 继续排查
    │    │
    │    └─► 比较两个 run 的 idempotencyKey 哈希是否一致
    │
    ├─► 维度3：Run 状态检查
    │    │
    │    ├─► 第一次 run 的 status 是什么？
    │    │    │
    │    │    ├─► FAILED 类（INTERRUPTED/CRASHED/TIMED_OUT 等）
    │    │    │    └─► V2: shouldIdempotencyKeyBeCleared → 清除 → 允许新 run
    │    │    │    └─► V1: 不检查 → 可能阻止新 run
    │    │    │
    │    │    ├─► EXPIRED
    │    │    │    ├─► 检查 run.ttl 是否合理
    │    │    │    ├─► DEV 环境默认 10m，worker 未运行会触发
    │    │    │    └─► V2: 清除幂等键；V1: 不清除
    │    │    │
    │    │    ├─► COMPLETED_SUCCESSFULLY / CANCELED
    │    │    │    └─► 【排除】这些状态保留幂等键，不应导致重复
    │    │    │
    │    │    └─► PENDING / EXECUTING（进行中又创建新 run？）
    │    │         └─► 【重点排查并发链路】
    │    │
    │    └─► 检查 idempotencyKeyExpiresAt 是否已过期？
    │         ├─► 是 → 【原因】幂等键 TTL 过期
    │         └─► 否 → 继续排查
    │
    ├─► 维度4：环境与任务隔离
    │    │
    │    ├─► runtimeEnvironmentId 是否相同？
    │    │    └─► 否 → 【原因】环境隔离
    │    │
    │    └─► taskIdentifier 是否相同？
    │         └─► 否 → 【原因】任务隔离
    │
    ├─► 维度5：Run TTL 与 EXPIRED 的排查
    │    │
    │    ├─► Run 的 ttl 字段值是什么？
    │    │    ├─► undefined → PROD 默认，永不过期
    │    │    ├─► "10m" → DEV 默认
    │    │    └─► 自定义值 → 检查是否合理
    │    │
    │    ├─► Run 变为 EXPIRED 的时序
    │    │    ├─► 是否在 PENDING 状态超时？
    │    │    ├─► Worker 是否在运行？
    │    │    └─► lockedAt 是否为 null？
    │    │
    │    └─► EXPIRED 发生后幂等键是否被清除？
    │         ├─► V2: shouldIdempotencyKeyBeCleared("EXPIRED") = true
    │         └─► V1: 不清除（Bug）
    │
    └─► 维度6：并发与日志检查
         │
         ├─► 日志中是否有 RunDuplicateIdempotencyKeyError？
         │    ├─► 有 → 并发保护生效，重试后命中缓存（正常）
         │    └─► 无但出现了两个 run → 可能在 V1 路径
         │
         ├─► 数据库唯一约束是否生效？
         │    └─► 确认 @@unique 约束存在
         │
         ├─► 检查重试逻辑是否正确处理 P2002
         │
         └─► 维度7：batchTrigger 场景检查
              │
              ├─► 是否通过 batchTrigger 触发？
              │    │
              │    ├─► 是 batchTrigger
              │    │    │
              │    │    ├─► 同一 batch 内是否有多个不同 task？
              │    │    │    │
              │    │    │    ├─► 是，且不同 task 使用相同 idempotencyKey
              │    │    │    │    └─► 【高风险】batchTriggerV3 误配：后匹配的 task 被静默跳过
              │    │    │    │
              │    │    │    └─► 否 → 继续
              │    │    │
              │    │    ├─► 第一次 run 的 status 是 FAILED/EXPIRED？
              │    │    │    └─► 【原因】batchTriggerV3 缺 shouldIdempotencyKeyBeCleared 检查，旧 run 仍返回 isCached=true
              │    │    │
              │    │    ├─► 幂等键 TTL 过期时，是否只清了 key 没清 ExpiresAt？
              │    │    │    └─► 【已知缺陷】batchTriggerV3 与 V1 行为相同
              │    │    │
              │    │    └─► 是否是 batchTriggerAndWait（有 dependentAttempt）？
              │    │         └─► 【排除】该场景跳过缓存检查，不会有误配
              │    │
              │    └─► 否 → 继续排查
```

### 12.2 常见重复原因速查表

| 现象 | 引擎版本 | 可能原因 | 验证方法 | 解决方案 |
|------|---------|---------|---------|---------|
| 失败的旧 run 仍阻止新 trigger | V1 | V1 不检查 `shouldIdempotencyKeyBeCleared` | 检查旧 run.status 是否 FAILED | 升级到 V2 |
| EXPIRED run 阻止新 trigger | V1 | 同上 | 检查旧 run.status 是否 EXPIRED | 升级到 V2 |
| 不同父 run 触发都创建了新 run | V1/V2 | 使用了默认 `run` scope | 检查 idempotencyKeyOptions.scope | 需要全局去重用 `scope: "global"` |
| DEV 环境 10 分钟后出现新 run | V1/V2 | Run TTL 过期 → EXPIRED → 清除幂等键 | 检查 run.ttl 和 run.status | 调整 run.ttl 或确保 worker 运行 |
| 长时间后相同 key 创建了新 run | V1/V2 | 幂等键 TTL 过期 | 检查 idempotencyKeyExpiresAt | 调整 idempotencyKeyTTL |
| dev 和 prod 环境都创建了 run | V1/V2 | 环境隔离 | 检查 runtimeEnvironmentId | 设计预期 |
| 不同 task 用相同 key 都运行了 | V1/V2 | 任务隔离 | 检查 taskIdentifier | 设计预期 |
| 无效 TTL 格式导致 30 天后重复 | V1/V2 | `resolveIdempotencyKeyTTL` 静默返回 undefined | 检查 TTL 字符串格式 | 使用标准格式 `5m`/`1h`/`7d` |
| batch 中某 task 没运行但没报错 | V1/V2 | **batchTriggerV3 误配**：不同 task 用相同 key | 检查 batch 内是否有多个 task 用相同 key | 确保 batch 内不同 task 使用不同 key，或升级修复 |
| batch 中 FAILED run 仍阻止重试 | V1/V2 | **batchTriggerV3 缺陷**：缺 `shouldIdempotencyKeyBeCleared` 检查 | 检查旧 run.status 是否 FAILED + 触发方式是否 batch | 升级修复，或避免 batch 场景重试 |
| 跨进程 reset 失败报错 | V1/V2 | **catalog 失效**：scope fallback 为 run 需 parentRunId | 检查 reset 调用是否跨进程，参数是否完整 | 传 64 字符哈希，或显式传 scope 和 parentRunId |
| `createIdempotencyKey(已哈希key)` 去重失败 | V1/V2 | **双重哈希**：create 没 64 字符检测，reset 有 | 检查是否传入已有的 64 字符哈希给 create | 直接传哈希给 trigger，不要传给 createIdempotencyKey |

---

## 十三、幂等保护边界总结

**幂等保护不覆盖的场景：**
1. ✗ **不同 task** - `taskIdentifier` 不同，即使 key 相同
2. ✗ **不同环境** - `runtimeEnvironmentId` 不同
3. ✗ **父 run 不同且 scope=run** - hash 中包含 `parentRunId`
4. ✗ **父 run 重试且 scope=attempt** - hash 中包含 `attemptNumber`
5. ✗ **第一次 run FAILED 后（V2）** - `shouldIdempotencyKeyBeCleared` 清除幂等键
6. ✗ **第一次 run EXPIRED 后（V2）** - 单独判断清除幂等键
7. ✗ **幂等键 TTL 过期后** - `idempotencyKeyExpiresAt` 超期
8. ✗ **V1 引擎** - 不检查 `shouldIdempotencyKeyBeCleared`
9. ✗ **batchTriggerV3 跨 task 同 key** - 缓存匹配时只看 key，不校验 task，后匹配的 task 被静默跳过
10. ✗ **batchTriggerV3 FAILED run** - 缺少 `shouldIdempotencyKeyBeCleared` 检查，失败 run 阻止重试
11. ✗ **跨进程 catalog 失效 + reset 传原始 key** - scope fallback 为 run，缺少 parentRunId 抛出异常
12. ✗ **createIdempotencyKey 传入已哈希的 64 字符 key** - 被双重哈希，去重失效

---

## 十四、代码关键点索引

| 模块 | 文件位置 | 核心逻辑 | 行号 |
|------|----------|----------|------|
| SDK 转发层 | `packages/trigger-sdk/src/v3/idempotencyKeys.ts` | 纯 re-export | 1-7 |
| Core 创建 | `packages/core/src/v3/idempotencyKeys.ts` | scope 注入 + SHA256 哈希 | 127-142 |
| Core hash | `packages/core/src/v3/idempotencyKeys.ts` | `generateIdempotencyKey` | 163-165 |
| Core reset 64 字符检测 | `packages/core/src/v3/idempotencyKeys.ts` | 64 字符直接使用，不哈希 | 223-226 |
| Core reset fallback | `packages/core/src/v3/idempotencyKeys.ts` | scope fallback + 重建 hash | 237-268 |
| Crypto | `packages/core/src/v3/utils/crypto.ts` | `digestSHA256` | 7-15 |
| Catalog 接口 | `packages/core/src/v3/idempotency-key-catalog/catalog.ts` | `registerKeyOptions/getKeyOptions` | 1-11 |
| 服务端解析 | `packages/core/src/v3/serverOnly/idempotencyKeys.ts` | 从 DB 提取原始 key 和 scope | 10-68 |
| API Schema | `packages/core/src/v3/schemas/api.ts` | `IdempotencyKeyOptionsSchema` | 153-156, 209, 267, 381 |
| V1/V2 分流 | `apps/webapp/app/v3/services/triggerTask.server.ts` | `determineEngineVersion` + switch | 53-79 |
| 引擎版本决策 | `apps/webapp/app/v3/engineVersion.server.ts` | 6 层决策链 | 17-76 |
| V1 幂等逻辑 | `apps/webapp/app/v3/services/triggerTaskV1.server.ts` | 内联，缺状态检查 | 70-113 |
| V2 幂等逻辑 | `apps/webapp/app/runEngine/concerns/idempotencyKeys.server.ts` | `IdempotencyKeyConcern` | 20-137 |
| V2 TTL 解析 | `apps/webapp/app/runEngine/services/triggerTask.server.ts` | 3 级 Run TTL 优先级 | 282-291 |
| 状态分类 | `apps/webapp/app/v3/taskStatus.ts` | `shouldIdempotencyKeyBeCleared` | 133-134 |
| V1 EXPIRED | `apps/webapp/app/v3/services/expireEnqueuedRun.server.ts` | `FinalizeTaskRunService` | 25-99 |
| V2 EXPIRED | `internal-packages/run-engine/src/engine/systems/ttlSystem.ts` | PENDING+unlocked 检查 | 25-134 |
| V2 写入保护 | `internal-packages/run-engine/src/engine/index.ts` | P2002 → `RunDuplicateIdempotencyKeyError` | 700-733 |
| V2 重试 | `apps/webapp/app/runEngine/services/triggerTask.server.ts` | 捕获后重试 | 574-584 |
| V1 重试 | `apps/webapp/app/v3/services/triggerTaskV1.server.ts` | 手动解析 meta.target | 606-659 |
| batchTriggerV3 幂等 | `apps/webapp/app/v3/services/batchTriggerV3.server.ts` | `#prepareRunData` | 323-418 |
| batchTriggerV3 误配 | `apps/webapp/app/v3/services/batchTriggerV3.server.ts` | find 只按 key，缺 task 校验 | 378-379, 363-368 |
| batchTriggerV3 缺陷 | `apps/webapp/app/v3/services/batchTriggerV3.server.ts` | 缺 `shouldIdempotencyKeyBeCleared` | 381-391 |
| batchTriggerV3 过期 | `apps/webapp/app/v3/services/batchTriggerV3.server.ts` | 只清 key，没清 ExpiresAt | 410-414 |
| DB 唯一约束 | `internal-packages/database/prisma/schema.prisma` | `@@unique([envId, taskId, key])` | 1076 |
| TTL 解析函数 | `apps/webapp/app/utils/idempotencyKeys.server.ts` | `resolveIdempotencyKeyTTL` | 10-41 |
