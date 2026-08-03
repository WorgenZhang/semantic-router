不是“无脑”运行所有分类器，而是**“按需并行”**。

系统会先扫描你的 `decisions` 列表，统计出所有规则中**实际用到**了哪些信号类型。只有被用到的信号分类器才会被启动；一旦启动，它们是**并行（并发）**运行的，互不阻塞。

由于你的 `config.yaml` 配置非常丰富，几乎涉及了所有信号类型，所以在实际运行时，确实是多个分类器齐头并进。

以下用一个具体的例子来演示全流程。

---

### 场景示例
**用户 Query**: `"Please explain the concept of Quantum Entanglement specifically."`
**(请具体解释量子纠缠的概念)**

### 1. 预处理阶段：确定“作战计划”
系统启动时或配置加载时，已知 `decisions` 依赖以下信号：
*   Preference (优先级 200 规则用到)
*   Embedding (优先级 180 规则用到)
*   Domain (优先级 100 规则用到)
*   Keyword (优先级 100 规则用到)
*   Language (优先级 80 规则用到)

### 2. 信号计算阶段 (并行执行)
请求到达后，系统通过 Goroutines 同时通过以下路径处理 Query。总耗时取决于**最慢**的那条路径。

| 并行路径 | 动作 | 计算过程 | 结果 | 耗时估算 |
| :--- | :--- | :--- | :--- | :--- |
| **Path A (Keywords)** | 正则匹配 | 扫描 "math_keywords" (calculate...), "code_keywords" (function...) | `无匹配` | < 1ms |
| **Path B (Embeddings)** | 向量计算 | 1. 将 Query 转向量<br>2. 与 `deep_thinking` 候选词 ("explain your reasoning" 等) 算相似度 | **命中 `deep_thinking`**<br>(相似度 0.82 > 阈值 0.75) | ~15ms |
| **Path C (Domains)** | 模型分类 | 调用 ModernBERT 模型判断领域 | **命中 `physics`**<br>(置信度 0.98) | ~25ms |
| **Path D (Preferences)** | **LLM 调用** | 调用外部小模型分析意图 (Prompt: "Is this code generation?...") | **命中 `other`**<br>(意图属于解释概念，非写代码) | ~400ms |
| **Path E (Language)** | 语言检测 | 统计字符特征 | **命中 `en`** | < 1ms |

> **注意**: Path D (Preferences) 通常是最慢的，因为要网络请求外部 LLM。其他本地模型（Embedding/Domain）非常快。

### 3. 结果汇总
所有线程返回，汇总出的 **Signal Context** 如下：
```json
{
  "keywords": [],
  "embeddings": ["deep_thinking"],
  "domains": ["physics"],
  "preferences": ["other"],
  "language": "en"
}
```

### 4. 决策匹配阶段 (顺序执行)
路由引擎拿着上面的结果，按 `priority` 从高到低逐个检查 `config.yaml` 中的规则：

1.  **检查 Priority 200: `preference_code_generation`**
    *   **规则**: 需要 `preferences` 包含 `"code_generation"`。
    *   **现状**: 实际是 `"other"`。
    *   **结果**: ❌ Mismatch (跳过)。

2.  **检查 Priority 200: `preference_bug_fixing`**
    *   **规则**: 需要 `preferences` 包含 `"bug_fixing"`。
    *   **现状**: 实际是 `"other"`。
    *   **结果**: ❌ Mismatch (跳过)。

3.  **检查 Priority 180: `deep_thinking_route`**
    *   **规则**: 需要 `embeddings` 包含 `"deep_thinking"`。
    *   **现状**: 实际包含 `"deep_thinking"` (在 Path B 中命中)。
    *   **结果**: ✅ **HIT (命中!)**

### 5. 最终执行
系统选定 `deep_thinking_route`，并执行该规则定义的动作：
*   **目标模型**: `openai/gpt-oss-120b`
*   **参数调整**: 强制开启 `use_reasoning: true` (因为解释量子纠缠需要推理)，并设置 `reasoning_effort: "high"`。
*   **Prompt 注入**: 自动在用户 Query 前插入插件定义的 System Prompt：
    > *"Think deeply and systematically. Break down the problem, analyze each step carefully..."*

### 总结
虽然是“并行计算”，但**短板效应**存在——整个路由的延迟主要取决于最慢的那个分类器（通常是 Preference LLM）。

**优化建议**：
如果你希望极致的低延迟，可以在配置中移除 `preferences` 相关的 `decisions`，这样系统检测到不需要 Preference 信号，就不会去调用那个最慢的外部 LLM，路由速度会从 ~400ms 提升到 ~30ms (纯本地模型推理时间)。
