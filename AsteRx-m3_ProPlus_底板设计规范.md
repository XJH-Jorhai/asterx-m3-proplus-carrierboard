# AsteRx-m3 Pro+ 底板设计规范

版本：V0.5  
日期：2026-04-28  
依据文件：

- `asterx-m3_product_group_hardware_manual_2.5.0.pdf`
- `Septentrio_AsteRx-m3_ProPlus_LR.pdf`
- `asterx_m3_proplus_carrier_feature_boundary.csv`
- 接收机权限文件 `lif, permissions` 输出，序列号 `4006604`

## 1. 设计目标

本项目设计一套基于 Septentrio AsteRx-m3 Pro+ OEM 模块的底板系统，用于将模块的电源、通信、授时、日志、指示和控制接口转换为整机可用接口。

整机由三部分组成：

| 编号 | 组件 | 设计边界 |
|---|---|---|
| 1 | AsteRx 主板 | Septentrio AsteRx-m3 Pro+ OEM 模块本体，不修改模块硬件 |
| 2 | AsteRx 底板 | 承载 Hirose 30-pin/60-pin 连接器、电源转换、USB、以太网PHY、串口转换、MicroSD、时频接口、保护和机械固定 |
| 3 | 按键接口板 | 通过线束连接底板，承载 reset、LOG Button、LOGLED、GPLED、电源/待机开关等低速人机接口 |

## 2. 权限与功能确认

当前设备权限文件确认以下功能可用：

| 功能 | 权限状态 | 设计结论 |
|---|---:|---|
| USB | `usb=1` | USB Device 可正式设计 |
| Ethernet | `ethernet=1` | RJ45 以太网可正式设计，但底板必须实现 RMII PHY |
| Logging | `logging=1` | MicroSD 日志功能可正式设计 |
| TimedEvent | `timedevent=1` | EventA/EventB 时间标记可正式设计 |
| TimeSync | `timesync=1` | Event 输入可作为外部 PPS 同步源 |
| FreqSync | `freqsync=1` | 10MHz REF IN 可正式设计 |
| 双天线 | `nb_ant=2`, `attitude=2` | 模块支持双天线姿态，本版 MAIN 与 AUX1 均需通过 MMCX-SMA 转接线引出 |
| 最大数据率 | `datarate=100` | 支持最高 100Hz 输出 |

不纳入本版设计的授权或功能：CAN、UHF、INS、PPP、Wide-Area Differential。  
其中 CAN 和 UHF 虽在权限文件中出现，但当前硬件手册的 30-pin/60-pin pinout 未给出明确物理引脚，因此本版不设计对应硬件接口。

## 3. 总体实现边界

### 3.1 正式纳入功能

| 功能域 | 功能项 | 实现边界 |
|---|---|---|
| 电源 | USB-C、DC 端子两路供电输入 | 两路输入经防反灌选择后生成 5V 母线，再经 LDO/低噪声后级生成模块 3.3V |
| 电源 | VANT 天线供电 | 通过编码/拨码选择 3.3V 或 5V，送入 30-pin `VANT` |
| USB | USB Device Type-C | 同时用于 USB Device 通信和系统 5V 输入路径 |
| 以太网 | RJ45 网口 | 底板实现 RMII PHY、磁性器件/RJ45 和 ESD；不实现 PoE |
| 存储 | MicroSD 卡座 | 用于模块 logging 功能 |
| 串口 | COM1-RS232、COM2-RS485、COM3/COM4-LVTTL GH1.25 | 见第 8 节串口分配 |
| 时频 | 10MHz REF IN | 外部 SMA/BNC 输入，做阻抗和幅度适配 |
| 时频 | PPS1/PPS2 | SMA 引出，支持跳线/0R选择 |
| 时频 | REF OUT | 由模块板上 u.FL 通过同轴引出到 SMA |
| 时频 | EventA/EventB | EventA/EventB 均由 SMA 引出，同时通过逻辑芯片接 GH1.25 输入，允许 SMA 或 GH1.25 触发 |
| 指示/控制 | reset、LOG、GPLED、LOGLED、电源开关 | 通过按键接口板实现 |
| 机械 | MAIN/AUX1 天线面板引出 | 模块 MAIN/AUX1 均使用 MMCX 到 SMA 转接线引出 |
| EMC/ESD | 外部接口保护 | USB、RJ45、串口、SMA、SD、电源口均需保护 |

### 3.2 本版不实现

