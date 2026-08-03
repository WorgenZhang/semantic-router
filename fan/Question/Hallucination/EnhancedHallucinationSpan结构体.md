# EnhancedHallucinationSpan

```go
type EnhancedHallucinationSpan struct {
	Text                    string  `json:"text"`
	Start                   int     `json:"start"`
	End                     int     `json:"end"`
	HallucinationConfidence float32 `json:"hallucination_confidence"`
	NLILabel                string  `json:"nli_label"` // ENTAILMENT, NEUTRAL, or CONTRADICTION
	NLIConfidence           float32 `json:"nli_confidence"`
	Severity                int     `json:"severity"`    // 0-4: 0=low, 4=critical
	Explanation             string  `json:"explanation"` // Human-readable explanation
}
```


```go
 假设用户问："特斯拉是哪一年成立的？"，LLM 返回如下回答：

  "特斯拉由埃隆·马斯克于 2002年 在加利福尼亚创立，是全球最大的电动汽车制造商。"

  实际上特斯拉成立于 2003年，且创始人是 Martin Eberhard 和 Marc Tarpenning，马斯克是后来加入的。NLI 幻觉检测后会生成类似这样的结构：

  // 幻觉片段 1：错误的年份
  EnhancedHallucinationSpan{
      Text:                    "2002年",
      Start:                   30,    // 在回答文本中的起始字符位置
      End:                     35,    // 结束字符位置
      HallucinationConfidence: 0.92,  // 92% 置信度认为是幻觉
      NLILabel:                "CONTRADICTION",  // NLI 判定：与事实矛盾
      NLIConfidence:           0.95,  // NLI 模型 95% 确信是矛盾
      Severity:                3,     // 严重（事实性错误）
      Explanation:             "Tesla was founded in 2003, not 2002. The year is factually incorrect.",
  }

  // 幻觉片段 2：错误的创始人归属
  EnhancedHallucinationSpan{
      Text:                    "由埃隆·马斯克",
      Start:                   6,
      End:                     14,
      HallucinationConfidence: 0.88,
      NLILabel:                "CONTRADICTION",  // 与事实矛盾
      NLIConfidence:           0.91,
      Severity:                3,
      Explanation:             "Tesla was founded by Martin Eberhard and Marc Tarpenning. Elon Musk joined later as chairman and lead investor.",
  }

  // 非幻觉片段（作为对比）
  EnhancedHallucinationSpan{
      Text:                    "全球最大的电动汽车制造商",
      Start:                   42,
      End:                     55,
      HallucinationConfidence: 0.15,  // 低置信度，不太可能是幻觉
      NLILabel:                "ENTAILMENT",  // NLI 判定：与事实一致
      NLIConfidence:           0.82,
      Severity:                0,     // 低风险
      Explanation:             "Tesla is widely recognized as the world's largest EV manufacturer by volume.",
  }

  NLILabel 三种取值的含义

  ┌───────────────┬────────────────────────────┬────────────────────────┐
  │     标签      │            含义            │        典型场景        │
  ├───────────────┼────────────────────────────┼────────────────────────┤
  │ ENTAILMENT    │ 回答与已知事实一致         │ 正确的陈述             │
  ├───────────────┼────────────────────────────┼────────────────────────┤
  │ NEUTRAL       │ 无法判断真假（无足够证据） │ 主观观点、缺少参考资料 │
  ├───────────────┼────────────────────────────┼────────────────────────┤
  │ CONTRADICTION │ 回答与已知事实矛盾         │ 事实性错误，即幻觉     │
  └───────────────┴────────────────────────────┴────────────────────────┘

  Severity 等级

  ┌─────┬──────────┬────────────────────────────────────────────┐
  │ 值  │   级别   │                    说明                    │
  ├─────┼──────────┼────────────────────────────────────────────┤
  │ 0   │ low      │ 无关紧要的表述差异                         │
  ├─────┼──────────┼────────────────────────────────────────────┤
  │ 1   │ minor    │ 细微偏差，不影响核心含义                   │
  ├─────┼──────────┼────────────────────────────────────────────┤
  │ 2   │ moderate │ 有误导性但影响有限                         │
  ├─────┼──────────┼────────────────────────────────────────────┤
  │ 3   │ high     │ 明确的事实性错误                           │
  ├─────┼──────────┼────────────────────────────────────────────┤
  │ 4   │ critical │ 可能造成严重后果的错误（如医疗、法律信息） │
  └─────┴──────────┴────────────────────────────────────────────┘

  这些结构体最终会被 res_filter_hallucination.go 中的函数（如
  addHallucinationWarningHeaders、prependHallucinationWarningToBody）消费，将警告信息注入到返回给用户的响应头或响应体中
```
