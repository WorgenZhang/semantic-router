把Looper四大算法详解.md内容在这里打印出来，我喜欢在这里看

# Looper 四大算法详解

## 一、Confidence（置信度级联）算法

### 算法机制

Confidence 算法的核心思想来自 **AutoMix 论文（arXiv:2310.12963）**：先用小模型尝试回答，如果小模型"很自信"就直接采用，否则逐步升级到更大的模型。本质是一个**级联（Cascade）策略**，用置信度作为"是否需要升级"的判断依据。

### 执行流程

```
1. 将模型按参数量/价格从小到大排序：[8B, 13B, 70B]
2. 从最小模型开始调用
3. 评估置信度：
   - 如果 confidence >= threshold → 采用该回答，停止
   - 如果 confidence < threshold → 尝试下一个更大的模型
4. 重复直到置信度满足或所有模型用完
```

### 置信度评估方法（4种）

#### 方法1：avg_logprob（平均对数概率）

模型输出每个 token 时，都有一个 logprob 值（对数概率），表示模型对该 token 的确定程度。

**归一化公式**：

```
normalized_confidence = (avg_logprob + 3.0) / 3.0
```

- 输入范围：avg_logprob ∈ [-∞, 0]（越接近 0 越自信）
- 输出范围：[0, 1]（1 = 最自信）
- 低于 -3.0 的值被截断为 0

**示例**：

```
模型输出 "Hello world" 的 token logprobs:
  "Hello" → -0.2
  " world" → -0.1
  avg_logprob = (-0.2 + -0.1) / 2 = -0.15
  
  normalized = (-0.15 + 3.0) / 3.0 = 0.95  ← 很自信
```

#### 方法2：margin（top-1 与 top-2 的边际差）

看模型在每个位置选择的 top-1 token 和 top-2 token 的概率差距。差距越大，越确定。

**单 token margin**：

```
margin_i = logprob(top1_token_i) - logprob(top2_token_i)
```

**归一化公式（sigmoid 变换）**：

```
normalized_margin = 1 - exp(-avg_margin / 3.0)
```

- margin = 0 → normalized = 0（完全不确定，两个候选一样好）
- margin = 2 → normalized ≈ 0.49
- margin = 5 → normalized ≈ 0.81
- margin = 10 → normalized ≈ 0.96

**示例**：

```
Token 位置 1:
  top1: "Paris"  logprob = -0.1
  top2: "London" logprob = -2.5
  margin = -0.1 - (-2.5) = 2.4  ← 模型很确定是 Paris

Token 位置 2:
  top1: "is"    logprob = -0.3
  top2: "was"   logprob = -0.4
  margin = -0.3 - (-0.4) = 0.1  ← 模型不确定 is 还是 was

avg_margin = (2.4 + 0.1) / 2 = 1.25
normalized = 1 - exp(-1.25/3) = 0.34  ← 中等偏低置信度
```

#### 方法3：hybrid（混合方法）

结合 logprob 和 margin 两种方法，加权平均：

**公式**：

```
confidence = w_logprob × normalized_logprob + w_margin × normalized_margin
```

默认 w_logprob = 0.5, w_margin = 0.5

#### 方法4：self_verify（自我验证，来自 AutoMix 论文）

让模型自己评估自己的回答质量。不需要 logprobs，而是额外做一次 LLM 调用：

```
第1步：模型 A 回答用户问题 → 得到 response
第2步：再问模型 A：
  "原始问题是 X，AI 的回答是 Y，请评分 0.0-1.0"
第3步：模型 A 返回 {"confidence": 0.85, "reason": "..."}
第4步：如果 0.85 >= threshold，采用
```

### 模型排序策略（3种）

| 策略        | 排序依据            | 公式                                                        |
| ----------- | ------------------- | ----------------------------------------------------------- |
| `size`    | 参数量从小到大      | 按 param_size 升序（8B < 13B < 70B）                        |
| `cost`    | 价格从低到高        | 按 pricing.prompt_per_1m 升序                               |
| `automix` | POMDP 成本-质量权衡 | `value = (1-λ) × quality + λ × (1 - normalized_cost)` |

