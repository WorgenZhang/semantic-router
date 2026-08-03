# 操作日志

> 记录知识库的所有变更历史

---

## 2026-05-18 batch-ingest | 模型选择算法系列（2 篇）

**新增素材摘要**：
- wiki/sources/2026-05-18-ModelSelection算法类型及举例.md
- wiki/sources/2026-05-18-AutoMix算法公式解释.md

**新增实体页**：
- wiki/entities/ModelSelector-Registry.md
- wiki/entities/Static选择算法.md
- wiki/entities/Elo评分系统.md
- wiki/entities/RouterDC.md
- wiki/entities/AutoMix.md
- wiki/entities/Hybrid混合加权.md
- wiki/entities/KNN选择算法.md
- wiki/entities/KMeans选择算法.md
- wiki/entities/SVM选择算法.md
- wiki/entities/MLP选择算法.md
- wiki/entities/Linfa.md
- wiki/entities/Candle.md

**新增主题页**：
- wiki/topics/模型选择算法.md

**更新页面**：
- wiki/entities/Decision-Engine.md（模型选择算法列表改为双向链接）
- wiki/topics/模型路由机制.md（模型选择算法部分补充链接）

---

## 2026-05-15 ingest | handleResponseBody 补完

**更新素材摘要**：
- wiki/sources/2026-05-13-handleResponseBody举例解释.md（从空占位 → 完整 16 步摘要）

**更新实体页**：
- wiki/entities/幻觉检测.md（追加：执行时机与检测举例）

**更新主题页**：
- wiki/topics/ExtProc四阶段处理流水线.md（素材来源补入 handleResponseBody）

**更新索引**：
- index.md（handleResponseBody 描述从"待补充"改为实际摘要）

---

## 2026-05-13 batch-ingest | Decision 设计分析系列（3 篇新增）

**新增素材摘要**：
- wiki/sources/2026-05-13-Modality横跨两个阶段.md
- wiki/sources/2026-05-13-Signal评估宽进窄出设计.md
- wiki/sources/2026-05-13-Decision前后顺序设计.md

**更新实体页**：
- wiki/entities/Decision-Engine.md（追加：per-Decision 配置与信号消费分类）
- wiki/entities/Signal-Evaluation.md（追加：宽进窄出设计与 Modality 特殊性）

---

## 2026-05-13 batch-ingest | ExtProc 四阶段系列（8 篇）

**新增素材摘要**：
- wiki/sources/2026-05-13-ExtProc四阶段初窥.md
- wiki/sources/2026-05-13-ExtProc四阶段处理流水线详解.md
- wiki/sources/2026-05-13-handleRequestHeaders举例解释.md
- wiki/sources/2026-05-13-handleRequestBody举例解释.md
- wiki/sources/2026-05-13-handleResponseHeaders举例解释.md
- wiki/sources/2026-05-13-handleResponseBody举例解释.md（空内容，待补充）
- wiki/sources/2026-05-13-流式响应SSE.md
- wiki/sources/2026-05-13-分布式追踪和Span概念.md

**新增实体页**：
- wiki/entities/ExtProc.md
- wiki/entities/Decision-Engine.md
- wiki/entities/RequestContext.md
- wiki/entities/Signal-Evaluation.md
- wiki/entities/流式响应-SSE.md
- wiki/entities/分布式追踪.md
- wiki/entities/幻觉检测.md

**新增主题页**：
- wiki/topics/ExtProc四阶段处理流水线.md
- wiki/topics/模型路由机制.md

---

## 2026-05-13 — 初始化

- **操作**：创建知识库
- **主题**：vLLM 语义路由
- **状态**：完成
