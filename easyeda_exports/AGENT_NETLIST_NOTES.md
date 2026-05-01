# EasyEDA 网表离线交接说明

## 用途

本目录保存当前 EasyEDA 原理图网表快照，供新对话或后续 agent 先离线阅读。除非需要最新原理图状态、坐标/摆放信息、视觉检查或 DRC 结果，否则不要一开始就重新调用 EasyEDA API 扫描整套工程。

## 快照来源

- 项目：`AsteRXm3ProPlus底板`
- 板卡：`底板v1`
- 原理图：`Schematic1`
- 导出时间：`2026-05-02 02:25:42 +08:00`
- 可用 API：`window._EXTAPI_ROOT_.sch_ManufactureData.getNetlistFile(fileName, netlistType)`
- 已验证导出类型：`JLCEDA`、`EasyEDA`、`Allegro`、`PADS`、`Protel2`
- 不推荐慢接口：`window._EXTAPI_ROOT_.sch_Netlist.getNetlist('JLCEDA')`，该封装会做额外 footprint 名称查询，在本工程上曾超过 30 秒超时。

## 文件选择

- `AsteRXm3ProPlus_carrier_netlist_2026-05-02.asc`
  - 首选离线网表。
  - 格式：PADS ASCII。
  - 内容：完整器件表和网络连接表。
  - 解析方式：读取 `*PART*` 段得到 `Designator -> Footprint`，读取 `*NET*` 段下每个 `*SIGNAL* <net>` 及后续 `Designator.Pin` token 得到连接。
- `AsteRXm3ProPlus_carrier_netlist_2026-05-02.net`
  - 详细离线网表。
  - 格式：Protel2。
  - 适合查看更完整的器件属性、器件元数据和冗长连接信息。
- `AsteRXm3ProPlus_carrier_netlist_2026-05-02.enet`
  - EasyEDA/JLCEDA 辅助导出。
  - 注意：此文件看起来像 JSON，但在当前桥接工作流下不应作为唯一依据；直接 `JSON.parse` 看到的 component 段不完整。保留它用于与 EasyEDA/JLCEDA 原生导出对照。
- `agent_netlist_summary_2026-05-02.json`
  - 面向 agent 的摘要索引。
  - 含器件/网络数量、器件前缀统计、最大网络和重点网络的 pin 列表。

## 当前摘要

- 器件数：`149`
- 网络数：`188`
- 命名网络：`164`
- 自动生成网络：`24`
- 器件前缀统计：`U=16`、`Q=2`、`C=50`、`R=37`、`L=8`、`F=4`、`DC=1`、`D=11`、`TP=2`、`J=10`、`FB=1`、`USB=1`、`CARD=1`、`DSUB=1`、`TV=2`、`SW=1`、`P=1`

## 重点网络检查提示

- `GND`：141 pins。
- `PGND`：18 pins，包含 USB、microSD、RJ45/DSUB 屏蔽及若干 TVS/接口相关 pin。
- `5V_DC`：13 pins。
- `5V_SYS`：9 pins。
- `5V_USB_PROT`：6 pins。
- `3V3_ASTERX`：3 pins，见 `J30.1`、`J30.2`、`FB1.1`。
- `4.2V_ANT`：3 pins，见 `U16.5`、`C49.1`、`C50.1`。
- `VANT`：当前摘要中只见 `J30.18`，未与 `4.2V_ANT` 同网；PCB/原理图交付前仍需重点确认。
- `USB_DP`/`USB_DN`：各 2 pins，连接 `J30` 与 `L3`。
- `ETH_TXP`、`ETH_TXN`、`ETH_RXP`、`ETH_RXN`：各 4 pins，连接 PHY、MagJack、匹配/ESD 器件。
- `PPS1`、`PPS2`：各 1 pin，当前仅出现在 `J30`。
- `REF_IN`、`REF_OUT`、`CHASSIS`、`I/O_GND`：当前 PADS 摘要中未出现同名网络；若后续任务涉及这些网络，必须回 EasyEDA 或最新网表确认。
- `RS485_A_DB9`、`RS485_B_DB9`：各 5 pins。
- `RS232_RXD_DB9`、`RS232_TXD_DB9`：各 3 pins。

## 新对话使用规则

1. 先读本文件和 `agent_netlist_summary_2026-05-02.json`。
2. 需要完整连接时读 `AsteRXm3ProPlus_carrier_netlist_2026-05-02.asc`。
3. 需要器件属性时读 `AsteRXm3ProPlus_carrier_netlist_2026-05-02.net`。
4. 若 EasyEDA 原理图在本快照后被修改，先重新导出网表，再基于新文件判断连接。
5. 网表只能证明连接关系和器件清单；不能替代原理图视觉检查、摆放坐标检查、线段是否等长、文字/网络标签是否重叠、DRC 结果或 EasyEDA 页面布局检查。
