```markdown
Modality 实际上横跨两个阶段

  从源码看，Modality 的处理分成两步：

  第一步：Modality 信号评估（在 Decision 期间）

  req_filter_classification.go — 9d. Modality 分类
  如果匹配了 Modality 信号，设置 ctx.ModalityClassification

  Modality 信号确实参与 Decision 评估。比如你可以配一条规则：IF modality=DIFFUSION THEN route-to-image-decision。这一步和 Keyword/Domain 一样是
  Decision 的输入条件。

  第二步：Modality 路由执行（在 Decision 之后，步骤 14）

  // processor_req_body.go:164-186
  如果 Response API 请求包含 image_generation tool → 强制设置 modality
  如果 modality 是 DIFFUSION 或 BOTH → 执行图片生成并短路返回

  这一步是执行动作而不是信号判断。它做的事情：
  1. Response API 的 image_generation tool 强制覆盖 modality（不走信号，直接硬判）
  2. 实际调用 diffusion 模型生成图片并短路返回

  为什么执行要放在 Decision 之后？

  因为你需要知道 Decision 上下文才能决定：
  - 用哪个 diffusion 端点（不同 Decision 可能配不同的图片模型）
  - 是否需要先走完安全检查（某些 Decision 可能要求图片请求也过 jailbreak 检测）
  - 如果是 BOTH（同时需要文本+图片），文本部分的模型选择依赖 Decision

  修正后的分类

  ┌───────────────────────────┬─────────────────────────────────────────────────────────┐
  │           阶段            │                     Modality 做什么                     │
  ├───────────────────────────┼─────────────────────────────────────────────────────────┤
  │ 信号评估（Decision 期间） │ 判断请求是 AR/DIFFUSION/BOTH → 参与 Decision 规则匹配   │
  ├───────────────────────────┼─────────────────────────────────────────────────────────┤
  │ 路由执行（Decision 之后） │ 根据 Decision 结果，实际调用 diffusion 模型或走混合路径 │
  └───────────────────────────┴─────────────────────────────────────────────────────────┘

  所以你的理解是对的——Modality 信号确实参与决策。我之前说"可以延后"不准确，准确说法是：Modality 的信号检测不能延后，但它的路由执行（真正调模型）必须在
   Decision 之后，因为执行依赖 Decision 的配置上下文。
```
