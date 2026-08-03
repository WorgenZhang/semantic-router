---
title: Hybrid 混合加权
type: entity
entity_type: concept
tags: [模型选择, 集成方法, 鲁棒性]
created: 2026-05-18
updated: 2026-05-18
sources: [2026-05-18-ModelSelection算法类型及举例]
---

# Hybrid 混合加权

论文：Hybrid LLM: Cost-Efficient Quality-Aware Query Routing（arXiv:2404.14618）。

将 [[Elo评分系统]]、[[RouterDC]]、[[AutoMix]] 三种方法的分数加权融合，取各家之长。

## 核心公式

```
FinalScore(m) = (w_elo × S_elo(m) + w_dc × S_dc(m) + w_am × S_am(m)) / (w_elo + w_dc + w_am)
```

默认权重：`w_elo=0.3, w_dc=0.3, w_am=0.2, w_cost=0.2`

**成本调整：**
```
FinalScore(m) *= (1 + (1 - NormalizedCost(m)) × CostWeight × w_cost)
```

**置信度（组件一致性）：**
```
Confidence = (AgreementRatio + AvgComponentConfidence) / 2
```

## 特点

- 任一组件失效仍可工作
- 组件一致性提供置信度度量
- 计算开销最大（运行 3 个 selector）

## 相关页面

- [[Elo评分系统]] | [[RouterDC]] | [[AutoMix]] — 三个子组件
- [[ModelSelector-Registry]] — 所属注册表
- [[模型选择算法]] — 所属主题