其中 AutoMix 策略的 value 公式：

```
value_i = (1 - tradeoff) × quality_i + tradeoff × costScore_i

quality_i = 配置的 quality_score，或根据参数量估算：0.3 + 0.7 × min(param_size/70B, 1.0)
costScore_i = 1.0 - (cost_i - min_cost) / (max_cost - min_cost)
tradeoff ∈ [0, 1]，默认 0.3
```

- tradeoff = 0：纯按质量排序
- tradeoff = 1：纯按成本排序
- tradeoff = 0.3：偏向质量但兼顾成本

### 具体场景举例

#### 场景1：简单问答 → 小模型搞定

```
用户问："法国的首都是哪里？"

配置：
  models: [llama-3-8b, llama-3-70b]
  threshold: 0.6
  method: avg_logprob

执行：
  第1轮 → llama-3-8b
    回答："法国的首都是巴黎。"
    各 token logprobs: [-0.05, -0.02, -0.01, -0.08, -0.03, -0.01]
    avg_logprob = -0.033
    normalized = (-0.033 + 3.0) / 3.0 = 0.989
  
    0.989 >= 0.6 ✓ → 采用 llama-3-8b 的回答

结果：只调了 1 次小模型，省钱省时间
```

#### 场景2：复杂推理 → 升级到大模型

```
用户问："证明 √2 是无理数"

配置：
  models: [llama-3-8b, llama-3-13b, llama-3-70b]
  threshold: 0.5
  method: margin

执行：
  第1轮 → llama-3-8b
    回答：（含部分逻辑错误的证明）
    avg_margin = 0.8
    normalized = 1 - exp(-0.8/3) = 0.234
  
    0.234 < 0.5 ✗ → 不够自信，升级

  第2轮 → llama-3-13b
    回答：（基本正确但不完整）
    avg_margin = 1.5
    normalized = 1 - exp(-1.5/3) = 0.393
  
    0.393 < 0.5 ✗ → 还是不够，继续升级

  第3轮 → llama-3-70b
    回答：（完整正确的归谬法证明）
    avg_margin = 3.2
    normalized = 1 - exp(-3.2/3) = 0.655
  
    0.655 >= 0.5 ✓ → 采用 llama-3-70b 的回答

结果：复杂问题自动升级到大模型，保证质量
```

---

## 二、Ratings（并行竞技评分）算法

### 算法机制

Ratings 算法的思想非常简单粗暴：**同时调所有模型，把所有回答都返回给客户端**。类似 ChatBot Arena 的模式——让用户（或下游系统）对比选择最好的。

这不是一个"选择最优"的算法，而是一个**信息收集**算法，适用于需要多样性或人工评审的场景。

### 执行流程

```
1. 确定并发数 max_concurrent（默认 = 模型数量）
2. 并行调用所有模型（用信号量控制并发上限）
3. 等待所有模型返回
4. 将所有成功的回答打包成 choices[] 数组返回
```

### 数学模型

没有复杂的数学公式，核心是**并发控制**：

```
信号量容量 = min(max_concurrent, len(models))
每个模型调用在独立 goroutine 中执行
```

### 响应格式

```json
{
  "model": "model-A,model-B,model-C",
  "choices": [
    {"index": 0, "message": {"content": "A的回答"}, "model": "model-A"},
    {"index": 1, "message": {"content": "B的回答"}, "model": "model-B"},
    {"index": 2, "message": {"content": "C的回答"}, "model": "model-C"}
  ]
}
```

和标准 OpenAI API 的区别：正常 API 只返回 1 个 choice，Ratings 返回 N 个 choice（每个模型一个）。

### 具体场景举例

#### 场景1：创意写作竞技

```
用户请求："写一首关于AI的俳句"

配置：
  models: [gpt-4, claude-3, llama-3-70b]
  max_concurrent: 3

执行（并行）：
  goroutine 1 → gpt-4:      "硅谷梦想者 / 数据中流淌智慧 / 人机共吟诗"
  goroutine 2 → claude-3:    "电子脉搏跳 / 算法编织春秋梦 / 思维无疆界"
  goroutine 3 → llama-3-70b: "代码如流水 / 神经网络映星辰 / 未来已可期"

返回：3 个 choices，客户端 dashboard 展示给人类评审打分
```

