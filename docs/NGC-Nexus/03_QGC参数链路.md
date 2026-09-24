# QGC 参数子系统链路：MAVLink 报文 → Fact 对象

- 分析对象：QGroundControl 源码 `E:\04-Workspace_workbudy\15_NGC-Nexus\repo`
- 版本：HEAD `25185047e855937d694a0174c1541d8f7d794a74`（master，浅克隆 depth=1，无 `.gitmodules`，顶层无 `libs/`）
- 方法：仅源码阅读（本机无 Qt6 / MSVC，不编译）。所有行号均由工具实际读取确认，路径相对 `repo` 根。
- 结论口径：本文只描述该 HEAD 的实际代码行为；与 `docs/` 下开发文档口径不一致处已在 §3.1 指出。
- 引用约定：所有路径相对 `repo` 根；写作 `:NNN` / `:NNN-MMM`（省略文件名）时，指同一句/同一表格行内**紧邻前一个**完整路径的同一文件。

---

## 0. 一句话链路

```
LinkInterface::bytesReceived
  └─> MAVLinkProtocol::receiveBytes()                      （字节 → mavlink_message_t）
        └─> MAVLinkProtocol::_updateStatus() → emit messageReceived(link, message)
              └─> Vehicle::_mavlinkMessageReceived()        （sysid/compid 过滤 + 插件预处理）
                    └─> ParameterManager::mavlinkMessageReceived()   （只认 PARAM_VALUE）
                          └─> ParameterManager::_handleParamValue()
                                ├─> Fact::setMetaData(CompInfoParam::factMetaDataForName(...))
                                └─> Fact::containerSetRawValue(value)
                                      └─> Fact::valueChanged / rawValueChanged / vehicleUpdated
                                            └─> QML / FactControls / 各 VehicleComponent
写回方向：
Fact::setRawValue() → emit containerRawValueChanged
  └─> ParameterManager::_factRawValueUpdated()
        └─> ParameterManager::_mavlinkParamSet()  → PARAM_SET 状态机 → Vehicle::sendMessageOnLinkThreadSafe()
              └─> 等 PARAM_VALUE / PARAM_ERROR 应答（WaitForParamResponseState）
```

---

## 1. 入口：MAVLink 消息在哪里被接收与分发

### 1.1 各层角色（含"不是什么"）

| 层 | 类 / 文件 | 实际角色 | 关键证据 |
|---|---|---|---|
| 传输层 | `LinkInterface` 各实现（`UDPLink` / `TCPLink` / `SerialLink` / `BluetoothLink` / `LogReplayLink` / `MockLink`） | 把链路收到的裸字节 `emit bytesReceived(this, data)` | `src/Comms/UDPLink.cc:588`、`src/Comms/TCPLink.cc:327`、`src/Comms/Serial/SerialLink.cc:446`、`src/Comms/Bluetooth/BluetoothLink.cc:120`、`src/Comms/LogReplayLink.cc:504`、`src/Comms/MockLink/MockLink.cc:1106` |
| 链路管理 | `LinkManager` | 把链路的 `bytesReceived` 直接接到 `MAVLinkProtocol::receiveBytes` | `src/Comms/LinkManager.cc:194`；断开见 `:204`、`:321` |
| 协议层 | `MAVLinkProtocol`（**在 `src/Comms/`，不在 `src/MAVLink/`**） | 逐字节组帧、签名校验、转发、日志，然后广播 `messageReceived` | `src/Comms/MAVLinkProtocol.cc:102`（`receiveBytes`）、`:115`（`mavlink_parse_char`）、`:150`（调用 `_updateStatus`）、`:285`（`_updateStatus`）、`:294`（`emit messageReceived(link, message)`） |
| 报文定义（编译期） | `src/MAVLink/MAVLinkLib.h:24`（`#include <mavlink.h>`）、`src/MAVLink/MAVLinkMessageType.h`、以及**构建期生成**的 `MAVLinkEnums.h`（生成器 `tools/generators/mavlink_enums.py:1-11`，CMake 规则 `src/MAVLink/CMakeLists.txt:64-83`，输出到 `${CMAKE_BINARY_DIR}`，故源码树中不存在该文件） | 只是 mavlink C 库与消息/枚举类型定义，**没有运行时收发逻辑** | `src/MAVLink/MAVLinkLib.h:1-30` 全文；`src/MAVLink/` 目录下仅有 `MAVLinkFTP`/`ImageProtocolManager`/`StatusTextHandler`/`Signing`/`LibEvents` |
| 机型/固件分类工具 | `QGCMAVLink`（QML singleton） | 提供 `firmwareClass`/`vehicleClass`/`motorCount` 等分类函数 | `src/MAVLink/QGCMAVLink.h:12`、`:39-45` |
| 整车模型 | `Vehicle` | 过滤 sysid、驱动各子模块，最后 `emit mavlinkMessageReceived` 供等待型状态机使用 | `src/Vehicle/Vehicle.cc:525`、`:577`、`:757`；信号声明 `src/Vehicle/Vehicle.h:756` |
| 参数子系统 | `ParameterManager` | 唯一处理 `PARAM_VALUE` 的整车级对象 | `src/FactSystem/ParameterManager.cc:107-136` |

`Vehicle::_mavlinkMessageReceived` 内部的处理顺序（`src/Vehicle/Vehicle.cc:525-592`）：

1. `:527-532` sysid 过滤（非本机 sysid 直接丢弃；`RADIO_STATUS` 例外）；
2. `:535` `_vehicleLinkManager->mavlinkMessageReceived(link, message)`；
3. `:564` `_firmwarePlugin->adjustIncomingMavlinkMessage(this, &message)`，返回 false 则终止；
4. `:569` `QGCCorePlugin::instance()->mavlinkMessage(this, link, message)`，返回 false 则终止；
5. `:573` `_terrainProtocolHandler->mavlinkMessageReceived(message)`，返回 false 则终止；
6. `:576` `_ftpManager->_mavlinkMessageReceived(message)`；
7. `:577` `_parameterManager->mavlinkMessageReceived(message)`；
8. `:578` `ImageProtocolManager`（经 `QMetaObject::invokeMethod`）、`:579` `_remoteIDManager`、`:581` `_reqMsgCoord->handleReceivedMessage`；
9. `:584-585` 动态 FactGroup 列表（Battery / EscStatus）；`:588-590` 遍历 `factGroups()` 调 `handleMessage`；`:592` `this->handleMessage(this, message)`；
10. `:594` 起按 msgid 分派本类专属处理；
11. `:757` `emit mavlinkMessageReceived(message)`——这是 `WaitForParamResponseState`、`MissionManager`、`PlanManager`、`GimbalController`、`QGCCameraManager` 等的统一订阅点。

注意 `Vehicle` 并未被 `MultiVehicleManager` 调用转交消息：连接在构造函数里以信号方式建立（`src/Vehicle/Vehicle.cc:123`）。`MultiVehicleManager` 只负责在心跳到达时建车并传入 `componentId`（`src/Vehicle/MultiVehicleManager.cc:118`）。

### 1.2 `PARAM_VALUE` 的处理位置

唯一处理点：`ParameterManager::mavlinkMessageReceived`（`src/FactSystem/ParameterManager.cc:107`）。

- `:109` `if (message.msgid == MAVLINK_MSG_ID_PARAM_VALUE)`；
- `:111` `mavlink_msg_param_value_decode`；
- `:114-116` 用定长数组 + `strncpy` 为 `param_id` 补 `\0`（MAVLink 的 `param_id` 不保证终止）；
- `:120-123` 若启用 FTP 参数文件下载（`_tryftp`）、compid 为 `MAV_COMP_ID_AUTOPILOT1`、初始加载未完成且参数名不是 `_HASH_CHECK`，则**丢弃**该 `PARAM_VALUE`（FTP 路径用文件代替流）；
- `:125-127` 用 `mavlink_param_union_t` 承载原始字节；
- `:130` `_mavlinkParamUnionToVariant()` 转 `QVariant`，失败即返回；
- `:134` 调 `_handleParamValue(message.compid, name, param_count, param_index, param_type, value)`。

另一处处理 `PARAM_VALUE` 的是"等待应答"状态机：`WaitForParamResponseState::_messageReceived`（`src/Utilities/StateMachine/States/WaitForParamResponseState.cc:32`），它订阅的是 `Vehicle::mavlinkMessageReceived`（`:20`），用于 PARAM_SET / PARAM_REQUEST_READ 的应答确认，**不建 Fact**。

### 1.3 `PARAM_EXT_VALUE` 的处理位置（不属于 ParameterManager）

`PARAM_EXT_*` 在本 HEAD 只服务**相机参数**（MAVLink 相机协议），与整车参数链路并行、互不共享存储：

