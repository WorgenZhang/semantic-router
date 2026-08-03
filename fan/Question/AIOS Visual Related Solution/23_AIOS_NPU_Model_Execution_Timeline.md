@"fan/Question/AIOS Visual Related Solution/21_AIOS_Visual_Decision_Engine_具体示例.md" 根据Step5输出激活的 Use Case列表，最终模型在NPU上的执行顺序是怎样的，如何配合到按优先级排序的UC,
  多路流和多个模型，注意ViT模型跑在单独NPU上，CV模型跑在单独NPU上，所以ViT和CV模型之间可以并行跑，帮我保存到新makrdown文档上

## NPU 执行时序：基于 Step 5 激活 UC 列表的模型调度

### 硬件约束

```
┌─────────────────────────────────────────────────────────┐
│  NPU-ViT (专用于 ViT Backbone + Heads)                  │
│  约束: 同一时刻只跑一个 Backbone 推理                    │
│  同一 Backbone 输出可同时过多个 Head (Head ≈ 0.1ms)     │
├─────────────────────────────────────────────────────────┤
│  NPU-CV (专用于 CV 小模型)                               │
│  约束: 同一时刻只跑一个 CV 模型                          │
│  单个 CV 模型延迟: 8~20ms                                │
├─────────────────────────────────────────────────────────┤
│  两个 NPU 之间: 完全并行                                 │
└─────────────────────────────────────────────────────────┘
```

languagelanguage---

### 激活的 UC 列表回顾

| UC   | Priority | Tier        | Pipeline Type   | 调度频率       | 涉及流              |
| ---- | -------- | ----------- | --------------- | -------------- | ------------------- |
| uc17 | P100     | safety      | cv_vit_parallel | 10Hz (每100ms) | MPIC                |
| uc11 | P100     | safety      | cv_vit_parallel | 10Hz (每100ms) | FWC+SCF_L+SCF_R     |
| uc13 | P95      | safety      | cv_vit_parallel | 10Hz (每100ms) | FWC+FTC+SCF_L+SCF_R |
| uc12 | P80      | core        | only_vit        | 2Hz (每500ms)  | FWC+SCF_L+SCF_R     |
| uc10 | P80      | core        | only_vit        | 1Hz (每1000ms) | FWC                 |
| uc5  | P75      | core        | cv_vit_parallel | 5Hz (每200ms)  | MPIC                |
| uc8  | P40      | enhancement | —              | DISABLED       | —                  |
| uc3  | P40      | enhancement | —              | DISABLED       | —                  |

---

### 核心调度逻辑

**每 100ms 为一个主周期**（由 safety tier 的 10Hz 决定）。

在每个主周期内：

- Safety UC (uc17/uc11/uc13): **每轮必调度**
- Core UC (uc12): 每5轮调度1次 (2Hz / 10Hz = 1/5)
- Core UC (uc10): 每10轮调度1次 (1Hz / 10Hz = 1/10)
- Core UC (uc5): 每2轮调度1次 (5Hz / 10Hz = 1/2)

---

### 单周期（100ms）执行时序 — Safety UC 满载轮

以第1轮（所有UC都需要调度）为例：

