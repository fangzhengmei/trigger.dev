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

## 八、幂等键的 Reset 机制

### 8.1 SDK 层面的 reset

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

### 8.2 服务端 Reset API

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

## 九、多维度排查决策树

### 9.1 问题定位决策树

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
         └─► 检查重试逻辑是否正确处理 P2002
```

### 9.2 常见重复原因速查表

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

---

## 十、幂等保护边界总结

**幂等保护不覆盖的场景：**
1. ✗ **不同 task** - `taskIdentifier` 不同，即使 key 相同
2. ✗ **不同环境** - `runtimeEnvironmentId` 不同
3. ✗ **父 run 不同且 scope=run** - hash 中包含 `parentRunId`
4. ✗ **父 run 重试且 scope=attempt** - hash 中包含 `attemptNumber`
5. ✗ **第一次 run FAILED 后（V2）** - `shouldIdempotencyKeyBeCleared` 清除幂等键
6. ✗ **第一次 run EXPIRED 后（V2）** - 单独判断清除幂等键
7. ✗ **幂等键 TTL 过期后** - `idempotencyKeyExpiresAt` 超期
8. ✗ **V1 引擎** - 不检查 `shouldIdempotencyKeyBeCleared`

---

## 十一、代码关键点索引

| 模块 | 文件位置 | 核心逻辑 | 行号 |
|------|----------|----------|------|
| SDK 转发层 | `packages/trigger-sdk/src/v3/idempotencyKeys.ts` | 纯 re-export | 1-7 |
| Core 创建 | `packages/core/src/v3/idempotencyKeys.ts` | scope 注入 + SHA256 哈希 | 127-142 |
| Core hash | `packages/core/src/v3/idempotencyKeys.ts` | `generateIdempotencyKey` | 163-165 |
| Crypto | `packages/core/src/v3/utils/crypto.ts` | `digestSHA256` | 7-15 |
| Catalog 接口 | `packages/core/src/v3/idempotency-key-catalog/catalog.ts` | `registerKeyOptions/getKeyOptions` | 1-11 |
| 服务端解析 | `packages/core/src/v3/serverOnly/idempotencyKeys.ts` | 从 DB 提取原始 key 和 scope | 10-68 |
| API Schema | `packages/core/src/v3/schemas/api.ts` | `IdempotencyKeyOptionsSchema` | 153-156 |
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
| DB 唯一约束 | `internal-packages/database/prisma/schema.prisma` | `@@unique([envId, taskId, key])` | 1076 |
| TTL 解析函数 | `apps/webapp/app/utils/idempotencyKeys.server.ts` | `resolveIdempotencyKeyTTL` | 10-41 |
