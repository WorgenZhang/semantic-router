# vLLM Semantic Router 架构详解

## 1. 架构总览

vLLM Semantic Router 是一个**系统级智能路由层**，位于用户请求和 LLM 模型之间，通过高性能分类实现语义路由、安全防护和智能决策。

### 核心设计理念
- **Mixture-of-Models (MoM)**：多模型协作的集体智能
- **信号驱动决策**：从请求、响应和上下文中捕获缺失信号
- **高性能优先**：Rust 核心 + Go 服务层 + Envoy 云原生代理

---

## 2. 系统数据流：从请求到响应

```
用户请求 → EnvoyProxy → ExtProc(Go) → 分类引擎(Rust/Candle) → 决策引擎 → 模型路由 → LLM Backend
                                            ↑                        ↓
                                     ModernBERT Base Model      LoRA Adapters
                                     + Enhanced LoRA Adapter    (多任务增强)
```

### 完整请求处理流程

```
                          ┌─────────────────────────────────┐
                          │    1. 用户发送 OpenAI 格式请求    │
                          └───────────────┬─────────────────┘
                                          ▼
                          ┌─────────────────────────────────┐
                          │  2. Cloud Native EnvoyProxy     │
                          │  (高性能反向代理，流量入口)       │
                          └───────────────┬─────────────────┘
                                          ▼
                          ┌─────────────────────────────────┐
                          │  3. ExtProc gRPC 调用            │
                          │  (Envoy External Processor)      │
                          └───────────────┬─────────────────┘
                                          ▼
                ┌─────────────────────────────────────────────────┐
                │  4. Go 服务层 (OpenAIRouter)                     │
                │  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
                │  │请求头处理 │→│请求体处理 │→│ 请求过滤器链     │ │
                │  └──────────┘ └──────────┘ │  - Classification │ │
                │                            │  - Jailbreak      │ │
                │                            │  - PII Detection  │ │
                │                            │  - Cache Check    │ │
                │                            │  - RAG            │ │
                │                            │  - Reasoning      │ │
                │                            └─────────┬────────┘ │
                └──────────────────────────────────────┼──────────┘
                                                       ▼
                ┌─────────────────────────────────────────────────┐
                │  5. Rust 核心分类引擎 (via Candle Binding FFI)   │
                │  ┌────────────────────┐ ┌─────────────────────┐ │
                │  │ Small Base Model   │ │ Enhanced LoRA       │ │
                │  │ (ModernBERT)       │ │ Adapter 1 & 2       │ │
                │  │ Intent分类         │ │ (任务增强适配器)     │ │
                │  └────────┬───────────┘ └──────────┬──────────┘ │
                │           │      ┌─────────────────┘            │
                │           ▼      ▼                              │
                │  ┌─────────────────────┐                        │
                │  │ Continuous Batching  │                        │
                │  │ (连续批处理)         │                        │
                │  └─────────┬───────────┘                        │
                │            ▼                                    │
                │  ┌─────────────────────┐                        │
                │  │ DualPathRouter      │                        │
                │  │ (双路径智能路由)     │                        │
                │  │ Traditional vs LoRA │                        │
                │  └─────────┬───────────┘                        │
                └────────────┼────────────────────────────────────┘
                             ▼
                ┌─────────────────────────────────────────────────┐
                │  6. 决策引擎 (Decision Engine)                   │
                │  - 综合分类结果、置信度、上下文                   │
                │  - 选择目标模型和端点                            │
                │  - 应用路由策略 (auto/specified/anthropic)       │
                └───────────────────┬─────────────────────────────┘
                                    ▼
                ┌─────────────────────────────────────────────────┐
                │  7. 路由到目标 LLM Backend                       │
                │  (vLLM / OpenAI / Anthropic Claude)              │
                └─────────────────────────────────────────────────┘
```

---

## 3. 各核心模块详解

### 3.1 Cloud Native EnvoyProxy

| 属性 | 描述 |
|------|------|
| **角色** | 高性能云原生反向代理（流量入口） |
| **技术** | Envoy Proxy，支持 gRPC External Processing |
| **配置** | `config/envoy.yaml` |
| **功能** | HTTP/gRPC 请求拦截、负载均衡、TLS 终止、Observability |

**工作方式**：Envoy 收到用户请求后，通过 **ExtProc filter** 将请求转发给 Go 服务层进行语义分析和决策，再根据决策结果将请求路由到正确的后端模型。

---

### 3.2 Golang with Rust Binding for Envoy (ExtProc) Interface

