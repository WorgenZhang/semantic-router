# AIOS Visual Routing Solution

## 一、核心设计思想

### 1.1 从 Semantic Router 到 Visual Router 的映射

vLLM Semantic Router 的核心架构：

```
多路信号输入（keyword/embedding/domain/complexity...）
    → 信号分类器（classifiers）
    → Decision Engine（规则树匹配 + 优先级策略）
    → 路由到对应的 Model + Prompt + Plugin
```

languagelanguagelanguagelanguageAIOS Visual Router 的对应映射：

```
多路信号输入（车辆信号/CV小模型信号/传感器信号）
    → 场景分类器（Scene Classifier）
    → Visual Decision Engine（规则树匹配 + 优先级调度）
    → 激活对应的 Camera Streams + Models + Inference Pipeline
```

### 1.2 核心类比

| Semantic Router                | Visual Router                     | 说明         |
| ------------------------------ | --------------------------------- | ------------ |
| User Query (文本)              | Sensor Signals (车辆/CV信号)      | 输入信号     |
| Domain Classifier              | Scene Classifier                  | 大场景识别   |
| Categories (business/math/...) | Scenes (Driving/Parking/...)      | 场景类别     |
| Decisions                      | Use Case Decisions                | 路由决策     |
| Rules (AND/OR/NOT)             | Activation Rules                  | 触发条件     |
| Priority Strategy              | Priority + Power-Aware Strategy   | 调度策略     |
| ModelRefs                      | Camera Streams + Model Pipeline   | 执行资源     |
| Plugins (cache/prompt/pii)     | Plugins (fusion/escalation/power) | 处理插件     |
| SignalMatches                  | VisualSignalMatches               | 信号匹配结果 |

---

## 二、整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AIOS Visual Router                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────────────────────────────┐                          │
│  │         Signal Ingestion Layer               │                          │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐     │                          │
│  │  │Vehicle   │ │CV Trigger│ │Sensor    │     │                          │
│  │  │Signals   │ │Signals   │ │Signals   │     │                          │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘     │                          │
│  └───────┼─────────────┼────────────┼───────────┘                          │
│          │             │            │                                       │
│          ▼             ▼            ▼                                       │
│  ┌──────────────────────────────────────────────┐                          │
│  │         Scene Classifier                     │                          │
│  │  Input Signals → Scene Detection             │                          │
│  │  Output: Active Scene(s) + Confidence        │                          │
│  └──────────────────────┬───────────────────────┘                          │
│                         │                                                   │
│                         ▼                                                   │
│  ┌──────────────────────────────────────────────┐                          │
│  │         Visual Decision Engine               │                          │
│  │  ┌────────────┐  ┌────────────────────────┐  │                          │
│  │  │Rule Tree   │  │Priority + Power-Aware  │  │                          │
│  │  │Evaluator   │  │Strategy Selector       │  │                          │
│  │  └────────────┘  └────────────────────────┘  │                          │
│  └──────────────────────┬───────────────────────┘                          │
│                         │                                                   │
│                         ▼                                                   │
│  ┌──────────────────────────────────────────────┐                          │
│  │         Resource Orchestrator                │                          │
│  │  ┌───────────────┐  ┌────────────────────┐   │                          │
│  │  │Stream Manager │  │Model Scheduler     │   │                          │
│  │  │(Camera On/Off)│  │(NPU/CPU Dispatch)  │   │                          │
│  │  └───────────────┘  └────────────────────┘   │                          │
│  └──────────────────────┬───────────────────────┘                          │
│                         │                                                   │
│                         ▼                                                   │
│  ┌──────────────────────────────────────────────┐                          │
│  │         Inference Pipeline                   │                          │
│  │  ┌────────┐ ┌────────┐ ┌────────┐          │                          │
│  │  │CV Model│ │ViT+Head│ │OCR     │          │                          │
│  │  │Stage   │ │Stage   │ │Stage   │          │                          │
│  │  └───┬────┘ └───┬────┘ └───┬────┘          │                          │
│  │      └───────────┴──────────┘               │                          │
│  │              │                               │                          │
│  │              ▼                               │                          │
│  │      ┌──────────────┐                       │                          │
│  │      │Late Fusion / │                       │                          │
│  │      │Decision Gate │                       │                          │
│  │      └──────┬───────┘                       │                          │
│  └─────────────┼───────────────────────────────┘                          │
│                │                                                           │
│                ▼                                                           │
│  ┌──────────────────────────────────────────────┐                          │
│  │         Use Case Trigger                     │                          │
│  │  匹配触发的 Use Case → 按优先级排序 → 执行    │                          │
│  └──────────────────────────────────────────────┘                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

