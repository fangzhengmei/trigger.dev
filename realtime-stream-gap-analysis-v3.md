# Realtime Stream 写入入口深度分析 (v3)

## 一、写入入口全景：五条路由、两种模式

Trigger.dev 有**五条独立的写入入口**，覆盖 run 流、session 流、input 流三种场景，以及批量流式写入和单条记录写入两种模式。

```
                          ┌───────────────────────────┐
                          │  写入入口总览              │
                          └───────────────────────────┘
                                       │
              ┌──────────────────┬─────────────────────┬─────────────────┐
              │                  │                     │                 │
          Run 流              Session 流          Input 流        Session 流
        (V1/V2 兼容)        (V2 Only)          (V1/V2 兼容)    Records 查询
              │                  │                     │                 │
┌─────────────────────┐  ┌────────────────┐  ┌─────────────────┐  ┌───────────────┐
│ $runId/$target/$key │  │ sessions/$io   │  │ input/$streamId │  │ records       │
│   - PUT 初始化      │  │   - PUT init   │  │   - POST 发送   │  │ - GET JSON    │
│   - POST 批量写入   │  │   - GET SSE    │  │   - GET SSE     │  │   wait=0      │
│   - HEAD 查询进度   │  │   - HEAD 兼容  │  │                 │  │   (非SSE)     │
│   - append 单条     │  │   - append 单条│  │                 │  │               │
└─────────────────────┘  └────────────────┘  └─────────────────┘  └───────────────┘
```

| 路由文件 | 场景 | 支持方法 | 数据模式 | 版本支持 |
|---------|------|---------|---------|---------|
| `$runId/$target/$streamId.ts` | Run 流 | PUT, POST, HEAD | 批量流式 (POST) + 单条 (append) | V1 / V2 |
| `$runId/$target/$streamId.append.ts` | Run 流 append | POST | 单条记录 | V1 / V2 |
| `sessions/$session/$io.ts` | Session 流 | PUT, GET, HEAD | 流式 SSE | V2 Only |
| `sessions/$session/$io.append.ts` | Session 流 append | POST | 单条记录 | V2 Only |
| `input/$streamId.ts` | Input 流 | POST, GET | 单条记录 + SSE 读取 | V1 / V2 |
| `sessions/$session/$io.records.ts` | Session 记录查询 | GET | JSON 批量 (wait=0) | V2 Only |

---

## 二、统一时序：PUT → (POST/append) → HEAD → 续传

### 2.1 Run 流完整写入时序（含断点续传）