| 属性 | 描述 |
|------|------|
| **角色** | ExtProc 服务层 & Rust 绑定桥接 |
| **语言** | Go + CGO (调用 Rust 库) |
| **入口** | `src/semantic-router/pkg/extproc/server.go` |
| **核心** | `src/semantic-router/pkg/extproc/router.go` → `OpenAIRouter` |

**ExtProc 处理阶段**：

| 阶段 | 文件 | 功能 |
|------|------|------|
| 请求头处理 | `processor_req_header.go` | 解析 HTTP 头，提取路由信息 |
| **请求体处理** | `processor_req_body.go` | **核心决策点**：分类、安全检查、路由 |
| 响应头处理 | `processor_res_header.go` | 处理后端响应头 |
| 响应体处理 | `processor_res_body.go` | 幻觉检测、PII 脱敏 |

**请求过滤器链**（在请求体处理阶段依次执行）：

| 过滤器 | 文件 | 功能 |
|--------|------|------|
| Classification | `req_filter_classification.go` | 意图分类，决定路由目标 |
| Jailbreak | `req_filter_jailbreak.go` | 越狱攻击检测 |
| PII | `req_filter_pii.go` | 个人信息检测 |
| Cache | `req_filter_cache.go` | 语义缓存命中检查 |
| RAG | `req_filter_rag.go` | 检索增强生成 |
| Reasoning | `req_filter_reason.go` | 推理决策（是否需要推理模型） |
| System Prompt | `req_filter_sys_prompt.go` | 系统提示词注入 |
| Tools | `req_filter_tools.go` | 工具调用路由 |
| Response API | `req_filter_response_api.go` | OpenAI Response API 处理 |

---

### 3.3 Small Base Model for Intent Classification（ModernBERT 基座模型）

| 属性 | 描述 |
|------|------|
| **角色** | 轻量级意图分类基座模型 |
| **架构** | ModernBERT（现代高效 BERT 变体） |
| **训练** | `src/training/classifier_model_fine_tuning/` |
| **推理** | 通过 Candle (Rust) 执行高性能推理 |
| **功能** | 将用户请求分类到预定义的意图类别 |

**工作方式**：
1. 接收用户请求文本
2. 通过 HuggingFace Tokenizers 进行分词
3. 使用 ModernBERT 进行意图分类推理
4. 输出分类结果（类别 + 置信度）

---

### 3.4 Enhanced LoRA Adapter 1 & 2

| 属性 | 描述 |
|------|------|
| **角色** | 基座模型的增强适配器 |
| **技术** | LoRA (Low-Rank Adaptation) |
| **训练代码** | `src/training/training_lora/` |
| **Rust 实现** | `candle-binding/src/model_architectures/lora/` |

**LoRA Adapter 1**: 增强分类任务（如 prompt guard、MMLU-Pro 等）
**LoRA Adapter 2**: 增强安全检测任务（如 jailbreak 检测、PII 检测等）

**核心文件**：
- `bert_lora.rs` — BERT + LoRA 融合推理
- `lora_adapter.rs` — LoRA 适配器加载和管理

**双路径架构**（DualPathRouter）:

```
请求 → DualPathRouter
        ├── Traditional Path (传统完整模型)
        │   └── 完整 ModernBERT 推理
        └── LoRA Path (LoRA 增强路径)
            └── Base Model + LoRA Adapter 融合推理
```

选择策略：
- **Automatic**：基于任务要求自动选择
- **Performance-based**：基于历史性能数据选择
- **Intelligent**：超级智能模式，综合多维因素决策

---

### 3.5 Continuous Batching（连续批处理）

| 属性 | 描述 |
|------|------|
| **角色** | 高效批处理推理请求 |
| **位置** | Rust 核心引擎内部 |
| **功能** | 将多个分类请求批量处理，提升吞吐量 |

连续批处理允许：
- 动态合并来自不同请求的分类任务
- 在 GPU/CPU 上并行执行推理
- 最大化硬件利用率

---

### 3.6 Rust Core for High-Performance Classification

| 属性 | 描述 |
|------|------|
| **角色** | 核心高性能分类引擎 |
| **框架** | Candle（Rust ML 框架） |
| **代码** | `candle-binding/src/` |
| **FFI** | `candle-binding/src/ffi/` — C/Go 外部函数接口 |

**子模块**：

| 子模块 | 路径 | 功能 |
|--------|------|------|
| classifiers | `src/classifiers/` | 统一分类器（unified）、LoRA 分类器、传统分类器 |
| model_architectures | `src/model_architectures/` | 模型工厂、嵌入模型、生成模型、LoRA 架构、路由逻辑 |
| core | `src/core/` | 核心数据结构和工具 |
| ffi | `src/ffi/` | Go/C 语言外部函数接口绑定 |