```
时间轴 (ms)    NPU-ViT                              NPU-CV
─────────────────────────────────────────────────────────────────────────
0              ┌─────────────────────────┐           ┌──────────────────┐
               │ Backbone_A (MPIC帧)     │           │ cv_child_det     │
               │ 输入: 720p MPIC frame   │           │ (MPIC帧, 10ms)   │
               │ 延迟: 25ms             │           │ → uc17/uc5       │
               │                         │           └──────────────────┘
10             │                         │           ┌──────────────────┐
               │                         │           │ cv_fatigue_det   │
               │                         │           │ (MPIC帧, 12ms)   │
               │                         │           │ → uc17/uc5       │
22             │                         │           └──────────────────┘
               │                         │           ┌──────────────────┐
25             └─────────────────────────┘           │ cv_ped_gesture   │
               ↓ Head 输出 (共享, 各0.1ms)           │ (FWC帧, 15ms)    │
               → head_a2 (人群分类) → uc17/uc5      │ → uc11           │
               → head_a3 (行为分类) → uc17/uc5      │                  │
               → head_a4 (疲劳等级) → uc17          │                  │
               ≈ 25.3ms 完成                         │                  │
                                                     │                  │
26             ┌─────────────────────────┐           │                  │
               │ Backbone_B (FWC帧)      │           │                  │
               │ 输入: 1080p FWC frame   │           │                  │
               │ 延迟: 30ms             │           │                  │
               │                         │           │                  │
37             │                         │           └──────────────────┘
               │                         │           ┌──────────────────┐
               │                         │           │ cv_traffic_police│
               │                         │           │ (FWC帧, 15ms)    │
               │                         │           │ → uc11           │
               │                         │           │                  │
52             │                         │           └──────────────────┘
               │                         │           ┌──────────────────┐
               │                         │           │ cv_road_sign     │
56             └─────────────────────────┘           │ (FWC帧, 12ms)    │
               ↓ Head 输出 (共享, 各0.1ms)           │ → uc13           │
               → head_b3 (特殊车辆) → uc11/uc12     │                  │
               → head_b4 (场景类型) → uc13          │                  │
               → head_b1 (天气)     → uc10          │                  │
               → head_b2 (光照)     → uc10          │                  │
               ≈ 56.4ms 完成                         │                  │
                                                     │                  │
57             ┌─────────────────────────┐           │                  │
               │ Backbone_B (SCF_L帧)    │           │                  │
               │ 延迟: 30ms             │           │                  │
               │                         │           └──────────────────┘
64             │                         │           ┌──────────────────┐
               │                         │           │ cv_road_marking  │
               │                         │           │ (FWC帧, 10ms)    │
               │                         │           │ → uc13           │
74             │                         │           └──────────────────┘
               │                         │           ┌──────────────────┐
               │                         │           │ cv_special_lane  │
               │                         │           │ (FWC帧, 10ms)    │
               │                         │           │ → uc13           │
84             │                         │           └──────────────────┘
               │                         │           ┌──────────────────┐
               │                         │           │ cv_zone_area     │
87             └─────────────────────────┘           │ (FWC帧, 10ms)    │
               ↓ Head 输出                           │ → uc13           │
               → head_b3 (特殊车辆) → uc11/uc12     │                  │
               ≈ 87.1ms 完成                         │                  │
                                                     │                  │
88             ┌─────────────────────────┐           │                  │
               │ Backbone_B (SCF_R帧)    │           │                  │
               │ 延迟: 30ms             │           │                  │
               │                         │           └──────────────────┘
94             │                         │           ┌──────────────────┐
               │                         │           │ cv_road_sign     │
               │                         │           │ (FTC帧, 12ms)    │
               │                         │           │ → uc13 secondary │
               │                         │           │                  │
100  ══════════╪═══ 主周期结束 ═══════════╪═══════════╪══════════════════╪══
               │                         │           │                  │
106            │                         │           └──────────────────┘
               │                         │
118            └─────────────────────────┘
               ↓ Head 输出
               → head_b3 (特殊车辆) → uc11/uc12
               ≈ 118.1ms 完成
```

languagelanguage---

### 超时分析

```
uc17 (max_latency_ms: 100):
  ViT完成: 25.3ms ✅
  CV完成:  22ms   ✅
  Parallel Merge: max(25.3, 22) = 25.3ms ✅✅ 远低于100ms

uc11 (max_latency_ms: 100, sync_late_fusion 3路):
  CV完成:  52ms (cv_ped_gesture + cv_traffic_police)
  ViT FWC: 56.4ms
  ViT SCF_L: 87.1ms
  ViT SCF_R: 118.1ms ❌ 超出100ms!
  
  → 问题: 3路 Backbone_B 串行导致 SCF_R 超时

uc13 (max_latency_ms: 100):
  CV完成: 94ms (road_sign×2 + marking + lane + zone)
  ViT FWC: 56.4ms
  Late Fusion需等CV: 94ms ✅ 刚好在100ms内
```

languagelanguage---

### 优化方案：流水线重排

**问题根因**：Backbone_B 需要处理 FWC/SCF_L/SCF_R/FTC 四路帧，串行跑完需要 4×30ms=120ms，超出 100ms 周期。

**解决方案：按 UC 优先级决定 Backbone_B 的输入顺序**

```
优先级排序:
  1. FWC帧    → 服务 uc11(P100) + uc13(P95) + uc12(P80) + uc10(P80)
  2. SCF_L帧  → 服务 uc11(P100) + uc12(P80)
  3. SCF_R帧  → 服务 uc11(P100) + uc12(P80)
  4. FTC帧    → 仅服务 uc13(P95) secondary
```