```
SDK StreamsWriterV1                   服务端路由层                        DB / Redis / S2
      │                                   │                                   │
      │ 1. PUT /streams/:runId/self/:key  │                                   │
      │    X-Stream-Version: v1           │                                   │
      │ ─────────────────────────────────▶│                                   │
      │                                   │ ┌─ 查找 run                      │
      │                                   │ ├─ target 解析 (self/parent/root) │
      │                                   │ ├─ 检查已完成？                    │
      │                                   │ ├─ streamId 注册到 realtimeStreams│
      │                                   │ │  （若不存在则 push）            │
      │                                   │ ├─ initializeStream()             │
      │                                   │ │  V1: 空操作                     │
      │                                   │ │  V2: S2 凭证 + basin 初始化     │
      │                                   │ └─ 返回 { version } + headers     │
      │ ◀──────── 202 Accepted ───────────│                                   │
      │                                   │                                   │
      │ 2. POST /streams/:runId/self/:key │                                   │
      │    X-Client-Id: a1b2c3d4          │                                   │
      │    X-Resume-From-Chunk: 0         │                                   │
      │    body: line0\nline1\n...        │                                   │
      │ ─────────────────────────────────▶│                                   │
      │                                   │ ┌─ parse headers                  │
      │                                   │ ├─ target 解析                     │
      │                                   │ ├─ ingestData(body, resumeFrom=0) │
      │                                   │ │  V1: TextDecoderStream          │
      │                                   │ │    → reader.read()              │
      │                                   │ │    → xadd with chunkIndex       │
      │                                   │ │    递增分配                      │
      │                                   │ │  V2: throw Error (直连 S2)      │
      │                                   │ └─ return 200                     │
      │ ◀──────── 200 OK ─────────────────│                                   │
      │                                   │                                   │
      ═══════ 网络中断 ═══════            │                                   │
      │                                   │                                   │
      │ 3. HEAD /streams/:runId/self/:key │                                   │
      │    X-Client-Id: a1b2c3d4          │                                   │
      │ ─────────────────────────────────▶│                                   │
      │                                   │ ┌─ getLastChunkIndex(clientId)    │
      │                                   │ │  V1: xrevrange + clientId 过滤  │
      │                                   │ │  V2: throw Error                │
      │                                   │ └─ X-Last-Chunk-Index: 42        │
      │ ◀─── 200 (X-Last-Chunk-Index: 42) │                                   │
      │                                   │                                   │
      │ 计算续传起点: 42 + 1 = 43         │                                   │
      │ 从 ringBuffer 过滤 index >= 43    │                                   │
      │                                   │                                   │
      │ 4. POST (续传)                    │                                   │
      │    X-Resume-From-Chunk: 43        │                                   │
      │    body: line43\nline44\n...      │                                   │
      │ ─────────────────────────────────▶│                                   │
      │                                   │ ┌─ ingestData(body, resumeFrom=43)│
      │                                   │ │  chunkIndex 从 43 递增          │
      │                                   │ └─ return 200                     │
      │ ◀──────── 200 OK ─────────────────│                                   │
```

### 2.2 append 路由的独立时序

append 路由用于单条记录写入，不参与断点续传协议（无 X-Resume-From-Chunk）：

```
SDK / 外部调用者                          服务端 append 路由                    存储层
      │                                         │                                  │
      │ POST /streams/:runId/self/:key/append    │                                  │
      │ X-Part-Id: (可选, 不传则生成 nanoid)     │                                  │
      │ Content-Length: <= 512KB                 │                                  │
      │ body: raw JSON string                    │                                  │
      │ ────────────────────────────────────────▶│                                  │
      │                                         │ ┌─ 校验 run 未完成               │
      │                                         │ ├─ streamId 注册到 realtimeStreams│
      │                                         │ │  （若不存在则 push）            │
      │                                         │ ├─ partId = header || nanoid(7)   │
      │                                         │ ├─ appendPart(part, partId, ...)  │
      │                                         │ │  V1: xadd with                  │
      │                                         │ │    clientId: ""                 │
      │                                         │ │    chunkIndex: "0"              │
      │                                         │ │    data: JSON.stringify(part)+"\n"│
      │                                         │ │  V2: s2Append with              │
      │                                         │ │    body: { data: part, id: partId }│
      │                                         │ └─ return { ok: true }            │
      │ ◀──────── 200 { ok: true } ─────────────│                                  │
```

### 2.3 Session 流 append 时序（AI Agent 场景）

Session 流 append 额外包含 `ensureRunForSession` 和 `drainSessionStreamWaitpoints` 两步：

```
前端发消息 → append 路由                          S2              任务引擎
      │                                         │                │
      │ POST /sessions/:session/in/append        │                │
      │ body: 用户输入 JSON                      │                │
      │ ────────────────────────────────────────▶│                │
      │                                         │ ┌─ session 存在？未关闭？        │
      │                                         │ ├─ ensureRunForSession()         │
      │                                         │ │  无运行中任务则触发新 run       │
      │                                         │ │  （失败不影响 append）          │
      │                                         │ ├─ appendPartToSessionStream()   │
      │                                         │ │  s2Append → 写入 S2            │──┐
      │                                         │ │                                │  │
      │                                         │ ├─ drainSessionStreamWaitpoints()│  │
      │                                         │ │  从 Redis 取关联的 waitpoint   │  │
      │                                         │ ├─ engine.completeWaitpoint()    │  │
      │                                         │ │  通知任务继续执行               │──┼──▶
      │                                         │ └─ return { ok: true }            │  │
      │ ◀──────── 200 { ok: true } ─────────────│                │                │  │
      │                                         │                │                │  │
      │                                         │                │◀────────────────┘  │
      │                                         │                │  Agent 处理消息    │
      │                                         │◀───────────────────────────────────│
      │                                         │  Agent 输出通过 SSE 返回            │
```

