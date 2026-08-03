---
title: Signal Evaluation
type: entity
entity_type: concept
tags: [信号评估, 分类, 路由]
---

# Signal Evaluation

ExtProc 的信号评估系统，分析用户请求内容并生成多维信号，供 [[Decision Engine]] 做路由决策。

## 11 种信号类型

1. **Keyword** — 关键词规则匹配
2. **Embedding** — 语义嵌入相似度
3. **Domain** — 域分类（coding/math/creative 等）
4. **Fact-check** — 事实核查需求评估
5. **User Feedback** — 用户反馈信号
6. **Preference** — 用户偏好
7. **Language** — 语言检测
8. **Context** — 上下文信号（token 计数等）
9. **Complexity** — 复杂度评估
10. **Modality** — 模态分类（AR/DIFFUSION/BOTH）
11. **Authz** — 授权评估

## "宽进窄出"设计

一次性评估全部 11 种信号，但消费者不同：

- **Decision 规则条件**（7 种）：Keyword, Embedding, Domain, Complexity, Language, Context, Authz
- **后续步骤消费**（4 种）：Fact-check → [[幻觉检测]]，Modality → 路由执行，User Feedback/Preference → 辅助

为什么不拆开做：共享 Embedding 向量计算，拆开反而更贵。

## Modality 信号的特殊性

Modality 横跨两个阶段：信号检测参与 Decision 规则匹配，但路由执行（调 diffusion 模型）在 Decision 之后。

## 代码位置

`req_filter_classification.go:75` — `r.Classifier.EvaluateAllSignalsWithHeaders()`

## 相关页面

- [[Decision Engine]] | [[handleRequestBody]] | [[幻觉检测]]
