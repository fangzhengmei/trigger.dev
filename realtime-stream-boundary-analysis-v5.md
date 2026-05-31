# Realtime Stream 边界口径分析 (v5)

## 一、核心机制：`createActionApiRoute` 的 action/loader 行为

### 1.1 `createActionApiRoute` loader 的真实行为

**文件**：`apps/webapp/app/services/routeBuilders/apiBuilder.server.ts:730-736`

```typescript
export function createActionApiRoute(options, handler) {
  async function loader({ request, params }: LoaderFunctionArgs) {
    if (corsStrategy !== "none" && request.method.toUpperCase() === "OPTIONS") {
      return apiCors(request, json({}));
    }
    return new Response(null, { status: 405 });  // ← 永远 405！
  }

  async function action({ request, params }: ActionFunctionArgs) {
    // ... 完整的 action 处理逻辑
  }

  return { loader, action };
}
```

**关键发现**：`createActionApiRoute` 返回的 `loader` 是一个**硬编码 405 的占位符**，与传入的 options/handler 完全无关。它只处理两件事：
1. OPTIONS 请求 → CORS 响应（如果 `corsStrategy !== "none"`）
2. 其他任何请求 → `405 Method Not Allowed`

### 1.2 `createActionApiRoute` action 的 method 过滤

**文件**：`apiBuilder.server.ts:739-750`

```typescript
async function action({ request, params }: ActionFunctionArgs) {
  if (options.method) {
    if (request.method.toUpperCase() !== options.method) {
      return wrapResponse(
        request,
        json({ error: "Method not allowed" }, { status: 405, headers: { Allow: options.method } }),
        corsStrategy !== "none"
      );
    }
  }
  // ... 后续处理
}
```

如果配置了 `method: "POST"`（或其他方法），action 会：
- 只接受指定的 HTTP 方法
- 其他方法返回 405，并在 `Allow` 响应头中告知支持的方法

### 1.3 `createLoaderApiRoute` vs `createActionApiRoute`

| 特性 | `createLoaderApiRoute` | `createActionApiRoute` |
|------|------------------------|------------------------|
| loader | 完整实现：认证、参数解析、授权、handler 执行 | 硬编码 405 占位符 |
| action | 无 | 完整实现：method 过滤、认证、参数解析、授权、body 解析、handler 执行 |
| body 解析 | 无 | 支持 schema 校验 |
| CORS | 支持 | 支持 |
| JWT | 可选 (`allowJWT`) | 可选 (`allowJWT`) |

**结论**：
- 真实的读取能力 **只能** 来自 `createLoaderApiRoute`
- `createActionApiRoute` 的 loader **永远不提供任何数据读取能力**，只返回 405
- 即使 `createActionApiRoute` 导出了 `{ action, loader }`，也不意味着这个路由有读取能力

---

## 二、按文件逐路由核对真实读写能力

### 2.1 路由分类标准

| 导出形式 | 生成函数 | 真实能力 |
|---------|----------|---------|
| `export { action }` | `createActionApiRoute` | 纯写（无读取能力） |
| `export { action, loader }` from `createActionApiRoute` | `createActionApiRoute` | 纯写（loader 永远 405） |
| `export { action, loader }` 其中 loader 来自 `createLoaderApiRoute` | 混合 | 读写混合 |
| `export const loader = createLoaderApiRoute(...)` | `createLoaderApiRoute` | 纯读 |
| 独立 `export async function action` + `export const loader` | 混合 | 读写混合 |

---

### 2.2 路由 A：`$runId/$streamId.ts`（兼容路由）

**文件**：`apps/webapp/app/routes/realtime.v1.streams.$runId.$streamId.ts`

| 导出 | 生成方式 | HTTP 方法 | 真实能力 |
|------|---------|-----------|---------|
| `export async function action` | 独立函数 | POST | **写**：`ingestData()` 批量流式写入 |
| `export const loader = createLoaderApiRoute(...)` | `createLoaderApiRoute` | GET | **读**：`streamResponse()` SSE 流式读取 |

