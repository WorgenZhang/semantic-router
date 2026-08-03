# AIOS Visual Related Use Cases

## 概览

本文档整理了 AIOS 视觉相关的全部 Use Case，按场景类别分类，涵盖舱内感知、行车场景、停车场景、下客场景和哨兵模式。

---

## 按场景分类汇总

| 场景类别 | Use Case 数量 | 典型能力 |
|---------|:---:|---------|
| In-Cabin | 3 | 手势识别、乘客特征/行为、ADAS 舱内感知 |
| Driving | 8 | 景色识别、环境感知、手势/特殊车辆、路标、用户指令 |
| Parking | 3 | 停车助手、车位限制理解、语音/手势泊车 |
| Out-Cabin | 1 | 车外用户身份识别 |
| Drop-Off | 1 | 盲区影像（开门安全） |
| Sentry Mode | 1 | 哨兵模式（停车监控） |

---

## In-Cabin（舱内场景）

### UC-4: 游戏（手势识别操控）

| 字段 | 内容 |
|------|------|
| **Priority** | 3 / 10 |
| **Description** | 休闲游戏：手势识别操控游戏 |
| **User Scenario** | 提供座舱娱乐的更多方式 |
| **AIOS Related** | Strongly-Related |
| **Camera Streams** | 1×MPIC |
| **Models Needed** | CV: 手势识别 |
| **Execution Sequence** | Only CV |
| **Model Status** | CV: Already Exists |
| **Frequency** | 15 Hz |
| **Accuracy** | 90% |
| **Frequency Req** | High |
| **Accuracy Req** | Medium |

---

### UC-5: 车内乘客特征/行为识别

| 字段 | 内容 |
|------|------|
| **Priority** | 7 / 10 |
| **Description** | 判定车内人的年龄、性别等；判定车内人的行为（睡觉、危险动作等） |
| **User Scenario** | 针对老年/儿童/身体障碍人群的功能（放大字体音量、儿童模式等）；识别动作做功能推送（睡眠模式、危险动作语音警示） |
| **AIOS Related** | Strongly-Related |
| **Camera Streams** | 1×MPIC |
| **Models Needed** | CV: 儿童识别、睡眠识别 (TBD: 老人、障碍人群、危险动作)<br>VLM: 老人/障碍人群/危险动作 (CV Cover 不住的场景) |
| **Execution Sequence** | CV + VLM Parallel |
| **Model Status** | CV: Exist 儿童识别、疲劳检测；TBD: 老人、障碍人群、危险动作<br>VLM: TBD |
| **Frequency** | 5 Hz |
| **Accuracy** | 95% |
| **Frequency Req** | High |
| **Accuracy Req** | High |

---

### UC-17: ADAS 舱内乘客状态感知

| 字段 | 内容 |
|------|------|
| **Priority** | 10 / 10 |
| **Description** | Recognize driver's/passenger status (uncomfortable, child crying), takeover and pull over; Recognize driver and passenger, adjust driving styles based on preference and scenarios (Eating, Making Up) |
| **User Scenario** | 舱内人员状态、舱内人员 ID/type |
| **AIOS Related** | Strongly-Related |
| **Camera Streams** | 1×MPIC |
| **Models Needed** | CV: 儿童识别、睡眠识别 (TBD: 老人、障碍人群、危险动作、吃东西、化妆)<br>VLM: 老人/障碍人群/危险动作 (CV Cover 不住的场景) |
| **Execution Sequence** | CV + VLM Parallel |
| **Model Status** | CV: Exist 儿童识别、疲劳检测；TBD: 老人、障碍人群、危险动作<br>VLM: TBD |
| **Frequency** | 10 Hz |
| **Accuracy** | 99% |
| **Frequency Req** | Real-time |
| **Accuracy Req** | Critical |

---

## Out-Cabin（舱外场景）

### UC-7: 车外用户身份识别

| 字段 | 内容 |
|------|------|
| **Priority** | 6 / 10 |
| **Description** | 检测到车外有人，自动识别用户身份 |
| **User Scenario** | 车主走向车辆 → 自动识别身份 → 解锁/迎宾灯/调座椅；配合车外语音交互（声纹+人脸） |
| **AIOS Related** | Strongly-Related |
| **Camera Streams** | 4×SVC + 2×SCR + 2×SCF<br>(4个 SVC 低帧率 5fps 覆盖 360° → 检测到人后唤醒 1 个 SC) |
| **Models Needed** | CV1: 车外乘客识别 (SVC)<br>CV2: 车外乘客 FaceId 识别 (SCR or SCF) |
| **Execution Sequence** | CV1 → CV2 |
| **Model Status** | CV1: TBD (目前没有可用舱外识别 CV 小模型)<br>CV2: TBD |
| **Frequency** | 1 Hz |
| **Accuracy** | 98% |
| **Frequency Req** | Low |
| **Accuracy Req** | High |

