# ArduPilot(APM) 参数系统差异与 QGC 双固件适配

> 分析对象：QGroundControl 二次开发项目 NGC-Nexus。
> QGC 源码：`E:\04-Workspace_workbudy\15_NGC-Nexus\repo`，HEAD `25185047e855937d694a0174c1541d8f7d794a74`（master，浅克隆，无子模块，无 `libs/`）。
> 本机**没有** ArduPilot 源码，APM 侧结论一律联网核实并给出 URL；无法核实的一律标注"未验证"。
> 本机无 Qt6/MSVC，无法编译，本文为源码阅读级分析。
> 引用纪律：QGC 侧给「文件:行号」；APM 侧给 URL；两者都给不出的一律写"未验证"。

## 0. 结论摘要

1. **APM 的参数元数据不随 MAVLink 参数报文推送。** ArduPilot 的参数元数据在**编译期/发布期**由 `Tools/autotest/param_metadata/param_parse.py` 从源码注释生成 `apm.pdef.json`，发布在独立仓库 `ArduPilot/ParameterRepository`；QGC 在 **CMake 配置阶段**用 CPM 拉取该仓库并把每个 `apm.pdef.json` 编译进 Qt 资源。
2. **"元数据随 `PARAM_VALUE` 推送"的说法不成立**，`PARAM_VALUE` 报文本身只有 `param_id / param_value / param_type / param_count / param_index`（`src/FactSystem/ParameterManager.cc:107-136`）。
3. **"元数据随 `PARAM_EXT_VALUE` 推送"的说法同样不成立**：`PARAM_EXT_*` 是 MAVLink 官方文档明确说明"为相机协议发明、飞行栈普遍不支持"的协议；QGC 侧 `PARAM_EXT_*` 只出现在相机子系统；ArduPilot 4.7 自带的 MAVLink 支持表把 5 条 `PARAM_EXT_*` 全部标为 `UNSUPPORTED`。
4. **随参数一起下来的只有默认值**：APM 走 MAVLink FTP 下载 `@PARAM/param.pck?withdefaults=1`（`src/FactSystem/ParameterManager.cc:44`、`:673`、`:690`），解析后写入 `FactMetaData` 的默认值（`:1885`、`:1901`）。
5. **APM 不支持 MAVLink Component Metadata Protocol**，因此 QGC 的 `COMP_METADATA_TYPE_PARAMETER` 请求对 APM 必然失败，`CompInfoParam::_resolveMetaData` 落入"固件内置元数据"分支（`src/Vehicle/ComponentInformation/CompInfoParam.cc:88-100`），QGC 自己也知道这一点（`src/Vehicle/ComponentInformation/RequestMetaDataTypeStateMachine.cc:441-443`）。
6. **APM 与 PX4 的元数据 JSON 结构完全不同**，由 `ParameterMetaData` 抽象基类的两个子类分别解析：`APMParameterMetaData`（`src/FirmwarePlugin/APM/APMParameterMetaData.cc`）与 `PX4ParameterMetaData`（`src/FirmwarePlugin/PX4/PX4ParameterMetaData.cc`）。
7. **QGC 仓库里没有任何 APM 参数元数据文件本体**：`src/FirmwarePlugin/APM/` 下只有 6 个 `APM-MavCmdInfo*.json`（任务命令元数据）与 4 个 `*.OfflineEditing.params`（离线编辑用的参数值快照），真正的 `APMParameterFactMetaData.<Vehicle>.<M.m>.json` 全部来自 CPM 外部依赖。

---

## 1. APM 侧参数链路（联网核实）

### 1.1 参数定义在哪里

ArduPilot 的参数定义**不集中在某个参数表文件里**，而是以 C++ 静态对象声明的方式散落在各库源码中。参数名、默认值、取值范围、单位、枚举含义等以 `// @Param:` / `// @Values:` / `// @Range:` 等注释块写在声明旁边。核实证据链：

- `AP_MAX_NAME_SIZE` 定义为 `16`：<https://github.com/ArduPilot/ardupilot/blob/master/libraries/AP_Param/AP_Param.h>（本机下载副本第 37 行 `#define AP_MAX_NAME_SIZE 16`）。
- `AP_Param` 的类型枚举含 `AP_PARAM_NONE / INT8 / INT16 / INT32 / FLOAT / VECTOR3F / GROUP`，QGC 侧 `ParameterManager::_parseParamFile` 逐字复制了同一套枚举（`src/FactSystem/ParameterManager.cc:1715-1723`），可用作两侧类型系统一致的旁证。

### 1.2 元数据怎么产生（编译期/发布期，不是运行期）

生成脚本：`Tools/autotest/param_metadata/param_parse.py`（<https://github.com/ArduPilot/ardupilot/blob/master/Tools/autotest/param_metadata/param_parse.py>，本机下载副本 30757 字节）。

关键事实（均来自该脚本与其调用方）：

- 该脚本通过 `--format` 选择发射器，`json` 对应 `JSONEmit`（下载副本第 22-24 行 `from jsonemit import JSONEmit`、第 729-730 行 `'json': JSONEmit, 'xml': XmlEmit`）。
- `JSONEmit` 的输出文件名硬编码为 `apm.pdef.json`：<https://github.com/ArduPilot/ardupilot/blob/master/Tools/autotest/param_metadata/jsonemit.py>（本机下载副本 `def output_fname(self): return 'apm.pdef.json'`）。
- 该发射器的 JSON 根结构是 `{"json": {"version": 0}}`，随后逐"参数组"写入：`self.content = {"json": {"version": 0}}`，`json.dump(self.content, self.f, indent=2, sort_keys=True)`（同上文件 `__init__` 与 `close`）。
- 发布流程在 `ArduPilot/ParameterRepository` 的 `scripts/run_parsers.py`：对每个发版 tag checkout 后执行 `param_parse.py --vehicle <Vehicle>`，再把工作目录下的 `apm.pdef.*` 拷贝到 `{Vehicle}-{Major}.{Minor}/`。URL：<https://github.com/ArduPilot/ParameterRepository/blob/main/scripts/run_parsers.py>（关键片段：`subprocess.run([f'{self.repository_path}/Tools/autotest/param_metadata/param_parse.py', '--vehicle', vehicle_type], ...)`、`for data in glob.glob(f'{self.repository_path}/apm.pdef.*'): shutil.copy2(data, dest)`）。
- 仓库自述：`ArduPilot/ParameterRepository` 是"生成的参数集合 + 每版本 MAVLink 消息文档"，生成逻辑在 `scripts/`，其余目录都是生成产物。URL：<https://github.com/ArduPilot/ParameterRepository/blob/main/README.md>。

