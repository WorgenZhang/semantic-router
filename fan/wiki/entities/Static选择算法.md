---
title: Static 选择算法
type: entity
entity_type: concept
tags: [模型选择, 冷启动]
created: 2026-05-18
updated: 2026-05-18
sources: [2026-05-18-ModelSelection算法类型及举例]
---

# Static 选择算法

最简单的模型选择方式：管理员为每个「类别-模型」组合预设固定分数，路由时选分数最高的模型。不学习，不适应。

## 公式

```
selected_model = argmax_m { Score(category, m) }
```

## 适用场景

- 冷启动（没有历史数据时的默认策略）
- 规则明确、不需要动态调整的场景
- 需要可预测行为的生产环境

## 相关页面

- [[ModelSelector-Registry]] — 所属注册表
- [[模型选择算法]] — 所属主题