---

## Driving（行车场景）

### UC-1: 行车记录仪 (DVR)

| 字段 | 内容 |
|------|------|
| **Priority** | 6 / 10 |
| **Description** | 主视角/全视角录制、循环录制、紧急录制、鸣笛录制、叠加车辆信息 |
| **User Scenario** | 正常行驶/倒车/转弯，同时录制主视角和全视角，全面记录路况 |
| **AIOS Related** | Non-Related |
| **Camera Streams** | — |
| **Models Needed** | — |
| **Execution Sequence** | — |
| **Model Status** | — |
| **Frequency** | N/A |
| **Accuracy** | N/A |

---

### UC-3: 相机 & 图库

| 字段 | 内容 |
|------|------|
| **Priority** | 4 / 10 |
| **Description** | 车外/车内录制拍照、内外双摄、一键旅拍、AIGC 拍照、AIGC 艺术屏保 |
| **User Scenario** | 自驾旅行偶遇美景时便捷多角度拍摄；停车等人/充电间隙通过 AIGC 功能创造作品 |
| **AIOS Related** | Maybe-Related (主动识别美景) |
| **Camera Streams** | 1×FWC + 2×SCF |
| **Models Needed** | VLM: 车外美景识别 (草原、烟花、大海等) |
| **Execution Sequence** | Only VLM |
| **Model Status** | VLM: TBD |
| **Frequency** | 0.5 Hz |
| **Accuracy** | 85% |
| **Frequency Req** | Low |
| **Accuracy Req** | Medium |

---

### UC-8: 车外 AI 识别景色

| 字段 | 内容 |
|------|------|
| **Priority** | 4 / 10 |
| **Description** | 主动识别推荐/记录美好瞬间；景色识别→氛围生成；景色识别→主动拍照 |
| **User Scenario** | 行车中利用舱外摄像头识别地标/动物/植物，自动弹出介绍并拍照留念；根据景色在舱内生成沉浸式氛围 |
| **AIOS Related** | Strongly-Related |
| **Camera Streams** | 1×FWC + 2×SCF |
| **Models Needed** | VLM: 地标识别/动植物/日出日落/湖海山林 |
| **Execution Sequence** | Only VLM |
| **Model Status** | VLM: TBD |
| **Frequency** | 1 Hz |
| **Accuracy** | 90% |
| **Frequency Req** | Medium |
| **Accuracy Req** | Medium |

---

### UC-10: 舱外环境感知

| 字段 | 内容 |
|------|------|
| **Priority** | 8 / 10 |
| **Description** | 舱外环境感知通用理解能力；车外景点视觉识别；车外识物 |
| **User Scenario** | 通过外部摄像头感知周围环境（盲区车辆/行人/天气/光照），为安全提醒和主动推荐提供数据；车外通用视觉能力联动前后左右摄像头；景点/楼栋等舱外信息识别；用户可询问车外地理位置 |
| **AIOS Related** | Strongly-Related |
| **Camera Streams** | 1×FWC + 2×SCR + 2×SCF<br>(车外行人/车辆安全提醒场景见盲区检测 case) |
| **Models Needed** | 端 VLM: 天气分类 (晴/阴/雨/雪/雾) / 光照判断 (白天/黄昏/夜间/隧道/逆光)<br>云 VLM: 通用视觉查询/闲聊 |
| **Execution Sequence** | 端 VLM: 高频 Perception<br>云 VLM: 根据 Query 复杂度/场景决定是否上云 |
| **Model Status** | 端 VLM: TBD<br>云 VLM: TBD |
| **Frequency** | 2 Hz |
| **Accuracy** | 95% |
| **Frequency Req** | High |
| **Accuracy Req** | High |

---

### UC-11: ADAS 舱外动态目标感知 — 手势识别