### 1.3 `apm.pdef.json` 的实际字段（实测，非记忆）

本机通过 GitHub blobs API 拉取 `Copter-4.7/apm.pdef.json`（2229014 字节，blob sha `fed3622e6b174c2434ab2addea24bf439d0e6432`）后实测：

- 顶层是「参数组 → 参数名 → 字段」两层对象，另有 `json` 根键；实测顶层组数 **387**，其中包含 `json`、`Copter`、`SERVO1_`…`SERVO32_`、`MAV1`…`MAV32`、`SIM_*` 等。
- 全量扫描后，参数对象层出现过的字段名只有 12 个：`Bitmask`、`Calibration`、`Description`、`DisplayName`、`Increment`、`Range`、`ReadOnly`、`RebootRequired`、`Units`、`User`、`Values`、`Volatile`。
- 根键实测值：`"json": {"version": 0}`。
- 字段抽样（均来自同一文件实测）：

| 字段 | 实测样例参数 | 实测值（节选） |
|---|---|---|
| `Values`（枚举） | `ADSB_EMIT_TYPE` | `{"-1": ...}` 形式不存在；实测为 `{"0":"NoInfo","1":"Light",...,"14":"UAV","15":"Space",...}`，键为**字符串**数字 |
| `Bitmask` | `ADSB_OPTIONS` | `{"0":"Ping200X Send GPS","1":"Squawk 7400 on RC failsafe",...}`，键为**位序号**字符串 |
| `Range` | `ATC_ANG_PIT_P` | `{"low":"3.000","high":"12.000"}`，值为**字符串** |
| `Increment` | `ATC_ANG_PIT_P` | `"0.01"`，字符串 |
| `ReadOnly` | `ARSPD2_DEVID` | `"True"`，**字符串**（不是 JSON bool） |
| `RebootRequired` | `SERVO1_FUNCTION` | `"True"`，字符串 |
| `Volatile` | `BARO1_GND_PRESS` | `"True"`，字符串（该文件内 `Volatile` 出现于遥测类参数） |
| `Calibration` | `COMPASS_DIA2_X` | `"1"`，字符串 |
| `Units` / `User` / `DisplayName` / `Description` | `ATC_RAT_PIT_FLTD` 等 | `"Hz"` / `"Standard"` 或 `"Advanced"` / 短名 / 长描述 |

> 注意：`jsonemit.py` 里对 `displayName` / `description` / `user` 走的是小写键赋值，但**实测发布产物用的是大写键** `DisplayName`/`Description`/`User`。原因是这些大写键直接来自参数对象 `__dict__`（源码注释里的大写标签），发射器把小写赋值与小写键写进了同一对象后被大写键覆盖/并存。**本文以实测产物为准**：QGC 侧解析的就是大写键。

### 1.4 元数据怎么通过 MAVLink 传给地面站 —— 核实结论

**结论：不传。APM 参数元数据是纯离线文件，MAVLink 上只传参数值。**

三条互不依赖的证据：

**(a) MAVLink 官方文档：`PARAM_EXT_*` 不是给飞控参数用的。**
<https://mavlink.io/en/services/parameter_ext.html> 原文：扩展参数协议"invented for the Camera Protocol"，"at time of writing the protocol is supported by QGroundControl for this purpose, but is not otherwise supported by flight stacks"。即 `PARAM_EXT_VALUE` 不携带飞行器参数元数据。

**(b) ArduPilot 自己的 MAVLink 支持表：`PARAM_EXT_*` 全部 UNSUPPORTED。**
ArduPilot 为 Copter-4.7 发布的自动生成文档 `MAVLinkMessages.rst`（<https://github.com/ArduPilot/ParameterRepository/blob/main/Copter-4.7/MAVLinkMessages.rst>，本机下载副本 124976 字节）中：

```
#324, PARAM_EXT_ACK,            UNSUPPORTED, common
#321, PARAM_EXT_REQUEST_LIST,   UNSUPPORTED, common
#320, PARAM_EXT_REQUEST_READ,   UNSUPPORTED, common
#323, PARAM_EXT_SET,            UNSUPPORTED, common
#322, PARAM_EXT_VALUE,          UNSUPPORTED, common
```

(id 与"UNSUPPORTED"标记为原文，本机副本第 612-616 行。)

**(c) ArduPilot 不支持 Component Metadata Protocol，也没有参数元数据的 MAVLink 通道。**
同一份 `MAVLinkMessages.rst` 中检索 `COMPONENT_INFORMATION` / `COMPONENT_METADATA` / `COMP_METADATA` **零命中**；而 `PARAM_REQUEST_LIST` / `PARAM_REQUEST_READ` / `PARAM_SET` / `PARAM_VALUE` 四条全部命中并标注实现文件 `GCS_MAVLink/GCS_Common.cpp`、`GCS_MAVLink/GCS_Param.cpp`（同文件第 90-93、322、435 行）。也就是说 ArduPilot 侧参数链路的 MAVLink 面只有**值**，没有**元数据**。

**(d) 版本差异：有人在做，但截至核实时刻尚未落地。**
ArduPilot/ardupilot PR **#32599** "Tools: add MAVLink COMP_METADATA_TYPE_PARAMETER JSON emitter" 试图给 `param_parse.py` 加 `--format mavlink_compinfo`，输出符合 MAVLink `COMP_METADATA_TYPE_PARAMETER` schema 的 `compinfo-parameter.json`（PR 描述原文：`type` 字段先硬编码为 `"Float"`、`default` 暂不输出，属 "phase 1"）。**该 PR 状态为 `closed`、`merged: false`、`merged_at: null`**（本机 `gh api repos/ArduPilot/ardupilot/pulls/32599` 实测）。因此该能力**不能认为已经存在**。
同源的另一条线索：ArduPilot/ardupilot PR #31355 "Create versioned parameters.* apm.pdef.* and LogMessages.* files during build, including links in manifest.json"（<https://github.com/ArduPilot/ardupilot/pull/31355>），与"构建期生成版本化 pdef"有关，同样属工具链改进而非 MAVLink 传输。**其合并状态未验证。**

