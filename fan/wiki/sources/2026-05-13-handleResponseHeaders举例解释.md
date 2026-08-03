---
title: handleResponseHeaders 每步举例解释
date: 2026-05-13
type: source
source_path: raw/notes/2026-05-13-handleResponseHeaders举例解释.md
tags: [handleResponseHeaders, 响应头, VSR Headers, TTFT, 流式模式]
sources: []
images: 0
image_paths: []
---

# handleResponseHeaders 每步举例解释

## 核心观点

- 从 `:status` 伪头提取上游 HTTP 状态码，检测 content-type 判断是否流式
- 结束在请求体阶段启动的 UpstreamSpan，记录上游请求耗时
- 非流式响应在此阶段记录 TTFT（Time To First Token）
- 注入 20+ 个 `x-vsr-*` 响应头，提供完整的路由决策可观测性
- 流式响应设置 ModeOverride=STREAMED，让后续 body chunk 逐个转发

## 关键数据流示例

**输入**（上游返回的响应头）：
```
:status = 200
content-type = text/event-stream
```

**输出**（附加到响应的 headers）：
```
x-vsr-selected-category: code_generation
x-vsr-selected-decision: route-to-large-model
x-vsr-selected-confidence: 0.9234
x-vsr-selected-model: llama-3-70b
x-vsr-matched-keywords: implement,function
```

## 关键概念

- [[handleResponseHeaders]]：第三阶段处理器
- [[VSR Headers]]：路由决策元数据响应头
- [[分布式追踪]]：UpstreamSpan 结束
- [[流式响应 SSE]]：ModeOverride=STREAMED