| 字段 | 内容 |
|------|------|
| **Priority** | 10 / 10 |
| **Description** | 交警手势识别、行人手势识别 (rider) |
| **User Scenario** | 行人手势、交警手势 |
| **AIOS Related** | Strongly-Related |
| **Camera Streams** | 1×FWC + 2×SCF |
| **Models Needed** | CV1: 行人手势识别<br>CV2: 交警手势识别<br>VLM: CV Cover 不住时的手势场景 |
| **Execution Sequence** | CV + VLM Parallel |
| **Model Status** | CV1/CV2: TBD<br>VLM: TBD |
| **Frequency** | 10 Hz |
| **Accuracy** | 99% |
| **Frequency Req** | Real-time |
| **Accuracy Req** | Critical |

---

### UC-12: ADAS 舱外动态目标感知 — 特殊车辆识别

| 字段 | 内容 |
|------|------|
| **Priority** | 8 / 10 |
| **Description** | Special vehicle recognition (ambulance, police, fire truck) |
| **User Scenario** | 特殊车辆识别（警车、救护车、消防车） |
| **AIOS Related** | Strongly-Related |
| **Camera Streams** | 1×FWC + 2×SCF + 2×SCR |
| **Models Needed** | VLM: 特殊车辆识别 (警车、救护车、消防车) |
| **Execution Sequence** | Only VLM |
| **Model Status** | VLM: TBD |
| **Frequency** | 5 Hz |
| **Accuracy** | 95% |
| **Frequency Req** | High |
| **Accuracy Req** | High |

---

### UC-13: ADAS 舱外静态目标感知 — 路标/车道/区域

| 字段 | 内容 |
|------|------|
| **Priority** | 9 / 10 |
| **Description** | Road sign, marking; Special lane (bus lane, tidal lanes, variable lanes, ETC lane); Zone/area (school zones, crosswalks, accident scene) |
| **User Scenario** | Road Sign (限速牌/禁止牌/指示牌/警告牌)<br>Road Marking (车道线/箭头/文字标线/停止线/减速标线)<br>Special Lane (公交专用道/可变车道/潮汐车道/ETC 车道)<br>Zone/Area (学校区域/人行横道/事故现场) |
| **AIOS Related** | Strongly-Related |
| **Camera Streams** | 1×FWC (路牌/标线/区域全覆盖) + 1×FTC (远距离路牌文字识别) + 2×SCF |
| **Models Needed** | CV1: Road Sign<br>CV2: Road Marking<br>CV3: Special Lane<br>CV4: Zone/Area<br>VLM: Cover CV 无法覆盖场景 |
| **Execution Sequence** | CV + VLM Parallel |
| **Model Status** | CV: TBD<br>VLM: TBD |
| **Frequency** | 10 Hz |
| **Accuracy** | 99% |
| **Frequency Req** | Real-time |
| **Accuracy Req** | Critical |

---

### UC-15: ADAS 驾驶场景下用户指令识别

| 字段 | 内容 |
|------|------|
| **Priority** | 7 / 10 |
| **Description** | Call for ADAS action (stop/follow/lane change); Call for question (car brand/price ahead, bus lane availability); Target object as condition (dynamic & static); User memory & preference (lane, route, specific object) |
| **User Scenario** | 目标指令："在那栋蓝色玻璃大楼前面停车"、"在那家星巴克旁边停"、"让那个推婴儿车的人先过"、"跟着前面那辆红色的车"、"以后这条路下班时段都走最右道" |
| **AIOS Related** | Strongly-Related |
| **Camera Streams** | 1×FWC (正前方) + 2×SCR + 2×SCF (旁边/右边) |
| **Models Needed** | VLM: 泛化图片中物体解析 (蓝色玻璃大楼、推婴儿车的人、红色车等) |
| **Execution Sequence** | Only VLM? |
| **Model Status** | VLM: TBD |
| **Frequency** | 1 Hz |
| **Accuracy** | 95% |
| **Frequency Req** | Medium |
| **Accuracy Req** | High |

---

## Parking（停车场景）

### UC-6: 停车助手

