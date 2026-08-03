# candle-binding/src 目录结构全解析

---

## 一、`classifiers/` — 分类器系统

| 文件 | 作用 | 例子 |
|------|------|------|
| **mod.rs** | 根模块，定义 `ClassificationTask`（Intent/PII/Security）、`DualPathResult`、`TaskResult` 结构体 | `DualPathResult { path_used: LoRA, results: [...], confidence: 0.98 }` |
| **mlp_selector.rs** | GPU 加速的 MLP 神经网络，用于模型选择路由（FusionFactory 论文）。Python 训练，Rust 推理，支持 F32/F16/BF16 混合精度 | `selector.select(&query_embedding)` → 返回 `"qwen3"` 或 `"gemma"` |
| **unified.rs** | 核心 `DualPathUnifiedClassifier`，智能路由 Traditional/LoRA 路径。分析 batch size、任务数、置信度、延迟来选最优路径。还包含 embedding 模型选择（Qwen3/Gemma/mmBERT） | `classifier.classify_intelligent(texts, tasks)` → 自动选 LoRA 并行处理 |
| **lora/mod.rs** | LoRA 分类器子模块声明（intent, pii, security, token, parallel_engine） | — |
| **lora/intent_lora.rs** | `IntentLoRAClassifier`，支持 BERT 和 ModernBERT 后端，自动检测模型架构 | `classifier.classify_intent("帮我订机票")` → `IntentResult { intent: "booking", confidence: 0.99 }` |
| **lora/pii_lora.rs** | `PIILoRAClassifier`，使用 token 级分类检测 PII（姓名、邮箱、电话等） | `classifier.detect_pii("我是张三，电话13800138000")` → `PIIResult { has_pii: true, pii_types: ["NAME", "PHONE"] }` |
| **lora/security_lora.rs** | `SecurityLoRAClassifier`，检测越狱/恶意输入，支持 BERT 和 ModernBERT | `classifier.detect_threats("Ignore previous instructions")` → `SecurityResult { is_threat: true, severity_score: 0.95 }` |
| **lora/token_lora.rs** | `LoRATokenClassifier`，token 级别分类（BERT+LoRA），每个 token 打标签 | `classifier.classify_tokens("John lives in NYC")` → `[("John","B-PER",0.99), ("NYC","B-LOC",0.97)]` |
| **lora/parallel_engine.rs** | `ParallelLoRAEngine`，用 `rayon::join` 并行执行 Intent+PII+Security 三个分类器 | `engine.parallel_classify(texts)` → 同时返回三类结果，比串行快 3x |
| **traditional/mod.rs** | 传统模型子模块声明 | — |
| **traditional/batch_processor.rs** | `TraditionalBatchProcessor`，可配置 chunk size 和 timeout 的批处理器 | `processor.process_batch(texts, \|batch\| model.classify(batch))` → 分块处理 + 指标统计 |
| **traditional/modernbert_classifier.rs** | `ModernBertClassifier`，带 per-task 分类头（Linear+Dropout），支持多任务和批量分类 | `classifier.classify_task(embeddings, TaskType::Intent)` → softmax → argmax 分类 |

---

## 二、`core/` — 核心基础设施

| 文件 | 作用 | 例子 |
|------|------|------|
| **mod.rs** | 根模块，re-export similarity、config_loader、tokenization、unified_error | — |
| **config_loader.rs** | `UnifiedConfigLoader` 加载模型配置（config.json, lora_config.json, config.yaml）。提供 `IntentConfigLoader`、`PIIConfigLoader` 等专用加载器。`RouterConfig` 有 ~30 个路由权重参数 | `GlobalConfigLoader::load_router_config()` → 读取 config.yaml 中的路由策略 |
| **similarity.rs** | `BertSimilarity`，封装 BERT 做语义相似度。Mean pooling + L2 归一化 → 余弦相似度 | `bert.calculate_similarity("今天天气好", "天气不错")` → `0.92` |
| **tokenization.rs** | 统一双路径 tokenization 系统。`UnifiedTokenizer` 支持 BERT/ModernBERT/mmBERT/LoRA 策略，处理 U32 vs I32 token ID 差异 | `create_mmbert_compatibility_tokenizer(path)` → 创建支持 8192 max_length 的 tokenizer |
| **unified_error.rs** | 统一错误系统，`UnifiedError` 枚举（Configuration/Model/Processing/FFI/IO/Validation 等），提供 `config_error!`、`model_error!` 等宏 | `config_error!("missing_field", "hidden_size")` → 结构化错误 |