#### 场景2：A/B 测试新微调模型

```
用户请求："解释 Kubernetes 的 Pod 概念"

配置：
  models: [base-model, fine-tuned-v1, fine-tuned-v2]
  max_concurrent: 2  ← 限制并发，保护 GPU

执行：
  第1批并行：base-model + fine-tuned-v1
  第2批等位：fine-tuned-v2

返回：3 个回答供产品团队对比微调效果
```

---

## 三、ReMoM（Reasoning for Mixture of Models）算法

### 算法机制

ReMoM 灵感来自 **PaCoRe 论文（arXiv:2601.05593）**，核心是**多轮并行推理 + 综合（Synthesis）**：

1. 第一轮：多个模型各自独立思考同一个问题
2. 后续轮：将前一轮的所有回答作为"参考"，让模型综合出更好的答案
3. 最终轮：只调一个模型做最终综合

类比：像一个小组讨论——先各自发言，然后看了所有人的意见后再给出总结。

### 执行流程

```
BreadthSchedule = [4, 2, 1]  （3轮：4→2→1，加上自动追加的最终轮1）

实际执行 schedule = [4, 2, 1, 1]：

Round 1: 4 个并行调用（原始问题）
  → 收集 4 个不同回答

Round 2: 2 个并行调用（综合 prompt + Round 1 的 4 个参考回答）
  → 收集 2 个综合回答

Round 3: 1 个调用（综合 prompt + Round 2 的 2 个参考回答）
  → 得到 1 个综合回答

Round 4 (final): 1 个调用（最终综合）
  → 最终输出
```

### 关键配置参数

| 参数                    | 含义                   | 默认值     |
| ----------------------- | ---------------------- | ---------- |
| `breadth_schedule`    | 每轮并行调用数         | [4]        |
| `model_distribution`  | 模型分配策略           | "weighted" |
| `temperature`         | 采样温度（增加多样性） | 1.0        |
| `include_reasoning`   | 是否包含推理过程       | false      |
| `compaction_strategy` | 参考回答压缩策略       | "full"     |
| `compaction_tokens`   | 压缩保留的 token 数    | 1000       |

### 综合 Prompt 模板

```
你被给定了一个问题和一组参考回答。你的任务是分析这些参考，给出你自己的回答。

原始问题：
{{.OriginalContent}}

参考回答：
{{range $i, $resp := .ReferenceResponses}}
参考 {{$i+1}} ({{$resp.Model}}):
{{$resp.Content}}
{{end}}

现在，基于原始问题和上述参考回答，请给出你自己的综合解答。
```

### 模型分配策略

```
weighted:    按权重分配（当前等同于 equal）
equal:       平均分配，如 4 calls / 2 models = 每个模型 2 次
first_only:  所有调用都用第一个模型（PaCoRe 兼容模式）
```

### 压缩策略

```
full:          完整保留参考回答（默认）
last_n_tokens: 只保留最后 N 个 token（约 N×4 个字符）
```

### 具体场景举例

#### 场景1：复杂代码架构设计

```
用户问："设计一个支持百万并发的实时消息系统"

配置：
  models: [llama-3-70b, codellama-34b, mistral-large]
  breadth_schedule: [3]
  model_distribution: "equal"
  temperature: 0.8

执行：

Round 1（3个并行调用，每个模型1次）：
  llama-3-70b → "建议使用 Kafka + WebSocket，分区策略为..."
  codellama-34b → "核心数据结构用 ring buffer + skip list..."
  mistral-large → "推荐 actor 模型 + CRDT 解决一致性..."

Round 2（自动追加的最终轮，1个调用）：
  综合 prompt = 原始问题 + 3个参考回答
  → llama-3-70b（随机选中）综合所有方案：
    "综合分析：采用分层架构：
     - 接入层：WebSocket + actor 模型（参考3）
     - 消息层：Kafka 分区 + ring buffer（参考1、2）
     - 存储层：CRDT 用于多活同步（参考3）
     ..."

最终输出：综合了三个模型各自优势的完整方案
```

