---
title: AutoMix
type: entity
entity_type: concept
tags: [模型选择, POMDP, 级联路由, 成本优化]
created: 2026-05-18
updated: 2026-05-18
sources: [2026-05-18-ModelSelection算法类型及举例, 2026-05-18-AutoMix算法公式解释]
---

# AutoMix

论文：Automatically Mixing Language Models（arXiv:2310.12963, NeurIPS 2024）。

基于 POMDP（部分可观测马尔可夫决策过程）的级联路由算法：先用小模型回答，通过自我验证判断质量，不够则升级到大模型。

## 核心公式

```
V(model) = Quality(model) - λ × Cost(model) + γ × (1 - P_verify) × E[V(larger_model)]
```

| 变量 | 含义 | 来源 |
|------|------|------|
| Quality | 模型平均正确率 | benchmark 评估 |
| λ | 成本敏感度（默认 0.5） | 手动超参 |
| Cost | 归一化推理价格 = raw_cost / max_cost | API 定价 |
| γ | 升级折扣因子（默认 0.95） | 手动超参 |
| P_verify | 模型自信能搞定的概率 | Few-shot 自我验证统计 |
| E[V(larger)] | 升级后的期望价值 | 从最大模型反向递归计算 |

## 级联决策逻辑

```
if confidence < threshold (0.7):
    escalate to next larger model
else:
    accept answer
```

## 关键洞察

小模型的期望价值往往最高（便宜 + 失败了还能升级），所以系统优先尝试小模型。递归计算从最大模型开始反向推导：

```
V(405b) = Quality(405b) - λ×Cost(405b)           // 最大模型无升级选项
V(70b)  = ... + γ×(1-P_verify_70b)×V(405b)
V(8b)   = ... + γ×(1-P_verify_8b)×V(70b)
```

## 自我验证机制（P_verify）

1. 模型生成回答
2. 将 (question, answer) 再次输入同一模型，few-shot prompt 问"这个回答正确吗？"
3. 模型输出置信度 0~1
4. P_verify = 历史平均验证通过率

## 优缺点

| 优点 | 缺点 |
|------|------|
| 优化成本-质量权衡 | 级联增加总延迟 |
| 简单问题用小模型省钱 | 需要自我验证机制 |
| POMDP 理论保证最优 | 实现复杂度高 |

## 相关页面

- [[ModelSelector-Registry]] — 所属注册表
- [[Hybrid混合加权]] — 作为 Hybrid 的子组件之一
- [[模型选择算法]] — 所属主题
