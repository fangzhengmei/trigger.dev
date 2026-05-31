# Realtime Stream 主链路补齐分析

## 一、V1 写入端 `req.write` 与服务端 `ingestData / xadd` 的分工

### 1.1 数据流全景

```
用户代码 yield chunk
       │
       ▼
source.tee()
  ├── consumerStream  ──▶ 本地消费（async iterator）
  └── serverStream    ──▶ startBuffering() ──▶ ringBuffer[]
                                      │
                                      ▼
                              makeRequest(startFromChunk)
                                      │
                          ┌───────────┴───────────┐
                          │  Node http.request POST │
                          │  headers:               │
                          │    X-Client-Id          │
                          │    X-Resume-From-Chunk  │
                          │    X-Stream-Version     │
                          │  body:                  │
                          │    JSON.stringify(c.data)+"\n"  ← req.write()
                          └───────────┬───────────┘
                                      │  HTTP
                                      ▼
                          ┌─────────────────────────┐
                          │  服务端 action()         │
                          │  解析 headers            │
                          │  调用 ingestData()       │
                          └───────────┬─────────────┘
                                      │
                                      ▼
                          ┌─────────────────────────┐
                          │  ingestData()            │
                          │  TextDecoderStream       │
                          │  逐行 reader.read()      │
                          │  redis.xadd(...)         │
                          └─────────────────────────┘
```

### 1.2 客户端 `req.write` 的职责

**文件**：`packages/core/src/v3/realtimeStreams/streamsWriterV1.ts:220-249`

```typescript
const processStream = async () => {
  let lastSentIndex = startFromChunk - 1;

  while (true) {
    while (lastSentIndex < this.highestBufferedIndex) {
      lastSentIndex++;
      const chunk = this.ringBuffer.find((c) => c.index === lastSentIndex);

      if (chunk) {
        const stringified = JSON.stringify(chunk.data) + "\n";
        req.write(stringified);
        this.currentChunkIndex = lastSentIndex + 1;
      }
    }

    if (this.streamComplete && lastSentIndex >= this.highestBufferedIndex) {
      req.end();
      break;
    }

    await this.delay(10);
  }
};
```

**职责清单**：

| 职责 | 实现方式 | 代码位置 |
|------|---------|---------|
| 序列化 | `JSON.stringify(chunk.data) + "\n"` | V1 Writer :231 |
| 分帧 | 每条 chunk 独立一行，`\n` 作为分隔符 | V1 Writer :231 |
| 索引追踪 | `lastSentIndex` 从 `startFromChunk` 开始递增 | V1 Writer :222 |
| 背压协调 | 10ms 轮询等待 ringBuffer 填充新数据 | V1 Writer :244 |
| 流终止 | `streamComplete && lastSentIndex >= highestBufferedIndex` → `req.end()` | V1 Writer :238-240 |
| 续传偏移 | 通过 `X-Resume-From-Chunk` header 通知服务端跳过前 N 条 | V1 Writer :103 |
| 缓冲查询 | 从 `ringBuffer` 中按 index 查找需要（重）发的 chunk | V1 Writer :228 |

**关键发现**：

1. **`req.write` 不携带 chunkIndex**：客户端只在 HTTP header 中声明 `X-Resume-From-Chunk`，body 里是纯 JSON 行流，每行本身不嵌套索引编号。索引的分配和对应完全由服务端完成。

2. **`ringBuffer.find` 是 O(n) 线性查找**：每次发一条都要遍历整个 ringBuffer 找匹配 index 的 chunk。在缓冲区大时（默认 10000）效率不高，但因写入通常是顺序的且 buffer 通常不大，实际影响有限。

3. **`\n` 分帧是关键契约**：客户端用 `JSON.stringify(data) + "\n"` 分帧，服务端用 `TextDecoderStream` + `reader.read()` 按行读取。两端的分帧逻辑必须对齐。

### 1.3 服务端 `ingestData / xadd` 的职责

**文件**：`apps/webapp/app/services/realtime/redisRealtimeStreams.server.ts:308-377`