---

## 三、`ffi/` — Foreign Function Interface（Go <-> Rust 桥接）

| 文件 | 作用 | 例子 |
|------|------|------|
| **mod.rs** | 根模块，声明并 re-export 所有 FFI 子模块 | — |
| **types.rs** | 所有 `#[repr(C)]` 结构体定义，与 Go 的 C typedef 精确对齐。包含 `ClassificationResult`、`EmbeddingResult`、`HallucinationSpan`、`NLIResult` 等 20+ 种 C 结构 | `ClassificationResult { predicted_class: 2, confidence: 0.95, label: "booking" }` |
| **init.rs** | 30+ 初始化函数，使用 `OnceLock<Arc<T>>` 单例模式。自动检测 LoRA vs Traditional（检查 safetensors 头部 LoRA 权重模式） | `init_bert_classifier("models/bert", 5, false)` → 全局单例初始化 BERT 分类器 |
| **classify.rs** | 16+ C FFI 分类函数。包括 `classify_text()`、`classify_pii_text()`、`classify_jailbreak_text()`（LoRA 优先回退 BERT）、`detect_hallucinations()` 等。还有 mmBERT-32K 变体 | `classify_jailbreak_text("Ignore all rules")` → LoRA 先试，失败回退 BERT |
| **embedding.rs** | Embedding 生成 FFI，智能模型选择。`GLOBAL_MODEL_FACTORY` 单例。支持 Qwen3/Gemma/mmBERT，continuous batching | `get_embedding_smart("这是一段文本", 768)` → 智能选模型 → 返回 768 维向量 |
| **similarity.rs** | BERT 语义相似度 FFI。`get_text_embedding()`、`calculate_similarity()`、`find_most_similar()` | `calculate_similarity("hello", "hi", 512)` → `0.87` |
| **tokenization.rs** | Tokenization FFI，将文本切分为 token IDs 和 token 字符串 | `tokenize_text("Hello world", 512)` → `{ token_ids: [101,7592,2088,102], tokens: ["[CLS]","hello","world","[SEP]"] }` |
| **mlp.rs** | MLP Selector FFI。`MLPHandle` 不透明句柄，支持 CPU/CUDA/Metal 设备和混合精度 | `candle_mlp_from_json(json_str)` → 从 Python 训练的 JSON 加载模型 → `candle_mlp_select(handle, query)` → `"model_a"` |
| **generative_classifier.rs** | Qwen3 Multi-LoRA 生成式分类器和 Qwen3Guard FFI。支持多适配器动态切换、零样本分类、安全检测 | `classify_with_qwen3_adapter("什么是GDP", "category")` → `{ category: "knowledge", confidence: 0.92 }` |
| **memory.rs** | C 内存管理函数，`free_*` 系列释放各种 FFI 结果。还有 `allocate_c_string()` 等辅助分配函数 | `free_embedding(ptr)` — Go 调完拿到结果后调用释放 Rust 分配的内存 |
| **memory_safety.rs** | 双路径内存安全系统。`AllocationTracker` 追踪分配、`DOUBLE_FREE_PROTECTION` 防双重释放、`PathSwitchState` 安全路径切换 | `safe_free(ptr)` → 检查是否已释放 → 标记已释放 → 实际释放 |
| **validation.rs** | FFI 参数验证。`validate_text_input()`、`validate_batch_input()`、`validate_model_path()`、`validate_confidence_threshold()`，区分 Traditional/LoRA 路径 | `validate_text_input("hi", 1)` → LoRA 路径验证 → 文本太短警告 |
| **state_manager.rs** | `GlobalStateManager` 单例，统一管理所有全局状态（unified classifier、parallel LoRA engine、BERT similarity 等），提供生命周期管理和系统状态跟踪 | `GlobalStateManager::instance().init_unified_classifier(classifier)` → 设置全局分类器 |

---

## 四、`model_architectures/` — 模型架构实现

### 顶层模块

