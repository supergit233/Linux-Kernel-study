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

## 1. `U-Boot` 在移植里解决什么问题

- 最早的串口输出
- DDR 初始化
- 时钟和复位初始化
- 存储介质读取
- 环境变量与启动命令
- 把内核、`dtb`、可选 `initramfs` 放到正确内存位置

## 2. 最核心的两段流程

### 2.1 `board_init_f()`

这一段通常还在早期内存环境里运行，重点是：

- 建最基础运行环境
- 开串口
- 初始化时钟
- 初始化 DDR

### 2.2 `board_init_r()`

完成重定位后，开始进入完整 `U-Boot` 运行阶段：

- 初始化控制台
- 初始化存储设备
- 读取环境变量
- 进入自动启动命令

## 3. 移植时真正容易卡住的点

- 串口不通：早期时钟、pinmux、UART 基址
- DDR 不稳：训练参数、时序、驱动强度
- 存储不通：SPI NAND/eMMC/SD 初始化
- 启动命令错误：镜像地址、`dtb` 地址、根文件系统参数

## 4. 最短主链

```text
上电
-> board_init_f()
-> DDR 初始化
-> 代码重定位
-> board_init_r()
-> 读取环境变量
-> bootcmd
-> bootm / bootz
-> 跳入 Linux
```

## 5. 一句话总结

`U-Boot` 移植的核心目标不是“功能很多”，而是“能稳定把下一层交给 Linux 内核”。
