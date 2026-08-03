Router Replay是什么意思 帮我举个例子

## Step 7 — Router Replay API 是什么？

**Router Replay 是一个"请求录像回放"系统**。它会把每次路由决策的完整过程记录下来（像录像机一样），然后提供一个 API 让你事后查询这些"录像"。

### 它记录了什么？

每条 Replay 记录（`RoutingRecord`）包含：

| 字段                       | 说明                                        |
| -------------------------- | ------------------------------------------- |
| `decision`               | 匹配到的决策名（如 `"coding"`)            |
| `category`               | 域分类结果（如 `"STEM"`)                  |
| `original_model`         | 用户请求的模型（如 `"auto"`)              |
| `selected_model`         | 路由后实际选择的模型（如 `"llama3-70b"`)  |
| `reasoning_mode`         | 推理模式 on/off                             |
| `signals`                | 匹配到的信号（keyword/embedding/domain 等） |
| `request_body`           | 原始请求体（可选捕获）                      |
| `response_body`          | 响应体（可选捕获）                          |
| `response_status`        | HTTP 状态码                                 |
| `jailbreak_detected`     | 是否检测到越狱攻击                          |
| `pii_detected`           | 是否检测到 PII                              |
| `hallucination_detected` | 是否检测到幻觉                              |

### Step 7 具体在干什么？

```go
if replayResp := r.handleRouterReplayAPI(method, path); replayResp != nil {
    return replayResp, nil  // 短路！直接返回，不进入 body 阶段
}
```

它检查当前请求是不是在**查询** replay 记录。如果是，直接返回结果，不需要继续处理 body。

### 具体例子

**例子 1：列出所有 replay 记录**

```http
GET /v1/router_replay HTTP/1.1
Host: my-router.example.com:8801
```

`handleRouterReplayAPI` 匹配到 `path == "/v1/router_replay"` 且 `method == "GET"`，直接返回：

```json
{
  "object": "router_replay.list",
  "count": 3,
  "data": [
    {
      "id": "rr-001",
      "request_id": "req-abc-123",
      "decision": "coding",
      "category": "STEM",
      "original_model": "auto",
      "selected_model": "deepseek-coder-v2",
      "reasoning_mode": "off",
      "response_status": 200,
      "streaming": true,
      "signals": { "keyword": ["排序", "算法"], "embedding": [...] },
      "request_body": "{\"model\":\"auto\",\"messages\":[{\"role\":\"user\",\"content\":\"写一个快速排序\"}]}",
      "timestamp": "2026-04-28T10:30:00Z"
    },
    {
      "id": "rr-002",
      "request_id": "req-def-456",
      "decision": "general_chat",
      "original_model": "auto",
      "selected_model": "llama3-8b",
      "jailbreak_detected": true,
      "jailbreak_type": "prompt_injection",
      "response_status": 403,
      "timestamp": "2026-04-28T10:31:00Z"
    },
    ...
  ]
}
```

这个请求在 header 阶段就**短路返回**了，Envoy 收到 `ImmediateResponse`（而非 `CONTINUE`），不会再发 body。

**例子 2：查询单条 replay 记录**

```http
GET /v1/router_replay/rr-001 HTTP/1.1
```

匹配到 `strings.HasPrefix(path, "/v1/router_replay/")`，提取 `replayID = "rr-001"`，在所有 recorder 里查找，找到后返回该条记录的完整 JSON。

**例子 3：正常的聊天请求（你文档中的例子）**

```http
POST /v1/chat/completions HTTP/1.1
```

`method="POST"`, `path="/v1/chat/completions"` — 既不匹配 `/v1/router_replay` 也不匹配 `/v1/router_replay/{id}`，函数返回 `nil`，不短路，继续执行 Step 8+。

### 整个 Replay 的生命周期

```
请求进来 ──────────────────────────────────────────────────────────────────────
                                                                            
  Step 7: GET /v1/router_replay?  ──是──> 直接返回录像列表（短路）            
                │                                                           
                否（正常请求）                                                
                ↓                                                           
  Body 阶段: 路由决策完成后 ──> startRouterReplay() 创建一条新记录            
                │               记录: decision, model, signals, request_body  
                ↓                                                           
  Response 阶段 ──> updateRouterReplayStatus() 更新状态码                    
                ──> attachRouterReplayResponse() 附加响应体                  
                ──> updateRouterReplayHallucinationStatus() 附加幻觉检测结果   
                                                                            
  录像完成，存在内存/Redis/Postgres 中，等待后续 GET 查询                     
```

**简单类比**：Router Replay 就像飞机的**黑匣子**。每次路由决策（航班）都会被记录，事后你可以通过 `GET /v1/router_replay` 这个 API 调出所有"飞行记录"来分析路由器的行为——它为什么选了这个模型、匹配了哪些信号、是否触发了安全检测等。Step 7 就是检查"当前请求是不是来查黑匣子的"，如果是就直接返回数据，不走正常的路由流程。