- 订阅与分派：`QGCCameraManager::_mavlinkMessageReceived`（`src/Camera/QGCCameraManager.cc:155`），连接点 `:88`（订阅 `Vehicle::mavlinkMessageReceived`）；`:186-191` 分派 `MAVLINK_MSG_ID_PARAM_EXT_ACK` / `MAVLINK_MSG_ID_PARAM_EXT_VALUE`；
- 解码：`:462` `mavlink_msg_param_ext_ack_decode`、`:472` `mavlink_msg_param_ext_value_decode`；
- 处理：`VehicleCameraControl::handleParamExtAck`（`src/Camera/VehicleCameraControl.cc:1290`）、`VehicleCameraControl::handleParamExtValue`（`:1309`）；两者都按 `param_id` 查 `_paramIO`，未知名只告警；
- 请求全量：`VehicleCameraControl::_requestAllParameters`（`:1256`）→ `:1269` `mavlink_msg_param_ext_request_list_pack_chan`；
- 单参数读：`QGCCameraParamIO` 内 `mavlink_msg_param_ext_request_read_pack_chan`（`src/Camera/QGCCameraIO.cc:356`）；
- 写：`mavlink_msg_param_ext_set_encode_chan`（`src/Camera/QGCCameraIO.cc:202`）；应答 `QGCCameraParamIO::handleParamAck`（`:215`）用 `_fact->containerSetRawValue(val)`（`:223`）回填相机 Fact。

即：**相机的 `PARAM_EXT_VALUE` 走 `QGCCameraParamIO` 自己的 `Fact` 实例，车辆级 `ParameterManager` 完全不参与**。

### 1.4 `PARAM_REQUEST_LIST` / `PARAM_REQUEST_READ` / `PARAM_SET` 的发出位置

| 报文 | 发出函数 | 位置 |
|---|---|---|
| `PARAM_REQUEST_LIST` | `ParameterManager::_startParameterDownload`（`mavlink_msg_param_request_list_pack_chan`） | `src/FactSystem/ParameterManager.cc:718-725` |
| `PARAM_REQUEST_READ`（按索引，补漏用） | `ParameterManager::_sendParamRequestReadIndex` | `src/FactSystem/ParameterManager.cc:991-1010` |
| `PARAM_REQUEST_READ`（按名，单参数刷新） | `ParameterManager::_mavlinkParamRequestRead` 内的编码器 | `src/FactSystem/ParameterManager.cc:1012-1026` |
| `PARAM_REQUEST_READ`（PX4 `_HASH_CHECK`） | `ParameterManager::_requestHashCheck` | `src/FactSystem/ParameterManager.cc:965-989` |
| `PARAM_SET`（写参数） | `ParameterManager::_mavlinkParamSet` 内的编码器 | `src/FactSystem/ParameterManager.cc:306-329` |
| `PARAM_SET`（回写缓存 CRC 握手） | `ParameterManager::_tryCacheHashLoad` 尾部 | `src/FactSystem/ParameterManager.cc:1228-1245` |
| `PARAM_EXT_REQUEST_LIST`（相机） | `VehicleCameraControl::_requestAllParameters` | `src/Camera/VehicleCameraControl.cc:1269` |

全仓不存在 `MAVLINK_MSG_ID_PARAM_EXT_REQUEST_LIST` / `..._PARAM_EXT_REQUEST_READ` / `..._PARAM_EXT_SET` 常量名的使用点（grep 无匹配），相机侧一律直接调用 `mavlink_msg_param_ext_*` 打包函数。

---

## 2. ParameterManager：请求、接收、缓存、写入、重试、多组件

文件：`src/FactSystem/ParameterManager.h`（260 行）、`src/FactSystem/ParameterManager.cc`（1947 行）。

### 2.1 实例化与生命周期

- `Vehicle` 构造函数中创建：`_parameterManager = new ParameterManager(this);`（`src/Vehicle/Vehicle.cc:283`），parent 为 Vehicle，故随车销毁。
- 构造参数：`ParameterManager::ParameterManager(Vehicle *vehicle)`（`src/FactSystem/ParameterManager.cc:38`）。构造期确定三件事：
  - `:41` `_logReplay`：主链路是日志回放链路；
  - `:42` `_disableAllRetries(_logReplay)`：回放模式下彻底关闭重试；
  - `:43` `_waitForParamValueAckMs`：单测 50 ms / 正常运行 `kWaitForParamValueAckMs = 1000 ms`（常量见 `src/FactSystem/ParameterManager.h:113`）；
  - `:44` `_tryftp`：`apmFirmware() || px4Firmware()`，即 PX4 与 ArduPilot 都可能走 FTP 参数文件路径；
  - `:48-51` 离线编辑车直接走 `_loadOfflineEditingParams()` 并 return，不启动任何定时器；
  - `:57-69` 三个单次定时器：`_hashCheckTimer`（1 s / 单测 200 ms，`:58`）、`_paramRequestListTimer`（5 s / 单测 500 ms，`:62`）、`_waitingParamTimeoutTimer`（3 s / 单测 500 ms，`:66`）；
  - `:72` 确保参数缓存目录存在。

### 2.2 全量参数请求的三条路径

入口：`ParameterManager::refreshAllParameters(uint8_t componentId)`（`src/FactSystem/ParameterManager.cc:611`）。它先 `_resetHashCheck()`、`setParameterDownloadSkipped(false)`、置 `_refreshAllProgressActive = true`，再调 `_startParameterDownload(componentId)`（`:646`）。无参版本 `refreshAllParameters()` 以 `MAV_COMP_ID_ALL` 调用（`src/FactSystem/ParameterManager.h:54`）。

`_startParameterDownload`（`src/FactSystem/ParameterManager.cc:646-730`）按优先级分支：

1. `:648-651` 取不到主链路直接返回；
2. `:653-662` 高延迟链路或日志回放：直接把 `_parametersReady = true`、`_missingParameters = true`、`_initialLoadComplete = true`，发信号后结束，**不下载**；
3. `:664-672` **PX4 且初始加载未完成且未做过 hash check**：起 `_hashCheckTimer`，调 `_requestHashCheck(componentId)` 先问 `_HASH_CHECK`，命中缓存就免下载；`componentId == MAV_COMP_ID_ALL` 时改问 `MAV_COMP_ID_AUTOPILOT1`（`:669-671`）；
4. `:673-700` **FTP 路径**（`_tryftp` 且组件是 ALL 或 AUTOPILOT1）：起 `_paramRequestListTimer`（若未完成初始加载），连接 `FTPManager::downloadComplete`，调 `ftpManager->download(MAV_COMP_ID_AUTOPILOT1, "@PARAM/param.pck?withdefaults=1", TempLocation, "param.pck", false)`（`:689-693`）；`_ftpDownloadInProgress` 时拒绝重入（`:674-679`）；
5. `:701-726` **常规 `PARAM_REQUEST_LIST` 路径**：先按 `_paramCountMap` 重建 `_waitingReadParamIndexMap` 等待表（`:707-716`），再 `mavlink_msg_param_request_list_pack_chan(..., _vehicle->id(), componentId)` 并经 `_vehicle->sendMessageOnLinkThreadSafe(...)` 发出（`:718-725`）。

`_paramRequestListTimer` 溢出即 `ParameterManager::_paramRequestListTimeout()`（`:1501`）：回放模式直接宣告完成（`:1503-1513`）；否则 `++_initialRequestRetryCount <= _maxInitialRequestListRetry`（4，常量 `src/FactSystem/ParameterManager.h:114`）时重发（`:1515-1519`），耗尽后发 `initialParametersRequestFailed()`（`:1528`）。

FTP 路径的收敛逻辑在 `ParameterManager::_ftpDownloadComplete`（`:547`）：成功则 `_parseParamFile(fileName)`（`:558`）；`"File Not Found"`（`:565`）或进度过慢（`:570`）或重试次数用尽（`:572`）时把 `_tryftp = false`、`_initialRequestRetryCount = 0`，切到常规下载（`:579-593`）；`_ftpDownloadProgress`（`:596`）在进度 > 0.001 时停掉 request-list 定时器（`:600-602`）。

### 2.3 接收与"参数 → Fact"的落点

`ParameterManager::_handleParamValue`（`src/FactSystem/ParameterManager.cc:138-279`）是链路核心：

