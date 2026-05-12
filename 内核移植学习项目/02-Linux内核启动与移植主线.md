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

## 新人先读：本章先认识这些概念

| 概念 | 先这样理解 |
| --- | --- |
| `start_kernel()` | Linux 通用启动主入口，可以把它理解成内核世界的“主函数” |
| `setup_arch()` | 架构相关初始化入口，内存布局、设备树早期解析常在这里接上 |
| 设备树 / DTS / DTB | DTS 是文本描述，DTB 是编译后的二进制；Linux 用它知道板上有什么硬件 |
| `bootargs` | U-Boot 传来的命令行参数，Linux 用它决定控制台、rootfs、init 等行为 |
| driver model | Linux 管理设备、驱动、总线的通用框架 |
| platform device | 设备树里很多板载硬件节点，进入 Linux 后会变成这种 device |
| platform driver | 和 platform device 匹配的驱动 |
| `probe()` | device 和 driver 匹配成功后，内核调用驱动的初始化函数 |
| initcall | 内核启动时按等级执行的初始化函数机制，很多内建驱动靠它注册 |

## 这一章按什么逻辑展开

这一章按“U-Boot 交接 -> 内核早期入口 -> 架构和设备树初始化 -> initcall 和驱动模型 -> rootfs 准备”的顺序展开。

这样拆的原因是：内核启动失败时，不能只看 `start_kernel()`，还要先确认 U-Boot 传进来的 `dtb`、`bootargs`、内存布局是否正确；驱动不 probe 时，也要回到设备树和 initcall 顺序。

## 1. 内核层在移植里负责什么

- 建立 MMU 与内存管理
- 建立中断和异常处理
- 解析设备树
- 初始化总线和驱动模型
- 让关键驱动 probe
- 为根文件系统挂载和 `init` 运行做好准备

## 2. 入口从哪里来：U-Boot 跳入 Linux

内核入口不是自己触发的，它来自上一章 `bootm` / `bootz` 的最后一步：

```text
bootm / bootz                         // U-Boot 准备 kernel image、dtb、bootargs
-> 跳转到内核镜像入口                  // 架构相关入口接管 CPU
-> 解压或搬移内核                      // `zImage` 等形式可能先经历解压阶段
-> 架构早期启动代码                    // 建立最早页表、栈和 C 运行环境
-> start_kernel()                     // 进入通用 Linux 内核主入口
```

如果“U-Boot 正常，跳内核后完全无输出”，优先怀疑的是这段交接链：

- kernel image 地址或格式不对
- `dtb` 地址不对或被覆盖
- `bootargs` 中 `console=` 不对
- 早期解压、MMU、内存布局异常

## 3. 启动主链

```text
内核入口                              // U-Boot 跳转进来的架构入口
-> start_kernel()                     // 通用内核主入口，开始初始化核心子系统
-> setup_arch()                       // 架构初始化，解析设备树和内存布局
-> mm_init()                          // 建立内存管理基础
-> init_IRQ() / time_init()           // 建立中断、时钟和定时器基础
-> do_basic_setup()                   // 初始化 driver model、bus、class 等基础框架
-> do_initcalls()                     // 按 initcall 等级调用内建驱动初始化函数
-> rest_init()                        // 创建内核线程并进入后续初始化
-> kernel_init()                      // 继续完成用户态启动前准备
-> prepare_namespace()                // 准备挂载 rootfs，下一章从这里接上
```

## 4. 设备树和 bootargs 被谁消费

| 输入 | 来源 | 内核消费位置 | 后续影响 |
| --- | --- | --- | --- |
| `dtb` | U-Boot 加载并传给内核 | 架构早期和 `setup_arch()` 阶段解析 | 生成内存节点、`chosen`、CPU、interrupt-controller、timer、platform device 等 |
| `bootargs` | U-Boot 环境变量或 `chosen.bootargs` | 内核参数解析路径消费 | 决定 `console=`、`root=`、`init=`、调试参数和部分驱动行为 |
| `memory` 节点 | DTS 中描述，也可能被 U-Boot 修正 | 早期内存初始化消费 | 影响可用内存范围、保留内存和早期崩溃风险 |
| `compatible` | DTS 节点声明硬件身份 | platform/OF match 消费 | 决定对应 platform driver 是否能 probe |
| `interrupts` / clock / reset | DTS 节点描述板级资源 | IRQ、clock、reset 子系统和具体驱动消费 | 决定驱动 probe 后能否正常工作 |

这也是内核移植里最常见的判断线：设备没 probe，先看 `compatible` 和节点状态；probe 了但工作异常，再看 `reg`、`interrupts`、clock、reset、pinctrl、regulator 等资源。

## 5. initcall 到驱动 probe 的关系

内建驱动不是随机开始 probe 的，而是先通过 initcall 注册 driver，再由 driver model 和设备树生成的 device 匹配。

```text
do_initcalls()                       // 内核按等级执行 initcall
-> driver init function              // 某个总线或 platform driver 注册入口被调用
-> platform_driver_register()        // driver 注册到 platform bus
-> of_platform_populate()            // 设备树节点被转换成 platform_device
-> platform_match() / of_match       // device 的 compatible 和 driver of_match_table 对比
-> driver.probe()                    // 匹配成功后进入具体驱动 probe
```

所以“驱动不 probe”不能只看驱动源码，还要回头检查：

- DTS 节点是否存在并 `status = "okay"`
- `compatible` 是否和 driver 的 `of_match_table` 一致
- driver 是否被编进内核或模块是否加载
- 依赖的 bus、clock、reset、pinctrl、regulator 是否准备好

## 6. 移植时最容易出的问题

- `dtb` 不匹配：驱动不 probe
- 内存描述错误：早期崩溃、随机异常
- 定时器/时钟不工作：系统卡顿、延时异常
- 中断控制器配置错误：驱动 probe 了但中断不来
- 控制台参数错误：内核在跑但看不到日志

## 7. 与板级 Bring-up 的关系

内核移植里最先稳定的通常是：

- 控制台
- 内存
- 定时器
- 中断控制器
- 存储

只有这些底盘稳定以后，后面的网卡、显示、音频、外设才有可靠基础。

## 8. 全流程总结

```text
U-Boot bootm/bootz                   // 上一层把 image、dtb、bootargs 准备好
-> kernel entry                      // CPU 跳入内核架构入口
-> start_kernel()                    // 通用内核初始化开始
-> setup_arch()                      // 消费 dtb、内存描述和架构参数
-> parse_args()                      // 消费 bootargs，得到 console/root/init 等参数
-> mm_init() / init_IRQ() / time_init() // 建立内存、中断、时钟基础
-> driver model init                 // 建立 device/driver/bus/class 等模型
-> do_initcalls()                    // 内建驱动按 initcall 等级注册
-> of_platform_populate()            // 把设备树节点变成 platform_device
-> driver probe                      // device 与 driver 匹配后进入具体 probe
-> prepare_namespace()               // 准备 rootfs 挂载，进入下一章
```

## 9. 一句话总结

内核移植的核心，不是“编过”，而是让设备树、内存、中断、驱动模型这些基础层全部接上板级硬件。