languagelanguagelanguagelanguage---

## 三、详细设计

### 3.1 Signal Ingestion Layer（信号采集层）

三类输入信号，类比 Semantic Router 的 keyword/embedding/domain：

#### Vehicle Signals（车辆信号 → 类比 keyword 规则）

确定性信号，直接触发场景切换：

| 信号                           | 含义       | 触发场景             |
| ------------------------------ | ---------- | -------------------- |
| `gear == D && speed > 5`     | 行驶中     | → Driving           |
| `gear == R`                  | 倒车       | → Parking           |
| `gear == P && engine == ON`  | 停车未熄火 | → Parking/Drop-Off  |
| `gear == P && engine == OFF` | 熄火停车   | → Sentry Mode       |
| `door_opening == true`       | 开门动作   | → Drop-Off          |
| `turn_signal == ON`          | 转向       | → Drop-Off (BSV)    |
| `adas_active == true`        | ADAS 激活  | → Driving (ADAS UC) |

#### CV Trigger Signals（CV 小模型触发信号 → 类比 embedding 规则）

概率性信号，CV 小模型低功耗持续运行产出：

| 信号                                 | 来源          | 含义       |
| ------------------------------------ | ------------- | ---------- |
| `person_detected(svc, confidence)` | SVC 人形检测  | 车外有人   |
| `motion_detected(svc, confidence)` | SVC 运动检测  | 有物体移动 |
| `gesture_detected(mpic, type)`     | MPIC 手势识别 | 舱内手势   |
| `child_detected(mpic, confidence)` | MPIC 儿童识别 | 检测到儿童 |
| `fatigue_detected(mpic, level)`    | MPIC 疲劳检测 | 驾驶员疲劳 |

#### System Signals（传感器/系统信号 → 类比 context 规则）

| 信号                  | 含义                                 |
| --------------------- | ------------------------------------ |
| `power_mode`        | 当前功耗模式 (high/normal/low/sleep) |
| `npu_load`          | NPU 当前负载 (0-100%)                |
| `thermal_state`     | 热状态 (normal/warm/throttle)        |
| `user_query_active` | 用户主动发起语音/手势指令            |
| `time_of_day`       | 当前时间 (用于判断光照条件)          |

### 3.2 Scene Classifier（场景分类器）

类比 Semantic Router 的 Domain Classifier，负责将输入信号映射到大场景。

```
┌─────────────────────────────────────────────────────┐
│               Scene Classifier                       │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Vehicle Signals ─┐                                 │
│  CV Triggers    ──┼──→ Scene Detection Logic ──→ Active Scenes │
│  Sensor Signals ──┘      (Rule-based + CV)          │
│                                                     │
│  Output:                                            │
│    - primary_scene: "driving"                       │
│    - secondary_scenes: ["in_cabin"]                 │
│    - power_profile: "normal"                        │
│    - confidence: 0.95                               │
└─────────────────────────────────────────────────────┘
```

languagelanguagelanguagelanguage**场景定义：**

| Scene                 | 识别方式            | 触发条件                                          |
| --------------------- | ------------------- | ------------------------------------------------- |
| **Driving**     | Vehicle Signal      | `gear==D && speed>5`                            |
| **Parking**     | Vehicle Signal + CV | `gear==R \|\| (speed<5 && parking_area_detected)` |
| **Drop-Off**    | Vehicle Signal      | `door_opening \|\| turn_signal`                   |
| **Sentry Mode** | Vehicle Signal      | `gear==P && engine==OFF && sentry_enabled`      |
| **In-Cabin**    | Always Active       | 只要有人在车内（DMS active）                      |
| **Out-Cabin**   | CV Trigger          | `person_detected(svc) && gear==P`               |

**重要：场景可叠加**
例如 Driving + In-Cabin 同时激活（行车时舱内感知依然运行）。

