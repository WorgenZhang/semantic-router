# 知识库索引

> 最后更新：2026-05-13

---

## 概览

- 主题：vLLM 语义路由
- 素材总数：13
- Wiki 页面总数：33

---

## 实体页

> 人物、组织、概念、工具等

- [[ExtProc]] — Envoy gRPC External Processor 核心组件
- [[Decision-Engine]] — 信号评估 → Decision 匹配 → 模型选择
- [[RequestContext]] — 贯穿请求生命周期的上下文结构体
- [[Signal-Evaluation]] — 11 种信号类型评估系统
- [[流式响应-SSE]] — SSE 协议与 token 逐步推送
- [[分布式追踪]] — OpenTelemetry Trace/Span 体系
- [[幻觉检测]] — 响应体阶段的 NLI 幻觉检测
- [[ModelSelector-Registry]] — 9 种算法的选择器注册表
- [[Static选择算法]] — 固定分数选择
- [[Elo评分系统]] — Bradley-Terry 动态评分
- [[RouterDC]] — 双对比学习 embedding 路由
- [[AutoMix]] — POMDP 级联路由（小模型优先）
- [[Hybrid混合加权]] — Elo+RouterDC+AutoMix 加权融合
- [[KNN选择算法]] — K 近邻质量加权投票
- [[KMeans选择算法]] — 聚类+簇绑定模型
- [[SVM选择算法]] — RBF 核决策边界
- [[MLP选择算法]] — 神经网络 Softmax 输出
- [[Linfa]] — Rust ML 推理框架
- [[Candle]] — Rust GPU 神经网络推理框架

---

## 主题页

> 研究主题、知识领域

- [[ExtProc四阶段处理流水线]] — 请求头→请求体→响应头→响应体完整流水线
- [[模型路由机制]] — 智能模型选择与端点路由
- [[模型选择算法]] — 9 种选择算法全景与选型指南

---

## 素材摘要

> 每个消化过的素材都有一篇摘要

- [[2026-05-13-ExtProc四阶段初窥]] — 核心事件循环架构总览
- [[2026-05-13-ExtProc四阶段处理流水线详解]] — 四阶段 18 步详细实现
- [[2026-05-13-handleRequestHeaders举例解释]] — 请求头阶段数据流示例
- [[2026-05-13-handleRequestBody举例解释]] — 请求体阶段（核心路由）示例
- [[2026-05-13-handleResponseHeaders举例解释]] — 响应头阶段 VSR headers 示例
- [[2026-05-13-handleResponseBody举例解释]] — 响应体阶段 16 步数据流与幻觉检测
- [[2026-05-13-流式响应SSE]] — 流式响应概念与状态矩阵
- [[2026-05-13-分布式追踪和Span概念]] — Trace/Span 概念与可视化
- [[2026-05-13-Modality横跨两个阶段]] — Modality 信号检测 vs 路由执行
- [[2026-05-13-Signal评估宽进窄出设计]] — 11 种信号的消费时机分析
- [[2026-05-13-Decision前后顺序设计]] — per-Decision 配置与顺序决策
- [[2026-05-18-ModelSelection算法类型及举例]] — 9 种模型选择算法原理与实例
- [[2026-05-18-AutoMix算法公式解释]] — AutoMix POMDP 价值函数公式详解

---

## 对比分析

> 对比不同方案、工具、观点

（暂无）

---

## 综合分析

> 跨素材的深度分析

（暂无）
