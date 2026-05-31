# Realtime Stream 入口边界分析

## 一、总览：10 条路由、3 种流、2 种后端

以下分析严格按代码中每个路由文件实际导出的 `action`（写）/ `loader`（读）逐条核对，不混用读写口径。

```
realtime.v1 路由树
├── streams/
│   ├── $runId/
│   │   ├── $streamId.ts              ← Run 流（兼容路由）
│   │   ├── $target/$streamId.ts      ← Run 流（认证路由）
│   │   ├── $target/$streamId.append.ts ← Run 流 append
│   │   └── input/$streamId.ts        ← Input 流
├── sessions/
│   ├── $session/$io.ts               ← Session 流
│   ├── $session/$io.append.ts        ← Session 流 append
│   └── $session/$io.records.ts       ← Session 流 records 查询
├── runs.ts                           ← Run 列表实时流（Electric）
├── runs/$runId.ts                    ← 单 Run 实时流（Electric）
└── batches/$batchId.ts              ← Batch 实时流（Electric）
```

**runs / runs.$runId / batches.$batchId 三条路由走 Electric 长轮询，不涉及 stream 写入**，下文不再展开。

---

## 二、Run 流——逐路由读写核对

### 2.1 路由 A：`$runId/$streamId.ts`（兼容路由，无 $target）

**文件**：`apps/webapp/app/routes/realtime.v1.streams.$runId.$streamId.ts`

| 导出 | HTTP 方法 | 职责 | 调用的服务层方法 | 读/写 |
|------|-----------|------|-----------------|-------|
| `action` | POST | 批量流式写入 | `ingestData()` | **写** |
| `loader` | GET | SSE 实时读取 | `streamResponse()` | **读** |

#### 写入证据（:18-82）

```typescript
export async function action({ request, params }: ActionFunctionArgs) {
  // 无认证——向后兼容旧客户端
  const clientId = request.headers.get("X-Client-Id") || "default";
  const streamVersion = request.headers.get("X-Stream-Version") || "v1";
  const resumeFromChunkNumber = ...;

  return realtimeStream.ingestData(
    request.body,       // ← 写入数据来源
    run.friendlyId,
    streamId,
    clientId,
    resumeFromChunkNumber
  );
}
```

**关键特征**：
- 无认证（向后兼容旧 SDK）
- 硬编码 target = self（无 parent/root 选择）
- 不支持 PUT（无初始化/凭证发放）
- 不支持 HEAD（无断点续传查询）
- 调用 `ingestData()` → V1 写 Redis，V2 抛错

#### 读取证据（:84-156）

```typescript
export const loader = createLoaderApiRoute({
  allowJWT: true,
  corsStrategy: "all",
  authorization: { action: "read", ... },
}, async ({ params, request, resource: run, authentication }) => {
  const lastEventId = request.headers.get("Last-Event-ID") || undefined;
  const timeoutInSeconds = ...;

  return realtimeStream.streamResponse(
    request, run.friendlyId, params.streamId,
    getRequestAbortSignal(),
    { lastEventId, timeoutInSeconds }
  );
});
```

**关键特征**：
- JWT + CORS 认证
- 支持 `Last-Event-ID` 断点续传
- 支持 `Timeout-Seconds`（1–600）控制长轮询时长
- 返回 SSE 流

---

### 2.2 路由 B：`$runId/$target/$streamId.ts`（认证路由，含 $target）

**文件**：`apps/webapp/app/routes/realtime.v1.streams.$runId.$target.$streamId.ts`

| 导出 | HTTP 方法 | 职责 | 调用的服务层方法 | 读/写 |
|------|-----------|------|-----------------|-------|
| `action` | PUT | 初始化流 + 凭证发放 | `initializeStream()` | **写**（元数据注册 + S2 凭证） |
| `action` | POST | 批量流式写入 | `ingestData()` | **写** |
| `loader` | HEAD | 查询服务端进度 | `getLastChunkIndex()` | **读**（仅查询，不写） |

#### 写入证据 1——PUT（:63-110）

