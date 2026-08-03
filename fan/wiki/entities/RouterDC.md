---
title: RouterDC
type: entity
entity_type: concept
tags: [模型选择, embedding, 对比学习]
created: 2026-05-18
updated: 2026-05-18
sources: [2026-05-18-ModelSelection算法类型及举例]
---

# RouterDC（双对比学习路由）

论文：Query-Based Router by Dual Contrastive Learning（arXiv:2409.19886）。

将 query 和 model 都映射到同一个 embedding 空间，通过余弦相似度匹配最适合的模型。

## 核心公式

```
cos_sim(q, m) = (q · m) / (||q|| × ||m||)
similarity(q, m) = σ(cos_sim(q, m) / τ)          // τ=0.07 温度参数
P(model_i | query) = exp(sim_i / τ) / Σ_j exp(sim_j / τ)
```

## 工作方式

- 模型的 embedding 来自其描述文本（如"擅长数学推理、逻辑分析"）
- query 的 embedding 来自用户输入
- 温度参数 τ 越小，分布越尖锐（更确定）

## 优缺点

| 优点 | 缺点 |
|------|------|
| 语义级匹配 | 依赖 embedding 质量 |
| 无需历史反馈即可工作 | 需要为模型写好描述 |
| 对新 query 泛化好 | 计算 embedding 有延迟 |

## 相关页面

- [[ModelSelector-Registry]] — 所属注册表
- [[Hybrid混合加权]] — 作为 Hybrid 的子组件之一
- [[模型选择算法]] — 所属主题