**(e) ArduPilot 侧实际传的参数字节流。**
- `GCS_MAVLINK::send_parameter_value(const char *param_name, ap_var_type param_type, float param_value)` —— 函数签名本身就说明"值走 float 通道 + 类型字段单独给"，见 <https://github.com/ArduPilot/ardupilot/blob/master/libraries/GCS_MAVLink/GCS_Param.cpp>（本机下载副本第 332-341 行，`mavlink_msg_param_value_send(..., param_value, mav_param_type(param_type), ...)`；另有 `GCS::send_parameter_value` 在 `:349-366`）。
- QGC 侧对应的"APM 把任何类型都塞进 float 字段"修正逻辑：`src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:129-189`（入向 `_handleIncomingParamValue`，注释原文 "APM stack passes all parameter values in mavlink_param_union_t.param_float no matter what type they are. Fix that up to correct usage."）与 `:191-245`（出向 `_handleOutgoingParamSetThreadSafe`，注释原文 "Fix it back to the wrong way on the way out."）。两处互证。
- 参数**文件**（含默认值）通道：MAVLink FTP 路径 `@PARAM/param.pck`。QGC 侧见 `src/FactSystem/ParameterManager.cc:690`；QGC 自带 MockLink 也实现了该路径与两种 magic（`src/Comms/MockLink/MockLinkFTP.cc:163-193`、`:776-787`，magic `0x671B`/`0x671C` 与 `ParameterManager.cc:1711-1712` 一致）。

### 1.5 APM 参数名长度边界

`AP_MAX_NAME_SIZE` = 16（`AP_Param.h:37`），与 MAVLink `PARAM_VALUE.param_id` 的 16 字符上限一致；QGC 侧同样按 16 字节做 `strncpy` 并补 `\0`（`src/FactSystem/ParameterManager.cc:114-116`）。
但 PR #32599 描述提到 "4 ArduPilot params that exceed the MAVLink 16-char spec limit"。**这 4 个具体参数名及它们如何上线传输，未验证。**

---

## 2. QGC 侧 APM 专用实现

### 2.1 `src/FirmwarePlugin/APM/`（30 个文件）

| 文件 | 作用 | 关键行 |
|---|---|---|
| `APMFirmwarePluginFactory.cc` | 按 `MAV_AUTOPILOT_ARDUPILOTMEGA` + `MAV_TYPE` 分发到 Copter/Plane/Rover/Sub 四个子插件；未覆盖的车型返回 `nullptr`（落 Generic 插件） | `:22-27`、`:29-72` |
| `APMFirmwarePlugin.cc/.h` | APM 通用行为：MAVLink 质量修正、流速率、能力位、任务命令表、内置元数据文件定位 | 见下 |
| `ArduCopterFirmwarePlugin.cc/.h` | 飞行模式枚举 `APMCopterMode`、参数名版本重映射表、`offlineEditingParamFile` | `:71-165`、`.h:49-53` |
| `ArduPlaneFirmwarePlugin.cc/.h` | Plane 版重映射（含 `4.5`、`4.7` 两档） | `:71-143` |
| `ArduRoverFirmwarePlugin.cc/.h` | Rover 版重映射（`4.7`） | `:49-59` |
| `ArduSubFirmwarePlugin.cc/.h` | Sub 版重映射（`4.7`） | `:129-191` |
| `APMParameterMetaData.cc/.h` | **APM 元数据 JSON 解析器**（`ParameterMetaData` 子类） | 全文件 |
| `APM-MavCmdInfo{Common,FixedWing,MultiRotor,Rover,Sub,VTOL}.json` | 任务命令（`MAV_CMD`）元数据，**不是参数元数据** | `APMFirmwarePlugin.cc:588-607` |
| `Copter/Plane/Rover/Sub.OfflineEditing.params` | 离线编辑用的参数值快照（QGC 格式文本） | `ArduCopterFirmwarePlugin.h:53` 等 |
| `APM*.qml`、`CMakeLists.txt` | 状态指示器与构建 | `CMakeLists.txt:28-37`、`:111-114` |

`APMFirmwarePlugin.cc` 中与参数系统直接相关的关键点：

- `_handleIncomingParamValue`（`:129-189`）：入向 `PARAM_VALUE` 的"假 float → 真类型"重解释，并用 MAVLink 1.0 通道重编码。
- `adjustIncomingMavlinkMessage`（`:292-330`）：只对"确认属于 ArduPilot 栈"的组件做重解释（`_ardupilotComponentMap`，由 HEARTBEAT 的 `autopilot == MAV_AUTOPILOT_ARDUPILOTMEGA` 填充，`:286`；ESP8266 组件被强制置 false，`:289`）。
- `_handleOutgoingParamSetThreadSafe`（`:191-245`）：出向 `PARAM_SET` 反向修正。
- `_internalParameterMetaDataFile`（`:710-746`）：由车类型 + 固件版本定位 `:/FirmwarePlugin/APM/APMParameterFactMetaData.<Vehicle>.<Major>.<Minor>.json`；从当前版本向下逐 minor 回溯（minor 到 0 时 `currMajor--`、`currMinor=10`，`:724-734`），全找不到则回到 `4.0`..`4.9` 找最老的（`:736-742`）。
- `_createParameterMetaData`（`:609-612`）返回 `new APMParameterMetaData(this)`。
- `initializeVehicle`（`:512-527`）：离线编辑车时用 `*.OfflineEditing.params` 的头部反推固件版本（`_parseParamsHeader`，`:448-510`）。

### 2.2 `APMParameterMetaData`：APM JSON 的解析细节

| 行为 | 位置 | 说明 |
|---|---|---|
| 两层遍历 | `src/FirmwarePlugin/APM/APMParameterMetaData.cc:31-57` | 外层 group、内层 param；非对象项直接跳过 |
| 组名推导 | `:24-29` | 取参数名第一个 `_` 之前的前缀，再去掉**尾部数字**；因此 `SERVO1_MIN` → 组 `SERVO` |
| 单成员组降级 | `:59-71` | 组内只有一个参数时，组名改为 `FactMetaData::defaultGroup()`（`"Misc"`，`src/FactSystem/FactMetaData.h:111`、`:242`） |
| `DisplayName`→短描述 | `:87-90` | 空值不覆盖 |
| `Description`→长描述 | `:92-95` | 空值不覆盖 |
| `Units`→`setRawUnits` | `:97-100` | |
| `User`→**category** | `:102-105` | 值实测为 `"Standard"` / `"Advanced"` |
| `ReadOnly` | `:107-109` | 经 `jsonToBool`，同时接受 JSON bool 与字符串 `"True"`（`src/FirmwarePlugin/ParameterMetaData.h:40-41`） |
| `RebootRequired` | `:110-112` | 同上 |
| `Increment` | `:114-121` | 字符串转 double |
| `Range.low/high` | `:123-135` | 同时写入 `rawMin/rawUserMin` 与 `rawMax/rawUserMax`（经 `setRawConvertedValue` 做类型校验，`src/FirmwarePlugin/ParameterMetaData.cc:153-163`） |
| `Values`→枚举 | `:137-140`、`:178-197` | 键按数值排序（`_sortedNumericPairs`，`:150-176`）；**int8 类型时把 128..255 重解释为有符号**（`:186-194`，注释举的例子是 Sub-4.5+ 的 `BTN#_FUNCTION`） |
| `Bitmask`→位掩码 | `:142-145`、`:199-202` | 用组名/位序号字符串；基类对 int8/int16/int32/int64 的符号位另有重解释（`src/FirmwarePlugin/ParameterMetaData.cc:210-247`） |
| 未收录参数 | `:204-210` | category 强制 `"Advanced"`，组名仍按名字前缀推 |
| `_P/_I/_D` 浮点 | `:212-218` | 小数位强制 6 位 |