```typescript
if (request.method === "PUT") {
  // 1. 注册 streamId → realtimeStreams 数组
  if (!target.realtimeStreams.includes(params.streamId)) {
    await prisma.taskRun.update({
      where: { id: target.id },
      data: { realtimeStreams: { push: params.streamId } },
    });
  }

  // 2. 初始化流（V1 空操作，V2 发放 S2 凭证）
  const { responseHeaders } = await realtimeStream.initializeStream(targetId, params.streamId);

  return json({ version: target.realtimeStreamsVersion }, { status: 202, headers: responseHeaders });
}
```

**PUT 做了两件事**：
1. **DB 写入**：将 streamId push 到 `taskRun.realtimeStreams` 数组
2. **S2 凭证发放**（V2 场景）：调用 `initializeStream()` 生成临时 access token

两者都是**写操作**——修改了 DB 状态和（间接）S2 流状态。

#### 写入证据 2——POST（:111-145）

```typescript
else {
  const clientId = request.headers.get("X-Client-Id") || "default";
  const streamVersion = request.headers.get("X-Stream-Version") || "v1";
  const resumeFromChunkNumber = ...;

  return realtimeStream.ingestData(
    request.body, targetId, params.streamId, clientId, resumeFromChunkNumber
  );
}
```

与兼容路由的 POST 相同，但**需要 API Key 认证**（`createActionApiRoute` 默认策略）。

#### 读取证据——HEAD（:149-226）

```typescript
const loader = createLoaderApiRoute({
  allowJWT: false,
  corsStrategy: "none",
}, async ({ request, params, resource: run, authentication }) => {
  if (request.method !== "HEAD") {
    return new Response("Only HEAD requests are allowed for this endpoint", { status: 405 });
  }

  const clientId = request.headers.get("X-Client-Id") || "default";
  const lastChunkIndex = await realtimeStream.getLastChunkIndex(targetId, params.streamId, clientId);

  return new Response(null, {
    status: 200,
    headers: { "X-Last-Chunk-Index": lastChunkIndex.toString() },
  });
});
```

**HEAD 是纯读操作**：
- 只查询 Redis 中该 clientId 的最后 chunkIndex
- 不修改任何状态
- V1 实现：`xrevrange` + clientId 过滤
- V2 实现：抛错（S2 不需要 HEAD，`appendSession` 内置确认）

**关键特征**：
- loader 显式拒绝非 HEAD 方法（:199-201）
- 不支持 GET/SSE（该路由无流式读取能力）
- 不支持 JWT（`allowJWT: false`）
- CORS 策略为 none

---

### 2.3 路由 C：`$runId/$target/$streamId.append.ts`

**文件**：`apps/webapp/app/routes/realtime.v1.streams.$runId.$target.$streamId.append.ts`

| 导出 | HTTP 方法 | 职责 | 调用的服务层方法 | 读/写 |
|------|-----------|------|-----------------|-------|
| `action` | POST | 单条记录写入 | `appendPart()` | **写** |

**无 loader 导出**——此路由是纯写路由。

```typescript
const { action } = createActionApiRoute(
  { params: ParamsSchema, maxContentLength: MAX_APPEND_BODY_BYTES },
  async ({ request, params, authentication }) => {
    // 1. 注册 streamId（若未注册）
    if (!targetRun.realtimeStreams.includes(params.streamId)) {
      await prisma.taskRun.update({ ... });
    }

    // 2. 写入单条记录
    const part = await request.text();
    const partId = request.headers.get("X-Part-Id") ?? nanoid(7);
    await realtimeStream.appendPart(part, partId, targetId, params.streamId);

    return json({ ok: true }, { status: 200 });
  }
);

export { action };  // ← 只导出 action，无 loader
```

**写入证据**：
1. **DB 写入**：streamId 注册到 `realtimeStreams` 数组（:89-100）
2. **存储层写入**：`appendPart()` → V1 写 Redis XADD，V2 写 S2 AppendRecords
3. **无任何读取逻辑**

---

### 2.4 Run 流读写职责汇总

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        Run 流 — 写入口                                     │
│                                                                            │
│  路由 A (兼容) POST    → ingestData()      → Redis XADD / S2 抛错          │
│  路由 B (认证) PUT     → initializeStream() → DB push + S2 凭证            │
│  路由 B (认证) POST    → ingestData()      → Redis XADD / S2 抛错          │
│  路由 C (append) POST  → appendPart()      → Redis XADD / S2 AppendRecords │
│                                                                            │
├────────────────────────────────────────────────────────────────────────────┤
│                        Run 流 — 读入口                                     │
│                                                                            │
│  路由 A (兼容) GET     → streamResponse()   → SSE 长流                     │
│  路由 B (认证) HEAD    → getLastChunkIndex() → 返回 X-Last-Chunk-Index     │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