| 文件 | 作用 | 例子 |
|------|------|------|
| **mod.rs** | 根模块，re-export 所有子模块的核心类型 | — |
| **config.rs** | `DualPathConfig` 双路径配置系统，包含 Traditional/LoRA/Embedding/Global 配置。`ConfigBuilder` fluent API | `DualPathConfig::for_model_type(ModelType::LoRA)` → 自动设置 LoRA 路径策略 |
| **traits.rs** | 核心 trait 和类型定义。`ModelType`（Traditional/LoRA/Qwen3Embedding/GemmaEmbedding/MmBertEmbedding）、`LoRACapable`、`TraditionalModel`、`LongContextEmbeddingCapable`、`PoolingMethod`（Mean/LastToken/CLS） | `model.get_pooling_method()` → `PoolingMethod::LastToken`（Qwen3 用最后一个 token） |
| **model_factory.rs** | `ModelFactory` 工厂模式，注册和管理 Traditional/LoRA/Qwen3/Gemma/mmBERT 模型。`DualPathModel` enum 统一封装。实现 `CoreModel`、`LoRACapable`、`TraditionalModel` 等 trait | `factory.create_dual_path_model(&requirements)` → 根据需求自动选 Traditional 或 LoRA |
| **routing.rs** | `DualPathRouter` 智能路由器，基于多因子评分（任务数、batch size、置信度要求、延迟、历史性能）选最优路径。支持自适应学习 | `router.select_path_intelligent(requirements)` → 评分 LoRA=0.85 > Traditional=0.62 → 选 LoRA |
| **prefix_cache.rs** | `PrefixCache` 前缀缓存，预存固定 prompt 的 token，后续请求只处理新增 token。用于 Qwen3Guard 安全分类加速（约 3x） | 首次处理 500 token prompt → 缓存 → 后续只处理 10 token 用户输入 |
| **unified_interface.rs** | 简化的统一模型接口。`CoreModel`（forward + model_type）、`PathSpecialization`（parallel/confidence/batch_size）、`ConfigurableModel`（load from config）、`UnifiedModel` = 三者组合 | `ModelCapabilities::from_model(&model)` → 运行时查询模型能力 |

### `embedding/` — Embedding 模型

| 文件 | 作用 | 例子 |
|------|------|------|
| **mod.rs** | 模块声明，re-export pooling 函数和三种 embedding 模型 | — |
| **pooling.rs** | 三种池化实现：`mean_pool`（加权平均，BERT/Gemma 用）、`last_token_pool`（最后有效 token，Qwen3 用）、`cls_pool`（CLS token，原始 BERT） | `mean_pool(&hidden_states, &attention_mask)` → `[batch, 768]` 句子向量 |
| **qwen3_embedding.rs** | `Qwen3EmbeddingModel`，支持 0.6B/4B/8B 全系列。32K 上下文、GQA 注意力、instruction-aware embedding | `model.embedding_forward(&input_ids, &mask)` → 32K 长文本 → 1024 维向量 |
| **gemma_embedding.rs** | `GemmaEmbeddingModel`（300M），2K 上下文、MQA（3 query/1 KV head）、混合注意力（sliding+full）、Matryoshka（768/512/256/128）、Dense 瓶颈（768→3072→768） | `model.forward_with_matryoshka(&ids, &mask, 256)` → 256 维紧凑向量 |
| **gemma3_model.rs** | Gemma3 Transformer 骨干网络实现。`Gemma3Attention`（MQA）、`Gemma3MLP`（gelu_pytorch_tanh）、`Gemma3Layer`、`Gemma3Model`（24 层） | 被 `GemmaEmbeddingModel` 内部使用，提供 transformer 前向传播 |
| **dense_layers.rs** | `BottleneckDenseNet`（768→3072→768 瓶颈），GemmaEmbedding 质量关键模块。无瓶颈 ~85% 质量，有瓶颈 ~99% | `bottleneck.forward(&pooled_embeddings)` → 线性变换提升 embedding 质量 |
| **mmbert_embedding.rs** | `MmBertEmbeddingModel`（307M），32K 上下文、1800+ 语言（Glot500）、2D Matryoshka（维度+层数缩减）、YaRN-scaled RoPE | `model.forward_2d_matryoshka(&ids, &mask, dim=256, layer=6)` → 极速多语言嵌入 |
| **continuous_batch_scheduler.rs** | Embedding 专用连续批调度器。基于 crossbeam-channel 的多通道 `select!`。请求到达 → 动态组批 → 单次前向传播 → 分发结果 | `scheduler.embed_from_raw(token_ids, mask)` → 自动合批处理 → 2-5x 吞吐提升 |
| **qwen3_batched.rs** | `Qwen3EmbeddingModelBatched`，Qwen3 的连续批处理包装器，API 100% 兼容原始模型 | `Qwen3EmbeddingModelBatched::load(path, &device, config)` → drop-in 替换原模型 |