---

## 三、append 路由三要素：MAX_APPEND_BODY_BYTES、X-Part-Id、realtimeStreams 注册

### 3.1 MAX_APPEND_BODY_BYTES

**定义**：所有 append 路由（run 流、session 流、playground）统一使用 `1024 * 512 = 512KB`。

```typescript
// $runId/$target/$streamId.append.ts:16-21
// S2 enforces a 1 MiB per-record limit (metered as
// `8 + 2*H + Σ(header name+value) + body`). Cap the raw HTTP body at
// 512 KiB so the JSON wrapper, string escaping, and any future per-record
// header additions all stay well under S2's ceiling.
const MAX_APPEND_BODY_BYTES = 1024 * 512;

// 传入 createActionApiRoute 做内容长度校验
const { action } = createActionApiRoute(
  { params: ParamsSchema, maxContentLength: MAX_APPEND_BODY_BYTES },
  // ...
);
```

**输入流路由例外**：`input.$streamId.ts` 使用 `1024 * 1024 = 1MB`，因为它的 body 是 `{ data: ... }` 的 JSON 信封，实际 payload 在 `data` 字段内。

#### V1 vs V2 下的作用差异

| 维度 | V1 (Redis) | V2 (S2) |
|------|-----------|---------|
| **约束来源** | 服务端自定，无底层硬限制 | S2 协议 1 MiB 硬限制 |
| **校验时机** | `createActionApiRoute` 在进入 handler 前校验 | 同左 + S2 服务端二次校验 |
| **超限行为** | 413 Payload Too Large（由 remix 框架返回） | 413 或 S2 返回 4xx |
| **margin 计算** | 无底层限制，512KB 是人为约定 | `1 MiB - (JSON wrapper ~500B + escaping margin)` → 512KB 保守值 |
| **header 占用** | 无额外 header，只有 Redis 字段开销 | S2 per-record 开销：`8 + 2*H + Σ(header name+value)` |

**S2 记录大小公式**：
```
总大小 = 8 (固定) + 2*H (header 数量) + Σ(header name + header value) + body
body = JSON.stringify({ data: part, id: partId })
```

假设 part 是 512KB，`id` 是 7 字符 nanoid，`data` 字段经 JSON 转义后可能膨胀：
- 无特殊字符：`body ≈ 512KB + 30B (wrapper)` → 远低于 1MB
- 大量引号/反斜杠：`body` 可能翻倍 → 512KB 限制提供安全 margin

### 3.2 X-Part-Id

**定义**：可选 HTTP header，客户端可以传入一个标识本次 append 的唯一 ID。不传时服务端自动生成 `nanoid(7)`。

```typescript
// $runId/$target/$streamId.append.ts:110
const partId = request.headers.get("X-Part-Id") ?? nanoid(7);

realtimeStream.appendPart(part, partId, targetId, params.streamId);
```

#### V1 vs V2 下的作用差异

| 维度 | V1 (Redis) | V2 (S2) |
|------|-----------|---------|
| **存储位置** | 写入 S2 记录 body 的 `id` 字段（`{ data, id }`） | 同左，写入 S2 记录 body 的 `id` 字段 |
| **Redis 字段** | 不直接写入 Redis 任何字段（V1 用硬编码值） | N/A |
| **客户端去重用** | 不参与去重（V1 读取端无 seenIds 逻辑） | 用于读取端 `seenIds` Set 去重 |
| **幂等性** | ❌ 无幂等保证（重复 append 会重复写入） | ✅ 客户端可通过传入相同 X-Part-Id 实现幂等（读取端去重） |
| **默认生成** | `nanoid(7)` | `nanoid(7)` |
| **服务端可见性** | 仅作为参数透传，不存 DB | 仅作为参数透传，不存 DB |