**注意**：路由 B 的 loader 只处理 HEAD，不处理 GET/SSE。Run 流的 SSE 读取统一走路由 A 的 loader。

---

## 三、Session 流——逐路由读写核对

### 3.1 路由 D：`sessions/$session/$io.ts`

**文件**：`apps/webapp/app/routes/realtime.v1.sessions.$session.$io.ts`

| 导出 | HTTP 方法 | 职责 | 调用的服务层方法 | 读/写 |
|------|-----------|------|-----------------|-------|
| `action` | PUT | 初始化 S2 channel + 凭证发放 | `initializeSessionStream()` | **写** |
| `loader` | HEAD | 返回固定 X-Last-Chunk-Index: 0 | 无服务层调用 | **读**（兼容占位） |
| `loader` | GET | SSE 实时读取 | `streamResponseFromSessionStream()` | **读** |

#### 写入证据——PUT（:26-84）

```typescript
const { action } = createActionApiRoute({
  method: "PUT",
  allowJWT: true,
  corsStrategy: "all",
  authorization: { action: "write", resource: ... },
}, async ({ params, authentication }) => {
  const realtimeStream = getRealtimeStreamInstance(authentication.environment, "v2", { ... });

  if (!(realtimeStream instanceof S2RealtimeStreams)) {
    return new Response("Session channels require the S2 realtime backend", { status: 501 });
  }

  const { responseHeaders } = await realtimeStream.initializeSessionStream(addressingKey, params.io);
  return json({ version: "v2" }, { status: 202, headers: responseHeaders });
});
```

**关键特征**：
- `method: "PUT"` 显式限定 action 只处理 PUT
- 硬编码版本 `"v2"`，session 流不支持 V1/Redis
- 调用 `initializeSessionStream()` → 发放 S2 凭证
- 不做 DB realtimeStreams 注册（session 流不关联此数组）
- 非 S2 后端直接返回 501

#### 读取证据 1——HEAD（:146-154）

```typescript
if (request.method === "HEAD") {
  return new Response(null, {
    status: 200,
    headers: { "X-Last-Chunk-Index": "0" },
  });
}
```

**HEAD 是兼容占位**：S2 不支持 `getLastChunkIndex()` 语义，返回硬编码 0。客户端通过 SSE 的 `Last-Event-ID` 实现断点续传，不需要 HEAD。

#### 读取证据 2——GET（:156-191）

```typescript
const lastEventId = request.headers.get("Last-Event-ID") ?? undefined;
const timeoutInSeconds = ...; // 校验 1-600
const peekSettled = request.headers.get("X-Peek-Settled") === "1";

return realtimeStream.streamResponseFromSessionStream(
  request, resource.addressingKey, params.io,
  getRequestAbortSignal(),
  { lastEventId, timeoutInSeconds, peekSettled }
);
```

**关键特征**：
- 纯读操作，不修改任何状态
- 支持 `X-Peek-Settled` 安定检测（仅影响内部 wait 参数，不改写数据）
- authorization action 为 "read"

---

### 3.2 路由 E：`sessions/$session/$io.append.ts`

**文件**：`apps/webapp/app/routes/realtime.v1.sessions.$session.$io.append.ts`

| 导出 | HTTP 方法 | 职责 | 调用的服务层方法 | 读/写 |
|------|-----------|------|-----------------|-------|
| `action` | POST | 单条记录写入 + ensureRun + drainWaitpoints | `appendPartToSessionStream()` | **写** |

**无 loader 导出**——此路由是纯写路由。

