## 具体示例：正常行驶中突然检测到交警手势

### 场景设定

车辆在城市道路行驶，速度 40km/h，ADAS 已激活，舱内有驾驶员+一名儿童乘客。此时前方出现交警做出停车手势。

---

### Step 1: 收集所有 VisualSignalMatches

```go
signals := VisualSignalMatches{
    SceneSignals:    ["driving", "in_cabin"],
    VehicleSignals:  ["gear_d", "speed_40", "adas_active"],
    CVTriggers:      ["person_detected(fwc, 0.91)", "child_detected(mpic, 0.95)", "gesture_detected(fwc, 0.72)"],
    SystemSignals:   ["power_normal", "npu_load_65", "thermal_normal", "dms_active"],
    SignalConfidences: {
        "cv:person_detected":  0.91,
        "cv:child_detected":   0.95,
        "cv:gesture_detected": 0.72,  // CV小模型对手势不太确定
    },
}
```

go---

### Step 2: 遍历所有 Use Case Decisions，评估规则树

基于 `aios_visual_router_config.yaml` 中定义的 decisions 规则逐一匹配：

| Decision Name                        | Rules (YAML定义)                                                                   | 匹配结果  | 说明                    |
| ------------------------------------ | ---------------------------------------------------------------------------------- | --------- | ----------------------- |
| `uc17_adas_cabin_perception`       | `scene=in_cabin AND scene=driving`                                               | ✅ 匹配   | 两个scene都激活         |
| `uc11_adas_gesture_recognition`    | `scene=driving AND adas_active=true`                                             | ✅ 匹配   | ADAS已激活              |
| `uc13_adas_static_targets`         | `scene=driving AND adas_active=true`                                             | ✅ 匹配   | 同上                    |
| `uc2_blind_spot_view`              | `(scene=driving AND turn_signal=true) OR (scene=drop_off AND door_opening=true)` | ❌ 不匹配 | 无转向/开门信号         |
| `uc12_special_vehicle_recognition` | `scene=driving`                                                                  | ✅ 匹配   | 行驶中                  |
| `uc14_parking_restriction`         | `scene=parking`                                                                  | ❌ 不匹配 | 非停车场景              |
| `uc10_environment_perception`      | `scene=driving`                                                                  | ✅ 匹配   | 行驶中                  |
| `uc5_passenger_behavior`           | `scene=in_cabin`                                                                 | ✅ 匹配   | dms_active=true         |
| `uc15_driving_user_command`        | `scene=driving AND user_query_active=true`                                       | ❌ 不匹配 | 无用户指令              |
| `uc16_parking_user_command`        | `scene=parking AND user_query_active=true`                                       | ❌ 不匹配 | 非停车场景              |
| `uc9_sentry_mode`                  | `scene=sentry_mode`                                                              | ❌ 不匹配 | 非哨兵模式              |
| `uc7_user_identity`                | `scene=out_cabin AND person_detected_svc>0.7`                                    | ❌ 不匹配 | 非out_cabin场景         |
| `uc6_parking_assistant`            | `scene=parking AND NOT user_query_active`                                        | ❌ 不匹配 | 非停车场景              |
| `uc8_scenery_recognition`          | `scene=driving AND speed>30 AND NOT adas_critical_event`                         | ✅ 匹配   | speed=40>30，无紧急事件 |
| `uc3_camera_gallery`               | `scene=driving AND NOT adas_critical_event`                                      | ✅ 匹配   | 行驶中无紧急事件        |
| `uc4_game_gesture`                 | `scene=in_cabin AND game_mode_active=true`                                       | ❌ 不匹配 | 游戏模式未开启          |

---

### Step 3: 过滤已匹配的 Decisions

匹配通过的 Decision 列表：

```
uc17_adas_cabin_perception
uc11_adas_gesture_recognition
uc13_adas_static_targets
uc12_special_vehicle_recognition
uc10_environment_perception
uc5_passenger_behavior
uc8_scenery_recognition
uc3_camera_gallery
```

language---

### Step 4: 按 Strategy 排序选择

YAML 定义 `strategy: "priority_power_aware"`，按以下子策略逐步筛选：

#### 4a. Priority 排序（YAML中定义的数值优先级）

```
P100: uc17_adas_cabin_perception       — tier: safety, preemptible: false
P100: uc11_adas_gesture_recognition    — tier: safety, preemptible: false
P95:  uc13_adas_static_targets         — tier: safety, preemptible: false
P80:  uc12_special_vehicle_recognition — tier: core,   preemptible: true
P80:  uc10_environment_perception      — tier: core,   preemptible: true
P75:  uc5_passenger_behavior           — tier: core,   preemptible: true
P40:  uc8_scenery_recognition          — tier: enhancement, preemptible: true
P40:  uc3_camera_gallery               — tier: enhancement, preemptible: true

  - preemptible: true（可抢占）：当更高优先级的 UC 需要 NPU 资源时，这个 UC 的推理任务可以被中断、降频或跳过，把算力让给别人。
  - preemptible: false（不可抢占）：无论系统多繁忙，这个 UC 的推理任务必须按时完成，不能被任何其他 UC 打断或延迟。

```

