# Trigger.dev 幂等键（Idempotency Key）机制深度分析

## 一、核心问题背景

用户反映**同一外部事件导致两次 task 运行**，需要深入理解 trigger.dev 的幂等保护覆盖范围，从客户端携带幂等键到服务端处理、再到 dashboard 体现的完整代码链路。

---

## 二、幂等键的作用范围与层级

### 2.1 数据库层面的唯一性约束

**文件位置：** `internal-packages/database/prisma/schema.prisma:1076`

```prisma
@@unique([runtimeEnvironmentId, taskIdentifier, idempotencyKey])
```

**作用范围层级：**
1. **环境隔离** - `runtimeEnvironmentId`：dev/staging/prod 环境完全隔离
2. **任务隔离** - `taskIdentifier`：不同 task 之间的幂等键互不影响
3. **键值匹配** - `idempotencyKey`：相同环境 + 相同 task + 相同键值才会去重

### 2.2 SDK 层面的 scope 机制

**文件位置：** `packages/core/src/v3/idempotencyKeys.ts:127-142`

| Scope | Hash 组成 | 适用场景 |
|-------|----------|----------|
| `run`（默认） | `key + parentRunId` | 防止父任务重试时重复触发子任务 |
| `attempt` | `key + parentRunId + attemptNumber` | 允许每次重试都重新触发子任务 |
| `global` | `key` | 全局去重，跨所有父任务 |

**代码实现：**
```typescript
function injectScope(scope: IdempotencyKeyScope): string[] {
  switch (scope) {
    case "run": {
      if (taskContext?.ctx) {
        return [taskContext.ctx.run.id];
      }
      break;
    }
    case "attempt": {
      if (taskContext?.ctx) {
        return [taskContext.ctx.run.id, taskContext.ctx.attempt.number.toString()];
      }
      break;
    }
  }
  return [];
}
```

---

## 三、完整代码链路分析

### 3.1 客户端：幂等键创建与携带

**入口文件：** `packages/trigger-sdk/src/v3/idempotencyKeys.ts`

**创建流程：**
1. 调用 `idempotencyKeys.create(key, { scope })`
2. 根据 scope 注入上下文信息（runId / attemptNumber）
3. SHA256 哈希生成 64 字符幂等键
4. 原始 key 和 scope 存入本地 catalog 供后续 reset 使用

**触发时携带：**
```typescript
// 方式1：trigger 单任务
await childTask.trigger(payload, { 
  idempotencyKey: key,
  idempotencyKeyTTL: "300s"
});

// 方式2：triggerAndWait
await childTask.triggerAndWait(payload, { idempotencyKey: key });

// 方式3：batchTrigger
await tasks.batchTrigger("task-id", [
  { payload, options: { idempotencyKey: key } }
]);
```

### 3.2 API 层：TriggerTaskService

**文件位置：** `apps/webapp/app/v3/services/triggerTask.server.ts:53-122`

```
客户端请求
    ↓
TriggerTaskService.call()
    ↓
determineEngineVersion() → V2
    ↓
RunEngineTriggerTaskService.call()
```

### 3.3 核心处理：IdempotencyKeyConcern

**文件位置：** `apps/webapp/app/runEngine/concerns/idempotencyKeys.server.ts:13-137`

**这是幂等保护的核心逻辑：**