```typescript
const { action, loader } = createActionApiRoute({
  method: "POST",
  maxContentLength: MAX_APPEND_BODY_BYTES,
  authorization: { action: "write", resource: ... },
}, async ({ request, params, authentication, resource: session }) => {
  // 1. ensureRunForSession — 确保有活跃 run 消费数据
  const [ensureError] = await tryCatch(ensureRunForSession({ ... }));

  // 2. 写入 S2
  const part = await request.text();
  const partId = request.headers.get("X-Part-Id") ?? nanoid(7);
  await realtimeStream.appendPartToSessionStream(part, partId, addressingKey, params.io);

  // 3. drainSessionStreamWaitpoints — 通知等待中的任务继续执行
  const [drainError, waitpointIds] = await tryCatch(
    drainSessionStreamWaitpoints(addressingKey, params.io)
  );
  if (waitpointIds?.length > 0) {
    await Promise.all(waitpointIds.map(id => engine.completeWaitpoint({ id, ... })));
  }

  return json({ ok: true }, { status: 200 });
});
```

**写入证据（三步写入链）**：

| 步骤 | 操作 | 读/写 | 失败策略 |
|------|------|-------|---------|
| `ensureRunForSession` | 确保活跃 run（可能触发新 run） | **写**（可能创建 run） | Best-effort：失败仍继续 append |
| `appendPartToSessionStream` | 写入 S2 记录 | **写** | 失败返回错误给客户端 |
| `drainSessionStreamWaitpoints` + `completeWaitpoint` | 通知等待的任务 | **写**（推进任务状态） | Best-effort：失败不影响 append 成功 |

**注意**：虽然变量名解构了 `loader`，但此路由的 `createActionApiRoute` 指定了 `method: "POST"`，loader 实际上只处理 method 判断逻辑，不对外暴露 GET 读取能力。

---

### 3.3 路由 F：`sessions/$session/$io.records.ts`（只读）

**文件**：`apps/webapp/app/routes/realtime.v1.sessions.$session.$io.records.ts`

| 导出 | HTTP 方法 | 职责 | 调用的服务层方法 | 读/写 |
|------|-----------|------|-----------------|-------|
| `loader` | GET | 非SSE批量读取记录 | `readSessionStreamRecords()` | **读** |

**无 action 导出**——此路由是纯读路由。

```typescript
export const loader = createLoaderApiRoute({
  searchParams: SearchSchema,
  allowJWT: true,
  corsStrategy: "all",
  authorization: { action: "read", resource: ... },
}, async ({ params, authentication, resource, searchParams }) => {
  const afterSeqNum = searchParams.afterEventId !== undefined
    ? Number(searchParams.afterEventId)
    : undefined;

  const records = await realtimeStream.readSessionStreamRecords(
    resource.addressingKey,
    params.io,
    afterSeqNum
  );

  return json({ records });
});
```

#### 只读证据清单

| 证据 | 位置 | 说明 |
|------|------|------|
| 只导出 `loader`，无 `action` | :39 | Remix 中 loader 只处理 GET/HEAD，不可能写入 |
| authorization action = "read" | :59-60 | RBAC 层面限制为只读 |
| 调用 `readSessionStreamRecords()` | :90-94 | 方法名以 `read` 开头，返回 `StreamRecord[]` |
| 返回 `json({ records })` | :96 | 无副作用，纯数据返回 |
| 无 `request.text()` / `request.json()` | 全文件 | 不读取请求体，无法携带写入数据 |
| 无 `prisma.taskRun.update` | 全文件 | 不修改 DB 状态 |
| 无 `engine.completeWaitpoint` | 全文件 | 不推进任务状态 |
| 无 `s2Append` / `xadd` | 全文件 | 不写入存储层 |
| searchParams 使用 `afterEventId` 查询参数 | :22 | 游标用 URL 参数传递，非写入数据 |

**`readSessionStreamRecords` 实现确认**：
```typescript
// s2realtimeStreams.server.ts:197-202
async readSessionStreamRecords(friendlyId, io, afterSeqNum?) {
  return this.#readRecordsByName(this.toSessionStreamName(friendlyId, io), afterSeqNum);
}
```
底层调用 S2 的 GET records API（`wait=0`，`count` 有限），纯读操作。

---

### 3.4 Session 流读写职责汇总