```typescript
async ingestData(
  stream: ReadableStream<Uint8Array>,
  runId: string,
  streamId: string,
  clientId: string,
  resumeFromChunk?: number
): Promise<Response> {
  const redis = this.sharedRedis;
  const streamKey = `stream:${runId}:${streamId}`;
  const startChunk = resumeFromChunk ?? 0;
  let currentChunkIndex = startChunk;

  const textStream = stream.pipeThrough(new TextDecoderStream());
  const reader = textStream.getReader();

  while (true) {
    const { done, value } = await reader.read();
    if (done || !value) break;

    await redis.xadd(
      streamKey,
      "MAXLEN", "~", String(env.REALTIME_STREAM_MAX_LENGTH),
      "*",
      "clientId",  clientId,
      "chunkIndex", currentChunkIndex.toString(),
      "data",      value
    );

    currentChunkIndex++;
  }

  await redis.expire(streamKey, env.REALTIME_STREAM_TTL);
  return new Response(null, { status: 200 });
}
```

**职责清单**：

| 职责 | 实现方式 | 代码位置 |
|------|---------|---------|
| 反序列化 | `TextDecoderStream` 将 bytes → text，`reader.read()` 按流式块读取 | Redis Server :322-323 |
| 索引分配 | `currentChunkIndex` 从 `resumeFromChunk` 起递增，写入 Redis 条目 | Redis Server :317-319, :351 |
| 客户端标识 | 每条 XADD 记录都携带 `clientId` 字段 | Redis Server :348-349 |
| 流长度限制 | `MAXLEN ~ 1000`（近似裁剪，默认值） | Redis Server :344-346 |
| TTL 清理 | 写入结束后 `redis.expire(key, 86400)` (1 天) | Redis Server :360 |
| 续传起点 | `resumeFromChunk ?? 0`，从服务端视角继续编号 | Redis Server :317 |

### 1.4 分工边界——谁负责什么

```
┌───────────────────────────────────────────────────────────────┐
│                      客户端 (StreamsWriterV1)                 │
│                                                               │
│  ✓ 数据源读取 (source.tee → serverStream)                     │
│  ✓ 环形缓冲区管理 (startBuffering → ringBuffer)               │
│  ✓ 序列化 + 分帧 (JSON.stringify + "\n")                      │
│  ✓ 索引追踪 (lastSentIndex, 仅用于判断进度)                    │
│  ✓ 续传请求头 (X-Resume-From-Chunk, X-Client-Id)              │
│  ✓ 网络重试 + 指数退避                                        │
│  ✓ 查询服务端进度 (HEAD + queryServerLastChunkIndex)           │
│  ✓ 流终止信号 (req.end)                                       │
│                                                               │
│  ✗ 不决定 Redis 条目的 chunkIndex 值                           │
│  ✗ 不处理 Redis MAXLEN / TTL                                  │
│  ✗ 不做应用层确认（靠 HTTP 200 隐式确认）                      │
└───────────────────────────────────────────────────────────────┘
                              │
                         HTTP POST
                              │
┌───────────────────────────────────────────────────────────────┐
│                      服务端 (ingestData)                       │
│                                                               │
│  ✓ 字节流 → 文本流 (TextDecoderStream)                        │
│  ✓ 按块读取 (reader.read，不是严格按行)                        │
│  ✓ chunkIndex 分配 (从 resumeFromChunk 递增)                  │
│  ✓ clientId 打标 (写入 Redis 条目)                            │
│  ✓ Redis 写入 (XADD + MAXLEN + TTL)                          │
│  ✓ 续传偏移解析 (X-Resume-From-Chunk → resumeFromChunkNumber) │
│                                                               │
│  ✗ 不校验 chunk 内容是否真的是 JSON                             │
│  ✗ 不做 chunkIndex 与客户端的严格一致性校验                     │
│  ✗ 不做逐条确认（HTTP 是全量请求，200 = 全部成功）             │
└───────────────────────────────────────────────────────────────┘
```

### 1.5 关键 Gap：`reader.read()` 不是严格按行

客户端用 `JSON.stringify(data) + "\n"` 分帧，但服务端 `ingestData` 用的是 `reader.read()` 而非逐行读取。这意味着一个 `read()` 调用可能返回：

- 一行完整数据 + 下一行的一部分
- 多行数据
- 一行数据的中间片段

服务端直接将整个 `value` 作为 `"data"` 字段写入 Redis XADD。这之所以能工作，是因为 **读取端**（`streamResponse`）有自己的 **跨行缓冲 Transform**（`redisRealtimeStreams.server.ts:227-270`）来重新组装完整行。

但这引入了一个微妙问题：**ingestData 写入的每条 Redis 记录的 `data` 字段可能包含不完整的行，也可能包含多行**。读取端的 Transform 1 通过换行符缓冲来解决这个不一致。

