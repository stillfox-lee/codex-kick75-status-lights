# NuPhyIO Kick75 IO 能力与协议地图

> 研究快照：2026-08-03  
> 官方客户端：`https://drive.nuphyio.com/static/js/main.f6f60294.js` 及其动态 chunk  
> 适用设备：NuPhy Kick75 IO，VID `0x19f5`、PID `0x1026`、客户端内部设备枚举 `Kick75 = 5`

本文用于指导本项目后续扩展，不是 NuPhy 官方协议规范。分析只读取官方公开 Web 客户端静态资源和仓库既有真机证据；本轮没有向键盘发送任何 HID 报告。

## 1. 结论

NuPhyIO 不是普通网页表单，而是通过 WebHID 直接访问键盘固件的官方 Web 驱动。Kick75 IO 的能力边界可分为四层：

```text
NuPhyIO 页面
  └─ Kick75 能力开关与固件版本门槛
      └─ window.keyboardApi 设备 API
          └─ 64 字节 Raw HID 命令
              └─ 键盘固件中的配置区与运行状态
```

当前可以确认 Kick75 IO 支持：

- Mac、Windows 两个模式，以及各模式的分层键位映射；默认矩阵数据对应每个模式 4 层。
- 普通键、媒体键、鼠标键、鼠标滚轮、灯光键和特殊键映射；旋钮作为三个独立键位处理。
- 宏的读取、保存、绑定和删除；Kick75 宏动作类型限键盘、鼠标键、鼠标滚轮，客户端标记为不支持宏循环。
- 主键区与侧灯的状态读取和设置，以及自定义灯效数据。
- 自动休眠、浅睡时间、深睡时间、游戏优化及若干系统键禁用项。
- 电池状态、固件信息、键盘基础信息和模式状态通知。
- TGL 和 Tap Dance，但受固件版本门槛控制。
- 通过 U1 接收器的无线相关能力和接收器固件/回报率管理入口。
- 配置导入、导出、单模式重置、整机重置和固件升级；这些均属于高风险写操作，不应直接复用到本项目的第一阶段实现。

当前官方 Kick75 能力配置明确关闭：游戏手柄模式、可选控制器类型、独立灯区控制、渐变模式。客户端通用类中即使存在相关方法，也不能据此认定 Kick75 支持。

官方后端 `keyBoardList` 返回的 `type = 3` 是产品分类字段，与客户端协议分派使用的内部枚举
`Kick75 = 5` 不是同一概念，不能混用。

## 2. 证据等级

| 等级 | 含义 | 可用于什么结论 |
| --- | --- | --- |
| A | 本仓库对 Kick75 IO 真机完成过读写并恢复 | 可直接实现，但仍需保留快照和回滚 |
| B | 官方客户端 Kick75 专属能力配置、虚拟设备数据或专属 UI 布局 | 可作为实现规格，写入前仍需只读真机验证 |
| C | 官方客户端设备 API 类或命令枚举中存在，但不是 Kick75 专属启用证据 | 只能说明协议栈具备该能力，不能直接开放 |

当前 A 级证据只有临时会话、完整灯光状态读取、侧灯 8 字节写入和原样恢复，详见 [PROTOCOL.md](PROTOCOL.md)。键位、宏、休眠、整机设置和固件相关能力均未在本仓库真机写入验证。

## 3. Kick75 专属能力矩阵

下表来自官方客户端主包中的 Kick75 能力对象；版本号按客户端原始字符串保留。

| 领域 | Kick75 配置 | 证据 | 备注 |
| --- | --- | --- | --- |
| 模式 | `Mac`、`Windows` | B | 默认矩阵显示每模式 4 层 |
| 键位映射 | 支持 | B | 包括普通键、媒体键、鼠标键、滚轮、灯光键、特殊键 |
| 旋钮 | 三个独立位置 | B | 索引 14=左旋、15=按压、16=右旋；默认分别为音量减、静音、音量加 |
| 宏动作类型 | 键盘、鼠标键、鼠标滚轮 | B | `macroLoop.supported=false` |
| TGL | 支持，最低 `4.0.4.5` | B | 版本门槛由客户端控制 |
| Tap Dance | 支持，最低 `4.0.6.5` | B | 版本门槛由客户端控制 |
| 自动休眠 | 支持 | B | 包含开关、浅睡、深睡配置 |
| 无线 | 支持 | B | 接收器类型为 `U1` |
| 应用快捷键 | 支持 | B | 属于扩展键能力，不等同于 macOS 全局快捷键注册 |
| 自定义灯效 | 支持 | B | `customLightAll.supported=true` |
| 主键灯/侧灯 | 80/5 个逻辑灯位 | B | 映射包含跳过的旋钮位置及侧灯颜色索引 |
| 灯区独立控制 | 不支持 | B | `lightSeparate.supported=false` |
| 渐变模式 | 不支持 | B | `gradientMode.supported=false` |
| 游戏手柄模式 | 不支持 | B | 通用 API 存在，但 Kick75 禁用 |
| 可选控制器类型 | 不支持 | B | `hasSelectControl=false` |
| 轴体类型 | 机械轴 | B | `isMechanical=true` |

