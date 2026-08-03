---
title: MLP 选择算法
type: entity
entity_type: concept
tags: [模型选择, ML方法, 神经网络, GPU]
created: 2026-05-18
updated: 2026-05-18
sources: [2026-05-18-ModelSelection算法类型及举例]
---

# MLP 选择算法

论文：FusionFactory（arXiv:2507.10540）。

用多层感知机神经网络直接学习从 query 特征到最佳模型的映射。通过 [[Candle]]（Rust GPU 框架）加速推理。

## 公式

```
h₁ = ReLU(W₁ × x + b₁)          // 隐藏层1
h₂ = ReLU(W₂ × h₁ + b₂)         // 隐藏层2
output = Softmax(W₃ × h₂ + b₃)   // 输出层 → 各模型概率
selected_model = argmax_m { output[m] }
```

**训练损失：** 交叉熵 `Loss = -Σ_i y_i × log(output_i)`

## 典型网络结构

```
Input (778) → Hidden1 (256, ReLU) → Hidden2 (128, ReLU) → Output (N_models, Softmax)
```

输入 = 768 维 query embedding + 10 维 category one-hot

## 特点

- 表达能力最强（非线性映射）
- GPU 加速推理延迟极低
- 需要大量训练数据，有过拟合风险
- 黑箱不可解释

## 相关页面

- [[ModelSelector-Registry]] — 所属注册表
- [[Candle]] — GPU 推理框架
- [[模型选择算法]] — 所属主题