- `:152-155` ArduPilot 会在未请求时抢跑流式 `PARAM_VALUE`（`param_index == 65535`）：若 `_paramRequestListTimer` 仍活动且名字不是 `_HASH_CHECK`，直接丢弃；
- `:157-164` PX4 且 `_HASH_CHECK`：停 hash 定时器，初始加载未完成时调 `_tryCacheHashLoad(_vehicle->id(), componentId, parameterValue)` 后返回（该分支**不产生 Fact**）；
- `:166` 收到任意有效参数即停 `_paramRequestListTimer`；`:185` 停 `_waitingParamTimeoutTimer`；
- `:187-194` 首次见到某 componentId 时记录 `_paramCountMap[componentId]` 并累加 `_totalParamCount`；
- `:197-205` 首次见到该组件时把 `0..parameterCount-1` 全部索引登记进 `_waitingReadParamIndexMap`（值 0 表示重试次数）；
- `:212-216` 从等待表移除该索引，并从批量队列 `_indexBatchQueue` 移除，随后 `_fillIndexBatchQueue(false)` 立刻补发下一批；
- `:224-240` 仍有余量则重启 `_waitingParamTimeoutTimer`；若连默认组件的参数都还没有，也再等一轮；否则不重启；
- `:242` `_updateProgressBar()`；
- `:245-260` **Fact 创建/复用**：
  - 已有 → `fact = _mapCompId2FactMap[componentId][parameterName]`；
  - 新建 → `new Fact(componentId, parameterName, mavTypeToFactType(mavParamType), this)`（`:250`），随后 `fact->setMetaData(_vehicle->compInfoManager()->compInfoParam(componentId)->factMetaDataForName(parameterName, fact->type()))`（`:251-252`），写入 `_mapCompId2FactMap`（`:254`），连接 `Fact::containerRawValueChanged → ParameterManager::_factRawValueUpdated`（`:257`），`emit factAdded(componentId, fact)`（`:259`）；
- `:262` `fact->containerSetRawValue(parameterValue)`——**车辆来的值不触发回写**（见 §4.3）；
- `:264-272` 参数缓存只在 PX4 上写（ArduPilot 参数易变、Solo 会在飞行中流式刷新参数）：当"上一轮等待数 != 0 且本轮为 0"即所有读刚完成时，调 `_writeLocalParamCache(_vehicle->id(), componentId)`；
- `:276` `_checkInitialLoadComplete()`。

### 2.4 超时、重试与"放弃组件"

- `ParameterManager::_waitingParamTimeout()`（`:900`）：回放模式直接返回（`:902`）；置 `_indexBatchQueueActive = true`（`:909`）；`_giveUpOnUnresponsiveComponents()`（`:911`）；`_fillIndexBatchQueue(true)`（`:914`，超时路径会先清空队列再重填）；若仍无任何默认组件参数，再等一轮（`:915-922`）；最后 `_checkInitialLoadComplete()` + `_updateProgressBar()`（`:925-927`）。
- `ParameterManager::_fillIndexBatchQueue(bool)`（`:852`）：
  - 单批在飞上限 `kIndexBatchMaxOutstanding = 10`（`:878`，常量 `src/FactSystem/ParameterManager.h:116`）；
  - 每次入队先自增重试计数（`:882`）；
  - `_disableAllRetries` 或计数 > `_maxInitialLoadRetrySingleParam = 5`（常量 `src/FactSystem/ParameterManager.h:115`）时，把索引记入 `_failedReadParamIndexMap` 并放弃（`:883-887`）；
  - 否则入 `_indexBatchQueue` 并 `_sendParamRequestReadIndex(componentId, paramIndex)`（`:890-891`）。
- `ParameterManager::_giveUpOnUnresponsiveComponents()`（`:935`）：跳过默认组件（`:940-942`，默认组件永不提前放弃）；在飞请求数 < `kUnresponsiveMinOutstanding = 5`（`:946`）或本轮有收到值（`:946`）则不计；连续静默周期达到 `kUnresponsiveSilentCycles = 2`（`:950`，常量 `src/FactSystem/ParameterManager.h:118`）即把剩余索引全部标记失败并清表（`:954-959`）；末尾清空本轮计数（`:962`）。
- PARAM_SET 重试常量 `kParamSetRetryCount = 2`（`src/FactSystem/ParameterManager.h:111`），PARAM_REQUEST_READ 重试常量 `kParamRequestReadRetryCount = 2`（`:112`）。
- 批量刷新 `ParameterManager::bulkRefresh(int componentId, const QStringList &names, bool notifyFailure)`（`:768`）：支持 `*` 前缀展开（`:779-790`）、交叉去重（`:786`、`:792`）、未知名告警（`:797`），随后构造 `BulkRefreshJob`（`:806-811`）。`BulkRefreshJob` 的批量重试为**指数退避**：`kMaxRetryRounds = 3`、`kRetryBaseDelayMs = 1000`（`src/FactSystem/BulkRefreshJob.h:22-23`），`delayMs = _retryBaseDelayMs << _round`（`src/FactSystem/BulkRefreshJob.cc:74`），成功/失败由 `_paramRequestReadSuccess` / `_paramRequestReadFailure` 驱动（`src/FactSystem/BulkRefreshJob.cc:22-23`、`:36`、`:49`），耗尽后按 `notifyFailure` 弹一条汇总提示（`src/FactSystem/BulkRefreshJob.cc:85-89`）。

### 2.5 初始加载完成的判定与对外信号

`ParameterManager::_checkInitialLoadComplete()`（`:1380`）：

- `:1387-1392` **只有默认组件决定 readiness**；其它组件（如 PX4 桥接的 DroneCAN 节点）在后台继续加载；
- `:1394-1397` 还没有默认组件参数则不算完成；
- `:1400-1402` 置 `_initialLoadComplete = true`、清 `_refreshAllProgressActive`、进度归零；
- `:1418-1431` `_missingParameters = !_failedReadParamIndexMap[defaultCompId].isEmpty()`，缺失时打印失败索引并 `QGC::showAppMessage`；
- `:1434-1437` `_parametersReady = true`；调 `_vehicle->autopilotPlugin()->parametersReadyPreChecks()`；`emit parametersReadyChanged(true)`；`emit missingParametersChanged(...)`；
- `:1439` `_checkOtherComponentsLoadComplete()`（`:1452`）——非默认组件全部结束或放弃后，汇总一条"未能取回组件 X 参数"的提示（`:1479-1486`）。

`AutoPilotPlugin::parametersReadyPreChecks()`（`src/AutoPilotPlugins/AutoPilotPlugin.cc:45`）负责重算 setup 完成度并连接各 `VehicleComponent::setupCompleteChanged`——这是参数系统与 `AutoPilotPlugins` 的唯一硬连接点。

### 2.6 写入路径

触发链：`Fact::setRawValue` → `emit containerRawValueChanged` → `ParameterManager::_factRawValueUpdated`（`:536`）→ `_mavlinkParamSet(fact->componentId(), fact->name(), fact->type(), rawValue)`（`:544`）。

`ParameterManager::_mavlinkParamSet`（`:306`）用 `QGCStateMachine` 显式建模（注释见 `:382-394`）：

```
sendParamSet → 增 pending 写计数 → 等应答 → 减计数 → 成功
     ↑                                  │
     └── 超时：减计数后回到 sendParamSet ─┘
     PARAM_ERROR / 重试耗尽 → 减计数 → 失败 → 用户提示 → 刷新该参数（DOES_NOT_EXIST 时跳过）
```

- 发送状态 `SendMavlinkMessageState(stateMachine, paramSetEncoder, kParamSetRetryCount)`（`:398`）；
- 计数状态 `FunctionState` 包装 `_incrementPendingWriteCount`（`:399-401`）/ `_decrementPendingWriteCount`（`:402-410`）；
- 等待状态 `WaitForParamResponseState(stateMachine, _waitForParamValueAckMs, checkForCorrectParamValue, checkForParamError)`（`:411`）；应答校验函数 `:331-366`（先比 compid、再比名字、再比 `QVariant` 类型，`REAL32` 用 `QGC::fuzzyCompare`），`PARAM_ERROR` 校验 `:368-380`；
- 失败后置位 `pendingWrites` 的状态由 `_incrementPendingWriteCount`（`:1928`，首个写置 true 并发 `pendingWritesChanged(true)`）与 `_decrementPendingWriteCount`（`:1936`，归零发 false）维护；`pendingWrites()` 查 `_pendingWritesCount > 0`（`:1687`）；
- `PARAM_ERROR` 语义到文案的映射在 `WaitForParamResponseState::_paramErrorToString`（`src/Utilities/StateMachine/States/WaitForParamResponseState.cc:63-87`）。

### 2.7 本地参数缓存（仅 PX4）

