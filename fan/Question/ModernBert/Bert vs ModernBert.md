# BERT-base vs ModernBERT-base：MoM 模型为什么用了两种不同的基座？

---

## 一、先看事实：每个 MoM 模型实际用的是哪种架构

通过提取每个 `models/mom-*/config.json` 中的关键字段，得到如下对照表：

| 模型 | config.json 中的 architectures | model_type | 层数 | 词表大小 | 最大上下文 | 位置编码 |
|------|-------------------------------|------------|------|---------|-----------|---------|
| mom-domain-classifier | `BertForSequenceClassification` | bert | 12 | 30,522 | 512 | 绝对位置 |
| mom-pii-classifier | `BertForTokenClassification` | bert | 12 | 30,522 | 512 | 绝对位置 |
| mom-embedding-light | `BertModel` | bert | 12 | 30,522 | 512 | 绝对位置 |
| **mom-jailbreak-classifier** | `ModernBertForSequenceClassification` | **modernbert** | **22** | **50,368** | **8192** | **RoPE** |
| **mom-halugate-sentinel** | `ModernBertForSequenceClassification` | **modernbert** | **22** | **50,368** | **8192** | **RoPE** |
| **mom-halugate-detector** | `ModernBertForTokenClassification` | **modernbert** | **22** | **50,368** | **8192** | **RoPE** |
| **mom-halugate-explainer** | `ModernBertForSequenceClassification` | **modernbert** | **22** | **50,368** | **2048** | **RoPE** |
| **mom-feedback-detector** | `ModernBertForSequenceClassification` | **modernbert** | **22** | **50,368** | **8192** | **RoPE** |
| mom-embedding-pro | `Qwen3ForCausalLM` | qwen3 | 28 | 151,669 | 32768 | RoPE |

**结论**：项目中实际存在 3 种基座架构：BERT、ModernBERT 和 Qwen3。

> 注意：`registry.go` 中 sentinel 被描述为 "BERT-base (110M)"，但其 config.json 实际是 `ModernBertForSequenceClassification`（22 层, 149M）。说明 sentinel 已经升级到了 ModernBERT，文档描述滞后。

---

## 二、BERT-base 与 ModernBERT-base 的架构区别

### 对比表

| 特性 | BERT-base (2018) | ModernBERT-base (2024) |
|------|------------------|------------------------|
| **参数量** | 110M | 149M |
| **层数** | 12 | 22 |
| **隐藏维度** | 768 | 768 |
| **注意力头数** | 12 | 12 |
| **位置编码** | 绝对位置编码 (Absolute) | **RoPE** (旋转位置编码) |
| **最大上下文** | **512 tokens** | **8192 tokens** |
| **词表大小** | 30,522 | 50,368 |
| **注意力机制** | 全局注意力 (Full Attention) | **交替局部/全局注意力** |
| **局部注意力窗口** | N/A | 128 tokens |
| **全局注意力频率** | N/A | 每 3 层一次 (`global_attn_every_n_layers: 3`) |
| **激活函数** | GELU | GELU |
| **Flash Attention** | 不支持 | **支持** |
| **Dropout** | 0.1（训练期） | 0.0（推理更稳定） |
| **训练数据** | BookCorpus + Wikipedia | **2T tokens 多源数据** |

### 关键架构差异详解

#### 1. 位置编码：绝对 vs RoPE

```
BERT:       token_embedding + position_embedding[0..511]  ← 固定 512 个位置
ModernBERT: token_embedding + RoPE(theta=160000)          ← 可扩展到任意长度
```

BERT 的绝对位置编码把位置信息作为可学习参数写死在权重里，最多 512 个位置。ModernBERT 使用 RoPE（Rotary Position Embedding），位置信息编码在注意力计算中，天然支持长上下文。

代码证据 (`models/mom-halugate-sentinel/config.json`):
```json
"global_rope_theta": 160000.0,
"local_rope_theta": 10000.0
```

