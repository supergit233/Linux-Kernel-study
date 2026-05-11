# Linux 内核启动与移植主线

## 导读

### 本章定位

这一章承接 `U-Boot`，重点看 Linux 内核从入口到驱动模型、设备树、用户态准备完成的主线。

### 核心对象

- `start_kernel()`
  - Linux 内核主入口
- `setup_arch()`
  - 架构级初始化与设备树入口
- `initcall`
  - 各级驱动初始化顺序
- `device tree`
  - 板级硬件描述
- `platform_driver / bus / driver model`
  - 设备与驱动匹配框架

### 关键函数

- `start_kernel()`
- `setup_arch()`
- `mm_init()`
- `rest_init()`
- `kernel_init()`
- `do_basic_setup()`
- `do_initcalls()`

### 主流程

内核入口 -> 架构初始化 -> 内存管理建立 -> 中断/时钟/调度初始化 -> 驱动模型与 initcall -> 设备树驱动匹配 -> 根文件系统准备。

## 1. 内核层在移植里负责什么

- 建立 MMU 与内存管理
- 建立中断和异常处理
- 解析设备树
- 初始化总线和驱动模型
- 让关键驱动 probe
- 为根文件系统挂载和 `init` 运行做好准备

## 2. 启动主链

```text
内核入口
-> start_kernel()
-> setup_arch()
-> mm_init()
-> do_basic_setup()
-> do_initcalls()
-> rest_init()
-> kernel_init()
-> prepare_namespace()
```

## 3. 移植时最容易出的问题

- `dtb` 不匹配：驱动不 probe
- 内存描述错误：早期崩溃、随机异常
- 定时器/时钟不工作：系统卡顿、延时异常
- 中断控制器配置错误：驱动 probe 了但中断不来
- 控制台参数错误：内核在跑但看不到日志

## 4. 与板级 Bring-up 的关系

内核移植里最先稳定的通常是：

- 控制台
- 内存
- 定时器
- 中断控制器
- 存储

只有这些底盘稳定以后，后面的网卡、显示、音频、外设才有可靠基础。

## 5. 一句话总结

内核移植的核心，不是“编过”，而是让设备树、内存、中断、驱动模型这些基础层全部接上板级硬件。