**V2 (S2) 不存在此问题**：S2 的 `appendSession` 以 `AppendRecord` 为单位写入，每条记录是完整的 JSON 信封 `{ data, id }`，不存在行边界问题。

---

## 二、`$target` 路由中 PUT / POST / HEAD 组合的断点续传路径

### 2.1 两条路由的定位

Trigger.dev 有两条写入路由，形成"兼容旧客户端 + 认证新客户端"的双轨制：

| 路由文件 | URL Pattern | 认证 | 用途 |
|---------|-------------|------|------|
| `realtime.v1.streams.$runId.$streamId.ts` | `/realtime/v1/streams/:runId/:streamId` | 无认证（向后兼容） | POST 写入 + GET/SSE 读取 |
| `realtime.v1.streams.$runId.$target.$streamId.ts` | `/realtime/v1/streams/:runId/:target/:streamId` | 需认证 | PUT 初始化 + POST 写入 + HEAD 查询 |

### 2.2 `$target` 路由的三方法协议

**文件**：`apps/webapp/app/routes/realtime.v1.streams.$runId.$target.$streamId.ts`

#### PUT — 初始化流（Create）

```typescript
// :63-110
if (request.method === "PUT") {
  // 1. 查找目标 run（支持 self/parent/root 三种 target）
  // 2. 检查 run 是否已完成（已完成则 400）
  // 3. 将 streamId 推入 run.realtimeStreams 数组（若尚未存在）
  // 4. 调用 initializeStream() → S2 返回凭证或 V1 返回空
  // 5. 响应 202 + { version } + responseHeaders
  return json({ version: target.realtimeStreamsVersion }, { status: 202, headers: responseHeaders });
}
```

**PUT 做了什么**：
1. **注册流元数据**：将 `streamId` 推入 `taskRun.realtimeStreams` 数组，让读取端知道有哪些流可用
2. **S2 凭证发放**：如果是 V2，调用 `initializeStream()` 生成临时 access token 并通过响应头返回
3. **版本声明**：返回 `version: "v1" | "v2"`，客户端据此选择写入器

**PUT 的响应头（V2 场景）**：
```
X-S2-Access-Token: <temporary-token>
X-S2-Stream-Name: /runs/:runId/:streamId
X-S2-Basin: my-basin
X-S2-Flush-Interval-Ms: 100
X-S2-Max-Retries: 10
X-S2-Endpoint: <custom-endpoint>  (可选)
```

#### POST — 写入数据（Ingest）

```typescript
// :111-145
else {
  // 1. 解析 X-Client-Id, X-Stream-Version
  // 2. 解析 X-Resume-From-Chunk → resumeFromChunkNumber
  // 3. 调用 ingestData(request.body, targetId, streamId, clientId, resumeFromChunkNumber)
  return realtimeStream.ingestData(...);
}
```

**POST 做了什么**：
1. **数据接入**：将 HTTP body 中的字节流逐块写入 Redis (V1) 或抛错 (V2 — S2 数据由客户端直连写入)
2. **断点续传**：通过 `X-Resume-From-Chunk` 通知服务端从哪个索引开始重新编号
3. **客户端标识**：通过 `X-Client-Id` 关联写入来源，用于 HEAD 查询

**V2 场景下 POST 的特殊行为**：`S2RealtimeStreams.ingestData()` 直接 `throw new Error("S2 streams are written to S2 via the client, not from the server")`。也就是说，V2 模式下 POST 写入路径不可用，数据流走 S2 `appendSession` 直连。

#### HEAD — 查询服务端进度

```typescript
// loader :149-226
if (request.method !== "HEAD") {
  return new Response("Only HEAD requests are allowed for this endpoint", { status: 405 });
}

const clientId = request.headers.get("X-Client-Id") || "default";
const streamVersion = request.headers.get("X-Stream-Version") || "v1";

const lastChunkIndex = await realtimeStream.getLastChunkIndex(targetId, params.streamId, clientId);

return new Response(null, {
  status: 200,
  headers: {
    "X-Last-Chunk-Index": lastChunkIndex.toString(),
  },
});
```

**HEAD 做了什么**：
1. **只允许 HEAD 方法**：非 HEAD 请求返回 405
2. **按 clientId 查询**：在 Redis 中逆序扫描，找到该客户端最后写入的 chunkIndex
3. **返回 X-Last-Chunk-Index**：客户端用此值计算续传起点

