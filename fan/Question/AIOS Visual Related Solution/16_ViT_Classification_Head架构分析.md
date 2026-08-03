这是个很好的架构问题。简短回答：**不需要每类都独立一个完整模型，但每类需要独立的 Classification Head（推荐）**。

核心架构思路：

```
┌─────────────────────────────────────────────┐
│           Shared ViT Backbone               │
│     (小尺寸, e.g. ViT-Tiny/Small)           │
│         一次推理，提取通用特征                 │
└──────────────────┬──────────────────────────┘
                   │ [CLS] token / pooled features
       ┌───────────┼───────────┬──────────────┐
       ▼           ▼           ▼              ▼
  ┌─────────┐ ┌─────────┐ ┌─────────┐  ┌─────────┐
  │Head 1   │ │Head 2   │ │Head 3   │  │Head N   │
  │光照分类  │ │天气分类  │ │特殊车辆  │  │场景分类  │
  │5 classes│ │5 classes│ │4 classes│  │K classes│
  └─────────┘ └─────────┘ └─────────┘  └─────────┘
```

**推荐独立 Head 的原因：**

1. **类别空间不同** — 光照是 5 类（白天/黄昏/夜间/隧道/逆光），天气是 5 类（晴/阴/雨/雪/雾），特殊车辆是 3-4 类，混在一起会互相干扰
2. **按需激活** — 行车时激活路标/车辆 Head，停车时激活车位/规则牌 Head，不用的 Head 不推理（几乎零额外开销）
3. **独立迭代** — 某个分类效果差可以单独重训 Head（几分钟），不动 backbone

**Head 本身极轻量：**
- 就是 1-2 层 Linear（768→256→N_classes）
- 每个 Head 参数量 < 200K，推理 < 0.1ms
- 真正的算力成本在 backbone 的一次前向，Head 可以忽略不计

**主要 tradeoff 是：共享 backbone 还是分组 backbone？**

| 方案 | 优点 | 缺点 |
|------|------|------|
| **1 个 backbone + N 个 Head** | 算力最优，只推理一次 | 若任务差异大（如舱内 vs 舱外），特征不通用 |
| **2-3 个 backbone 分组** | 舱内/舱外/远景分别优化 | 多一些算力，但特征更 match |

我的建议是按 Camera 源分 2-3 个 backbone：
- **Backbone A (MPIC)** → 舱内相关 Head（手势/人群/行为）
- **Backbone B (FWC/SCF)** → 舱外相关 Head（天气/光照/车辆/路标/景色）
- **Backbone C (SVC)**（可选） → 近距离低分辨率场景（车位/人形）

这样每个 backbone 对自己的 Camera 输入特征做了针对性优化，同时 Head 仍然独立可插拔。

---

对，没错。**无论 backbone 怎么分组，每个 backbone 下面仍然需要多个独立的 Classification Head**。

分组只是解决了"特征提取针对性"的问题，不解决"输出任务多样性"的问题：

```
Backbone A (MPIC)
  ├── Head A1: 手势分类 (5 classes: 握拳/挥手/OK/指向/张掌)
  ├── Head A2: 人群分类 (4 classes: 儿童/成人/老人/障碍人群)
  ├── Head A3: 行为分类 (6 classes: 正常/睡觉/吃东西/化妆/打电话/危险动作)
  └── Head A4: 疲劳等级 (3 classes: 清醒/轻度疲劳/重度疲劳)

Backbone B (FWC/SCF)
  ├── Head B1: 天气分类 (5 classes: 晴/阴/雨/雪/雾)
  ├── Head B2: 光照分类 (5 classes: 白天/黄昏/夜间/隧道/逆光)
  ├── Head B3: 特殊车辆 (4 classes: 无/警车/救护车/消防车)
  ├── Head B4: 场景分类 (N classes: 城市/高速/乡村/山路/隧道...)
  └── Head B5: 景色分类 (N classes: 草原/大海/日出/山林/地标...)

Backbone C (SVC)
  ├── Head C1: 人形检测置信度 (2 classes: 有人/无人)
  ├── Head C2: 车位状态 (3 classes: 空/占用/半占用)
  └── Head C3: 事件分类 (4 classes: 路过/逗留/触碰/砸窗)
```

本质上是个 **Multi-Task Learning** 架构 — backbone 是共享特征提取器，Head 是任务特化层。这两层的职责完全不同：

- **Backbone** 回答："这张图的通用语义特征是什么"
- **Head** 回答："基于这个特征，在某个具体维度上属于哪一类"

同一张 FWC 画面，Backbone B 推理一次，Head B1 告诉你"下雨"，Head B2 告诉你"夜间"，Head B3 告诉你"前方有救护车" — 三个答案来自同一次 backbone 前向传播，只是过了不同的 Head。