| 功能 | 边界说明 |
|---|---|
| USB Host / Type-A | 明确不设计、不预留；底板仅保留 USB-C Device 接口 |
| CAN | 权限允许但当前 pinout 未确认，不设计 |
| UHF Radio | 权限允许但当前 pinout 未确认，不设计 |
| INS/IMU 融合 | 权限不允许，不设计 |

## 4. 电源设计规范

### 4.1 输入电源

底板支持两类输入，两路输入必须互相防反灌：

| 输入 | 功能 | 设计要求 |
|---|---|---|
| USB-C VBUS | USB Device 通信口，同时作为 5V 输入源之一 | 经输入保护/限流后进入 5V 电源选择节点；同时连接模块 `USB_VBUS` 检测 |
| DC 端子 | 外部 DC 输入 | 标称 12V 输入；电源前端和 DC/DC 需支持至少 24V 容错输入，先转换到 5V，再进入 5V 电源选择节点 |

DC 端子按 12V 标称适配器使用场景设计，但必须能承受 24V 误插而不损坏后级器件。该能力应通过合适的宽压 DC/DC 和输入保护实现，不能单纯依赖 TVS 钳位。输入侧仍需具备过压、过流、反接和浪涌/ESD 保护。

### 4.2 两路输入防反灌与电源路径

用户给出的电源路径按能量流向整理如下：

```text
USB-C VBUS 5V -> 输入保护/限流 -> 5V_SYS -> LDO/低噪声后级 -> AsteRx 3.3V Vin
DC端子12V标称/24V容错 -> 宽压DC/DC电源模块 -> 5V_SYS -> LDO/低噪声后级 -> AsteRx 3.3V Vin
```

两路 5V 来源必须通过电源 OR-ing、理想二极管、电源 MUX 或电源模块自带防反灌能力隔离。任一路输入存在时，不得向另一路输入端反灌。

不能将 5V_SYS 或 USB-C VBUS 直接接到 30-pin Pin 1/2 `Vin`，也不能用 30-pin Pin 7 `USB_VBUS` 替代模块主电源输入。

推荐策略：

- USB-C 与 DC/DC 输出统一在 5V_SYS 层级参与电源选择。
- 电源优先级按 5V_SYS 入口电压自动选择，默认电压较高的一路接管系统供电。
- 如需固定优先级，应在电源 MUX 中显式实现，例如 DC端子 > USB-C。
- USB-C VBUS 仍连接到模块 `USB_VBUS` 检测路径，用于 USB Device 枚举。
- 模块主电源始终由底板稳压后的 `3.3V` 提供。

### 4.3 AsteRx 模块主电源

| 项目 | 要求 |
|---|---|
| 模块输入 | 30-pin Pin 1/2 `Vin` |
| 电压 | `3.3V ±5%` |
| 峰值能力 | 至少 `1A` |
| 接地 | 所有 GND pin 必须连接到底板地 |
| 稳压建议 | DC/DC 后接低噪声后级，保证 GNSS 性能 |

### 4.4 外围电源

底板需为以下外围器件提供电源预算，并分割敏感电源域：

- Ethernet PHY
- RS232/RS485 收发器
- USB Type-C 保护/配置电路
- MicroSD 卡座
- 时频 buffer / level shifter
- LED 和按键接口板

外围器件的输出不得在模块未上电或 standby 时驱动模块输入。所有会驱动模块输入的器件必须由 `IO_EN` 或等效电源有效信号控制。

RS232、RS485、时频逻辑和其他外部 I/O 不应共用同一个未分割电源支路。推荐至少划分：

| 电源域 | 供电对象 | 要求 |
|---|---|---|
| `3V3_ASTERX` | AsteRx 模块 Vin | 低噪声，独立滤波，优先保证 GNSS 性能 |
| `3V3_PHY` | Ethernet PHY/RMII 相关逻辑 | 与模块 3.3V 分开滤波 |
| `VCC_RS232` | COM1 RS232 收发器 | 独立 LDO 或负载开关，输出受 `IO_EN`/PGOOD 约束 |
| `VCC_RS485` | COM2 RS485 收发器 | 独立 LDO 或负载开关，输出受 `IO_EN`/PGOOD 约束 |
| `VCC_EVENT` | Event 输入逻辑芯片/施密特触发器 | 低抖动、低噪声，延迟可评估 |
| `VCC_SD` | MicroSD | 独立滤波，避免 SD 时钟噪声进入 GNSS/时频区域 |