### 3.3 Visual Decision Engine（视觉决策引擎）

核心模块，类比 Semantic Router 的 DecisionEngine。

#### 信号匹配结构（类比 SignalMatches）

```go
type VisualSignalMatches struct {
    // 场景信号
    SceneSignals      []string  // "driving", "parking", "sentry_mode"
  
    // 车辆信号
    VehicleSignals    []string  // "gear_d", "speed_high", "door_opening"
  
    // CV 触发信号
    CVTriggers        []string  // "person_detected", "gesture_detected"
  
    // 系统信号
    SystemSignals     []string  // "power_normal", "npu_available", "user_query"
  
    // 信号置信度
    SignalConfidences  map[string]float64  // "cv:person_detected" → 0.92
}
```

gogogo#### 决策评估流程（类比 EvaluateDecisionsWithSignals）

```
1. 收集所有 VisualSignalMatches
2. 遍历所有 Use Case Decisions:
   - 评估 Rules（AND/OR/NOT 规则树）
   - 匹配成功 → 计算 confidence
3. 过滤已匹配的 Decisions
4. 按 Strategy 排序选择:
   - Priority（优先级高的先执行）
   - Power-Aware（同优先级下，低功耗方案优先）
   - Resource-Constrained（NPU 满载时降级）
5. 输出: 激活的 Use Case 列表 + 对应资源方案
```

#### 优先级 + 功耗感知策略

```
Strategy: "priority_power_aware"

排序规则:
  1. 按 Priority 降序（安全优先）
  2. 同 Priority 按 Power Profile 匹配度排序
  3. 资源冲突时按 Priority 抢占
  
资源约束:
  - NPU 同时只运行 1 个模型 → 串行队列
  - 摄像头总功耗上限 → 按需唤醒
  - 热节流时 → 降低非安全类 UC 频率
```

### 3.4 Resource Orchestrator（资源编排器）

类比 Semantic Router 中选择 model + endpoint 的逻辑，但这里管理的是 Camera Streams 和 Model Pipeline。

#### Stream Manager（摄像头管理）

```
┌─────────────────────────────────────────────┐
│            Stream Manager                    │
├─────────────────────────────────────────────┤
│                                             │
│  Camera Pool:                               │
│    FWC  → [active/standby/off]              │
│    FTC  → [active/standby/off]              │
│    MPIC → [active/standby/off]              │
│    SCF  → [active/standby/off] ×2          │
│    SCR  → [active/standby/off] ×2          │
│    SVC  → [active/standby/off] ×4          │
│    RVC  → [active/standby/off]              │
│                                             │
│  状态转换:                                   │
│    off → standby: ~100ms (预热)             │
│    standby → active: ~10ms (即时)           │
│    active → standby: 立即                   │
│    standby → off: 超时 30s 无需求           │
│                                             │
│  功耗等级:                                   │
│    LOW:    SVC×4 (5fps) + MPIC (10fps)      │
│    NORMAL: +FWC (30fps) + SCF×2 (30fps)    │
│    HIGH:   +SCR×2 + FTC + RVC (全量)        │
│    SLEEP:  SVC×4 (5fps) only                │
└─────────────────────────────────────────────┘
```

#### Model Scheduler（模型调度器）

```
┌─────────────────────────────────────────────┐
│           Model Scheduler                    │
├─────────────────────────────────────────────┤
│                                             │
│  NPU Queue (Priority-based):                │
│    ┌──────────────────────────────────────┐ │
│    │ [P10] UC-17 MPIC → Backbone_A        │ │
│    │ [P10] UC-11 FWC  → Backbone_B        │ │
│    │ [P9]  UC-13 FWC  → Backbone_B        │ │
│    │ [P8]  UC-12 FWC  → Backbone_B        │ │
│    │ ...                                   │ │
│    └──────────────────────────────────────┘ │
│                                             │
│  CPU Tasks (Lightweight):                    │
│    - Classification Heads (< 0.1ms each)    │
│    - Late Fusion Decision                   │
│    - CV small models (if CPU-capable)       │
│                                             │
│  Scheduling Strategy:                        │
│    - Real-time UC: 每轮必调度               │
│    - High UC: 每 N 轮调度一次              │
│    - Medium/Low UC: 按需触发               │
└─────────────────────────────────────────────┘
```