**实测存在的字段缺口（QGC 未消费）**：`apm.pdef.json` 实测字段共 12 个，上表覆盖 10 个；**`Volatile` 与 `Calibration` 没有被 `APMParameterMetaData` 读取**。对照 PX4 解析器会把 `volatileValue()` 参数强制只读（`src/FirmwarePlugin/PX4/PX4ParameterMetaData.cc:67-69`），APM 侧的 `Volatile` 参数（实测样例 `BARO1_GND_PRESS`）在 QGC 里**不会被自动置只读**。该差异对参数 UI 是可观察的：APM 的 volatile 参数仍可编辑并触发 `PARAM_SET`。

### 2.3 离线元数据文件在不在仓库里

**不在。** 实测：

- `15_NGC-Nexus/repo` 全局 `glob **/*.pdef.xml` 零命中；`glob **/APMParameterFactMetaData*` 零命中（`repo` 为浅克隆，`.git/shallow` 存在，无 `.gitmodules`，`libs/` 不存在）。
- `src/FirmwarePlugin/APM/CMakeLists.txt:41-53` 用 CPM 拉取外部仓库：
  ```
  CPMAddPackage(NAME ArduPilotParams GITHUB_REPOSITORY ArduPilot/ParameterRepository GIT_TAG main GIT_SHALLOW TRUE DOWNLOAD_ONLY TRUE)
  ```
- `:55-64` 定义排除正则 `QGC_APM_PARAMS_EXCLUDE`，默认排除 `AP_Periph-.*`、`Blimp-.*`、`^Copter-3[.]`、`^Plane-3[.]`、`^Rover-3[.]`。
- `:69` 按目录布局 `{Vehicle}-{Major}.{Minor}/apm.pdef.json` 做 `file(GLOB ...)`；`:99` 生成资源别名 `FirmwarePlugin/APM/APMParameterFactMetaData.${_vehicle}.${_version_dots}.json`；`:111-114` 用 `qt_add_resources` 打进二进制。
- 上游仓库目录实测（`gh api` 列目录）：`Copter-{3.5,3.6,3.7,4.0..4.8}`、`Plane-{3.8,3.9,3.10,4.0..4.8}`、`Rover-{3.4,3.5,3.6,4.0,4.1,4.2,4.4..4.8}`、`Sub-{3.4,3.5,3.6,4.0,4.1,4.5,4.7,4.8}`、`Tracker-{4.5..4.8}`、`AP_Periph-*`、`Blimp-{4.7,4.8}`。经过滤后进入 QGC 二进制的是各车型 4.x 系列。
- 每个版本目录内容（实测 `Copter-4.7`）：`apm.pdef.json`、`apm.pdef.xml`、`Parameters.md/.rst/.html/.rst(Latex)`、`MAVLinkMessages.rst`。**QGC 只打包 `apm.pdef.json`**（`:69` 的 glob 只看 json）。

### 2.4 元数据怎么被使用（QGC 运行时）

1. `ParameterManager` 收到 `PARAM_VALUE` → 新建 `Fact` → 取元数据：`_vehicle->compInfoManager()->compInfoParam(componentId)->factMetaDataForName(parameterName, fact->type())`（`src/FactSystem/ParameterManager.cc:250-252`）。
2. `CompInfoParam::_resolveMetaData`（`src/Vehicle/ComponentInformation/CompInfoParam.cc:88-112`）先看 `_noJsonMetadata`：
   - `_noJsonMetadata == true`（车辆没下发 JSON）→ `_getParameterMetaData()` → `FirmwarePlugin::loadParameterMetaData()`（`:138-148`）；
   - `_noJsonMetadata == false` → 查车辆下发的 JSON 元数据表，再查"带 `{n}` 索引模板"的正则表（`:114-136`）。
3. `FirmwarePlugin::loadParameterMetaData`（`src/FirmwarePlugin/FirmwarePlugin.cc:523-540`）→ `_createParameterMetaData()` → `_cachedParameterMetaDataFile(vehicle)` → `ParameterMetaData::loadParameterFactMetaDataFile`。
4. `_cachedParameterMetaDataFile`（`:542-593`）在"内置文件版本"与"缓存目录里同名同 major 的 json"之间选新者；缓存目录 `QStandardPaths::CacheLocation/ParameterMetaData`（`:513-521`）。版本号先按文件名 `\.(\d+)\.(\d+)\.json$` 解析（`src/FirmwarePlugin/ParameterMetaData.cc:124-132`），再退回 JSON 内 `parameter_version_major/minor`（`:99-109`）。APM 的 pdef 里没有这两个键（实测根键只有 `json.version == 0`），所以**APM 走文件名解析路径**。
5. `APMParameterMetaData` 的 `parseParameterJson` 一次性把整个 pdef 收进 `_rawParams`（`:31-57`），`_lookupMetaData` 再按需构造 `FactMetaData` 并缓存（基类 `src/FirmwarePlugin/ParameterMetaData.cc:63-74`）。

同一套内置文件的第二处消费者是日志查看器：`src/AnalyzeView/LogViewer/LogViewerParamMetaData.cc:27-55`、`:101-136` 复用同样的文件名回溯逻辑（注释明确写 "mirroring APMFirmwarePlugin::_internalParameterMetaDataFile logic"）来给 `.bin` 日志里的参数补单位/枚举。

### 2.5 `src/AutoPilotPlugins/APM/`（95 个文件）与元数据的关系

该目录下的组件几乎都是**"参数组合 + 少量 MAV_CMD"**的封装，本身不引入第二套元数据源：

