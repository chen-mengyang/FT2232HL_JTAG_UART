# FT2232HL_JTAG_UART

基于 FTDI FT2232HL 双通道 USB 桥接芯片的下载调试工具：一端通过 USB Type-C 连电脑，另一端用一根 2.54mm 12P 排针（H1）同时引出 JTAG 与 TTL 串口——A 通道 JTAG（配合 OpenOCD 对 HPM5E/HPM6E 开发板下载调试，支持 Segger Embedded Studio 工程），B 通道 TTL 串口（3.3V 电平，用于日志打印与命令交互）。归入协议转换与工具线。

> 状态：设计中（原理图与 PCB 已按嘉立创 EDA 工程核对，接口定义同步更新；软硬件接口已逐脚核对）｜文档更新：2026-11-11

- 产品线：协议转换与工具（C）
- 产品编码：FT2232HL_JTAG_UART（C=协议转换与工具线，FT2232HL=主控芯片，JTAG_UART=两路功能规格码）
- 适用客户：使用 HPM5E/HPM6E（先楫）系列 MCU 的研发工程师、开发板配套用户、产线烧录场景
- 供电方式：USB 总线供电，无需外接电源

## 功能特性

| # | 功能 | 说明 |
| --- | --- | --- |
| 1 | USB 上位机接口 | Type-C 16P 座（CC 带 2×5.1K 下拉），USB 2.0 高速，总线供电 |
| 2 | JTAG 下载通道 | A 通道 MPSSE（TCK/TDI/TDO/TMS 硬件固定），引至 H1 排针，配合 OpenOCD 下载调试 |
| 3 | TTL 串口通道 | B 通道独立 TTL 串口（3.3V），FTDI VCP 虚拟 COM 口，TXD/RXD 引至 H1 排针 |
| 4 | 复位输出 | nTRST（ACBUS1）与 nSRST（ACBUS0）各经 10K 汇聚至 H1 单一复位脚 |
| 5 | 双通道并行 | JTAG 下载与串口打印同时使用，一根 USB 线完成烧录 + 调试日志 |
| 6 | 流控预留 | RTS/CTS（BDBUS2/3）、DTR（BDBUS4）网络已定义，未引出排针 |
| 7 | EEPROM 配置 | 实焊 93LC56B，可经 FTDI MPROG 自定义 VID/PID 与通道模式 |

工作原理：FT2232HL 单芯片集成两个相互独立的 USB 通道——A 通道的 MPSSE 引擎在硬件上产生 JTAG 时序（TCK/TMS/TDI/TDO），OpenOCD 通过 FTDI 驱动连接 HPM5E/HPM6E 目标芯片，并对 Segger Embedded Studio / GDB 提供 GDB Server，完成固件下载与在线调试；B 通道桥接为标准 UART，TTL 电平（3.3V）直出排针，电脑侧枚举为虚拟串口。两路通道互不影响，可同时工作。

应用场景：HPM5E/HPM6E 开发板的固件烧录与在线调试（OpenOCD + JTAG）；Segger Embedded Studio 工程的下载调试链路；目标板 TTL 串口日志查看、命令行交互；一根 USB 线同时完成「下载 + 串口打印」的研发与产线场景。

## 硬件接口（对外接口与接线）

### 通道划分

| 通道 | 引脚组 | 功能 | 工作模式 |
| --- | --- | --- | --- |
| A 通道 | ADBUS0～3 + ACBUS0/ACBUS1 | JTAG + 复位输出 | MPSSE |
| B 通道 | BDBUS0/BDBUS1（预留 BDBUS2/3/4） | TTL 串口（虚拟 COM 口） | UART |

两通道 VCCIO 均接 3.3V，全部 IO 为 3.3V TTL 电平；USB 总线供电。

### 供电要求

| 项目 | 规格 |
| --- | --- |
| 供电方式 | USB 总线供电（VBUS），无需外接电源 |
| 供电链路 | VBUS → F3 自恢复保险丝（BSMD0603-050-6V，500mA）→ +5V → LDO1（ME6211C33M5G-N）→ +3.3V |
| 内核电源 | 1.8V 由芯片内部 LDO 自产（VREGIN→VREGOUT→VCORE），无需外置 1.8V 轨 |

### 对外接口总览

H1（2.54mm 12P，A254W12-0B12P51）为整机唯一对外接口，JTAG 与串口合用一根排针（已按 PCB 网络表核对）：