#### 4b. Power-Aware 判断

当前 `power_mode=NORMAL`（对应 YAML 中 `max_npu_load_percent: 70`）。

逐个检查 `min_power_mode`：

- uc17: `min_power_mode: "normal"` → ✅ 满足
- uc11: `min_power_mode: "normal"` → ✅ 满足
- uc13: `min_power_mode: "normal"` → ✅ 满足
- uc12: `min_power_mode: "normal"` → ✅ 满足
- uc10: `min_power_mode: "normal"` → ✅ 满足
- uc5:  `min_power_mode: "low"` → ✅ 满足（normal > low）
- uc8:  `min_power_mode: "normal"` → ✅ 满足
- uc3:  `min_power_mode: "normal"` → ✅ 满足

全部通过 Power-Aware 检查。

#### 4c. Resource-Constrained 降级决策

YAML 调度约束：

```yaml
npu_config:
  max_concurrent_models: 1          # NPU同时只跑一个模型
  scheduling_strategy: "priority_preempt"
  time_slice_ms: 30                 # 每模型30ms时间片
resource_conflict_resolution:
  same_backbone_shared: true        # 共享backbone不构成冲突
  vlm_async: true                   # VLM异步不阻塞
```

yamlNPU 调度分析（100ms 一个推理周期）：

```
Backbone_A (MPIC输入, 25ms):
  → head_a2_person_type (0.1ms)    [uc17 + uc5 共享]
  → head_a3_behavior (0.1ms)       [uc17 + uc5 共享]
  → head_a4_fatigue (0.1ms)        [uc17]
  小计: ~26ms

Backbone_B (FWC输入, 30ms):
  → head_b1_weather (0.1ms)        [uc10]
  → head_b2_lighting (0.1ms)       [uc10]
  → head_b3_special_vehicle (0.1ms)[uc12 + uc11]
  → head_b4_scene_type (0.1ms)     [uc13]
  → head_b5_scenery (0.1ms)        [uc8 + uc3]
  小计: ~31ms

CV Models 串行 (time_slice_ms=30 each):
  → cv_child_detection (10ms)       [uc17 + uc5]
  → cv_fatigue_detection (12ms)     [uc17 + uc5]
  → cv_pedestrian_gesture (15ms)    [uc11]
  → cv_traffic_police_gesture (15ms)[uc11]
  → cv_road_sign (12ms)            [uc13]
  → cv_road_marking (10ms)         [uc13]
  → cv_special_lane (10ms)         [uc13]
  → cv_zone_area (10ms)            [uc13]
  小计: ~94ms (可在一个周期内完成)

总计单周期: 26 + 31 + 94 = ~151ms (需要2个time slice完成)
```

languageNORMAL模式 NPU上限 70%，当前基础负载 65%，8个UC全量运行会超限。

**降级决策（priority_preempt 策略）：**

- `safety_tier_preempts_all: true` → uc17/uc11/uc13 保证每轮调度
- P80 的 uc12/uc10：`preemptible: true`，降低调度频率
- P75 的 uc5：`preemptible: true`，共享 Backbone_A 输出不额外占用
- P40 的 uc8/uc3：`degradation.disable_entirely: true`，资源紧张时完全关闭

**最终降级方案：**

```
uc8:  disable_entirely (YAML定义: reason: "lowest priority, disable under resource pressure")
uc3:  disable_entirely (同上)
uc12: reduce_fps_to: 2 (从5Hz降到2Hz), drop_secondary_streams: true (去掉SCR×2)
uc10: reduce_fps_to: 1 (从2Hz降到1Hz), drop_secondary_streams: true (去掉SCF)
```

language---

### Step 5: 输出激活的 Use Case 列表 + 对应资源方案