**读写混合** ✅：
- 写：独立的 `action` 函数处理 POST
- 读：`createLoaderApiRoute` 生成的 `loader` 处理 GET/SSE

---

### 2.3 路由 B：`$target/$streamId.ts`（认证路由）

**文件**：`apps/webapp/app/routes/realtime.v1.streams.$runId.$target.$streamId.ts`

| 导出 | 生成方式 | HTTP 方法 | 真实能力 |
|------|---------|-----------|---------|
| `action` (from createActionApiRoute) | `createActionApiRoute` | PUT, POST | **写** |
| `loader` (from createActionApiRoute) | `createActionApiRoute` | 任何 | 405 ❌ |

**等等，这个结论不对！**

让我重新核对——这个路由的 loader **不是** 来自 `createActionApiRoute`：

```typescript
// 第 16-147 行：action 来自 createActionApiRoute
const { action } = createActionApiRoute(
  { params: ParamsSchema },
  async ({ request, params, authentication }) => {
    if (request.method === "PUT") { ... }  // 初始化
    else { ... }  // POST 批量写入
  }
);

// 第 149-226 行：loader 来自 createLoaderApiRoute
const loader = createLoaderApiRoute(
  { params: ParamsSchema, allowJWT: false, corsStrategy: "none", ... },
  async ({ request, params, resource: run, authentication }) => {
    if (request.method !== "HEAD") {
      return new Response("Only HEAD requests are allowed for this endpoint", { status: 405 });
    }
    // ... getLastChunkIndex
  }
);

export { action, loader };  // ← 分别来自两个 builder！
```

**更正后的真实能力**：

| 导出 | 生成方式 | HTTP 方法 | 真实能力 |
|------|---------|-----------|---------|
| `action` | `createActionApiRoute`（无 method 限制） | PUT | **写**：元数据注册 + S2 凭证 |
| `action` | `createActionApiRoute`（无 method 限制） | POST | **写**：`ingestData()` 批量流式写入 |
| `loader` | `createLoaderApiRoute` | HEAD | **读**：`getLastChunkIndex()` 查询进度 |
| `loader` | `createLoaderApiRoute` | GET | 405（被 handler 内部拒绝） |

**读写混合** ✅：
- 写：action 处理 PUT/POST
- 读：loader 处理 HEAD（不支持 GET/SSE）

---

### 2.4 路由 C：`$target/$streamId.append.ts`

**文件**：`apps/webapp/app/routes/realtime.v1.streams.$runId.$target.$streamId.append.ts`

```typescript
const { action } = createActionApiRoute(
  { params: ParamsSchema, maxContentLength: MAX_APPEND_BODY_BYTES },
  handler
);

export { action };  // ← 只导出 action，不导出 loader！
```

| 导出 | 生成方式 | HTTP 方法 | 真实能力 |
|------|---------|-----------|---------|
| `action` | `createActionApiRoute`（无 method 限制） | POST | **写**：`appendPart()` 单条记录写入 |

**纯写路由** ✅：
- 无 loader 导出
- 不支持任何读取操作

---

### 2.5 路由 D：`sessions/$session/$io.ts`

**文件**：`apps/webapp/app/routes/realtime.v1.sessions.$session.$io.ts`

```typescript
const { action } = createActionApiRoute(
  { params: ParamsSchema, method: "PUT", allowJWT: true, corsStrategy: "all", ... },
  async ({ params, authentication }) => {
    // initializeSessionStream() 发放 S2 凭证
  }
);

const loader = createLoaderApiRoute(
  { params: ParamsSchema, allowJWT: true, corsStrategy: "all", ... },
  async ({ params, request, authentication, resource }) => {
    if (request.method === "HEAD") {
      return new Response(null, { status: 200, headers: { "X-Last-Chunk-Index": "0" } });
    }
    // streamResponseFromSessionStream() SSE 读取
  }
);

export { action, loader };  // ← 分别来自两个 builder！
```