language**方案 A：交错调度（Interleaved Scheduling）**

将 SCF_L 和 SCF_R 分配到不同周期：

```
周期 N:   Backbone_B 处理 FWC + SCF_L  (2×30ms = 60ms)
周期 N+1: Backbone_B 处理 FWC + SCF_R  (2×30ms = 60ms)
每5周期:  额外插入 FTC (uc13 secondary, 5Hz)
```

languageLate Fusion 改为滑动窗口融合：

```
uc11 weights: {FWC_current: 0.5, SCF_L_current_or_prev: 0.25, SCF_R_current_or_prev: 0.25}
```

language**方案 B：降低非 FWC 流的 ViT 频率**

```
FWC:   每轮都过 Backbone_B (10Hz)  → 保证主路实时
SCF_L: 每2轮过一次 (5Hz)           → 辅助路降频
SCF_R: 每2轮过一次 (5Hz)           → 辅助路降频
FTC:   每2轮过一次 (5Hz)           → 已在YAML中定义为5Hz
```

languagelanguage---

### 采用方案 B 后的优化时序（单周期 100ms）

```
时间轴 (ms)    NPU-ViT                              NPU-CV
─────────────────────────────────────────────────────────────────────────
0              ┌─────────────────────────┐           ┌──────────────────┐
               │ Backbone_A (MPIC)       │           │ cv_child_det     │
               │ 25ms                    │           │ 10ms → uc17/uc5  │
10             │                         │           └──────────────────┘
               │                         │           ┌──────────────────┐
               │                         │           │ cv_fatigue_det   │
               │                         │           │ 12ms → uc17/uc5  │
22             │                         │           └──────────────────┘
               │                         │           ┌──────────────────┐
25             └─────────────────────────┘           │ cv_ped_gesture   │
               → Heads A2/A3/A4 (0.3ms)             │ 15ms → uc11      │
               ↓ uc17/uc5 ViT结果就绪               │                  │
                                                     │                  │
26             ┌─────────────────────────┐           │                  │
               │ Backbone_B (FWC)        │           │                  │
               │ 30ms                    │           │                  │
37             │                         │           └──────────────────┘
               │                         │           ┌──────────────────┐
               │                         │           │ cv_traffic_police│
               │                         │           │ 15ms → uc11      │
52             │                         │           └──────────────────┘
               │                         │           ┌──────────────────┐
               │                         │           │ cv_road_sign     │
56             └─────────────────────────┘           │ 12ms → uc13     │
               → Heads B1-B4 (0.4ms)                │                  │
               ↓ uc11/uc13/uc12/uc10                │                  │
                 FWC路结果就绪                        │                  │
                                                     │                  │
57             ┌─────────────────────────┐           │                  │
               │ Backbone_B (SCF_L)      │           │                  │
               │ 30ms                    │           │                  │
               │ (本轮调度SCF_L,          │           │                  │
               │  下轮调度SCF_R)          │           │                  │
64             │                         │           └──────────────────┘
               │                         │           ┌──────────────────┐
               │                         │           │ cv_road_marking  │
               │                         │           │ 10ms → uc13      │
74             │                         │           └──────────────────┘
               │                         │           ┌──────────────────┐
               │                         │           │ cv_special_lane  │
               │                         │           │ 10ms → uc13      │
84             │                         │           └──────────────────┘
               │                         │           ┌──────────────────┐
87             └─────────────────────────┘           │ cv_zone_area     │
               → Head B3 (0.1ms)                    │ 10ms → uc13      │
               ↓ uc11/uc12 SCF_L路结果就绪           │                  │
                                                     │                  │
88             NPU-ViT 空闲                          │                  │
               (本轮不跑SCF_R,省30ms)                │                  │
               可选: 插入 FTC 帧                      │                  │
               给 uc13 secondary                     │                  │
94             │                         │           └──────────────────┘
                                                     │
               ┌─────────────────────────┐           NPU-CV 空闲 (6ms)
               │ Backbone_B (FTC)        │
               │ 30ms (若本轮是5Hz轮)     │
               │ 或 NPU空闲              │
100  ══════════╪═══ 主周期结束 ═══════════╪══════════════════════════════════
```