### 3.5 Inference Pipeline（推理管线）

每个 Use Case 的推理方案，类比 Semantic Router 中 Decision 对应的 modelRefs + plugins。

#### Pipeline 类型

| 类型                      | 描述                           | 适用 UC                          |
| ------------------------- | ------------------------------ | -------------------------------- |
| **only_cv**         | 仅 CV 小模型                   | UC-4, UC-9                       |
| **only_vit**        | 仅 ViT Backbone + Head         | UC-3, UC-8, UC-12, UC-15         |
| **cv_then_vit**     | CV 先判断，不确定时调 ViT      | UC-2, UC-7 (级联)                |
| **cv_vit_parallel** | CV + ViT 同时运行，Late Fusion | UC-5, UC-14, UC-17, UC-11, UC-13 |
| **cv_ocr_vit**      | CV + OCR + ViT 混合            | UC-16                            |

#### Multi-Stream Inference（多路流推理）

基于前面文档分析的方案选择：

```
┌────────────────────────────────────────────────────────┐
│  Multi-Stream Strategy per UC                          │
├────────────────────────────────────────────────────────┤
│                                                        │
│  UC-13 (路标识别, P9, Real-time):                      │
│    Strategy: Primary-Secondary                         │
│    Primary: FWC (10Hz, 每帧推理)                       │
│    Secondary: FTC (2Hz, 远距离补充)                     │
│    Fusion: FWC高置信→直接用; 低置信→融合FTC结果         │
│                                                        │
│  UC-11 (手势识别, P10, Real-time):                     │
│    Strategy: Sync + Late Fusion                        │
│    Streams: FWC + SCF_L + SCF_R (同步截帧)             │
│    NPU: 串行推理 3路 (~90ms cycle)                     │
│    Fusion: 加权平均 (FWC:0.5, SCF_L:0.25, SCF_R:0.25) │
│                                                        │
│  UC-9 (哨兵模式, P7, High):                            │
│    Strategy: Trigger-based Escalation                  │
│    Always-on: SVC×4 (5fps, 人形+运动检测)              │
│    On-trigger: SCF/SCR (唤醒做高清录像+事件分类)        │
│                                                        │
│  UC-10 (环境感知, P8, High):                           │
│    Strategy: Primary-Secondary                         │
│    Primary: FWC (2Hz)                                  │
│    Secondary: SCF (0.5Hz, 边缘case辅助)                │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 3.6 Power Management（功耗管理）

类比 Semantic Router 中对不同 model 的 reasoning_effort 控制。

```
┌─────────────────────────────────────────────────────────┐
│              Power Profiles                              │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  SLEEP Mode (哨兵模式/熄火停车):                          │
│    Active Cameras: SVC×4 (5fps)                         │
│    Active Models: CV-人形检测, CV-运动检测               │
│    NPU Load: ~10%                                       │
│    Trigger → Escalate to LOW                            │
│                                                         │
│  LOW Mode (待机/低速):                                   │
│    Active Cameras: SVC×4 (10fps) + MPIC (10fps)        │
│    Active Models: CV Pool + ViT Backbone_C              │
│    NPU Load: ~30%                                       │
│    UC Enabled: UC-7, UC-9, UC-17(低频)                  │
│                                                         │
│  NORMAL Mode (正常行驶):                                 │
│    Active Cameras: FWC+MPIC+SCF×2 (30fps)              │
│    Active Models: CV Pool + Backbone_A + Backbone_B    │
│    NPU Load: ~70%                                       │
│    UC Enabled: All Tier1 + Tier2                        │
│                                                         │
│  HIGH Mode (ADAS激活/用户指令):                          │
│    Active Cameras: ALL (全量30fps)                      │
│    Active Models: ALL (CV Pool + Backbone_A/B/C)       │
│    NPU Load: ~95%                                       │
│    UC Enabled: All                                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

languagelanguagelanguagelanguage**功耗升降级规则：**

```
SLEEP → LOW:   person_detected(svc) || motion_detected(svc)
LOW → NORMAL:  gear==D && speed>5
NORMAL → HIGH: adas_active || user_query_active
HIGH → NORMAL: 5s无ADAS事件 && 无用户指令
NORMAL → LOW:  gear==P && speed==0 持续30s
LOW → SLEEP:   engine==OFF && 无CV触发 持续60s
```