**V2 场景下 HEAD 的特殊行为**：`S2RealtimeStreams.getLastChunkIndex()` 同样 `throw new Error(...)`。V2 不需要 HEAD，因为 S2 的 `appendSession` 内置了确认和重连机制。

### 2.3 完整的断点续传时序

```
客户端 StreamsWriterV1                     服务端 $target 路由
        │                                         │
        │  ── PUT /streams/:runId/self/:key ──▶   │
        │                                         │ 注册 streamId
        │  ◀── 202 { version: "v1" } ────────    │
        │                                         │
        │  ── POST (首次, Resume-From-Chunk: 0) ▶ │
        │     body: line0\nline1\nline2\n         │ ingestData()
        │                                         │ xadd index=0,1,2
        │                                         │
        │     ... 持续写入 ...                     │
        │                                         │
        ═══════ 网络中断 ═══════                   │
        │                                         │
        │  ── HEAD (X-Client-Id: a1b2c3d4) ──▶   │
        │                                         │ getLastChunkIndex()
        │                                         │ xrevrange → 找到 clientId
        │  ◀── 200 X-Last-Chunk-Index: 42 ────   │
        │                                         │
        │  计算续传起点: resumeFrom = 42 + 1 = 43  │
        │  从 ringBuffer 获取 index >= 43 的 chunks │
        │                                         │
        │  ── POST (Resume-From-Chunk: 43) ───▶   │
        │     body: line43\nline44\n...           │ ingestData()
        │                                         │ xadd index=43,44,...
        │  ◀── 200 ────────────────────────────   │
```

### 2.4 $target 路由的 target 解析

```typescript
// realtime.v1.streams.$runId.$target.$streamId.ts:49-61
const targetRun =
  params.target === "self"
    ? run
    : params.target === "parent"
    ? run.parentTaskRun
    : run.rootTaskRun;
```

这允许子任务将流写入父任务或根任务的上下文中，实现任务层级间的数据传递。`target` 参数影响：
1. 哪个 run 的 `realtimeStreams` 数组被更新（PUT 时）
2. 哪个 run 的 Redis stream key 被写入（POST 时）
3. 哪个 run 的 basin 上下文被用于 S2 凭证（PUT V2 时）

### 2.5 无 $target 的兼容路由

**文件**：`apps/webapp/app/routes/realtime.v1.streams.$runId.$streamId.ts`

此路由不带 `$target` 参数，仅支持 POST 写入和 GET/SSE 读取：

```typescript
// action — 无认证，向后兼容旧客户端
export async function action({ request, params }: ActionFunctionArgs) {
  // 直接从 run 查找，不检查 environment
  // 硬编码 target = self（隐式）
  return realtimeStream.ingestData(request.body, run.friendlyId, streamId, clientId, resumeFromChunkNumber);
}

// loader — GET/SSE 读取
export const loader = createLoaderApiRoute({
  // 需要 JWT 认证
  // 解析 Last-Event-ID, Timeout-Seconds
  // 调用 streamResponse()
});
```

**关键差异**：
- 无 `$target` 路由的 **action 不做认证**（向后兼容），`$target` 路由的 action 需要 API Key 认证
- 无 `$target` 路由 **不支持 PUT 和 HEAD**，只做 POST + GET
- 无 `$target` 路由的 **GET 支持 JWT 和 CORS**，`$target` 路由的 loader 不支持 JWT 且 CORS 策略为 none

---

## 三、历史查询与实时跟随在 Redis 和 S2 上的超时与 wait 行为差异

### 3.1 接口层统一入口

两种模式共享同一个 API 入口，通过 HTTP 头组合控制行为：

```
GET /realtime/v1/streams/:runId/:streamId
  Headers:
    Accept: text/event-stream
    Last-Event-ID: <id>           ← 断点续传位置
    Timeout-Seconds: <seconds>    ← 服务端长轮询超时
```

路由层校验（`realtime.v1.streams.$runId.$streamId.ts:130-143`）：
- `Timeout-Seconds` 必须 1–600 之间
- 不提供时由后端决定默认值

### 3.2 Redis (V1) 的超时与 wait 行为

**文件**：`apps/webapp/app/services/realtime/redisRealtimeStreams.server.ts:54-306`

#### 核心参数

| 参数 | 默认值 | 来源 | 说明 |
|------|--------|------|------|
| `blockTimeMs` | 5000 | 硬编码 | 单次 XREAD BLOCK 的阻塞毫秒数 |
| `inactivityTimeoutMs` | 60000 (1min) | `env.REALTIME_STREAMS_INACTIVITY_TIMEOUT_MS` | 无数据时关闭流的总时长 |
| `pingIntervalMs` | 10000 | 硬编码 | SSE ping 心跳间隔 |
| `maxRetries` | 3 | 硬编码 | XREAD 错误重试次数 |