| 导出 | 生成方式 | HTTP 方法 | 真实能力 |
|------|---------|-----------|---------|
| `action` | `createActionApiRoute(method: "PUT")` | PUT | **写**：`initializeSessionStream()` 发放 S2 凭证 |
| `action` | `createActionApiRoute(method: "PUT")` | 非 PUT | 405 |
| `loader` | `createLoaderApiRoute` | GET | **读**：`streamResponseFromSessionStream()` SSE 读取 |
| `loader` | `createLoaderApiRoute` | HEAD | **读**：返回硬编码 `X-Last-Chunk-Index: 0`（兼容占位） |

**读写混合** ✅：
- 写：action 处理 PUT（凭证发放）
- 读：loader 处理 GET/SSE + HEAD（占位）

---

### 2.6 路由 E：`sessions/$session/$io.append.ts`

**文件**：`apps/webapp/app/routes/realtime.v1.sessions.$session.$io.append.ts`

```typescript
const { action, loader } = createActionApiRoute(
  {
    params: ParamsSchema,
    method: "POST",      // ← 显式指定 method
    maxContentLength: MAX_APPEND_BODY_BYTES,
    allowJWT: true,
    corsStrategy: "all",
    findResource: async (params, auth) => ...,
    authorization: { action: "write", ... },
  },
  async ({ request, params, authentication, resource: session }) => {
    // ensureRunForSession → appendPartToSessionStream → drainSessionStreamWaitpoints
  }
);

export { action, loader };  // ← 都来自 createActionApiRoute！
```

**关键**：虽然导出了 `loader`，但它是 `createActionApiRoute` 返回的 loader——**永远返回 405**。

| 导出 | 生成方式 | HTTP 方法 | 真实能力 |
|------|---------|-----------|---------|
| `action` | `createActionApiRoute(method: "POST")` | POST | **写**：`appendPartToSessionStream()` 单条记录写入 |
| `action` | `createActionApiRoute(method: "POST")` | 非 POST | 405 |
| `loader` | `createActionApiRoute`（占位实现） | 任何 | **405** ❌ |

**纯写路由** ✅：
- 表面上导出了 loader，但实际上无任何读取能力
- loader 只可能返回 405

---

### 2.7 路由 F：`sessions/$session/$io.records.ts`

**文件**：`apps/webapp/app/routes/realtime.v1.sessions.$session.$io.records.ts`

```typescript
export const loader = createLoaderApiRoute(
  { params: ParamsSchema, searchParams: SearchSchema, allowJWT: true, corsStrategy: "all", ... },
  async ({ params, authentication, resource, searchParams }) => {
    const records = await realtimeStream.readSessionStreamRecords(
      resource.addressingKey,
      params.io,
      afterSeqNum
    );
    return json({ records });
  }
);
```

| 导出 | 生成方式 | HTTP 方法 | 真实能力 |
|------|---------|-----------|---------|
| `loader` | `createLoaderApiRoute` | GET | **读**：`readSessionStreamRecords()` JSON 批量返回 |
| 无 action 导出 | - | 非 GET | 404/405 |

**纯读路由** ✅：
- 只导出 loader，不导出 action
- 不支持任何写入操作

---

### 2.8 路由 G：`$runId/input/$streamId.ts`

**文件**：`apps/webapp/app/routes/realtime.v1.streams.$runId.input.$streamId.ts`

```typescript
const { action } = createActionApiRoute(
  { params: ParamsSchema, maxContentLength: 1024 * 1024, allowJWT: true, corsStrategy: "all", ... },
  async ({ request, params, authentication }) => {
    // appendPart() → completeWaitpoint()
  }
);

const loader = createLoaderApiRoute(
  { params: ParamsSchema, allowJWT: true, corsStrategy: "all", ... },
  async ({ params, request, resource: run, authentication }) => {
    // streamResponse() SSE 读取
  }
);

export { action, loader };  // ← 分别来自两个 builder！
```