| 字段 | 内容 |
|------|------|
| **Priority** | 5 / 10 |
| **Description** | 进场识别；舱外收费二维码 |
| **User Scenario** | 进场识别：车辆成功进入/离开停车场（抬杆识别）<br>舱外收费二维码：行车中任何角度扫描车外二维码（支付宝算法要求：像素密度 >1.4 pixel/bar） |
| **AIOS Related** | Maybe-Related (抬杆识别、收费亭识别)<br>TBD: ADAS 目前是否能完成 |
| **Camera Streams** | 1×FWC + 2×SCF |
| **Models Needed** | VLM: 抬杆识别、收费亭识别 |
| **Execution Sequence** | Only VLM |
| **Model Status** | VLM: TBD |
| **Frequency** | 1 Hz |
| **Accuracy** | 90% |
| **Frequency Req** | Low |
| **Accuracy Req** | Medium |

---

### UC-14: ADAS 停车场景舱外感知 — 车位限制理解

| 字段 | 内容 |
|------|------|
| **Priority** | 8 / 10 |
| **Description** | Parking restriction understanding: private spots, monthly spots, fire lanes, visitor spots, charging-only spots, time-limited parking |
| **User Scenario** | Private Spots (地面喷字/地锁/车位号)<br>Monthly Spots (地面喷字/车位牌/编号)<br>Fire Lanes (黄色网格线 + 禁止停车/路缘石涂装/墙面标识牌) |
| **AIOS Related** | Strongly-Related |
| **Camera Streams** | 地面喷字: FWC (看远) + SVC_F (看近处地面)<br>地锁/充电桩: FWC + SVC_F (近距离低矮物体)<br>路侧/墙面标识牌: FWC + SC_FL/SC_FR (侧方) |
| **Models Needed** | CV1: 车位检测 (Private/Monthly/Visitor)<br>CV2: Fire Lanes<br>CV3: Charging-Only Spots<br>VLM: 复杂自然语言规则牌理解 |
| **Execution Sequence** | CV or VLM? |
| **Model Status** | CV: TBD<br>VLM: TBD |
| **Frequency** | 5 Hz |
| **Accuracy** | 95% |
| **Frequency Req** | High |
| **Accuracy Req** | High |

---

### UC-16: ADAS 停车场景用户指令识别（语音/手势泊车）

| 字段 | 内容 |
|------|------|
| **Priority** | 7 / 10 |
| **Description** | Voice-guided parking spot search (near elevator, specific floor/zone, charging spot, accessible spot, spacious spot); Voice/gesture guided park-in/out; 舱外人员 gesture recognition |
| **User Scenario** | 语音找车位 (elevator, specific floor/zone, charging-only)；手势引导泊入泊出 |
| **AIOS Related** | Strongly-Related |
| **Camera Streams** | 语音找车位: 1×FWC + 2×SCF + 2×SCR + 4×SVC<br>手势引导: 1×FWC + 4×SVC + 2×SCF + 2×SCR + 1×RVC |
| **Models Needed** | 语音找车位:<br>- CV1: 车位检测 (空/占用/半占用)<br>- CV2: 目标检测 (充电桩/电梯门/指示牌/柱子/障碍物)<br>- OCR: 读取楼层号 & 指示牌<br>- VLM: 兜底<br>手势引导: CV 手势识别 |
| **Execution Sequence** | CV or VLM? |
| **Model Status** | CV: TBD<br>VLM: TBD |
| **Frequency** | 2 Hz |
| **Accuracy** | 95% |
| **Frequency Req** | Medium |
| **Accuracy Req** | High |

---

## Drop-Off（下客场景）

### UC-2: 盲区影像 (BSV)

| 字段 | 内容 |
|------|------|
| **Priority** | 9 / 10 |
| **Description** | 行车转向时展示左右盲区影像；开门前展示后侧视频流 |
| **User Scenario** | 驾驶员：转向时弹出侧后方摄像头画面辅助转向<br>乘客：开门时弹出侧后方画面判断开门安全性 |
| **AIOS Related** | Maybe-Related (乘客开门场景)<br>TBD: ADAS 目前是否能完成 |
| **Camera Streams** | 2×SCR |
| **Models Needed** | CV: 车外乘客识别 (更多类别可能 Cover 不住，如自行车/电动车/汽车)<br>VLM: 车外非乘客识别 (CV Cover 不住的场景) |
| **Execution Sequence** | CV + VLM Parallel |
| **Model Status** | CV: TBD (目前没有可用舱外识别 CV 小模型)<br>VLM: TBD |
| **Frequency** | 15 Hz |
| **Accuracy** | 97% |
| **Frequency Req** | Real-time |
| **Accuracy Req** | High |

---

## Sentry Mode（哨兵模式）

### UC-9: 哨兵模式