#### 模式行为矩阵

```
                    ┌─────────────────────────────────────────┐
                    │  Timeout-Seconds 传入？                  │
                    ├──────────────┬──────────────────────────┤
                    │  未传入       │  传入 (如 60)             │
┌───────────────────┼──────────────┼──────────────────────────┤
│ 无 Last-Event-ID  │  从 "0" 开始  │  从 "0" 开始              │
│ (全新订阅)        │  BLOCK 5s    │  BLOCK 5s                │
│                   │  空闲 60s 关闭│  空闲 60s 关闭            │
├───────────────────┼──────────────┼──────────────────────────┤
│ 有 Last-Event-ID  │  从 ID 之后   │  从 ID 之后               │
│ (断点续传)        │  BLOCK 5s    │  BLOCK 5s                │
│                   │  空闲 60s 关闭│  空闲 timeout 关闭        │
└───────────────────┴──────────────┴──────────────────────────┘
```

**Redis 没有真正的"仅历史"模式**：

Redis 的 `streamResponse` 始终使用 `XREAD BLOCK 5000`，即每 5 秒轮询一次。它不会"返回历史后立即关闭"——只要客户端连接不断，它就会一直等待新数据，直到无数据超时。

超时阈值由 `inactivityThresholdMs` 控制：
```typescript
// :97-99
const inactivityThresholdMs = options?.timeoutInSeconds
  ? options.timeoutInSeconds * 1000
  : this.inactivityTimeoutMs;  // 默认 60000ms = 1min
```

- **传入 `Timeout-Seconds`**：用该值 × 1000 作为无数据超时
- **未传入**：使用 `inactivityTimeoutMs`（默认 60s）

**关键发现：Redis 的 `inactivityTimeoutMs` 默认 60s ≠ env 默认 60s**

```typescript
// env.server.ts:295
REALTIME_STREAMS_INACTIVITY_TIMEOUT_MS: z.coerce.number().int().default(60000), // 1 minute
```

env 默认 60000ms (1 分钟)，但 Redis 构造函数中：
```typescript
this.inactivityTimeoutMs = options.inactivityTimeoutMs ?? 15000; // Default: 15 seconds
```

这行代码的 `?? 15000` 是 fallback，实际值由 `v1StreamsGlobal.server.ts:26` 传入 `env.REALTIME_STREAMS_INACTIVITY_TIMEOUT_MS`（60000）。所以运行时的默认是 **60 秒**，不是 15 秒。

#### Redis 的 SSE 心跳机制

```typescript
// :89-94
const timeSinceLastEnqueue = Date.now() - lastEnqueueTime;
if (timeSinceLastEnqueue >= pingIntervalMs) {  // 10s
  controller.enqueue({ type: "ping" });
  lastEnqueueTime = Date.now();
}
```

Ping 在 Transform 2 中被转换为 SSE comment：
```typescript
// :277-278
if (chunk.type === "ping") {
  controller.enqueue(`: ping\n\n`);
}
```

这确保了：即使没有日志数据，客户端也能每 10 秒收到一个 SSE comment，防止中间代理/浏览器因空闲超时而断开连接。

### 3.3 S2 (V2) 的超时与 wait 行为

**文件**：`apps/webapp/app/services/realtime/s2realtimeStreams.server.ts:464-483`

#### 核心参数

| 参数 | 默认值 | 来源 | 说明 |
|------|--------|------|------|
| `s2WaitSeconds` | 60 | `env.REALTIME_STREAMS_S2_WAIT_SECONDS` | S2 GET records 的 `wait` 查询参数 |
| `timeoutInSeconds` | (客户端传入) | `Timeout-Seconds` 头 | 覆盖 s2WaitSeconds |

#### S2 的 wait 参数语义

S2 的 `wait` 参数行为与 Redis `XREAD BLOCK` **本质不同**：

| 维度 | Redis XREAD BLOCK | S2 wait |
|------|-------------------|---------|
| 语义 | "最多阻塞 N 毫秒" | "如果无新数据，保持连接 N 秒" |
| 有数据时 | 立即返回，不等待 | 流式返回，连接保持 |
| 无数据时 | 阻塞到超时后返回空 | 保持连接直到 wait 秒后关闭 |
| 返回后 | 需要重新 XREAD | 连接关闭，客户端重连 |
| 历史数据 | 先返回历史，然后阻塞等新 | 先返回历史，然后阻塞等新 |