- 目录与文件名：`ParameterManager::parameterCacheDir()`（`:1152`）返回 `<QSettings目录>/<appName>/ParamCache`；`parameterCacheFile(vehicleId, componentId)`（`:1161`）返回 `<vehicleId>_<componentId>.v2`；
- 写：`_writeLocalParamCache(int vehicleId, int componentId)`（`:1133`）把 `QMap<name, (ValueType_t, rawValue)>` 用 `QDataStream` 落盘（`:1142-1149`）；
- 读与校验：`_tryCacheHashLoad(int vehicleId, int componentId, const QVariant &hashValue)`（`:1166`）反序列化缓存，按 `QGC::crc32` 对"参数名 + 定长原始值"逐一累计 CRC（`:1196-1209`），跳过 `volatileValue()` 参数（`:1200-1202`）；CRC 与车辆给的 hash 相同则 `_hashCheckDone = true`，把缓存值逐个喂回 `_handleParamValue(...)` 重建 Fact（`:1219-1224`），再发一个 `PARAM_SET` 把 `_HASH_CHECK` 写回以告知"不要再推流"（`:1228-1245`），并播放一段 750 ms 进度动画（`:1249-1266`）；不匹配则发 `Parameter cache CRC match failed` 并回落 `_startParameterDownload(MAV_COMP_ID_ALL)`（`:1267-1288`）；
- 独立 hash 检查：`tryHashCheckCacheLoad()`（`:619`）用于"飞行中只想读缓存"的场景，失败发 `cacheCheckOnlyFailed()`（`:626`、`:632`、`:642`、`:1179`、`:1281`、`:1494`），由 `InitialConnectStateMachine` 消费（`src/Vehicle/InitialConnectStateMachine.cc:395-400`）。

### 2.8 多组件（componentId）处理

数据结构（`src/FactSystem/ParameterManager.h:207`、`:243-247`）：

| 成员 | 键 → 值 | 用途 |
|---|---|---|
| `_mapCompId2FactMap` | compId → (参数名 → `Fact*`) | 唯一参数存储，Fact 的 `componentId` 即来源组件 |
| `_paramCountMap` | compId → 该组件参数总数 | `componentIds()` 返回其 keys（`:1682-1685`） |
| `_waitingReadParamIndexMap` | compId → (索引 → 重试次数) | 缺失索引补读 |
| `_failedReadParamIndexMap` | compId → 失败索引列表 | 缺失判定 |
| `_paramValuesReceivedThisCycle` | compId → 本轮收到数 | 判定组件是否静默 |
| `_silentCycleCount` | compId → 连续静默周期数 | 放弃阈值 |

要点：
- 默认组件由 `_actualComponentId()`（`:732`）把外部传入的哨兵值 `defaultComponentId = -1`（`src/FactSystem/ParameterManager.h:108`）解析为 `_vehicle->defaultComponentId()`；解析失败会告警（`:736-738`）；
- 只有默认组件阻塞 `parametersReady`（`:1387-1392`），非默认组件在后台继续，失败单独汇总（`:1452-1487`）；
- 非默认组件连续静默 2 轮且至少有 5 个在飞请求时被整体放弃（`_giveUpOnUnresponsiveComponents`，`:935-963`）；
- 参数名重映射 `_remapParamNameToVersion`（`:1531`）在 `parameterExists`（`:820`）与 `getParameter`（`:830`）里对**入参**做版本回退映射，`noremap.` 前缀可跳过映射（`:1533-1536`），映射表来自 `_vehicle->firmwarePlugin()->paramNameRemapMajorVersionMap()`（`:1546`，虚函数声明 `src/FirmwarePlugin/FirmwarePlugin.h:317`）。

### 2.9 离线编辑与重置

- 离线编辑车：`_loadOfflineEditingParams()`（`:1570`）读 `firmwarePlugin()->offlineEditingParamFile(_vehicle)`（`:1572`），解析 5 列 TSV，逐行建 Fact（`:1626-1631`，注意这里统一落在 `defaultComponentId` 键下），最后置 `_parametersReady = _initialLoadComplete = true`（`:1634-1635`）；
- `resetAllParametersToDefaults()`（`:1639`）发 `MAV_CMD_PREFLIGHT_STORAGE`（参数 2 = 恢复默认）；
- `resetAllToVehicleConfiguration()`（`:1648`）把 `SYS_AUTOCONFIG` 置 2（`:1651-1654`）；
- 导出：`writeParametersToStream(QTextStream&)`（`:1291`）输出 `Vehicle-Id Component-Id Name Value Type` 五列（`:1306`、`:1312`）。

### 2.10 已知类型边界

`ParameterManager::_mavlinkParamUnionToVariant`（`:506`）只覆盖 `REAL32 / UINT8 / INT8 / UINT16 / INT16 / UINT32 / INT32`，其余（含 `REAL64`、`UINT64`、`INT64`）走 `:530-533` 的 `qCCritical` 并返回 false。而 `factTypeToMavType`（`:1322`）与 `mavTypeToFactType`（`:1351`）是支持 64 位类型与 `REAL64` 的。即：类型映射表与实际的 `PARAM_VALUE` 解码路径能力不对称，64 位参数的入向解码在当前实现下会被拒绝（该不对称属源码事实，是否可达取决于固件是否发送此类 `param_type`）。

---

## 3. ParameterLoader / ParameterEditorController 的关系与职责边界

### 3.1 本 HEAD 已不存在 `ParameterLoader` 类

事实：

- `src/FactSystem/` 下无 `ParameterLoader.*`（`glob **/ParameterLoader.*` 在 `repo/src` 下无结果）；`src/FactSystem/CMakeLists.txt:6-26` 的编译清单只有 `Fact`、`FactGroup`、`FactGroupListModel`、`FactGroupWithId`、`FactMetaData`、`FactValueSliderListModel`、`ParameterManager`、`SettingsFact`、`BulkRefreshJob`；
- 全仓 `ParameterLoader` 只剩文档字符串：`docs/en|zh|tr|ko/qgc-dev-guide/communication_flow.md:13` 仍写 "The ParameterLoader associated with the vehicle object sends a `PARAM_REQUEST_LIST` …"，`docs/*/qgc-dev-guide/command_line_options.md:49-50` 仍列 `ParameterLoaderLog` 日志类别；
- 代码中的日志类别已是 `FactSystem.ParameterManager`（`src/FactSystem/ParameterManager.cc:33-36`）。

结论：任务书里"ParameterLoader"对应的现存角色就是 **`ParameterManager`**，它同时承担"全量加载器"与"参数仓库"两职；`docs/` 下的描述属未同步的开发文档。

### 3.2 `ParameterEditorController` 的职责边界

位置：`src/QmlControls/ParameterEditorController.h:126`（`class ParameterEditorController : public FactPanelController`）。配套的四个非 QObject 数据类同文件：`ParameterTableModel`（`:13`）、`ParameterEditorGroup`（`:61`）、`ParameterEditorCategory`（`:78`）、`ParameterEditorDiff`（`:95`）。

它**不持有参数存储**，只做视图模型：

- 构造即绑定车辆参数管理器：`_parameterMgr(_vehicle->parameterManager())`（`src/QmlControls/ParameterEditorController.cc:155`），并立即 `_buildLists()`（`:159`）、连接 `ParameterManager::factAdded → _factAdded`（`:171`）；
- `_buildListsForComponent(int compId)`（`:184`）遍历 `_parameterMgr->parameterNames(compId)` + `getParameter(compId, ...)`，按 `Fact::category()` → `_categories`、`Fact::group()` → `category->groups` 三层归并，最终 `group->facts.append(fact)`（`:214`）；
- `_buildLists()`（`:218`）保证自动驾驶仪组件在最前（`:228`）、`Standard` 分类置顶（`:231-238`）、`FactMetaData::kDefaultCategory` 置底（`:241-250`）、其余组件追加（`:253-257`）、默认组置底（`:260-272`）；
- `_factAdded(int compId, Fact* fact)`（`:275`）在增量到达时按名称有序插入分类/组/参数（`:293-302`、`:317-326`、`:331-337`），保证长列表也能边下边显示；
- 写操作只做两件事：`saveToFile()`（`:340`）转调 `ParameterManager::writeParametersToStream`（`:358`）；`buildDiffFromFile`（`:434`）+ `sendDiff`（`:400`）解析参数文件后用 `Fact::setRawValue()`（`:423`）走正常回写，或在车辆上不存在的参数上直接调 `ParameterManager::_mavlinkParamSet`（`:418`）——这正是 `src/FactSystem/ParameterManager.h:30` 声明 `friend class ParameterEditorController;` 的原因；
- `refresh()`（`:634`）→ `_parameterMgr->refreshAllParameters()`；`resetAllToDefaults()`（`:639`）与 `resetAllToVehicleConfiguration()`（`:645`）各自先调 ParameterManager 再刷新；
- 过滤/搜索在控制器侧完成：`_shouldShow(Fact*)`（`:651`，只读/仅显示已修改/仅收藏）与 `_performSearch()`（`:687`）。

QML 侧实例化：`src/QmlControls/ParameterEditor.qml:25-26`（`ParameterEditorController { id: controller }`）。

### 3.3 职责对照