languagelanguagelanguagelanguage---

## 四、Use Case Decisions 详细定义

### 4.1 决策规则设计（类比 config.yaml 的 decisions）

每个 Use Case Decision 结构：

```yaml
- name: "uc_identifier"
  description: "Use Case 描述"
  priority: 10       # 1-10, 越高越优先
  tier: "safety"     # safety / core / enhancement
  
  rules:             # 触发条件（AND/OR/NOT 规则树）
    operator: "AND"
    conditions:
      - type: "scene"
        name: "driving"
      - type: "vehicle"
        name: "speed_above_5"
  
  stream_refs:       # 需要的摄像头资源
    - camera: "FWC"
      fps: 30
      required: true
    - camera: "SCF"
      fps: 30
      required: false
  
  model_pipeline:    # 推理管线定义
    type: "cv_vit_parallel"
    stages:
      - name: "cv_stage"
        models: ["gesture_recognition_cv", "traffic_police_gesture_cv"]
        backbone: null
      - name: "vit_stage"
        backbone: "backbone_b"
        heads: ["gesture_classification"]
    fusion:
      strategy: "confidence_gate"
      cv_threshold: 0.8      # CV > 0.8 直接用CV结果
      fallback: "vit_stage"  # CV < 0.8 时融合ViT结果
  
  multi_stream:      # 多路流策略
    strategy: "sync_late_fusion"
    primary: "FWC"
    weights:
      FWC: 0.5
      SCF_L: 0.25
      SCF_R: 0.25
  
  scheduling:        # 调度参数
    frequency_hz: 10
    max_latency_ms: 100
    preemptible: false     # 是否可被抢占
  
  power_config:      # 功耗配置
    min_power_mode: "normal"  # 最低需要的功耗模式
    degradation:              # 功耗不足时的降级策略
      reduce_fps_to: 5
      drop_secondary_streams: true
```

yamlyamlyamlyaml### 4.2 Tier 1 — Safety Critical Decisions

#### UC-17: ADAS 舱内乘客状态感知

```
Trigger:   scene=driving|parking AND mpic_active
Priority:  10 (最高)
Pipeline:  CV(儿童/疲劳) + ViT Backbone_A (行为分类) Parallel
Streams:   MPIC (10Hz, 必须)
Frequency: 10 Hz
Latency:   < 100ms
Power:     NORMAL 以上
Preempt:   不可被抢占
```

#### UC-11: ADAS 舱外手势识别

```
Trigger:   scene=driving AND adas_active
Priority:  10
Pipeline:  CV(行人手势/交警手势) + ViT Backbone_B (手势分类) Parallel
Streams:   FWC(10Hz, 必须) + SCF×2(10Hz, 必须)
Frequency: 10 Hz
Latency:   < 100ms
Multi-stream: Sync + Late Fusion
Power:     NORMAL 以上
```

#### UC-13: ADAS 舱外静态目标

```
Trigger:   scene=driving AND adas_active
Priority:  9
Pipeline:  CV(RoadSign/RoadMarking/SpecialLane/Zone) + ViT Backbone_B Parallel
Streams:   FWC(10Hz, 必须) + FTC(5Hz, 必须) + SCF×2(5Hz, 可选)
Frequency: 10 Hz (FWC), 5 Hz (FTC)
Multi-stream: Primary-Secondary (FWC主, FTC辅)
Power:     NORMAL 以上
```

#### UC-2: 盲区影像 (BSV)

```
Trigger:   (scene=driving AND turn_signal) OR (scene=drop_off AND door_opening)
Priority:  9
Pipeline:  CV(车外障碍物) + ViT(兜底) CV-then-ViT
Streams:   SCR×2 (15Hz, 必须)
Frequency: 15 Hz
Latency:   < 67ms
Power:     NORMAL 以上
```

### 4.3 Tier 2 — Core Experience Decisions

#### UC-12: 特殊车辆识别

```
Trigger:   scene=driving
Priority:  8
Pipeline:  Only ViT Backbone_B + Head(特殊车辆)
Streams:   FWC(5Hz) + SCF×2(5Hz) + SCR×2(2Hz)
Multi-stream: Sync + Late Fusion
Power:     NORMAL 以上
```