**V1 appendPart 实现细节（Redis）**：
```typescript
// redisRealtimeStreams.server.ts:379-399
async appendPart(part: string, partId: string, runId: string, streamId: string) {
  await redis.xadd(
    streamKey,
    "MAXLEN", "~", String(env.REALTIME_STREAM_MAX_LENGTH),
    "*",
    "clientId", "",          // ← 硬编码空字符串
    "chunkIndex", "0",        // ← 硬编码 0
    "data", JSON.stringify(part) + "\n"
  );
}
```

注意 V1 的 `appendPart` **完全忽略 partId**，也不分配真实的 chunkIndex。这意味着：
1. 通过 append 写入 V1 流的记录，`chunkIndex` 永远是 0，无法参与断点续传
2. 读取端按行解析时，这些记录能正常显示，但没有递增索引
3. 这是一个设计取舍——append 是"一次性写入"语义，不支持续传

**V2 appendPart 实现细节（S2）**：
```typescript
// s2realtimeStreams.server.ts:155-176
async appendPart(part: string, partId: string, runId: string, streamId: string) {
  return this.#appendPartByName(part, partId, this.toStreamName(runId, streamId));
}

async #appendPartByName(part: string, partId: string, s2Stream: string) {
  await this.s2Append(s2Stream, {
    records: [{ body: JSON.stringify({ data: part, id: partId }) }],
  });
}
```

V2 的 `partId` 写入 S2 记录 body 的 `id` 字段，读取端 `runStream.ts` 用 `seenIds` Set 去重。

### 3.3 realtimeStreams 注册逻辑

**定义**：`taskRun.realtimeStreams` 是一个 `string[]` 字段，存储该 run 下所有已创建的流 ID。读取端通过这个数组知道有哪些流可用。

```typescript
// $runId/$target/$streamId.append.ts:89-100
if (!targetRun.realtimeStreams.includes(params.streamId)) {
  await prisma.taskRun.update({
    where: { id: targetRun.id },
    data: { realtimeStreams: { push: params.streamId } },
  });
}
```

#### V1 vs V2 下的作用差异

| 维度 | V1 (Redis) | V2 (S2) |
|------|-----------|---------|
| **注册时机** | PUT 初始化 + append 首次写入（双保险） | PUT 初始化 + append 首次写入（双保险） |
| **读取用途** | 前端 UI 展示流列表 | 前端 UI 展示流列表 |
| **S2 凭证关联** | 不关联（V1 无 S2 凭证） | 不关联（S2 stream name 由 runId + streamId 推导，不依赖 DB） |
| **幂等处理** | `includes()` 检查后才 push | `includes()` 检查后才 push |
| **PUT 路由注册** | ✅ PUT 中同样逻辑 | ✅ PUT 中同样逻辑 |
| **POST 批量写入注册** | ❌ POST ingestData 不做注册（依赖 PUT 前置） | N/A（V2 POST 抛错） |

**关键发现**：POST 批量写入路由（`$runId/$target/$streamId.ts` 的 POST 分支）**不做 realtimeStreams 注册**。它假设 PUT 已经完成了注册。如果客户端跳过 PUT 直接 POST，流数据能写入 Redis，但 `realtimeStreams` 数组不会更新，前端 UI 看不到这个流。

**PUT 中的注册逻辑**：
```typescript
// $runId/$target/$streamId.ts:88-95
if (!target.realtimeStreams.includes(params.streamId)) {
  await prisma.taskRun.update({
    where: { id: target.id },
    data: { realtimeStreams: { push: params.streamId } },
  });
}
```