languagelanguage---

### 优化后各 UC 延迟验证

```
┌────────────────────────────────────────────────────────────────────┐
│ UC              │ 依赖完成时间点          │ Deadline │ 结果        │
├────────────────────────────────────────────────────────────────────┤
│ uc17 (P100)    │ max(ViT:25.3, CV:22)    │ 100ms   │ 25.3ms ✅✅ │
│                │ = 25.3ms                │         │             │
├────────────────────────────────────────────────────────────────────┤
│ uc11 (P100)    │ CV: 52ms                │ 100ms   │ 87.1ms ✅   │
│ sync_late_fus  │ ViT FWC: 56.4ms         │         │             │
│                │ ViT SCF_L: 87.1ms       │         │             │
│                │ (SCF_R 用上轮结果)       │         │             │
│                │ Fusion: 87.1ms          │         │             │
├────────────────────────────────────────────────────────────────────┤
│ uc13 (P95)     │ CV: 94ms                │ 100ms   │ 94ms ✅     │
│ primary_sec    │ ViT FWC: 56.4ms         │         │             │
│                │ Primary结果: 94ms        │         │             │
│                │ (FTC secondary异步)      │         │             │
├────────────────────────────────────────────────────────────────────┤
│ uc12 (P80)     │ ViT FWC: 56.4ms         │ 200ms   │ 87.1ms ✅✅ │
│ 2Hz, 每5轮     │ ViT SCF_L: 87.1ms       │         │             │
│                │ (无CV, only_vit)         │         │             │
├────────────────────────────────────────────────────────────────────┤
│ uc10 (P80)     │ ViT FWC: 56.4ms         │ 500ms   │ 56.4ms ✅✅ │
│ 1Hz, 每10轮    │ (仅FWC, 降级后无SCF)     │         │             │
├────────────────────────────────────────────────────────────────────┤
│ uc5 (P75)      │ max(ViT:25.3, CV:22)    │ 200ms   │ 25.3ms ✅✅ │
│ 5Hz, 每2轮     │ (与uc17完全共享)         │         │             │
└────────────────────────────────────────────────────────────────────┘
```

languagelanguage---

### 优先级如何体现在执行顺序中

```
规则 1: Backbone 输入帧的排队顺序 = UC 优先级
  NPU-ViT 队列: Backbone_A(MPIC) → Backbone_B(FWC) → Backbone_B(SCF_L) → Backbone_B(FTC)
  理由: MPIC服务P100的uc17, FWC服务P100/P95, SCF_L服务P100, FTC仅服务P95 secondary

规则 2: CV 模型的排队顺序 = UC 优先级
  NPU-CV 队列: cv_child(uc17 P100) → cv_fatigue(uc17 P100) → cv_ped_gesture(uc11 P100) 
               → cv_traffic_police(uc11 P100) → cv_road_sign(uc13 P95) → ...
  理由: P100的UC对应的CV模型排在前面

规则 3: 非 Safety UC 按降频调度
  uc12 (P80): 每5轮才跑一次 → 不是每轮都占NPU-ViT的SCF_L/SCF_R时间
  uc10 (P80): 每10轮才跑一次 → 大部分轮次不需要额外Head计算
  uc5 (P75): 每2轮才需要 → 奇数轮 Backbone_A 的 Head 输出可以不给uc5

规则 4: same_backbone_shared 使低优先级UC"搭便车"
  uc12/uc10 不需要额外的 Backbone_B 推理 → 只要 Backbone_B 为 uc11/uc13 跑了,
  uc12/uc10 的 Head 就能直接用同一个 feature → 零额外NPU开销
```

languagelanguage---