- `APMAutoPilotPlugin::vehicleComponents()`（`APMAutoPilotPlugin.cc:61-199`）按"参数是否存在"和车类型/版本决定页面是否生成，例如 `ARSPD_TYPE`（`:86`）、`MOT_PWM_TYPE` / `Q_M_PWM_TYPE`（`:97-98`）、`SERVO1_MIN`（`:110`）、`FOLL_ENABLE`（`QT_DEBUG` 下，`:126`）、Sub 版本门控（`:76`、`:104`、`:160`）。
- 唯一自带独立元数据文件的 APM 组件是 Follow：`APMFollowComponent.FactMetaData.json`（该组件在 `#ifdef QT_DEBUG` 下才装配，`:124-131`）。

---

## 3. PX4 vs APM 差异对照表

| 差异项 | PX4 做法 | APM 做法 | 差异在 QGC 里的处理位置（文件:行号） | 对前端改造的影响 |
|---|---|---|---|---|
| 参数元数据来源 | 仓库内置 `PX4ParameterFactMetaData.json`；同时**支持**车辆通过 Component Metadata Protocol 下发更准的 JSON | **无内置文件在仓库**，靠 CPM 拉 `ArduPilot/ParameterRepository` 的 `apm.pdef.json`；MAVLink 侧**不下发**元数据 | `src/FirmwarePlugin/PX4/PX4FirmwarePlugin.h:57`；`src/FirmwarePlugin/APM/CMakeLists.txt:41-53, 69, 99, 111-114`；`src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:710-746` | 前端不能假设"元数据一定是当前固件版本的"：APM 侧版本靠文件名回溯，最坏落到 4.0；显示范围/单位可能与机上固件不符，UI 需容忍 |
| 元数据 JSON 结构 | 顶层 `{"version": >=1, "parameters": [ {name, ...} ]}` 平铺数组 | 顶层 `{"json":{"version":0}, "<Group>": {"<PARAM>": {...}}}` 两层映射 | 解析器：`src/FirmwarePlugin/PX4/PX4ParameterMetaData.cc:22-57` vs `src/FirmwarePlugin/APM/APMParameterMetaData.cc:31-57` | 任何"直接解析元数据文件"的前端脚本/工具必须分两套；不能写一个通用 JSON reader |
| 元数据字段命名 | camelCase（`shortDesc`/`longDesc`/`units`/`min`/`max`/`values`/`bitmask`…） | PascalCase（`DisplayName`/`Description`/`Units`/`Range.low`+`Range.high`/`Values`/`Bitmask`/`User`/`Increment`/`ReadOnly`/`RebootRequired`/`Volatile`/`Calibration`） | `src/FirmwarePlugin/APM/APMParameterMetaData.cc:87-145`；PX4 走 `FactMetaData::createFromJsonObject`（`src/FirmwarePlugin/PX4/PX4ParameterMetaData.cc:48`） | 字段映射不可复用；APM 侧枚举/位掩码键是**字符串数字**，需自行排序（`:150-176`） |
| 分类（category）来源 | JSON 的 category 字段；空则置 `"Standard"` | 取 `User` 字段（实测值 `Standard` / `Advanced`）；未收录参数强制 `Advanced` | `src/FirmwarePlugin/PX4/PX4ParameterMetaData.cc:63-65`；`src/FirmwarePlugin/APM/APMParameterMetaData.cc:102-105`、`:204-210` | 参数编辑器左侧分类树的语义两侧一致（Standard 置顶、Other 置底），但"谁进 Standard"由各固件元数据决定，前端不可硬编码参数名单 |
| 分组（group）来源 | JSON 的 group 字段 | 由参数名前缀推导并去掉尾数字；单成员组降级为 `"Misc"` | `src/FirmwarePlugin/APM/APMParameterMetaData.cc:24-29`、`:59-71`；兜底逻辑 `src/Vehicle/ComponentInformation/CompInfoParam.cc:103-107` | APM 组名是"启发式"的，可能出现拼写相近的组（实测上游同时存在 `ARSPD`、`ARSPD2_`…`ARSPD6_`、`ARSPD_`）；前端若做"组=功能模块"的强映射会不可靠 |
| 元数据版本选择 | 单一内置文件 + 车辆下发 JSON 覆盖；缓存目录择优 | 按 `车型+major+minor` 选文件，minor 逐级回溯到 4.0 | `src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:710-746`；`src/FirmwarePlugin/ParameterMetaData.h:21-24`、`src/FirmwarePlugin/ParameterMetaData.cc:99-132` | 前端需要展示"元数据实际来自哪个版本"时，只能拿文件名；`vehicle->firmwareVersion()` 与元数据版本可能不同 |
| 参数写入的类型编码 | 按标准 MAVLink union 编码 | 出向强制"塞回 float 字段"，且只对 ArduPilot 组件生效 | `src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:191-245`；`ParameterManager::_fillMavlinkParamUnion`（`src/FactSystem/ParameterManager.cc:466-504`） | 前端自定义的"直接发 PARAM_SET"若绕过 `ParameterManager`，APM 上会写错值；必须走 `Fact::setRawValue` |
| 参数读取的类型解码 | 标准解码 | 入向对 ArduPilot 组件做"假 float → 真类型"重解释 | `src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:129-189`、`:292-330` | 自定义 MAVLink 处理链若在 `FirmwarePlugin` 之前截获 `PARAM_VALUE`，会拿到未修正的值 |
| 参数本地缓存 | 有（CRC 校验的 `_HASH_CHECK` + 落盘） | **无**（ArduPilot 参数 volatile，缓存会失效） | 判定与写入：`src/FactSystem/ParameterManager.cc:157-164`、`:264-272`、`:619-644`；落盘函数 `:1133-1150`、`:1161-1164` | 前端"秒开参数页"的体验只能在 PX4 实现；APM 每次连接都必须等全量参数（FTP 或 stream），UI 需给更明确的进度/等待态 |
| 参数全量获取通道 | 优先 `_HASH_CHECK` → 命中则本地缓存；否则 component metadata / FTP / stream | 优先 MAVLink FTP `@PARAM/param.pck?withdefaults=1`；失败/太慢回退 `PARAM_REQUEST_LIST` | `src/FactSystem/ParameterManager.cc:44`、`:646-730`、`:547-594`、`:1501-1529` | APM 首次连接耗时与链路带宽强相关；`_ftpDownloadProgress`（`:596-603`）可驱动新 UI 进度条 |
| 参数默认值 | 来自元数据 | **由机上 `param.pck?withdefaults=1` 提供**，并强制覆盖元数据默认值 | `src/FactSystem/ParameterManager.cc:1881-1886`、`:1900-1902`；magic `0x671C`（`:1712`、`:1749`） | APM 上"Modified"页签与"值≠默认值"高亮依赖机上默认值；默认值缺失时 `defaultValueAvailable` 为假，前端需降级显示 |
| 参数名跨版本重命名 | 基类空映射表（本 HEAD 未覆写） | 有显式重映射表：Copter 4.0/4.7、Plane 4.5/4.7、Rover 4.7、Sub 4.7 | 基类 `src/FirmwarePlugin/FirmwarePlugin.cc:230-235`；Copter `src/FirmwarePlugin/APM/ArduCopterFirmwarePlugin.cc:71-165`；Plane `ArduPlaneFirmwarePlugin.cc:71-143`；Rover `ArduRoverFirmwarePlugin.cc:49-59`；Sub `ArduSubFirmwarePlugin.cc:129-191`；消费 `src/FactSystem/ParameterManager.cc:1531-1568` | 前端引用参数名**必须用新版名**（注释明确：`src/FirmwarePlugin/FirmwarePlugin.h:93-104`）；绕过 `ParameterManager` 直接按名字查会漏；`noremap.` 前缀可绕过映射（`:1533-1536`） |
| 多组件参数 | 支持（component metadata / 多组件 ID） | 支持（FTP 只覆盖 `MAV_COMP_ID_AUTOPILOT1`，其余组件靠 stream 逐个读） | `src/FactSystem/ParameterManager.cc:1914`（FTP 路径只填 autopilot1）、`:1387-1398`（默认组件门控）、`:1452-1487`（其他组件单独报告） | 前端多组件参数树在 APM 上可能长期处于"部分加载"；`missingParameters` 与 `_otherComponentsReported` 两条提示要区分展示 |
| 未请求参数的处理 | 无此怪癖 | ArduPilot 会推送未请求的参数，QGC 在初始列表响应前丢弃 | `src/FactSystem/ParameterManager.cc:150-155` | 自定义连接状态机若假定"收到 PARAM_VALUE 即有效"，在 APM 上会误判参数已就绪 |
| 离线编辑参数 | `:/FirmwarePlugin/PX4/PX4.OfflineEditing.params` | `:/FirmwarePlugin/APM/{Copter,Plane,Rover,Sub}.OfflineEditing.params`（实测 Copter 文件 1395 行、头部 `# Stack: ArduPilot` / `# Vehicle: Multi-Rotor` / `# Version: 4.7.0`） | 接口 `src/FirmwarePlugin/FirmwarePlugin.h:331`；加载 `src/FactSystem/ParameterManager.cc:1570-1637`；头部解析 `src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:448-510`、`:512-519` | 离线编辑的"可用参数集合"是固件版本快照；APM 侧比 PX4 多一层"由文件头反推车型/版本"，前端展示离线车辆版本时应读 `Vehicle` 而非猜 |
| Component Metadata Protocol | 支持（QGC 主动请求 `COMP_METADATA_TYPE_PARAMETER`） | 不支持，请求必然失败（QGC 已有专门降噪分支） | 请求链 `src/Vehicle/ComponentInformation/ComponentInformationManager.cc:59-69`、`:210-216`、`:246-250`；失败降噪 `src/Vehicle/ComponentInformation/RequestMetaDataTypeStateMachine.cc:438-447` | 前端若新增"元数据管理/健康度"面板，APM 上必须显示"元数据为地面站内置"而不是"下发失败" |

