Step 10：Response API 检查是什么意思 帮我举个例子

## Step 10 — Response API 检查是什么？

**Response API 是 OpenAI 在 2025 年推出的新一代对话接口（`/v1/responses`）**，用来替代经典的 `/v1/chat/completions`。它最大的特点是**内置了对话历史管理**——你不需要每次都把全部 messages 传进去，只需要传一个 `previous_response_id` 就能接着上一轮对话继续。

我们的 Semantic Router 通过 `ResponseAPIFilter` 做了一层**协议翻译**：
- 外部客户端可以用 Response API 格式发请求
- Router 内部把它翻译成 `/v1/chat/completions` 格式发给后端模型
- 把后端的 Chat Completions 响应再翻译回 Response API 格式返回给客户端

### Step 10 具体代码

```go
if r.ResponseAPIFilter != nil && r.ResponseAPIFilter.IsEnabled() && strings.HasPrefix(path, "/v1/responses") {
    // GET /v1/responses/{id}/input_items - 获取输入项
    if method == "GET" && strings.HasSuffix(path, "/input_items") {
        responseID := extractResponseIDFromInputItemsPath(path)
        return r.ResponseAPIFilter.HandleGetInputItems(ctx.TraceContext, responseID)
    }

    // GET /v1/responses/{id} - 获取一个响应
    if method == "GET" {
        responseID := extractResponseIDFromPath(path)
        return r.ResponseAPIFilter.HandleGetResponse(ctx.TraceContext, responseID)
    }

    // DELETE /v1/responses/{id} - 删除一个响应
    if method == "DELETE" {
        responseID := extractResponseIDFromPath(path)
        return r.ResponseAPIFilter.HandleDeleteResponse(ctx.TraceContext, responseID)
    }

    // POST /v1/responses - 创建新响应（标记，等 body 阶段处理）
    if method == "POST" {
        ctx.ResponseAPICtx = &ResponseAPIContext{IsResponseAPIRequest: true}
    }
}
```

**关键逻辑**：
- `GET` 和 `DELETE` 请求 → **短路返回**（和 Step 7 的 Router Replay、Step 9 的 GET /v1/models 一样）
- `POST` 请求 → **不短路**，只做标记，等 body 阶段再翻译和处理

---

### 具体例子

#### 例子 1：POST /v1/responses — 创建新响应（不短路）

客户端发送：
```http
POST /v1/responses HTTP/1.1
Content-Type: application/json

{
  "model": "gpt-4o",
  "input": [
    {"role": "user", "content": "北京今天天气怎么样？"}
  ],
  "instructions": "你是一个天气助手，用简洁的语言回答。"
}
```

**Step 10 处理**：
- `path = "/v1/responses"` → 前缀匹配 ✓
- `method = "POST"` → 不走 GET/DELETE 分支
- 执行 `ctx.ResponseAPICtx = &ResponseAPIContext{IsResponseAPIRequest: true}`
- **不返回，继续往下走** Step 11（CONTINUE）

**后续 Body 阶段会做什么**：
1. 解析 body，识别出这是 Response API 请求
2. 调用 `TranslateRequest()`，将其翻译成 Chat Completions 格式：
   ```json
   {
     "model": "gpt-4o",
     "messages": [
       {"role": "system", "content": "你是一个天气助手，用简洁的语言回答。"},
       {"role": "user", "content": "北京今天天气怎么样？"}
     ]
   }
   ```
3. 发给后端模型
4. 收到响应后，用 `TranslateResponse()` 翻译回 Response API 格式：
   ```json
   {
     "id": "resp_abc123",
     "object": "response",
     "model": "gpt-4o",
     "status": "completed",
     "output": [
       {
         "type": "message",
         "role": "assistant",
         "content": [{"type": "output_text", "text": "北京今天晴，气温25°C，微风。"}]
       }
     ],
     "conversation_id": "conv_xyz789"
   }
   ```
5. 将响应存入 ResponseStore（供后续 `previous_response_id` 查询）

---

#### 例子 2：带 previous_response_id 的多轮对话（不短路）

```http
POST /v1/responses HTTP/1.1
Content-Type: application/json

{
  "model": "gpt-4o",
  "input": [
    {"role": "user", "content": "那明天呢？"}
  ],
  "previous_response_id": "resp_abc123"
}
```

**Step 10 处理**：同上，标记 `ctx.ResponseAPICtx`

**Body 阶段**：
1. 发现 `previous_response_id = "resp_abc123"`
2. 从 ResponseStore 取出对话历史链（上一轮的 input + output）
3. 拼接成完整的 messages：
   ```json
   {
     "model": "gpt-4o",
     "messages": [
       {"role": "system", "content": "你是一个天气助手，用简洁的语言回答。"},
       {"role": "user", "content": "北京今天天气怎么样？"},
       {"role": "assistant", "content": "北京今天晴，气温25°C，微风。"},
       {"role": "user", "content": "那明天呢？"}
     ]
   }
   ```
