## Envoy、gRPC、External Processor 三者关系

### 概念说明

**三者的关系是：Envoy 通过 gRPC 协议与 External Processor 通信。**

| 组件 | 角色 |
|------|------|
| **Envoy** | 高性能反向代理/边车代理，处理所有入站流量 |
| **gRPC** | 通信协议，是 Envoy 和 ExtProc 之间的"电话线" |
| **External Processor (ExtProc)** | 你的自定义逻辑服务器（本项目中是 `OpenAIRouter`），对请求/响应进行检查和修改 |

### 在本项目中的具体实现

从 `config/envoy.yaml` 可以看到：

1. **Envoy** 监听 `:8801`，接收客户端 HTTP 请求
2. Envoy 通过 `ext_proc` filter 把请求的 header/body 经 **gRPC** 发送到 `127.0.0.1:50051`（ExtProc 服务）
3. **ExtProc**（`OpenAIRouter.Process()`）收到 `ProcessingRequest`，做语义路由、分类、PII检测等处理，返回 `ProcessingResponse` 告诉 Envoy 如何修改 header（比如设置 `x-selected-model`、`x-vsr-destination-endpoint`）
4. Envoy 根据修改后的 header 路由到对应的 upstream（vLLM / OpenAI / Anthropic）

### 时序图

```
Client                    Envoy (:8801)              ExtProc (:50051/gRPC)          Upstream (vLLM/OpenAI/...)
  │                            │                            │                            │
  │── HTTP POST /v1/chat ─────>│                            │                            │
  │                            │                            │                            │
  │                            │── gRPC: ProcessingRequest ─>│                            │
  │                            │   (RequestHeaders)          │                            │
  │                            │                            │ 解析 path, method, headers   │
  │                            │                            │ 检查缓存/rate limit          │
  │                            │<─ ProcessingResponse ──────│                            │
  │                            │   (CONTINUE)               │                            │
  │                            │                            │                            │
  │                            │── gRPC: ProcessingRequest ─>│                            │
  │                            │   (RequestBody)             │                            │
  │                            │                            │ 解析 JSON body              │
  │                            │                            │ 语义分类 + 模型选择           │
  │                            │                            │ PII/Jailbreak 检测          │
  │                            │                            │ 设置 x-selected-model       │
  │                            │                            │ 设置 x-vsr-destination-endpoint│
  │                            │<─ ProcessingResponse ──────│                            │
  │                            │   (HeaderMutation + Body)   │                            │
  │                            │                            │                            │
  │                            │── 根据 header 路由 ─────────────────────────────────────>│
  │                            │   HTTP POST (转发请求)       │                            │
  │                            │                            │                            │
  │                            │<─────────────────────────── HTTP Response ──────────────│
  │                            │                            │                            │
  │                            │── gRPC: ProcessingRequest ─>│                            │
  │                            │   (ResponseHeaders)         │                            │
  │                            │                            │ 记录 metrics, 注入追踪 header │
  │                            │<─ ProcessingResponse ──────│                            │
  │                            │                            │                            │
  │                            │── gRPC: ProcessingRequest ─>│                            │
  │                            │   (ResponseBody)            │                            │
  │                            │                            │ 幻觉检测 / 缓存响应          │
  │                            │<─ ProcessingResponse ──────│                            │
  │                            │                            │                            │
  │<── HTTP Response ─────────│                            │                            │
  │                            │                            │                            │
```

### 关键点

1. **gRPC 双向流**：Envoy 和 ExtProc 之间是一个 **bidirectional gRPC stream**（`Process(stream)` 方法）。一次 HTTP 请求对应一个 stream，stream 内按顺序传递 4 个阶段：RequestHeaders → RequestBody → ResponseHeaders → ResponseBody。

2. **ExtProc 不转发流量**：ExtProc 只做"旁路检查+修改指令"，真正的流量转发由 Envoy 完成。ExtProc 告诉 Envoy "把这个 header 改成 X"，Envoy 自己去执行路由。

3. **processing_mode 控制粒度**：`envoy.yaml` 里配置了 `request_body_mode: BUFFERED`，意味着 Envoy 会等 body 完整接收后再发给 ExtProc。这样 ExtProc 才能解析完整 JSON 做语义分类。

4. **failure_mode_allow: true**：如果 ExtProc 服务挂了，Envoy 会放行请求（不阻塞流量），这是一种优雅降级策略。