---

## 4. 同一套参数 UI 兼容两种固件：需要适配的点

以下按"从数据到界面"分层列出，每条给现有位置与建议改动位置。

### A. 元数据层（数据正确性）

1. **参数名解析必须走 `ParameterManager::getParameter` / `parameterExists`，不得自行拼名字查表。**
   依据：APM 有 4 档参数重映射（§3 表"参数名跨版本重命名"行）。建议改动位置：QML 侧统一走 `FactPanelController` 的 `getParameterFact(int componentId, const QString &name, bool reportMissing)` 与 `parameterExists(int componentId, const QString &name)`（`src/FactSystem/FactControls/FactPanelController.h:25-26`），C++ 侧走 `ParameterManager::getParameter` / `parameterExists`；若确有需要按新名硬引用，改用 `noremap.` 前缀显式声明（`ParameterManager.cc:1533-1536`）。

2. **元数据缺失的参数必须能显示。**
   依据：APM 参数集随固件变化，`apm.pdef.json` 不可能穷尽；未收录参数走 `_createDefaultMetaData`（`APMParameterMetaData.cc:204-210`，category=`Advanced`）或 `CompInfoParam` 的裸 `FactMetaData` 兜底（`CompInfoParam.cc:103-111`，无描述/单位/范围）。建议改动位置：`src/QmlControls/ParameterEditorDialog.qml` 的数值输入分支与 min/max 展示块（`:94-204`，其中 min/max 显示在 `:186-204`，已用 `!fact.minIsDefaultForType` / `!fact.maxIsDefaultForType` 做条件显示）——对无 min/max 的参数不要报"值超范围"，而应提示"该参数无地面站元数据"。

3. **不要依赖 `category` 字符串做功能判断。**
   依据：PX4 由 JSON category 决定、空则 `Standard`（`PX4ParameterMetaData.cc:63-65`）；APM 由 `User` 字段决定（`APMParameterMetaData.cc:102-105`）。两边都只保证 `Standard` 与 `Other`（`FactMetaData.h:241`）的排序约定（`ParameterEditorController.cc:230-250`：`Standard` 提到首位、`Other` 挪到末位）。建议：任何"按分类筛选业务参数"的逻辑改成按参数名/组名前缀白名单。

4. **`Volatile` / `Calibration` 字段的处理需要补齐或显式声明不支持。**
   依据：实测 `apm.pdef.json` 含这两个字段，但 `APMParameterMetaData::_lookupMetaData`（`:73-148`）不读取；PX4 侧有 `volatileValue() → setReadOnly(true)`（`PX4ParameterMetaData.cc:67-69`）。建议改动位置：`APMParameterMetaData.cc:107-112` 附近增加 `Volatile` → `metaData->setVolatileValue(true)`（setter 存在，`src/FactSystem/FactMetaData.h:203`；取值 `:155`；JSON 键常量 `:469`），并确认是否同时置只读（需先确认 `FactMetaData` 的 volatile 语义与参数缓存 CRC 路径的耦合，见 `ParameterManager.cc:1200-1208`；注意该路径只对 PX4 生效）。**该改动是否与上游意图一致未验证。**

### B. 参数加载/写入层

