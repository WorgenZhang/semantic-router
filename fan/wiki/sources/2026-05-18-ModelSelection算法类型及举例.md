---
title: ModelSelection 算法类型及举例
type: source
tags: [模型选择, 算法, Elo, RouterDC, AutoMix, KNN, SVM, MLP, KMeans]
created: 2026-05-18
updated: 2026-05-18
sources: []
---

# ModelSelection 算法类型及举例

> 详解 ModelSelector Registry 的 9 种模型选择算法工作原理、核心公式和实际应用示例

## 基本信息

- 类型：技术文档 / 算法解析
- 主题：模型选择算法全景

## 核心观点

1. **9 种算法分为两大类**：在线方法（Static、Elo、RouterDC、AutoMix、Hybrid）和 ML 方法（KNN、KMeans、SVM、MLP）
2. **在线方法可动态学习**，适合有持续反馈的场景；ML 方法需离线训练但推理更快
3. **技术栈分工**：ML 方法用 Python 训练，通过 [[Linfa]]（Rust）和 [[Candle]]（Rust GPU）做生产推理
4. **Hybrid 是最鲁棒的**：融合 Elo + RouterDC + AutoMix 三者加权分数，但计算开销最大
5. **AutoMix 最具创新性**：基于 POMDP 的级联路由，先小模型后大模型，优化成本-质量权衡

## 算法对比总览

| 算法 | 需要训练数据 | 使用 Embedding | 在线学习 | 推理延迟 | 适用场景 |
|------|:---:|:---:|:---:|:---:|------|
| [[Static选择算法]] | 否 | 否 | 否 | 极低 | 冷启动、规则明确 |
| [[Elo评分系统]] | 需反馈 | 否 | 是 | 低 | 有持续用户反馈 |
| [[RouterDC]] | 否(需描述) | 是 | 可选 | 中 | 模型能力差异明确 |
| [[AutoMix]] | 可选 | 否 | 是 | 高(级联) | 成本敏感场景 |
| [[Hybrid混合加权]] | 继承各组件 | 是 | 是 | 高 | 需要高鲁棒性 |
| [[KNN选择算法]] | 是 | 是 | 增量 | 中 | 历史数据丰富 |
| [[KMeans选择算法]] | 是 | 是 | 否 | 低 | 查询模式有明显聚类 |
| [[SVM选择算法]] | 是 | 是 | 否 | 低 | 决策边界清晰 |
| [[MLP选择算法]] | 是(大量) | 是 | 否 | 极低(GPU) | 数据充足、追求精度 |

## 架构关系

- 在线方法：直接在 Go 运行时执行
- ML 方法：Python 训练 → [[Linfa]](Rust) / [[Candle]](GPU) 推理

## 参考论文

| 算法 | 论文 | 链接 |
|------|------|------|
| Elo | RouteLLM: Learning to Route LLMs | arXiv:2406.18665 |
| RouterDC | Query-Based Router by Dual Contrastive Learning | arXiv:2409.19886 |
| AutoMix | Automatically Mixing Language Models | arXiv:2310.12963 |
| Hybrid | Hybrid LLM: Cost-Efficient Quality-Aware Query Routing | arXiv:2404.14618 |
| KNN/SVM/MLP | FusionFactory | arXiv:2507.10540 |
| KMeans | Avengers-Pro | arXiv:2508.12631 |

## 相关页面

- [[Decision-Engine]] — 上游决策引擎
- [[模型路由机制]] — 所属主题
- [[ModelSelector-Registry]] — 选择器注册表
- [[2026-05-18-AutoMix算法公式解释]] — AutoMix 公式详解
