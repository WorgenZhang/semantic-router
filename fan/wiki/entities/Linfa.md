---
title: Linfa
type: entity
entity_type: tool
tags: [Rust, ML推理, 工具]
created: 2026-05-18
updated: 2026-05-18
sources: [2026-05-18-ModelSelection算法类型及举例]
---

# Linfa

Rust 实现的机器学习框架，用于 [[ModelSelector-Registry]] 中 ML 方法的生产推理。

## 支持的算法

- [[KNN选择算法]]
- [[KMeans选择算法]]
- [[SVM选择算法]]

## 技术栈定位

Python 训练模型 → 导出参数 → Linfa (Rust) 加载并执行推理

与 [[Candle]] 的区别：Linfa 做传统 ML 算法，Candle 做神经网络 GPU 推理。

## 相关页面

- [[Candle]] — 神经网络推理框架
- [[ModelSelector-Registry]] — 所属注册表