5. **写参数一律走 `Fact::setRawValue`，不得自组 `PARAM_SET`。**
   依据：APM 出向需要"塞回 float 字段"的修正（`APMFirmwarePlugin.cc:191-245`），且该修正依赖 `_ardupilotComponentMap`（`:203-206`）。建议：前端如需"批量写入"，复用 `ParameterManager::bulkRefresh` / diff 写入路径（`ParameterEditorController.cc:400-432`）。

6. **连接等待态要区分 PX4 缓存命中与 APM 全量下载。**
   依据：APM 无本地参数缓存（`ParameterManager.cc:264-272` 只在 `px4Firmware()` 时写缓存），且优先走 FTP（`:673-700`）。建议改动位置：`src/Vehicle/InitialConnectStateMachine.cc` 的参数状态段（`:388-416`）：`loadProgressChanged` 连接在 `:403-404`，`cacheCheckOnlyFailed`（仅 PX4 缓存检查用）在 `:395-400`，`initialParametersRequestFailed` 在 `:407-411`；对 APM 追加"参数文件下载中/回退到流式"的文案分支。

7. **"Reset to vehicle's configuration defaults" 必须对 APM 隐藏或改语义。**
   依据：`src/QmlControls/ParameterEditor.qml:53-60` 已用 `visible: !_activeVehicle.apmFirmware` 隐藏；底层 `ParameterManager::resetAllToVehicleConfiguration` 写的是 PX4 的 `SYS_AUTOCONFIG=2`（`ParameterManager.cc:1648-1655`）。建议：若产品需要在 APM 上提供同类能力，应改为调用 APM 自己的参数重置语义（**APM 侧对应机制未验证**），不要复用 `SYS_AUTOCONFIG`。

8. **"Clear all RC to Param" 在 APM 无对应物。**
   依据：`src/QmlControls/ParameterEditor.qml:19`（`_showRCToParam: _activeVehicle.px4Firmware`）、`:82-87`。建议：保留现有门控，不要为"UI 一致性"强行打开。

### C. 展示层

9. **枚举/位掩码渲染依赖元数据完整性，需为 APM 做退化路径。**
   依据：`ParameterEditorDialog.qml:25`（`_showCombo = enumStrings.length !== 0 && bitmaskStrings.length === 0`）、`:114`、`:154-158`。APM 枚举来自 pdef 的 `Values`，位掩码来自 `Bitmask`，二者都可能缺失。建议：缺枚举时回落到数值输入并展示原始数值提示。

10. **`RebootRequired` 提示两侧都可得，但要确认字段确实被读到。**
    依据：APM `RebootRequired` → `setVehicleRebootRequired`（`APMParameterMetaData.cc:110-112`），消费点 `ParameterEditorDialog.qml:208`。实测 `SERVO1_FUNCTION` 带 `RebootRequired: "True"`。建议：验收时用带该字段的参数（如 `SERVO1_FUNCTION`）实机确认提示出现。

11. **参数名长度与大小写不要在前端做假设。**
    依据：两侧都受 16 字节限制（APM `AP_MAX_NAME_SIZE 16`；QGC `ParameterManager.cc:114-116`）。建议：前端截断/对齐显示按字节与字符双写测试，避免 16 字符满长的参数名被截断成两个不同参数。

12. **多组件参数树要能表达"加载中/缺失"。**
    依据：`ParameterManager.cc:1387-1398`、`:1418-1431`、`:1452-1487`。建议：在参数页顶部增加 `missingParameters` 与 `_otherComponentsReported` 对应的两档提示，而不是复用同一句"参数缺失"文案。

### D. 固件能力层

13. **能力判断用 `FirmwarePlugin::isCapable` 与 `Vehicle::*Firmware()`，不要在 QML 里堆固件分支。**
    依据：APM 能力位在 `APMFirmwarePlugin.cc:65-82`（含/不含哪些 capability）。建议：新增功能时先在插件层加 capability 或虚函数，避免继续增加 `ParameterEditor.qml:19/55`、`RadioComponent.qml:42` 这类散点分支。

---

## 5. "QGC 有 UI 但 APM 不支持 / 反之"的典型区域

| 区域 | 方向 | 依据 | 备注 |
|---|---|---|---|
| "Reset to vehicle's configuration defaults" 菜单项 | QGC 有 UI，APM 无对应 | `src/QmlControls/ParameterEditor.qml:53-60`（按 `apmFirmware` 隐藏）；`src/FactSystem/ParameterManager.cc:1648-1655`（PX4 `SYS_AUTOCONFIG`） | 已用可见性门控处理，属"已适配" |
| "Clear all RC to Param" 菜单项 | QGC 有 UI，APM 无对应 | `src/QmlControls/ParameterEditor.qml:19`、`:82-87` | 同上 |
| 参数本地缓存 / 秒开 | PX4 有，APM 无 | `src/FactSystem/ParameterManager.cc:264-272`、`:619-644`、`:1133-1150` | 不是 UI 缺失，是性能特性缺失 |
| Component Metadata 驱动的参数元数据 | PX4 有，APM 无 | `src/Vehicle/ComponentInformation/ComponentInformationManager.cc:59-69`、`:246-250`；`RequestMetaDataTypeStateMachine.cc:441-443` | QGC 已在日志层降噪，但 UI 层没有"元数据来源"提示 |
| 元数据里的 `Volatile` 语义 | PX4 有，APM 的 `Volatile` 字段被忽略 | `src/FirmwarePlugin/PX4/PX4ParameterMetaData.cc:67-69` vs `src/FirmwarePlugin/APM/APMParameterMetaData.cc:73-148` | 见 §4 第 4 条；APM 的 volatile 参数可被误编辑 |
| `OrbitModeCapability` | APM 未声明 | `src/FirmwarePlugin/APM/APMFirmwarePlugin.cc:65-82` 的 `available` 位集合不含 `OrbitModeCapability` | 绕点飞行相关 UI 需由 capability 门控 |
| `VTOLMulticopterTakeoffCapability` | APM 未声明 | 同上一行 | 对 VTOL 机型有可见差异 |
| 传感器/标定页 | QGC 有，且两侧实现不同 | APM：`src/AutoPilotPlugins/APM/APMSensorsComponent*.{h,cc,qml}`；PX4：`src/AutoPilotPlugins/PX4/SensorsComponent*` | 非参数 UI，但标定过程大量写参数，改造时两侧要分别验收 |
| 遥控器（Radio）页 | QGC 有，两侧参数名/流程不同 | `src/AutoPilotPlugins/Common/RadioComponentController.cc:269`（`rgStickFunctionParamsPX4` vs `rgStickFunctionParamsAPM`）、`:106`、`:141-144`、`:176`；`RadioComponent.qml:42` | 典型"同一 UI、两套参数名" |
| 天线跟踪（Antenna Tracker） | APM 有车类型，但 QGC 无对应 APM 插件 | `src/FirmwarePlugin/APM/APMFirmwarePluginFactory.cc:29-71` 无 `MAV_TYPE_ANTENNA_TRACKER` 分支，落 Generic 插件；上游参数仓库存在 `Tracker-4.5..4.8` | 属"反之"：APM 侧有，QGC 侧无专用 UI。**QGC 是否有意不支持未验证** |
| 命令（任务项）元数据 | 两侧各自一套 JSON | APM：`src/FirmwarePlugin/APM/APM-MavCmdInfo*.json` + `APMFirmwarePlugin.cc:588-607`；PX4：`src/FirmwarePlugin/PX4/PX4-MavCmdInfo*.json` | 不是参数系统，但属于同一类"固件差异必须在插件层落地"的模式 |
| 参数"Modified"页签的默认值高亮 | 两侧都有 UI，数据来源不同 | `src/QmlControls/ParameterEditor.qml:428-429`、`ParameterEditorController.cc:656-660`；APM 默认值来自机上 `param.pck`（`ParameterManager.cc:1881-1886`） | APM 上若机上未提供默认值，该页签会缺少高亮基线 |