| 关注点 | `ParameterManager` | `ParameterEditorController` |
|---|---|---|
| 参数存储 | 唯一存储（`_mapCompId2FactMap`） | 无，只持有 `Fact*` 引用 |
| 与 MAVLink 交互 | 全部（请求/接收/写入） | 无 |
| 元数据解析 | 触发点（调 `CompInfoParam::factMetaDataForName`） | 只读 `Fact` 暴露的 category/group/描述 |
| 分类/分组/排序 | 无 | 有（`_buildLists*`、`_factAdded`） |
| 搜索/收藏/差异文件 | 无（仅 `bulkRefresh` 与 `writeParametersToStream`） | 有 |
| 归属目录 | `src/FactSystem/` | `src/QmlControls/` |

---

## 4. Fact / FactGroup / FactMetaData：类层次与生命周期

### 4.1 类层次（本 HEAD 实测）

| 类 | 基类 | 定义位置 | 说明 |
|---|---|---|---|
| `Fact` | `QObject` | `src/FactSystem/Fact.h:16` | 单值载体 + 元数据 + 单位换算 + 回写信号；`QML_ELEMENT` |
| `SettingsFact` | `Fact` | `src/FactSystem/SettingsFact.h:11` | 把值持久化到 `QSettings`，多一个 `userVisible` 属性（`:19`） |
| `FactBitset` | `Fact` | `src/Vehicle/Actuators/Common.h:33` | 执行机构用的位集合视图 |
| `FactFloatAsBool` | `Fact` | `src/Vehicle/Actuators/Common.h:53` | 浮点当布尔用 |
| `FactGroup` | `QObject` | `src/FactSystem/FactGroup.h:15` | Fact / 子 FactGroup 的层级容器，可按固定周期批量发 `valueChanged` |
| `FactGroupWithId` | `FactGroup` | `src/FactSystem/FactGroupWithId.h:9` | 带 id 的组（电池、ESC 列表用） |
| `FactMetaData` | `QObject` | `src/FactSystem/FactMetaData.h:16` | 元数据：类型、min/max、单位、枚举、位掩码、分类、分组、只读、重启要求、易变值、换算函数 |
| `FactValueSliderListModel` | `QAbstractListModel` | `src/FactSystem/FactValueSliderListModel.h` | 由 `Fact::valueSliderModel()`（`src/FactSystem/Fact.cc:925`）惰性创建 |

`FactMetaData::ValueType_t` 共 14 个取值（`src/FactSystem/FactMetaData.h:24-39`），含 `valueTypeElapsedTimeInSeconds`（内部 double、显示 HH:MM:SS）与 `valueTypeCustom`（内部 `QByteArray`）。

参数子系统只创建 `Fact`：`ParameterManager` 中全部是 `new Fact(componentId, name, type, this)`（`src/FactSystem/ParameterManager.cc:250`、`:1626`、`:1890`），**不使用** `SettingsFact`。

### 4.2 一个参数变成 Fact 的完整过程

以"车辆开机后首个 `PARAM_VALUE`"为例，逐步可复现：

1. `Vehicle` 构造创建 `ParameterManager`（`src/Vehicle/Vehicle.cc:283`）；
2. 连接完成后 `InitialConnectStateMachine::_requestParameters`（`src/Vehicle/InitialConnectStateMachine.cc:387`）调 `vehicle()->_parameterManager->refreshAllParameters(MAV_COMP_ID_ALL)`（`:431`）；
3. `ParameterManager::refreshAllParameters`（`:611`）→ `_startParameterDownload`（`:646`）→ PX4 先 `_requestHashCheck`（`:672`），或 FTP 下载（`:689`），或 `PARAM_REQUEST_LIST`（`:718-725`）；
4. 车辆回 `PARAM_VALUE` → `MAVLinkProtocol::_updateStatus` 发 `messageReceived`（`src/Comms/MAVLinkProtocol.cc:294`）→ `Vehicle::_mavlinkMessageReceived`（`src/Vehicle/Vehicle.cc:525`）→ `_parameterManager->mavlinkMessageReceived(message)`（`:577`）；
5. `ParameterManager::mavlinkMessageReceived`（`:107`）解码，`_mavlinkParamUnionToVariant` 转 `QVariant`，调 `_handleParamValue`（`:134`）；
6. `_handleParamValue` 中：
   a. 丢弃抢跑的 `param_index == 65535` 流（`:152-155`）；
   b. 首次见到组件 → 建 `_paramCountMap` 与索引等待表（`:191-205`）；
   c. 从等待表摘除该索引并补发下一批（`:212-216`）；
   d. 重启/停止超时定时器（`:224-240`），更新进度（`:242`）；
   e. 建 `Fact`（若不存在）：`new Fact(componentId, parameterName, mavTypeToFactType(mavParamType), this)`（`:250`）；
   f. 取元数据并挂载：`fact->setMetaData(_vehicle->compInfoManager()->compInfoParam(componentId)->factMetaDataForName(parameterName, fact->type()))`（`:251-252`）——该调用内部走 §5 的元数据分支；
   g. 入库 `_mapCompId2FactMap[componentId][parameterName] = fact`（`:254`），连接回写信号（`:257`），`emit factAdded`（`:259`）；
   h. `fact->containerSetRawValue(parameterValue)`（`:262`）；
   i. PX4 且本轮读完后写本地缓存（`:267-272`）；
   j. `_checkInitialLoadComplete()`（`:276`）→ 默认组件读完则 `parametersReady = true`、`parametersReadyPreChecks()`、发信号（`:1434-1437`）。
7. `emit factAdded` 的两个消费者：`ParameterEditorController::_factAdded`（`src/QmlControls/ParameterEditorController.cc:171`、`:275`）增量插入参数表；其它订阅者按各自需要挂接。
8. QML 侧通过 `Fact` 的 `Q_PROPERTY`（`src/FactSystem/Fact.h:21-65`）读取/写入，控件在 `src/FactSystem/FactControls/`（如 `src/FactSystem/FactControls/FactTextField.qml`、`src/FactSystem/FactControls/FactComboBox.qml`、`src/FactSystem/FactControls/FactBitmask.qml`）。

`Fact` 构造本身会先挂一个空元数据占位，再被真实元数据覆盖：
`Fact::Fact(int componentId, const QString &name, ValueType_t type, QObject *parent)`（`src/FactSystem/Fact.cc:28`）→ `:36` `new FactMetaData(_type, this)` → `:37` `setMetaData(metaData)` → `:39` `_init()`（`:81`，只连接 `containerRawValueChanged → _checkForRebootMessaging`）。
`Fact::setMetaData(FactMetaData *metaData, bool setDefaultFromMetaData)`（`src/FactSystem/Fact.cc:741`）默认**不**用元数据默认值覆盖当前值，仅在 `setDefaultFromMetaData == true` 时调 `setRawValue(rawDefaultValue())`（`:744-746`），末尾发一次 `valueChanged(cookedValue())`（`:747`）。

### 4.3 值域与信号语义（回写方向的正确用法）

| 方法 | 语义 | 信号行为 |
|---|---|---|
| `Fact::setRawValue(const QVariant&)`（`src/FactSystem/Fact.cc:134`） | 用户/代码设值，经 `convertAndValidateRaw(convertOnly=true)` 只做类型转换（`:140`），值变化时发 `valueChanged`(cooked) + `containerRawValueChanged` + `rawValueChanged`（`:150-156`） | **会**触发回写车辆 |
| `Fact::forceSetRawValue(const QVariant&)`（`src/FactSystem/Fact.cc:111`） | 同 `setRawValue`，但**即使值相同也发信号**（`:126-127`），用于重发语义 | 会触发回写 |
| `Fact::containerSetRawValue(const QVariant&)`（`src/FactSystem/Fact.cc:198`） | **车辆来的值**，注释明确"This does NOT send a `_containerRawValueChanged` signal"（`src/FactSystem/Fact.h:179-180`） | 发 `valueChanged`(changed 时)、`rawValueChanged`；**恒发** `vehicleUpdated`（`:223`，用于等待应答/`forceSetRawValue` 配合） |
| `Fact::vehicleUpdated(const QVariant&)`（`src/FactSystem/Fact.h:199`） | "param write ack 已回来" | 由 `containerSetRawValue` 触发 |
| `Fact::containerRawValueChanged(const QVariant&)`（`src/FactSystem/Fact.h:202`） | 专供容器实现把变更送到车辆 | `src/FactSystem/ParameterManager.cc:257` 正是订阅它 → `_factRawValueUpdated`（`:536`）→ `_mavlinkParamSet`（`:544`） |

配套的 rate-limit 机制：`Fact::setSendValueChangedSignals(bool)`（`src/FactSystem/Fact.cc:825`）与 `Fact::_sendValueChangedSignal`（`:833`）——关闭时把变更延后，由 `Fact::sendDeferredValueChangedSignal()`（`:843`）补发；`FactGroup` 用固定周期定时器调用它（`src/FactSystem/FactGroup.cc:145-150`）。

