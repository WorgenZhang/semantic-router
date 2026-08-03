---
title: RequestContext
type: entity
entity_type: concept
tags: [数据结构, 请求生命周期, 状态管理]
---

# RequestContext

贯穿请求完整生命周期的上下文结构体，在 Process() 循环开始时创建，存储所有中间状态。

## 定义位置

`processor_req_header.go:39-151`

## 关键字段分组

**基础信息**：StartTime, ProcessingStartTime, RequestID, RequestModel, Headers

**路由决策**：VSRSelectedModel, VSRSelectionMethod, VSRSelectedDecisionName, VSRSelectedDecisionConfidence

**流式状态**：ExpectStreamingResponse, IsStreamingResponse, StreamingComplete, StreamingAborted, StreamingChunks, StreamingContent

**安全检查**：JailbreakDetected, PIIBlocked, FactCheckNeeded, HallucinationDetected

**追踪**：TraceContext, UpstreamSpan, RouterReplayID

**内容**：OriginalRequestBody, UserContent, RAGRetrievedContext, MemoryContext

## 生命周期

```
handleRequestHeaders: 创建 ctx, 填充 Headers/RequestID/TraceContext/ExpectStreamingResponse
handleRequestBody:    填充路由决策/安全检查结果/缓存状态
handleResponseHeaders: 更新 IsStreamingResponse, 结束 UpstreamSpan
handleResponseBody:    更新 StreamingComplete/StreamingAborted, 记录指标
```

## 相关页面

- [[ExtProc]] | [[handleRequestHeaders]] | [[handleRequestBody]]