```
┌───────────────────────────────────────────────────────────────────────────┐
│                     Session 流 — 写入口                                   │
│                                                                           │
│  路由 D PUT       → initializeSessionStream() → S2 凭证发放               │
│  路由 E POST      → appendPartToSessionStream() → S2 AppendRecords       │
│                    + ensureRunForSession() → 可能创建新 run               │
│                    + drainSessionStreamWaitpoints() → 推进任务            │
│                                                                           │
├───────────────────────────────────────────────────────────────────────────┤
│                     Session 流 — 读入口                                   │
│                                                                           │
│  路由 D GET/SSE   → streamResponseFromSessionStream() → SSE 长流          │
│  路由 D HEAD      → 返回 X-Last-Chunk-Index: 0（兼容占位）                │
│  路由 F GET/JSON  → readSessionStreamRecords() → JSON 批量返回            │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 四、Input 流——逐路由读写核对

### 4.1 路由 G：`$runId/input/$streamId.ts`

**文件**：`apps/webapp/app/routes/realtime.v1.streams.$runId.input.$streamId.ts`

| 导出 | HTTP 方法 | 职责 | 调用的服务层方法 | 读/写 |
|------|-----------|------|-----------------|-------|
| `action` | POST | 发送数据到 input 流 | `appendPart()` | **写** |
| `loader` | GET | SSE 实时读取 input 流 | `streamResponse()` | **读** |

#### 写入证据（:27-104）

```typescript
const { action } = createActionApiRoute({
  maxContentLength: 1024 * 1024, // 1MB
  allowJWT: true,
  corsStrategy: "all",
  authorization: { action: "write", resource: ... },
}, async ({ request, params, authentication }) => {
  const body = BodySchema.safeParse(await request.json());

  // 1. 写入存储
  const recordId = `inp_${crypto.randomUUID().replace(/-/g, "").slice(0, 12)}`;
  const record = JSON.stringify(body.data.data);
  await realtimeStream.appendPart(record, recordId, run.friendlyId, `$trigger.input:${streamId}`);

  // 2. 触发 waitpoint（如有）
  const waitpointId = await getInputStreamWaitpoint(params.runId, params.streamId);
  if (waitpointId) {
    await engine.completeWaitpoint({ id: waitpointId, output: { value: ..., isError: false } });
    await deleteInputStreamWaitpoint(params.runId, params.streamId);
  }

  return json({ ok: true });
});
```

**写入链**：
1. `appendPart()` → V1 Redis XADD / V2 S2 AppendRecords
2. `getInputStreamWaitpoint()` → 读 Redis cache（查找关联 waitpoint）
3. `engine.completeWaitpoint()` → 推进任务状态（写）
4. `deleteInputStreamWaitpoint()` → 删除 Redis cache key（写）

**Input 流的 streamId 特殊命名**：`$trigger.input:${params.streamId}`——加了 `$trigger.input:` 前缀，与用户创建的普通 run 流区分。

#### 读取证据（:107-179）

```typescript
const loader = createLoaderApiRoute({
  allowJWT: true,
  corsStrategy: "all",
  authorization: { action: "read", resource: ... },
}, async ({ params, request, resource: run, authentication }) => {
  const lastEventId = request.headers.get("Last-Event-ID") || undefined;
  const timeoutInSeconds = ...;

  return realtimeStream.streamResponse(
    request, run.friendlyId, `$trigger.input:${params.streamId}`,
    getRequestAbortSignal(),
    { lastEventId, timeoutInSeconds }
  );
});
```

**关键特征**：
- 纯读操作
- 读取内部流名 `$trigger.input:${streamId}`
- 支持 `Last-Event-ID` 和 `Timeout-Seconds`

### 4.2 Input 流读写职责汇总

```
┌───────────────────────────────────────────────────────────────────────────┐
│                     Input 流 — 写入口                                     │
│                                                                           │
│  路由 G POST → appendPart()                → Redis XADD / S2 AppendRecords│
│             + engine.completeWaitpoint()    → 推进任务状态                 │
│             + deleteInputStreamWaitpoint()  → 清理 Redis cache            │
│                                                                           │
├───────────────────────────────────────────────────────────────────────────┤
│                     Input 流 — 读入口                                     │
│                                                                           │
│  路由 G GET/SSE → streamResponse() → SSE 长流                             │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 五、不混用读写口径的完整链路图

### 5.1 写链路（数据从客户端到存储）