重启提示：`Fact::_checkForRebootMessaging()`（`src/FactSystem/Fact.cc:934`）在值变化时检查 `vehicleRebootRequired()` / `qgcRebootRequired()` 并弹对应提示（`:939-942`）。

### 4.4 FactGroup 与参数系统的边界

`FactGroup` **不参与参数链路**，它面向遥测报文：

- 构造 `FactGroup(int updateRateMsecs, const QString &metaDataFile, ...)`（`src/FactSystem/FactGroup.cc:9`）用 `FactMetaData::createMapFromJsonFile(metaDataFile, this)` 一次性载入元数据（`:16`），这些 JSON 位于 `src/Vehicle/FactGroups/*.json`（如 `src/Vehicle/FactGroups/BatteryFact.json`、`src/Vehicle/FactGroups/GPSFact.json`）；
- `FactGroup::_addFact(Fact *fact, const QString &name)`（`:116`）按 `_updateRateMSecs == 0` 决定是否立即发信号（`:123`），并从 `_nameToFactMetaDataMap` 补元数据（`:124-126`）；
- 各具体组的取值入口是虚函数 `FactGroup::handleMessage(Vehicle*, const mavlink_message_t&)`（`src/FactSystem/FactGroup.h:49`），由 `Vehicle::_mavlinkMessageReceived` 遍历调用（`src/Vehicle/Vehicle.cc:588-590`）；
- 参数类 `Fact` 与遥测类 `Fact` 是同一 C++ 类型，但**元数据来源不同**：参数走 `CompInfoParam::factMetaDataForName`，FactGroup 走 `createMapFromJsonFile`/`createMapFromJsonArray`（`src/FactSystem/FactGroup.cc:16`、`:36`）。

---

## 5. 元数据来源分支

### 5.1 唯一决策点：`CompInfoParam::_resolveMetaData`

`src/Vehicle/ComponentInformation/CompInfoParam.cc`：

```
factMetaDataForName(name, valueType)                 // :77
  ├─ _nameToMetaDataMap 命中直接返回                  // :79-81
  └─ _resolveMetaData(name, valueType)               // :88
       ├─ _noJsonMetadata == true  → _getParameterMetaData()   // :90-94  → 固件内置元数据
       │                              → fwMeta->getMetaDataForFact(name, valueType)
       ├─ _noJsonMetadata == false → _lookupJsonMetaData(name) // :95-100 → 车辆下发的 JSON
       └─ 都没命中 → 通用兜底 FactMetaData(type)               // :103-111
                      group = 名字首个 '_' 之前；非 AUTOPILOT1 组件给 "Component N" 分类
```

- `_noJsonMetadata` 默认 `true`（`src/Vehicle/ComponentInformation/CompInfoParam.h:31`）；
- 只在 `CompInfoParam::setJson()` 成功解析后置 `false`（`src/Vehicle/ComponentInformation/CompInfoParam.cc:54`）。

因此分支判据就是"**该组件是否拿到了车辆下发的参数元数据 JSON**"。

### 5.2 车辆下发 JSON 路径（PX4 component information）

1. 时序上元数据请求**先于**参数请求：`InitialConnectStateMachine` 的连通流程为 `RequestAutopilotVersion → RequestStandardModes → RequestCompInfo → RequestParameters → …`（状态创建 `src/Vehicle/InitialConnectStateMachine.cc:49`、`:69`、`:78`、`:88`；连线 `:188-190`）；`InitialConnectStateMachine::_requestCompInfo`（`:370`）调 `ComponentInformationManager::requestAllComponentInformation`（`:384`）。
2. `ComponentInformationManager` 为 `MAV_COMP_ID_AUTOPILOT1` 预建四类 `CompInfo`，其中参数类是 `CompInfoParam`（`src/Vehicle/ComponentInformation/ComponentInformationManager.cc:27`）；请求顺序 `GENERAL → PARAMETER → EVENTS → ACTUATORS`（状态定义 `:60-93`、连线 `:108-123`），`_requestCompInfoParam`（`:210`）触发参数元数据。
3. `RequestMetaDataTypeStateMachine` 执行：
   - `_requestCompInfo`（`src/Vehicle/ComponentInformation/RequestMetaDataTypeStateMachine.cc:232`）请求 `MAVLINK_MSG_ID_COMPONENT_METADATA`（`:263`），成功则 `_compInfo->setUriMetaData(uri, file_crc)`（`:272`）；
   - 若拿不到 CRC，回退请求废弃的 `MAVLINK_MSG_ID_COMPONENT_INFORMATION`（`_requestCompInfoDeprecated`，`:279`；消息 id `:305`；`_handleCompInfoResult` 用 `general_metadata_uri` / `general_metadata_file_crc`，`:314`）；
   - `_requestMetaDataJson`（`:334`）按 URI 走 MAVLink FTP 或 HTTP（`_requestFile`，`:461`；FTP 判定 `_uriIsMAVLinkFTP`，`:619`），带 CRC 时优先用 `ComponentInformationCache` 缓存（`:483-494`）；
   - 下载后 `_completeRequest`（`:408`）调 `_compInfo->setJson(...)`（`:414` 或 `:417`）。
4. `CompInfoParam::setJson(const QString &metadataJsonFileName)`（`src/Vehicle/ComponentInformation/CompInfoParam.cc:20`）：JSON 解析（`:29`）→ schema 校验（`:35`）→ 版本必须为 1（`:49`）→ 置 `_noJsonMetadata = false`（`:54`）→ 逐条 `FactMetaData::createFromJsonObject(..., ParameterMetaData::kEmptyDefines, this)`（`:65`）；名字含 `{n}` 的存入 `_indexedNameMetaDataList` 并编译成正则（`:67-70`），其余进 `_nameToMetaDataMap`（`:72`）。
5. 取值时 `_lookupJsonMetaData`（`:114`）先用精确名（`_nameToMetaDataMap`，经 `factMetaDataForName` 的 `:79`），再按 `{n}` 正则实例化模板并替换描述中的索引（`:117-133`）。

ArduPilot 在该协议上的实际状态：`RequestMetaDataTypeStateMachine::_completeRequest` 明确写着 "ArduPilot doesn't support the component metadata protocol, so failure is expected"（`src/Vehicle/ComponentInformation/RequestMetaDataTypeStateMachine.cc:441-443`），失败只记 debug 级日志（`:443`）。

### 5.3 固件内置元数据路径（`_noJsonMetadata == true`）

调用链：`CompInfoParam::_getParameterMetaData()`（`src/Vehicle/ComponentInformation/CompInfoParam.cc:138`）——仅对 `MAV_COMP_ID_AUTOPILOT1` 惰性创建（`:140`）——`vehicle->firmwarePlugin()->loadParameterMetaData(vehicle)`（`:141`）。

- `FirmwarePlugin::loadParameterMetaData(const Vehicle*)`（`src/FirmwarePlugin/FirmwarePlugin.cc:523`）：先 `_createParameterMetaData()`（`:525`），为 nullptr 时告警并返回（`:526-533`，generic 固件走 debug 级）；再 `_cachedParameterMetaDataFile(vehicle)`（`:535`），非空则 `metaData->loadParameterFactMetaDataFile(metaDataFile)`（`:537`）。
- `FirmwarePlugin::_cachedParameterMetaDataFile`（`:542`）：内置文件版本优先/缓存版本择优——若内置文件版本未知或处于单测，直接用内置（`:557-560`）；否则在 `CacheLocation/ParameterMetaData` 下按 `ParameterFactMetaData_<autopilot>.<maj>.<min>.json` 找同主版本的最高次版本（`:564-579`），缓存版本更高才用缓存（`:586-592`）。
- 固件相关虚函数：
  - `FirmwarePlugin::_internalParameterMetaDataFile(const Vehicle*)` 默认返回空串（`src/FirmwarePlugin/FirmwarePlugin.h:417`）；
  - `FirmwarePlugin::_createParameterMetaData()` 默认返回 `nullptr`（`src/FirmwarePlugin/FirmwarePlugin.h:419`）；
  - **PX4**：`src/FirmwarePlugin/PX4/PX4FirmwarePlugin.h:57` 覆写 `_internalParameterMetaDataFile` 返回 `:/FirmwarePlugin/PX4/PX4ParameterFactMetaData.json`（资源打包见 `src/FirmwarePlugin/PX4/CMakeLists.txt:22-27`）；`PX4FirmwarePlugin::_createParameterMetaData()`（`src/FirmwarePlugin/PX4/PX4FirmwarePlugin.cc:271`）返回 `new PX4ParameterMetaData(this)`；
  - **ArduPilot**：`APMFirmwarePlugin::_createParameterMetaData()`（`src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:609`）返回 `new APMParameterMetaData(this)`；`APMFirmwarePlugin::_internalParameterMetaDataFile(const Vehicle*)`（`:710`）按车型与固件版本向上回溯找 `:/FirmwarePlugin/APM/APMParameterFactMetaData.<Vehicle>.<Major>.<Minor>.json`（`:720-734`），找不到则从 `4.0` 起找最老的可用文件（`:736-742`）；这些资源由 CMake 从 ArduPilot 参数目录的 `apm.pdef.json` 生成别名（`src/FirmwarePlugin/APM/CMakeLists.txt:42`、`:69`、`:99`）。
