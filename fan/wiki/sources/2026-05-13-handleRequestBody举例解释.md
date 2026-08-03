---
title: handleRequestBody 每步举例解释
date: 2026-05-13
type: source
source_path: raw/notes/2026-05-13-handleRequestBody举例解释.md
tags: [handleRequestBody, 请求体, Decision Engine, 模型路由, 安全检查]
sources: []
images: 0
image_paths: []
---

# handleRequestBody 每步举例解释

## 核心观点

- 请求体阶段是 ExtProc 处理的核心，包含 16+ 步处理
- Decision 评估通过 11 种信号分析用户内容，匹配最佳决策并选择模型
- 每一步都可能"短路"返回（403/429/缓存命中等），正常流程才走完所有步骤
- 最终输出是 HeaderMutation（路由信息）+ BodyMutation（修改后的请求体，model 字段被改写）

## 关键数据流示例

**输入**：
```json
{"model": "auto", "messages": [{"role": "user", "content": "帮我写一个排序算法"}], "stream": true}
```

**Decision 评估输出**：
- decisionName = "coding_decision"
- selectedModel = "qwen14b-rack1"
- confidence = 0.85

**最终输出**（HeaderMutation + BodyMutation）：
```
SetHeaders: [x-selected-model: qwen14b-rack1, x-gateway-destination-endpoint: 10.0.1.5:8000, Authorization: Bearer sk-xxx]
BodyMutation: {"model":"Qwen/Qwen2.5-14B-Instruct","messages":[...],"stream":true}
```

## 关键概念

- [[handleRequestBody]]：第二阶段处理器
- [[Decision Engine]]：信号评估 → Decision 匹配 → 模型选择
- [[Signal Evaluation]]：Keyword/Embedding/Domain/Complexity 等 11 种信号
- [[Jailbreak 检测]]：安全防护
- [[语义缓存]]：Cache.FindSimilarWithThreshold()
- [[RAG 插件]]：上下文检索注入
