```markdown
❯ ctx.ToolResultsContext = "工具返回的上下文内容" 举几个例子ToolResultsContext返回的是什么

⏺ 让我先追溯 ToolResultsContext 的两个数据来源的具体内容。

⏺ Searched for 2 patterns, read 1 file (ctrl+o to expand)

⏺ 现在让我看一下 RAG 检索具体返回的内容格式。

⏺ Read 1 file (ctrl+o to expand)

⏺ 好的，现在我对 ToolResultsContext 的数据来源有完整理解了。它有 两个来源，以下用具体例子说明：

  ---
  ToolResultsContext 的两个来源 + 实际内容示例

  来源一：RAG 检索的上下文（优先级更高）

  当 Decision 配置了 RAG 插件，路由器会从向量数据库（Milvus/外部 API/MCP）检索相关文档，然后以 tool role 消息注入到请求中。此时 ToolResultsContext = RAG
  检索返回的文本。

  示例场景：用户问 "vLLM 支持哪些量化格式？"，RAG 从知识库检索到相关文档

  ToolResultsContext =
  "vLLM supports the following quantization methods: GPTQ, AWQ, SqueezeLLM,
  and FP8. GPTQ uses 4-bit quantization with group size of 128 by default.
  AWQ (Activation-aware Weight Quantization) achieves similar accuracy to GPTQ
  with faster inference. FP8 quantization is supported on Ada Lovelace and
  Hopper GPUs (SM 89+). SqueezeLLM uses non-uniform quantization..."

  示例场景：用户问 "公司Q3营收多少？"，RAG 从企业知识库检索到财报数据

  ToolResultsContext =
  "根据2024年Q3财报，公司实现营业收入52.3亿元，同比增长18.7%。
  其中：主营业务收入48.1亿元，其他业务收入4.2亿元。
  净利润8.6亿元，同比增长12.3%。毛利率为35.2%。"

  此时幻觉检测会拿这段文本作为 grounding，检查 LLM 的回答是否与之矛盾。

  ---
  来源二：请求中 role=tool 的历史消息

  当用户的多轮对话中包含工具调用结果（function calling / tool use 的返回值），代码会从 messages 数组中提取所有 role: "tool" 的 content，用 \n\n 拼接。

  示例场景：用户先让 LLM 调天气 API，再追问总结

  请求 body 中的 messages：
  {
    "messages": [
      {"role": "user", "content": "北京和上海今天天气怎么样？"},
      {"role": "assistant", "content": null, "tool_calls": [
        {"id": "call_1", "function": {"name": "get_weather", "arguments": "{\"city\":\"北京\"}"}},
        {"id": "call_2", "function": {"name": "get_weather", "arguments": "{\"city\":\"上海\"}"}}
      ]},
      {"role": "tool", "tool_call_id": "call_1", "content": "北京：晴，气温28°C，湿度35%，北风3级"},
      {"role": "tool", "tool_call_id": "call_2", "content": "上海：多云转阴，气温31°C，湿度78%，东南风2级"},
      {"role": "user", "content": "帮我总结一下两个城市的天气"}
    ]
  }

  提取后：
  ToolResultsContext =
  "北京：晴，气温28°C，湿度35%，北风3级

  上海：多云转阴，气温31°C，湿度78%，东南风2级"

  如果 LLM 回答说"上海今天是晴天，气温25°C"，幻觉检测就会发现这与 tool results 矛盾。

  ---
  示例场景：用户让 LLM 查数据库，然后基于结果回答

  {"role": "tool", "tool_call_id": "call_db", "content": "Query result: [{\"user_id\": 1001, \"name\": \"张三\", \"balance\": 5230.50}, {\"user_id\": 1002,
  \"name\": \"李四\", \"balance\": 12800.00}]"}

  提取后：
  ToolResultsContext =
  "Query result: [{\"user_id\": 1001, \"name\": \"张三\", \"balance\": 5230.50}, {\"user_id\": 1002, \"name\": \"李四\", \"balance\": 12800.00}]"

  如果 LLM 说"张三的余额是 8000 元"，幻觉检测会标记这是不一致的。

  ---
  关键区别

  ┌──────────┬─────────────────────────────────┬─────────────────────────────────────────┐
  │          │       来源一：RAG 上下文        │          来源二：Tool 历史消息          │
  ├──────────┼─────────────────────────────────┼─────────────────────────────────────────┤
  │ 代码位置 │ req_filter_rag.go:284           │ req_filter_fact_check.go:84             │
  ├──────────┼─────────────────────────────────┼─────────────────────────────────────────┤
  │ 优先级   │ 更高（先检查，有则直接 return） │ 后检查                                  │
  ├──────────┼─────────────────────────────────┼─────────────────────────────────────────┤
  │ 数据来源 │ 向量数据库/外部 API 检索        │ 请求 messages 中 role:"tool" 的 content │
  ├──────────┼─────────────────────────────────┼─────────────────────────────────────────┤
  │ 谁产生的 │ 路由器 RAG 插件自动注入         │ 客户端发来的（工具调用历史）            │
  ├──────────┼─────────────────────────────────┼─────────────────────────────────────────┤
  │ 核心用途 │ 知识库 grounding                │ 工具执行结果 grounding                  │
  └──────────┴─────────────────────────────────┴─────────────────────────────────────────┘

  两者本质相同：都是可信的事实基础，用于对比 LLM 的回答是否存在编造。
```