- 解析与取值基类：`ParameterMetaData::loadParameterFactMetaDataFile`（`src/FirmwarePlugin/ParameterMetaData.cc:26`）→ `parseParameterJson`（虚函数，声明于 `src/FirmwarePlugin/ParameterMetaData.h`，调用点 `src/FirmwarePlugin/ParameterMetaData.cc:52`）；取值 `ParameterMetaData::getMetaDataForFact`（`:55`）走 `_lookupMetaData`（`:67`）→ 未命中 `_createDefaultMetaData`（`:69`）→ `_postProcessMetaData`（`:72`）→ `_cachedMetaData`（`:73`）。
  - `PX4ParameterMetaData::parseParameterJson`（`src/FirmwarePlugin/PX4/PX4ParameterMetaData.cc:22`）读顶层 `version`（需 ≥1）与 `parameters` 数组，逐条 `FactMetaData::createFromJsonObject`（`:48`）；`_postProcessMetaData`（`:59`）把空分类/默认分类改成 `Standard`（`:63-65`）、`volatileValue()` 参数强制只读（`:67-69`）、把描述里的换行压成空格（`:71-78`）。
  - `APMParameterMetaData::parseParameterJson`（`src/FirmwarePlugin/APM/APMParameterMetaData.cc:31`）按"分组 → 参数"两层读，组名取自参数名前缀并去掉尾部数字（`_groupFromParameterName`，`:24-29`），随后 `_correctGroupMemberships`（`:59`）把"只有 1 个成员"的组降级为默认组；`_lookupMetaData`（`:73`）映射 APM 字段：`DisplayName`→短描述（`:87-90`）、`Description`→长描述（`:92-95`）、`Units`→单位（`:97-100`）、`User`→分类（`:102-105`）、`ReadOnly`（`:107-109`）、`RebootRequired`（`:110-112`）、`Increment`（`:114-121`）、`Range.low/high`→min/max 与 userMin/userMax（`:123-135`）、`Values`→枚举（`:137-140`，`_applyEnumValues` 还处理 int8 的 128..255 无符号重解释，`:178-197`）、`Bitmask`→位掩码（`:142-145`）；`_createDefaultMetaData`（`:204`）把未收录参数标为 `Advanced`；`_postProcessMetaData`（`:212`）把 `*_P/_I/_D` 浮点参数小数位设为 6。

### 5.4 对任务书"ArduPilot 元数据随参数一起推送"的核实结论

在本 HEAD 的 QGC 代码里，**没有**任何路径从车辆获取 ArduPilot 的参数元数据：

- 全仓 grep `pdef` 只命中 `src/FirmwarePlugin/APM/CMakeLists.txt:67-101`（编译期把 `apm.pdef.json` 打包为资源）与 `src/AnalyzeView/LogViewer/LogViewerParamMetaData.cc:26`（日志查看器复用同一套内置文件），没有任何运行时下载/接收 `pdef` 的代码；
- `PARAM_VALUE` 报文本身只带 `param_id / param_value / param_type / param_count / param_index`，`ParameterManager::mavlinkMessageReceived`（`:107-136`）也只解出这些字段，未携带任何描述/单位/范围信息；
- ArduPilot 走的确实包含"随参数一起下来的**默认值**"：`_tryftp` 对 APM 为真（`src/FactSystem/ParameterManager.cc:44`），FTP 请求 URI 是 `@PARAM/param.pck?withdefaults=1`（`:690`），`_parseParamFile` 解析出 `defaultValue` 后调 `fact->metaData()->setRawDefaultValueFirmwareForce(defaultValue)`（`:1885`、`:1901`），该函数绕过 min/max 校验直接置默认值（`src/FactSystem/FactMetaData.cc:165-169`，对比受校验的 `setRawDefaultValue`，`:151-163`）。

因此准确表述是：**ArduPilot 的元数据来自 QGC 内置的 `apm.pdef.json` 资源；"随参数一起"下发的只是参数默认值（`param.pck?withdefaults=1`），不是元数据。** 该点与任务书原始假设不符，此处以源码为准。

### 5.5 通用兜底

两条分支都没命中时，`CompInfoParam::_resolveMetaData` 的最后一段（`src/Vehicle/ComponentInformation/CompInfoParam.cc:103-111`）新建裸 `FactMetaData(valueType, this)`：`group` 取名字首个 `_` 之前（`:104-107`），非 `AUTOPILOT1` 组件给分类 `Component N`（`:108-110`）。此时 `Fact` 有类型和值，但无描述/单位/范围——这是"元数据缺失"在 UI 上的直接后果。

### 5.6 固件相关类 / 函数清单

| 固件相关点 | 位置 | PX4 | ArduPilot | Generic |
|---|---|---|---|---|
| `_createParameterMetaData()` | `src/FirmwarePlugin/FirmwarePlugin.h:419` | `src/FirmwarePlugin/PX4/PX4FirmwarePlugin.cc:271` | `src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:609` | 基类返回 nullptr |
| `_internalParameterMetaDataFile()` | `src/FirmwarePlugin/FirmwarePlugin.h:417` | `src/FirmwarePlugin/PX4/PX4FirmwarePlugin.h:57` | `src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:710` | 基类空串 |
| 元数据解析器子类 | `ParameterMetaData` | `src/FirmwarePlugin/PX4/PX4ParameterMetaData.cc:22` | `src/FirmwarePlugin/APM/APMParameterMetaData.cc:31` | 无 |
| 是否走 component information | — | 支持（`src/Vehicle/ComponentInformation/RequestMetaDataTypeStateMachine.cc`） | 不支持（注释证据 `:441-443`） | 不支持 |
| 参数缓存是否落盘 | `src/FactSystem/ParameterManager.cc:267` | 是 | 否 | 否 |
| 参数 FTP 文件下载 | `src/FactSystem/ParameterManager.cc:44`、`:673` | 是（且 `_HASH_CHECK` 优先） | 是（默认值来源） | 否 |
| 参数名版本重映射 | `src/FirmwarePlugin/FirmwarePlugin.h:317` | 有映射表 | 有映射表 | 基类空实现 |
| 离线编辑参数文件 | `src/FirmwarePlugin/FirmwarePlugin.h:331`、`src/FactSystem/ParameterManager.cc:1572` | `V1.4.OfflineEditing.params` | `Copter/Plane/Rover/Sub.OfflineEditing.params` | 空 |

---

## 6. ParameterManager 与 Vehicle 的通信接口

### 6.1 方向总表