| 导出 | 生成方式 | HTTP 方法 | 真实能力 |
|------|---------|-----------|---------|
| `action` | `createActionApiRoute`（无 method 限制） | POST | **写**：`appendPart()` 单条记录写入 + waitpoint 处理 |
| `loader` | `createLoaderApiRoute` | GET | **读**：`streamResponse()` SSE 读取 |

**读写混合** ✅：
- 写：action 处理 POST
- 读：loader 处理 GET/SSE

---

### 2.9 7 条路由真实能力汇总表

| 路由文件 | 写入口 | 读入口 | 类型 |
|---------|--------|--------|------|
| **A** `$runId/$streamId.ts` | POST → `ingestData()` | GET/SSE → `streamResponse()` | 读写混合 |
| **B** `$target/$streamId.ts` | PUT → 元数据 + 凭证<br>POST → `ingestData()` | HEAD → `getLastChunkIndex()` | 读写混合 |
| **C** `$target/$streamId.append.ts` | POST → `appendPart()` | ❌ 无读取能力 | 纯写 |
| **D** `sessions/$session/$io.ts` | PUT → `initializeSessionStream()` | GET/SSE → `streamResponseFromSessionStream()`<br>HEAD → 兼容占位 | 读写混合 |
| **E** `sessions/$session/$io.append.ts` | POST → `appendPartToSessionStream()` | ❌ loader 永远 405 | 纯写 |
| **F** `sessions/$session/$io.records.ts` | ❌ 无写入能力 | GET → `readSessionStreamRecords()` | 纯读 |
| **G** `$runId/input/$streamId.ts` | POST → `appendPart()` | GET/SSE → `streamResponse()` | 读写混合 |

---

## 三、不夸大读能力的分层链路图

### 3.1 分层原则

1. **严格按代码导出划分**：不根据文件名推测能力，只看实际导出的 action/loader
2. **区分"导出了 loader"和"有真实读取能力"**：`createActionApiRoute` 导出的 loader 不算
3. **精确到 HTTP 方法**：同一个导出可能支持多个方法，也可能只支持部分

### 3.2 写链路（分层）

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          写链路 — 3 类纯写入口                                  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Run 流 批量写入                                                           │
│     ├── 路由 A POST (兼容)   → ingestData()     → Redis XADD / S2 抛错        │
│     └── 路由 B POST (认证)   → ingestData()     → Redis XADD / S2 抛错        │
│                                                                              │
│  2. Run 流 单条写入                                                           │
│     └── 路由 C POST (append) → appendPart()     → Redis XADD / S2 AppendRecords│
│                                                                              │
│  3. Run 流 初始化 + 凭证                                                      │
│     └── 路由 B PUT           → DB push + initializeStream() → S2 凭证发放    │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  4. Session 流 初始化 + 凭证                                                  │
│     └── 路由 D PUT           → initializeSessionStream() → S2 凭证发放       │
│                                                                              │
│  5. Session 流 单条写入                                                       │
│     └── 路由 E POST (append) → appendPartToSessionStream()                   │
│                          + ensureRunForSession()                             │
│                          + drainSessionStreamWaitpoints()                    │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  6. Input 流 单条写入                                                         │
│     └── 路由 G POST           → appendPart()                                  │
│                          + completeWaitpoint()                                │
│                          + deleteInputStreamWaitpoint()                       │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 读链路（分层）

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          读链路 — 4 类纯读入口                                  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Run 流 SSE 流式读取                                                       │
│     └── 路由 A GET/SSE       → streamResponse() → SSE 长流                   │
│         支持: Last-Event-ID, Timeout-Seconds (1-600)                         │
│                                                                              │
│  2. Run 流 断点续传查询                                                       │
│     └── 路由 B HEAD          → getLastChunkIndex() → X-Last-Chunk-Index      │
│         只支持 HEAD，不支持 GET/SSE！                                         │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  3. Session 流 SSE 流式读取                                                   │
│     └── 路由 D GET/SSE       → streamResponseFromSessionStream() → SSE 长流   │
│         支持: Last-Event-ID, Timeout-Seconds (1-600), X-Peek-Settled         │
│                                                                              │
│  4. Session 流 兼容占位查询                                                   │
│     └── 路由 D HEAD          → 返回 X-Last-Chunk-Index: 0 (硬编码)            │
│         S2 不需要 HEAD 续传，只为兼容 API 形状保留                            │
│                                                                              │
│  5. Session 流 JSON 批量查询（records）                                       │
│     └── 路由 F GET           → readSessionStreamRecords() → { records: [...] }│
│         支持: afterEventId 查询参数，wait=0 立即返回                          │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  6. Input 流 SSE 流式读取                                                     │
│     └── 路由 G GET/SSE       → streamResponse() → SSE 长流                   │
│         读取内部流名: $trigger.input:${streamId}                              │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 读写边界总览图（不夸大版）