```yaml
activated_use_cases:
  - decision: uc17_adas_cabin_perception
    priority: 100
    tier: safety
    streams:
      - camera: MPIC
        fps: 10
        required: true
    pipeline:
      type: cv_vit_parallel
      cv_models: [cv_child_detection, cv_fatigue_detection]
      backbone: backbone_a
      heads: [head_a2_person_type, head_a3_behavior, head_a4_fatigue]
      fusion: parallel_merge
    multi_stream:
      strategy: single
      primary: MPIC
    scheduling:
      frequency_hz: 10
      max_latency_ms: 100
      preemptible: false

  - decision: uc11_adas_gesture_recognition
    priority: 100
    tier: safety
    streams:
      - camera: FWC
        fps: 10
        required: true
      - camera: SCF_L
        fps: 10
        required: true
      - camera: SCF_R
        fps: 10
        required: true
    pipeline:
      type: cv_vit_parallel
      cv_models: [cv_pedestrian_gesture, cv_traffic_police_gesture]
      backbone: backbone_b
      heads: [head_b3_special_vehicle]
      fusion:
        strategy: confidence_gate
        cv_threshold: 0.8
        fallback: vlm_edge
    multi_stream:
      strategy: sync_late_fusion
      primary: FWC
      weights: {FWC: 0.5, SCF_L: 0.25, SCF_R: 0.25}
    scheduling:
      frequency_hz: 10
      max_latency_ms: 100
      preemptible: false
    note: "gesture_confidence=0.72 < cv_threshold=0.8 → 触发ViT验证 + vlm_edge fallback"

  - decision: uc13_adas_static_targets
    priority: 95
    tier: safety
    streams:
      - camera: FWC
        fps: 10
        required: true
      - camera: FTC
        fps: 5
        required: true
      - camera: SCF_L
        fps: 5
        required: false
      - camera: SCF_R
        fps: 5
        required: false
    pipeline:
      type: cv_vit_parallel
      cv_models: [cv_road_sign, cv_road_marking, cv_special_lane, cv_zone_area]
      backbone: backbone_b  # 与uc11共享，same_backbone_shared=true
      heads: [head_b4_scene_type]
      fusion:
        strategy: confidence_gate
        cv_threshold: 0.85
        fallback: vlm_edge
    multi_stream:
      strategy: primary_secondary
      primary: FWC
      secondary: FTC
      secondary_purpose: long_range_text_recognition
      primary_confidence_threshold: 0.7
    scheduling:
      frequency_hz: 10
      max_latency_ms: 100
      preemptible: false

  - decision: uc12_special_vehicle_recognition
    priority: 80
    tier: core
    streams:
      - camera: FWC
        fps: 2          # 降级: reduce_fps_to: 2 (原5Hz)
        required: true
      - camera: SCF_L
        fps: 2
        required: true
      - camera: SCF_R
        fps: 2
        required: true
      # SCR_L/SCR_R dropped: drop_secondary_streams: true
    pipeline:
      type: only_vit
      backbone: backbone_b  # 共享，不额外占用NPU
      heads: [head_b3_special_vehicle]
      fusion: none
    multi_stream:
      strategy: sync_late_fusion
      primary: FWC
      weights: {FWC: 0.4, SCF_L: 0.3, SCF_R: 0.3}  # 去掉SCR后重分配
    scheduling:
      frequency_hz: 2   # 降级后
      max_latency_ms: 200
      preemptible: true

  - decision: uc10_environment_perception
    priority: 80
    tier: core
    streams:
      - camera: FWC
        fps: 1          # 降级: reduce_fps_to: 1 (原2Hz)
        required: true
      # SCF_L/SCF_R dropped: drop_secondary_streams: true
    pipeline:
      type: only_vit
      backbone: backbone_b  # 共享
      heads: [head_b1_weather, head_b2_lighting, head_b4_scene_type]
      fusion: none
    multi_stream:
      strategy: primary_secondary
      primary: FWC
      primary_confidence_threshold: 0.6
    scheduling:
      frequency_hz: 1   # 降级后
      max_latency_ms: 500
      preemptible: true

  - decision: uc5_passenger_behavior
    priority: 75
    tier: core
    streams:
      - camera: MPIC
        fps: 5
        required: true
    pipeline:
      type: cv_vit_parallel
      cv_models: [cv_child_detection, cv_fatigue_detection]  # 与uc17共享CV结果
      backbone: backbone_a  # 与uc17共享，same_backbone_shared=true
      heads: [head_a2_person_type, head_a3_behavior]
      fusion: parallel_merge
    multi_stream:
      strategy: single
      primary: MPIC
    scheduling:
      frequency_hz: 5
      max_latency_ms: 200
      preemptible: true

  # --- 以下因资源约束被关闭 ---
  - decision: uc8_scenery_recognition
    priority: 40
    tier: enhancement
    status: DISABLED
    reason: "degradation.disable_entirely=true, NPU负载超限时最低优先级完全关闭"

  - decision: uc3_camera_gallery
    priority: 40
    tier: enhancement
    status: DISABLED
    reason: "degradation.disable_entirely=true, 同上"
```

yaml---

### 关键洞察

这个例子体现了 `aios_visual_router_config.yaml` 定义的四个核心机制：

1. **`same_backbone_shared: true`**：uc11/uc12/uc13/uc10/uc8 共享 `backbone_b`（30ms推理一次），多个 Head 并行输出（每个仅 0.1ms），不构成 NPU 资源竞争
2. **`confidence_gate` 融合策略**：uc11 定义 `cv_threshold: 0.8`，当前 gesture_confidence=0.72 < 0.8，自动触发 ViT backbone_b 二次验证，若仍不足则 fallback 到 `vlm_edge`
3. **`priority_preempt` + `safety_tier_preempts_all`**：P100/P95 的 safety tier（uc17/uc11/uc13）设置 `preemptible: false`，永远保证调度；P40 的 enhancement tier 设置 `disable_entirely: true`，资源不足时直接关闭
4. **`degradation` 分级降级**：YAML 中每个 decision 定义了独立降级策略——core tier 通过 `reduce_fps_to` + `drop_secondary_streams` 降档运行，enhancement tier 通过 `disable_entirely` 完全让出资源
