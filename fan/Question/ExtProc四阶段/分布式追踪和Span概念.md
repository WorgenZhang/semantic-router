## 分布式追踪 & Span 概念

### 类比：快递追踪系统

你在淘宝买了个东西，物流追踪页面会显示：

```
📦 订单 #12345 (= Trace)
  ├── 09:00 卖家发货 → 杭州仓库         (= Span 1)
  ├── 10:30 杭州仓库 → 上海中转站        (= Span 2)  
  ├── 14:00 上海中转站 → 北京分拣中心     (= Span 3)
  └── 18:00 北京分拣中心 → 你家门口       (= Span 4)
```

- **Trace** = 一个完整请求的全程追踪（一个快递单号）
- **Span** = 追踪中的一个阶段/步骤（快递在两站之间的一段旅程）
- **Trace ID** = 贯穿所有服务的唯一 ID（快递单号）
- **Parent Span** = 上一级步骤（每一站知道自己从哪来）

---

### 在本项目中的时序图

```
Client                  Envoy                   ExtProc                  vLLM
  │                       │                       │                       │
  │── POST /v1/chat ─────>│                       │                       │
  │   (带 traceparent     │                       │                       │
  │    header 或不带)      │                       │                       │
  │                       │                       │                       │
  │                       │── ProcessingRequest ─>│                       │
  │                       │                       │                       │
  │                       │               ┌───────┴────────┐              │
  │                       │               │ ExtractTraceContext            │
  │                       │               │ 从 header 中取出              │
  │                       │               │ traceparent, 还原             │
  │                       │               │ Trace ID (或新建一个)          │
  │                       │               ├────────────────┤              │
  │                       │               │                │              │
  │                       │               │  Span 1: "request_received"   │
  │                       │               │  ├─ start: 09:00:00.000       │
  │                       │               │  ├─ attributes:               │
  │                       │               │  │   request_id: "abc-123"    │
  │                       │               │  │   http.method: "POST"      │
  │                       │               │  │   http.path: "/v1/chat"    │
  │                       │               │  └─ end: 09:00:00.005         │
  │                       │               │                │              │
  │                       │               │  Span 2: "semantic_classify"  │
  │                       │               │  ├─ parent: Span 1            │
  │                       │               │  ├─ start: 09:00:00.005       │
  │                       │               │  ├─ attributes:               │
  │                       │               │  │   model: "llama-3"         │
  │                       │               │  │   category: "coding"       │
  │                       │               │  └─ end: 09:00:00.050         │
  │                       │               │                │              │
  │                       │               │  Span 3: "upstream_request"   │
  │                       │               │  ├─ parent: Span 1            │
  │                       │               │  ├─ start: 09:00:00.051       │
  │                       │               └───────┬────────┘              │
  │                       │                       │                       │
  │                       │<─ ProcessingResponse ─│                       │
  │                       │                       │                       │
  │                       │── HTTP POST ──────────────────────────────────>│
  │                       │   (header 带 traceparent:                     │
  │                       │    trace_id=aaa, parent_span=span3)           │
  │                       │                       │                       │
  │                       │<───────────────────── Response ───────────────│
  │                       │                       │                       │
  │                       │── ProcessingRequest ─>│                       │
  │                       │               ┌───────┴────────┐              │
  │                       │               │  Span 3 end: 09:00:01.200     │
  │                       │               │  (upstream 耗时 1.149s)        │
  │                       │               │                               │
  │                       │               │  Span 4: "response_process"   │
  │                       │               │  ├─ hallucination_check       │
  │                       │               │  └─ cache_store               │
  │                       │               └───────┬────────┘              │
  │                       │<─ ProcessingResponse ─│                       │
  │                       │                       │                       │
  │<── HTTP Response ────│                       │                       │
```

---

### 最终在 Jaeger/Grafana 里看到的效果

```
Trace ID: aaa-bbb-ccc-ddd               总耗时: 1.250s
│
├── Span 1: request_received ─────────────────────────────── 0ms ~ 1250ms (整体)
│   ├── Span 2: semantic_classify ──────── 5ms ~ 50ms (45ms)
│   ├── Span 3: upstream_request ───────── 51ms ~ 1200ms (1149ms) ← 最耗时！
│   └── Span 4: response_process ───────── 1200ms ~ 1250ms (50ms)
```

看到这张图，你立刻就知道：
- **总耗时 1.25s**，其中 **91.9% 的时间花在 upstream（vLLM 推理）**
- 语义分类只用了 45ms，响应处理 50ms
- 如果要优化延迟，该优化 vLLM 而不是 ExtProc

---

### 回到源码中的三行关键代码

```go
// 第 170 行：从上游传来的 header 中恢复 Trace ID（如果没有就新建）
ctx.TraceContext = tracing.ExtractTraceContext(baseCtx, headerMap)

// 第 173 行：在这个 Trace 下创建一个新 Span，命名为 "request_received"
spanCtx, span := tracing.StartSpan(ctx.TraceContext, tracing.SpanRequestReceived)

// 第 211 行：给这个 Span 打标签，方便查询
tracing.SetSpanAttributes(span, attribute.String("http.method", method))
```

**一句话总结**：Trace 是一个请求的完整旅程，Span 是旅程中的一段路。通过 Trace ID 把分散在不同服务里的 Span 串起来，就能看到整个请求在哪里慢、在哪里出错。
