```markdown
为什么ENTAILMENT 且 confidence >= nli_entailment_threshold → 移除(误报) 不是confidence越高越说明明显出现幻觉吗 为什么还会认为误报

⏺ 这是一个很好的问题。关键在于理解 NLI 三个标签的语义方向。

  NLI 的 premise-hypothesis 框架

  在这个 pipeline 中，NLI 模型的输入是：

  premise   (前提)  = ToolResultsContext + UserQuestion  ← 事实基础/上下文
  hypothesis (假设) = 被标记为幻觉的 span 文本            ← LLM 回答的片段

  NLI 问的问题是："前提（上下文）是否支持假设（这个片段）？"

  ┌───────────────┬─────────────────┬────────────────────────────────┐
  │   NLI 标签    │      含义       │      confidence 高意味着       │
  ├───────────────┼─────────────────┼────────────────────────────────┤
  │ ENTAILMENT    │ 前提 支持 假设  │ 上下文确实能证实这个片段       │
  ├───────────────┼─────────────────┼────────────────────────────────┤
  │ NEUTRAL       │ 前提 与假设无关 │ 上下文找不到关于这个片段的证据 │
  ├───────────────┼─────────────────┼────────────────────────────────┤
  │ CONTRADICTION │ 前提 与假设矛盾 │ 上下文明确反驳这个片段         │
  └───────────────┴─────────────────┴────────────────────────────────┘

  为什么 ENTAILMENT + 高置信度 = 误报

  举个具体例子：

  上下文 (premise):
    "阿司匹林：常规剂量300-600mg/次，每日最大剂量4g"

  LLM 回答:
    "阿司匹林常规剂量是300-600mg/次，每日最大剂量6g，每2小时服用一次"

  Step 1: 模型 B（Token 级幻觉检测器） 对每个 token 做二分类：

  "阿司匹林"     → SUPPORTED    ✓
  "常规剂量"     → SUPPORTED    ✓
  "300-600mg/次" → SUPPORTED    ✓  ← 正确，有上下文支撑
  "每日最大剂量"  → SUPPORTED    ✓
  "6g"           → HALLUCINATED ✗  ← 正确！上下文说4g
  "每2小时"      → HALLUCINATED ✗  ← 正确！上下文说4-6小时
  "服用一次"     → HALLUCINATED ✗  ← 可能误标！"服用一次"本身不矛盾

  模型 B 是 token 级别的分类器，它逐 token 判断，粒度很细但容易过度敏感。比如 "300-600mg/次" 这个 span，虽然上下文完全支撑，但 token 级模型有时也可能把它标为
  HALLUCINATED（因为 token 级别的局部判断不够精准）。

  Step 2: 模型 C（NLI） 对每个被标记的 span 做验证：

  span "6g":
    NLI(premise="阿司匹林：最大剂量4g", hypothesis="6g")
    → CONTRADICTION, confidence=0.97
    → ✅ 确认是幻觉（上下文说4g，回答说6g，矛盾）

  span "每2小时服用一次":
    NLI(premise="阿司匹林：每4-6小时一次", hypothesis="每2小时服用一次")
    → CONTRADICTION, confidence=0.94
    → ✅ 确认是幻觉

  假设模型B误标了 "300-600mg/次":
    NLI(premise="阿司匹林：常规剂量300-600mg/次", hypothesis="300-600mg/次")
    → ENTAILMENT, confidence=0.98
    → ❌ 模型B搞错了！上下文明明支持这个说法！→ 移除（误报）

  核心逻辑

  模型 B 说: "这个 span 是幻觉！"
  模型 C 说: "等一下，让我验证一下..."

    CONTRADICTION → "没错，上下文确实反驳了它" → 确认幻觉 (severity=4)
    NEUTRAL       → "上下文里找不到相关信息"   → 可能是编造 (severity=2)
    ENTAILMENT    → "上下文明明支持这个说法！" → 模型 B 搞错了，是误报 → 移除

  所以：

  - ENTAILMENT confidence 高 ≠ "幻觉程度高"
  - ENTAILMENT confidence 高 = "上下文对这个片段的支持程度高" = 这个片段不是幻觉

  模型 C 的角色是纠错者：模型 B（token 级）可能过于敏感，模型 C（句子级 NLI）提供第二意见，把被误标的 span 筛掉。这就是代码中 enable_nli_filtering
  这个配置项存在的原因——用 NLI 的 ENTAILMENT 判断来过滤模型 B 的假阳性。
```
