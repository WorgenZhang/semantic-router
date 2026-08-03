---
title: Decision Engine
type: entity
entity_type: concept
tags: [路由, 信号评估, 模型选择, 核心组件]
---

# Decision Engine

ExtProc 请求体阶段的核心路由决策引擎。通过信号评估 → Decision 规则匹配 → 模型选择三步完成智能路由。

## 工作流程

1. **信号评估**（[[Signal Evaluation]]）：评估 11 种信号
2. **Decision 匹配**：根据 AND/OR 组合规则匹配最佳 Decision
3. **模型选择**：使用 ModelSelector Registry 根据算法选择模型

## 信号类型

| 信号 | 说明 | 存储字段 |
|------|------|---------|
| Keyword | 关键词匹配 | ctx.VSRMatchedKeywords |
| Embedding | 语义嵌入相似度 | ctx.VSRMatchedEmbeddings |
| Domain | 域分类 | ctx.VSRMatchedDomains |
| Fact-check | 事实核查需求 | ctx.VSRMatchedFactCheck |
| User Feedback | 用户反馈 | ctx.VSRMatchedUserFeedback |
| Preference | 偏好 | ctx.VSRMatchedPreference |
| Language | 语言检测 | ctx.VSRMatchedLanguage |
| Context | 上下文/token 计数 | ctx.VSRMatchedContext |
| Complexity | 复杂度评估 | ctx.VSRMatchedComplexity |
| Modality | 模态分类 | ctx.VSRMatchedModality |
| Authz | 授权评估 | ctx.VSRMatchedAuthz |

## 模型选择算法

通过 [[ModelSelector-Registry]] 执行，支持 9 种算法：

- 在线方法：[[Static选择算法]]、[[Elo评分系统]]、[[RouterDC]]、[[AutoMix]]、[[Hybrid混合加权]]
- ML 方法：[[KNN选择算法]]、[[KMeans选择算法]]、[[SVM选择算法]]、[[MLP选择算法]]

详见 [[模型选择算法]] 主题页。

## 设计决策：为什么安全检查在 Decision 之后

Jailbreak/PII/缓存/RAG/Memory 都放在 Decision 评估之后，因为这些检查是 **per-Decision 配置**的：
- 不同 Decision 有不同 Jailbreak 阈值
- 不同 Decision 可以启用/禁用 PII 插件
- 不同 Decision 有不同缓存相似度阈值和 TTL

Decision 是配置的 pivot point — 没有它就不知道该用什么策略。

## 信号消费分类

| 消费时机 | 信号 | 说明 |
|---------|------|------|
| Decision 规则条件 | Keyword, Embedding, Domain, Complexity, Language, Context, Authz | 必须提前获取 |
| 后续步骤按需消费 | Fact-check, Modality, User Feedback, Preference | 理论上可延后 |

一次性全部评估的原因：共享 Embedding 计算、架构简洁、未来可将任何信号升级为规则条件。

## 代码位置

- `req_filter_classification.go:21-100+` — performDecisionEvaluation()
- `req_filter_classification.go:75` — EvaluateAllSignalsWithHeaders()

## 相关页面

- [[Signal Evaluation]] | [[handleRequestBody]]
- [[ExtProc]] | [[Jailbreak 检测]]