```typescript
async handleTriggerRequest(request, parentStore): Promise<IdempotencyKeyConcernResult> {
  // 1. 提取幂等键和过期时间
  const idempotencyKey = request.options?.idempotencyKey ?? request.body.options?.idempotencyKey;
  const idempotencyKeyExpiresAt = request.options?.idempotencyKeyExpiresAt 
    ?? resolveIdempotencyKeyTTL(request.body.options?.idempotencyKeyTTL) 
    ?? new Date(Date.now() + 24 * 60 * 60 * 1000 * 30); // 默认30天

  // 2. 查询已存在的 run
  const existingRun = await this.prisma.taskRun.findFirst({
    where: {
      runtimeEnvironmentId: request.environment.id,
      idempotencyKey,
      taskIdentifier: request.taskId,
    },
    include: { associatedWaitpoint: true },
  });

  if (existingRun) {
    // 3. 检查是否过期
    if (existingRun.idempotencyKeyExpiresAt && existingRun.idempotencyKeyExpiresAt < new Date()) {
      // 过期：清除键，允许新 run
      await this.prisma.taskRun.updateMany({
        where: { id: existingRun.id, idempotencyKey },
        data: { idempotencyKey: null, idempotencyKeyExpiresAt: null },
      });
      return { isCached: false, idempotencyKey, idempotencyKeyExpiresAt };
    }

    // 4. 检查是否需要清除（失败或过期状态）
    if (shouldIdempotencyKeyBeCleared(existingRun.status)) {
      // 清除键，允许新 run
      await this.prisma.taskRun.updateMany({...});
      return { isCached: false, idempotencyKey, idempotencyKeyExpiresAt };
    }

    // 5. 命中缓存：返回已有 run
    // triggerAndWait 场景：用 waitpoint 阻塞父 run
    if (resumeParentOnCompletion && parentRunId) {
      const associatedWaitpoint = await this.engine.getOrCreateRunWaitpoint({...});
      await this.engine.blockRunWithWaitpoint({...});
    }

    return { isCached: true, run: existingRun };
  }

  return { isCached: false, idempotencyKey, idempotencyKeyExpiresAt };
}
```

### 3.4 状态判断逻辑

**文件位置：** `apps/webapp/app/v3/taskStatus.ts:133-134`

```typescript
export function shouldIdempotencyKeyBeCleared(status: TaskRunStatus): boolean {
  return isFailedRunStatus(status) || status === "EXPIRED";
}
```

**FAILED_RUN_STATUSES 包括：**
- `INTERRUPTED`
- `COMPLETED_WITH_ERRORS`
- `SYSTEM_FAILURE`
- `CRASHED`
- `TIMED_OUT`
- `EXPIRED`

**保留幂等键的状态：**
- 成功：`COMPLETED_SUCCESSFULLY`, `CANCELED`
- 进行中：`PENDING`, `EXECUTING`, `WAITING_TO_RESUME`, `RETRYING_AFTER_FAILURE` 等

### 3.5 RunEngine：数据库写入与并发保护

**文件位置：** `internal-packages/run-engine/src/engine/index.ts:447-850`

**并发保护机制：**

1. **数据库唯一约束** - `@@unique([runtimeEnvironmentId, taskIdentifier, idempotencyKey])`
2. **Prisma 异常捕获** - 捕获 `P2002` 唯一约束冲突
3. **自动重试** - 抛出 `RunDuplicateIdempotencyKeyError` 后上层自动重试

```typescript
try {
  taskRun = await prisma.taskRun.create({...});
} catch (error) {
  if (error instanceof Prisma.PrismaClientKnownRequestError) {
    if (error.code === "P2002") {
      // ...检查是否是 oneTimeUseToken 冲突
      
      // 幂等键冲突：抛出特定异常
      throw new RunDuplicateIdempotencyKeyError(
        `Run with idempotency key ${idempotencyKey} already exists`
      );
    }
  }
  throw error;
}
```

**上层重试逻辑：** `apps/webapp/app/runEngine/services/triggerTask.server.ts:574-584`

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

### 3.6 Trace 与日志：Dashboard 体现

**文件位置：** `apps/webapp/app/runEngine/concerns/traceEvents.server.ts:62-126`

**去重命中时的 trace 记录：**

