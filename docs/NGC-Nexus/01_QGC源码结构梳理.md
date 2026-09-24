# QGroundControl 源码结构梳理（NGC-Nexus 基线）

## 0. 文档口径与证据说明

| 项 | 值 |
|---|---|
| 源码根目录 | `E:\04-Workspace_workbudy\15_NGC-Nexus\repo` |
| 上游 | mavlink/qgroundcontrol，master 分支 |
| HEAD | `25185047e855937d694a0174c1541d8f7d794a74`（2026-09-25） |
| 克隆方式 | 浅克隆，`.git\shallow` 内容为该 commit（实测存在） |
| 子模块 | 无 `.gitmodules`（实测 `Test-Path` 为 `False`） |
| 分析方式 | 纯源码阅读，未编译 |

**证据纪律**：本文所有论断均给「相对路径:行号」，路径相对 `repo\`。行号为 read/grep 工具返回的行号（1 基）。
行数统计一律用 PowerShell 5.1 的 `(Get-Content <file>).Count`；**未使用** `Measure-Object -Line`，因为该 cmdlet 不统计空行，会系统性少算（例如 `src/main.cc` 用 `.Count` 为 71 行，用 `-Line` 只有 56 行）。
凡未实际读取确认的内容，标注「未验证」。

**本机环境事实（影响可行性）**：本机无 Qt6 安装（`C:\Qt`、`C:\Qt6`、`D:\Qt`、`E:\Qt`、`C:\Program Files\Qt` 均实测不存在），`cl.exe` 不在 PATH。因此本文只做源码阅读级分析，不涉及编译产物行为。

---

## 1. 仓库总体规模与顶层目录

### 1.1 实测规模数字

| 指标 | 实测值 | 统计口径 |
|---|---|---|
| 全仓库文件数（排除 `.git`） | 5498 | `Get-ChildItem -Recurse -File -Force` |
| 顶层条目 | 15 个目录 + 39 个文件 | 顶层列举 |
| `src/` 文件数 | 2243 | 递归 |
| `src/` 下 C/C++ 源文件（.h/.hpp/.cc/.cpp） | 1303 | 递归 |
| `src/` 下 C/C++ 行数 | 226953 | 逐文件 `.Count` 求和 |
| `src/` 下 QML 文件数 | 478 | 递归 |
| `src/` 下 QML 行数 | 66772 | 逐文件 `.Count` 求和 |
| `src/` 一级子目录数 | 32 | 实测 |
| 翻译文件 `.ts` 数 | 48 | `translations\*.ts` |

### 1.2 顶层目录职责

| 目录 | 文件数 | 职责 |
|---|---|---|
| `src/` | 2243 | 全部应用源码（C++ + QML），见第 2 节 |
| `cmake/` | 110 | CMake 模块、平台配置、find-modules、打包脚本、安装规则 |
| `tools/` | 267 | Python 工具链：配置、构建、代码生成、静态分析、翻译、发布 |
| `test/` | 927 | 单元/集成测试，按 `src/` 模块镜像组织（26 个一级子目录） |
| `docs/` | 1012 | VitePress 用户文档站点，多语言（`en/`、`ko/`、`tr/`、`zh/`） |
| `resources/` | 417 | 图标、字体、音频、校准图片、地图瓦片兜底数据等运行时资源（QML 文件数为 0） |
| `deploy/` | 60 | Windows/macOS/Linux 打包与安装器素材 |
| `custom-example/` | 65 | 二次开发（定制构建）官方模板，见 4.4 节 |
| `android/` | 58 | Android 平台清单与资源 |
| `translations/` | 48 | Qt `.ts` 翻译源 |
| `.github/` | 242 | CI 工作流、`build-config.json`（构建配置唯一事实源）、贡献指南 |
| `.clusterfuzzlite/` | 5 | 持续模糊测试配置 |
| `.devcontainer/` | 1 | 开发容器定义 |
| `.vscode/` | 4 | 编辑器工作区配置 |

顶层构建入口文件：`CMakeLists.txt`（562 行）、`CMakePresets.json`（16 行）、`justfile`（159 行）、`package.json`、`.pre-commit-config.yaml`、`Doxyfile`、`crowdin.yml`。说明：`vcpkg.json` 未出现在顶层 39 个文件中（实测），本仓库是否使用 vcpkg 未验证。

---

## 2. `src/` 模块总览

### 2.1 模块规模实测表

`h` = `.h`/`.hpp`，`cc` = `.cc`/`.cpp`，`qml` = `.qml`，`其他` = 余量（json/svg/png/mm 等）。

| 模块 | 总文件 | h | cc | qml | 其他 | QML 行数 |
|---|---|---|---|---|---|---|
| ADSB | 8 | 4 | 3 | 0 | 1 | 0 |
| AnalyzeView | 71 | 22 | 20 | 17 | 12 | 4395 |
| Android | 21 | 10 | 9 | 0 | 2 | 0 |
| API | 7 | 3 | 3 | 0 | 1 | 0 |
| AppSettings | 46 | 0 | 0 | 31 | 15 | 6039 |
| AutoPilotPlugins | 306 | 55 | 55 | 84 | 112 | 11605 |
| Camera | 19 | 7 | 7 | 0 | 5 | 0 |
| Comms | 62 | 25 | 25 | 0 | 12 | 0 |
| FactSystem | 41 | 10 | 10 | 18 | 3 | 947 |
| FirmwarePlugin | 58 | 16 | 14 | 7 | 21 | 433 |
| FirstRunPromptDialogs | 3 | 0 | 0 | 2 | 1 | 135 |
| FlightMap | 72 | 0 | 0 | 31 | 41 | 5006 |
| FlyView | 66 | 0 | 0 | 65 | 1 | 6939 |
| FollowMe | 3 | 1 | 1 | 0 | 1 | 0 |
| GeoMap | 69 | 22 | 21 | 22 | 4 | 2725 |
| Gimbal | 6 | 2 | 2 | 0 | 2 | 0 |
| GPS | 219 | 118 | 74 | 3 | 24 | 299 |
| Joystick | 7 | 3 | 3 | 0 | 1 | 0 |
| LogManager | 12 | 5 | 5 | 1 | 1 | 206 |
| MainWindow | 2 | 0 | 0 | 2 | 0 | 890 |
| MAVLink | 37 | 19 | 14 | 0 | 4 | 0 |
| MissionManager | 101 | 39 | 37 | 0 | 25 | 0 |
| PlanView | 45 | 0 | 0 | 44 | 1 | 8283 |
| QmlControls | 173 | 34 | 33 | 100 | 6 | 9900 |
| QtLocationPlugin | 49 | 26 | 21 | 0 | 2 | 0 |
| Settings | 83 | 28 | 28 | 0 | 27 | 0 |
| Terrain | 14 | 7 | 6 | 0 | 1 | 0 |
| Toolbar | 76 | 0 | 0 | 30 | 46 | 5213 |
| Utilities | 232 | 106 | 94 | 0 | 32 | 0 |
| Vehicle | 165 | 64 | 63 | 12 | 26 | 2627 |
| VideoManager | 109 | 54 | 45 | 0 | 10 | 0 |
| Viewer3D | 55 | 14 | 12 | 9 | 20 | 1130 |

**纯 QML 模块（无 C++，h=cc=0）**：AppSettings、FlightMap、FlyView、MainWindow、PlanView、Toolbar、FirstRunPromptDialogs。这 7 个模块是页面级 UI 的全部载体，共 205 个 QML 文件、32505 行（按上表逐项相加）。

### 2.2 模块职责表

| 模块 | 职责 | 关键类（文件:行号） | 主要源文件（行数） |
|---|---|---|---|
| ADSB | 通过 TCP 接收周边民航/无人机广播态势并按车辆列表建模 | `ADSBTCPLink`（`src/ADSB/ADSBTCPLink.h:18`）、`ADSBVehicle`（`src/ADSB/ADSBVehicle.h:11`）、`ADSBVehicleManager`（`src/ADSB/ADSBVehicleManager.h:15`） | `src/ADSB/ADSBVehicleManager.cc`、`src/ADSB/ADSBVehicle.cc`、`src/ADSB/ADSBTCPLink.cc` |
| AnalyzeView | 「分析」页容器与各子页（GeoTag、日志查看器、MAVLink 控制台/检查器、机载日志） | `GeoTagController`（`src/AnalyzeView/GeoTag/GeoTagController.h:48`）、`LogViewerController`（`src/AnalyzeView/LogViewer/LogViewerController.h:11`）、`MAVLinkInspectorController`（`src/AnalyzeView/MAVLinkInspector/MAVLinkInspectorController.h:18`）、`OnboardLogController`（`src/AnalyzeView/OnboardLogs/OnboardLogController.h:18`） | `src/AnalyzeView/AnalyzePage.qml`、`src/AnalyzeView/AnalyzeView.qml` |
| Android | JNI 桥接、Android 串口、事件转发 | `AndroidInterface` 命名空间（`src/Android/AndroidInterface.h:9`）、`AndroidSerial` 命名空间（`src/Android/AndroidSerial.h:10`）、`AndroidEvents`（`src/Android/AndroidEvents.h:6`） | `src/Android/AndroidInterface.cc`、`src/Android/AndroidSerial.cc` |
| API | 应用级扩展点：核心插件、选项、分析页描述 | `QGCCorePlugin`（`src/API/QGCCorePlugin.h:37`）、`QGCOptions`（`src/API/QGCOptions.h:45`）、`QmlComponentInfo`（`src/API/QmlComponentInfo.h:8`） | `src/API/QGCCorePlugin.cc`（448）、`src/API/QGCCorePlugin.h`（252）、`src/API/QGCOptions.cc` |
| AppSettings | 设置界面（纯 QML）：13 个 `SettingsUI.json` 驱动的生成页 + 手写页 | 无 C++ 类 | `src/AppSettings/pages/SettingsPages.json`、`src/AppSettings/SettingsPage.qml`、`src/AppSettings/CMakeLists.txt` |
| AutoPilotPlugins | 机型设置（Vehicle Setup）UI 与控制器，分 Common/APM/PX4 三层 | `AutoPilotPlugin`（`src/AutoPilotPlugins/AutoPilotPlugin.h:18`）、`VehicleComponent`（`src/AutoPilotPlugins/VehicleComponent.h:18`）、`PX4AutoPilotPlugin`（`src/AutoPilotPlugins/PX4/PX4AutoPilotPlugin.h:21`）、`APMAutoPilotPlugin`（`src/AutoPilotPlugins/APM/APMAutoPilotPlugin.h:32`） | `src/AutoPilotPlugins/PX4/AirframeComponentController.cc`、`src/AutoPilotPlugins/APM/APMFlightModesComponentController.cc` |
| Camera | 相机/云台参数（FactGroup）与视频流信息抽象 | `QGCCameraManager`（`src/Camera/QGCCameraManager.h:26`）、`MavlinkCameraControlInterface`（`src/Camera/MavlinkCameraControlInterface.h:20`）、`QGCVideoStreamInfo`（`src/Camera/QGCVideoStreamInfo.h:11`）、`CameraMetaData`（`src/Camera/CameraMetaData.h:7`） | `src/Camera/QGCCameraManager.cc`、`src/Camera/MavlinkCameraControlInterface.cc` |
| Comms | 链路层：串口/UDP/TCP/蓝牙/日志回放/MockLink 与 MAVLink 协议收发 | `LinkManager`（`src/Comms/LinkManager.h:28`）、`LinkInterface`（`src/Comms/LinkInterface.h:18`）、`LinkConfiguration`（`src/Comms/LinkConfiguration.h:13`）、`MAVLinkProtocol`（`src/Comms/MAVLinkProtocol.h:17`）、`UDPLink`（`src/Comms/UDPLink.h:150`）、`TCPLink`（`src/Comms/TCPLink.h:91`）、`SerialLink`（`src/Comms/Serial/SerialLink.h:166`）、`BluetoothLink`（`src/Comms/Bluetooth/BluetoothLink.h:13`） | `src/Comms/MAVLinkProtocol.cc`、`src/Comms/LinkManager.cc`；子目录 `Serial/`(10)、`Bluetooth/`(11)、`MockLink/`(24) |
| FactSystem | 参数系统内核：单值 Fact、元数据、Fact 分组、参数管理 | `Fact`（`src/FactSystem/Fact.h:16`）、`FactMetaData`（`src/FactSystem/FactMetaData.h:16`）、`FactGroup`（`src/FactSystem/FactGroup.h:16`）、`SettingsFact`（`src/FactSystem/SettingsFact.h:11`）、`ParameterManager`（`src/FactSystem/ParameterManager.h:20`）、`FactPanelController`（`src/FactSystem/FactControls/FactPanelController.h:13`） | `src/FactSystem/ParameterManager.cc`（1947）、`src/FactSystem/Fact.cc`（945）、`src/FactSystem/FactMetaData.h`（471）、`src/FactSystem/Fact.cc` |
| FirmwarePlugin | 固件差异抽象：PX4/ArduPilot 行为、飞行模式表、参数元数据 | `FirmwarePlugin`（`src/FirmwarePlugin/FirmwarePlugin.h:72`）、`FirmwarePluginFactory`（`src/FirmwarePlugin/FirmwarePluginFactory.h:9`）、`FirmwarePluginManager`（`src/FirmwarePlugin/FirmwarePluginManager.h:12`）、`PX4FirmwarePlugin`（`src/FirmwarePlugin/PX4/PX4FirmwarePlugin.h:6`）、`APMFirmwarePlugin`（`src/FirmwarePlugin/APM/APMFirmwarePlugin.h:22`） | `src/FirmwarePlugin/FirmwarePlugin.cc`（664）、`src/FirmwarePlugin/FirmwarePlugin.h`（477）、`src/FirmwarePlugin/ParameterMetaData.h` |
| FirstRunPromptDialogs | 首次运行提示对话框（纯 QML） | 无 C++ 类 | `src/FirstRunPromptDialogs/FirstRunPrompt.qml`、`src/FirstRunPromptDialogs/InitialSetupPrompt.qml` |
| FlightMap | 通用地图控件库（纯 QML）：`FlightMap`、`MapFitFunctions` 等 | 无 C++ 类 | 31 个 QML，5006 行 |
| FlyView | 飞行视图（纯 QML）：HUD、仪表、工具栏、引导操作 | 无 C++ 类 | 65 个 QML，6939 行 |
| FollowMe | 跟随模式单例 | `FollowMe`（`src/FollowMe/FollowMe.h:9`） | `src/FollowMe/FollowMe.cc` |
| GeoMap | 3D 地理场景（Quick3D 几何、瓦片金字塔、相机） | `GeoMapCamera`（`src/GeoMap/GeoMapCamera.h:34`）、`FlightPathGeometry`（`src/GeoMap/FlightPathGeometry.h:41`）、`FenceWallGeometry`（`src/GeoMap/FenceWallGeometry.h:37`）、`ElevationTilePyramid`（`src/GeoMap/ElevationTilePyramid.h:35`） | `src/GeoMap/GeoMapCamera.cc`、`src/GeoMap/FlightPathGeometry.cc` |
| Gimbal | 云台 FactGroup 与控制器 | `Gimbal`（`src/Gimbal/Gimbal.h:8`）、`GimbalController`（`src/Gimbal/GimbalController.h:13`） | `src/Gimbal/Gimbal.cc`、`src/Gimbal/GimbalController.cc` |
| GPS | GNSS 栈：驱动、协议解码、RTK/RTCM 改正、NTRIP、定位源 | `GPSManager`（`src/GPS/GPSManager.h:14`）、`GPSCorrectionManager`（`src/GPS/Corrections/GPSCorrectionManager.h:19`）、`NTRIPManager`（`src/GPS/NTRIP/NTRIPManager.h:29`）、`GPSRtk`（`src/GPS/RTK/GPSRtk.h:23`）、`QGCPositionManager`（`src/GPS/Positioning/PositionManager.h:13`） | 8 个子目录：`Driver/`(91)、`Corrections/`(25)、`NTRIP/`(25)、`Positioning/`(25)、`Transport/`(14)、`Core/`(12)、`Receiver/`(12)、`RTK/`(10) |
| Joystick | SDL 手柄输入、按键动作映射 | `Joystick`（`src/Joystick/Joystick.h:62`）、`JoystickSDL`（`src/Joystick/JoystickSDL.h:18`）、`JoystickManager`（`src/Joystick/JoystickManager.h:12`） | `src/Joystick/Joystick.cc`、`src/Joystick/JoystickSDL.cc` |
| LogManager | 应用日志模型与表单化输出 | `LogManager`（`src/LogManager/LogManager.h:19`）、`LogModel`（`src/LogManager/LogModel.h:14`）、`LogEntryTableModel`（`src/LogManager/LogEntryTableModel.h:13`） | `src/LogManager/LogManager.cc`、`src/LogManager/LoggingCategoriesDialog.qml` |
| MainWindow | 顶层窗口 QML（纯 QML） | 无 C++ 类 | `src/MainWindow/MainWindow.qml`（800）、`src/MainWindow/MainWindowSavedState.qml`（90） |
| MAVLink | 协议辅助层：FTP、图片协议、流速率配置、状态文本、枚举绑定 | `QGCMAVLink`（`src/MAVLink/QGCMAVLink.h:15`，QML 单例）、`MAVLinkFTP`、`ImageProtocolManager`、`StatusTextHandler`、`SysStatusSensorInfo` | `src/MAVLink/MAVLinkFTP.cc`、`src/MAVLink/StatusTextHandler.cc`、`src/MAVLink/CMakeLists.txt`；子目录 `Signing/`、`LibEvents/` |
| MissionManager | 任务/测绘/地理围栏/返航点模型与控制器 | `MissionManager`（`src/MissionManager/MissionManager.h:9`）、`MissionController`（`src/MissionManager/MissionController.h:36`）、`PlanMasterController`（`src/MissionManager/PlanMasterController.h:20`）、`PlanManager`（`src/MissionManager/PlanManager.h:13`）、`MissionCommandTree`（`src/MissionManager/MissionCommandTree.h:38`） | `src/MissionManager/MissionController.cc`、`src/MissionManager/SimpleMissionItem.cc`、`MavCmdInfo*.json` |
| PlanView | 规划视图（纯 QML，44 个文件） | 无 C++ 类 | 44 个 QML，8283 行 |
| QmlControls | 全应用共享 QML 控件 + QML 全局单例 + 地图/仪表 C++ 类型 | `QGroundControlQmlGlobal`（`src/QmlControls/QGroundControlQmlGlobal.h:42`）、`ScreenToolsController`（`src/QmlControls/ScreenToolsController.h:8`）、`QGCPalette`（`src/QmlControls/QGCPalette.h:82`）、`FactValueGrid`（`src/QmlControls/FactValueGrid.h:13`）、`ParameterEditorController`（`src/QmlControls/ParameterEditorController.h:126`）、`QmlObjectListModel`（`src/QmlControls/QmlObjectListModel.h:5`） | 100 个 QML、9900 行；`QGroundControlQmlGlobal.cc`（401）、`FactValueGrid.cc`（363） |
| QtLocationPlugin | Qt Location 地图插件：瓦片缓存、引擎、URL 引擎 | `QGCMapEngine`（`src/QtLocationPlugin/QGCMapEngine.h:9`）、`QGeoTiledMapQGC`（`src/QtLocationPlugin/QGeoTiledMapQGC.h:7`）、`QGCMapEngineManager`（`src/QtLocationPlugin/QGCMapEngineManager.h:16`） | `src/QtLocationPlugin/QGCMapUrlEngine.cpp`、`src/QtLocationPlugin/QGeoFileTileCacheQGC.cpp`；子目录 `Providers/`(16) |
| Settings | 应用设置（非参数）系统：SettingsGroup + SettingsFact + JSON 元数据 | `SettingsManager`（`src/Settings/SettingsManager.h:38`）、`SettingsGroup`（`src/Settings/SettingsGroup.h:44`）、`AppSettings`（`src/Settings/AppSettings.h:9`） | `src/Settings/SettingsManager.cc`、`src/Settings/AppSettings.cc`（421）；30 个 `*.SettingsGroup.json` |
| Terrain | 地形高程查询（点/路径/区域） | `TerrainAtCoordinateBatchManager`（`src/Terrain/TerrainQuery.h:27`）、`TerrainPathQuery`（`src/Terrain/TerrainQuery.h:99`）、`TerrainAreaQuery`（`src/Terrain/TerrainQuery.h:129`） | `src/Terrain/TerrainQuery.cc`、`src/Terrain/TerrainTile.cc` |
| Toolbar | 顶部/侧边工具栏（纯 QML，30 个文件） | 无 C++ 类 | 30 个 QML，5213 行 |
| Utilities | 底层工具库集合（19 个子目录） | `Platform` 命名空间（`src/Utilities/Platform/Platform.h:11`）、`QGCCommandLineParser` 命名空间（`src/Utilities/QGCCommandLineParser.h:8`）、`QGCStateMachine`（`src/Utilities/StateMachine/QGCStateMachine.h:62`） | 子目录 `StateMachine/`(91)、`Geo/`(19)、`Compression/`(18)、`Parsing/`(17)、`Network/`(16)、`Platform/`(11)、`Timing/`(9)、`Logging/`(8)、`FileSystem/`(7) 等 |
| Vehicle | 载具核心模型、多机管理、连接状态机、参数与任务接线 | `Vehicle`（`src/Vehicle/Vehicle.h:86`）、`MultiVehicleManager`（`src/Vehicle/MultiVehicleManager.h:11`）、`VehicleLinkManager`（`src/Vehicle/VehicleLinkManager.h:14`） | `src/Vehicle/Vehicle.cc`（3573）、`src/Vehicle/Vehicle.h`（1327）；子目录 `FactGroups/`(62)、`VehicleSetup/`(27)、`ComponentInformation/`(19)、`Actuators/`(17) |
| VideoManager | 视频后端（GStreamer 等）、QML 视频项、字幕 | `VideoManager`（`src/VideoManager/VideoManager.h:23`）、`VideoReceiver`（`src/VideoManager/VideoReceiver/VideoReceiver.h:17`）、`SubtitleWriter`（`src/VideoManager/SubtitleWriter.h:11`） | `src/VideoManager/VideoManager.cc`；子目录 `VideoReceiver/`(104) |
| Viewer3D | 3D 视图管理与地图渲染数据提供 | `Viewer3DManager`（`src/Viewer3D/Viewer3DManager.h:11`）、`Viewer3DCameraController`（`src/Viewer3D/Viewer3DCameraController.h:28`）、`Viewer3DInstancing`（`src/Viewer3D/Viewer3DInstancing.h:11`） | `src/Viewer3D/Viewer3DManager.cc`；子目录 `Providers/Osm/`（OSM 解析） |

### 2.3 QML 模块 URI 注册表

实测 `qt_add_qml_module(... URI ...)` 共 **26 处**：`src/` 下 25 处，另有应用级模块 `QGC` 定义在顶层 `CMakeLists.txt:422-428`。下表逐项列出：

| URI | 定义位置 | 承载 target |
|---|---|---|
| `QGC` | `CMakeLists.txt:422-428` | 可执行目标（应用级 QML 模块） |
| `QGroundControl` | `src/CMakeLists.txt:15-25` | `QGroundControlModule`（静态库） |
| `QGroundControl.Controls` | `src/QmlControls/CMakeLists.txt:88-91` | `QGroundControlControlsModule` |
| `QGroundControl.FactControls` | `src/FactSystem/FactControls/CMakeLists.txt:11-14` | `FactControlsModule` |
| `QGroundControl.FlightMap` | `src/FlightMap/CMakeLists.txt:8-11` | `FlightMapModule` |
| `QGroundControl.FlyView` | `src/FlyView/CMakeLists.txt:8-11` | `FlyViewModule` |
| `QGroundControl.PlanView` | `src/PlanView/CMakeLists.txt:12-15` | `PlanViewModule` |
| `QGroundControl.Toolbar` | `src/Toolbar/CMakeLists.txt:3-7` | `ToolbarModule` |
| `QGroundControl.GeoMap` | `src/GeoMap/CMakeLists.txt:33-36` | `GeoMapModule` |
| `QGroundControl.AnalyzeView` | `src/AnalyzeView/CMakeLists.txt:19-22` | `AnalyzeViewModule` |
| `QGroundControl.AppSettings` | `src/AppSettings/CMakeLists.txt:112-116` | `AppSettingsModule` |
| `QGroundControl.LogManager` | `src/LogManager/CMakeLists.txt:14-17` | `QGCLogManager` |
| `QGroundControl.Viewer3D` | `src/Viewer3D/CMakeLists.txt:40-43` | `Viewer3DModule` |
| `QGroundControl.VehicleSetup` | `src/Vehicle/VehicleSetup/CMakeLists.txt:32-35` | `VehicleSetupModule` |
| `QGroundControl.FirstRunPromptDialogs` | `src/FirstRunPromptDialogs/CMakeLists.txt:3-6` | `FirstRunPromptDialogsModule` |
| `QGroundControl.AutoPilotPlugins.Common` | `src/AutoPilotPlugins/Common/CMakeLists.txt:35-38` | `AutoPilotPluginsCommonModule` |
| `QGroundControl.AutoPilotPlugins.APM` | `src/AutoPilotPlugins/APM/CMakeLists.txt:111-114` | `AutoPilotPluginsAPMModule` |
| `QGroundControl.AutoPilotPlugins.PX4` | `src/AutoPilotPlugins/PX4/CMakeLists.txt:96-99` | `AutoPilotPluginsPX4Module` |
| `QGroundControl.FirmwarePlugin.APM` | `src/FirmwarePlugin/APM/CMakeLists.txt:118-121` | `APMFirmwareModule` |
| `QGroundControl.FirmwarePlugin.PX4` | `src/FirmwarePlugin/PX4/CMakeLists.txt:33-36` | `PX4FirmwareModule` |
| `QGroundControl.AnalyzeView.GeoTag` | `src/AnalyzeView/GeoTag/CMakeLists.txt:28-31` | `GeoTagModule` |
| `QGroundControl.LogViewer` | `src/AnalyzeView/LogViewer/CMakeLists.txt:37-40` | `LogViewerModule` |
| `QGroundControl.GPS.NTRIP` | `src/GPS/NTRIP/CMakeLists.txt:78-82` | `GPSNTRIPModule` |
| `QGroundControl.Compression` | `src/Utilities/Compression/CMakeLists.txt:27-30` | `QGCCompression` |
| `QGroundControl.Logging` | `src/Utilities/Logging/CMakeLists.txt:13-16` | `QGCLogging` |
| `QGroundControl.Geo` | `src/Utilities/Geo/Formats/CMakeLists.txt:16-19` | `QGCGeoFormats` |

所有模块的 `RESOURCE_PREFIX` 均为 `/qml`（上表各 CMakeLists 同行下方一行），因此 URI 与资源路径一一对应。

---

## 3. 参数系统与前端 UI 重点模块

### 3.1 FactSystem（参数系统内核）

- `Fact` 是单值载体，继承 `QObject` 并标注 `QML_ELEMENT`（`src/FactSystem/Fact.h:16-19`），向 QML 暴露 45 个 `Q_PROPERTY`（`src/FactSystem/Fact.h:21-65`），其中与 UI 直接相关的是 `value`/`rawValue`/`valueString`/`enumStrings`/`enumValues`/`min`/`max`/`units`/`decimalPlaces`/`hasControl`/`readOnly`/`typeIsString`/`typeIsBool`。
- p 值类型体系定义在 `FactMetaData::ValueType_t`，共 14 个枚举值：uint8/int8/uint16/int16/uint32/int32/uint64/int64/float/double/string/bool/elapsedTimeInSeconds/custom（`src/FactSystem/FactMetaData.h:24-39`），并以 `Q_ENUM` 导出到 QML（`src/FactSystem/FactMetaData.h:40`）。`valueTypeCustom` 内部以 `QByteArray` 存储（`src/FactSystem/FactMetaData.h:38` 注释）。
- `FactMetaData` 同时是 QML 元素（`src/FactSystem/FactMetaData.h:16-19`），支持从 JSON 文件/数组批量构造（`src/FactSystem/FactMetaData.h:58-61`），并携带枚举字符串/值、单位类型（`src/FactSystem/FactMetaData.h:319-331`）与 JSON 键名常量（`src/FactSystem/FactMetaData.h:402-411`）。`src/FactSystem/factmetadata.schema.json` 是该 JSON 的 schema。
- `FactGroup` 把 Fact 组织为对象层级，`QML_UNCREATABLE`（`src/FactSystem/FactGroup.h:16-20`），提供 `getFact`/`getFactGroup`/`setLiveUpdates` 等 `Q_INVOKABLE` 接口（`src/FactSystem/FactGroup.h:29-43`）。
- `SettingsFact` 是 `Fact` 的子类，值落在 `QSettings`，并额外暴露 `userVisible`（`src/FactSystem/SettingsFact.h:11-20`）。这是「设置」与「载具参数」两条数据路径在 Fact 层的汇合点。
- `ParameterManager`（`src/FactSystem/ParameterManager.h:20`）负责载具参数（MAVLink `PARAM_*`）的读写与缓存，`QML_UNCREATABLE`（`src/FactSystem/ParameterManager.h:24`）；`ParameterManager.cc` 达 1947 行，是 FactSystem 中最大的实现文件。
- 控件层与控制器层单独成模块：`src/FactSystem/FactControls/`（21 个文件）注册 `QGroundControl.FactControls`（`src/FactSystem/FactControls/CMakeLists.txt:9-14`），包含 `Fact*`/`LabelledFact*` QML 控件与 `FactPanelController`（`src/FactSystem/FactControls/FactPanelController.h:13`）。
- **实测不一致点**：`FactControls/` 目录磁盘上有 18 个 `.qml`，但 `CMakeLists.txt` 的 `QML_FILES` 只登记 17 个——`FactTextFieldRow.qml`（19 行）在磁盘上存在，却未出现在 `QML_FILES` 中（对 `CMakeLists.txt` 检索该文件名命中 0 次，检索 `FactTextFieldGrid` 命中 1 次）。该文件因此不会进入 QML 模块资源；原因未验证。
- QML 侧消费示例：`src/QmlControls/ParameterEditorController.h:126`（参数编辑器控制器）、`src/QmlControls/FactValueGrid.h:13`（Fact 值网格）。

**结论（面向二次开发）**：新增/改造参数展示，入口是 `Fact` 的 `Q_PROPERTY` 与 `QGroundControl.FactControls` 里的现成控件；新增参数元数据则走 `FactMetaData` 的 JSON schema 路径。

### 3.2 Settings（应用设置系统，注意与 AppSettings 区分）

**关键区分**：C++ 的 `AppSettings` 类位于 `src/Settings/`（`src/Settings/AppSettings.h:9`），而 `src/AppSettings/` 是**纯 QML 设置界面模块**（0 个 .h、0 个 .cc）。二者名字相同、职责完全不同。

- `SettingsManager` 继承 `QQmlPropertyMap`，`QML_ELEMENT` + `QML_UNCREATABLE`（`src/Settings/SettingsManager.h:38-42`），以 24 个 `Q_PROPERTY`（`src/Settings/SettingsManager.h:68-92`）向 QML 暴露 24 个设置组，例如 `appSettings`、`mavlinkSettings`、`flyViewSettings`、`unitsSettings`、`videoSettings`。
- 设置组的统一基类 `SettingsGroup`（`src/Settings/SettingsGroup.h:44`）用宏 `DECLARE_SETTINGGROUP` / `DEFINE_SETTINGFACT` / `DECLARE_SETTINGSFACT`（`src/Settings/SettingsGroup.h:7-37`）批量生成 Fact 访问器；JSON 模板路径硬编码为 `:/json/%1.SettingsGroup.json`（`src/Settings/SettingsGroup.h:75`）。
- 这些 JSON 由 `src/Settings/CMakeLists.txt:72` 的 `qgc_add_json_resources(json_app_settings)` 打进 Qt 资源；该函数定义在 `cmake/Helpers.cmake:523`，默认 `PREFIX` 为 `/json`（`cmake/Helpers.cmake:538`）、默认 `PATTERN` 为 `*.json`（`cmake/Helpers.cmake:541`）。
- 全 `src/` 共 30 个 `*.SettingsGroup.json`，每个对应一个 `*Settings.h/.cc` 对。
- 自定义构建扩展点：`SettingsManager::adjustSettingMetaData`（`src/Settings/SettingsManager.h:105`）允许在 Fact 创建前覆写元数据；`registerCustomSettingsGroup`（`src/Settings/SettingsManager.h:113`）允许注册自定义设置组，注释明确要求访问器名必须是「JSON 文件名主干 camelCase + Settings」。

### 3.3 QmlControls（共享控件与 QML 全局单例）

- 模块 target `QGroundControlControlsModule`，URI `QGroundControl.Controls`（`src/QmlControls/CMakeLists.txt:82-91`）。目录 173 个文件，其中 100 个 QML、9900 行，是 UI 复用层的主体。
- 全局单例 `QGroundControlQmlGlobal`：`QML_NAMED_ELEMENT(QGroundControl)` + `QML_SINGLETON`（`src/QmlControls/QGroundControlQmlGlobal.h:45-46`），在 QML 中以 `QGroundControl.` 前缀访问。它聚合了 `linkManager`、`multiVehicleManager`、`settingsManager`、`videoManager`、`corePlugin`、`gpsManager`、`ntripManager` 等（`src/QmlControls/QGroundControlQmlGlobal.h:64-103`），是 QML 侧访问 C++ 单例的主通道。
- 其它 QML 单例：`ScreenToolsController`（`src/QmlControls/ScreenToolsController.h:8`，`QML_SINGLETON` 在第 12 行）、`QGCFileDialogController`（`src/QmlControls/QGCFileDialogController.h:11`）、`LogManager`（`src/LogManager/LogManager.h:19`，第 23 行 `QML_SINGLETON`）、`QGCLoggingCategoryManager`（`src/Utilities/Logging/QGCLoggingCategoryManager.h:20`）、`ShapeFileHelper`（`src/Utilities/Geo/Formats/ShapeFileHelper.h:14`）、`GeoTagController`（`src/AnalyzeView/GeoTag/GeoTagController.h:48`，第 52 行）、`OnboardLogController`（`src/AnalyzeView/OnboardLogs/OnboardLogController.h:18`，第 22 行）、`Viewer3DManager`（`src/Viewer3D/Viewer3DManager.h:11`，第 14-15 行 `QML_NAMED_ELEMENT(QGCViewer3DManager)`+`QML_SINGLETON`）。
- C++ 侧地图/仪表类型：`QGCPalette`（`src/QmlControls/QGCPalette.h:82`）、`QGCMapPolygon`/`QGCMapCircle`/`QGCMapPolyline`、`FactValueGrid`（`src/QmlControls/FactValueGrid.h:13`）、`QmlObjectListModel`（`src/QmlControls/QmlObjectListModel.h:5`）、`TerrainProfile`、`FlightPathSegment`。

### 3.4 AppSettings（设置界面层，纯 QML + 代码生成）

- `AppSettingsModule`（`src/AppSettings/CMakeLists.txt:107`）的 QML 文件分为两类：
  1. **生成页**（`src/AppSettings/CMakeLists.txt:61-76` 列出的 14 个名字，含 `SettingsPagesModel.qml`），由 `qgc_add_qml_codegen(GenerateSettingsQml ... GENERATE_AT_CONFIGURE ...)` 在 configure 阶段执行 `tools.generators.settings_qml.generate_pages` 生成（`src/AppSettings/CMakeLists.txt:84-100`）。生成器实现见 `cmake/modules/QGCQmlCodegen.cmake:3`。
  2. **手写页**（`src/AppSettings/CMakeLists.txt:121-151` 列的 30 个），如 `BluetoothSettings.qml`、`NtripConnectionSettings.qml`、`OfflineMapSettings.qml`、`PX4LogUploadSettings.qml`。
- 生成输入是 `src/AppSettings/pages/` 下 13 个 `*.SettingsUI.json` 加 `SettingsPages.json`（实测该目录 14 个文件），元数据输入是 `src/Settings/*.SettingsGroup.json`（`src/AppSettings/CMakeLists.txt:22-23`）。
- 自定义构建可覆盖页面清单：`--custom-pages-dir` / `--custom-settings-dir`（`src/AppSettings/CMakeLists.txt:28-39`）。

**结论**：设置页「加一项」的最小改动路径是改 `SettingsGroup.json` + `SettingsUI.json`（不改 C++、不改 QML），这是 QGC 前端可配置度最高的一层。

### 3.5 FirmwarePlugin（固件差异抽象）

- 基类注释明确：这是「唯一应该放飞控栈特定代码的地方」（`src/FirmwarePlugin/FirmwarePlugin.h:66-70`），其余源码对 MAVLink 通用实现保持中立。
- 飞行模式表结构 `FirmwareFlightMode`（`src/FirmwarePlugin/FirmwarePlugin.h:26-59`），含 `standard_mode`、`custom_mode`、`canBeSet`、`advanced`、`fixedWing`、`multiRotor` 字段，并提供 `FlightModeList`/`FlightModeCustomModeMap` 别名（`src/FirmwarePlugin/FirmwarePlugin.h:63-64`）。
- 工厂与管理：`FirmwarePluginFactory`（`src/FirmwarePlugin/FirmwarePluginFactory.h:9`）按固件类型产出插件，`FirmwarePluginManager`（`src/FirmwarePlugin/FirmwarePluginManager.h:12`）对上层提供查询。
- 实现分目录：`src/FirmwarePlugin/APM/`（30 个文件）与 `src/FirmwarePlugin/PX4/`（19 个文件），分别注册 QML 模块 `QGroundControl.FirmwarePlugin.APM` / `.PX4`（`src/FirmwarePlugin/APM/CMakeLists.txt:118-121`、`src/FirmwarePlugin/PX4/CMakeLists.txt:33-36`）。
- `ParameterMetaData`（`src/FirmwarePlugin/ParameterMetaData.h`）承担「固件参数元数据」的补充来源，与 `FactMetaData` 分工：前者描述固件参数集合，后者描述单个 Fact。

### 3.6 AutoPilotPlugins（机型设置 UI，三层结构）

| 层 | 目录 | 文件数 | 模块 URI |
|---|---|---|---|
| 通用基类 | `src/AutoPilotPlugins/`（顶层） | 顶层 `AutoPilotPlugin.h:18`、`VehicleComponent.h:18` | — |
| Common | `src/AutoPilotPlugins/Common/` | 77 | `QGroundControl.AutoPilotPlugins.Common`（`Common/CMakeLists.txt:35-38`） |
| APM | `src/AutoPilotPlugins/APM/` | 110 | `QGroundControl.AutoPilotPlugins.APM`（`APM/CMakeLists.txt:111-114`） |
| PX4 | `src/AutoPilotPlugins/PX4/` | 112 | `QGroundControl.AutoPilotPlugins.PX4`（`PX4/CMakeLists.txt:96-99`） |
| Generic | `src/AutoPilotPlugins/Generic/` | 2 | 未验证是否注册独立 QML 模块 |

- 每个设置页由「Component（模型/入口）」+「Controller（Fact 面板控制器）」成对实现，例如 `AirframeComponent.h:5` / `AirframeComponentController.h:12`（`src/AutoPilotPlugins/PX4/`）、`APMFlightModesComponent.h:5` / `APMFlightModesComponentController.h:12`（`src/AutoPilotPlugins/APM/`）。控制器统一继承 `FactPanelController`（`src/AutoPilotPlugins/Common/SyslinkComponentController.h:11` 等多处可证）。
- 机型设置页面本体归属 `src/Vehicle/VehicleSetup/`（27 个文件、12 个 QML），URI `QGroundControl.VehicleSetup`（`src/Vehicle/VehicleSetup/CMakeLists.txt:32-35`），包含 `VehicleConfigView.qml`、`VehicleSummary.qml`、`RemoteControlCalibration.qml`、`FirmwareUpgrade.qml` 等。

### 3.7 MAVLink（协议层）

- `src/MAVLink/` 仅 37 个文件，且**不含** `MAVLinkProtocol`——该类实际位于 `src/Comms/MAVLinkProtocol.h:17`。`src/MAVLink/` 承担协议辅助职责：`MAVLinkFTP`、`ImageProtocolManager`、`MAVLinkStreamConfig`、`StatusTextHandler`、`SysStatusSensorInfo`、`QGCMAVLink`、`MAVLinkMessageType.h`（文件清单见 `src/MAVLink/CMakeLists.txt:6-23`）。
- `QGCMAVLink` 以 `QML_NAMED_ELEMENT(MAVLink)` + `QML_SINGLETON` 暴露给 QML（`src/MAVLink/QGCMAVLink.h:15-16`），是 QML 侧读取 MAVLink 枚举/常量的入口。
- 上游库通过 CPM 拉取：`CPMAddPackage(NAME mavlink GIT_REPOSITORY ${QGC_MAVLINK_GIT_REPO} GIT_TAG ${QGC_MAVLINK_GIT_TAG} ...)`（`src/MAVLink/CMakeLists.txt:46-55`），仓库地址与 commit 在 `cmake/CustomOptions.cmake:165-172`：`https://github.com/mavlink/mavlink.git` @ `c409cf690454db6d3e004bd14173bc6c7ff1e0ff`，dialect 为 `all`、协议版本 2.0（`cmake/CustomOptions.cmake:173-180`）。
- 生成的 mavlink 头文件位于**构建树**而非源码树：`${mavlink_BINARY_DIR}/include/mavlink`（`src/MAVLink/CMakeLists.txt:60-62`）。
- 另有自研代码生成：`tools/generators/mavlink_enums.py` 产出 `MAVLinkEnums.h`、`MAVLinkEnumsQml.h/.cc`（`src/MAVLink/CMakeLists.txt:66-83`），后者用 `Q_NAMESPACE`/`Q_ENUM_NS` 把全部 MAVLink 枚举暴露给 QML。

### 3.8 Vehicle（载具模型）

- `Vehicle` 同时继承 `VehicleFactGroup` 与 `VehicleTypes`（`src/Vehicle/Vehicle.h:86`），`QML_ELEMENT` + `QML_UNCREATABLE`（`src/Vehicle/Vehicle.h:89-90`）；`Vehicle.cc` 3573 行、`Vehicle.h` 1327 行，是单文件体量最大的模块。
- 聚合了 30 余个子系统前向声明（`src/Vehicle/Vehicle.h:25-84`），包括 `ParameterManager`、`MissionManager`、`Autotune`、`FirmwarePlugin`、`QGCCameraManager`、`GimbalController`、`RemoteIDManager` 等——即「载具对象是各子系统的容器」。
- `MultiVehicleManager`（`src/Vehicle/MultiVehicleManager.h:11`）多机管理；`VehicleLinkManager`（`src/Vehicle/VehicleLinkManager.h:14`）链路状态。二者在启动序列中被显式初始化（见 5.2 节）。
- Fact 分组集中在 `src/Vehicle/FactGroups/`（62 个文件），如 `VehicleGPSFactGroup`、`BatteryFactGroupListModel`、`EscStatusFactGroupListModel`。

---

## 4. 构建与依赖解析

### 4.1 构建入口与版本要求

| 项 | 值 | 证据 |
|---|---|---|
| CMake 最低版本 | 3.25 | `CMakeLists.txt:5`；`.github/build-config.json:24` |
| Qt 最低版本 | 6.11.0 | `.github/build-config.json:5` |
| Qt 最高（锁定）版本 | 6.11.1 | `.github/build-config.json:4` |
| CMake preset 格式版本 | 6 | `CMakePresets.json:2` |
| 默认构建类型 | Release | `CMakeLists.txt:26-32` |
| GStreamer 默认版本 | 1.28.4（最低 1.20.0） | `.github/build-config.json:29-30` |
| 构建配置事实源 | `.github/build-config.json` | `cmake/BuildConfig.cmake:7`，缺失时直接 `FATAL_ERROR`（`cmake/BuildConfig.cmake:9-11`） |

`CMakeLists.txt` 第 60 行注释明确「build-config.json is the source of truth」。该 JSON 由 `cmake/BuildConfig.cmake:17-39` 的 `qgc_config_get_value` 用 CMake 原生 `string(JSON ...)` 解析成缓存变量（`cmake/BuildConfig.cmake:41-56`）。

### 4.2 依赖解析方式（三条路径）

**路径 A：系统 Qt（外部安装）**

- `find_package(Qt6 ${QGC_QT_MINIMUM_VERSION} QUIET COMPONENTS Core)`，失败即 `FATAL_ERROR` 并提示设置 `CMAKE_PREFIX_PATH`（`CMakeLists.txt:241-250`）。
- 必需组件 24 个：Graphs/Concurrent/Core/Gui/HttpServer/LinguistTools/Location/LocationPrivate/Multimedia/Network/Positioning/Qml/QmlIntegration/Quick/QuickControls2/QuickVectorImage/QuickWidgets/Sensors/Sql/Svg/TextToSpeech/Xml/Quick3D/StateMachine（`CMakeLists.txt:252-279`）；可选组件 6 个：Bluetooth/MultimediaQuickPrivate/OpenGL/QuickTest/SerialPort/Test（`CMakeLists.txt:278`）；Linux 额外要 WaylandClient（`CMakeLists.txt:281-283`）。
- Windows 下 Qt 通过 toolchain 文件注入：`"toolchainFile": "$penv{QT_ROOT_DIR}/lib/cmake/Qt6/qt.toolchain.cmake"`（`cmake/presets/Windows.json:10`），即**依赖环境变量 `QT_ROOT_DIR`**。
- 交叉构建需要 `QT_HOST_PATH`（`cmake/presets/Windows.json:35,44`；`cmake/presets/Linux.json:47,56`）。
- Android 用 `CMAKE_PREFIX_PATH: "$penv{QT_TARGET_ROOT_DIR}"`（`cmake/presets/Android.json:14`）。

**路径 B：CPM（CMake Package Manager）按需下载第三方源码**

- CPM 被 `include(CPM)` 引入（`CMakeLists.txt:163`），模块实现位于 `cmake/modules/CPM.cmake`。
- CPM 源码缓存默认落在源码树内的 `.cache/CPM`：`set(CPM_SOURCE_CACHE "${CMAKE_SOURCE_DIR}/.cache/CPM" CACHE PATH ...)`（`CMakeLists.txt:150-162`），可被环境变量 `CPM_SOURCE_CACHE` 覆盖。
- **实测 `.cache` 目录不存在**（`Test-Path` 为 `False`），因为本仓库从未 configure 过——这直接解释了「树里看不到第三方依赖源码」。
- **实测**：全仓库（排除 `.git`）`CPMAddPackage(` 文本命中 **36 处**，其中 5 处位于 `cmake/modules/CPM.cmake` 自身（第 373、381、775、794、1016 行，属 CPM 实现内部调用而非项目使用），项目实际使用点 **31 处**；这 31 处里有 4 处在 `test/` 下（`test/GPS/Driver/Protocols/Comparisons/` 的对比测试），另有 1 处不在 `CMakeLists.txt` 中：`src/Utilities/Geo/GeographicLib.cmake:22`。

实际拉取的第三方依赖（按模块）：

| 依赖 | 仓库 | 调用点 |
|---|---|---|
| mavlink | `mavlink/mavlink` @ `c409cf69…` | `src/MAVLink/CMakeLists.txt:46` |
| libevents | `mavlink/libevents` | `src/MAVLink/LibEvents/CMakeLists.txt:24` |
| ArduPilot ParameterRepository | `ArduPilot/ParameterRepository` | `src/FirmwarePlugin/APM/CMakeLists.txt:41` |
| zlib / xz / zstd / libarchive | `madler/zlib`、`tukaani-project/xz`、`facebook/zstd`、`libarchive/libarchive` | `src/Utilities/Compression/CMakeLists.txt:48,79,134,278` |
| lz4 / bzip2（可选，默认 OFF） | `lz4/lz4`、`gitlab.com/bzip2/bzip2` | `src/Utilities/Compression/CMakeLists.txt:191,229` |
| SDL / SDL_GameControllerDB | `libsdl-org/SDL`、`mdqinc/SDL_GameControllerDB` | `src/Utilities/SDL/CMakeLists.txt:46,131` |
| libexif / ulog_cpp / valijson | `libexif/libexif`、`PX4/ulog_cpp`、`tristanpenman/valijson` | `src/Utilities/Parsing/CMakeLists.txt:29,114,126` |
| shapelib | `OSGeo/shapelib` | `src/Utilities/Geo/Formats/CMakeLists.txt:28` |
| GeographicLib r2.7 | `geographiclib/geographiclib` | `src/Utilities/Geo/GeographicLib.cmake:22` |
| libexpat / protozero / libosmium / earcut.hpp | `libexpat/libexpat`、`mapbox/protozero`、`osmcode/libosmium`、`mapbox/earcut.hpp` | `src/Viewer3D/Providers/Osm/CMakeLists.txt:39,70,81,117` |
| qtandroidextensions / QtAndroidTools | `2gis/qtandroidextensions`、`FalsinSoft/QtAndroidTools` | `src/Android/CMakeLists.txt:36,101` |
| Vulkan-Headers（仅 Windows 且未装 Vulkan SDK 时） | `KhronosGroup/Vulkan-Headers` @ `vulkan-sdk-1.4.341.0` | `CMakeLists.txt:215-233` |
| create-dmg（仅 macOS 打包） | `create-dmg/create-dmg` @ v1.2.3 | `cmake/install/Install.cmake:262-267` |

mavlink 与 GeographicLib 使用 `CPMPatchCache` 生成自定义缓存键，因为「CPM 默认缓存键不对 PATCHES 文件内容做哈希」（`src/MAVLink/CMakeLists.txt:36-44,50-51`；`src/Utilities/Geo/GeographicLib.cmake:12-27`）。

**路径 C：GStreamer SDK**

- GStreamer 原生库走平台分支：Linux 用系统 pkg-config（`cmake/find-modules/FindGStreamer.cmake`）；Android/macOS/iOS/Windows 下载 SDK，`cmake/GStreamer/platform/Android.cmake:42` 是下载调用点之一。
- `cmake/CustomOptions.cmake:111-122` 明确「Fail closed：只有走 SDK 下载的平台会命中该路径；Linux 用系统 pkg-config，从不下载」。
- 校验和与插件清单在 `.github/build-config.json:36-94`（含各平台 SHA256 与 CA bundle）。

### 4.3 为什么源码树里看不到 `libs/`

综合上述三条路径，答案是**这个版本的上游已经不使用 vendored 依赖目录**：

1. 无 `.gitmodules`（实测 `Test-Path` 为 `False`），依赖不通过子模块引入。`justfile:37-38` 虽定义了 `submodules` 配方（`git submodule update --init --recursive`），但在无 `.gitmodules` 的仓库上不产生实际效果；`cmake/CustomOptions.cmake:83` 的 `option(GIT_SUBMODULE ... OFF)` 默认也是关。
2. 第三方 C 库全部由 CPM 在 configure 阶段下载到 `.cache/CPM`（`CMakeLists.txt:150-162`），该目录在源码树内但**尚未生成**（实测不存在），且通常被 `.gitignore` 排除。
3. Qt6 本身是外部安装，通过 `CMAKE_PREFIX_PATH` / `QT_ROOT_DIR` 定位（`CMakeLists.txt:241-250`、`cmake/presets/Windows.json:10`）。
4. mavlink 生成头文件落在构建树 `${mavlink_BINARY_DIR}/include/mavlink`（`src/MAVLink/CMakeLists.txt:60-62`）。

**对本项目的直接含义**：任何离线环境下的源码级分析都不需要、也无法通过 `libs/` 找到依赖；同时，正是因为依赖与 Qt 均缺失，本机在不安装 Qt 6.11 与 MSVC 的前提下无法编译（本机实测无 Qt 安装、无 `cl.exe`）。

### 4.4 二次开发的构建扩展点

- 自定义构建目录变量 `QGC_CUSTOM_DIR` 默认值为 `"custom"`（`cmake/CustomOptions.cmake:11-14`）。
- `CMakeLists.txt:47-54` 强制该值必须是相对路径、非空、不含 `..`；`CMakeLists.txt:65-76` 检查该目录**必须同时包含** `CMakeLists.txt` 与 `cmake/CustomOverrides.cmake`，否则 `FATAL_ERROR`；满足后才置 `QGC_CUSTOM_BUILD=ON` 并把其 `cmake/` 加入 `CMAKE_MODULE_PATH`。
- 自定义 overlay 可追加：源文件 `CUSTOM_SOURCES`、库 `CUSTOM_LIBRARIES`、包含目录 `CUSTOM_INCLUDE_DIRECTORIES`、宏 `CUSTOM_DEFINITIONS`、Qt 组件 `CUSTOM_QT_COMPONENTS`（`src/CMakeLists.txt:202-224`）。
- 官方参考模板 `custom-example/`（65 个文件）结构：`custom-example/CMakeLists.txt`、`custom-example/cmake/CustomOverrides.cmake`、`custom-example/src/CustomPlugin.cc/.h`、`src/FirmwarePlugin/CustomFirmwarePlugin*`、`src/AutoPilotPlugin/CustomAutoPilotPlugin*`、`src/Settings/Custom.SettingsGroup.json` + `CustomSettings.*`、`src/AppSettings/pages/Custom.SettingsUI.json` + `SettingsPages.json`、`res/Custom/Widgets/Custom*.qml`、`res/json/PerimeterScan.SettingsGroup.json`。
- 面向「前端可用性改造」的可插入点（均有明确 hook）：`QGCCorePlugin::createRootWindow`（`src/API/QGCCorePlugin.h:120`，可换根窗口）、`::createQmlApplicationEngine`（`src/API/QGCCorePlugin.h:112`，可加 import path 与 context property）、`::paletteOverride`（`src/API/QGCCorePlugin.h:106`，可改配色）、`::factValueGridCreateDefaultSettings`（`src/API/QGCCorePlugin.h:108`）、`::toolBarIndicators`（`src/API/QGCCorePlugin.h:215`）、`::analyzePages`（`src/API/QGCCorePlugin.h:66`）、`::firstRunPromptStdIds`/`firstRunPromptCustomIds`（`src/API/QGCCorePlugin.h:203,208`）、`::overrideSettingsGroupVisibility`（`src/API/QGCCorePlugin.h:79`）。

---

## 5. 运行时骨架

### 5.1 `main()` 启动顺序（`src/main.cc`，共 71 行）

| 步骤 | 调用 | 行号 |
|---|---|---|
| 1 | `QGCCommandLineParser::parse(argc, argv)` | `src/main.cc:16` |
| 2 | `QGCCommandLineParser::handleParseResult(args)`，非空则直接返回退出码 | `src/main.cc:17-19` |
| 3 | `Platform::initialize(argc, argv, args)`，非空则返回 | `src/main.cc:22-24` |
| 4 | 构造 `QGCApplication app(argc, argv, args)` | `src/main.cc:26` |
| 5 | `LogManager::installHandler(args.logOutput)` | `src/main.cc:28` |
| 6 | `Platform::setupPostApp()` | `src/main.cc:30` |
| 7 | `app.init()` | `src/main.cc:32` |
| 8 | `LogManager::applyEnvironmentLogLevel()`（注释说明必须在 `app.init()` 内 `installFilter()` 之后） | `src/main.cc:34-35` |
| 9 | `QGCCommandLineParser::determineAppMode(args)` 分支：`ListTests`/`Test`/`BootTest`/`Gui` | `src/main.cc:40-56` |
| 10 | `Gui` 模式进入 `app.exec()` | `src/main.cc:53-55` |
| 11 | `app.shutdown()` | `src/main.cc:63` |
| 12 | `delete LogManager::instance()`（注释：在 Qt 仍完整可用时销毁，避免静态析构期问题） | `src/main.cc:67-68` |

相关支撑文件：`Platform` 命名空间声明于 `src/Utilities/Platform/Platform.h:11`，`Platform::initialize` 定义于 `src/Utilities/Platform/Platform.cc:157`，`Platform::setupPostApp` 定义于 `src/Utilities/Platform/Platform.cc:238`。命令行解析器声明于 `src/Utilities/QGCCommandLineParser.h:8`（`CommandLineParseResult` 结构体在 `src/Utilities/QGCCommandLineParser.h:15`，`enum class AppMode` 在 `src/Utilities/QGCCommandLineParser.h:78`）。

### 5.2 `QGCApplication` 构造与 `init()`

**构造函数**（`src/QGCApplication.cc:53-173`，共 818 行的文件）：

1. 初始化网络代理支持（`src/QGCApplication.cc:64`）。
2. 设置缺失 Fact 提示定时器（`src/QGCApplication.cc:71-73`，超时常量 `_missingParamsDelayedDisplayTimerTimeout = 1000`，`src/QGCApplication.h:149`）。
3. 决定 `applicationName`：单元测试/启动自检时加 `_unittest_` 后缀以隔离 `QSettings` 空间（`src/QGCApplication.cc:76-95`）；daily build 加 " Daily" 后缀（`src/QGCApplication.cc:88-91`）。
4. `QSettings::setDefaultFormat(QSettings::IniFormat)`，并打印设置文件路径与可写性（`src/QGCApplication.cc:103-110`）。
5. 设置版本管理：`settings.value(_settingsVersionKey) != QGC_SETTINGS_VERSION` 时清空设置并置 `_settingsUpgraded`（`src/QGCApplication.cc:131-138`）；`QGC_SETTINGS_VERSION` 当前值为 `"9"`（`cmake/CustomOptions.cmake:47-50`）。
6. 日志过滤器初始化（`src/QGCApplication.cc:155-156`）。
7. `setLanguage()`（`src/QGCApplication.cc:165`，实现见 175-225 行；含韩文字体加载 188-196、`qgc_source_` 与 `qgc_json_` 翻译加载 208-217、已有引擎则 `retranslate()` 220-222）。
8. 非 daily build 时检查新版本（`src/QGCApplication.cc:170-172`）。

**`init()`**（`src/QGCApplication.cc:229-258`）：

| 顺序 | 动作 | 行号 |
|---|---|---|
| 1 | `SettingsManager::instance()->init()` | `src/QGCApplication.cc:231` |
| 2 | 命令行指定 system ID 时写入 `mavlinkSettings()->gcsMavlinkSystemID()` | `src/QGCApplication.cc:232-235` |
| 3 | `LogManager::instance()->init()` | `src/QGCApplication.cc:237` |
| 4 | 加载 OpenSans 字体（regular + demibold） | `src/QGCApplication.cc:241-247` |
| 5 | 分支：`_simpleBootTest` → `_initVideo()` + `_initQmlRootWindow()`；`_runningUnitTests` → 什么都不做；否则 → `_initForNormalAppBoot()` | `src/QGCApplication.cc:249-257` |

**`_initQmlRootWindow()`**（`src/QGCApplication.cc:274-296`）——QML 引擎创建与根窗口加载的核心：

```
276: QQuickStyle::setStyle("Basic")
277: QGCCorePlugin::instance()->init()
278: MAVLinkProtocol::instance()->init()
279: MultiVehicleManager::instance()->init()
280: _qmlAppEngine = QGCCorePlugin::instance()->createQmlApplicationEngine(this)
281-282: 连接 objectCreationFailed → QCoreApplication::quit
286: addImageProvider("QGCImages", new QGCImageProvider())
287: addImageProvider(ColoredSvgImageProvider::ProviderId, new ColoredSvgImageProvider())
289: QGCCorePlugin::instance()->createRootWindow(_qmlAppEngine)
293: GraphicsSetup::configureMainWindow(mainRootWindow())
295: return mainRootWindow() != nullptr
```

其中 `addImageProvider` 必须在 `createRootWindow` 之前完成，源码注释给出理由：根 QML 引用了 `QGCColoredImage`，加载时即解析 `image://coloredsvg/...`（`src/QGCApplication.cc:284-285`）。

**`_initForNormalAppBoot()`**（`src/QGCApplication.cc:298-368`）的后续初始化顺序：`_initVideo()`（300）→ `_initQmlRootWindow()`（302）→ `AudioOutput`（304）→ `FollowMe`（306）→ `QGCPositionManager`（307）→ `LinkManager`（308）→ `GPSManager`（309）→ `NTRIPManager`（310）→ `VideoManager::init(mainRootWindow())`（311）→ 主窗口就绪后才允许弹错误框（322）→ `MAVLinkProtocol::checkForLostLogFiles()`（352）→ `LinkManager::loadLinkConfigurationList()`（355）→ `JoystickManager::init()`（358）→ 设置升级提示（360-364）→ `LinkManager::startAutoConnectedLinks()`（367）。Linux 段还有 /etc/group `dialout` 权限检查（`src/QGCApplication.cc:324-349`）。

### 5.3 QML 引擎如何创建、根窗口从哪个 QML 进入

| 环节 | 实现 | 证据 |
|---|---|---|
| 引擎创建（默认实现） | `new QQmlApplicationEngine(parent)`，随后 `addImportPath("qrc:/qml")`，并注入 context property `joystickManager` | `src/API/QGCCorePlugin.cc:300-306` |
| 根窗口加载（默认实现） | `qmlEngine->load(QUrl(QStringLiteral("qrc:/qml/QGroundControl/MainWindow.qml")))` | `src/API/QGCCorePlugin.cc:313-316` |
| 根窗口获取 | `QGCApplication::mainRootWindow()`（声明 `src/QGCApplication.h:64`，实现 `src/QGCApplication.cc:522`） | — |
| 引擎销毁钩子 | `destroyQmlApplicationEngine`（`src/API/QGCCorePlugin.cc:308-311`）；`shutdown()` 中以注释说明引擎必须经该钩子销毁，否则 `~QGCApplication` 的父子析构会绕过插件的 per-engine 状态释放（`src/QGCApplication.cc:771-777`） | — |

**根 QML 入口即 `src/MainWindow/MainWindow.qml`**（800 行）。其资源路径 `qrc:/qml/QGroundControl/MainWindow.qml` 与构建配置吻合：`qt_add_qml_module(QGroundControlModule URI QGroundControl ... RESOURCE_PREFIX /qml ...)`（`src/CMakeLists.txt:15-25`）配合 `QT_RESOURCE_ALIAS` 把该文件扁平化为 `MainWindow.qml`（`src/CMakeLists.txt:10-13`）。

### 5.4 根 QML 结构（`src/MainWindow/MainWindow.qml`）

- 顶层是 `ApplicationWindow`（`src/MainWindow/MainWindow.qml:17-18`），窗口属性含 Android 特化 flags（第 21 行）与四边 padding 归零（第 23-29 行，注释解释 Qt 6.9+ 会按安全区自动加 inset）。
- `Component.onCompleted` 启动首次运行提示链 `firstRunPromptManager.nextPrompt()`（`src/MainWindow/MainWindow.qml:31-34`）；提示管理器实现于第 41-66 行，末尾调用 `showPreFlightChecklistIfNeeded()`。
- 全局状态对象 `globals`（`src/MainWindow/MainWindow.qml:73-90`）持有 `activeVehicle`（取自 `QGroundControl.multiVehicleManager.activeVehicle`）、默认字体尺寸、`validationErrorCount`、`navigationBlockedReason` 等；全局调色板 `QGCPalette { id: qgcPal }` 在第 93 行。
- 三视图切换：`FlyView { id: flyView }`（`src/MainWindow/MainWindow.qml:329-330`）、`PlanView { id: planView }`（第 335-336 行）、`showFlyView()`/`showPlanView()` 函数（第 136、142 行）。
- 分析页与工具抽屉：工具抽屉可调 `showSettingsPage(settingsPage)`（第 180 行）、`toolDrawerToolbar`（第 434 行）、`createAnalyzePage(source)`（第 712 行）与 `createWindowedAnalyzePage(...)`（第 733 行）。
- 导入的 QGC 模块（`src/MainWindow/MainWindow.qml:7-13`）：`QGroundControl`、`QGroundControl.Controls`、`QGroundControl.FactControls`、`QGroundControl.FlyView`、`QGroundControl.FlightMap`、`QGroundControl.PlanView`、`QGroundControl.Toolbar`。
- 窗口位置/尺寸持久化委托给 `MainWindowSavedState.qml`（`src/MainWindow/MainWindow.qml:37-39`，该文件 90 行）。

---

## 6. 前端 UI 技术栈

### 6.1 版本口径

| 项 | 值 | 证据 |
|---|---|---|
| QML 模块版本 | `VERSION 1.0`（全部 25 个 `qt_add_qml_module` 均显式声明） | 例：`src/CMakeLists.txt:17`、`src/QmlControls/CMakeLists.txt:90` |
| Qt 依赖区间 | `>= 6.11.0` 且 `<= 6.11.1`（`find_package(... ${MIN}...${MAX} REQUIRED ...)`） | `CMakeLists.txt:252-253`、`cmake/CustomOptions.cmake:345-352` |
| `qt_standard_project_setup` | `REQUIRES ${QGC_QT_MINIMUM_VERSION} SUPPORTS_UP_TO ${QGC_QT_MAXIMUM_VERSION} I18N_SOURCE_LANGUAGE en` | `CMakeLists.txt:289-291` |
| Qt 策略 | QTP0001–QTP0005 全部 `NEW`（现代资源处理/URI/类型注册/翻译/部署） | `CMakeLists.txt:296-312` |
| 废弃 API 闸门 | `QT_DISABLE_DEPRECATED_UP_TO=0x060B00`、`QT_ENABLE_STRICT_MODE_UP_TO=0x060B00`（即 6.11.0） | `cmake/CustomOptions.cmake:377-384`，注入于 `src/CMakeLists.txt:167-171` |
| QML 语法版本 | QML 文件 `import` 语句**不带版本号**（如 `import QtQuick`、`import QtQuick.Controls`），依赖 Qt 6 的隐式版本 | `src/MainWindow/MainWindow.qml:1-5` |
| QML 缓存生成 | `QT_QML_NO_CACHEGEN` 在 Debug 默认 ON（跳过 qmlcachegen） | `cmake/CustomOptions.cmake:73`、`CMakeLists.txt:314-316` |
| QML 调试 | `QGC_DEBUG_QML` 在 Debug 默认 ON，非 Release 时定义 `QT_QML_DEBUG` | `cmake/CustomOptions.cmake:72`、`src/CMakeLists.txt:176` |
| QML 静态检查配置 | `qmllint` 受 `.qmllint.ini`、`.qmlformat.ini` 约束（顶层文件实测存在） | 顶层目录清单 |
| qmllint 上下文导出 | `QT_QMLLINT_CONTEXT_PROPERTY_DUMP` 默认 ON（Qt 6.11+） | `cmake/CustomOptions.cmake:374` |
| qmlls 配置生成 | `QT_QML_GENERATE_QMLLS_INI` 非 CI 环境默认 ON | `cmake/CustomOptions.cmake:367-372` |

### 6.2 QML 目录分布

按 QML 行数排序（实测，见 2.1 节表）：AutoPilotPlugins 11605 > QmlControls 9900 > PlanView 8283 > FlyView 6939 > AppSettings 6039 > Toolbar 5213 > FlightMap 5006 > AnalyzeView 4395 > GeoMap 2725 > Vehicle 2627 > Viewer3D 1130 > FactSystem 947 > MainWindow 890 > FirmwarePlugin 433 > GPS 299 > LogManager 206 > FirstRunPromptDialogs 135。

全 `src/` 共 478 个 QML 文件、66772 行；`resources/` 下 QML 文件数为 0（实测），即**全部 QML 都规整在 `src/` 内的模块目录里**，没有散落在资源目录。

### 6.3 C++ 侧如何把类型注册给 QML

本版本使用 **Qt6 声明式注册**（`QML_ELEMENT` 系列宏 + `qt_add_qml_module` 生成注册代码），而不是 `qmlRegisterType` 手工调用。实测依据：

- 全 `src/` 命中 `QML_ELEMENT|QML_SINGLETON|QML_UNCREATABLE|QML_NAMED_ELEMENT|QML_ANONYMOUS|QML_FOREIGN` 共 **280 处**，分布在 21 个模块中，按模块计数：

| 模块 | 注册宏命中数 |
|---|---|
| Settings | 54 |
| QmlControls | 35 |
| Vehicle | 27 |
| AutoPilotPlugins | 21 |
| GPS | 20 |
| MissionManager | 17 |
| AnalyzeView | 14 |
| Comms | 13 |
| FactSystem | 13 |
| GeoMap | 13 |
| Viewer3D | 9 |
| LogManager | 8 |
| Utilities | 7 |
| Camera | 6 |
| API | 5 |
| MAVLink | 5 |
| VideoManager | 4 |
| Joystick | 4 |
| QtLocationPlugin | 2 |
| Gimbal | 2 |
| ADSB | 1 |

- 命中数最多的单文件：`src/GPS/GPSQmlTypes.h`（10 处，用 `QML_FOREIGN` + `QML_NAMED_ELEMENT` 把无 `Q_OBJECT` 的类型挂进 QML，第 15-55 行）、`src/AutoPilotPlugins/PX4/AirframeComponentController.h`（5 处）、`src/Utilities/Logging/LoggingCategoryModel.h`（4 处）、`src/Settings/RTKSettings.h`（4 处）、`src/API/QGCOptions.h`（4 处）。
- 三种典型用法：
  1. **单例**：`QML_SINGLETON`，如 `QML_NAMED_ELEMENT(QGroundControl)`+`QML_SINGLETON`（`src/QmlControls/QGroundControlQmlGlobal.h:45-46`）、`MAVLink` 单例（`src/MAVLink/QGCMAVLink.h:15-16`）。
  2. **不可创建实例**：`QML_UNCREATABLE("...")`，用于只能由 C++ 创建、仅作为属性类型暴露的类，如 `Vehicle`（`src/Vehicle/Vehicle.h:90`）、`FactGroup`（`src/FactSystem/FactGroup.h:20`）、`SettingsManager`（`src/Settings/SettingsManager.h:42`）。
  3. **可创建实例**：仅 `QML_ELEMENT`，如 `Fact`（`src/FactSystem/Fact.h:19`）、`FactMetaData`（`src/FactSystem/FactMetaData.h:19`）、`Vehicle` 之外的 C++ 值类型。
- 另有一条非声明式通道：`QQmlApplicationEngine::rootContext()->setContextProperty("joystickManager", JoystickManager::instance())`（`src/API/QGCCorePlugin.cc:304`）。
- 自定义构建可替换 `createQmlApplicationEngine`（`src/API/QGCCorePlugin.h:112`）与 `createRootWindow`（`src/API/QGCCorePlugin.h:120`），这是改 UI 骨架的官方入口。
- **源码树内无 `qmldir`、无 `*.qmltypes`**（实测递归搜索结果为 0）：这两类文件由 `qt_add_qml_module` 在构建期生成。

### 6.4 图像提供者与其它 QML 支撑

- 引擎注册两个 image provider：`QGCImages`（`QGCImageProvider`）与 `ColoredSvgImageProvider::ProviderId`（着色 SVG，供 `QGCColoredImage` 使用），均在根窗口加载前注册（`src/QGCApplication.cc:286-287`）。
- 着色图像定义：`src/QmlControls/ColoredSvgImageProvider.h` / `.cc`；对应 QML 控件 `src/QmlControls/QGCColoredImage.qml`。
- 控件样式强制为 `Basic`（`QQuickStyle::setStyle("Basic")`，`src/QGCApplication.cc:276`），并通过 `qtquickcontrols2.conf` 资源接管（`CMakeLists.txt:519-525`，资源别名同样设置于第 519 行）。
- 图标/矢量资源经 `qt_add_resources` 打进可执行目标（`CMakeLists.txt:375-387`），字体资源前缀 `/fonts`（`CMakeLists.txt:375-380`），应用资源前缀 `/res`（`CMakeLists.txt:382-387`）。
- 翻译：`qt_add_translations` 对 `translations/qgc_*.ts` 编译，Debug 或多配置生成器下排除伪本地化 `_eo.ts`（`CMakeLists.txt:495-514`）；运行时从 `:/i18n` 加载 `qgc_source_` / `qgc_json_`（`src/QGCApplication.cc:208-217`）。

---

## 7. 对二次开发的客观提示（基于以上结构）

1. **改「参数显示」**：改 `Fact` 的 QML 暴露面（`src/FactSystem/Fact.h:21-65`）或复写 `QGroundControl.FactControls` 控件（`src/FactSystem/FactControls/CMakeLists.txt:11-14` 列出的 17 个 QML），不需要动 C++ 核心。
2. **改「设置页」**：改 `src/Settings/*.SettingsGroup.json` 与 `src/AppSettings/pages/*.SettingsUI.json` 即可，页面由 `generate_pages` 在 configure 期生成（`src/AppSettings/CMakeLists.txt:84-100`）。
3. **改「主界面骨架」**：替换 `QGCCorePlugin::createRootWindow`（`src/API/QGCCorePlugin.cc:313-316`）或在自定义 overlay 中重写该方法；根 QML 为 `src/MainWindow/MainWindow.qml`。
4. **改「配色/主题」**：`QGCCorePlugin::paletteOverride`（`src/API/QGCCorePlugin.h:106`）+ `QGCPalette`（`src/QmlControls/QGCPalette.h:82`）。
5. **改「固件差异行为」**：不得在主流程写固件分支，必须走 `FirmwarePlugin` 子类（`src/FirmwarePlugin/FirmwarePlugin.h:66-70` 的架构约定）。
6. **构建前提**：本机需先具备 Qt 6.11.x 并设置 `QT_ROOT_DIR`（`cmake/presets/Windows.json:10`）与 MSVC 工具链，否则 `CMakeLists.txt:241-250` 会在 configure 阶段直接 `FATAL_ERROR`。当前机器两项均缺失（实测）。

---

## 8. 未验证项与边界声明

1. **未编译**：本文所有结论来自源码静态阅读，未运行 QGC，未验证任何运行期行为（如 QML 实际加载顺序中的时序细节、启动耗时、渲染路径）。
2. **`src/AutoPilotPlugins/Generic/`（2 个文件）**：仅确认目录存在与文件数，未核对其是否注册独立 QML 模块。
3. **顶层 `vcpkg.json` / `package.json` 的作用**：`package.json` 实测存在但仅确认其存在，未读取其 scripts 内容；`vcpkg.json` 在顶层文件清单中**未出现**，本仓库是否使用 vcpkg 未验证。
4. **GStreamer 在 Windows 的完整下载链路**：仅确认 `cmake/GStreamer/platform/Android.cmake:42` 与校验和配置位置，Windows 分支的具体实现文件未逐一读取。
5. **`cmake/install/` 与 `deploy/` 的打包细节**：未展开分析（不在本任务范围）。
6. **QML 模块间的 `DEPENDENCIES` 完整图**：仅 Read 了 `QGroundControl.AppSettings` 的 `DEPENDENCIES QGroundControl.Controls QGroundControl.LogManager`（`src/AppSettings/CMakeLists.txt:117`），其余模块的依赖边未逐一提取。
7. **`QGC_CUSTOM_DIR` 的默认路径 `custom/` 在源码树中不存在**（顶层目录清单中无 `custom/`），即上游默认构建不带 overlay；`custom-example/` 需被复制/重命名为 `custom/` 才会生效——该重命名步骤在官方文档中的描述未验证。
8. **行数口径**：所有行数含空行与注释，非有效代码行（LOC 口径为「文件总行数」）。
