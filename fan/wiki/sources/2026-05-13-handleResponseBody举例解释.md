---
title: handleResponseBody 每步举例解释
date: 2026-05-13
type: source
source_path: raw/notes/2026-05-13-handleResponseBody举例解释.md
tags: [handleResponseBody, 响应体, 第四阶段, 指标, 缓存, 幻觉检测]
sources: [processor_res_body.go]
images: 0
image_paths: []
---

# handleResponseBody 每步举例解释

> ExtProc 第四阶段（响应体处理）的完整 16 步数据流解析，每步配以具体输入输出示例。

## 核心摘要

`handleResponseBody` 是 ExtProc 流水线的最后一步，负责 LLM 响应的后处理。涵盖：延迟计算、格式转换（Anthropic→OpenAI、Response API）、流式/非流式分支、Token 指标记录、成本核算、语义缓存更新、记忆提取和幻觉检测。

## 16 步处理流程

| 步骤 | 操作 | 关键数据 |
|------|------|---------|
| 1 | 计算完成延迟 + defer 递减活跃请求 | completionLatency, active_requests-- |
| 2 | Looper 内部请求短路 | ctx.LooperRequest → 捕获响应 → CONTINUE |
| 3 | Anthropic 格式转换 | Anthropic JSON → OpenAI ChatCompletion |
| 4 | 流式响应分支（TTFT→累积 chunks→[DONE]缓存） | SSE chunks, StreamingContent |
| 5 | 解析 Token 用量 | promptTokens, completionTokens |
| 6 | Rate Limiter 上报 | TPM 预算扣减 |
| 7 | 记录指标（TPOT、窗口化） | time_per_token, windowed metrics |
| 8 | 计算成本 | pricing × tokens → USD |
| 9 | 更新语义缓存 | embedding→response 映射，TTL |
| 10 | 异步记忆提取（goroutine） | MemoryExtractor.ProcessResponse |
| 11 | Response API 格式转换 | ChatCompletion → Response API |
| 12 | 构建 Body/Header Mutation | BodyMutation, content-length 移除 |
| 13 | 幻觉检测 | NLI 模型，CONTRADICTION 判定 |
| 14 | 应用幻觉警告 | header/body/block/none |
| 15 | 更新 Router Replay 幻觉状态 | 审计记录 |
| 16 | 附加 Replay 响应 → 返回 | finalBody → Envoy |

## 两条执行路径

- **流式响应**（步骤 4 后直接 return）：逐 chunk 转发，记录 TTFT，[DONE] 时重建完整响应并缓存
- **非流式响应**（步骤 5-16）：一次性解析 → 指标 → 缓存 → 格式转换 → 幻觉检测

## 关键设计洞察

| 设计 | 原因 |
|------|------|
| 记忆提取用 goroutine | 不增加响应延迟 |
| 幻觉检测在缓存之后 | 即使检测失败也已缓存原始响应 |
| Mutation 机制修改响应 | 通过 Envoy ExtProc 协议，不重建整个 HTTP 响应 |
| 格式透明 | 客户端始终看到 OpenAI/Response API 格式，不知后端是 Anthropic 还是 vLLM |
| 指标全面覆盖 | 延迟/Token/TPOT/TTFT/成本/幻觉率 → 驱动路由决策 |

## 提取概念

- [[幻觉检测]] — NLI 模型判定 CONTRADICTION，支持 header/body/block/none 四种 action
- [[流式响应-SSE]] — TTFT 记录、chunk 累积、[DONE] 时重建缓存
- [[语义缓存]] — handleRequestBody 注册 pending，handleResponseBody 用实际响应完成条目
- [[记忆提取]] — Response API 的 auto_store 特性，异步提取对话关键信息
- [[Rate Limiter]] — TPM 预算追踪，token 用量上报

## 相关页面

- [[2026-05-13-ExtProc四阶段处理流水线详解]] — 全局视角
- [[2026-05-13-handleRequestBody举例解释]] — 第二阶段（互补：注册缓存 vs 完成缓存）
- [[ExtProc四阶段处理流水线]] — 主题总览
- [[ExtProc]] — 组件实体