媒体键和扩展特殊键还存在固件版本门槛：

- `extendedV3` 媒体键：最低 `4.0.15.6`。
- `extendedV4` 媒体键：最低 `1.0.15.6`。
- `KC_F_REVERSE` 扩展特殊键：最低 `4.0.8.6`。

这些版本字符串来自官方客户端，可能对应不同固件分支，不能按普通语义版本互相比较。

## 4. 设备数据模型

### 4.1 键盘与模式

官方虚拟设备模块为 `Kick75`，导出以下配置区快照：

| 数据区 | 客户端导出名 | 用途 |
| --- | --- | --- |
| 基础信息 | `baseData` | 模式数、层数、矩阵和能力基础参数 |
| 固件信息 | `firmwareInfoData` | 固件版本及相关版本信息 |
| 默认键位矩阵 | `defaultKeyMatrixData` | Mac/Windows 各层的出厂映射 |
| 键盘功能 | `keyboardFuncData` | 每个模式的功能标志 |
| 灯光状态 | `lightStateData` | 每个模式的灯效状态 |
| 灯光颜色 | `lightColorData` | 按键灯颜色表 |
| 休眠信息 | `sleepInfoData` | 休眠开关和时间参数 |
| SOCD 数据 | `socdData` | 高级键数据区的兼容存储 |
| App 自定义区大小 | `appDefineSizeData` | 模式名、提醒时间等自定义数据的布局依据 |

`defaultKeyMatrixData` 是 2 字节键码序列。客户端将键盘抽象为模式 → 层 → 物理位置 → KeyInfo；旋钮左旋、按压、右旋只是三个特殊物理位置，因此调整旋钮映射不需要监听旋钮事件，而是修改对应层的三个 KeyInfo。

### 4.2 键位对象

客户端对单键使用统一的 KeyInfo 聚合模型，普通映射包含键码、类型、层和物理位置；高级行为在同一对象上附加数据：

- `Macro`：宏 ID、动作序列和绑定信息。
- `MT`：Mod-Tap 类双行为和延迟。
- `TGL`：锁定/切换键行为。
- `TapDance`：多击行为和时间参数。
- `SOCD`：一对按键及其冲突处理规则。
- `DKS`、`RS`、`HT`、`RT`：客户端通用高级键结构，主要服务磁轴型号；它们出现在通用 API 类中，不等于 Kick75 UI 已启用。

### 4.3 灯光状态

Kick75 的灯光状态为 17 字节：主键区 9 字节、侧灯 8 字节。客户端模型统一为：

```text
LightState {
  lightType, mode, brightness, speed,
  direction?, isRGB, colorIndex, color
}
```

主键区和侧灯支持的模式清单及 8 字节侧灯编码见 [PROTOCOL.md](PROTOCOL.md)。

### 4.4 宏

客户端提供按模式读取全部宏、保存宏、同步宏区、绑定宏键、查询空闲宏槽和删除宏。宏动作先转换为内部动作序列，再写入宏存储区，最后更新键位绑定。它不是单一“把快捷键文本写入键盘”的操作，因此实现时必须把宏内容和键位引用视为一个一致性边界。

## 5. HID 命令枚举

Kick75 IO 使用 64 字节协议，主机方向字为 `0x55`，设备响应为 `0xaa`。完整帧、校验及 XOR 会话掩码见 [PROTOCOL.md](PROTOCOL.md)。下表来自官方客户端模块 `40877` 的命令枚举。

### 5.1 读取命令

