## ModelSelection 与 Looper 四大算法的关系

### 一句话概括

**ModelSelection = "选谁"（路由决策）**，**Looper = "怎么调"（执行编排）**。它们是流水线中前后两个阶段。

---

### 流水线位置

```
用户 Query
    │
    ▼
┌──────────────────────────────┐
│  Stage 1: Decision Engine    │  ← 分类 + 语义理解
│  (类别匹配 / Embedding)      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Stage 2: ModelSelection     │  ← 9种算法在这里
│  "这个 query 该用哪个模型？"   │
│  输出：候选模型 + 分数排名     │
└──────────────┬───────────────┘
               │ 候选列表 [model-A:0.92, model-B:0.78, model-C:0.45]
               ▼
┌──────────────────────────────┐
│  Stage 3: Looper             │  ← 4种算法在这里
│  "拿到候选后，怎么调用？"      │
│  输出：最终回答               │
└──────────────┬───────────────┘
               │
               ▼
          返回给用户
```

---

### 核心区别

| 维度 | ModelSelection（选谁） | Looper（怎么调） |
|------|----------------------|-----------------|
| **时机** | 执行前（pre-execution） | 执行时（execution-time） |
| **输入** | query + 所有注册模型 | query + 候选模型列表 |
| **输出** | 模型排名/分数 | 最终回答文本 |
| **关注点** | query与模型的匹配度 | 调用策略、质量保障 |
| **是否调LLM** | 不调（纯打分逻辑） | 调（实际发请求） |
| **类比** | 老师分配作业给最合适的学生 | 学生怎么完成作业（独立做/小组讨论/请教学长） |

---

### 具体协作方式

#### 场景1：ModelSelection(Static) + Looper(Confidence)

```
Query: "法国首都是哪里？"

Stage 2 - Static 选模型：
  math-lora: 0.30
  general-lora: 0.90  ← 选中
  code-lora: 0.20
  → 候选排序：[general-lora, math-lora, code-lora]

Stage 3 - Confidence 级联执行：
  先调 general-lora → confidence=0.98 → 直接采用
  
最终：general-lora 的回答
```

#### 场景2：ModelSelection(RouterDC) + Looper(ReMoM)

```
Query: "设计一个分布式缓存系统"

Stage 2 - RouterDC 语义匹配：
  code-lora:    0.88
  system-lora:  0.85
  general-lora: 0.30
  → 候选 top-3：[code-lora, system-lora, general-lora]

Stage 3 - ReMoM 多轮综合：
  Round 1: 并行调 code-lora + system-lora + general-lora
  Round 2: 综合三个回答，输出最终设计方案
  
最终：融合了代码实现细节 + 系统架构思考的综合方案
```

#### 场景3：ModelSelection(Elo) + Looper(RL-Driven)

```
Query: "写一封商务邮件"

Stage 2 - Elo 评分选模型：
  writing-lora: Rating=1620 → P=0.80
  general-lora: Rating=1380 → P=0.20
  → 候选：[writing-lora, general-lora]

Stage 3 - RL-Driven Thompson Sampling：
  writing-lora: Beta(α=50, β=5) → sample=0.89
  general-lora: Beta(α=10, β=8) → sample=0.52
  → 选 writing-lora 执行

用户反馈点赞 → 
  Elo 更新 writing-lora 的 Rating（ModelSelection层学习）
  RL-Driven 更新 Beta(α=51, β=5)（Looper层学习）
```

---

### 功能重叠与互补

有些算法在两层之间存在**概念重叠**，但作用粒度不同：

| ModelSelection 中的 | Looper 中的 | 重叠点 | 区别 |
|---|---|---|---|
| AutoMix | Confidence | 都是级联升级 | ModelSelection的AutoMix算**期望价值排序**；Looper的Confidence做**实际调用+置信度评估** |
| Elo | RL-Driven | 都从反馈中学习 | Elo学"哪个模型适合这类query"；RL学"这次调用结果好不好" |
| Hybrid（融合多信号） | ReMoM（融合多回答） | 都是"综合" | Hybrid融合**分数**选一个模型；ReMoM融合**回答文本**出最终结果 |

---

### 选择指南

```
                        ModelSelection 选的是"谁最匹配"
                                    ↓
                     ┌──────────────┼──────────────┐
                     │              │              │
               选出1个最佳     选出top-K候选    选出所有候选
                     │              │              │
                     ▼              ▼              ▼
              直接单次调用    Confidence/RL    Ratings/ReMoM
              (不需要Looper)   (级联/学习)     (并行/综合)
```

**简单理解**：
- ModelSelection 回答的是 **"哪个模型最懂这个问题"**
- Looper 回答的是 **"知道用谁之后，怎么调才能拿到最好的结果"**