#### 场景2：学术论文润色（包含推理过程）

```
用户提交一段学术英文段落请求润色

配置：
  models: [writing-model-v1]
  breadth_schedule: [4, 2]
  model_distribution: "first_only"
  temperature: 1.2  ← 高温度增加多样性
  include_reasoning: true

执行：

Round 1（4个并行调用，都用同一个模型但高温度产生不同结果）：
  call-1 → reasoning: "这段话主语不明确..." content: 润色版本A
  call-2 → reasoning: "时态不一致需要统一..." content: 润色版本B
  call-3 → reasoning: "被动语态过多..." content: 润色版本C
  call-4 → reasoning: "逻辑连接词可以更精确..." content: 润色版本D

Round 2（2个并行调用，综合 Round 1 的 4 个版本）：
  综合 prompt 包含 4 个参考回答 + 各自的 reasoning
  → 综合版本 E（结合了 A 的主语修正 + B 的时态统一 + D 的连接词改进）
  → 综合版本 F（另一种综合方式）

Round 3（最终轮，1个调用）：
  → 从 E 和 F 中综合出最终最佳版本

好处：单模型通过多次采样 + 多轮综合，达到了类似多模型协作的效果
```

---

## 四、RL-Driven（强化学习驱动）算法

### 算法机制

RL-Driven 算法基于 **Router-R1 论文（arXiv:2506.09033）**，使用 **Thompson Sampling（汤普森采样）** 来动态学习每个模型的质量，并做出"探索 vs 利用"的平衡决策。

核心思想：

- 每个模型维护一个 **Beta 分布**，表示"该模型回答好的概率"
- 每次请求时，从各模型的 Beta 分布中**随机采样**，选采样值最高的模型
- 根据用户反馈更新 Beta 分布参数
- 多轮模式下，选 top-K 个模型并行调用，最后聚合

### 数学公式

#### Beta 分布

每个模型 $m$ 维护参数 $(\alpha_m, \beta_m)$：

```
P(θ_m) = Beta(α_m, β_m)

均值 = α_m / (α_m + β_m)
方差 = (α_m × β_m) / ((α_m + β_m)² × (α_m + β_m + 1))
```

- $\alpha_m$：成功次数 + 1（先验）
- $\beta_m$：失败次数 + 1（先验）
- 初始值：$\alpha = 1, \beta = 1$（均匀先验，无偏见）

#### Thompson Sampling 采样

从 Beta 分布采样使用 **Gamma 分布法**：

```
X ~ Beta(α, β) 的采样方法：
  g1 = Gamma(α, 1)  ← 从 Gamma 分布采样
  g2 = Gamma(β, 1)
  sample = g1 / (g1 + g2)
```

其中 Gamma 分布采样使用 **Marsaglia-Tsang 方法**（对 α ≥ 1）。

#### 多轮选择

```
对每个候选模型 m：
  sample_m = Thompson_Sample(α_m, β_m)
  
  如果启用成本感知：
    sample_m = sample_m × cost_bonus(m)

按 sample 值降序排列
选择 top-K 个模型（K = min(max_rounds, num_candidates)）
```

#### 反馈更新

当收到用户反馈（显式评分或隐式信号）：

```
如果用户选择了 model_winner：
  α_winner += weight
  β_loser += weight

weight = feedback_weight × confidence
  （隐式反馈的 weight 通常 < 显式反馈）
```

#### 响应聚合

多轮模式收集到多个回答后，使用**置信度加权选择**：

```
best_model = argmax_m { score_m }

其中 score_m 是该模型在 Thompson Sampling 中的采样值
```

当前实现选择分数最高的模型的回答作为最终输出（未做文本级融合）。

### 探索 vs 利用的平衡

Thompson Sampling 天然具备探索-利用平衡：