#### 模式行为矩阵

```
                    ┌───────────────────────────────────────────┐
                    │  Timeout-Seconds 传入？                    │
                    ├───────────────┬───────────────────────────┤
                    │  未传入        │  传入 (如 60)              │
┌───────────────────┼───────────────┼───────────────────────────┤
│ 无 Last-Event-ID  │  seq_num=0    │  seq_num=0                │
│ (全新订阅)        │  clamp=true   │  clamp=true               │
│                   │  wait=60      │  wait=60                   │
│                   │  返回所有历史  │  返回所有历史               │
│                   │  + 等待 60s    │  + 等待 60s                │
├───────────────────┼───────────────┼───────────────────────────┤
│ 有 Last-Event-ID  │  seq_num=N+1  │  seq_num=N+1              │
│ (断点续传)        │  clamp=true   │  clamp=true               │
│                   │  wait=60      │  wait=60                   │
│                   │  返回 N 后历史 │  返回 N 后历史              │
│                   │  + 等待 60s    │  + 等待 60s                │
└───────────────────┴───────────────┴───────────────────────────┘
```

#### `clamp=true` 的作用

```typescript
// s2realtimeStreams.server.ts:475
qs.set("clamp", "true");
```

`clamp` 告诉 S2：如果请求的 `seq_num` 小于流中现存最早记录的 seq_num，自动调整到最早可用位置。这防止了因 S2 的 trim 操作导致旧数据已删除时返回错误——而是从现存最早的数据开始返回。

#### S2 的 `parseLastEventId` 行为

```typescript
// :659-665
private parseLastEventId(lastEventId?: string): number | undefined {
  if (!lastEventId) return undefined;
  const digits = lastEventId.split("-")[0];
  const n = Number(digits);
  return Number.isFinite(n) && n >= 0 ? n + 1 : undefined;
}
```

注意 `n + 1`：Last-Event-ID 指向的是客户端已收到的最后一条记录的 seq_num，所以续传从 `seq_num + 1` 开始。这与 V1 中 Redis stream ID 的处理不同——Redis 的 XREAD 使用 `>` 表示"仅新消息"，使用具体 ID 表示"该 ID 之后的消息"，客户端的 `lastId` 直接传给 XREAD 即可（XREAD 本身就是"ID 之后"的语义）。

### 3.4 关键行为差异对比

| 维度 | Redis (V1) | S2 (V2) |
|------|-----------|---------|
| **超时单位** | 毫秒 (`blockTimeMs=5000`) | 秒 (`wait=60`) |
| **超时粒度** | 每 5s 一次轮询周期 | 单次长轮询，由 S2 服务端控制 |
| **默认无数据超时** | 60s (`env` 可配) | 60s (`env` 可配) |
| **仅历史模式** | ❌ 不支持，总会 BLOCK 等待 | ✅ `wait=0` 返回已有数据后立即关闭 |
| **历史+实时模式** | ✅ 先返回历史，持续 BLOCK | ✅ 先返回历史，wait 秒内等待新数据 |
| **心跳** | 每 10s 发 SSE ping comment | 无应用层心跳（S2 协议层可能有自己的 keepalive） |
| **续传起点** | Last-Event-ID 直接传给 XREAD | Last-Event-ID → `seq_num + 1` |
| **越界处理** | 无越界问题（Redis 自动处理） | `clamp=true` 自动调整到有效范围 |
| **空闲检测** | 两个分支（有消息/无消息）都检测 | 由 S2 wait 参数控制 |
| **服务端重试** | XREAD 错误重试 3 次，线性退避 | 无（S2 连接失败直接抛错给客户端） |
| **连接模型** | 每次请求创建独立 Redis 连接 | 代理 S2 的 SSE 响应，无状态 |

### 3.5 `Timeout-Seconds` 的完整传递链

```
客户端 SSEStreamSubscription
  │  headers: { "Timeout-Seconds": "120" }
  ▼
路由 loader (realtime.v1.streams.$runId.$streamId.ts:130)
  │  解析 → timeoutInSeconds = 120
  │  校验 1 <= 120 <= 600
  ▼
streamResponse(request, runId, streamId, signal, { timeoutInSeconds: 120 })
  │
  ├── Redis: inactivityThresholdMs = 120 * 1000 = 120000ms
  │         → 无数据 120s 后关闭流
  │
  └── S2:    wait = timeoutInSeconds ?? s2WaitSeconds = 120
            → S2 保持连接 120s，期间有数据则流式返回
```

