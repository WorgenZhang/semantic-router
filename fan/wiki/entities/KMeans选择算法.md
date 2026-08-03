---
title: KMeans 选择算法
type: entity
entity_type: concept
tags: [模型选择, ML方法, 聚类]
created: 2026-05-18
updated: 2026-05-18
sources: [2026-05-18-ModelSelection算法类型及举例]
---

# KMeans 选择算法

论文：Avengers-Pro（arXiv:2508.12631）。

将所有历史 query 聚成 K 个簇，每个簇绑定一个最佳模型。新 query 到来时判断属于哪个簇，选该簇的最佳模型。

## 公式

```
clusters = KMeans(all_feature_vectors, K=num_models)
best_model(cluster_i) = argmax_m { (1-λ)×Quality(m, cluster_i) + λ×Efficiency(m) }
// 推理时：
cluster = argmin_c { distance(query_feature, centroid_c) }
selected_model = best_model(cluster)
```

## 特点

- 推理极快（只需计算到质心的距离）
- 簇边界是硬划分，对 outlier 不友好
- 通过 [[Linfa]]（Rust）实现

## 相关页面

- [[ModelSelector-Registry]] — 所属注册表
- [[Linfa]] — 推理框架
- [[模型选择算法]] — 所属主题