---

### 3.7 Candle Framework

| 属性 | 描述 |
|------|------|
| **角色** | Rust 原生 ML 推理框架 |
| **用途** | 替代 PyTorch，在 Rust 中直接执行模型推理 |
| **优势** | 零 Python 开销、内存安全、接近 C++ 的性能 |

---

### 3.8 HuggingFace Tokenizers Library

| 属性 | 描述 |
|------|------|
| **角色** | 高性能文本分词库 |
| **用途** | 将用户请求文本转换为模型可处理的 token |
| **优势** | Rust 实现，速度极快，支持所有主流分词器 |

---

### 3.9 Fine-tuning ModernBERT for Intent Classification

| 属性 | 描述 |
|------|------|
| **角色** | 模型训练流水线 |
| **代码** | `src/training/` |

**训练任务**：

| 训练任务 | 路径 | 描述 |
|----------|------|------|
| 基座分类模型 | `classifier_model_fine_tuning/` | 意图分类基座模型微调 |
| LoRA 分类适配器 | `training_lora/classifier_model_fine_tuning_lora/` | LoRA 分类增强 |
| LoRA Prompt Guard | `training_lora/prompt_guard_fine_tuning_lora/` | 越狱检测 LoRA |
| LoRA PII 检测 | `training_lora/pii_model_fine_tuning_lora/` | PII 检测 LoRA |
| ModernBERT 情感 | `modernbert_dissat_pipeline/` | 不满意检测 |
| 双分类器 | `dual_classifier/` | 双任务分类器 |
| 缓存嵌入 | `cache_embeddings/` | 语义缓存嵌入模型 |

---

### 3.10 Go 层分类器（Classification Package）

| 分类器 | 文件 | 功能 |
|--------|------|------|
| **Classifier**（主分类器） | `classifier.go` | 统一管理所有子分类器 |
| UnifiedClassifier | `unified_classifier.go` | 统一分类接口 |
| EmbeddingClassifier | `embedding_classifier.go` | 嵌入式分类 |
| VLLMClassifier | `vllm_classifier.go` | 外部 vLLM 模型分类 |
| KeywordClassifier | `keyword_classifier.go` | 关键词分类 |
| LanguageClassifier | `language_classifier.go` | 语言检测 |
| LatencyClassifier | `latency_classifier.go` | 延迟感知分类 |
| PreferenceClassifier | `preference_classifier.go` | 用户偏好分类 |
| HallucinationDetector | `hallucination_detector.go` | 幻觉检测 |
| FactCheckClassifier | `fact_check_classifier.go` | 事实核查 |
| MCPClassifier | `mcp_classifier.go` | MCP 工具分类 |
| ModelDiscovery | `model_discovery.go` | 自动模型发现 |

---

## 4. 数据集（Datasets）

| 数据集 | 用途 |
|--------|------|
| **MMLU-Pro Dataset** | 多任务学习评估，训练基座分类模型识别学科领域 |
| **Jailbreak Datasets** | 越狱攻击样本，训练 Prompt Guard 安全检测 |
| **Microsoft Presidio Dataset** | PII（个人信息）检测训练数据 |
| **Customer Dataset** | 客户自定义领域数据 |
| **Private Customer Dataset** | 私有客户数据（隐私保护） |

---

## 5. 决策引擎（Decision Engine）

决策引擎位于 `src/semantic-router/pkg/decision/engine.go`，负责：

1. **综合多信号**：将分类结果、置信度、推理决策等信号综合
2. **匹配决策规则**：根据配置决定路由目标
3. **模型选择**：选择最优模型（auto routing / specified model / reasoning model）
4. **端点选择**：选择最佳后端端点

---

## 6. 技术栈总结

| 层级 | 技术 | 用途 |
|------|------|------|
| **代理层** | Envoy Proxy | 云原生高性能代理 |
| **服务层** | Go (Golang) | ExtProc 服务、业务逻辑 |
| **核心引擎** | Rust + Candle | 高性能 ML 推理 |
| **分词** | HuggingFace Tokenizers (Rust) | 文本分词 |
| **基座模型** | ModernBERT | 意图分类 |
| **模型增强** | LoRA Adapters | 任务特定适配 |
| **训练** | Python + PyTorch | 模型微调 |
| **部署** | Docker + Kubernetes | 容器化部署 |
| **可观测性** | OpenTelemetry | 追踪、指标 |
