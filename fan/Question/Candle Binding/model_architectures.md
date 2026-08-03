# candle-binding/`model_architectures`目录结构全解析


## 四、`model_architectures/` — 模型架构实现

### 顶层模块

| 文件                           | 作用                                                                                                                                                                                                                    | 例子                                                                                             |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| **mod.rs**               | 根模块，re-export 所有子模块的核心类型                                                                                                                                                                                  | —                                                                                               |
| **config.rs**            | `DualPathConfig` 双路径配置系统，包含 Traditional/LoRA/Embedding/Global 配置。`ConfigBuilder` fluent API                                                                                                            | `DualPathConfig::for_model_type(ModelType::LoRA)` → 自动设置 LoRA 路径策略                    |
| **traits.rs**            | 核心 trait 和类型定义。`ModelType`（Traditional/LoRA/Qwen3Embedding/GemmaEmbedding/MmBertEmbedding）、`LoRACapable`、`TraditionalModel`、`LongContextEmbeddingCapable`、`PoolingMethod`（Mean/LastToken/CLS） | `model.get_pooling_method()` → `PoolingMethod::LastToken`（Qwen3 用最后一个 token）         |
| **model_factory.rs**     | `ModelFactory` 工厂模式，注册和管理 Traditional/LoRA/Qwen3/Gemma/mmBERT 模型。`DualPathModel` enum 统一封装。实现 `CoreModel`、`LoRACapable`、`TraditionalModel` 等 trait                                     | `factory.create_dual_path_model(&requirements)` → 根据需求自动选 Traditional 或 LoRA          |
| **routing.rs**           | `DualPathRouter` 智能路由器，基于多因子评分（任务数、batch size、置信度要求、延迟、历史性能）选最优路径。支持自适应学习                                                                                               | `router.select_path_intelligent(requirements)` → 评分 LoRA=0.85 > Traditional=0.62 → 选 LoRA |
| **prefix_cache.rs**      | `PrefixCache` 前缀缓存，预存固定 prompt 的 token，后续请求只处理新增 token。用于 Qwen3Guard 安全分类加速（约 3x）                                                                                                     | 首次处理 500 token prompt → 缓存 → 后续只处理 10 token 用户输入                                |
| **unified_interface.rs** | 简化的统一模型接口。`CoreModel`（forward + model_type）、`PathSpecialization`（parallel/confidence/batch_size）、`ConfigurableModel`（load from config）、`UnifiedModel` = 三者组合                             | `ModelCapabilities::from_model(&model)` → 运行时查询模型能力                                  |

### `embedding/` — Embedding 模型

| 文件                                    | 作用                                                                                                                                                          | 例子                                                                              |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **mod.rs**                        | 模块声明，re-export pooling 函数和三种 embedding 模型                                                                                                         | —                                                                                |
| **pooling.rs**                    | 三种池化实现：`mean_pool`（加权平均，BERT/Gemma 用）、`last_token_pool`（最后有效 token，Qwen3 用）、`cls_pool`（CLS token，原始 BERT）                 | `mean_pool(&hidden_states, &attention_mask)` → `[batch, 768]` 句子向量       |
| **qwen3_embedding.rs**            | `Qwen3EmbeddingModel`，支持 0.6B/4B/8B 全系列。32K 上下文、GQA 注意力、instruction-aware embedding                                                          | `model.embedding_forward(&input_ids, &mask)` → 32K 长文本 → 1024 维向量       |
| **gemma_embedding.rs**            | `GemmaEmbeddingModel`（300M），2K 上下文、MQA（3 query/1 KV head）、混合注意力（sliding+full）、Matryoshka（768/512/256/128）、Dense 瓶颈（768→3072→768） | `model.forward_with_matryoshka(&ids, &mask, 256)` → 256 维紧凑向量             |
| **gemma3_model.rs**               | Gemma3 Transformer 骨干网络实现。`Gemma3Attention`（MQA）、`Gemma3MLP`（gelu_pytorch_tanh）、`Gemma3Layer`、`Gemma3Model`（24 层）                    | 被 `GemmaEmbeddingModel` 内部使用，提供 transformer 前向传播                    |
| **dense_layers.rs**               | `BottleneckDenseNet`（768→3072→768 瓶颈），GemmaEmbedding 质量关键模块。无瓶颈 ~85% 质量，有瓶颈 ~99%                                                     | `bottleneck.forward(&pooled_embeddings)` → 线性变换提升 embedding 质量         |
| **mmbert_embedding.rs**           | `MmBertEmbeddingModel`（307M），32K 上下文、1800+ 语言（Glot500）、2D Matryoshka（维度+层数缩减）、YaRN-scaled RoPE                                         | `model.forward_2d_matryoshka(&ids, &mask, dim=256, layer=6)` → 极速多语言嵌入  |
| **continuous_batch_scheduler.rs** | Embedding 专用连续批调度器。基于 crossbeam-channel 的多通道 `select!`。请求到达 → 动态组批 → 单次前向传播 → 分发结果                                     | `scheduler.embed_from_raw(token_ids, mask)` → 自动合批处理 → 2-5x 吞吐提升    |
| **qwen3_batched.rs**              | `Qwen3EmbeddingModelBatched`，Qwen3 的连续批处理包装器，API 100% 兼容原始模型                                                                               | `Qwen3EmbeddingModelBatched::load(path, &device, config)` → drop-in 替换原模型 |