---

## 6. 未验证清单

以下条目本文**无法给出确定结论**，列出以免被当作已核实事实：

1. `Volatile` / `Calibration` 字段在 ArduPilot 侧的**运行时**语义（是否表示"不应由 GCS 写入"）——只核实到字段存在于 `apm.pdef.json`，未核实 ArduPilot 源码中的消费方式。
2. PR #31355 的合并状态与最终产物形态（未验证）。
3. PR #32599 描述中"4 个超过 16 字符的参数名"的具体名单及其在 MAVLink 上的传输方式（未验证）。
4. QGC 是否**有意**不支持 Antenna Tracker（`Tracker-*` 参数目录存在但无 APM 插件分支）——只核实到代码现状。
5. APM 侧"恢复出厂/恢复配置默认"的对应机制是否存在、如何触发（未验证）。
6. `apm.pdef.json` 中 `json.version` 的语义（实测值为 `0`；QGC 侧 `ParameterMetaData::versionFromJsonData` 读的是 `parameter_version_major/minor`，`src/FirmwarePlugin/ParameterMetaData.cc:103-107`，**不读** `version`，而 PX4 解析器读 `version` 并拒绝 `<1`，`PX4ParameterMetaData.cc:24-28`）——该差异是否会导致 APM 元数据版本判定异常，未做运行时验证。
7. 本文所有 APM 侧数据均取自 GitHub `master` / `Copter-4.7` 发布产物，**未经实机或 SITL 验证**；本机无 Qt6/MSVC，QGC 侧结论均为静态阅读结论。

---

## 7. 引用来源索引（QGC 侧 + APM 侧）

### QGC 源码（本机，HEAD `25185047e855937d694a0174c1541d8f7d794a74`）

- `src/FirmwarePlugin/APM/CMakeLists.txt`、`APMFirmwarePlugin.cc/.h`、`APMParameterMetaData.cc/.h`、`APMFirmwarePluginFactory.cc`、`ArduCopter/Plane/Rover/SubFirmwarePlugin.cc/.h`
- `src/FirmwarePlugin/FirmwarePlugin.cc/.h`、`src/FirmwarePlugin/ParameterMetaData.cc/.h`
- `src/FirmwarePlugin/PX4/PX4ParameterMetaData.cc`、`PX4FirmwarePlugin.h`
- `src/FactSystem/ParameterManager.cc/.h`、`FactMetaData.h`、`FactControls/FactPanelController.h`
- `src/Vehicle/ComponentInformation/{CompInfoParam.cc,ComponentInformationManager.cc,RequestMetaDataTypeStateMachine.cc}`
- `src/QmlControls/{ParameterEditor.qml,ParameterEditorDialog.qml,ParameterEditorController.cc}`
- `src/AutoPilotPlugins/APM/APMAutoPilotPlugin.cc`、`src/AutoPilotPlugins/Common/RadioComponentController.cc`
- `src/AnalyzeView/LogViewer/LogViewerParamMetaData.cc`、`src/Comms/MockLink/MockLinkFTP.cc`
- `src/Vehicle/VehicleSetup/FirmwareImage.cc`

### ArduPilot 侧（联网）

| 用途 | URL |
|---|---|
| 参数元数据生成脚本 | <https://github.com/ArduPilot/ardupilot/blob/master/Tools/autotest/param_metadata/param_parse.py> |
| `apm.pdef.json` 发射器 | <https://github.com/ArduPilot/ardupilot/blob/master/Tools/autotest/param_metadata/jsonemit.py> |
| 参数名长度上限 | <https://github.com/ArduPilot/ardupilot/blob/master/libraries/AP_Param/AP_Param.h> |
| `PARAM_VALUE` 发送实现 | <https://github.com/ArduPilot/ardupilot/blob/master/libraries/GCS_MAVLink/GCS_Param.cpp> |
| 参数仓库说明 | <https://github.com/ArduPilot/ParameterRepository/blob/main/README.md> |
| 参数仓库发布脚本 | <https://github.com/ArduPilot/ParameterRepository/blob/main/scripts/run_parsers.py> |
| Copter-4.7 `apm.pdef.json` | <https://github.com/ArduPilot/ParameterRepository/blob/main/Copter-4.7/apm.pdef.json> |
| Copter-4.7 MAVLink 支持表（`PARAM_EXT_*` UNSUPPORTED 出处） | <https://github.com/ArduPilot/ParameterRepository/blob/main/Copter-4.7/MAVLinkMessages.rst> |
| MAVLink 扩展参数协议（"为相机发明、飞行栈不支持"） | <https://mavlink.io/en/services/parameter_ext.html> |
| MAVLink Component Metadata Protocol | <https://mavlink.io/en/services/component_metadata.html> |
| PR #32599（`COMP_METADATA_TYPE_PARAMETER` 发射器，closed 未合并） | <https://github.com/ArduPilot/ardupilot/pull/32599> |
| PR #31355（版本化 `apm.pdef.*` 构建产物） | <https://github.com/ArduPilot/ardupilot/pull/31355> |

### 相关文档

- 本目录 `01_QGC源码结构梳理.md`：QGC 源码结构与模块划分。
- 本目录 `03_QGC参数链路.md`：QGC 参数子系统从 MAVLink 报文到 Fact 对象的完整链路（本文 §2.4 与其 §5 有交集，交叉引用即可）。