```
                           写入链路
                           ═══════

  ┌─────────────┐    ┌──────────────────────────────────────────────────────────────┐
  │ SDK Writer   │    │  Run 流                                                       │
  │              │    │                                                                │
  │ streamsWriter│────▶│ PUT  /streams/:runId/:target/:key                             │
  │ V1/V2       │    │   ├── prisma.taskRun.update (realtimeStreams push)     :88-95  │
  │              │    │   └── initializeStream() → V1:空操作 / V2:S2凭证        :103  │
  │              │    │                                                                │
  │              │────▶│ POST /streams/:runId/:target/:key                             │
  │              │    │   └── ingestData() → V1:Redis XADD / V2:抛错            :138  │
  │              │    │                                                                │
  │              │────▶│ POST /streams/:runId/:target/:key/append                     │
  │ (单条)       │    │   ├── prisma.taskRun.update (realtimeStreams push)     :89-100 │
  │              │    │   └── appendPart() → V1:Redis XADD / V2:S2 AppendRecords:113 │
  └─────────────┘    └──────────────────────────────────────────────────────────────┘

  ┌─────────────┐    ┌──────────────────────────────────────────────────────────────┐
  │ Agent Writer│    │  Session 流 (V2 Only)                                        │
  │             │    │                                                                │
  │ session.out │────▶│ PUT  /sessions/:session/:io                                  │
  │ .writer()   │    │   └── initializeSessionStream() → S2凭证              :78-81  │
  │             │    │                                                                │
  │             │────▶│ POST /sessions/:session/:io/append                           │
  │ (外部输入)   │    │   ├── ensureRunForSession() → 可能创建新run           :111-124 │
  │             │    │   ├── appendPartToSessionStream() → S2 AppendRecords  :131-133 │
  │             │    │   └── drainSessionStreamWaitpoints() + completeWaitpoint     │
  │             │    │       → 推进任务状态                                   :156-188 │
  └─────────────┘    └──────────────────────────────────────────────────────────────┘

  ┌─────────────┐    ┌──────────────────────────────────────────────────────────────┐
  │ 外部调用者   │    │  Input 流                                                     │
  │             │    │                                                                │
  │ trigger.input│───▶│ POST /streams/:runId/input/:key                               │
  │ .send()     │    │   ├── appendPart() → V1:Redis XADD / V2:S2 AppendRecords :81 │
  │             │    │   ├── engine.completeWaitpoint() → 推进任务            :92-99  │
  │             │    │   └── deleteInputStreamWaitpoint() → 清理Redis cache   :100  │
  └─────────────┘    └──────────────────────────────────────────────────────────────┘
```

### 5.2 读链路（数据从存储到客户端）

```
                           读取链路
                           ═══════

  ┌─────────────┐    ┌──────────────────────────────────────────────────────────────┐
  │ 前端 SSE    │    │  Run 流                                                       │
  │ 客户端      │    │                                                                │
  │             │    │ GET /streams/:runId/:key → streamResponse() → SSE      :151  │
  │             │    │   ├── V1: Redis XREAD BLOCK → Transform → SSE               │
  │             │    │   └── V2: S2 GET records (wait) → 代理 SSE                   │
  │             │    │                                                                │
  │             │    │ HEAD /streams/:runId/:target/:key                             │
  │ (断点续传)   │────▶│   └── getLastChunkIndex() → X-Last-Chunk-Index       :213  │
  │             │    │       V1: xrevrange + clientId过滤 / V2: 抛错                │
  └─────────────┘    └──────────────────────────────────────────────────────────────┘

  ┌─────────────┐    ┌──────────────────────────────────────────────────────────────┐
  │ 前端 SSE    │    │  Session 流 (V2 Only)                                        │
  │ 客户端      │    │                                                                │
  │             │────▶│ GET /sessions/:session/:io → streamResponseFromSessionStream()│
  │             │    │   └── S2 GET records (wait) → 代理 SSE                        │
  │             │    │       settled 时 wait=0 → X-Session-Settled                   │
  │             │    │                                                                │
  │ (兼容占位)   │────▶│ HEAD /sessions/:session/:io                                 │
  │             │    │   └── 固定返回 X-Last-Chunk-Index: 0                    :152  │
  └─────────────┘    └──────────────────────────────────────────────────────────────┘

  ┌─────────────┐    ┌──────────────────────────────────────────────────────────────┐
  │ Agent SDK   │    │  Session 流 records 查询 (V2 Only, 只读)                      │
  │ 启动回放    │    │                                                                │
  │             │────▶│ GET /sessions/:session/:io/records → readSessionStreamRecords│
  │             │    │   └── S2 GET records (wait=0) → JSON { records }              │
  │             │    │       ★ 无 action 导出，无任何写入能力                         │
  └─────────────┘    └──────────────────────────────────────────────────────────────┘

  ┌─────────────┐    ┌──────────────────────────────────────────────────────────────┐
  │ Agent 进程内│    │  Input 流                                                     │
  │ SSE tail    │    │                                                                │
  │             │────▶│ GET /streams/:runId/input/:key → streamResponse() → SSE :169 │
  │             │    │   ├── V1: Redis XREAD BLOCK → Transform → SSE               │
  │             │    │   └── V2: S2 GET records (wait) → 代理 SSE                   │
  └─────────────┘    └──────────────────────────────────────────────────────────────┘
```