### 3.6 Session 流的特殊超时行为

S2 的 `streamResponseFromSessionStream` 增加了"安定检测"（settled detection）：

```typescript
// s2realtimeStreams.server.ts:347-359
let waitSeconds = options?.timeoutInSeconds ?? this.s2WaitSeconds;  // 默认 60

if (io === "out" && options?.peekSettled) {
  settled = await this.#peekIsSettled(s2Stream);
  if (settled) {
    waitSeconds = 0;  // ← 关键：安定时切换到仅历史模式
  }
}
```

| 状态 | waitSeconds | 行为 | X-Session-Settled |
|------|-------------|------|-------------------|
| Agent 正在输出 | 60 (或客户端指定) | 长轮询，持续流式返回 | 不设置 |
| Agent turn 完成 (settled) | 0 | 返回已有数据后立即关闭 | `true` |
| 未开启 peekSettled | 60 (或客户端指定) | 长轮询 | 不设置 |

这个机制解决了"Agent 已完成但客户端还在长轮询空等"的问题——安定时直接 `wait=0` 拉完历史就关闭。

---

## 四、Gap 分析与发现

### 4.1 V1 ingestData 的行边界问题

**问题**：客户端用 `\n` 分帧，但服务端 `ingestData` 的 `reader.read()` 不是按行读取，而是按流式块读取。一个块可能包含多行或半行。写入 Redis 的 `data` 字段可能包含不完整的 JSON 行。

**影响**：读取端的 Transform 1（跨行缓冲）能重新组装，但这意味着 Redis 中的原始数据不是自描述的——单条 Redis 记录可能无法独立解析为有效 JSON。

**建议**：如果需要 Redis 中的数据能独立解析，可在 `ingestData` 中使用与 `streamResponse` 相同的按行 Transform。

### 4.2 V2 的 POST/HEAD 路由不兼容

**问题**：`$target` 路由的 POST 分支对 V2 会调用 `ingestData()`，而 S2 的 `ingestData()` 直接 `throw new Error`。如果 V2 客户端错误地使用 POST 写入，会得到 500 错误。

**影响**：客户端 SDK 的 `StreamInstance` 通过 `parseCreateStreamResponse` 正确选择 V2 路径（直连 S2），不会走 POST。但如果有手工构造的客户端使用 V2 + POST，会失败。

**建议**：在 `$target` 路由的 POST 分支中，当 `streamVersion === "v2"` 时返回更明确的 400 错误而非 500。

### 4.3 Redis 心跳间隔 vs 无数据超时

**问题**：Redis 的 `pingIntervalMs=10000`（10s），但 `inactivityTimeoutMs` 默认 60000ms。Ping 只在 `lastEnqueueTime >= 10000ms` 时才发，而 `lastEnqueueTime` 只在有实际数据或 ping 时更新。但 **ping 不重置 `lastDataTime`**，而 `lastDataTime` 才是超时判断的依据。

```typescript
// :89-94 — ping 只更新 lastEnqueueTime
if (timeSinceLastEnqueue >= pingIntervalMs) {
  controller.enqueue({ type: "ping" });
  lastEnqueueTime = Date.now();  // ← 更新的是 enqueueTime
}

// :153-154 — 数据更新 lastDataTime
lastDataTime = Date.now();       // ← 更新的是 dataTime
lastEnqueueTime = Date.now();

// :168-169, :184-185 — 超时判断用的是 lastDataTime
const inactiveMs = Date.now() - lastDataTime;
if (inactiveMs >= inactivityThresholdMs) { controller.close(); }
```

**影响**：即使客户端持续收到 ping，如果 60s 内没有实际日志数据，流仍会被关闭。这是**正确行为**——ping 仅用于保持 TCP 连接活跃，不应影响业务超时。

### 4.4 S2 没有 Redis 的三层重试

**问题**：Redis 的 `streamResponse` 有内置的 XREAD 错误重试（3 次，线性退避 `1s × retryCount`），但 S2 的 `#streamResponseByName` 没有服务端重试——S2 连接失败直接抛错，由客户端的 `SSEStreamSubscription.retryConnection` 处理。