这是"PUT 初始化"语义的核心——PUT 不仅返回版本号，还完成元数据注册。

### 3.4 三要素在三条 append 路由中的对比

| 要素 | Run 流 append | Session 流 append | Input 流 POST |
|------|--------------|------------------|---------------|
| `MAX_APPEND_BODY_BYTES` | 512KB | 512KB | 1MB（body 含 JSON 信封） |
| `X-Part-Id` | `header || nanoid(7)` | `header || nanoid(7)` | `inp_` 前缀 + 12 位 UUID |
| `realtimeStreams` 注册 | ✅ 检查并 push | ❌ 不注册（session 流不关联此数组） | ❌ 不注册（input 流是内部流） |
| V1 支持 | ✅ | ❌（session 流 V2 only） | ✅ |
| V2 支持 | ✅ | ✅ | ✅ |
| waitpoint 处理 | ❌ | ✅（drainSessionStreamWaitpoints） | ✅（getInputStreamWaitpoint） |

---

## 四、wait=0 的可达性分析

### 4.1 公开 Run 流路由：Timeout-Seconds 最小值约束

**文件**：`realtime.v1.streams.$runId.$streamId.ts:137-139`

```typescript
if (timeoutInSeconds && timeoutInSeconds < 1) {
  return new Response("Timeout seconds must be greater than 0", { status: 400 });
}
```

**input 流同样约束**：`realtime.v1.streams.$runId.input.$streamId.ts:154-156`

```typescript
if (timeoutInSeconds !== undefined && timeoutInSeconds < 1) {
  return new Response("Timeout seconds must be greater than 0", { status: 400 });
}
```

**结论**：公开 Run 流和 Input 流的 GET/SSE 路由，**Timeout-Seconds 必须 ≥ 1**，因此：
- Redis 场景：`inactivityThresholdMs >= 1000ms`
- S2 场景：`wait >= 1`

**wait=0 在公开路由中不可达**。客户端无法通过传入 `Timeout-Seconds: 0` 获取"仅历史"模式。

### 4.2 Session 流路由：同样有最小值约束，但 settled 检测绕过

**Session 流 GET 路由校验**：`realtime.v1.sessions.$session.$io.ts:167-169`

```typescript
if (parsed < 1) {
  return new Response("Timeout seconds must be greater than 0", { status: 400 });
}
```

但 `streamResponseFromSessionStream` 中有 settled 检测逻辑，可以**覆盖** timeoutInSeconds：

```typescript
// s2realtimeStreams.server.ts:347-360
let waitSeconds = options?.timeoutInSeconds ?? this.s2WaitSeconds;  // 默认 60，或客户端传入的 >=1
let settled = false;

if (io === "out" && options?.peekSettled) {
  settled = await this.#peekIsSettled(s2Stream);
  if (settled) {
    waitSeconds = 0;  // ← 关键：无视最小值约束，直接设为 0
  }
}
```

**绕过机制**：
1. 客户端传入 `X-Peek-Settled: 1` header（重连场景使用）
2. 服务端调用 `#peekIsSettled()`，用 `wait=0` peek S2 流尾部 2 条记录
3. 如果尾部是 `[turn-complete, trim]`，则认为流已安定
4. 安定后 `waitSeconds = 0`，传给 `#streamResponseByName`

**这个绕过是有意设计的**：`TriggerChatTransport.reconnectToStream` 在页面刷新重连时使用，避免"Agent 已完成但客户端还在长轮询空等 60 秒"的问题。

### 4.3 `#peekIsSettled` 内部的 wait=0

`#peekIsSettled` 本身也使用 `wait=0`，这是另一个可达路径：

```typescript
// s2realtimeStreams.server.ts:386-393
async #peekIsSettled(s2Stream: string): Promise<boolean> {
  const qs = new URLSearchParams();
  qs.set("tail_offset", "2");
  qs.set("count", "2");
  qs.set("wait", "0");  // ← wait=0，立即返回

  // fetch S2 API with these params
}
```