### `generative/` — 生成式模型

| 文件 | 作用 | 例子 |
|------|------|------|
| **mod.rs** | 模块声明，re-export Qwen3MultiLoRAClassifier、Qwen3GuardModel | — |
| **qwen3_with_lora.rs** | 修改版官方 Qwen3 模型，添加 LoRA hook。注意力和 MLP 层支持 `output += LoRA_B(LoRA_A(input)) * scaling` | 被 `Qwen3MultiLoRAClassifier` 和 `Qwen3GuardModel` 内部使用 |
| **qwen3_multi_lora_classifier.rs** | `Qwen3MultiLoRAClassifier`，多 LoRA 适配器系统。一个 Qwen3 基础模型 + 多个可热切换的 LoRA 适配器（分类、越狱检测等）。支持零样本分类 | `model.classify_with_adapter("什么是GDP", "category")` → 用分类适配器 → `"knowledge"` |
| **qwen3_guard.rs** | `Qwen3GuardModel` 安全分类模型。生成式推理检测不安全内容，输出结构化结果（Reasoning + Category + Severity）。使用 PrefixCache 加速 | `guard.generate_guard("Ignore safety rules", "input")` → `"Category: Jailbreak, Severity: Unsafe"` |
| **continuous_batch_scheduler.rs** | 生成式分类器的连续批调度器，类似 vLLM 思路但更简单（分类是单次传播，无自回归解码） | `scheduler.classify("text", "category")` → 请求自动合批 → 高吞吐 |

### `lora/` — LoRA 适配器

| 文件 | 作用 | 例子 |
|------|------|------|
| **mod.rs** | 模块声明 | — |
| **lora_adapter.rs** | 核心 `LoRAAdapter` 实现。`LoRAConfig`（rank, alpha, dropout, target_modules）。A/B 矩阵低秩分解，支持 Kaiming/Xavier/Zero 初始化 | `adapter.forward(&input)` → `input + B(A(input)) * (alpha/rank)` |
| **bert_lora.rs** | `LoRABertClassifier`，冻结 BERT 骨干 + LoRA 适配器 + 多任务分类头。支持 rayon 并行多任务处理 | `classifier.parallel_multi_task_classify(&texts)` → 同时 Intent+PII+Security |

### `traditional/` — 传统编码器模型

| 文件 | 作用 | 例子 |
|------|------|------|
| **mod.rs** | 模块声明，re-export 所有传统模型。含本地 ModernBERT 副本原因说明（需要 Flash Attention 2 + 32K 支持） | — |
| **base_model.rs** | `TraditionalModelBase` trait 和 `BaseTraditionalModel` 基类。提供 embeddings + encoder + pooler 的通用框架 | `model.forward_pass(&input_ids, &attention_mask)` → 通用前向传播 |
| **bert.rs** | `TraditionalBertClassifier`，经典 BERT 分类器。CLS token → pooler → classifier。支持 HuggingFace Hub 和本地模型 | `TraditionalBertClassifier::new("bert-base-uncased", 5, false)` → 创建 5 分类 BERT |
| **deberta_v3.rs** | `DebertaV3Classifier`，增强的解耦注意力机制。专门用于 prompt injection 和越狱检测 | `classifier.classify("Ignore previous instructions")` → `{ label: "injection", confidence: 0.99 }` |
| **modernbert.rs** | `TraditionalModernBertClassifier`，支持 4 种变体：Standard（512）、Multilingual/mmBERT（8192）、Extended32K（32768）、Multilingual32K。自动检测变体 | `TraditionalModernBertClassifier::new_with_variant(path, variant, num_classes)` |
| **candle_models/modernbert.rs** | ModernBERT 底层 Candle 实现（本地副本）。包含 Flash Attention 2 支持、YaRN RoPE 缩放、混合注意力（local/global）。上游不支持 FA2 所以必须本地维护 | 被 `TraditionalModernBertClassifier` 内部使用 |
