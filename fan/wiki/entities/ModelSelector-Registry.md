---
title: ModelSelector Registry
type: entity
entity_type: concept
tags: [模型选择, 路由, 核心组件]
created: 2026-05-18
updated: 2026-05-18
sources: [2026-05-18-ModelSelection算法类型及举例]
---

# ModelSelector Registry

[[Decision-Engine]] 的下游组件，负责从 Decision 的 model_pool 中根据算法选择最优模型。

## 算法注册表

支持 9 种选择算法，分两大类：

### 在线方法（可动态学习）

| 算法 | 核心思路 |
|------|----------|
| [[Static选择算法]] | 固定分数，选最高 |
| [[Elo评分系统]] | Bradley-Terry 模型，用户 A/B 反馈动态调整 |
| [[RouterDC]] | query-model embedding 余弦相似度 |
| [[AutoMix]] | POMDP 级联，小模型优先 + 自我验证升级 |
| [[Hybrid混合加权]] | 融合 Elo + RouterDC + AutoMix 加权分数 |

### ML 方法（离线训练 → 在线推理）

| 算法 | 核心思路 | 推理框架 |
|------|----------|----------|
| [[KNN选择算法]] | K 近邻质量加权投票 | [[Linfa]] |
| [[KMeans选择算法]] | 聚类 + 簇-模型绑定 | [[Linfa]] |
| [[SVM选择算法]] | RBF 核决策边界 + One-vs-One 投票 | [[Linfa]] |
| [[MLP选择算法]] | 神经网络 Softmax 输出 | [[Candle]] (GPU) |

## 技术栈

- 在线方法：Go 运行时直接执行
- ML 方法：Python 训练 → Rust 推理（[[Linfa]] / [[Candle]]）

## 相关页面

- [[Decision-Engine]] — 上游决策引擎
- [[模型路由机制]] — 所属主题
- [[模型选择算法]] — 算法详解主题