#### UC-14: 停车场景舱外感知

```
Trigger:   scene=parking
Priority:  8
Pipeline:  CV(车位/消防通道/充电位) + ViT Backbone_B/C Parallel
Streams:   FWC(5Hz) + SVC_F(5Hz) + SCF×2(5Hz)
Multi-stream: Sync + Late Fusion
Power:     NORMAL 以上
```

#### UC-10: 舱外环境感知

```
Trigger:   scene=driving
Priority:  8
Pipeline:  ViT Backbone_B + Head(天气/光照)
Streams:   FWC(2Hz) + SCF×2(0.5Hz, 可选)
Multi-stream: Primary-Secondary (FWC主)
Power:     NORMAL 以上
```

#### UC-5: 车内乘客特征/行为识别

```
Trigger:   scene=in_cabin AND mpic_active
Priority:  7
Pipeline:  CV(儿童/睡眠) + ViT Backbone_A (年龄/行为) Parallel
Streams:   MPIC (5Hz)
Power:     LOW 以上
```

#### UC-15: 驾驶用户指令识别

```
Trigger:   scene=driving AND user_query_active
Priority:  7
Pipeline:  Only ViT Backbone_B + Head(目标解析)
Streams:   FWC(1Hz) + SCR×2(1Hz) + SCF×2(1Hz)
Power:     NORMAL 以上
```

#### UC-16: 停车用户指令识别

```
Trigger:   scene=parking AND user_query_active
Priority:  7
Pipeline:  CV(车位/目标) + OCR + ViT(兜底) CV-OCR-ViT
Streams:   FWC + SCF×2 + SCR×2 + SVC×4 + RVC
Power:     NORMAL 以上
```

#### UC-9: 哨兵模式

```
Trigger:   scene=sentry_mode
Priority:  7
Pipeline:  Only CV (人形/运动/事件分类) Parallel
Streams:   SVC×4(5Hz, 必须) + SCF/SCR(唤醒后)
Power:     SLEEP → 触发后升级到 LOW
Escalation: person_detected → 唤醒SCF/SCR高清录像
```

### 4.4 Tier 3 — Enhancement Decisions

#### UC-7: 车外用户身份识别

```
Trigger:   scene=out_cabin AND person_detected(svc)
Priority:  6
Pipeline:  CV1(SVC人形) → CV2(SCR/SCF FaceId) 级联
Streams:   SVC×4(5fps, 常驻) → SCR/SCF(唤醒)
Power:     LOW 以上
Escalation: person_detected → 唤醒SCR/SCF
```

#### UC-8: 车外AI识别景色

```
Trigger:   scene=driving AND speed>30 AND !adas_critical_event
Priority:  4
Pipeline:  Only ViT Backbone_B + Head(景色分类)
Streams:   FWC(1Hz) + SCF×2(0.5Hz, 可选)
Power:     NORMAL 以上
Note:      ADAS安全类UC繁忙时自动降级/暂停
```

#### UC-4: 游戏（手势识别）

```
Trigger:   scene=in_cabin AND game_mode_active
Priority:  3
Pipeline:  Only CV (手势识别)
Streams:   MPIC (15Hz)
Power:     NORMAL 以上
Note:      行车安全UC有资源冲突时被抢占
```

languagelanguagelanguagelanguage---

## 五、多模型处理架构

### 5.1 ViT Multi-Head 架构（共享Backbone）

