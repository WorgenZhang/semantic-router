---
title: KNN 选择算法
type: entity
entity_type: concept
tags: [模型选择, ML方法, K近邻]
created: 2026-05-18
updated: 2026-05-18
sources: [2026-05-18-ModelSelection算法类型及举例]
---

# KNN 选择算法

论文：FusionFactory（arXiv:2507.10540）。

找到历史上与当前 query 最相似的 K 条记录，用质量加权投票选择最佳模型。

## 公式

```
feature = [query_embedding, category_one_hot]
neighbors = K nearest records by EuclideanDistance(feature, record_i.feature)
Score(model_m) = Σ_{neighbor where model=m} quality_weight(neighbor)
```

## 特点

- 直观易理解，无需训练过程
- 推理时计算量大（遍历所有记录）
- 通过 [[Linfa]]（Rust）实现生产推理

## 相关页面

- [[ModelSelector-Registry]] — 所属注册表
- [[Linfa]] — 推理框架
- [[模型选择算法]] — 所属主题