| 字段 | 内容 |
|------|------|
| **Priority** | 7 / 10 |
| **Description** | 全天候停车监控、远程实时查看 |
| **User Scenario** | 车辆停放时进入值守状态，24 小时不间断监控周围环境；通过手机 App 远程调起车外摄像头查看（停车场找车/防盗） |
| **AIOS Related** | Strongly-Related |
| **Camera Streams** | 4×SVC (Low-Power) + 2×SCF + 2×SCR |
| **Models Needed** | CV1: 人形检测 (有人靠近 → 告警)<br>CV2: 运动检测 (有物体移动 → 开始录像)<br>CV3: 事件分类 (路过/逗留/触碰/砸窗，不同级别告警) |
| **Execution Sequence** | CV1 + CV2 + CV3 Parallel |
| **Model Status** | CV1/CV2/CV3: TBD |
| **Frequency** | 5 Hz |
| **Accuracy** | 90% |
| **Frequency Req** | High |
| **Accuracy Req** | Medium |

---

## Priority 排序（降序）

| Priority | Use Case No. | Use Case |
|:---:|:---:|---------|
| 10 | UC-17 | ADAS 舱内乘客状态感知 |
| 10 | UC-11 | ADAS 舱外动态目标感知 — 手势识别 |
| 9 | UC-13 | ADAS 舱外静态目标感知 — 路标/车道/区域 |
| 9 | UC-2 | 盲区影像 (BSV) |
| 8 | UC-10 | 舱外环境感知 |
| 8 | UC-12 | ADAS 舱外动态目标感知 — 特殊车辆识别 |
| 8 | UC-14 | ADAS 停车场景舱外感知 — 车位限制理解 |
| 7 | UC-5 | 车内乘客特征/行为识别 |
| 7 | UC-15 | ADAS 驾驶场景下用户指令识别 |
| 7 | UC-16 | ADAS 停车场景用户指令识别 |
| 7 | UC-9 | 哨兵模式 |
| 6 | UC-7 | 车外用户身份识别 |
| 6 | UC-1 | 行车记录仪 (DVR) |
| 5 | UC-6 | 停车助手 |
| 4 | UC-3 | 相机 & 图库 |
| 4 | UC-8 | 车外 AI 识别景色 |
| 3 | UC-4 | 游戏（手势识别） |

---

## 模型能力需求总结

### 按 Accuracy Requirement 分级

| 级别 | 要求 | Use Cases |
|------|------|-----------|
| **Critical** (99%) | Real-time + 零容错 | UC-17 舱内感知, UC-11 手势识别, UC-13 路标识别 |
| **High** (95-98%) | 高频 + 高精度 | UC-5 乘客行为, UC-7 身份识别, UC-10 环境感知, UC-12 特殊车辆, UC-14 车位限制, UC-15/16 用户指令, UC-2 盲区影像 |
| **Medium** (85-90%) | 中低频 + 容忍误差 | UC-4 游戏, UC-3 相机, UC-8 景色, UC-6 停车助手, UC-9 哨兵 |

### 模型类型分布

| 模型类型 | Use Case 数量 | 说明 |
|---------|:---:|------|
| Only CV | 3 | UC-4 游戏, UC-1 DVR, UC-9 哨兵 |
| Only VLM | 5 | UC-3 相机, UC-8 景色, UC-12 特殊车辆, UC-6 停车助手, UC-15 用户指令 |
| CV + VLM Parallel | 6 | UC-5 乘客行为, UC-17 舱内感知, UC-11 手势, UC-13 路标, UC-2 盲区, UC-14 车位 |
| CV → CV (级联) | 1 | UC-7 身份识别 |
| CV + OCR + VLM | 1 | UC-16 停车指令 |

### 摄像头使用频率

| 摄像头 | 使用次数 | 典型用途 |
|--------|:---:|---------|
| FWC | 12 | 主视角：路标/景色/环境/手势/车辆 |
| SCF (×2) | 11 | 侧前方：景色补充/盲区/特殊车辆 |
| SCR (×2) | 7 | 侧后方：特殊车辆/盲区/身份识别 |
| SVC (×4) | 5 | 环视：身份识别/哨兵/停车 |
| MPIC (×1) | 3 | 舱内：手势/乘客行为/状态感知 |
| FTC (×1) | 1 | 远距离文字识别 |
| RVC (×1) | 1 | 后视：手势泊车 |
