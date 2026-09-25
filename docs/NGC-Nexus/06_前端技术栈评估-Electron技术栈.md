# 前端技术栈评估：Electron / Chromium 技术栈替代 Qt QML

- 评估对象：QGroundControl `mavlink/qgroundcontrol`，HEAD `25185047e855937d694a0174c1541d8f7d794a74`
- 本地源码：`15_NGC-Nexus/repo`
- 需求澄清：目标**不是**"把地面站做成网页"，而是**采用 Electron 这类基于 Chromium 内核的软件技术栈**（自带原生宿主，不是浏览器）
- 方法：源码级实测统计（本机无 Qt6/MSVC，未编译）+ 外部同类项目核实（附 URL）
- 结论口径：数字为实测；架构判断标注推导依据；外部信息给 URL 并标注为外部证据

---

## 0. 结论

**这个方向成立，且是本项目最合理的技术路线。** 理由与上一版"纯浏览器"评估的结论有实质差异：

| 关键点 | 纯浏览器方案（上一版） | **Electron 方案（本版）** |
|---|---|---|
| 串口 / 蓝牙 | ❌ 浏览器无法访问，必须原生桥 | ✅ Node 侧 `serialport` 直接可用 |
| 摇杆 | ❌ 需额外原生桥 | ✅ Gamepad API / Node HID |
| 宿主 | 无（需用户开浏览器） | ✅ 自带 Chromium + Node 运行时 |
| 离线能力 | 受限 | ✅ 本地文件、本地库、本地缓存全可用 |
| 进程/系统集成 | 无 | ✅ 托盘、通知、自启、文件关联 |

**结论：Electron 消掉了我上一版列出的绝大多数"硬骨头"。** 真正剩下的难点只有图传，以及"要不要保留 QGC 的 C++ 逻辑层"这个决策。

---

## 1. 实测事实（不变）

| 层 | 文件数 | 行数 |
|---|---|---|
| QML（UI 层） | 478 | **57,273** |
| C/C++（业务逻辑层） | 1304 | **193,082** |

绑定密度：`Q_PROPERTY` 1272、`Q_INVOKABLE` 469、`QObject` 1188、`QML_ELEMENT` 116。

Qt6 组件清单（`CMakeLists.txt:252-279`）中：
- **`HttpServer` 已是 REQUIRED 组件**，但全仓 `QHttpServer` 零命中 → **嵌入式 HTTP 服务端现成可用，只是没用起来**
- `WebEngine` / `WebView` **不在清单里** → 若想"QGC 内嵌 WebView"需新增模块（本方案不需要，因为 Electron 自带内核）

平台：`deploy/` 覆盖 Windows / Linux / macOS / Android / iOS / Docker。

---

## 2. 先例：ArduDeck 已经证明这条路可行