若 RS485 需要隔离，应使用独立隔离电源模块和数字隔离器。

### 4.5 VANT 天线供电

| 项目 | 要求 |
|---|---|
| 模块引脚 | 30-pin Pin 18 `VANT` |
| 电压选择 | 编码/拨码/跳线选择 `3.3V` 或 `5V` |
| 范围 | `3-5.5V` |
| 限流 | 模块天线端每路最大 `150mA` |
| 保护 | 建议增加滤波、限流、TVS/浪涌保护和测试焊盘 |

`VANT` 同时供 MAIN 和 AUX1。本版 MAIN/AUX1 均引出到面板 SMA，因此 `VANT` 电压选择会同时影响两路有源天线。

## 5. AsteRx 模块连接器边界

### 5.1 30-pin 主连接器

30-pin 是必须连接的主接口。底板至少使用以下信号：

| 信号 | Pin | 用途 |
|---|---:|---|
| `Vin` | 1, 2 | 模块 3.3V 主电源 |
| `USB_D+` / `USB_D-` | 5 / 6 | USB Device |
| `USB_VBUS` | 7 | USB VBUS 检测；不能替代 Pin 1/2 `Vin` 作为模块主电源输入 |
| `nRST` | 8 | 复位，接按键接口板 |
| `TX1/RX1` | 9 / 10 | COM1，默认用于 RS232 |
| `PPS1` | 12 | PPS1 输出 |
| `TX2/RX2` | 13 / 14 | COM2，默认用于 RS485 |
| `TX3/RX3` | 15 / 16 | COM3，GH1.25 LVTTL 直出 |
| `VANT` | 18 | 天线供电输入 |
| `EventA` | 19 | Event1 外部输入 |
| `nPDN` | 20 | standby / 电源开关控制 |
| `PPS2` | 21 | PPS2 输出 |
| `GPLED` | 22 | 接按键接口板 |
| `Button` | 25 | LOG Button，接按键接口板 |
| `SD_CLK/SD_CMD/SD_DAT0` | 26 / 28 / 30 | MicroSD |
| `LOGLED` | 27 | 接按键接口板 |

### 5.2 60-pin 扩展连接器

60-pin 用于以太网、COM4、EventB、REF IN、IO_EN 和部分 GPIO：

| 信号 | Pin | 用途 |
|---|---:|---|
| `GP1` | 9 | GH1.25 引出 |
| `RTS2/CTS2` | 11 / 12 | COM2 流控或保留 |
| `RTS3/CTS3` | 13 / 14 | COM3 流控或保留 |
| `TX4/RX4` | 15 / 16 | COM4 GH1.25 LVTTL 直出 |
| RMII signals | 31-40, 54, 56 | Ethernet PHY |
| `GP2` | 44 | GH1.25 引出 |
| `EventB` | 57 | Event2，SMA 和 GH1.25 双输入逻辑触发 |
| `IO_EN` | 59 | 外部驱动使能 |
| `REF IN` | 60 | 10MHz 外部参考输入 |

Reserved 引脚保持悬空，除非后续资料明确说明。

## 6. USB 设计规范

### 6.1 USB Device Type-C

USB-C 口作为 AsteRx 的 USB Device 接口，同时可作为系统供电输入之一。

| 项目 | 要求 |
|---|---|
| 数据线 | 连接 30-pin `USB_D+` / `USB_D-` |
| VBUS | 连接 30-pin `USB_VBUS` 检测，并进入底板电源 MUX/OR-ing 作为系统供电来源之一 |
| 角色 | UFP / Device |
| 保护 | USB ESD + 共模扼流圈 |
| 走线 | USB 2.0 差分阻抗控制，短路径，连续参考地 |

USB-C VBUS 可作为 AsteRx 模块的供电来源，但必须先经过底板电源选择、保护和 3.3V 稳压后送入 30-pin Pin 1/2 `Vin`。30-pin Pin 7 `USB_VBUS` 只作为 USB VBUS 检测脚，不能替代 `Vin`。

### 6.2 USB Host / Type-A

本版明确不设计、不预留 USB Host 和 Type-A 接口。USB 相关硬件仅实现 USB-C Device。

## 7. 以太网设计规范

模块仅提供 RMII 接口，底板必须实现完整 10/100M 以太网物理层。本版不实现 PoE。

