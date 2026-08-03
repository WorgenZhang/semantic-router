## `DecrementModelActiveRequests` 与负载均衡

### 什么是 active_requests

系统为每个模型维护一个实时计数器：

```
IncrementModelActiveRequests("llama-3-70b")  ← 请求进入时 +1
DecrementModelActiveRequests("llama-3-70b")  ← 响应完成时 -1（defer 保证即使出错也会执行）
```

这个计数器就是该模型**当前正在处理中、还没返回的请求数**。

### 具体场景

假设你有 3 个 vLLM 实例部署了同一个模型：

```
模型池: llama-3-70b
  ├── 实例 A (10.0.1.1:8000)  active_requests = 12
  ├── 实例 B (10.0.1.2:8000)  active_requests = 3   ← 最空闲
  └── 实例 C (10.0.1.3:8000)  active_requests = 8
```

### 怎么用于负载均衡

当 `handleRequestBody` 中的 `selectEndpointForModel()` 需要选择端点时：

```
新请求来了，模型 = llama-3-70b

负载均衡策略 = "least_connections" (最少连接数):
  → 选实例 B（active_requests = 3，最少）

负载均衡策略 = "latency_aware" (延迟感知):
  → 综合考虑 active_requests + 历史 TPOT
  → 实例 B 虽然空闲，但 TPOT 偏高（GPU 慢）
  → 可能选实例 C（active=8 但每个 token 更快）
```

### 怎么用于队列深度估计

vLLM 自身有推理队列上限（比如 `max_num_seqs=256`）。如果某个实例 active_requests 接近这个阈值：

```
实例 A: active_requests = 250 / max_num_seqs = 256
  → 队列深度 ≈ 97.6%，接近饱和
  → 路由策略应该避免发更多请求过去
  → 否则 vLLM 会排队等待，TTFT 急剧上升
```

### 为什么用 defer

```go
defer metrics.DecrementModelActiveRequests(ctx.RequestModel)
```

`defer` 保证**无论函数怎么退出**（正常返回、幻觉检测 block、panic 恢复），计数器都会 -1。否则如果某个分支 return 时忘了减，active_requests 会虚高，导致路由策略误判该实例"很忙"。

### 一句话总结

active_requests 是实时"在途请求数"，路由器据此把新请求分配给最不繁忙的实例，避免单实例过载。`defer` 保证计数永远准确。