| 命令 | 值 | 作用 | 等级 |
| --- | ---: | --- | --- |
| `GetDongelName` | `0x2f` | 读取接收器名称 | C |
| `GetBase` | `0xa0` | 读取设备基础信息 | B |
| `GetFirmwareInfo` | `0xa1` | 读取固件信息 | B |
| `GetDefaultKeys` | `0xb1` | 读取默认键位矩阵 | B |
| `GetUseKeys` | `0xb2` | 读取当前键位矩阵 | B |
| `GetSOCD` | `0xb5` | 读取 SOCD 数据区 | C |
| `GetTapDance` | `0xb8` | 读取 Tap Dance 数据 | B，受版本门槛控制 |
| `GetTGL` | `0xbb` | 读取 TGL 数据 | B，受版本门槛控制 |
| `GetMacro` | `0xc2` | 读取宏区 | B |
| `GetLightCount` | `0xd1` | 读取灯位数量 | B |
| `GetKeyLightColor` | `0xd2` | 读取按键灯颜色 | B |
| `GetLightState` | `0xd5` | 读取灯光状态 | A |
| `GetKeyboardFunc` | `0xe1` | 读取模式功能配置 | B |
| `GetTouchBarConfig` | `0xe6` | 读取触控条配置 | C，Kick75 未启用 |
| `GetSleepInfo` | `0xf3` | 读取休眠配置 | B |
| `GetAppDefineSize` | `0xfa` | 读取 App 自定义区大小 | B |
| `GetAppDefine` | `0xfb` | 读取 App 自定义数据 | B |
| `GetDebugEnable` | `0xfd` | 读取调试开关 | C |

### 5.2 写入和破坏性命令

| 命令 | 值 | 作用 | 风险 |
| --- | ---: | --- | --- |
| `SetUseKeys` | `0xb3` | 写入当前键位矩阵片段 | 高 |
| `RestoreUseLayers` | `0xb4` | 恢复层映射 | 高 |
| `SetSOCD` / `ResetSOCD` | `0xb6` / `0xb7` | 写入/清除 SOCD 数据 | 高 |
| `SetTapDance` / `ResetTapDance` | `0xb9` / `0xba` | 写入/清除 Tap Dance | 高 |
| `SetTGL` / `ResetTGL` | `0xbc` / `0xbd` | 写入/清除 TGL | 高 |
| `SetKeyUpload` | `0xc1` | 控制键状态上传 | 中；改变运行状态 |
| `SetMacro` | `0xc3` | 写入宏区 | 高 |
| `SetLightState` | `0xd6` | 写入灯光状态 | A；仅侧灯片段已验证 |
| `SetKeyboardFunc` | `0xe2` | 写入模式功能配置 | 高 |
| `SetDebounceTime` / `TestDelayTime` | `0xe3` / `0xe4` | 防抖/测试参数 | 高，Kick75 未确认开放 |
| `SetTouchBarConfig` | `0xe5` | 写入触控条配置 | 高，Kick75 未启用 |
| `SetSecretKey` | `0xee` | 建立临时会话掩码 | A；协议握手 |
| `SetIapMode` | `0xef` | 进入固件升级模式 | 极高，禁止探测 |
| `RestoreFactory` | `0xf1` | 恢复出厂 | 极高，禁止探测 |
| `SetSleepCfg` | `0xf5` | 写入休眠配置 | 高 |
| `SetAppDefine` | `0xfc` | 写入 App 自定义区 | 高 |
| `SetDebugLog` | `0xfe` | 修改调试日志状态 | 中 |

设备还会上报异步事件：`ModeStateChange=0xa2`、`KeyUpload=0xc1`、`LightStateChange=0xd7`、`KeyboardReset=0xf2`、`SleepCfgChange=0xf4`、`LogUpload=0xfe`。

## 6. 官方客户端 API 清单

以下是与 Kick75 扩展最相关的 `window.keyboardApi` 能力。它们是比裸命令更合适的本地驱动接口边界。

### 6.1 设备和状态读取

- `getBaseInfo`、`getFirmwareInfo`、`getBatteryState`
- `getKeyboardFunc`、`getLightStates`、`getKeysLight`
- `getDefaultKeyboardMatrix`、`getUseKeyboardMatrix`
- `getAllDefaultKeyboardMatrix`、`getAllKeyRTInfo`、`getKeyRTInfo`
- `getAllModeMacroInfo`、`getFreeMacroCount`
- `getReminderTime`、`getAutoSelectKeyRT`

### 6.2 键位与高级行为

- `setKey`、`changeKey`、`restoreKey`、`restoreLayer`、`restoreLayers`
- `setMTKey`、`setTGLKey`、`setTapDance`
- `setSOCDKey`、`setRSKey`、`setHTKey`、`setDKSKey`
- `setKeyRT`、`setKeysRT`

其中 MT/TGL/Tap Dance 对 Kick75 有直接产品意义；SOCD/RS/HT/DKS/RT 属于通用类暴露能力，必须先证明 Kick75 页面实际开放且固件响应正常，不能直接进入产品 UI。