```
                    ┌──────────────────────────────────────────────┐
                    │         Realtime Stream 接口边界              │
                    └──────────────────────────────────────────────┘

┌─────────────┐       ┌──────────────────────────────────────────────────┐
│   SDK 写端   │       │             写入口 (7 个)                        │
│             │       │                                                    │
│  Run 流     │──────▶│  A POST  (兼容)    → ingestData                   │
│  Writer     │       │  B PUT           → 初始化 + 凭证                   │
│             │──────▶│  B POST  (认证)    → ingestData                   │
│             │       │  C POST  (append)  → appendPart                   │
│             │       │                                                    │
│  Session 流 │──────▶│  D PUT           → 初始化 + 凭证                   │
│  .writer()  │       │  E POST  (append)  → appendPartToSessionStream   │
│             │       │                                                    │
│  Input 流   │──────▶│  G POST           → appendPart                    │
│  .send()    │       │                                                    │
└─────────────┘       └──────────────────────────────────────────────────┘

                                ┊
                                ┊ 边界
                                ┊

┌─────────────┐       ┌──────────────────────────────────────────────────┐
│   SDK 读端   │       │             读入口 (6 个)                        │
│             │       │                                                    │
│  Run 流     │──────▶│  A GET/SSE       → streamResponse                 │
│  UI/日志    │       │  B HEAD          → getLastChunkIndex              │
│             │       │                                                    │
│  Session 流 │──────▶│  D GET/SSE       → streamResponseFromSessionStream│
│  Transport  │       │  D HEAD          → 兼容占位 (X-Last-Chunk-Index: 0)│
│             │──────▶│  F GET  (records) → readSessionStreamRecords      │
│             │       │       ★ 纯读，无任何写入能力                        │
│             │       │                                                    │
│  Input 流   │──────▶│  G GET/SSE       → streamResponse                 │
│  SSE tail   │       │       读取内部流 $trigger.input:*                  │
└─────────────┘       └──────────────────────────────────────────────────┘

                    ★  3 条路由纯写，1 条路由纯读，3 条路由读写混合
```

---

## 四、边界澄清的关键发现

### 4.1 容易误判的三个陷阱

| 陷阱 | 实际情况 | 证据 |
|------|---------|------|
| "导出了 loader 就有读取能力" | ❌ `createActionApiRoute` 的 loader 永远 405 | `apiBuilder.server.ts:730-736` |
| "E 路由（sessions append）是读写混合" | ❌ E 路由的 loader 来自 `createActionApiRoute`，永远 405 | `sessions.$session.$io.append.ts:37` + `apiBuilder.server.ts:730-736` |
| "B 路由（$target/$streamId）支持 GET/SSE" | ❌ B 路由的 loader 只处理 HEAD，GET 被 handler 内部拒绝 | `$target/$streamId.ts:199-201` |

### 4.2 E 路由导出 loader 但仅返 405 的语义

**文件**：`realtime.v1.sessions.$session.$io.append.ts:194`

```typescript
export { action, loader };
```

虽然导出了 loader，但：

