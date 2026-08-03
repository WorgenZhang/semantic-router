```markdown
@src/semantic-router/pkg/extproc/res_filter_hallucination.go  performHallucinationDetection函数具体做了哪些逻辑，请以一个例子帮我串一下
  ⎿  Read src/semantic-router/pkg/extproc/res_filter_hallucination.go (546 lines)

⏺ performHallucinationDetection 全流程示例

  假设用户发了这样一个请求：

  用户：2024年诺贝尔物理学奖颁给了谁？

  LLM 回答：2024年诺贝尔物理学奖颁给了 Geoffrey Hinton 和 John Hopfield，以表彰他们在2018年发明的深度卷积神经网络方面的贡献。

  （实际获奖原因是"机器学习的基础性发现和发明"，年份和具体技术都有误。）

  ---
  第一步：前置条件检查（第 23-25 行）

  if !r.shouldPerformHallucinationDetection(ctx) {
      return nil
  }

  检查是否满足幻觉检测条件（如：配置是否启用、请求是否需要 fact-check 等）。

  本例中 ctx.FactCheckNeeded = true（被上游分类器判定为事实性问题），继续。

  ---
  第二步：提取 LLM 回答内容（第 28-32 行）

  assistantContent := extractAssistantContentFromResponse(responseBody)

  从 OpenAI 格式的 JSON 响应中提取 choices[0].message.content：

  {
    "choices": [{
      "message": {
        "content": "2024年诺贝尔物理学奖颁给了 Geoffrey Hinton 和 John Hopfield，以表彰他们在2018年发明的深度卷积神经网络方面的贡献。"
      }
    }]
  }

  提取出 assistantContent = "2024年诺贝尔物理学奖颁给了..."

  ---
  第三步：判断是否启用 NLI 增强模式（第 35-46 行）

  useNLI := r.isNLIEnabledForDecision(ctx.VSRSelectedDecision)

  查询当前路由决策的 HallucinationConfig.UseNLI 配置：

  - useNLI = true → 走 performHallucinationDetectionWithNLI（更详细，有逐句分析）
  - useNLI = false → 走基础模式 Classifier.DetectHallucination

  路径 A：基础模式（第 49-53 行）

  result, err := r.Classifier.DetectHallucination(
      ctx.ToolResultsContext,  // 工具返回的参考资料（如搜索结果）
      ctx.UserContent,         // "2024年诺贝尔物理学奖颁给了谁？"
      assistantContent,        // LLM 的回答
  )

  分类器对比参考资料和LLM 回答，返回：

  result = &HallucinationResult{
      HallucinationDetected: true,
      Confidence:           0.88,
      UnsupportedSpans:     ["2018年", "深度卷积神经网络"],
  }

  结果写入 ctx（第 70-72 行）：
  ctx.HallucinationDetected = true
  ctx.HallucinationSpans    = ["2018年", "深度卷积神经网络"]
  ctx.HallucinationConfidence = 0.88

  路径 B：NLI 增强模式（第 99-169 行）

  result, err := r.Classifier.DetectHallucinationWithNLI(...)

  返回更丰富的结果，写入 ctx.EnhancedHallucinationInfo（第 128-144 行）：

  ctx.EnhancedHallucinationInfo = &EnhancedHallucinationInfo{
      Confidence: 0.91,
      Spans: []EnhancedHallucinationSpan{
          {
              Text:                    "2018年",
              Start:                   52,
              End:                     57,
              HallucinationConfidence: 0.95,
              NLILabel:                "CONTRADICTION",
              NLIConfidence:           0.97,
              Severity:                3,     // high
              Explanation:             "The Nobel Prize was awarded for foundational discoveries, no specific year of invention was cited.",
          },
          {
              Text:                    "深度卷积神经网络",
              Start:                   61,
              End:                     69,
              HallucinationConfidence: 0.89,
              NLILabel:                "CONTRADICTION",
              NLIConfidence:           0.92,
              Severity:                3,
              Explanation:             "The prize was for foundational discoveries in machine learning, not specifically deep CNNs.",
          },
      },
  }

  ---
  第四步：记录指标 & 确定动作（第 79-93 行）

  metrics.RecordPluginExecution("hallucination", decisionName, "detected", latency)
  action := r.getHallucinationActionForDecision(ctx.VSRSelectedDecision)
  // action = "header" 或 "body" 或 "none"

  注意：performHallucinationDetection 本身始终返回 nil，不拦截响应。它只负责检测并记录，实际的警告注入在后续的 applyHallucinationWarning 中完成。

  ---
  第五步：警告注入（由 processor_res_body.go 调用 applyHallucinationWarning）

  func applyHallucinationWarning(response, ctx, responseBody)

  根据 action 配置，三种处理方式：

  action = "header"

  在 HTTP 响应头中添加警告：

  X-Hallucination-Detected: true
  X-Fact-Check-Needed: true
  X-Hallucination-Spans: 2018年; 深度卷积神经网络

  客户端（前端）可以读取这些 header 来展示 UI 提示，响应体不变。

  action = "body"

  直接修改响应体，在 content 前面插入警告文字：

  基础模式（includeDetails = false）：
  [Hallucination Warning] This response may contain unsupported claims. Please verify the information independently.

  2024年诺贝尔物理学奖颁给了 Geoffrey Hinton 和 John Hopfield...

  NLI 增强模式（includeDetails = true）：
  [Hallucination Warning] This response may contain unsupported claims (confidence: 91%).
  Detailed analysis:
  [1] "2018年" (NLI: CONTRADICTION, confidence: 97%, severity: high) - The Nobel Prize was awarded for foundational discoveries, no specific year of invention
  was cited.
  [2] "深度卷积神经网络" (NLI: CONTRADICTION, confidence: 92%, severity: high) - The prize was for foundational discoveries in machine learning, not
  specifically deep CNNs.
  Please verify the information independently.

  2024年诺贝尔物理学奖颁给了 Geoffrey Hinton 和 John Hopfield...

  action = "none"

  检测结果记录到指标和日志中，但不通知客户端。

  ---
  完整调用链总结

  handleResponseBody (processor_res_body.go)
    |
    +-- performHallucinationDetection          ← 检测阶段
    |     +-- shouldPerformHallucinationDetection  (前置条件)
    |     +-- extractAssistantContentFromResponse  (提取回答)
    |     +-- isNLIEnabledForDecision              (判断模式)
    |     +-- Classifier.DetectHallucination       (基础检测)
    |     |   或
    |     +-- Classifier.DetectHallucinationWithNLI (NLI 增强检测)
    |     +-- 写入 ctx.HallucinationDetected / Spans / EnhancedInfo
    |     +-- return nil (不拦截)
    |
    +-- applyHallucinationWarning              ← 警告注入阶段
          +-- action="header" → addHallucinationWarningHeaders
          +-- action="body"   → prependHallucinationWarningToBody
          |                      +-- buildHallucinationWarningText
          |                      +-- buildEnhancedHallucinationWarningText (NLI)
          +-- action="none"   → 跳过
```