```typescript
async traceIdempotentRun(request, parentStore, options, callback) {
  return await repository.traceEvent(
    `${request.taskId} (cached)`,  // 显示 cached 标记
    {
      attributes: {
        properties: {
          [SemanticInternalAttributes.ORIGINAL_RUN_ID]: existingRun.friendlyId,
        },
        style: {
          icon: "task-cached",  // 使用缓存图标
        },
        runId: existingRun.friendlyId,
      },
      incomplete,
      isError,
      immediate: true,
    },
    async (event, traceContext, traceparent) => {
      // 记录去重日志消息
      await repository.recordEvent(
        `There's an existing run for idempotencyKey: ${idempotencyKey}`,
        {...}
      );
      return await callback(...);
    }
  );
}
```

---

## 四、过期策略（TTL）

### 4.1 TTL 解析

**文件位置：** `apps/webapp/app/utils/idempotencyKeys.server.ts:10-41`

```typescript
export function resolveIdempotencyKeyTTL(ttl: string | undefined | null): Date | undefined {
  const match = ttl.match(/^(\d+)([smhd])$/);
  // s=秒, m=分, h=时, d=天
}
```

### 4.2 默认值与优先级

**文件位置：** `apps/webapp/app/runEngine/concerns/idempotencyKeys.server.ts:25-28`

```
优先级：
1. request.options.idempotencyKeyExpiresAt （直接指定过期时间）
2. resolveIdempotencyKeyTTL(request.body.options.idempotencyKeyTTL) （TTL 字符串）
3. 默认 30 天
```

### 4.3 过期后的行为

1. **查询时检测过期** - `handleTriggerRequest` 中先检查 `idempotencyKeyExpiresAt`
2. **清除旧键** - 更新数据库将过期 run 的 `idempotencyKey` 设为 null
3. **允许新 run** - 返回 `isCached: false`，创建新 run

---

## 五、并发触发场景分析

### 5.1 并发保护的三层机制

```
并发请求
    │
    ├─► 第1层：IdempotencyKeyConcern 查询（读屏障）
    │      两个请求都没查到 existingRun
    │      ↓
    ├─► 第2层：数据库唯一约束（写屏障）
    │      第一个请求写入成功
    │      第二个请求触发 P2002 错误
    │      ↓
    └─► 第3层：异常捕获与重试（容错）
           抛出 RunDuplicateIdempotencyKeyError
           上层自动重试
           重试时命中第1层缓存
```

### 5.2 Racepoint 系统（可选）

**文件位置：** `apps/webapp/app/runEngine/services/triggerTask.server.ts:252-257`

```typescript
if (idempotencyKey) {
  await this.triggerRacepointSystem.waitForRacepoint({
    racepoint: "idempotencyKey",
    id: idempotencyKey,
  });
}
```

**作用：** 在某些测试或特定场景下，可以通过 Racepoint 控制并发时序。

### 5.3 triggerAndWait 场景的并发

当使用 `triggerAndWait` 命中缓存时：

1. 获取或创建已有 run 的 waitpoint
2. 用 waitpoint 阻塞父 run
3. 子 run 完成后自动唤醒父 run

---

## 六、与重试机制的交互

### 6.1 父任务重试 vs 子任务幂等

**场景：** 父任务失败重试，子任务使用幂等键

```
父任务 Attempt 1
    ├─► trigger 子任务（idempotencyKey="send-email-123"）
    │    创建子 run #child-001
    └─► 父任务失败 ← 在这里失败

父任务 Attempt 2（重试）
    └─► trigger 子任务（相同 idempotencyKey）
         │
         ├─► 查询 existingRun → 找到 #child-001
         ├─► 检查状态 → 进行中/成功
         └─► 返回已有 run，不创建新 run

结果：子任务只执行一次 ✓
```

### 6.2 scope 对重试的影响

| Scope | 重试时行为 |
|-------|-----------|
| `run` | 重用同一个子 run |
| `attempt` | 每次重试创建新的子 run |
| `global` | 全局唯一，跨所有父 run |

### 6.3 子任务失败后的重试

**关键逻辑：** `shouldIdempotencyKeyBeCleared()`

```
子任务失败（COMPLETED_WITH_ERRORS / CRASHED 等）
    ↓
父任务重试时再次 trigger
    ↓
查询 existingRun → 找到失败的 run
    ↓
shouldIdempotencyKeyBeCleared(FAILED) → true
    ↓
清除旧 run 的幂等键
    ↓