### 5.3 认证模型对照

```
                    写入口                          读入口
                 ┌──────────────────┐           ┌──────────────────┐
  Run 流         │ B: API Key       │           │ A: JWT + CORS    │
                 │ A: 无认证(兼容)   │           │ B: 不支持读       │
                 │ C: API Key       │           │                  │
                 ├──────────────────┤           ├──────────────────┤
  Session 流     │ D: JWT + CORS    │           │ D: JWT + CORS    │
                 │ E: JWT + CORS    │           │ F: JWT + CORS    │
                 ├──────────────────┤           ├──────────────────┤
  Input 流       │ G: JWT + CORS    │           │ G: JWT + CORS    │
                 └──────────────────┘           └──────────────────┘
```

---

## 六、服务层接口的读写分类

**文件**：`apps/webapp/app/services/realtime/types.ts`

```typescript
export interface StreamIngestor {     // ← 名字就说明是"摄入"（写）
  initializeStream(...): Promise<...>;   // 写：元数据注册 + 凭证
  ingestData(...): Promise<Response>;     // 写：批量流式写入
  appendPart(...): Promise<void>;         // 写：单条记录写入
  getLastChunkIndex(...): Promise<number>; // 读：查询进度
  readRecords(...): Promise<StreamRecord[]>; // 读：批量读取记录
}

export interface StreamResponder {    // ← 名字就说明是"响应"（读）
  streamResponse(...): Promise<Response>;  // 读：SSE 流式响应
}
```

| 方法 | 接口 | 读/写 | 修改存储 | 修改DB | 被调用路由 |
|------|------|-------|---------|--------|-----------|
| `initializeStream` | StreamIngestor | 写 | ✅ S2 (V2) | ✅ realtimeStreams push | B PUT |
| `ingestData` | StreamIngestor | 写 | ✅ Redis XADD | ❌ | A POST, B POST |
| `appendPart` | StreamIngestor | 写 | ✅ Redis/S2 | ❌ | C POST, G POST |
| `getLastChunkIndex` | StreamIngestor | 读 | ❌ | ❌ | B HEAD |
| `readRecords` | StreamIngestor | 读 | ❌ | ❌ | F GET |
| `streamResponse` | StreamResponder | 读 | ❌ | ❌ | A GET, G GET |
| `streamResponseFromSessionStream` | (S2专有) | 读 | ❌ | ❌ | D GET |
| `initializeSessionStream` | (S2专有) | 写 | ✅ S2 | ❌ | D PUT |
| `appendPartToSessionStream` | (S2专有) | 写 | ✅ S2 | ❌ | E POST |

---

## 七、边界发现与结论

### 7.1 同一文件内读写混合的路由

