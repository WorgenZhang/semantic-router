```markdown
现在信息完整了，以下是 isNLIEnabledForDecision 的完整链路梳理。

  ---
  isNLIEnabledForDecision 完整链路

  一、全景流程图

  ┌─────────────────────────────────────────────────────────────────────────┐
  │  ① YAML 配置文件 / Kubernetes CRD                                        │
  │                                                                         │
  │  decisions:                                                             │
  │    - name: "medical_qa"                                                 │
  │      plugins:                                                           │
  │        - type: hallucination                                            │
  │          configuration:                                                 │
  │            enabled: true                                                │
  │            use_nli: true          ← NLI 开关在这里定义                     │
  │            hallucination_action: body                                   │
  └──────────────────────┬──────────────────────────────────────────────────┘
                         │ YAML 解析
                         ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  ② Go 结构体 (config.go)                                                │
  │                                                                         │
  │  Decision{                                                              │
  │    Name: "medical_qa",                                                  │
  │    Plugins: []DecisionPlugin{                                           │
  │      {                                                                  │
  │        Type: "hallucination",                                           │
  │        Configuration: map[string]interface{}{                           │
  │          "enabled":  true,                                              │
  │          "use_nli":  true,         ← 此时是 map 中的 key-value           │
  │          "hallucination_action": "body",                                │
  │        },                                                               │
  │      },                                                                 │
  │    },                                                                   │
  │  }                                                                      │
  └──────────────────────┬──────────────────────────────────────────────────┘
                         │ 请求阶段：Decision 匹配
                         ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  ③ performDecisionEvaluation (req_filter_classification.go:176)         │
  │                                                                         │
  │  信号评估 → 匹配到 "medical_qa" decision                                  │
  │  ctx.VSRSelectedDecision = result.Decision  ← 将完整 Decision 存入上下文   │
  └──────────────────────┬──────────────────────────────────────────────────┘
                         │ 响应阶段：幻觉检测
                         ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  ④ performHallucinationDetection (res_filter_hallucination.go:35)       │
  │                                                                         │
  │  useNLI := r.isNLIEnabledForDecision(ctx.VSRSelectedDecision)           │
  └──────────────────────┬──────────────────────────────────────────────────┘
                         │
                         ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  ⑤ isNLIEnabledForDecision (res_filter_hallucination.go:194)           │
  │                                                                         │
  │  → decision.GetHallucinationConfig()                                    │
  │     │                                                                   │
  │     ▼  GetPluginConfig("hallucination")     (config.go:2304)            │
  │        遍历 decision.Plugins，找到 Type == "hallucination" 的插件          │
  │        返回 Configuration (map[string]interface{})                       │
  │     │                                                                   │
  │     ▼  unmarshalPluginConfig(config, result) (config.go:2472)           │
  │        将 map[string]interface{} 反序列化为 HallucinationPluginConfig      │
  │        map → JSON marshal → JSON unmarshal → 结构体                      │
  │     │                                                                   │
  │     ▼  返回 &HallucinationPluginConfig{                                  │
  │            Enabled: true,                                               │
  │            UseNLI:  true,          ← 最终读到的值                         │
  │            HallucinationAction: "body",                                 │
  │        }                                                                │
  │                                                                         │
  │  return halConfig.UseNLI   →  true                                      │
  └──────────────────────┬──────────────────────────────────────────────────┘
                         │
                         ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  ⑥ 分支选择 (res_filter_hallucination.go:42-46)                         │
  │                                                                         │
  │  if useNLI {                                                            │
  │      return r.performHallucinationDetectionWithNLI(ctx, content)        │
  │      → Classifier.DetectHallucinationWithNLI()                          │
  │      → 返回带 NLI 标签、置信度、严重性、解释 的增强 spans                    │
  │  } else {                                                               │
  │      Classifier.DetectHallucination()                                   │
  │      → 返回基础的 UnsupportedSpans                                       │
  │  }                                                                      │
  └─────────────────────────────────────────────────────────────────────────┘

  ---
  二、核心代码跳转路径

  ┌──────┬──────────────────────────────────┬───────────────────────────────────────────────────────────────────────┐
  │ 步骤 │            文件:行号             │                               做了什么                                │
  ├──────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
  │ 1    │ config.go:1872                   │ Decision struct 定义，含 Plugins []DecisionPlugin                     │
  ├──────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
  │ 2    │ config.go:2117                   │ DecisionPlugin struct: Type string + Configuration interface{}        │
  ├──────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
  │ 3    │ config.go:2185                   │ HallucinationPluginConfig struct 定义 UseNLI bool (yaml tag: use_nli) │
  ├──────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
  │ 4    │ req_filter_classification.go:176 │ 请求阶段匹配 decision → ctx.VSRSelectedDecision = result.Decision     │
  ├──────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
  │ 5    │ res_filter_hallucination.go:35   │ 响应阶段调 isNLIEnabledForDecision(ctx.VSRSelectedDecision)           │
  ├──────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
  │ 6    │ config.go:2304                   │ GetPluginConfig("hallucination") — 遍历 Plugins 找 Type 匹配          │
  ├──────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
  │ 7    │ config.go:2471-2472              │ unmarshalPluginConfig — map → JSON → HallucinationPluginConfig        │
  ├──────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
  │ 8    │ res_filter_hallucination.go:208  │ return halConfig.UseNLI                                               │
  └──────┴──────────────────────────────────┴───────────────────────────────────────────────────────────────────────┘

  ---
  三、具体例子

  场景：医疗问答 Decision，启用 NLI 增强幻觉检测

  YAML 配置：

  decisions:
    - name: "medical_qa"
      description: "医疗健康相关问题"
      rules:
        operator: OR
        items:
          - type: domain
            name: medical
      modelRefs:
        - model: gpt-4o
      plugins:
        - type: hallucination
          configuration:
            enabled: true
            use_nli: true                        # ← 启用 NLI
            hallucination_action: body
            include_hallucination_details: true

  用户请求：

  {
    "model": "auto",
    "messages": [
      {"role": "user", "content": "阿司匹林的最大日剂量是多少？"},
      {"role": "assistant", "tool_calls": [{"id": "call_1", "function": {"name": "drug_db_lookup", "arguments": "{\"drug\":\"aspirin\"}"}}]},
      {"role": "tool", "tool_call_id": "call_1", "content":
  "阿司匹林：常规剂量300-600mg/次，每日最大剂量4g。用于解热镇痛时建议每4-6小时一次。禁忌：消化道溃疡、出血性疾病。"},
      {"role": "user", "content": "总结一下阿司匹林的用法"}
    ]
  }

  运行时数据流：

  ① 请求阶段
     信号评估 → 匹配 domain=medical → 选中 "medical_qa" decision
     ctx.VSRSelectedDecision = &Decision{Name: "medical_qa", Plugins: [...]}

     fact_check 信号匹配 → ctx.FactCheckNeeded = true
     tool message 提取 → ctx.ToolResultsContext =
       "阿司匹林：常规剂量300-600mg/次，每日最大剂量4g。用于解热镇痛时建议每4-6小时一次。禁忌：消化道溃疡、出血性疾病。"

  ② LLM 返回响应（假设有幻觉）
     assistantContent = "阿司匹林常规剂量300-600mg/次，每日最大剂量6g，每2小时服用一次。"
                                                            ↑ 编造    ↑ 编造

  ③ 响应阶段 — isNLIEnabledForDecision
     decision.GetHallucinationConfig()
     → GetPluginConfig("hallucination")
       → 遍历 Plugins, 找到 Type=="hallucination"
       → 返回 Configuration: {"enabled":true, "use_nli":true, ...}
     → unmarshalPluginConfig → HallucinationPluginConfig{UseNLI: true}
     → return halConfig.UseNLI = true  ✅

  ④ 进入 NLI 增强检测分支
     performHallucinationDetectionWithNLI()
     → Classifier.DetectHallucinationWithNLI(
         premise:    "阿司匹林：常规剂量300-600mg/次，每日最大剂量4g...",  // ToolResultsContext
         userQuery:  "总结一下阿司匹林的用法",                             // UserContent
         hypothesis: "阿司匹林常规剂量300-600mg/次，每日最大剂量6g...",      // LLM回答
       )

  NLI 增强检测结果（对比基础模式）：

  基础模式 (use_nli: false) 只返回：
    HallucinationDetected: true
    UnsupportedSpans: ["每日最大剂量6g", "每2小时服用一次"]

  NLI 增强模式 (use_nli: true) 返回：
    HallucinationDetected: true
    Confidence: 0.92
    Spans:
      [1] Text: "每日最大剂量6g"
          NLILabel:    "CONTRADICTION"     ← 与前提直接矛盾（应为4g）
          NLIConfidence: 0.97
          Severity:     3 (high)
          Explanation:  "工具数据明确指出最大日剂量为4g，回答声称6g与之矛盾"

      [2] Text: "每2小时服用一次"
          NLILabel:    "CONTRADICTION"     ← 与前提直接矛盾（应为4-6小时）
          NLIConfidence: 0.94
          Severity:     3 (high)
          Explanation:  "工具数据建议每4-6小时一次，回答声称每2小时与之矛盾"

  最终写入响应 body 的警告（因为 hallucination_action: body + include_hallucination_details: true）：

  [Hallucination Warning] This response may contain unsupported claims (confidence: 92%).
  Detailed analysis:
   [1] "每日最大剂量6g" (NLI: CONTRADICTION, confidence: 97%, severity: high) - 工具数据明确指出最大日剂量为4g，回答声称6g与之矛盾
   [2] "每2小时服用一次" (NLI: CONTRADICTION, confidence: 94%, severity: high) - 工具数据建议每4-6小时一次，回答声称每2小时与之矛盾
  Please verify the information independently.

  阿司匹林常规剂量300-600mg/次，每日最大剂量6g，每2小时服用一次。

  ---
  四、如果 use_nli: false（或省略），区别是什么

  ┌────────────┬────────────────────────┬──────────────────────────────────────────┐
  │            │     use_nli: false     │              use_nli: true               │
  ├────────────┼────────────────────────┼──────────────────────────────────────────┤
  │ 检测函数   │ DetectHallucination    │ DetectHallucinationWithNLI               │
  ├────────────┼────────────────────────┼──────────────────────────────────────────┤
  │ 返回 spans │ 纯文本列表 []string    │ 带标签的结构化 EnhancedHallucinationSpan │
  ├────────────┼────────────────────────┼──────────────────────────────────────────┤
  │ NLI 标签   │ 无                     │ ENTAILMENT / NEUTRAL / CONTRADICTION     │
  ├────────────┼────────────────────────┼──────────────────────────────────────────┤
  │ 严重性分级 │ 无                     │ 0-4（low → critical）                    │
  ├────────────┼────────────────────────┼──────────────────────────────────────────┤
  │ 解释说明   │ 无                     │ 人类可读的矛盾解释                       │
  ├────────────┼────────────────────────┼──────────────────────────────────────────┤
  │ 性能开销   │ 较低（仅幻觉检测模型） │ 较高（幻觉检测 + NLI 模型推理）          │
  └────────────┴────────────────────────┴──────────────────────────────────────────┘

  use_nli 本质上是在基础幻觉检测之上叠加了一个 NLI（自然语言推理）模型，用于对每个疑似幻觉片段做 premise-hypothesis 级别的蕴含关系判断，给出更精确的矛盾分析。
```
