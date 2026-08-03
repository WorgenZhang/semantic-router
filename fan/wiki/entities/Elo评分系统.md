---
title: Elo 评分系统
type: entity
entity_type: concept
tags: [模型选择, 在线学习, 用户反馈]
created: 2026-05-18
updated: 2026-05-18
sources: [2026-05-18-ModelSelection算法类型及举例]
---

# Elo 评分系统

论文：RouteLLM: Learning to Route LLMs（arXiv:2406.18665）。

借鉴国际象棋 Elo 等级分，通过用户反馈的成对比较（A 比 B 好）动态更新模型评分。基于 Bradley-Terry 模型。

## 核心公式

**期望胜率：**
```
E_A = 1 / (1 + 10^((R_B - R_A) / 400))
```

**Rating 更新（K=32）：**
```
R_A' = R_A + K × (S_A - E_A)
```
- S_A = 实际结果（赢=1，平=0.5，输=0）

**推理时选择：**
```
P(model_i) = 10^(R_i / 400) / Σ_j 10^(R_j / 400)
```

## 特点

- 初始 Rating = 1500
- 支持按类别独立评分
- 自适应学习，但需要成对比较反馈
- 不考虑 query 语义

## 相关页面

- [[ModelSelector-Registry]] — 所属注册表
- [[Hybrid混合加权]] — 作为 Hybrid 的子组件之一
- [[模型选择算法]] — 所属主题