| 项目 | 要求 |
|---|---|
| 模块侧 | 60-pin RMII 接口 |
| 底板侧 | RMII PHY + 带灯内置变压器RJ45 |
| 参考设计 | 可参考手册中的 KSZ8041 方案；使用其他 PHY 需确认兼容性 |
| 供电 | PHY 所需电源由底板提供 |
| 保护 | RJ45 端加 ESD/浪涌保护 |
| 走线 | RMII 短且参考地连续；PHY 靠近模块；RJ45 靠近板边 |

RJ45 不作为供电输入，不能通过网口给底板供电。RJ45 具体料号后续统一选型；当前选型方向为普通带灯、内置变压器 RJ45。若统一选型选择千兆 RJ45，也仅按 10/100M 以太网使用其所需线对。

## 8. 串口设计规范

### 8.1 当前需求

当前串口分配按 4 路 AsteRx 原生 COM 一一对应实现：

### 8.2 默认推荐分配

| 模块串口 | 底板接口 | 连接器/形态 | 说明 |
|---|---|---|---|
| COM1 | RS232 x1 | 弯插 DB9 | 通过 RS232 收发器转换，收发器受 `IO_EN` 控制 |
| COM2 | 隔离 RS485 x1 | 弯插 DB9 | 半双工 RS485；隔离电源和数字隔离必须实现，方向控制策略原理图阶段确认 |
| COM3 | LVTTL 直出 x1 | GH1.25 | 3.3V LVTTL，直接引出 |
| COM4 | LVTTL 直出 x1 | GH1.25 | 3.3V LVTTL，直接引出 |

本版不再设计第二路 RS485。COM3、COM4 均为 3.3V LVTTL 原始串口，通过 GH1.25 直接引出。

### 8.3 串口电气要求

- 模块串口均为 3.3V LVTTL，空闲高。
- RS232/RS485 收发器不得在模块未上电或 standby 时驱动模块 RX 输入。
- 建议所有串口外部接口加 ESD 保护。
- RS485 必须隔离；终端电阻、偏置电阻和方向控制策略在原理图阶段确认。
- 若使用自动方向 RS485 收发器，需验证最高波特率和 turnaround 时序。
- RS232 与 RS485 使用不同电源芯片或独立负载开关，形成 `VCC_RS232` 与 `VCC_RS485` 两个电源域。

## 9. 时频接口设计规范

### 9.1 10MHz REF IN

| 项目 | 要求 |
|---|---|
| 模块引脚 | 60-pin Pin 60 `REF IN` |
| 频率 | 10MHz |
| 输入形式 | 正弦或方波 |
| 模块侧幅度 | `2-5Vpp` |
| 模块侧输入 | AC 耦合，约 `13kΩ` 输入阻抗 |
| 权限 | `freqsync=1` 已确认 |

外部接口建议使用 SMA 或 BNC，并按 50Ω 时频接口布线。若外部源为标准 50Ω 正弦源，需设计幅度适配/整形，保证进入模块 pin 的信号满足 `2-5Vpp`。

重要约束：

- 若使用 REF IN，10MHz 必须在接收机启动前已经存在。
- 运行期间 10MHz 必须持续存在。
- REF IN 丢失或启动时未稳定可能导致接收机工作异常或需要重启。

### 9.2 10MHz REF OUT

`REF OUT` 位于 AsteRx 模块板上的 u.FL 连接器，不在 30-pin/60-pin 连接器中。

| 项目 | 要求 |
|---|---|
| 输出 | 10MHz 方波 |
| 电平 | `0-2.8V` |
| 输出阻抗 | `50Ω` |
| 默认状态 | 默认开启，可由命令关闭 |
| 引出方式 | u.FL 转 SMA |

若需要将 REF OUT 与 PPS 共用一个外部 SMA，必须使用跳线或 0R 电阻做互斥选择，不允许两个源直接相连。当前确认 REF OUT 采用 u.FL 转 SMA。

### 9.3 PPS 输出

| 信号 | 模块引脚 | 外部形态 | 说明 |
|---|---:|---|---|
| PPS1 | 30-pin Pin 12 | SMA | 正式引出 |
| PPS2 | 30-pin Pin 21 | SMA 或跳线选择 | 正式引出或与 PPS1 选择输出 |

要求：

- LVTTL 输出，输出阻抗 50Ω，输出电流 24mA。
- 默认脉宽 5ms，可由命令配置。
- 上电最初几秒电平未定义。
- 若一分多路或面板长线输出，建议加低抖动 buffer。