```
┌─────────────────────────────────────────────────────────────────┐
│                  ViT Multi-Head Architecture                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Backbone_A (MPIC 输入, 舱内优化):                               │
│    Input: 720p MPIC frame                                       │
│    Output: [CLS] 768-dim feature                                │
│    ├── Head_A1: 手势分类 (5 classes)                            │
│    ├── Head_A2: 人群分类 (4 classes: 儿童/成人/老人/障碍)        │
│    ├── Head_A3: 行为分类 (6 classes: 正常/睡觉/吃东西/化妆/...)  │
│    └── Head_A4: 疲劳等级 (3 classes: 清醒/轻度/重度)            │
│                                                                 │
│  Backbone_B (FWC/SCF 输入, 舱外优化):                            │
│    Input: 1080p FWC/SCF frame                                   │
│    Output: [CLS] 768-dim feature                                │
│    ├── Head_B1: 天气分类 (5 classes: 晴/阴/雨/雪/雾)           │
│    ├── Head_B2: 光照分类 (5 classes: 白天/黄昏/夜间/隧道/逆光)   │
│    ├── Head_B3: 特殊车辆 (4 classes: 无/警车/救护车/消防车)      │
│    ├── Head_B4: 场景分类 (N classes: 城市/高速/乡村/山路/隧道)   │
│    └── Head_B5: 景色分类 (N classes: 草原/大海/日出/山林/地标)   │
│                                                                 │
│  Backbone_C (SVC 输入, 近距离低分辨率优化, 可选):                 │
│    Input: 480p/720p SVC frame                                   │
│    Output: [CLS] 512-dim feature                                │
│    ├── Head_C1: 人形检测置信度 (2 classes: 有人/无人)            │
│    ├── Head_C2: 车位状态 (3 classes: 空/占用/半占用)             │
│    └── Head_C3: 事件分类 (4 classes: 路过/逗留/触碰/砸窗)       │
│                                                                 │
│  按需激活:                                                       │
│    - Driving: Backbone_B全部Head + Backbone_A(Head_A3/A4)       │
│    - Parking: Backbone_B(Head_B1) + Backbone_C全部              │
│    - Sentry:  Backbone_C(Head_C1/C3) only                       │
│    - In-Cabin: Backbone_A全部                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 CV + ViT 协同策略

```
┌───────────────────────────────────────────────────────────┐
│          CV + ViT Collaboration Patterns                  │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  Pattern 1: CV-Gate (CV 高置信直接用, 低置信升级ViT)       │
│                                                           │
│    Frame → CV Model → confidence > 0.8? ─── Yes ──→ Use CV result
│                             │                             │
│                             No                            │
│                             ▼                             │
│                      ViT Backbone → Head → Final Result   │
│                                                           │
│    适用: UC-5, UC-7, UC-2                                 │
│    优势: 大部分帧CV就够用, 节省NPU算力                     │
│                                                           │
│  Pattern 2: Parallel + Late Fusion (同时运行, 融合结果)    │
│                                                           │
│    Frame ─┬─→ CV Models ──→ prob_cv ─┐                   │
│           │                           ├─→ Weighted Fusion → Final
│           └─→ ViT + Head ──→ prob_vit─┘                   │
│                                                           │
│    适用: UC-17, UC-11, UC-13 (安全关键, 需要冗余)          │
│    优势: 双重验证, 容错性高                                │
│                                                           │
│  Pattern 3: CV-Cascade (多CV串行级联)                      │
│                                                           │
│    SVC → CV1(人形检测) ─── detected ──→ 唤醒SCR/SCF       │
│                                            ↓              │
│                                    CV2(FaceId) → Result   │
│                                                           │
│    适用: UC-7 (车外身份识别), UC-9 (哨兵升级)              │
│    优势: 低功耗常驻, 按需升级                              │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

languagelanguagelanguagelanguage---

## 六、优先级冲突解决

### 6.1 资源竞争场景

当多个 UC 同时激活且 NPU 资源不足时：

```
┌─────────────────────────────────────────────────────┐
│          Priority Conflict Resolution               │
├─────────────────────────────────────────────────────┤
│                                                     │
│  规则 1: Safety > Core > Enhancement                │
│    UC-17(P10) 永远不被抢占                          │
│    UC-8(P4) 在 NPU 满载时可被暂停                   │
│                                                     │
│  规则 2: 同 Tier 内按 Priority 排序                  │
│    UC-11(P10) vs UC-13(P9): UC-11 优先              │
│                                                     │
│  规则 3: 低优先级 UC 的处理选项                      │
│    Option A: Drop (丢弃本轮)                        │
│    Option B: Defer (延迟到下一轮)                    │
│    Option C: Degrade (降低帧率/分辨率)              │
│                                                     │
│  规则 4: 共享Backbone的UC不额外占用                  │
│    UC-12 和 UC-10 共享 Backbone_B:                  │
│    Backbone_B 推理一次, 两个UC的Head同时输出         │
│    → 不构成资源竞争                                 │
│                                                     │
│  规则 5: 共享 Backbone 的多 Head 输出零额外开销      │
│    同一 Backbone 推理一次, 所有 Head 并行输出        │
│    不同 UC 的 Head 不构成 NPU 资源竞争              │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 6.2 场景切换时的资源回收

```
Driving → Parking 切换:
  1. 保留: MPIC(In-Cabin UC继续), FWC
  2. 新增: SVC×4, RVC
  3. 降级: SCF/SCR 从30fps降到10fps
  4. 关闭: FTC (远距离路牌停车不需要)
  5. 模型切换: 停用 Backbone_B Head(B3特殊车辆/B4场景), 激活 Backbone_C