### 多路流的处理模式对比

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Multi-Stream Strategy    │ NPU-ViT 处理方式          │ 融合时机        │
├─────────────────────────────────────────────────────────────────────────┤
│ single (uc17, uc5)       │ 1路帧 → 1次Backbone      │ 无需融合        │
│                          │ 最快, 25~30ms             │                 │
├─────────────────────────────────────────────────────────────────────────┤
│ sync_late_fusion         │ N路帧 → N次Backbone串行   │ 等所有路完成    │
│ (uc11, uc12)             │ 但可交错到不同周期        │ 后加权平均      │
│                          │ 本轮: FWC+SCF_L (60ms)    │ weights加权     │
│                          │ 下轮: FWC+SCF_R (60ms)    │                 │
├─────────────────────────────────────────────────────────────────────────┤
│ primary_secondary        │ Primary每轮都跑           │ Primary够置信   │
│ (uc13, uc10)             │ Secondary低频/按需        │ → 直接用        │
│                          │ FWC每轮, FTC每2轮         │ Primary不够     │
│                          │                           │ → 融合Secondary │
├─────────────────────────────────────────────────────────────────────────┤
│ trigger_escalation       │ Always-on路常驻           │ Trigger后才     │
│ (uc9, uc7 - 本例未激活)  │ Escalation路按需唤醒     │ 启动高清流推理  │
└─────────────────────────────────────────────────────────────────────────┘
```

languagelanguage---

### 完整 5 轮调度表（展示降频 UC 的调度分配）

```
轮次    NPU-ViT 任务                           NPU-CV 任务                    额外调度UC
─────────────────────────────────────────────────────────────────────────────────────────
#1      BA(MPIC) + BB(FWC) + BB(SCF_L)         cv_child + cv_fatigue +        uc5, uc12
        + BB(FTC)                               cv_ped + cv_police +
                                                cv_sign + cv_marking +
                                                cv_lane + cv_zone +
                                                cv_sign(FTC)

#2      BA(MPIC) + BB(FWC) + BB(SCF_R)         cv_child + cv_fatigue +        uc5
                                                cv_ped + cv_police +
                                                cv_sign + cv_marking +
                                                cv_lane + cv_zone

#3      BA(MPIC) + BB(FWC) + BB(SCF_L)         cv_child + cv_fatigue +        uc5
        + BB(FTC)                               cv_ped + cv_police +
                                                cv_sign + cv_marking +
                                                cv_lane + cv_zone +
                                                cv_sign(FTC)

#4      BA(MPIC) + BB(FWC) + BB(SCF_R)         cv_child + cv_fatigue +        uc5
                                                cv_ped + cv_police +
                                                cv_sign + cv_marking +
                                                cv_lane + cv_zone

#5      BA(MPIC) + BB(FWC) + BB(SCF_L)         cv_child + cv_fatigue +        uc5, uc12
        + BB(FTC)                               cv_ped + cv_police +           uc10
                                                cv_sign + cv_marking +
                                                cv_lane + cv_zone +
                                                cv_sign(FTC)

说明:
  - BA = Backbone_A, BB = Backbone_B
  - uc17/uc11/uc13: 每轮必调度 (Safety, preemptible=false)
  - uc5: 每2轮 (#1,#2,#3,#4,#5 都调度是因为5Hz=每轮, 但共享BA输出零开销)
  - uc12: 每5轮 (#1, #5), 但实际是搭便车 BB 输出中的 head_b3, 零额外开销
  - uc10: 每10轮 (#5 等), 同样搭便车 BB 输出中的 head_b1/b2, 零额外开销
  - SCF_L/SCF_R 交替: 奇数轮跑SCF_L, 偶数轮跑SCF_R
  - FTC: 每2轮跑一次 (5Hz), 奇数轮调度
```

languagelanguage---

### 关键结论

1. **双 NPU 并行是核心收益**：ViT (25~30ms/模型) 和 CV (8~15ms/模型) 同时执行，将串行 151ms 压缩到 ~94ms
2. **Backbone 共享是最大优化**：uc12/uc10 标记为 `only_vit` 但实际不占用额外 NPU-ViT 时间——它们的 Head 直接"搭便车"到 uc11/uc13 的 Backbone_B 输出上
3. **优先级通过排队顺序体现**：NPU-ViT 先跑 MPIC(服务P100 uc17) → FWC(服务P100/P95) → SCF(服务P100)；NPU-CV 先跑 uc17 的 CV → uc11 的 CV → uc13 的 CV
4. **多路流通过交错调度解决超时**：sync_late_fusion 的 3 路不在同一周期全部跑完，而是 SCF_L/SCF_R 交替，用上轮结果补位，保证单周期不超 100ms
5. **降频是资源让步而非性能损失**：uc12(2Hz)/uc10(1Hz) 降频后，它们需要 Backbone 输出的轮次减少，但由于 `same_backbone_shared=true`，即使每轮都能拿到结果——降频实际只是降低了"主动发起推理请求"的频率，不影响搭便车获取结果
