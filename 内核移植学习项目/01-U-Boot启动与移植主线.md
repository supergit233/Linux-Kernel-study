# U-Boot 启动与移植主线

## 导读

### 本章定位

这一章先看 Linux 启动前的第一层：`U-Boot` 怎样完成早期硬件初始化、镜像加载和启动参数组织，并把控制权交给 Linux 内核。

### 核心对象

- `board_init_f()`
  - 早期初始化入口
- `board_init_r()`
  - 重定位后的主初始化入口
- `gd_t`
  - `U-Boot` 的全局数据结构
- `bootcmd / bootargs`
  - 启动命令与内核参数
- `Image / zImage / uImage / dtb`
  - 交给 Linux 的镜像与硬件描述

### 关键函数

- `board_init_f()`
- `board_init_r()`
- `relocate_code()`
- `env_init()`
- `run_command()`
- `bootm` / `bootz`

### 主流程

早期 SRAM 阶段 -> 时钟/串口/DDR 初始化 -> 代码重定位到 DDR -> 初始化存储、环境变量、命令框架 -> 读取镜像和 `dtb` -> `bootm/bootz` 跳入内核。

## 新人先读：本章先认识这些概念

| 概念 | 先这样理解 |
| --- | --- |
| BootROM | 芯片上电后最先运行的固化代码，负责把第一段可执行程序从启动介质搬出来 |
| SPL / TPL | 完整 U-Boot 之前的小引导阶段，常见任务是把 DDR 初始化好 |
| U-Boot proper | 完整版 U-Boot，能执行命令、读环境变量、加载内核和设备树 |
| SRAM | 片内小内存，容量小但上电即可用，早期引导常先依赖它 |
| DDR | 板上主内存，初始化成功后才能放下完整 U-Boot 和 Linux |
| `board_init_f()` | U-Boot 早期 C 阶段入口，重点做最基础硬件初始化 |
| `board_init_r()` | U-Boot 重定位到 DDR 后的主初始化入口 |
| `bootcmd` | U-Boot 自动执行的启动命令脚本 |
| `bootargs` | U-Boot 传给 Linux 的参数字符串 |
| `dtb` | 设备树二进制文件，用来告诉 Linux 当前板子有哪些硬件 |

## 这一章按什么逻辑展开

这一章按“上游入口 -> U-Boot 自身初始化 -> 镜像加载 -> 参数组织 -> 跳入 Linux”的顺序展开。

这样拆的原因是：U-Boot 移植不是只改一个 `bootcmd`，而是要保证最早硬件环境可运行，并且把 Linux 需要的 image、`dtb`、`bootargs` 放到正确位置。

## 1. `U-Boot` 在移植里解决什么问题

- 最早的串口输出
- DDR 初始化
- 时钟和复位初始化
- 存储介质读取
- 环境变量与启动命令
- 把内核、`dtb`、可选 `initramfs` 放到正确内存位置

## 2. 入口从哪里来

从移植视角看，`board_init_f()` 不是凭空执行的。更完整的上游链路是：

```text
SoC reset / BootROM                  // 上电复位后，芯片内部 BootROM 先运行
-> 选择启动介质                       // BootROM 根据 strap/eFuse/启动脚选择 NAND/eMMC/SD/SPI 等介质
-> 加载 SPL/TPL 或 U-Boot 本体         // 把第一段可执行镜像搬到片上 SRAM 或约定地址
-> 进入 U-Boot 早期入口                // 架构启动代码建立最小 C 运行环境
-> board_init_f()                    // 开始板级早期初始化，串口/时钟/DDR 是重点
```

如果使用 SPL/TPL，链路会多一层：

```text
BootROM                              // 芯片固化启动代码
-> SPL/TPL                           // 更小的早期引导阶段，常负责 DDR 初始化
-> U-Boot proper                     // 完整 U-Boot，负责命令、环境变量、镜像加载
-> bootm / bootz                     // 最终跳入 Linux
```

因此，串口完全没输出时，不应直接怀疑 Linux；它通常还停在 BootROM、SPL/TPL、最早串口、时钟或 DDR 阶段。

## 3. 最核心的两段流程

### 3.1 `board_init_f()`

这一段通常还在早期内存环境里运行，重点是：

- 建最基础运行环境
- 开串口
- 初始化时钟
- 初始化 DDR

### 3.2 `board_init_r()`

完成重定位后，开始进入完整 `U-Boot` 运行阶段：

- 初始化控制台
- 初始化存储设备
- 读取环境变量
- 进入自动启动命令

## 4. U-Boot 交给 Linux 的关键交接物

| 交接物 | 来源/填充时机 | Linux 如何消费 | 调试检查点 |
| --- | --- | --- | --- |
| kernel image | `bootcmd` 从 flash/eMMC/SD/TFTP 等介质加载到内存 | `bootm` / `bootz` 跳转到内核入口 | 地址是否覆盖 `dtb`、initramfs 或 U-Boot 自身 |
| `dtb` | U-Boot 从存储或网络加载，也可能由环境变量指定地址 | Linux 早期解析设备树，创建内存描述、chosen、platform device 等 | `fdt addr`、`fdt print`、板型是否匹配 |
| `bootargs` | U-Boot 环境变量、脚本或板级代码拼接 | Linux 解析 `console=`、`root=`、`init=` 等参数 | `printenv bootargs`，确认控制台和 rootfs 参数 |
| initramfs | 可选，由 U-Boot 加载或打包进内核 | Linux 作为早期根文件系统使用 | 地址、大小、是否和 image/dtb 冲突 |

这一层最重要的是：U-Boot 不负责 Linux 驱动 probe，但它决定 Linux 能否拿到正确镜像、正确设备树和正确启动参数。

## 5. 移植时真正容易卡住的点

- 串口不通：早期时钟、pinmux、UART 基址
- DDR 不稳：训练参数、时序、驱动强度
- 存储不通：SPI NAND/eMMC/SD 初始化
- 启动命令错误：镜像地址、`dtb` 地址、根文件系统参数

## 6. 全流程总结

```text
上电复位                              // SoC 复位后进入 BootROM 或固定启动入口
-> BootROM 选择启动介质                // 根据启动配置找到 SPL/TPL 或 U-Boot 镜像
-> SPL/TPL 或 U-Boot 早期入口           // 建立最小运行环境，可能仍在 SRAM 中
-> board_init_f()                     // 初始化最早串口、时钟、DDR 等基础资源
-> relocate_code()                    // 把 U-Boot 搬到 DDR 中运行
-> board_init_r()                     // 初始化控制台、存储、环境变量、命令框架
-> env_init() / printenv              // 准备 `bootcmd`、`bootargs` 等启动变量
-> run_command(bootcmd)               // 执行自动启动命令，加载 image/dtb/initramfs
-> bootm / bootz                      // 校验镜像并准备跳转参数
-> 跳入 Linux                         // 把控制权交给内核入口，下一章从这里接上
```

## 7. 一句话总结

`U-Boot` 移植的核心目标不是“功能很多”，而是“能稳定把下一层交给 Linux 内核”。