#### 2. 注意力模式：全局 vs 交替局部/全局

```
BERT (12层):      [全局][全局][全局][全局][全局][全局][全局][全局][全局][全局][全局][全局]
ModernBERT (22层): [局部][局部][全局][局部][局部][全局][局部][局部][全局]... (每3层一次全局)
```

ModernBERT 的大部分层使用 128 token 的局部滑动窗口注意力（O(n)），每 3 层插入一次全局注意力（O(n^2)），在长文本上比 BERT 的全层全局注意力高效得多。

代码证据 (`models/mom-halugate-sentinel/config.json`):
```json
"global_attn_every_n_layers": 3,
"local_attention": 128
```

#### 3. 推理实现差异

在 Candle Rust 层，BERT 和 ModernBERT 有完全独立的实现：

| 文件 | 内容 |
|------|------|
| `candle-binding/src/model_architectures/traditional/bert.rs` | BERT 实现，使用 `candle_transformers::models::bert::BertModel` |
| `candle-binding/src/model_architectures/traditional/modernbert.rs` | ModernBERT 实现，自带交替注意力、RoPE、变体检测 |

---

## 三、为什么不同模型选择了不同的基座？

### 历史演进原因

这是一个 **渐进式迁移** 的过程，不是一开始就设计好的：

```
第一阶段 (早期): 所有分类器基于 BERT-base + LoRA
    → mom-domain-classifier  (BERT + LoRA)
    → mom-pii-classifier     (BERT + LoRA)
    → mom-jailbreak-classifier (BERT + LoRA, 后来升级)

第二阶段 (中期): 新模型直接用 ModernBERT
    → mom-halugate-* 系列   (全部 ModernBERT)
    → mom-feedback-detector  (ModernBERT)

第三阶段 (现在): 旧模型逐步迁移
    → mom-jailbreak-classifier 已迁移到 ModernBERT
    → mom-halugate-sentinel 已迁移到 ModernBERT
    → mom-domain-classifier / mom-pii-classifier 仍为 BERT (+ LoRA adapter)
```

### 每个模型的选择逻辑

| 模型 | 选 BERT 还是 ModernBERT | 原因 |
|------|------------------------|------|
| domain-classifier | **BERT** | 最早训练的模型，使用 LoRA adapter 微调，512 token 足够（查询通常很短） |
| pii-classifier | **BERT** | Token 级 BIO 标注任务，512 token 足够，LoRA adapter 效率高 |
| jailbreak-classifier | **ModernBERT** | 越狱攻击 prompt 可能很长（注入大段指令），需要更长上下文；且安全任务需要更高准确度 |
| halugate-sentinel | **ModernBERT** | 虽然只做二分类，但需要理解较长的事实声明上下文 |
| halugate-detector | **ModernBERT** | Token 级幻觉 span 检测，LLM 回答通常很长，**8192 token 上下文是刚需** |
| halugate-explainer | **ModernBERT** | NLI 需要同时理解 premise 和 hypothesis，长上下文有优势 |
| feedback-detector | **ModernBERT** | 较新的模型，直接采用更优架构 |
| embedding-light | **BERT (MiniLM)** | 极致轻量化需求，33M 参数，速度优先 |

### 核心决策因素

```
需要长上下文 (>512 tokens)?
  ├── 是 → ModernBERT (最大 8192 tokens)
  │        例: 幻觉检测需要分析完整 LLM 回答
  └── 否 → 看其他因素
        ├── 需要 LoRA 高效微调? → BERT (LoRA 生态成熟)
        │   例: domain/pii 用 LoRA adapter, ~4MB/任务
        ├── 需要极致速度? → BERT/MiniLM (更少参数)
        │   例: embedding-light 只有 33M
        └── 新训练的模型? → ModernBERT (默认选择更优架构)
            例: feedback-detector, halugate 系列
```

---

## 四、系统如何判断用哪种推理引擎？