### 6.3 宏

- `saveMacro`、`setMacroKey`、`deleteMacro`
- `getModeMacroInfo`、`getAllModeMacroInfo`
- `setMacroRecordState`

### 6.4 灯光、功耗和系统行为

- `setLightState`、`getCustomLight`、`setCustomLight`
- `setDisableWindows`、`setDisableAltTab`、`setDisableAltF4`
- `setSleepEnable`、`setShallowSleepTime`、`setDeepSleepTime`
- `setGameOptimization`、`setStabilizationLevel`

### 6.5 配置生命周期

- `exportModeConfig`、`importModeConfig`
- `resetMode`、`resetKeyboard`
- `enterIap`

`importModeConfig`、任何 reset 和 `enterIap` 都不应与状态灯守护进程共用默认权限。

## 7. 读写边界和推荐模块划分

为控制风险，本地实现应拆为三个权限不同的 Unix 风格模块：

```text
kick75-inspect（默认，只读）
  ├─ 设备/固件/电池
  ├─ 模式、层、键位和旋钮映射
  ├─ 灯光、休眠和宏元数据
  └─ 原始配置快照导出

kick75-config（显式授权，有限写入）
  ├─ 单键/旋钮映射
  ├─ 灯光
  └─ 休眠设置

kick75-maintenance（隔离，不进入常驻进程）
  ├─ 宏数据写入
  ├─ 恢复层/模式/出厂
  └─ 固件升级
```

任何写入流程都必须满足：读取完整原始区 → 保存带固件版本的快照 → 只改目标片段 → 回读比对 → 失败时只恢复同版本快照。不能用官方虚拟默认数据代替用户当前配置做回滚。

## 8. 面向“旋钮调整 reasoning effort”的最小实现路径

第一阶段不需要实现全部 NuPhyIO：

1. 只读实现 `GetBase`、`GetFirmwareInfo`、`GetUseKeys`，解析当前模式、层和索引 14/15/16。
2. UI 显示三个位置的当前键码，并允许用户选择一个不会与现有映射冲突的宿主快捷键。
3. 写入前导出整个当前键位区，而不是只保存旋钮三个键码。
4. 通过 `SetUseKeys` 只写当前模式、当前层、目标旋钮位置对应的最小片段。
5. 回读确认后，由 macOS 宿主监听该快捷键并切换 Codex reasoning effort。

键盘固件本身不知道 Codex reasoning effort。合理边界是“旋钮发出稳定键码，macOS 应用把键码解释成 Codex 操作”，不要把 Codex 领域逻辑塞进 HID 协议层。

## 9. 尚未闭合的证据

- 当前真机固件版本与 TGL、Tap Dance、扩展媒体键版本门槛的实际匹配尚未记录。
- `GetUseKeys` 的 Kick75 真机响应、各模式/层地址计算和 KeyInfo 完整解码尚未在仓库中形成可复现证据。
- 宏区、App 自定义区、休眠区的精确地址布局尚未逐字段命名。
- U1 接收器协议与有线键盘协议不是同一通道；当前灯光真机证据仍只覆盖 USB Raw HID。
- 通用 API 中的 MT、SOCD、RS、HT、DKS、RT 等高级键，哪些会在 Kick75 当前固件页面实际显示，仍需按设备与固件版本逐项确认。

因此，“全部解析”在当前阶段应理解为：已完成官方客户端对 Kick75 暴露能力、底层命令空间、数据区和风险边界的静态地图；尚未宣称所有字段都经过真机行为验证。

## 10. 一手来源

- NuPhyIO 官方 Web 驱动：<https://drive.nuphyio.com/>
- 官方主包快照：<https://drive.nuphyio.com/static/js/main.f6f60294.js>
- 官方设备目录接口：<https://drive.nuphyio.com/prod-api/api/nuphyIo/keyBoardList>
- 官方 Kick75 产品页：<https://nuphy.com/products/nuphy-kick75>
- 本仓库既有真机协议证据：[PROTOCOL.md](PROTOCOL.md)
- 本仓库官方客户端静态分析脚本：`research/inspect-nuphy-light-api.mjs`、`research/extract-webpack-module.mjs`

官方客户端关键证据位置：命令枚举 Webpack 模块 `40877`；Kick75 虚拟设备模块 `69798`（动态 chunk `417.392552f3.chunk.js`）；Kick75 能力对象位于主包设备配置表；旋钮三个特殊位置位于主包 Kick75 布局配置。