| H1 引脚 | 网络 | 功能 | 对接 EVK |
| --- | --- | --- | --- |
| 1 / 2 / 4 | GND | 地 | J10 偶数脚 |
| 3 | FT_TDO | JTAG 数据输出（收目标板） | J10 pin13 |
| 5 | FT_TCK | JTAG 时钟 | J10 pin9 |
| 7 | FT_TMS | JTAG 模式选择 | J10 pin7 |
| 9 | FT_TDI | JTAG 数据输入（送目标板） | J10 pin5 |
| 6 | FT_UART_RX | 串口接收（接目标 TXD） | P5 pin10（UART4.RXD） |
| 8 | FT_UART_TX | 串口发送（接目标 RXD） | P5 pin8（UART4.TXD） |
| 10 / 12 | +3.3V | VTREF / IO 电平输出 | J10 pin1（VTREF，板内 R31 默认 DNP） |
| 11 | RST | 复位输出（nTRST、nSRST 各经 10K 汇聚） | J10 pin3（TRST） |

复位说明：nTRST（ACBUS1，脚27，hpm_sdk 官方 cfg 位 0x0200）经 R10 10K、nSRST（ACBUS0，脚26，官方 cfg 位 0x0100）经 R9 10K，汇聚到 H1 pin11 单一脚；不接复位时 OpenOCD 用 TMS 序列复位 TAP。

### 接线步骤

1. 断开板载调试器：拔掉 EVK 上 J17 全部 5 个跳线帽（出厂 All Jumpers are ON ＝ 板载 FT2232 接管 JTAG），严禁与板载 FT2232 同时驱动 JTAG；用完恢复跳线可继续使用板载调试器；
2. USB 端：Type-C 接口连接电脑，确认设备枚举；
3. JTAG 对接：H1 的 TCK(pin5)/TMS(pin7)/TDO(pin3)/TDI(pin9) 逐根接 EVK J10 座 pin9/pin7/pin13/pin5，复位脚 H1 pin11 接 J10 pin3(TRST)，VTREF 用 H1 pin10/12 接 J10 pin1，GND 接 J10 偶数脚；
4. 串口对接：H1 pin6(FT_UART_RX) 接目标 P5 pin10(UART4.RXD)，H1 pin8(FT_UART_TX) 接目标 P5 pin8(UART4.TXD)，并共地；
5. 电平与共地：串口为 TTL 电平（3.3V），勿直连 RS232 或 5V 系统；JTAG、串口与目标板必须共地；EVK 板内每线已串 100Ω（R32～R36），TCK/TMS 板内有 10K 上拉（R38/R39）。

### 指示与保护

| LED | 接法 | 含义 |
| --- | --- | --- |
| LED2 | +3.3V → LED → R15 → GND | 电源指示，上电常亮 |
| LED3 | +3.3V → LED → R11（1.5K）→ FT_UART_TX | 串口发送指示，线为低时点亮 |
| LED1 | +3.3V → LED → R12（1.5K）→ FT_UART_RX | 串口接收指示 |

- 电源保护：VBUS 经 F3 自恢复保险丝（500mA）限流；D1（USBLC6-4SC6Y）作差分线 + VBUS ESD 防护；
- 串口线上拉由 LED+1.5K 路径提供（弱上拉），与 EVK 板载 10K 直接上拉方案不同，如目标板无上拉建议实测确认空闲电平稳定。

## 软件接口与协议

### FT2232HL 侧引脚分配（LQFP-64）

A 通道 JTAG（MPSSE 硬件固定，不可换脚；JTAG 四线直连排针，无串阻）：

| 脚号 | 引脚名 | 本产品信号 | 方向（dongle 视角） | 说明 |
| --- | --- | --- | --- | --- |
| 16 | ADBUS0 | FT_TCK | 输出 | MPSSE 硬件固定 |
| 17 | ADBUS1 | FT_TDI | 输出 | — |
| 18 | ADBUS2 | FT_TDO | 输入 | — |
| 19 | ADBUS3 | FT_TMS | 输出 | — |
| 27 | ACBUS1 | FT_HRST（nTRST） | 输出 | 经 R10 10K 至 H1 pin11 |
| 26 | ACBUS0 | FT_NRST（nSRST） | 输出 | 经 R9 10K 至 H1 pin11 |

B 通道串口：