Driving → Sentry Mode 切换:
  1. 全部关闭: FWC/FTC/SCF/SCR/MPIC/RVC
  2. 保留最小集: SVC×4 (5fps)
  3. 模型切换: 仅保留 CV(人形/运动检测), 关闭所有Backbone
  4. NPU 降频/休眠
```

languagelanguagelanguagelanguage---

## 七、核心流程示例

### 7.1 正常行驶场景

```
t=0: Vehicle Signal → gear=D, speed=60km/h
     Scene Classifier → primary_scene="driving", secondary="in_cabin"
     Power Profile → NORMAL

t=1: Visual Decision Engine 评估:
     UC-17 (P10): scene=driving ✓, mpic_active ✓ → ACTIVATE
     UC-11 (P10): scene=driving ✓, adas_active ✓ → ACTIVATE
     UC-13 (P9):  scene=driving ✓, adas_active ✓ → ACTIVATE
     UC-12 (P8):  scene=driving ✓ → ACTIVATE
     UC-10 (P8):  scene=driving ✓ → ACTIVATE
     UC-8  (P4):  scene=driving ✓, speed>30 ✓ → ACTIVATE (低优先)

t=2: Resource Orchestrator:
     Streams: FWC(30fps) + FTC(30fps) + MPIC(30fps) + SCF×2(30fps) + SCR×2(10fps)
     NPU Schedule:
       Cycle 1 (100ms):
         - Backbone_A(MPIC) → Head_A3, Head_A4          [UC-17]
         - Backbone_B(FWC)  → Head_B1-B5                [UC-10/11/12/13/8]
         - CV(FWC) → 行人手势/交警手势/RoadSign/Marking [UC-11/13]
       Cycle 2 (100ms):
         - Backbone_B(SCF_L) → Head_B3                  [UC-12]
         - Backbone_B(SCF_R) → Head_B3                  [UC-12]
         - CV(FWC) → 同上                               [UC-11/13]
   
     Late Fusion: 汇总各路结果 → UC 状态更新

t=3: Use Case Trigger:
     UC-11 检测到交警手势 (confidence=0.95) → 发送 ADAS Event
     UC-10 检测到天气=雨 → 更新环境状态
     UC-8  检测到景色=大海 → 低优先级通知
```

### 7.2 哨兵模式升级场景

```
t=0: Vehicle Signal → gear=P, engine=OFF, sentry=enabled
     Scene Classifier → primary_scene="sentry_mode"
     Power Profile → SLEEP

t=1: Active Resources:
     Streams: SVC×4 (5fps)
     Models: CV(人形检测), CV(运动检测)
     NPU Load: ~10%

t=2: [30分钟后] CV Signal → person_detected(svc_left, confidence=0.85)
   
t=3: Visual Decision Engine:
     UC-9 (P7): scene=sentry ✓, cv_trigger=person_detected ✓ → ESCALATE
   
t=4: Power Profile → SLEEP → LOW
     Stream Manager:
       SVC×4: 5fps → 10fps
       SCF_L: OFF → ACTIVE (30fps)  // 朝人的方向
     Model Scheduler:
       新增: CV(事件分类), CV(FaceId) on SCF_L
   
t=5: CV(事件分类) → "逗留" (confidence=0.88, 持续>30s)
     → 中级告警: 推送手机通知 + 开始录像
   
t=6: CV(事件分类) → "触碰车辆" (confidence=0.92)
     → 高级告警: 推送手机 + 响喇叭 + 闪灯 + 上传视频
```

languagelanguagelanguagelanguage---
