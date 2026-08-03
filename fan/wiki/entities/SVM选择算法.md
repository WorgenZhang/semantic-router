---
title: SVM 选择算法
type: entity
entity_type: concept
tags: [模型选择, ML方法, 支持向量机]
created: 2026-05-18
updated: 2026-05-18
sources: [2026-05-18-ModelSelection算法类型及举例]
---

# SVM 选择算法

论文：FusionFactory（arXiv:2507.10540）、Avengers-Pro（arXiv:2508.12631）。

在高维 embedding 空间中学习决策边界，使用 RBF 核 + One-vs-One 投票划分模型最佳适用区域。

## 公式

**RBF 核函数：**
```
K(x_i, x_j) = exp(-γ × ||x_i - x_j||²)
```

**决策函数：**
```
f(x) = Σ_i α_i × y_i × K(x, x_i) + b
```

**选择（多分类投票）：**
```
selected_model = argmax_m { votes(m) }
```

## 特点

- 处理高维数据能力强
- 最大间隔保证泛化能力
- 不直接输出概率，对参数 γ 和 C 敏感
- 通过 [[Linfa]]（Rust）实现

## 相关页面

- [[ModelSelector-Registry]] — 所属注册表
- [[Linfa]] — 推理框架
- [[模型选择算法]] — 所属主题