1. **loader 来自 `createActionApiRoute`** → 硬编码 405
2. **action 配置了 `method: "POST"`** → 非 POST 返回 405
3. **authorization action 是 `"write"`** → 即使绕过 405，也需要 write 权限

**为什么要导出一个永远 405 的 loader？**

这是 `createActionApiRoute` API 设计的附带产物：
- 函数签名固定返回 `{ loader, action }`
- 调用者通常解构赋值：`const { action, loader } = createActionApiRoute(...)`
- 如果不导出 loader，TS 会报错"已声明但未使用"
- 导出但不用，是最简单的处理方式

**语义本质**：这不是"提供了读取能力但返回 405"，而是"这个路由设计上就不提供读取能力，loader 只是满足 API 形状的占位符"。

### 4.3 B 路由 loader 支持 HEAD 但拒绝 GET 的语义

**文件**：`$target/$streamId.ts:199-201`

```typescript
if (request.method !== "HEAD") {
  return new Response("Only HEAD requests are allowed for this endpoint", { status: 405 });
}
```

**设计意图**：
- B 路由的 loader 专门用于断点续传的进度查询（HEAD）
- Run 流的 SSE 读取统一走 A 路由（兼容路由）
- 这样设计的好处：A 路由处理 CORS + JWT 的公开读取，B 路由处理需要认证的写入操作
- 分工明确：A 负责公开读取，B 负责认证写入 + 续传查询

### 4.4 F 路由（records）纯读的铁证

| 证据 | 位置 | 说明 |
|------|------|------|
| 只导出 loader，不导出 action | `sessions.$session.$io.records.ts:39` | Remix 中只有 loader 不可能写入 |
| loader 来自 `createLoaderApiRoute` | `sessions.$session.$io.records.ts:39` | 不生成 action |
| authorization action = `"read"` | `sessions.$session.$io.records.ts:59` | RBAC 层面限制为只读 |
| 唯一调用 `readSessionStreamRecords()` | `sessions.$session.$io.records.ts:90-94` | 方法语义为读 |
| 不读取请求体 | 全文件 | 无 `request.text()` / `request.json()` |
| 无 DB 写入 | 全文件 | 无 Prisma 写操作 |
| 无 S2 写入 | 全文件 | 无 `s2Append` |

---

## 五、代码引用速查

### 5.1 createActionApiRoute 核心行为

| 关注点 | 文件 | 行号 |
|--------|------|------|
| loader 硬编码 405 | `apiBuilder.server.ts` | 730-736 |
| action method 过滤 | `apiBuilder.server.ts` | 739-750 |
| createLoaderApiRoute 定义 | `apiBuilder.server.ts` | 232-390 |

### 5.2 各路由导出与生成来源

| 路由 | action 生成来源 | loader 生成来源 | 行号 |
|------|-----------------|-----------------|------|
| A `$runId/$streamId` | 独立函数 | `createLoaderApiRoute` | 18, 84 |
| B `$target/$streamId` | `createActionApiRoute` | `createLoaderApiRoute` | 16, 149 |
| C `$target/$streamId.append` | `createActionApiRoute` | 不导出 | 23, 145 |
| D `sessions/$io` | `createActionApiRoute` | `createLoaderApiRoute` | 26, 97 |
| E `sessions/$io.append` | `createActionApiRoute` | `createActionApiRoute` (占位) | 37, 194 |
| F `sessions/$io.records` | 不导出 | `createLoaderApiRoute` | 39 |
| G `input/$streamId` | `createActionApiRoute` | `createLoaderApiRoute` | 27, 108 |

### 5.3 关键验证点

| 验证点 | 文件 | 行号 |
|--------|------|------|
| B 路由 loader 只允许 HEAD | `$target/$streamId.ts` | 199-201 |
| E 路由 action method: "POST" | `sessions/$io.append.ts` | 40 |
| F 路由 authorization action: "read" | `sessions/$io.records.ts` | 59 |
| C 路由只导出 action | `$target/$streamId.append.ts` | 145 |
