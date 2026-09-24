# NGC-Nexus

**N**exus **G**round **C**ontrol —— 基于 QGroundControl 二次开发的无人机地面站，目标是**重构前端交互，把地面站改得更好用**。

- 上游基座：`mavlink/qgroundcontrol`（HEAD `25185047e855937d694a0174c1541d8f7d794a74`，2026-09-25）
- 本仓库：https://github.com/Whale-Talk/NGC-Nexus
- 本机工作目录：`E:\04-Workspace_workbudy\15_NGC-Nexus\repo`

---

## 1. 项目定位

QGC 功能完整但前端交互偏"工程师自用"：参数页面信息密度高、写入反馈缺失、跨固件（PX4 / ArduPilot）行为不一致。NGC-Nexus 的工作集中在**前端可用性重构**，不改飞控侧协议、不破坏上游可合并性。

主线方向（待细化）：

| 方向 | 依据（来自本项目代码梳理） |
|---|---|
| 参数写入反馈补全 | 写入成功当前**完全静默**，只写日志，无任何 UI 提示 |
| 参数检索能力增强 | 搜索只匹配 name/shortDesc/longDesc，**不匹配枚举文本**，按显示值搜不到参数 |
| 参数页分组排序修正 | `Other` 分类在多组件场景不置底；`Misc` 组置底判断存在下标误用 |
| 启用参数页自动推断控件 | 生成式参数页在缺 `control` 键时固定回落文本框，文档声称的自动推断未生效 |
| 双固件行为一致性 | PX4 与 APM 的元数据来源、只读判定、枚举呈现路径完全不同 |

---

## 2. 仓库结构

```
15_NGC-Nexus/
├─ repo/                    # QGC 源码（本仓库主体，fork 自 mavlink/qgroundcontrol）
├─ docs/                    # NGC-Nexus 分析文档（中文，工作副本）
├─ logs/                    # 克隆/推送/构建日志
├─ clone-qgc.ps1            # 克隆脚本（openssl TLS 后端修复已内置）
├─ fetch-history-retry.ps1  # 完整历史补全（带重试）
├─ push-fork.ps1 / fix-shallow-and-push.ps1
└─ scripts/                 # 环境修复与诊断脚本
```

> 分析文档同时以 `repo/docs/NGC-Nexus/` 形式进入本仓库版本管理，便于在 GitHub 上直接阅读。

---

## 3. 分析文档索引

| 文档 | 内容 |
|---|---|
| [00 环境诊断](docs/NGC-Nexus/00_环境诊断-失败原因.md) | 本机反复踩到的 6 类失败：TLS 后端、沙箱写盘、无 pwsh、命名管道、`gh` flag、PS 5.1 脚本编码 |
| [01 QGC 源码结构梳理](docs/NGC-Nexus/01_QGC源码结构梳理.md) | 顶层与 `src/` 32 个模块职责表、构建与依赖解析、启动骨架、QML 技术栈 |
| [02 PX4 参数链路](docs/NGC-Nexus/02_PX4参数链路.md) | PX4 侧：YAML 定义 → 生成器 → 固件/外部元数据 → Component Information → MAVLink |
| [03 QGC 参数链路](docs/NGC-Nexus/03_QGC参数链路.md) | QGC 侧：MAVLink 报文 → `ParameterManager` → `Fact` → 元数据裁决 |
| [04 参数到界面映射](docs/NGC-Nexus/04_参数到界面映射.md) | 元数据字段 → QML 控件、分组/搜索/单位/越界/reboot/降级 |
| [05 APM 参数系统对照](docs/NGC-Nexus/05_APM参数系统对照.md) | ArduPilot 差异、双固件适配点、"有 UI 无支持"清单 |
| [10 参数系统综述](docs/NGC-Nexus/10_参数系统综述.md) | 跨全部来源的整合报告与前端改造切入点 |

---

## 4. 构建（本机当前不可用）

QGC 需要 **Qt 6 + C++20 编译器**。本机实测：

| 依赖 | 状态 |
|---|---|
| CMake / Ninja | ✅ 已装 |
| Node / pnpm | ✅ 已装（QML 工具链用） |
| **Qt 6** | ❌ 未安装 |
| **MSVC (`cl`)** | ❌ 未安装 |

因此当前阶段所有代码结论均为**源码阅读级**，未经编译与实机验证。文档中所有 UI 表现类判断均标注了这一点。

构建命令（供环境就绪后使用，来自 QGC 上游 `AGENTS.md`）：

```bash
just configure   # CMake 配置
just build       # 增量构建
just test        # ctest
just lint        # 预提交检查
```

---

## 5. 本机环境注意事项（重要，避免重复踩坑）

1. **git 必须用 openssl TLS 后端**：默认 schannel 在本机报 `SEC_E_NO_CREDENTIALS`，任何网络 git 操作都会失败。
   ```powershell
   git -C repo config http.sslBackend openssl
   ```
2. **只有 Windows PowerShell 5.1，没有 pwsh**：脚本必须 5.1 兼容。
3. **给 5.1 的 `.ps1` 文件必须是纯 ASCII**：5.1 会按 GBK 解码无 BOM 的 UTF-8 脚本，中文字符串会破坏语法导致解析失败。
4. **沙箱禁止子进程创建命名管道**：`sh.exe`/`bash.exe` 无法启动，导致 `git submodule` 与 git `credential.helper`（shell 实现）失效；推送时凭证需改用 `gh auth token` 或原生助手。
5. **写工作区之外被拒**：如 `C:\Users\Marx\.gitconfig` 不可写，只能改仓库级配置。
6. **`gh 2.96` 不支持 `--add-topic`**：topics 需用 `gh api -X PUT .../topics` 一次性设置（分次调用会互相覆盖）。

---

## 6. 许可

上游 QGC 为 GPLv3 / Apache-2.0 双许可，本仓库继承上游许可，详见 `LICENSE-GPL` 与 `LICENSE-APACHE`。
