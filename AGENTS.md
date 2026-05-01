# AsteRx-m3 Pro+ Carrier Board Agent Notes

## Project Scope

- This repository documents the AsteRx-m3 Pro+ carrier board design boundary, component selections, and schematic handoff.
- The source-of-truth documents are `AsteRx-m3_ProPlus_底板设计规范.md`, `AsteRx-m3_ProPlus_器件选型评审_定稿.md`, `AsteRx-m3_ProPlus_器件选型清单_定稿.csv`, and `asterx_m3_proplus_carrier_feature_boundary.csv`.
- Treat vendor PDFs as local references only. They are ignored by git; use official Septentrio documentation for final authority.

## Frozen Design Decisions

- No PoE, no USB Host, and no Type-A connector.
- DC input is 12V nominal and must tolerate a 24V wrong adapter through the protection and wide-input buck design.
- DC input protection main part is `TPS26631RGER / C1850273`. `TPS26633RGER` is not the main choice for this project.
- DC path: DC input -> front-end PTC/TVS/CMC/MOS protection -> TPS26631 protection -> LM76002 buck -> `5V_DC` -> TPS2121 mux -> `5V_SYS`.
- USB path: USB-C VBUS -> TPS259472 -> `5V_USB_PROT` -> TPS2121 mux -> `5V_SYS`.
- AsteRx rail: `5V_SYS` -> TPS7A8300 -> `3V3_ASTERX`.
- Antenna feed is currently generated as fixed `4.2V_ANT`; verify the schematic link to AsteRx `VANT` before PCB handoff.
- COM1 uses `RSM232` isolated RS232 and COM2 uses `TD301D485H-A` automatic-direction isolated RS485 on one high-density D-Sub connector.
- RJ45 main part is `HR911130A / C54408`, used only as a 10/100M RMII MagJack interface with the selected PHY; no PoE.
- Interface board connection is a 12P 0.5mm FPC and carries only low-power human-interface signals.
- TNC antenna shield/chassis is separated from internal `I/O_GND`/`PGND`; it is not a `CHASSIS-PGND` hard-bond point.
- PPS/REF SMA connectors stay together on the interface side; their shells bond to the metal chassis and to `I/O_GND`/`PGND`.
- The SMA/USB/DC interface area is the only `CHASSIS-PGND` hard-bond area.
- USB Shield connects to `CHASSIS`; its ESD/common-mode current path should return through the metal shell/interface area and not through the RF area.
- `DC-` still passes through the input CMC. The DC connector shell may connect to `CHASSIS`, but `DC-` must not hard-bond to `CHASSIS` before the CMC.
- Do not create `CHASSIS-PGND` hard bonds in the buck area, RF front-end area, or TNC antenna area.

## EasyEDA Rules

- Follow the local `easyeda-schematic` skill before editing schematics.
- Before re-reading the full EasyEDA project through API in a new conversation, check `easyeda_exports/AGENT_NETLIST_NOTES.md` and the latest exported netlist files first. Treat them as an offline cache; refresh them after schematic edits or before final connectivity claims.
- Do not use schematic net ports. Use wires plus net labels; only GND may use the ground symbol.
- NC pins use no-connect markers directly, without net labels.
- Use a 0.05 inch grid. Keep wire stubs and placement aligned to that grid.
- Place each IC's support resistors, capacitors, ferrites, and pull components close to that IC.
- Schematic comments, page notes, and design intent text should be in Chinese.
- Do not run one large page-generation script. Break EasyEDA operations into small tasks, such as one IC placement, one IC pin breakout, or one local passive group.
- When the user has authorized subagents for schematic review, use a subagent as a real-time placement/layout checker before moving to the next task.