| 路由文件 | 写入口 | 读入口 | 混合程度 |
|---------|--------|--------|---------|
| `$runId/$streamId.ts` | POST (action) | GET/SSE (loader) | **中**：action 和 loader 分离 |
| `$runId/$target/$streamId.ts` | PUT+POST (action) | HEAD (loader) | **中**：PUT/POST 在 action 内分支，HEAD 在 loader |
| `sessions/$session/$io.ts` | PUT (action) | GET+HEAD (loader) | **低**：PUT 显式限定 method，loader 纯读 |
| `$runId/input/$streamId.ts` | POST (action) | GET/SSE (loader) | **中**：同第一行 |
| `$target/$streamId.append.ts` | POST (action) | 无 | **无**：纯写 |
| `sessions/$io.append.ts` | POST (action) | 无 | **无**：纯写 |
| `sessions/$io.records.ts` | 无 | GET (loader) | **无**：纯读 |

**结论**：4 条路由混合读写（action+loader 在同一文件），3 条路由职责单一（纯写或纯读）。混合路由的 action/loader 在 Remix 中天然隔离——action 只处理 POST/PUT/DELETE，loader 只处理 GET/HEAD，不会串扰。

### 7.2 `records` 路由只读的铁证

`sessions/$session/$io.records.ts` 不可能写入的原因：

1. **只导出 `loader`**，无 `action`——Remix 框架层面不可能处理 POST/PUT
2. **authorization action = "read"**——RBAC 层面限制为只读权限
3. **唯一调用的服务层方法是 `readSessionStreamRecords()`**——方法语义为读
4. **不读取请求体**——`searchParams` 通过 URL 传递，无 `request.text()` / `request.json()`
5. **不调用任何写入方法**——无 `appendPart`、`ingestData`、`initializeStream`、`completeWaitpoint`
6. **不修改 DB**——无 `prisma.taskRun.update` 或任何 Prisma 写操作

### 7.3 边界异常点

| 异常 | 说明 | 影响 |
|------|------|------|
| 兼容路由 POST 无认证 | `$runId/$streamId.ts` 的 action 不做认证 | 任何知道 runId 的人都可以写入 |
| 路由 B loader 不支持 GET | `$target/$streamId.ts` loader 只处理 HEAD | Run 流的 SSE 读取必须走路由 A |
| Session HEAD 返回硬编码 0 | S2 不支持按 clientId 查询进度 | 客户端必须用 SSE Last-Event-ID 续传 |
| Input 流无 PUT 初始化 | 没有 `initializeStream` 调用 | Input 流依赖 append 的 auto-create |
| 路由 B POST 对 V2 抛错 | V2 客户端不应走 POST 批量写入 | 错误信息不够明确（500 vs 400） |
| Session append 的 `ensureRun` best-effort | 创建 run 失败仍继续 append | 记录写入但可能无人消费 |

### 7.4 读写路径不相交的保证

```
写路径: action (POST/PUT) → StreamIngestor 方法 → 修改存储/DB
读路径: loader (GET/HEAD) → StreamResponder / StreamIngestor 读方法 → 不修改任何状态
```

Remix 框架保证了 action 和 loader 永远不会在同一请求中同时执行。即使在同一文件中混合导出，读写路径在运行时也不相交。

唯一需要额外注意的是**写入链中的副作用**：
- `appendPart` 后的 `completeWaitpoint` 推进任务状态
- `ensureRunForSession` 可能创建新 run
- `drainSessionStreamWaitpoints` 可能触发任务继续执行

这些副作用只存在于写路径中，读路径不涉及。

---

## 八、核心代码引用

| 路由 | 文件 | 写入行号 | 读取行号 |
|------|------|---------|---------|
| A: `$runId/$streamId` | `realtime.v1.streams.$runId.$streamId.ts` | :18-82 | :84-156 |
| B: `$target/$streamId` | `realtime.v1.streams.$runId.$target.$streamId.ts` | PUT :63-110, POST :111-145 | HEAD :199-225 |
| C: `$target/$streamId.append` | `realtime.v1.streams.$runId.$target.$streamId.append.ts` | :28-142 | 无 |
| D: `sessions/$io` | `realtime.v1.sessions.$session.$io.ts` | PUT :37-84 | GET :156-191, HEAD :146-154 |
| E: `sessions/$io.append` | `realtime.v1.sessions.$session.$io.append.ts` | :67-191 | 无 |
| F: `sessions/$io.records` | `realtime.v1.sessions.$session.$io.records.ts` | 无 | :75-97 |
| G: `input/$streamId` | `realtime.v1.streams.$runId.input.$streamId.ts` | :38-104 | :143-179 |
