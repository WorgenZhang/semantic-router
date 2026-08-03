# `candle-binding/src/ffi/validation.rs` 分析

## `validation.rs` 的作用

这是 **双路架构 (LoRA / Traditional) 的 FFI 参数验证层**，在 Go 通过 CGO 调用 Rust 推理之前，对输入参数做安全检查，防止无效数据进入 GPU 推理管道。

核心设计：**path_type 区分两条路径**（`0` = Traditional BERT 模型，`1` = LoRA adapter），每条路径有不同的验证规则。

---

### 例子 1：文本输入验证 (`validate_text_input`)

```
// Traditional 路径 — 允许短文本
validate_text_input("Hi", 0)          → ✅ 通过

// LoRA 路径 — 短文本会被拒绝（<10 字符不适合 LoRA）
validate_text_input("Hi", 1)          → ❌ ERROR_LORA_SPECIFIC
                                        "Text may be too short for optimal LoRA processing"
                                        建议: "Consider using Traditional path for very short texts"

// 空指针
validate_text_input(NULL, 0)          → ❌ ERROR_NULL_POINTER

// 超长文本（>10000 字符）
validate_text_input(very_long_text, 0) → ❌ ERROR_TEXT_TOO_LONG
```

**为什么**：LoRA 模型处理极短文本没有优势，应该走 Traditional BERT 路径更高效。

---

### 例子 2：批处理验证 (`validate_batch_input`)

```
// Traditional 路径 — 大批量会警告
validate_batch_input(texts, 200, 0)   → ❌ ERROR_TRADITIONAL_SPECIFIC
                                        "Large batch sizes may cause memory issues"
                                        建议: 减小 batch 或改用 LoRA 路径

// LoRA 路径 — 单条不划算（浪费并行优势）
validate_batch_input(texts, 1, 1)     → ❌ ERROR_LORA_SPECIFIC
                                        "Single item batches don't utilize LoRA parallel processing"
                                        建议: 改用 Traditional 路径或增大 batch
```

**为什么**：Traditional BERT 模型 batch > 100 容易 OOM；LoRA 的优势在于并行，单条请求不值得走 LoRA。

---

### 例子 3：置信度阈值验证 (`validate_confidence_threshold`)

```
// Traditional 路径 — 阈值 0.3 太低
validate_confidence_threshold(0.3, 0) → ❌ "confidence threshold too low"
                                        建议: >= 0.5

// LoRA 路径 — 阈值 0.6 不够
validate_confidence_threshold(0.6, 1) → ❌ "confidence threshold too low"
                                        建议: >= 0.8
```

**为什么**：LoRA 路由需要更高置信度（0.8+）才值得加载 adapter，否则应该 fallback 到通用模型。

---

### 例子 4：模型路径验证 (`validate_model_path`)

```
// 路径名包含 "bert" → Traditional ✅
validate_model_path("/models/modernbert-base", 0) → ✅

// 路径名包含 "lora" → LoRA ✅  
validate_model_path("/models/lora-adapter-v2", 1) → ✅

// 路径不匹配 → 警告
validate_model_path("/models/random-model", 1)    → ❌ "doesn't appear to be a LoRA model"
```

---

### 总结

| 验证函数 | 验证什么 | Traditional 规则 | LoRA 规则 |
|---|---|---|---|
| `validate_text_input` | 文本合法性 | 无控制字符 | 长度 >= 10 |
| `validate_batch_input` | 批量大小 | <= 100 | > 1 |
| `validate_confidence_threshold` | 置信度范围 | >= 0.5 | >= 0.8 |
| `validate_model_path` | 路径合理性 | 含 bert/traditional | 含 lora/adapter |
| `validate_memory_parameters` | 内存分配 | <= 1GB, 对齐为 2 的幂 | 同左 |

本质上是一个 **防御性编程层**，在 FFI 边界（Go → Rust）拦截非法参数，避免 GPU 推理时 crash 或产生无意义结果。