### 9.4 Event 输入

| 信号 | 模块引脚 | 本版实现 |
|---|---:|---|
| EventA / Event1 | 30-pin Pin 19 | SMA 输入，同时 GH1.25 输入经逻辑芯片触发同一 EventA |
| EventB / Event2 | 60-pin Pin 57 | SMA 输入，同时 GH1.25 输入经逻辑芯片触发同一 EventB |

EventA 和 EventB 均不连接按键接口板。

每路 Event 输入推荐结构：

```text
SMA输入 -> ESD/限幅/整形 -> 逻辑芯片
GH1.25输入 -> ESD/限幅/整形 -> 逻辑芯片
逻辑芯片输出 -> AsteRx EventA/EventB
```

逻辑芯片采用 OR 逻辑。要求：

- SMA 或 GH1.25 任一输入触发时，对应 Event 可被模块识别。
- 两个输入不得直接短接，必须通过逻辑/缓冲隔离。
- 若 Event 用作 TimeSync 精确输入，逻辑芯片延迟和抖动必须固定、可评估，并在系统标定中记录。
- GH1.25 输入若来自外部机械触点，需在外部或底板侧增加去抖；若用于精确同步，不得经过慢速去抖电路。

## 10. 存储与日志

底板实现 MicroSD 卡座。

| 项目 | 要求 |
|---|---|
| 信号 | `SD_CLK`, `SD_CMD`, `SD_DAT0` |
| 模式 | 1-bit SD |
| 电平 | 3.3V |
| 最高时钟 | 33MHz |
| 文件系统 | FAT32 |
| 容量 | 最大 32GB |

LOG Button 接按键接口板：

- 短按：切换 logging。
- 长按至少 5s：挂载/卸载 SD。
- 按键需外部消抖。

断电前应先卸载 SD，避免最后几秒数据丢失。

## 11. 按键接口板设计规范

按键接口板只承载低速人机接口，不承载 USB、Ethernet、RMII、REF IN 等高速或时频关键链路。

### 11.1 接口内容

| 信号 | 来源/去向 | 功能 |
|---|---|---|
| `nRST` | 30-pin Pin 8 | 复位按键，低有效 |
| `nPDN` | 30-pin Pin 20 | standby / 电源开关控制，低有效 |
| `Button` | 30-pin Pin 25 | LOG Button，低有效 |
| `LOGLED` | 30-pin Pin 27 | SD 挂载/记录状态 |
| `GPLED` | 30-pin Pin 22 | 定位/通用状态灯 |
| Power LED / Switch LED | 底板电源状态 | 带灯电源开关指示 |

### 11.2 带灯电源开关

用户候选料号 `C2761316?` 仅表示电源开关的大致规格形态，不作为最终料号。本规范仅定义功能边界：

- 电源开关优先作为系统使能或 standby 控制，不建议直接承载 DC 输入主电流，除非所选开关的电压、电流、浪涌和寿命规格已确认。
- 开关 LED 建议由 `POWER_GOOD`、`SYS_5V` 或受控 LED 电源驱动。
- 若开关 LED 与触点不独立，需按实际开关内部连接重新设计驱动方式。
- 最终选型必须确认安装孔、灯色、LED 额定电压/电流、触点形式、额定电压电流和寿命。

## 12. RF 与天线机械边界

### 12.1 MAIN 天线

MAIN 天线为必选。采购模块的 RF 连接器类型确认为 MMCX，模块未焊接 IPEX/u.FL 端子。底板和外壳需给模块 MAIN MMCX 连接器留出装配空间，并通过 MMCX 到 SMA 转接线引至面板 SMA。

要求：

- RF 阻抗 50Ω。
- 有源天线供电由 `VANT` 提供。
- 天线净增益范围按手册要求控制。
- 天线同轴和 SMA 周边远离高速数字和开关电源。

### 12.2 AUX1 天线

AUX1 天线同样正式引出。底板和外壳需给模块 AUX1 MMCX 连接器留出装配空间，并通过 MMCX 到 SMA 转接线引至面板 SMA。

要求：

- MAIN 和 AUX1 均不通过 Hirose 连接器传输 GNSS RF。
- MAIN/AUX1 使用同规格 MMCX-SMA 转接线。
- 面板 SMA 布局需保证两根同轴不与高速数字、电源电感或 DC/DC 电源区域交叉贴近。
- 若使用双天线姿态，MAIN/AUX1 天线净增益差建议不超过 10dB。