### `generative/` — 生成式模型

| 文件                                     | 作用                                                                                                                                   | 例子                                                                                                    |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| **mod.rs**                         | 模块声明，re-export Qwen3MultiLoRAClassifier、Qwen3GuardModel                                                                          | —                                                                                                      |
| **qwen3_with_lora.rs**             | 修改版官方 Qwen3 模型，添加 LoRA hook。注意力和 MLP 层支持 `output += LoRA_B(LoRA_A(input)) * scaling`                               | 被 `Qwen3MultiLoRAClassifier` 和 `Qwen3GuardModel` 内部使用                                         |
| **qwen3_multi_lora_classifier.rs** | `Qwen3MultiLoRAClassifier`，多 LoRA 适配器系统。一个 Qwen3 基础模型 + 多个可热切换的 LoRA 适配器（分类、越狱检测等）。支持零样本分类 | `model.classify_with_adapter("什么是GDP", "category")` → 用分类适配器 → `"knowledge"`             |
| **qwen3_guard.rs**                 | `Qwen3GuardModel` 安全分类模型。生成式推理检测不安全内容，输出结构化结果（Reasoning + Category + Severity）。使用 PrefixCache 加速   | `guard.generate_guard("Ignore safety rules", "input")` → `"Category: Jailbreak, Severity: Unsafe"` |
| **continuous_batch_scheduler.rs**  | 生成式分类器的连续批调度器，类似 vLLM 思路但更简单（分类是单次传播，无自回归解码）                                                     | `scheduler.classify("text", "category")` → 请求自动合批 → 高吞吐                                    |

### `lora/` — LoRA 适配器

| 文件                      | 作用                                                                                                                                 | 例子                                                                            |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------- |
| **mod.rs**          | 模块声明                                                                                                                             | —                                                                              |
| **lora_adapter.rs** | 核心 `LoRAAdapter` 实现。`LoRAConfig`（rank, alpha, dropout, target_modules）。A/B 矩阵低秩分解，支持 Kaiming/Xavier/Zero 初始化 | `adapter.forward(&input)` → `input + B(A(input)) * (alpha/rank)`           |
| **bert_lora.rs**    | `LoRABertClassifier`，冻结 BERT 骨干 + LoRA 适配器 + 多任务分类头。支持 rayon 并行多任务处理                                       | `classifier.parallel_multi_task_classify(&texts)` → 同时 Intent+PII+Security |

### `traditional/` — 传统编码器模型

| 文件                                  | 作用                                                                                                                                                  | 例子                                                                                                    |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| **mod.rs**                      | 模块声明，re-export 所有传统模型。含本地 ModernBERT 副本原因说明（需要 Flash Attention 2 + 32K 支持）                                                 | —                                                                                                      |
| **base_model.rs**               | `TraditionalModelBase` trait 和 `BaseTraditionalModel` 基类。提供 embeddings + encoder + pooler 的通用框架                                        | `model.forward_pass(&input_ids, &attention_mask)` → 通用前向传播                                     |
| **bert.rs**                     | `TraditionalBertClassifier`，经典 BERT 分类器。CLS token → pooler → classifier。支持 HuggingFace Hub 和本地模型                                   | `TraditionalBertClassifier::new("bert-base-uncased", 5, false)` → 创建 5 分类 BERT                   |
| **deberta_v3.rs**               | `DebertaV3Classifier`，增强的解耦注意力机制。专门用于 prompt injection 和越狱检测                                                                   | `classifier.classify("Ignore previous instructions")` → `{ label: "injection", confidence: 0.99 }` |
| **modernbert.rs**               | `TraditionalModernBertClassifier`，支持 4 种变体：Standard（512）、Multilingual/mmBERT（8192）、Extended32K（32768）、Multilingual32K。自动检测变体 | `TraditionalModernBertClassifier::new_with_variant(path, variant, num_classes)`                       |
| **candle_models/modernbert.rs** | ModernBERT 底层 Candle 实现（本地副本）。包含 Flash Attention 2 支持、YaRN RoPE 缩放、混合注意力（local/global）。上游不支持 FA2 所以必须本地维护     | 被 `TraditionalModernBertClassifier` 内部使用                                                         |