| 方向 | 形式 | 具体项 | 位置 |
|---|---|---|---|
| Vehicle → PM | **直接方法调用** | `_parameterManager->mavlinkMessageReceived(message)`（每条 MAVLink 报文，`PARAM_VALUE` 之外的在 PM 内被忽略） | `src/Vehicle/Vehicle.cc:577` |
| Vehicle → PM | 构造 | `new ParameterManager(this)` | `src/Vehicle/Vehicle.cc:283` |
| Vehicle → PM | Qt 信号 | `MultiVehicleManager::parameterReadyVehicleAvailableChanged → Vehicle::_vehicleParamLoaded` | `src/Vehicle/Vehicle.cc:135` |
| PM → Vehicle | **直接方法调用** | `_vehicle->sendMessageOnLinkThreadSafe(sharedLink.get(), msg)`（所有出向报文） | `src/FactSystem/ParameterManager.cc:725`、`:988`、`:1009`、`:1245`；实现 `src/Vehicle/Vehicle.cc:1392`（内部 `:1400` `adjustOutgoingMavlinkMessageThreadSafe`、`:1403` `link->sendMessageThreadSafe`） |
| PM → Vehicle | 直接方法调用 | `_vehicle->autopilotPlugin()->parametersReadyPreChecks()` | `src/FactSystem/ParameterManager.cc:1435`、`:1509` |
| PM → Vehicle | 直接方法调用 | `_vehicle->compInfoManager()->compInfoParam(componentId)->factMetaDataForName(...)` | `src/FactSystem/ParameterManager.cc:251`、`:1200`、`:1628`、`:1891` |
| PM → Vehicle | 直接方法调用 | `_vehicle->ftpManager()` 下载 `@PARAM/param.pck` 与连接 `downloadComplete` / `commandProgress` | `src/FactSystem/ParameterManager.cc:683-695`、`:553-554` |
| PM → Vehicle | 直接方法调用 | `_vehicle->vehicleLinkManager()->primaryLink()`（取主链路与 `mavlinkChannel`）、`()->isLogReplay()` | `src/FactSystem/ParameterManager.cc:41`、`:624`、`:648`、`:967`、`:993`、`:1226` |
| PM → Vehicle | 直接方法调用 | `_vehicle->firmwarePlugin()->...`：`offlineEditingParamFile`（`:1572`）、`paramNameRemapMajorVersionMap`（`:1546`）、`remapParamNameHigestMinorVersionNumber`（`:1556`） | 同左 |
| PM → Vehicle | 只读查询 | `_vehicle->id()`、`defaultComponentId()`、`px4Firmware()`、`apmFirmware()`、`genericFirmware()`、`isOfflineEditingVehicle()`、`firmwareMajorVersion()` / `firmwareMinorVersion()`、`setOfflineEditingDefaultComponentId()` | 散布于 `src/FactSystem/ParameterManager.cc` 全文，声明见 `src/Vehicle/Vehicle.h:501-503`、`:682`、`:685` |
| PM → Vehicle | **Qt 信号**（Vehicle 订阅） | `parametersReadyChanged → Vehicle::_parametersReady` | 连接 `src/Vehicle/Vehicle.cc:284`；槽 `:1668`（`:1676-1679` 完成后断开并 `_setupAutoDisarmSignalling`） |
| PM → Vehicle | Qt 信号 | `parametersReadyChanged` 再触发 `emit hasGripperChanged()` | `src/Vehicle/Vehicle.cc:285-287` |
| PM → Vehicle | Qt 信号 | `loadProgressChanged → Vehicle::_gotProgressUpdate` | `src/Vehicle/Vehicle.cc:290` |
| PM → MVM | Qt 信号 | `parametersReadyChanged → MultiVehicleManager::_vehicleParametersReadyChanged` | 连接 `src/Vehicle/MultiVehicleManager.cc:120`；槽 `:255`；对外 `_setParameterReadyVehicleAvailable`（`:365-369`，信号 `parameterReadyVehicleAvailableChanged`） |
| PM → InitialConnect | Qt 信号 | `parametersReadyChanged` / `initialParametersRequestFailed` / `cacheCheckOnlyFailed` / `loadProgressChanged` | `src/Vehicle/InitialConnectStateMachine.cc:395`、`:403`、`:407`、`:413` |
| PM → 状态机 | Qt 信号 | 内部信号 `_paramSetSuccess/_paramSetFailure/_paramRequestReadSuccess/_paramRequestReadFailure`（供 `BulkRefreshJob` 与单测消费） | 声明 `src/FactSystem/ParameterManager.h:138-141`；消费 `src/FactSystem/BulkRefreshJob.cc:22-23` |
| PM → 编辑器 | Qt 信号 | `factAdded(int componentId, Fact*)` | 声明 `src/FactSystem/ParameterManager.h:134`；消费 `src/QmlControls/ParameterEditorController.cc:171` |
| Vehicle → PM | 直接方法调用（读取） | `parameterExists(...)` / `getParameter(...)`（如预解锁检查、gripper、速度限制） | `src/Vehicle/Vehicle.cc:1069-1070`、`:2594-2596`、`:2605-2607` |
| 外部 → PM | 直接方法调用 | `refreshAllParameters`（`src/Vehicle/InitialConnectStateMachine.cc:431`）、`bulkRefresh`、`refreshParameter`、`tryHashCheckCacheLoad`（`:429`）、`resetAllParametersToDefaults`、`resetAllToVehicleConfiguration`、`setParameterDownloadSkipped`（`:105`、`:398`） | 见 §2 |

### 6.2 对外信号清单（`src/FactSystem/ParameterManager.h:126-141`）

| 信号 | 触发点 |
|---|---|
| `parametersReadyChanged(bool)` | `src/FactSystem/ParameterManager.cc:659`、`:1436`、`:1510` |
| `missingParametersChanged(bool)` | `:660`、`:1437`、`:1511` |
| `loadProgressChanged(float)` | `:1670`（由 `_setLoadProgress` 统一发） |
| `cacheCheckOnlyFailed()` | `:626`、`:632`、`:642`、`:1179`、`:1281`、`:1494` |
| `initialParametersRequestFailed()` | `:1528` |
| `pendingWritesChanged(bool)` | `:1932`、`:1945` |
| `parameterDownloadSkippedChanged()` | `:1678` |
| `factAdded(int, Fact*)` | `:259`、`:1904` |
| `_paramSetSuccess/_paramSetFailure` | `:430`、`:434` |
| `_paramRequestReadSuccess/_paramRequestReadFailure` | `:1098`、`:1102` |

### 6.3 QML 暴露

- `ParameterManager` 是 `QML_ELEMENT` + `QML_UNCREATABLE`（`src/FactSystem/ParameterManager.h:23-24`），并把 `parametersReady` / `missingParameters` / `loadProgress` / `pendingWrites` / `parameterDownloadSkipped` 暴露为只读属性（`:25-29`），`refreshAllParameters()` 为 `Q_INVOKABLE`（`:54`）；
- `Vehicle.parameterManager` 是 QML 常量属性（`src/Vehicle/Vehicle.h:229`，访问器 `:577-578`），QML 通过 `vehicle.parameterManager` 取用。

---

## 7. 沿源码复现要点（供交叉验证）

1. 找入口：从 `src/Comms/MAVLinkProtocol.cc:102` 读到 `:294`，确认 `messageReceived` 是唯一广播点；
2. 找订阅：`grep "MAVLinkProtocol::messageReceived" src/` → `src/Vehicle/Vehicle.cc:123`；
3. 找参数分支：`src/Vehicle/Vehicle.cc:577` → `src/FactSystem/ParameterManager.cc:107`；
4. 找建 Fact：`src/FactSystem/ParameterManager.cc:250-262`；
5. 找元数据：`src/FactSystem/ParameterManager.cc:251` → `src/Vehicle/ComponentInformation/CompInfoParam.cc:77` → `:88`（分支）→ `:141`（固件内置）或 `:114`（车辆 JSON）；
6. 找写回：`Fact::setRawValue`（`src/FactSystem/Fact.cc:134`）→ `containerRawValueChanged`（`:154`）→ `ParameterManager::_factRawValueUpdated`（`:536`）→ `_mavlinkParamSet`（`:306`）；
7. 找 UI 消费：`src/QmlControls/ParameterEditorController.cc:171/275` 与 `src/FactSystem/FactControls/`；
8. 交叉核对固件侧：`src/FirmwarePlugin/PX4/PX4FirmwarePlugin.h:57`、`src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:710`。

---

## 8. 边界与未验证声明

1. **无法编译验证**：本机无 Qt6 / MSVC，本文全部结论来自源码静态阅读，未做编译、未跑单测；状态机实际跳转顺序以代码中的 `addTransition` 声明为准，未做运行期验证。
2. **`ParameterLoader` 的删除时点未确认**：仓库是 `depth=1` 浅克隆，无历史，无法给出该类被移除的提交。本文只能确认当前 HEAD 的 `src/` 下不存在该类。
3. **`_loadMetaData` / `_clearMetaData` / `_tryCacheLookup` 是死声明**：三者声明于 `src/FactSystem/ParameterManager.h:165`、`:166`、`:152`，但全仓 `*.cc` 无定义、无调用点（grep 结果仅命中头文件声明行）。这是"元数据装载职责已从 ParameterManager 迁到 `CompInfoParam`"的残留痕迹，本文按现状记录。
4. **`src/MAVLink/` 目录不含协议收发实现**：任务书把 `src/MAVLink/` 列为关键目录，实际 `MAVLinkProtocol` 位于 `src/Comms/`；`src/MAVLink/` 在本 HEAD 只提供 mavlink 头聚合、消息类型/枚举、FTP 客户端、图像协议、状态文本、签名与事件库。
5. **工具链依赖未验证**：顶层无 `libs/`，mavlink C 库头（`#include <mavlink.h>`，`src/MAVLink/MAVLinkLib.h:24`）与 `MAVLinkEnums.h` 均需构建期生成，`ArduPilotParams_SOURCE_DIR`（`src/FirmwarePlugin/APM/CMakeLists.txt:42`）指向的 ArduPilot 参数仓内容亦未在本机获取；因此"内置 APM 元数据覆盖哪些车型/版本"只能从 `_internalParameterMetaDataFile` 的查找逻辑（`src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:710-742`）推断文件名规则，未能核对实际文件清单。
6. **64 位参数类型的实际可达性未验证**：`ParameterManager::_mavlinkParamUnionToVariant`（`src/FactSystem/ParameterManager.cc:506-534`）不支持 `REAL64/UINT64/INT64`；是否会有固件真的发送这些 `param_type`，需实机或日志验证，本文只陈述代码事实。