| 脚号 | 引脚名 | 本产品信号 | 方向（dongle 视角） | 说明 |
| --- | --- | --- | --- | --- |
| 38 | BDBUS0 | FT_UART_TX | 输出 | 经 LED3+R11（1.5K）接 3.3V，发送指示兼弱上拉 |
| 39 | BDBUS1 | FT_UART_RX | 输入 | 经 LED1+R12（1.5K）接 3.3V，接收指示兼弱上拉 |
| 40 | BDBUS2 | FT_UART_RTS | 输出 | 预留，未引出排针 |
| 41 | BDBUS3 | FT_UART_CTS | 输入 | 预留，未引出排针 |
| 43 | BDBUS4 | FT_UART_DTR | 输出 | 预留，未引出排针 |

默认 3 线制（TXD/RXD/GND），电脑侧枚举为 FTDI 虚拟 COM 口（VCP 驱动），常用 115200bps、8N1。

### 电源、USB 与系统引脚

| 脚号 | 引脚名 | 说明 |
| --- | --- | --- |
| 7 / 8 | DM / DP | 经 L1 共模电感（DLW21SN900SQ2L）接 USB 座；D1 作 ESD 防护 |
| 2 / 3 | OSCI / OSCO | 12MHz 晶振 X1（XL2EL89CSI）+ C1/C2 两颗 22pF 负载电容 |
| 50 / 49 | VREGIN / VREGOUT | 内部 1.8V LDO：VREGIN 接 3.3V，VREGOUT→+1.8V（C12/C13/C14 滤波） |
| 12 / 37 / 64 | VCORE | 内核电源 1.8V |
| 4 / 9 | VPHY / VPLL | 接 3.3V（按 DS_FT2232H 手册，未采用 EVK 的 1.8V 方案） |
| 10 / 6 | AGND / REF | REF（脚6）经 R3 12K 接 GND |
| 20 / 31 / 42 / 56 | VCCIO | 四个 IO 电源脚均接 3.3V（漏接任一脚芯片不工作） |
| 14 | RESET\# | R4 10K 上拉至 3.3V |
| 13 | TEST | 接 GND |
| 63 / 62 / 61 | EECS / EECLK / EED | 接 EEPROM U2 |
| 1 / 5 / 11 / 15 / 25 / 35 / 47 / 51 | GND | 地 |

### EEPROM 配置（实焊 93LC56B，U2）

| 脚号 | 引脚名 | 说明 |
| --- | --- | --- |
| 63 | EECS | 接 EEPROM CS，R7 4.7K 上拉至 3.3V |
| 62 | EECLK | 接 EEPROM CLK，R8 4.7K 上拉至 3.3V |
| 61 | EED | 直连 EEPROM DI；EEPROM DO 经 R6 2.2K 串阻回 EED，并 R5 4.7K 上拉至 3.3V |

用途：存储 USB VID/PID、序列号、产品描述字符串、各通道工作模式，可用 FTDI MPROG 经 USB 在线烧写；需 16 位组织 3V 版（93LC56B）。

### 未使用与预留引脚

- 悬空：ACBUS2～7（28～30、32～34）、ADBUS4～7（21～24）、BDBUS5～7（44～46）、BCBUS0～7（48、52～59）等；
- 预留未引出：FT_UART_RTS（BDBUS2）、FT_UART_CTS（BDBUS3）、FT_UART_DTR（BDBUS4），网络已命名；
- 原理图上另有 5×RH-5015（单脚，疑似测试点）、2×0Ω、1×ESD5Z5.0C，PCB 暂未放置，装焊时注意。

hpm_sdk 官方探针配置：boards/openocd/probes/ft2232.cfg（nTRST=0x0200/ACBUS1，nSRST=0x0100/ACBUS0，ftdi_layout_init 0x0208 0x020b）。

## 快速上手

1. 断开板载调试器：拔掉 EVK 上 J17 全部 5 个跳线帽；
2. 连接：USB 端接电脑；dongle H1 排针按上述引脚表逐根对接 EVK J10 座（TCK/TDI/TDO/TMS/TRST/VTREF/GND）与 P5 接口 pin8/pin10（UART4）、GND；
3. 装驱动：电脑安装 FTDI 驱动（VCP/D2XX），确认 B 通道虚拟 COM 口已枚举；
4. 串口通讯：用串口终端打开虚拟 COM 口（如 115200bps、8 数据位、1 停止位、无校验），查看目标板日志；
5. 下载调试：启动 OpenOCD（FTDI 接口配置 + HPM5E/HPM6E 目标配置），Segger Embedded Studio 通过 GDB Server 连接后下载固件、在线调试。

## 相关文档

设计文档位于原金山云文档同目录；本仓库 README 由「功能描述」与「软硬件接口说明」合并整理。
