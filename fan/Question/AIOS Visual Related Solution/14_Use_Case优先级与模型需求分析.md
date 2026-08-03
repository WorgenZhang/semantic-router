# AIOS Visual Use Cases - 优先级与模型需求分析

## 评估维度说明

### Priority（优先级）
- 评分范围：1-10（值越高优先级越高）
- 评估依据：
  - **安全相关性**：涉及行车安全/ADAS 的场景权重最高
  - **用户核心体验**：高频使用、直接影响用户体验的场景
  - **技术就绪度**：模型已有/部分已有的场景可更快落地
  - **商业差异化**：形成产品竞争力的场景

### Model Frequency Requirement（模型调用频率需求）
- **Real-time (实时)**：每帧或接近每帧处理，延迟 < 100ms
- **High (高频)**：每秒多次，延迟 < 500ms
- **Medium (中频)**：每秒或数秒一次，延迟 < 2s
- **Low (低频)**：事件触发或用户触发，延迟可接受 2-5s
- **On-demand (按需)**：用户主动触发，延迟可接受 > 5s

### Model Accuracy Requirement（模型精度需求）
- **Critical (关键)**：> 99%，错误会导致安全事故或严重后果
- **High (高)**：> 95%，错误影响驾驶决策或核心功能
- **Medium (中)**：> 90%，错误影响用户体验但不涉及安全
- **Low (低)**：> 80%，允许一定误差，可通过交互修正

---

## 全量 Use Case 分析

| UC# | Use Case | Scenario | Priority | Frequency Requirement | Frequency (Hz) | Accuracy Requirement | Accuracy (%) | 优先级理由 |
|-----|----------|----------|:--------:|----------------------|:--------------:|---------------------|:------------:|-----------|
| 17 | ADAS 舱内乘客状态感知 | In-Cabin | **10** | Real-time | 10 Hz | Critical | 99% | ADAS 安全核心功能，驾驶员状态异常需立即接管 |
| 11 | ADAS 舱外动态目标感知 (手势) | Driving | **10** | Real-time | 10 Hz | Critical | 99% | 交警/行人手势直接影响驾驶决策，误判可能导致事故 |
| 13 | ADAS 舱外静态目标感知 | Driving | **9** | Real-time | 10 Hz | Critical | 99% | 路标/标线/车道识别是 ADAS 基础能力，关乎行车安全 |
| 2 | 盲区影像 (BSV) | Drop-Off | **9** | Real-time | 15 Hz | High | 97% | 转向/开门盲区涉及行人安全，误判后果严重 |
| 12 | ADAS 舱外动态目标感知 (特殊车辆) | Driving | **8** | High | 5 Hz | High | 95% | 特殊车辆需及时避让，影响行车安全和合规 |
| 14 | ADAS 停车场景舱外感知 | Parking | **8** | High | 5 Hz | High | 95% | 停车限制误判可能导致违规停车或安全隐患 |
| 10 | 舱外环境感知 | Driving | **8** | High | 2 Hz | High | 95% | 天气/光照感知为 ADAS 链路基础输入，影响下游决策 |
| 5 | 车内乘客特征/行为识别 | In-Cabin | **7** | High | 5 Hz | High | 95% | 儿童/危险行为识别涉及安全，但非实时驾驶决策 |
| 15 | ADAS 驾驶用户指令识别 | Driving | **7** | Medium | 1 Hz | High | 95% | 用户指令需准确理解，但有确认环节可容错 |
| 16 | ADAS 停车用户指令识别 | Parking | **7** | Medium | 2 Hz | High | 95% | 停车场景速度低，安全要求略低于行车 |
| 9 | 哨兵模式 | Sentry Mode | **7** | High | 5 Hz | Medium | 90% | 停车安防高频使用场景，但漏报可通过循环检测补偿 |
| 1 | 行车记录仪 (DVR) | Driving | **6** | — (无模型) | N/A | — | N/A | 基础功能，用户期望标配，但无 AI 模型需求 |
| 7 | 车外用户身份识别 | Out-Cabin | **6** | Low | 1 Hz | High | 98% | 身份识别对准确率要求高，但触发频率低 |
| 6 | 停车助手 | Parking | **5** | Low | 1 Hz | Medium | 90% | 便利功能，错误不影响安全，可重试 |
| 8 | 车外 AI 识别景色 | Driving | **4** | Medium | 1 Hz | Medium | 90% | 体验增值功能，错误不影响安全，容忍度高 |
| 3 | 相机&图库 | Driving | **4** | Low | 0.5 Hz | Medium | 85% | 娱乐/记录功能，用户可手动补偿 |
| 4 | 游戏 | In-Cabin | **3** | High | 15 Hz | Medium | 90% | 娱乐功能，优先级低但频率要求高（实时交互） |