**影响**：S2 的可靠性完全依赖客户端重连机制，服务端不做保护。这在网络抖动场景下可能导致更频繁的 SSE 流中断和重建。

### 4.5 `parseLastEventId` 的 V1→V2 兼容性

**问题**：S2 的 `parseLastEventId` 对 V1 格式（`1699999999999-5`）做了兼容处理——取 `-` 前的数字部分。但 V1 的 Redis stream ID 是毫秒时间戳，S2 的 seq_num 是递增整数。如果 V1 客户端带着 `Last-Event-ID: 1699999999999-5` 连到 V2 的 S2 端点，`parseLastEventId` 会返回 `1699999999999 + 1 = 1700000000000`，这是一个远超 S2 实际 seq_num 范围的值。

**实际影响**：V1/V2 路由由 `realtimeStreamsVersion` 字段隔离，正常不会跨版本。但 `clamp=true` 提供了安全网——超范围的 seq_num 会被自动调整到有效范围。

### 4.6 `REALTIME_STREAM_MAX_LENGTH` 的近似裁剪

**问题**：`MAXLEN ~ 1000` 使用 Redis 的近似裁剪（`~` 标记），实际可能保留略多于 1000 条记录。在断点续传场景下，如果 HEAD 请求查询的 chunkIndex 对应的记录已被裁剪，`getLastChunkIndex` 会遍历完整个流都找不到，返回 -1。

**影响**：流数据量很大时，断点续传可能因旧数据被裁剪而无法准确定位续传起点。客户端会退回到 `resumeFromChunk = -1 + 1 = 0`，从头重发所有数据。

---

## 五、环境变量速查

| 变量 | 默认值 | 影响范围 |
|------|--------|---------|
| `REALTIME_STREAM_MAX_LENGTH` | 1000 | Redis XADD MAXLEN 限制 |
| `REALTIME_STREAM_TTL` | 86400 (1d) | Redis stream key 过期时间 |
| `REALTIME_STREAMS_INACTIVITY_TIMEOUT_MS` | 60000 (1min) | Redis SSE 无数据超时 |
| `REALTIME_STREAMS_S2_WAIT_SECONDS` | 60 | S2 默认 wait 参数 |
| `REALTIME_STREAMS_S2_FLUSH_INTERVAL_MS` | 100 | S2 BatchTransform 聚合间隔 |
| `REALTIME_STREAMS_S2_MAX_RETRIES` | 10 | S2 append 重试次数 |
| `REALTIME_STREAMS_S2_ACCESS_TOKEN_EXPIRATION_IN_MS` | 86400000 (1d) | S2 临时凭证有效期 |
| `REALTIME_STREAMS_DEFAULT_VERSION` | "v1" | 默认流版本 |

---

## 六、核心代码引用

| 关注点 | 文件 | 行号 |
|--------|------|------|
| 客户端 req.write 循环 | `streamsWriterV1.ts` | 220-249 |
| 服务端 ingestData | `redisRealtimeStreams.server.ts` | 308-377 |
| $target PUT 初始化 | `realtime.v1.streams.$runId.$target.$streamId.ts` | 63-110 |
| $target POST 写入 | `realtime.v1.streams.$runId.$target.$streamId.ts` | 111-145 |
| $target HEAD 查询 | `realtime.v1.streams.$runId.$target.$streamId.ts` | 149-226 |
| target 解析 (self/parent/root) | `realtime.v1.streams.$runId.$target.$streamId.ts` | 49-61 |
| Redis streamResponse 超时 | `redisRealtimeStreams.server.ts` | 97-99 |
| Redis XREAD BLOCK | `redisRealtimeStreams.server.ts` | 102-110 |
| Redis 心跳 ping | `redisRealtimeStreams.server.ts` | 89-94 |
| S2 streamResponse wait | `s2realtimeStreams.server.ts` | 474-479 |
| S2 parseLastEventId | `s2realtimeStreams.server.ts` | 659-665 |
| S2 settled 检测 | `s2realtimeStreams.server.ts` | 347-359 |
| S2 ingestData 抛错 | `s2realtimeStreams.server.ts` | 145-153 |
| 版本路由 | `v1StreamsGlobal.server.ts` | 55-85 |
| StreamInstance 版本选择 | `streamInstance.ts` | 42-82 |
| Timeout-Seconds 校验 | `realtime.v1.streams.$runId.$streamId.ts` | 130-143 |
| append 路由 (单条) | `realtime.v1.streams.$runId.$target.$streamId.append.ts` | 1-143 |