### 步骤 1: Go 层根据配置选择 CGO 初始化函数

在 `candle-binding/semantic-router.go` 中，不同模型调用不同的 C 函数：

| 配置项 | 调用的 CGO 函数 | 对应 Rust 实现 |
|--------|----------------|---------------|
| `classifier.category_model` | `init_classifier()` | `TraditionalBertClassifier` (BERT) |
| `classifier.pii_model` | `init_pii_classifier()` | `TraditionalBertClassifier` (BERT) |
| `prompt_guard` | `init_modernbert_jailbreak_classifier()` | `TraditionalModernBertClassifier` (ModernBERT) |
| `hallucination_mitigation.fact_check_model` | `init_fact_check_classifier()` | `TraditionalModernBertClassifier` (ModernBERT) |
| `hallucination_mitigation.hallucination_model` | 专用幻觉检测初始化 | `TraditionalModernBertClassifier` (ModernBERT) |
| `hallucination_mitigation.nli_model` | 专用 NLI 初始化 | `TraditionalModernBertClassifier` (ModernBERT) |
| feedback-detector | `init_feedback_detector()` | `TraditionalModernBertClassifier` (ModernBERT) |

### 步骤 2: Rust 层根据 config.json 自动检测

对于 ModernBERT，Rust 层还会自动检测变体（`modernbert.rs:102-210`）：

```rust
pub fn detect_from_config(config_path: &str) -> Result<Self, candle_core::Error> {
    let vocab_size = config_json.get("vocab_size")...;
    let max_position_embeddings = config_json.get("max_position_embeddings")...;
    let global_rope_theta = config_json.get("global_rope_theta")...;

    // vocab_size >= 200000 → mmBERT (多语言版)
    // training_config 有 YaRN + 32K → Extended32K 或 Multilingual32K
    // 否则 → Standard ModernBERT
}
```

### 步骤 3: 自动发现机制

`model_discovery.go` 扫描 `models/` 目录时，按架构优先级排序：

```go
architecturePriority := []string{"bert", "roberta", "modernbert"}
```

LoRA 模型优先于 Legacy 模型，BERT LoRA 优先于 ModernBERT（因为 LoRA 生态更成熟）。

---

## 五、未来趋势：全面迁移到 ModernBERT / mmBERT-32K

从 `registry.go` 中可以看到，项目已经注册了大量 **mmBERT-32K** 变体（32K 上下文、多语言、YaRN RoPE）：

```
models/mmbert32k-intent-classifier-lora      (替代 mom-domain-classifier)
models/mmbert32k-pii-detector-lora           (替代 mom-pii-classifier)
models/mmbert32k-jailbreak-detector-lora     (替代 mom-jailbreak-classifier)
models/mmbert32k-factcheck-classifier-lora   (替代 mom-halugate-sentinel)
models/mmbert32k-feedback-detector-lora      (替代 mom-feedback-detector)
```

这意味着最终所有 MoM 分类模型都会迁移到基于 ModernBERT 架构的 mmBERT-32K，统一到：
- **307M 参数**
- **32K 上下文**
- **1800+ 语言支持**
- **LoRA adapter 微调**

届时 BERT-base 版本将被完全替代。

---

## 六、总结

| 问题 | 回答 |
|------|------|
| 为什么有两种基座？ | **历史演进**——早期模型用 BERT + LoRA，新模型直接用 ModernBERT |
| BERT 的优势？ | 轻量（110M, 12层）、LoRA 生态成熟、512 token 短文本够用 |
| ModernBERT 的优势？ | 更深（22层, 149M）、长上下文（8192）、RoPE、交替注意力、更准确 |
| 谁决定用哪种？ | Go 配置层根据模型用途调用不同的 CGO 函数，Rust 层根据 config.json 自动适配 |
| 未来方向？ | 全面迁移到 mmBERT-32K（307M, 32K 上下文, 多语言） |