外部证据（[ArduDeck 项目博客](https://ardudeck.com/blog/introducing-ardudeck-ground-control-station/)、[GitHub](https://github.com/rubenCodeforges/ardudeck)）：

**ArduDeck 是一个已发布的开源跨平台地面站，技术栈正是 Electron**：

| 层 | ArduDeck 的选择 |
|---|---|
| 外壳 | **Electron**（同一应用支持 mac / Windows / Linux） |
| UI | **React 18 + TypeScript + Vite**，Tailwind CSS（深色主题，无 CSS 文件） |
| 状态管理 | **Zustand**，带 selector 优化（避免遥测刷新导致整页重渲染） |
| 工程结构 | **pnpm monorepo**，协议库独立成包 |
| 协议层 | 自研 `mavlink-ts`（MAVLink v1/v2）、`msp-ts`（MSP v1/v2/jumbo）、`comms`（USB 串口 / TCP / UDP 传输抽象）、`stm32-dfu`（固件刷写） |
| 进程通信 | Electron 主进程 ↔ 渲染进程走**类型化 IPC 通道** |
| 许可 | **GPL-3.0** |

它已实现：参数管理（含 ArduPilot 元数据）、任务规划、10 Hz 实时遥测、MAVLink FTP（参数秒级加载）、签名、STM32 DFU 固件刷写、加速度计/罗盘校准向导、多机（Beta 1 "Fly the Fleet"）。

**这件事的意义**：这不是理论推演，而是一个**已落地的实证案例**——用 Electron + TS 从零实现了 MAVLink 地面站的核心能力（含参数系统与 FTP，正是本项目最关心的部分）。

### 2.1 可从先例复用的部分

- `mavlink-ts`（GPL-3.0）与 `comms`（串口/TCP/UDP 抽象）—— 你的项目同为 GPL-3.0 衍生，**许可上兼容，可直接借鉴或复用**（需保留署名与许可）。
- 工程形态参考：pnpm monorepo + 协议库分包 + 类型化 IPC。
- 状态管理经验：Zustand selector 优化——**这是无人机地面站的真实性能要点**（10 Hz 遥测下不能让整页重渲染）。

---

## 3. 两条实现路线

### 路线 A：混合 —— 保留 QGC C++ 逻辑层，只换 UI 层（推荐起步）

```
┌─────────────────────────────┐
│  Electron 前端（Chromium）    │  React/Vue + TS
│  IPC / WebSocket / HTTP      │
└──────────────┬──────────────┘
               │  本地回环（Qt6 HttpServer 或 WebSocket）
┌──────────────┴──────────────┐
│  QGC C++ 核心（headless）      │  保留全部成熟逻辑：
│  Fact System / ParameterMgr  │  MAVLink 栈、参数协议、任务协议、
│  MissionManager / Terrain    │  地形、多机管理、固件插件
└─────────────────────────────┘
```

| 维度 | 评估 |
|---|---|
| 保留 | MAVLink 协议栈、参数系统、任务规划、地形、多机、PX4/APM 双固件适配——**全部成熟逻辑** |
| 改造量 | ① 剥离 QML UI；② 把 1272 个 `Q_PROPERTY` + 469 个 `Q_INVOKABLE` 定义成通信协议；③ Electron 前端从零写 |
| 通信层成本 | **低于预期**：`Qt6 HttpServer` 已在依赖中，无需引入新框架 |
| 硬难点 | **图传**：GStreamer 管线如何送到 Chromium（WebRTC / MSE / HLS 三选一，延迟待实测） |
| 移动端 | Android/iOS 需要各自宿主（Electron 不支持移动端，需 Capacitor/Tauri Mobile 或保留 Qt） |
| 工作量 | 数月至半年级（单人），但**可逐页增量交付** |
| 风险 | 中：新增进程间协议是新的复杂度来源，需良好的类型化契约（可参考 ArduDeck 的类型化 IPC） |

**这条路线的最大价值：QGC 的协议成熟度是多年积累，不该扔。**

### 路线 B：全栈重写 —— 纯 TS/Node，彻底不要 C++ 编译

完全对标 ArduDeck 的做法：`mavlink-ts` + `serialport` + Electron + React，**不保留任何 QGC C++ 代码**。

| 维度 | 评估 |
|---|---|
| 优势 | 单一技术栈、开发体验统一、**彻底摆脱 Qt6 + MSVC 构建环境**（本机目前装不上的问题直接消失） |
| 优势 | 移动端仍不在 Electron 覆盖范围内 |
| 必须重写 | MAVLink 栈、参数协议（含 PX4 Component Information 与 APM `apm.pdef.json` 解析）、任务协议、地形/高程、固件刷写、日志分析 |
| 关键缺口 | **PX4 的元数据通路没有现成 TS 实现**：PX4 走 Component Information（`MAV_CMD_REQUEST_MESSAGE(512,param1=397)` → `COMPONENT_METADATA` → MAVFTP 取 `parameters.json.xz`），这条链路需自行实现（ArduDeck 主要面向 ArduPilot/iNav/Betaflight） |
| 参考价值 | 本项目的分册 `02_PX4参数链路.md`、`03_QGC参数链路.md`、`05_APM参数系统对照.md` **正好是重写所需的协议规格文档** |
| 工作量 | 年级（单人），但起点比想象中高（有先例可参照） |
| 风险 | 高：放弃 QGC 的成熟度，等于重新踩它踩过的坑 |

---

## 4. 换技术栈会失去什么（修正版）

对比上一版，**Electron 消掉了大部分障碍**：

| 能力 | 纯浏览器 | **Electron** | 说明 |
|---|---|---|---|
| 串口 / 蓝牙遥测 | ❌ 必须原生桥 | ✅ `serialport` | ArduDeck 已验证 |
| 摇杆输入 | ❌ 需原生桥 | ✅ Gamepad API | — |
| 离线地图与地形 | ⚠️ 受限 | ✅ 本地缓存 | MapLibre GL 可离线 |
| 固件刷写（DFU） | ❌ 不可行 | ✅ Node USB/DFU | ArduDeck 已验证 `stm32-dfu` |
| 日志文件读写 | ❌ 受限 | ✅ Node fs | — |
| **低延迟图传** | ⚠️ | ⚠️ **仍是最大未知项** | 需 WebRTC/MSE，延迟指标未实测 |
| 移动端（Android/iOS） | ⚠️ | ❌ Electron 不支持 | 需另选宿主，或移动端保留 Qt |
| 单一代码库跨 6 平台 | — | 桌面 3 平台 ✅ / 移动 ❌ | QGC 目前桌面+移动全覆盖 |

**唯一没有变好的仍是图传**：10 Hz 遥测没问题，但视频流从 GStreamer 迁到 Chromium 的延迟与稳定性必须实测。这是路线 A、B 共同的关键未知项。

---

## 5. 许可影响（决策相关）

| 组件 | 许可 | 影响 |
|---|---|---|
| QGC 本体 | **GPL-3.0**（`LICENSE-GPL` 35 KB）+ 部分 Apache-2.0（`LICENSE-APACHE` 11.3 KB） | 衍生作品整体按 GPL-3.0 分发 |
| Electron | **MIT** | **不传染**，不强制你的应用开源 |
| ArduDeck | **GPL-3.0** | 与 QGC 衍生项目许可兼容，可借鉴复用（保留署名） |

**要点**：无论选 Qt/QML 还是 Electron，**限制都来自 QGC 的 GPL-3.0，与前端技术栈无关**。换 Electron 不会让项目变得"更封闭"，也不会更开放。

---

## 6. 建议路线

### 阶段 0：验证通信层（**先做这个，1–2 周**）

不碰 UI，先证明"QGC C++ 核心 ↔ Electron"这条路能通：

1. QGC 侧启用 `Qt6 HttpServer`（已在依赖中），把 `Fact` 的 name/value/unit/enum 以 JSON 暴露；
2. Electron 侧拉取并渲染一个最简参数列表；
3. **实测三个指标**：参数全量加载耗时、单参数写入往返延迟、10 Hz 遥测下的渲染帧率。

这三个数字决定后续所有决策，成本远低于直接开发 UI。

### 阶段 1：参数页完整替换（2–4 周）

参数页是最佳试点：只依赖 `Fact` 的有限字段，不碰地图、视频、任务协议。做完就能拿到"Electron 前端替代 QML 页面"的真实成本。

同时把本项目已查实的 4 个 QGC 前端缺陷（写入无反馈、搜索不含枚举、排序下标误用、控件推断失效）**在新前端里直接做对**，而不是回去修 QML。

### 阶段 2：决策是否全量迁移

依据阶段 0/1 的实测数字选择：
- 通信清爽、体验达标 → 按路线 A 逐页迁移（地图 → 任务 → 遥测 → 设置）；
- 若图传延迟无法接受 → 图传保留原生管线，其余页面走 Electron（**混合架构**）；
- 路线 B（全栈重写）仅在"确定要长期投入且不接受 C++ 构建链"时考虑。

### 阶段边界：移动端

QGC 目前覆盖 Android/iOS。Electron **不支持移动端**。若移动端是必需项，需提前决定：
- 移动端保留 Qt/QML 前端（双前端维护成本）；
- 或移动端换 Capacitor/Tauri Mobile（需重新验证性能）；
- 或放弃移动端。

**这个决定必须在阶段 1 之前做**，因为它影响通信协议设计（是否需要跨平台抽象）。

---

## 7. 未验证与边界声明

1. 本机无 Qt6/MSVC，**未编译、未运行 QGC**；所有 UI 与性能类判断均为源码推断。
2. **图传延迟未实测**——这是 Electron 方案最大的未知项，必须实测后再承诺指标。
3. 通信层设计（1272 个 `Q_PROPERTY` + 469 个 `Q_INVOKABLE` 的接口化方案）**尚未设计**，阶段 0 的目的就是验证它。
4. ArduDeck 的能力与实现细节来自其**官方博客与仓库自述**（外部信息，未审计其源码），作为可行性先例引用，不作为本项目性能依据。
5. PX4 元数据通路的 TS 实现**目前无现成库**；本项目分册文档可作规格输入，但重写工作量未细化。
6. Electron 的打包体积（Chromium 内核，通常 100 MB+ 量级）与内存占用未评估，对地面站属于可接受范围但需确认。
7. 移动端支持策略未定（见 §6 阶段边界），这是路线选择的前置决策。