创建新的子 run ✓
```

**设计意图：** 失败的任务应该允许重试，幂等保护不应该阻碍错误恢复。

---

## 七、批量触发（Batch）中的幂等

### 7.1 Batch 级别幂等键

**文件位置：** `internal-packages/database/prisma/schema.prisma:1921`

```prisma
@@unique([runtimeEnvironmentId, idempotencyKey])
```

### 7.2 Batch 内单个 item 的幂等键

**文件位置：** `apps/webapp/app/runEngine/services/batchTrigger.server.ts`

每个 batch item 单独处理幂等键，处理逻辑与单个 trigger 一致。

---

## 八、幂等键的 Reset 机制

### 8.1 SDK 层面的 reset

**文件位置：** `packages/core/src/v3/idempotencyKeys.ts:215-269`

```typescript
export async function resetIdempotencyKey(
  taskIdentifier: string,
  idempotencyKey: IdempotencyKey | string | string[],
  options?: ResetIdempotencyKeyOptions
): Promise<{ id: string }> {
  // 根据 scope 重新计算 hash
  // 调用 API 清除数据库中的幂等键
}
```

### 8.2 Reset 的使用场景

1. **成功后需要重新执行** - 成功的 run 默认保留幂等键
2. **手动测试** - Dashboard 中手动重置
3. **业务逻辑需要** - 特定条件下允许重复执行

---

## 九、常见问题与可能的重复原因

### 9.1 为什么会出现两次运行？

**可能原因 1：scope 不匹配**
- 预期是全局去重，但使用了默认 `run` scope
- 不同父 run 使用相同 key 但 scope 是 `run` → 会创建不同 run

**可能原因 2：第一次 run 失败**
- 第一次 run 失败后幂等键被清除
- 第二次 trigger 创建新 run

**可能原因 3：TTL 过期**
- 第一次 run 的 TTL 已过期
- 第二次 trigger 创建新 run

**可能原因 4：并发极端情况**
- 极高并发下，请求绕过了第一层查询检查
- 但数据库唯一约束应该能阻止最终写入（P2002 + 重试）
- **注意：** 如果重试逻辑有问题，可能导致意外行为

**可能原因 5：不同 task 或不同环境**
- 相同 key 但 taskIdentifier 不同
- 相同 key 但 runtimeEnvironmentId 不同

### 9.2 幂等保护的边界

**幂等保护不覆盖的场景：**
1. ✗ **不同 task** - taskIdentifier 不同，即使 key 相同
2. ✗ **不同环境** - runtimeEnvironmentId 不同
3. ✗ **父 run 不同且 scope=run** - hash 中包含 parentRunId
4. ✗ **第一次 run 失败后** - 幂等键被清除
5. ✗ **TTL 过期后**

---

## 十、代码关键点总结

| 模块 | 文件位置 | 核心逻辑 |
|------|----------|----------|
| SDK 创建 | `packages/core/src/v3/idempotencyKeys.ts` | scope + SHA256 哈希 |
| 核心检查 | `apps/webapp/app/runEngine/concerns/idempotencyKeys.server.ts` | 查询 existingRun + 状态判断 |
| 状态判断 | `apps/webapp/app/v3/taskStatus.ts` | 失败/过期状态清除键 |
| 并发写保护 | `internal-packages/run-engine/src/engine/index.ts` | Prisma P2002 捕获 |
| 自动重试 | `apps/webapp/app/runEngine/services/triggerTask.server.ts` | RunDuplicateIdempotencyKeyError |
| TTL 解析 | `apps/webapp/app/utils/idempotencyKeys.server.ts` | smhd 单位解析 |
| Dashboard 显示 | `apps/webapp/app/runEngine/concerns/traceEvents.server.ts` | cached 标记 + 日志 |
| DB 约束 | `internal-packages/database/prisma/schema.prisma` | 三元组唯一索引 |

---

## 十一、排查建议

如果遇到重复运行问题，按以下顺序排查：

1. **检查 scope** - 是否使用了正确的 scope（global/run/attempt）
2. **检查第一次 run 状态** - 是否是失败状态导致幂等键被清除
3. **检查 TTL** - 是否已过期
4. **检查 taskIdentifier** - 是否是同一个 task
5. **检查环境** - 是否在同一个 environment
6. **检查并发日志** - 查看是否有 `RunDuplicateIdempotencyKeyError` 及重试日志
7. **检查 idempotencyKey 值** - 确认两次请求的 key 完全相同（哈希后）

