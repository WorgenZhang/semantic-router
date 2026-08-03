---
title: ExtProc 四阶段处理流水线
type: topic
tags: [ExtProc, Envoy, gRPC, 核心架构]
---

# ExtProc 四阶段处理流水线

## 概述

ExtProc 通过 Envoy gRPC External Processor 接口，将 HTTP 请求/响应的处理拆分为四个阶段，形成一条完整的处理流水线。

## 四阶段

| 阶段 | 函数 | 职责 | 关键操作 |
|------|------|------|---------|
| 请求头 | [[handleRequestHeaders]] | 观察和记录 | 提取 trace、检测流式、短路拦截 |
| 请求体 | [[handleRequestBody]] | 核心路由 | Decision 评估、安全检查、模型选择 |
| 响应头 | [[handleResponseHeaders]] | 可观测性 | 注入 VSR headers、记录 TTFT |
| 响应体 | [[handleResponseBody]] | 后处理 | token 统计、缓存、幻觉检测 |

## 数据流

```
客户端 → Envoy → ExtProc(请求头) → ExtProc(请求体) → Envoy → 上游 vLLM
                                                                    ↓
客户端 ← Envoy ← ExtProc(响应体) ← ExtProc(响应头) ← Envoy ← 上游 vLLM
```

## 核心设计原则

1. **延迟决策**：请求头阶段不修改任何内容，等到 body 阶段才做路由决策
2. **短路优化**：每个阶段都有短路退出点（缓存命中、安全拦截等）
3. **完整可观测**：通过 VSR headers 和 Span 提供全链路追踪

## 素材来源

- [[2026-05-13-ExtProc四阶段初窥]]
- [[2026-05-13-ExtProc四阶段处理流水线详解]]
- [[2026-05-13-handleRequestHeaders举例解释]]
- [[2026-05-13-handleRequestBody举例解释]]
- [[2026-05-13-handleResponseHeaders举例解释]]
- [[2026-05-13-handleResponseBody举例解释]]

## 相关页面

- [[ExtProc]] | [[Decision Engine]] | [[RequestContext]]
- [[模型路由机制]] | [[安全防护机制]] | [[可观测性与追踪]]
