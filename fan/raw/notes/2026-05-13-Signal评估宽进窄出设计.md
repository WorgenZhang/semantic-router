好问题。让我从架构角度分析一下。

## 关键区分：驱动决策 vs 服务于后处理

从源码可以看到，`EvaluateAllSignalsWithHeaders()` 一次性评估所有信号，但它们的**消费者**不同：

### 必须在 Decision 评估前获取（参与 AND/OR 规则匹配）

这些信号是 Decision 规则的**输入条件**，没有它们 Decision Engine 无法匹配：

| 信号 | 作用 | 为什么必须提前 |
|------|------|--------------|
| **Keyword** | 关键词匹配 | 直接构成 Decision 规则条件 |
| **Embedding** | 语义相似度 | 直接构成 Decision 规则条件 |
| **Domain** | 域分类 | "coding 请求走大模型" 这类规则的基础 |
| **Complexity** | 复杂度 | "高复杂度走推理模型" 的判断依据 |
| **Language** | 语言检测 | "中文走本地模型" 等规则 |
| **Context** | token 计数 | "长上下文走 128k 模型" 的判断 |
| **Authz** | 授权评估 | 决定用户能访问哪些 Decision |

这些是 Decision 的 **IF 条件**：`IF keyword=coding AND complexity=high THEN route-to-reasoning-model`

### 理论上可以延迟到 Decision 之后（但当前实现一起做了）

| 信号 | 真正的消费者 | 为什么可以延后 |
|------|------------|--------------|
| **Fact-check** | 响应体阶段的[[幻觉检测]] | 只设置 `ctx.FactCheckNeeded`，不影响模型选择 |
| **Modality** | 请求体步骤 14 的 Modality 路由 | 在 Decision 之后才执行图片生成分支 |
| **User Feedback** | 可能微调模型选择权重 | 更像是辅助信号而非硬规则 |
| **Preference** | 用户偏好注入 | 影响的是请求改写而非路由 |

## 那为什么不拆开做？

当前实现选择**一次性全部评估**，原因是：

1. **共享计算**：Embedding 向量一旦算出来，Keyword/Domain/Complexity/Fact-check 可以复用同一次向量化结果，拆开反而更贵
2. **架构简洁**：一次调用返回所有信号，Decision Engine 拿到完整信号集直接匹配，不需要二次调度
3. **未来灵活性**：今天 Fact-check 不参与 Decision 规则，但产品可能随时加一条 `IF fact_check=needed THEN route-to-grounded-model`

## 核心洞察

```
信号评估是一个"宽进窄出"的设计：
  
  宽进：一次性算出所有 11 种信号（共享 embedding 计算）
  窄出：Decision Engine 只消费其中 5-7 种作为规则条件
        剩余信号在后续步骤按需消费（幻觉检测、modality 路由等）
```

这是**计算效率**和**架构灵活性**之间的平衡——比起"按需计算"节省的延迟，一次性计算带来的 embedding 复用和规则可扩展性更值得。
