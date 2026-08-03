---
title: Candle
type: entity
entity_type: tool
tags: [Rust, GPU, 神经网络推理, 工具]
created: 2026-05-18
updated: 2026-05-18
sources: [2026-05-18-ModelSelection算法类型及举例]
---

# Candle

Rust 实现的 GPU 推理框架，用于 [[MLP选择算法]] 的生产推理加速。

## 技术栈定位

Python 训练 MLP 模型 → 导出权重 → Candle (Rust + GPU) 高性能推理

与 [[Linfa]] 的区别：Candle 专注神经网络 GPU 推理，Linfa 做传统 ML 算法。

## 相关页面

- [[Linfa]] — 传统 ML 推理框架
- [[MLP选择算法]] — 使用 Candle 的算法
- [[ModelSelector-Registry]] — 所属注册表