4. 发给后端 → 翻译响应 → 存储 → 返回

**客户端不需要管理对话历史，只传一个 ID 就行！**

---

#### 例子 3：GET /v1/responses/{id} — 查询已有响应（短路）

```http
GET /v1/responses/resp_abc123 HTTP/1.1
```

**Step 10 处理**：
- `path = "/v1/responses/resp_abc123"` → 前缀匹配 ✓
- `method = "GET"` → 进入 GET 分支
- `extractResponseIDFromPath(path)` → 提取出 `"resp_abc123"`
- 调用 `HandleGetResponse(ctx, "resp_abc123")`
- 从 ResponseStore 中取出该响应，**直接返回 `ImmediateResponse`**（短路！）

返回给客户端：
```json
{
  "id": "resp_abc123",
  "object": "response",
  "model": "gpt-4o",
  "status": "completed",
  "output": [...],
  "previous_response_id": null,
  "conversation_id": "conv_xyz789"
}
```

---

#### 例子 4：GET /v1/responses/{id}/input_items — 查询输入项（短路）

```http
GET /v1/responses/resp_abc123/input_items HTTP/1.1
```

**Step 10 处理**：
- `path` 以 `/input_items` 结尾 → 进入 input_items 分支
- 提取 `responseID = "resp_abc123"`
- 调用 `HandleGetInputItems(ctx, "resp_abc123")`
- **短路返回**：

```json
{
  "object": "list",
  "data": [
    {"id": "item_001", "type": "message", "role": "system", "content": [{"type": "input_text", "text": "你是一个天气助手..."}], "status": "completed"},
    {"id": "item_002", "type": "message", "role": "user", "content": [{"type": "input_text", "text": "北京今天天气怎么样？"}], "status": "completed"}
  ],
  "first_id": "item_001",
  "last_id": "item_002",
  "has_more": false
}
```

---

#### 例子 5：DELETE /v1/responses/{id} — 删除响应（短路）

```http
DELETE /v1/responses/resp_abc123 HTTP/1.1
```

**Step 10 处理**：
- `method = "DELETE"` → 进入 DELETE 分支
- 从 ResponseStore 中删除该响应
- **短路返回**：

```json
{
  "id": "resp_abc123",
  "object": "response.deleted",
  "deleted": true
}
```

---

#### 例子 6：正常的 Chat Completions 请求（跳过）

```http
POST /v1/chat/completions HTTP/1.1
```

`path = "/v1/chat/completions"` → `strings.HasPrefix(path, "/v1/responses")` 为 `false` → **整个 if 块跳过**，继续走 Step 11。

---

### 总结对比

| 请求 | Step 10 行为 | 原因 |
|------|-------------|------|
| `POST /v1/responses` | 标记 `ctx.ResponseAPICtx`，不短路 | 需要读 body 才能翻译 |
| `GET /v1/responses/{id}` | 短路返回存储的响应 | 不需要 body，直接从 store 读 |
| `GET /v1/responses/{id}/input_items` | 短路返回输入项列表 | 同上 |
| `DELETE /v1/responses/{id}` | 短路返回删除确认 | 同上 |
| `POST /v1/chat/completions` | 完全跳过 | 路径不匹配 |

### Response API 的架构位置

```
客户端 (Response API 格式)
    │
    │  POST /v1/responses
    │  {"input": [...], "previous_response_id": "resp_xxx"}
    ↓
┌─────────────────────────────────────────────┐
│  Semantic Router (ExtProc)                  │
│                                             │
│  Header 阶段: 标记 IsResponseAPIRequest     │
│  Body 阶段:                                 │
│    1. TranslateRequest() → Chat格式         │
│    2. 从 ResponseStore 拉对话历史            │
│    3. 正常路由决策（选模型、选 endpoint）     │
│    4. 发给后端                              │
│  Response 阶段:                             │
│    5. TranslateResponse() → Response API格式│
│    6. StoreResponse() → 存入 ResponseStore  │
└─────────────────────────────────────────────┘
    │
    │  翻译后: POST /v1/chat/completions
    │  {"messages": [...]}
    ↓
后端模型 (vLLM/OpenAI)
```

**一句话总结**：Step 10 让 Router 同时支持两种 API 风格——经典的 Chat Completions 和新的 Response API。对于 GET/DELETE 操作，header 阶段直接短路处理；对于 POST 创建请求，只打个标记，等 body 阶段做协议翻译。
