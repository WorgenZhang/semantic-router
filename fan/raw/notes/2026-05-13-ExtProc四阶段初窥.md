# processor_core.go:Process 详细分析

```go
整体架构

  这是 Envoy ExtProc gRPC 流式处理的核心事件循环，采用经典的 for { stream.Recv() → switch → handle → sendResponse } 模式。

  处理流程

  Envoy gRPC Stream
         │
         ▼
     stream.Recv()  ──── 出错? ──→ 错误处理（见下方）
         │
         ▼ (成功)
     switch req.Request.(type)
         │
         ├─ RequestHeaders  → handleRequestHeaders()  → sendResponse()
         ├─ RequestBody     → handleRequestBody()     → sendResponse()
         ├─ ResponseHeaders → handleResponseHeaders() → sendResponse()
         ├─ ResponseBody    → handleResponseBody()    → sendResponse()
         └─ default         → 构造 CONTINUE 响应       → sendResponse()

  四个处理阶段

  ┌────────┬───────────────────────┬─────────────────────────┐
  │  阶段  │        处理器         │          文件           │
  ├────────┼───────────────────────┼─────────────────────────┤
  │ 请求头 │ handleRequestHeaders  │ processor_req_header.go │
  ├────────┼───────────────────────┼─────────────────────────┤
  │ 请求体 │ handleRequestBody     │ processor_req_body.go   │
  ├────────┼───────────────────────┼─────────────────────────┤
  │ 响应头 │ handleResponseHeaders │ processor_res_header.go │
  ├────────┼───────────────────────┼─────────────────────────┤
  │ 响应体 │ handleResponseBody    │ processor_res_body.go   │
  └────────┴───────────────────────┴─────────────────────────┘

  错误处理策略（第 27-66 行）
  分为四层，优先级从高到低：
1. 流式中断检测（第 30-33 行）— 如果 streaming 响应未完成就断开，标记 StreamingAborted = true，防止缓存不完整响应
2. EOF 正常结束（第 36-39 行）— 客户端主动关闭，返回 nil
3. gRPC 状态码处理（第 42-51 行）— Canceled 静默返回，DeadlineExceeded 记录超时指标
4. Go context 取消（第 54-62 行）— 服务端 context 取消/超时的兜底处理

```