```
模型A：α=50, β=5  → 均值=0.91，方差小 → 几乎总是采样出高值（利用）
模型B：α=3, β=2   → 均值=0.60，方差大 → 有时采样出高值（探索）
模型C：α=1, β=1   → 均值=0.50，方差最大 → 经常被探索到
```

- 已验证好用的模型（高α）→ 大概率被选中（利用已有知识）
- 新加入的模型（均匀先验）→ 有一定概率被探索到（发现新机会）
- 表现差的模型（高β）→ 几乎不会被选中（自动淘汰）

### 具体场景举例

#### 场景1：在线学习 + 个性化路由

```
初始状态（3个模型，均匀先验）：
  llama-3-8b:  Beta(α=1, β=1)  均值=0.50
  llama-3-13b: Beta(α=1, β=1)  均值=0.50
  llama-3-70b: Beta(α=1, β=1)  均值=0.50

--- 第1次请求 ---
Thompson Sampling 采样：
  llama-3-8b:  sample = 0.73（随机采样）
  llama-3-13b: sample = 0.41
  llama-3-70b: sample = 0.88  ← 最高，被选中
  
max_rounds = 2 → 选 top-2：[llama-3-70b, llama-3-8b]
并行调用两个模型，用分数加权选择 → 输出 llama-3-70b 的回答

用户反馈：点赞 llama-3-70b
  → llama-3-70b: Beta(α=2, β=1)  均值=0.67 ↑

--- 第10次请求后 ---
  llama-3-8b:  Beta(α=4, β=3)   均值=0.57
  llama-3-13b: Beta(α=2, β=5)   均值=0.29  ← 多次不佳，被淘汰
  llama-3-70b: Beta(α=8, β=2)   均值=0.80  ← 表现稳定优秀

此时 llama-3-70b 大概率被选中，llama-3-13b 几乎不会被选
但 llama-3-8b 偶尔还有机会（方差仍较大）
```

#### 场景2：新模型上线探索

```
已稳定运行（2个模型）：
  model-A: Beta(α=100, β=10)  均值=0.91，方差极小
  model-B: Beta(α=20, β=80)   均值=0.20，几乎被淘汰

新模型上线：
  model-C: Beta(α=1, β=1)     均值=0.50，方差大（均匀先验）

第一批请求中的 Thompson Sampling：
  model-A: sample ≈ 0.90（低方差，稳定高值）
  model-B: sample ≈ 0.22（低方差，稳定低值）
  model-C: sample = 0.35~0.85（高方差，波动大）← 有机会被选中！

如果 model-C 在前几次被探索到且用户反馈好：
  model-C: Beta(α=5, β=1) → 均值=0.83，快速建立信任
  
如果 model-C 表现不佳：
  model-C: Beta(α=1, β=5) → 均值=0.17，快速被淘汰

这就是 Thompson Sampling 的"自动探索新选项"能力
```

---

## 五、四大算法对比总结

| 维度               | Confidence       | Ratings      | ReMoM        | RL-Driven       |
| ------------------ | ---------------- | ------------ | ------------ | --------------- |
| **核心思想** | 级联升级         | 并行竞技     | 多轮综合     | 在线学习        |
| **模型调用** | 串行（1→N）     | 并行（全部） | 多轮并行     | 并行（top-K）   |
| **数学基础** | Logprob / Margin | 无           | Prompt模板   | Beta + Thompson |
| **调用次数** | 1~N（平均少）    | 固定N        | 轮数×宽度   | K（可配置）     |
| **适合场景** | 成本敏感         | 人工评审     | 需要最高质量 | 长期运营        |
| **反馈机制** | 无               | 外部打分     | 无           | 在线更新        |
| **论文来源** | AutoMix          | —           | PaCoRe       | Router-R1       |
| **最终输出** | 单个模型回答     | 所有模型回答 | 综合后的回答 | 最优模型回答    |

### 选择建议

```
"我想省钱"               → Confidence（小模型能搞定就不用大模型）
"我要对比模型效果"        → Ratings（同时看所有回答）
"我要最高质量的单一回答"  → ReMoM（多模型协作综合）
"我要系统越用越聪明"      → RL-Driven（从用户反馈中学习）
```