这是服务端内部调用 S2 API，不经过公开路由的校验，所以可以直接用 `wait=0`。

### 4.4 Session records 路由：显式的 wait=0 API

**文件**：`realtime.v1.sessions.$session.$io.records.ts:25-32`

```typescript
// GET: non-SSE, `wait=0` drain of a session channel. Returns a JSON body
// `{ records: StreamRecord[] }` with whatever records exist after
// `afterEventId` (or from the head if absent) and closes immediately.
//
// Used by the SDK's `replaySessionOutTail` at run boot — the SSE long-poll
// path costs ~1s per fresh chat regardless of stream content, which is
// unacceptable on the first-message TTFC budget.
```

这个路由是**显式设计**的 wait=0 API，不走 SSE，返回 JSON 格式。用于 Agent 启动时快速回放历史，避免 SSE 长轮询的开销。

### 4.5 wait=0 可达路径总结

```
wait=0 可达路径
├── 公开 Run 流 GET / SSE
│   └── ❌ 不可达（Timeout-Seconds 最小值 = 1）
├── Input 流 GET / SSE
│   └── ❌ 不可达（Timeout-Seconds 最小值 = 1）
├── Session 流 GET / SSE
│   ├── 正常路径：❌ 最小值 = 1
│   └── 安定检测路径：✅ 传入 X-Peek-Settled: 1 且流已安定
│       └── 服务端内部将 waitSeconds 覆盖为 0
├── Session records GET (JSON)
│   └── ✅ 显式 API，内部 readSessionStreamRecords 使用 wait=0
└── 服务端内部 S2 API 调用
    ├── #peekIsSettled()：✅ wait=0 peek 流尾部
    └── 其他内部调用：✅ 不受公开路由约束
```

### 4.6 Timeout-Seconds 校验位置总结

| 路由文件 | 最小值校验 | 位置 |
|---------|-----------|------|
| `$runId/$streamId.ts` (公开 Run 流) | `>= 1` | :137-139 |
| `$runId/input/$streamId.ts` (Input 流) | `>= 1` | :154-156 |
| `sessions/$session/$io.ts` (Session 流 SSE) | `>= 1` | :167-169 |
| `sessions/$session/$io.records.ts` (Session records) | 无校验（不使用 Timeout-Seconds） | N/A |

---

## 五、Gap 与设计取舍分析

### 5.1 V1 append 的 chunkIndex 硬编码问题

**问题**：`redisRealtimeStreams.appendPart()` 硬编码 `chunkIndex: "0"` 和 `clientId: ""`，忽略传入的 partId。

**影响**：
1. append 写入的记录无法参与断点续传（续传依赖 chunkIndex 递增）
2. 如果网络中断后 HEAD 查询，append 写入的记录的 clientId 是空字符串，无法通过 clientId 过滤
3. 多条 append 记录的 chunkIndex 都是 0，HEAD 查询可能返回错误的续传起点

**设计取舍**：append 被设计为"一次性写入"语义，不支持续传。如果客户端需要续传能力，应该使用 POST 批量流式写入。

### 5.2 PUT 与 POST 的注册职责不对称

**问题**：PUT 路由做 realtimeStreams 注册，但 POST 批量写入路由不做。如果客户端跳过 PUT 直接 POST，数据能写入但元数据不更新。

**影响**：
- 前端 UI 看不到未注册的流
- 流列表查询返回不完整

**设计取舍**：PUT 被设计为"初始化"语义，客户端 SDK 严格遵循 PUT → POST 顺序。这个约束在 SDK 内部保证，服务端做最小校验。

### 5.3 session settled 检测的竞态风险

**问题**：`#peekIsSettled()` 与 Agent 写入 `turn-complete` 之间存在竞态窗口。