## 13. GPIO 与指示

| 信号 | 实现 |
|---|---|
| `GP1` | GH1.25 引出 |
| `GP2` | GH1.25 引出 |
| `GPLED` | 接按键接口板 |
| `LOGLED` | 接按键接口板 |

GP1/GP2 为 LVTTL 输出，最大驱动电流 10mA，上电初期为三态。若外部设备依赖默认电平，需加上拉/下拉。

## 14. PCB 与 EMC 设计要求

建议使用 4 层或以上 PCB：

- L1：器件与关键信号
- L2：完整 GND
- L3：电源与低速信号
- L4：信号与接口

关键约束：

- 模块安装孔用金属支柱接地，但不能替代 I/O GND。
- 30-pin 和 60-pin 封装、keepout、底部高度限制按手册执行。
- USB、Ethernet、PPS、REF、Event、RF 同轴接口均需阻抗和回流路径控制。
- 开关电源、电感、以太网 PHY、USB、SD 等数字/开关噪声源远离天线和 REF/PPS/Event。
- 外部接口必须加 ESD/浪涌保护。
- GNSS 性能验收应观察 C/N0 和 spectrum，确认底板未引入明显干扰。

## 15. 测试点策略

CSV 中写明“测试点不需要设计考虑”。本规范按产品外部接口不设置专用调试排针处理。

风险提示：

- 首版调试会更难定位 3.3V、VANT、PHY 电源、IO_EN、PPS、REF IN、USB、RMII 等问题。
- 若完全不放测试焊盘，返板风险上升。

建议折中方案：

- 不放外部调试排针。
- 在 PCB 内部保留少量裸焊盘测试点，不作为产品接口。
- 至少保留：`3V3`, `VANT`, `GND`, `IO_EN`, `nRST`, `PPS1`, `REF_IN`, `USB_D+`, `USB_D-`, `RMII_CLK`。

若用户坚持完全不做测试点，应在设计评审中单独确认风险接受。

## 16. 验收与上电流程

### 16.1 不装模块检查

- DC 输入保护和稳压输出正常。
- USB-C 与 DC端子两路输入按 5V_SYS 入口电压自动选择或按电源 MUX 固定优先级选择，且任一路径不反灌另一路输入。
- 3.3V、5V、VANT 无短路。
- 收发器、PHY、SD、LED 电源轨正常。

### 16.2 装模块后检查

- 30-pin Pin1/2 上电为 `3.3V ±5%`。
- `IO_EN` 上电后变高。
- `nRST`、`nPDN`、LOG Button 动作正确。
- USB-C 能枚举 AsteRx 设备。
- 串口 RS232/RS485/LVTTL 能通信。
- RJ45 能建立 10/100M 链路。
- MicroSD 可挂载和记录。
- PPS1/PPS2 输出正确。
- EventA/EventB 可触发事件。
- 外部 10MHz REF IN 存在时接收机正常使用外部参考。
- REF OUT 由模块板载 REF OUT 连接器到 SMA 输出正常。

### 16.3 GNSS 性能检查

- MAIN 天线接入后能正常跟踪卫星。
- C/N0 与开阔环境预期一致。
- 使用 RxControl/RxTools 检查 spectrum，无明显由底板产生的干扰峰。
- 使用 USB-C 与 DC端子两种供电模式分别验证，并验证两路同时接入时不会反灌。

## 17. 已确认约束与后续选型项

| 编号 | 事项 | 当前处理 |
|---|---|---|
| C1 | DC端子输入 | 标称 12V，需支持 24V 误插容错；通过宽压 DC/DC 实现，不单靠 TVS |
| C2 | 以太网RJ45/magnetics选型 | 后续统一选型；当前方向为普通带灯、内置变压器 RJ45，不实现 PoE |
| C3 | USB Host / Type-A | 已确认不设计、不预留 |
| C4 | REF OUT | 已确认 u.FL 转 SMA |
| C5 | 带灯电源开关 | `C2761316?` 仅作规格形态参考，最终料号后续确认 |
| C6 | RS232/RS485连接器 | COM1 RS232 与 COM2 RS485 均使用弯插 DB9 |
| C7 | RS485隔离 | RS485 必须隔离；终端电阻、偏置电阻和方向控制在原理图阶段确认 |
| C8 | EventA/EventB逻辑 | SMA 与 GH1.25 输入采用 OR 逻辑 |