---

## 按优先级分层

### Tier 1 — 安全关键（Priority 9-10）

| UC# | Use Case | Priority | Frequency | Hz | Accuracy | % |
|-----|----------|:--------:|-----------|:--:|----------|:-:|
| 17 | ADAS 舱内乘客状态感知 | 10 | Real-time | 10 | Critical | 99% |
| 11 | ADAS 舱外动态目标感知 (手势) | 10 | Real-time | 10 | Critical | 99% |
| 13 | ADAS 舱外静态目标感知 | 9 | Real-time | 10 | Critical | 99% |
| 2 | 盲区影像 (BSV) | 9 | Real-time | 15 | High | 97% |

> 这些场景直接关系行车安全，模型必须实时响应（10-15 Hz）、高精度（≥97%），是系统的基础能力。

### Tier 2 — 核心体验（Priority 7-8）

| UC# | Use Case | Priority | Frequency | Hz | Accuracy | % |
|-----|----------|:--------:|-----------|:--:|----------|:-:|
| 12 | ADAS 舱外动态目标感知 (特殊车辆) | 8 | High | 5 | High | 95% |
| 14 | ADAS 停车场景舱外感知 | 8 | High | 5 | High | 95% |
| 10 | 舱外环境感知 | 8 | High | 2 | High | 95% |
| 5 | 车内乘客特征/行为识别 | 7 | High | 5 | High | 95% |
| 15 | ADAS 驾驶用户指令识别 | 7 | Medium | 1 | High | 95% |
| 16 | ADAS 停车用户指令识别 | 7 | Medium | 2 | High | 95% |
| 9 | 哨兵模式 | 7 | High | 5 | Medium | 90% |

> 这些场景为 ADAS 增强能力或高频使用的核心功能（2-5 Hz, ≥90%），是产品差异化的关键。

### Tier 3 — 增值功能（Priority 4-6）

| UC# | Use Case | Priority | Frequency | Hz | Accuracy | % |
|-----|----------|:--------:|-----------|:--:|----------|:-:|
| 1 | 行车记录仪 (DVR) | 6 | — | N/A | — | N/A |
| 7 | 车外用户身份识别 | 6 | Low | 1 | High | 98% |
| 6 | 停车助手 | 5 | Low | 1 | Medium | 90% |
| 8 | 车外 AI 识别景色 | 4 | Medium | 1 | Medium | 90% |
| 3 | 相机&图库 | 4 | Low | 0.5 | Medium | 85% |
| 4 | 游戏 | 3 | High | 15 | Medium | 90% |

> 这些场景为用户体验加分项（0.5-15 Hz, 85-98%），可在核心能力稳定后逐步迭代。

---

## 频率-精度矩阵

|  | Critical (≥99%) | High (95-98%) | Medium (85-90%) | Low (<85%) |
|--|----------------|-------------|---------------|------------|
| **Real-time (10-15 Hz)** | UC17(10Hz), UC11(10Hz), UC13(10Hz) | UC2(15Hz) | | |
| **High (5 Hz)** | | UC12(5Hz), UC14(5Hz), UC10(2Hz), UC5(5Hz) | UC9(5Hz), UC4(15Hz) | |
| **Medium (1-2 Hz)** | | UC15(1Hz), UC16(2Hz), UC7(1Hz) | UC8(1Hz), UC6(1Hz) | |
| **Low (≤0.5 Hz)** | | | UC3(0.5Hz) | |

> 左上角（Real-time 10-15Hz + Critical ≥99%）是技术最具挑战性的区域，需优先保障算力和模型质量。

---

## 关键洞察

1. **4 个 Use Case 需要 Real-time + Critical**：UC17/11/13/2，这些是系统的 "硬约束"，模型选型和硬件配置必须以此为基线。

2. **端侧 VLM 是瓶颈**：Real-time 场景中若 CV 无法覆盖需回退 VLM，但 VLM 推理延迟通常无法满足 Real-time 要求，需考虑：
   - CV 模型尽量覆盖高频 pattern
   - VLM 仅处理 CV 低置信度 case（异步+缓存）

3. **哨兵模式性价比高**：Priority 7 但模型全部为 CV 且可复用人形/运动检测通用能力。

4. **Parking 场景集群适合打包交付**：UC6/14/16 共享车位检测、规则牌理解等模型，可作为 "智能泊车" feature 整体交付。

---

*分析基于：AIOS Visual Related Use Cases_数据表_原始.xlsx*
*生成日期：2025-05-21*