```typescript
// #peekIsSettled
qs.set("tail_offset", "2");
qs.set("count", "2");
qs.set("wait", "0");
```

只读取尾部 2 条记录。如果 Agent 在 `turn-complete` 之后没有立即写 `trim`，或者在 peek 之后才写 `turn-complete`，会误判为未安定。

**设计取舍**：这是"最佳努力"检测——如果误判为未安定，最多多等 60 秒长轮询，不会产生错误。如果误判为已安定（极小概率），客户端会收到 `X-Session-Settled: true` 后关闭连接，下次重连时会重新检测。

### 5.4 MAX_APPEND_BODY_BYTES 的冗余校验

**问题**：`createActionApiRoute` 的 `maxContentLength` 是 Remix 框架层面的校验，但 S2 服务端会再校验一次。

**设计取舍**：
- 早失败优于晚失败——在入口层拒绝超大数据，避免浪费网络带宽
- S2 二次校验作为安全网
- 512KB 是保守值，实际可写入更大数据，但留 margin 给 JSON 转义

### 5.5 X-Part-Id 在 V1 下被忽略

**问题**：V1 `appendPart` 完全忽略 partId 参数，但 append 路由仍然生成和传递它。

**影响**：V1 场景下 X-Part-Id 无实际作用，客户端传入的自定义 ID 也被丢弃。

**设计取舍**：统一接口签名优于分支处理。V1 和 V2 共享相同的 append 路由和服务层接口，V1 忽略的参数不影响功能，只是"携带但不用"。切换到 V2 时自动获得去重能力。

### 5.6 wait=0 绕过机制的隐含假设

**问题**：`streamResponseFromSessionStream` 直接覆盖 `waitSeconds = 0`，绕过了路由层的 `>= 1` 校验。如果未来有人把 settled 检测移到路由层之前，这个绕过就会失效。

**设计取舍**：这是"内层覆盖外层"的模式，依赖调用顺序。代码注释中说明了 settled 检测的用途，后续修改时应注意这个依赖关系。

---

## 六、核心代码引用速查

| 关注点 | 文件 | 行号 |
|--------|------|------|
| Run 流 append MAX_APPEND_BODY_BYTES | `$runId/$target/$streamId.append.ts` | 21, 26 |
| Session 流 append MAX_APPEND_BODY_BYTES | `sessions/$session/$io.append.ts` | 35, 41 |
| X-Part-Id 生成（Run 流 append） | `$runId/$target/$streamId.append.ts` | 110 |
| X-Part-Id 生成（Session 流 append） | `sessions/$session/$io.append.ts` | 129 |
| realtimeStreams 注册（PUT） | `$runId/$target/$streamId.ts` | 88-95 |
| realtimeStreams 注册（append） | `$runId/$target/$streamId.append.ts` | 89-100 |
| V1 appendPart 硬编码 chunkIndex | `redisRealtimeStreams.server.ts` | 391-392 |
| V2 appendPart partId 写入 | `s2realtimeStreams.server.ts` | 175 |
| Run 流 Timeout-Seconds 最小值校验 | `$runId/$streamId.ts` | 137-139 |
| Input 流 Timeout-Seconds 最小值校验 | `input/$streamId.ts` | 154-156 |
| Session 流 Timeout-Seconds 最小值校验 | `sessions/$session/$io.ts` | 167-169 |
| settled 检测绕过 wait=0 | `s2realtimeStreams.server.ts` | 355-359 |
| #peekIsSettled 内部 wait=0 | `s2realtimeStreams.server.ts` | 393 |
| Session records API（显式 wait=0） | `sessions/$session/$io.records.ts` | 25-32, 90-94 |
| session append ensureRunForSession | `sessions/$session/$io.append.ts` | 111-124 |
| session append drainWaitpoints | `sessions/$session/$io.append.ts` | 156-188 |
| input 流 waitpoint 处理 | `input/$streamId.ts` | 88-101 |
