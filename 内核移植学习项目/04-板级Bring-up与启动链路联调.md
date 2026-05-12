# 板级 Bring-up 与启动链路联调

## 导读

### 本章定位

这一章把 `U-Boot`、内核、根文件系统三层压到同一条板级联调路径里，重点看一块新板子如何一步步点亮。

### 核心对象

- 串口控制台
  - 最早可见调试窗口
- DDR
  - 启动链稳定运行的基础
- `dtb`
  - 板级硬件描述
- 存储介质
  - 内核和根文件系统来源
- `bootargs`
  - 把各层串起来的参数桥梁

### 关键函数

- `board_init_f() / board_init_r()`
- `bootm / bootz`
- `start_kernel()`
- `prepare_namespace()`
- `run_init_process()`

### 主流程

先通串口 -> 再稳 DDR -> 再让 `U-Boot` 能读镜像 -> 再让内核有日志 -> 再让根文件系统可挂载 -> 最后进入稳定用户态。

## 新人先读：本章先认识这些概念

| 概念 | 先这样理解 |
| --- | --- |
| Bring-up | 新板从完全不可用到能稳定启动、逐步点亮外设的过程 |
| 串口控制台 | 最早、最重要的日志窗口，没有它就像闭眼调板子 |
| pinmux | 管脚复用配置，同一个物理引脚可能被配置成 UART、GPIO、SPI 等不同功能 |
| clock | 时钟资源，外设没有正确时钟通常不能工作 |
| reset | 复位控制，外设可能需要先解除复位才能访问 |
| regulator | 电源资源，某些外设 probe 成功前必须先供电 |
| 存储介质 | 放 U-Boot、内核、设备树、rootfs 的地方，例如 eMMC、SD、NAND、SPI flash |
| defconfig | 一块板子的默认编译配置，决定 U-Boot 或 Linux 默认打开哪些功能 |
| Kconfig / Makefile | 把新板、新驱动、新对象接入构建系统的入口 |
| 板级目录 | U-Boot 中放新板私有初始化代码的目录，例如 `board/<vendor>/<board>/` |
| 成功信号 | 每一层能证明自己已通过的证据，例如串口有输出、DDR 内存测试稳定、rootfs 挂载成功 |

## 这一章按什么逻辑展开

这一章按“板级基础确认 -> Linux 内核源码配置/DTS -> U-Boot 启动交接 -> rootfs 三条挂载路线 -> 外设联调”的顺序展开。

这样拆的原因是：当前学习主源码是 HI3516CV610 使用的 `linux-5.10.y`，所以本章的现实例子优先围绕 Linux 的 `defconfig`、DTS、驱动和 rootfs 参数展开；U-Boot 仍然是启动链上一层，负责把 kernel image、`dtb`、`bootargs` 交给 Linux。

## 1. Bring-up 的推荐顺序

1. 串口
2. DDR
3. 时钟和复位
4. 存储设备
5. Linux 内核配置/DTS 与 `U-Boot` 启动交接
6. Linux 控制台
7. 设备树关键节点
8. rootfs 路线选择：initramfs、NFS root、本地存储
9. 关键外设驱动

## 2. 每一步要验证什么

| 步骤 | 验证目标 | 成功信号 | 失败优先归属 |
| --- | --- | --- | --- |
| 串口 | 最早日志通道可用 | 能看到 BootROM/SPL/U-Boot 或早期打印 | pinmux、clock、UART 基址、波特率 |
| DDR | 代码和数据能稳定运行在外部内存 | U-Boot 重定位后不随机崩溃，内存测试稳定 | DDR 初始化、训练参数、时序、电源 |
| 时钟和复位 | 基础外设可被访问 | 串口、存储、定时器行为稳定 | CRG、reset、时钟树、门控 |
| 存储设备 | 能加载 kernel image、`dtb`、rootfs | U-Boot 能读镜像，内核能识别块设备或 MTD | eMMC/SD/NAND/SPI、分区、驱动 |
| Linux 源码基线与 DTS/defconfig | 能确认当前板子使用哪份内核配置和哪份 `dtb` | 能编出目标 kernel image 和 `hi3516cv610-*.dtb` | defconfig、`.config`、DTS、DTS Makefile、驱动配置 |
| U-Boot 启动命令 | 能把交接物放到正确地址 | `bootm/bootz` 能跳入内核 | image/dtb/initramfs 地址、格式、覆盖 |
| Linux 控制台 | 内核日志可见 | 出现 `start_kernel()` 后续日志或驱动打印 | `console=`、DTS chosen、串口驱动 |
| 设备树关键节点 | platform device 和关键驱动能匹配 | 关键控制器 probe，资源解析成功 | `compatible`、`reg`、`interrupts`、clock、reset |
| rootfs 路线 | 内核能进入用户态 | initramfs、NFS root 或本地 rootfs 至少一条跑通 | `root=`、`rootfstype=`、`nfsroot=`、存储/网络驱动 |
| 关键外设驱动 | 业务相关硬件可工作 | 网络、USB、WiFi、显示等功能可用 | IRQ、DMA、clock、regulator、驱动私有初始化 |

## 3. 现实例子一：基于 HI3516CV610 Linux 5.10.y 的完整移植例子

这个例子以当前学习源码为基线：

```text
E:\linux_share\驱动学习\HI3516CV610-SIM-开发板资料\linux-5.10.y // 当前学习使用的 Linux 5.10.y 源码
```

本例假设要在 HI3516CV610 系列上移植一块新板。为了不用脑补占位符，下面直接使用这份 `linux-5.10.y` 里真实存在的 HI3516CV610 参考文件作为起点，再派生出一个具体的新板示例文件名。

目标不是“能敲一次 `bootz`”，而是形成完整闭环：

```text
Linux 源码基线确认                      // 明确当前学习源码是 linux-5.10.y
-> 选择最接近的参考板                    // 用已存在的 demb-emmc/demb-flash 降低移植变量
-> 复制出新板 DTS 和 defconfig            // 从真实参考文件派生，不从空文件开始
-> 编译 kernel image 和 dtb              // 生成 zImage/Image/uImage 以及对应 dtb
-> U-Boot 加载 image 和 dtb              // U-Boot 作为上一层负责搬运交接物
-> U-Boot 传入 bootargs                  // console=、root=、nfsroot=、rdinit= 等决定后半段
-> Linux 能出 console                   // image、dtb、bootargs 交接正确
-> rootfs 三条路线至少跑通一条           // initramfs、NFS root、本地存储按阶段选择
```

### 3.1 先选最接近的参考板，不要从零写

真实移植通常不是从空 DTS、空 defconfig 开始，而是先在内核源码里找一块最接近的新板参考对象。这样做的核心目的，是先继承一条已经跑通过的启动链，再逐步替换差异点。

选择参考板时按下面顺序判断：

- SoC 是否相同，最好同为 HI3516CV610，至少也要是同系列 SoC
- 启动介质是否接近，例如同样从 eMMC、NAND、SPI flash 或 SD 启动
- DDR、串口、时钟、复位这些早期资源是否已经在参考板上跑通过
- 关键外设是否接近，例如 MMC/SDIO、SPI flash、网口、USB、GPIO
- rootfs 路线是否接近，例如参考板已经有 eMMC rootfs、UBIFS rootfs 或 NFS root 调试基础

在这份 `linux-5.10.y` 里，HI3516CV610 已经有几组可选参考对象：

```text
arch/arm/configs/hi3516cv610_defconfig       // HI3516CV610 通用内核配置
arch/arm/configs/hi3516cv610_emmc_defconfig  // eMMC 启动/存储路线的内核配置
arch/arm/configs/hi3516cv610_nand_mini_defconfig // NAND/精简路线的内核配置
arch/arm/boot/dts/hi3516cv610.dtsi           // SoC 级公共硬件描述
arch/arm/boot/dts/hi3516cv610-demb.dts       // DEMO Board 公共板级描述
arch/arm/boot/dts/hi3516cv610-demb-emmc.dts  // eMMC 板级变体
arch/arm/boot/dts/hi3516cv610-demb-flash.dts // flash 板级变体
arch/arm/boot/dts/Makefile                   // 决定哪些 dtb 会被编译出来
```

假设新板也是 HI3516CV610，并且主要从 eMMC 启动，最合适的参考组合就是：

```text
arch/arm/configs/hi3516cv610_emmc_defconfig  // 参考 eMMC 版内核配置
arch/arm/boot/dts/hi3516cv610-demb-emmc.dts  // 参考 eMMC 版板级 DTS
arch/arm/boot/dts/hi3516cv610.dtsi           // 复用 HI3516CV610 SoC 公共描述
```

然后复制出新板自己的文件。这里用 `hi3516cv610-study-emmc` 作为完整移植例子的新板名：

```text
cd E:\linux_share\驱动学习\HI3516CV610-SIM-开发板资料\linux-5.10.y // 进入当前学习源码
cp arch/arm/boot/dts/hi3516cv610-demb-emmc.dts arch/arm/boot/dts/hi3516cv610-study-emmc.dts // 从真实 eMMC 参考板复制新板 DTS
cp arch/arm/configs/hi3516cv610_emmc_defconfig arch/arm/configs/hi3516cv610_study_emmc_defconfig // 从真实 eMMC 配置复制新板 defconfig
修改 arch/arm/boot/dts/Makefile               // 把 hi3516cv610-study-emmc.dtb 加入 CONFIG_ARCH_HI3516CV610 的 dtb 列表
```

这一步的意义是减少变量。新板先继承 `hi3516cv610-demb-emmc.dts` 和 `hi3516cv610_emmc_defconfig` 这条已经存在的参考链路，再逐步替换板级差异：串口、memory、eMMC/SDIO、SPI flash、网口、GPIO、设备树节点、内核配置和 rootfs 参数。这样每次只改变一类变量，问题出现时更容易判断是哪一层引入的。

### 3.2 先认识这份 Linux 内核源码里会改哪些位置

| 位置 | 作用 | 本项目里重点看什么 |
| --- | --- | --- |
| `arch/arm/configs/hi3516cv610*_defconfig` | 板级默认内核配置 | 当前走 eMMC、flash、NAND、debug 还是 mini 路线 |
| `.config` | 实际参与本次编译的配置结果 | `CONFIG_ARCH_HI3516CV610`、MMC、MTD、UBIFS、NFS、initramfs 是否打开 |
| `arch/arm/boot/dts/hi3516cv610.dtsi` | SoC 公共硬件描述 | CPU、GIC、clock、reset、UART、MMC、SPI、GPIO 等基础控制器 |
| `arch/arm/boot/dts/hi3516cv610-demb*.dts` | 板级差异描述 | memory、chosen、串口状态、eMMC/flash/NAND/SDIO 使能差异 |
| `arch/arm/boot/dts/Makefile` | dtb 构建入口 | `hi3516cv610-demb.dtb`、`hi3516cv610-demb-emmc.dtb`、`hi3516cv610-demb-flash.dtb` 是否被编译 |
| `drivers/vendor/mmc/` | HI3516 相关 MMC/SDIO 控制器驱动 | `sdhci_nebula.c`、`CONFIG_MMC_SDHCI_NEBULA`、SDIO/eMMC probe 路径 |
| `drivers/mtd/`、`fs/ubifs/` | flash/UBI/UBIFS 路线 | NAND/SPI flash、UBI attach、UBIFS rootfs |
| `net/`、`fs/nfs/` | NFS root 路线 | 网卡驱动和 NFS client 必须在挂 rootfs 前可用 |
| `init/`、`fs/` | rootfs 挂载主线 | `prepare_namespace()`、`mount_root()`、`run_init_process()` |

这份 Linux 源码里还会反复遇到几个核心对象：

| 对象 | 先这样理解 | 调试意义 |
| --- | --- | --- |
| `struct device_node` | 设备树节点在内核里的对象 | DTS 节点是否被 Linux 解析出来 |
| `struct platform_device` | 由设备树节点等来源生成的设备对象 | 节点是否变成可匹配的 Linux device |
| `struct platform_driver` | platform 总线上的驱动对象 | `of_match_table` 是否能匹配 `compatible` |
| `struct resource` | `reg`、`interrupts` 等资源的内核表示 | 地址、中断、内存资源是否解析正确 |
| `struct mmc_host` / `struct sdhci_host` | MMC/SDIO 控制器在内核里的核心对象 | eMMC、SD、SDIO 枚举失败时要回到这里 |
| `struct mtd_info` / UBI volume | flash 和 UBI/UBIFS 路线的核心对象 | NAND/SPI flash rootfs 失败时要看这一层 |

### 3.3 把新板接进 Linux 内核构建系统

复制出新板 DTS 和 defconfig 只是第一步，还要让内核构建系统真的知道这个新板。

先把新板 `dtb` 加到 `arch/arm/boot/dts/Makefile` 的 HI3516CV610 列表里。下面这段尽量按真实 Makefile 写法保留，不把解释性注释放进续行中：

```text
dtb-$(CONFIG_ARCH_HI3516CV610) += \
    hi3516cv610-demb.dtb \
    hi3516cv610-demb-flash.dtb \
    hi3516cv610-demb-emmc.dtb \
    hi3516cv610-study-emmc.dtb
```

这里前三个 `dtb` 是源码里已经存在的参考板产物，最后一行 `hi3516cv610-study-emmc.dtb` 是新板新增产物。

然后用新板 defconfig 生成 `.config`，再编译 kernel image 和 dtb：

```text
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- hi3516cv610_study_emmc_defconfig // 载入新板默认配置
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- olddefconfig                     // 让 Kconfig 补齐新增或缺省选项
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- zImage dtbs -j8                  // 生成 kernel image 和所有 HI3516CV610 dtb
```

如果本机实际安装的交叉编译器前缀不同，只替换 `CROSS_COMPILE=` 后面的前缀，不改变 `ARCH=arm`、defconfig 和 dtb 主线。

配置检查示意：

```text
CONFIG_ARCH_HI3516CV610=y             // 目标 SoC 平台必须正确
CONFIG_MMC_SDHCI_NEBULA=y             // eMMC/SDIO 控制器驱动进入内核
CONFIG_MTD=y                          // flash/NAND/SPI flash 路线需要
CONFIG_UBIFS_FS=y                     // UBIFS rootfs 路线需要
CONFIG_NFS_FS=y                       // NFS root 路线需要
CONFIG_BLK_DEV_INITRD=y               // 外部 initrd/initramfs 路线需要
```

如果这一步失败，优先看：

- 是否选错了 `hi3516cv610`、`hi3516cv610_emmc`、`hi3516cv610_nand_mini` 等 defconfig
- `arch/arm/boot/dts/Makefile` 里是否包含目标 `.dtb`
- 新 DTS 是否正确 `#include "hi3516cv610.dtsi"` 或参考板 DTS
- `CONFIG_ARCH_HI3516CV610`、存储驱动、rootfs 文件系统是否真的进了 `.config`
- 交叉编译器前缀是否正确

### 3.4 第一目标：先让 U-Boot 出串口

串口是第一条生命线。没有串口日志，后面 DDR、存储、Linux 都会变成盲调。

```text
U-Boot early entry                    // CPU 进入 U-Boot 早期代码
-> pinmux / clock                     // 让 UART 引脚和时钟可用
-> serial driver                      // U-Boot 串口驱动初始化
-> puts()/printf()                    // 串口能输出字符
-> U-Boot prompt                      // 能进入命令行交互
```

检查点：

- UART 基地址是否和原理图、手册一致
- pinmux 是否把引脚切到 UART 功能
- 串口时钟是否打开
- `stdout-path` 或 U-Boot console 配置是否指向正确串口
- 波特率是否和终端一致

### 3.5 第二目标：DDR 稳定和 U-Boot 重定位

U-Boot 早期可能先在 SRAM 中运行，但完整 U-Boot 和 Linux 都需要 DDR。DDR 不稳时，现象会非常随机。

```text
SPL/TPL 或 board_init_f()             // 早期阶段开始做 DDR 初始化
-> DDR controller init                // 配置控制器、时序、训练参数
-> dram_init()                        // U-Boot 记录 DRAM 大小
-> relocate_code()                    // U-Boot 从早期地址搬到 DDR
-> board_init_r()                     // 重定位后继续初始化
-> U-Boot prompt                      // 如果能稳定进入 prompt，DDR 通过第一轮验证
```

验证命令可以包括：

```text
bdinfo                                // 看 DRAM base/size 是否合理
mtest <start> <end>                   // 做简单内存测试，范围不要覆盖 U-Boot 自身
```

### 3.6 第三目标：让 U-Boot 能读 Linux 镜像和 dtb

读镜像可以先选一种最容易的方式，常见是 TFTP、SD/eMMC、NAND/SPI flash。

TFTP 验证示例：

```text
ping ${serverip}                      // 先确认网络链路通
tftp ${kernel_addr_r} zImage          // 把 Linux 镜像读到 DDR
tftp ${fdt_addr_r} my_board.dtb       // 把 dtb 读到 DDR
iminfo ${kernel_addr_r}               // 如果是 uImage，可检查镜像头
fdt addr ${fdt_addr_r}                // 让 U-Boot 指向当前 dtb
fdt print /chosen                     // 检查 chosen、bootargs 等信息
```

本地存储验证示例：

```text
mmc list                              // 看 U-Boot 是否识别 MMC/SD 控制器
mmc dev 0                             // 选择设备
fatls mmc 0:1                         // 看分区里是否有镜像文件
fatload mmc 0:1 ${kernel_addr_r} zImage // 从 FAT 分区加载内核
fatload mmc 0:1 ${fdt_addr_r} my_board.dtb // 从 FAT 分区加载 dtb
```

这一阶段只要求 U-Boot 能把 image 和 `dtb` 放进 DDR，还不要求 rootfs 已经最终确定。

### 3.7 第四目标：Linux 内核能出 console

U-Boot 能加载 image 和 `dtb` 后，先让 Linux 出串口。

```text
setenv bootargs 'console=ttyS0,115200 earlycon' // 先把控制台打通，rootfs 可以后面再细化
bootz ${kernel_addr_r} - ${fdt_addr_r}          // 无 initramfs 时，用 - 表示没有 ramdisk
```

成功信号：

- 串口出现 `Booting Linux on physical CPU`
- 出现 `Linux version ...`
- 出现 `Kernel command line: ...`
- 能看到 `start_kernel()` 后续初始化日志

失败优先查：

- `console=` 串口名是否正确，例如 `ttyS0`、`ttyAMA0`、`ttyPS0`
- `dtb` 是否为当前板子
- `chosen/stdout-path` 是否正确
- kernel image 地址、`dtb` 地址是否互相覆盖

### 3.8 第五目标：rootfs 分三条路线走

Linux 能出 console 后，后半段不要混着讲。按 rootfs 形态分成三条路线最清楚。

| 路线 | 适合阶段 | U-Boot 需要加载什么 | Linux 依赖什么 |
| --- | --- | --- | --- |
| initramfs / initrd | 最早期验证“能进用户态” | kernel、`dtb`、可选 initrd | initramfs 支持、`/init`、基础命令 |
| NFS root | 开发调试 rootfs | kernel、`dtb` | 网卡驱动、IP autoconfig、NFS client |
| 本地存储 rootfs | 产品落地 | kernel、`dtb` | eMMC/SD/NAND/SPI 驱动、分区、文件系统 |

#### 3.8.1 路线 A：initramfs / initrd

这条路线适合最早期 Bring-up。它的好处是先绕开本地存储驱动和分区问题，只验证内核能否启动第一个用户态程序。

内置 initramfs 的思路：

```text
Linux .config                         // 配置 CONFIG_INITRAMFS_SOURCE 指向 rootfs 目录或 cpio
-> make zImage                        // initramfs 被打进内核镜像
-> U-Boot 加载 zImage + dtb            // 不需要额外加载 rootfs 分区
-> bootargs 可加 rdinit=/init          // 指定 initramfs 里的第一个程序
-> bootz ${kernel_addr_r} - ${fdt_addr_r} // 启动 Linux
```

外部 initrd 的思路：

```text
tftp ${kernel_addr_r} zImage          // 加载内核
tftp ${fdt_addr_r} my_board.dtb       // 加载设备树
tftp ${ramdisk_addr_r} rootfs.cpio.gz // 加载 initrd/initramfs 到 DDR
setenv ramdisk_size ${filesize}       // 记录 ramdisk 大小
setenv bootargs 'console=ttyS0,115200 rdinit=/init' // 指定内存 rootfs 的 init
bootz ${kernel_addr_r} ${ramdisk_addr_r}:${ramdisk_size} ${fdt_addr_r} // kernel + ramdisk + dtb
```

内核配置重点：

- `CONFIG_BLK_DEV_INITRD`
- `CONFIG_DEVTMPFS`
- `CONFIG_DEVTMPFS_MOUNT`
- BusyBox 尽量先用静态编译，减少动态库变量

成功信号：

- 能看到 `Unpacking initramfs`
- 能看到 `Run /init as init process`
- 能进入最小 shell

失败优先查：

- `bootz` 的 ramdisk 地址和大小是否正确
- `/init` 是否存在并有执行权限
- BusyBox 是否可执行，动态库是否缺失
- initramfs 是否被错误压缩或格式不对

#### 3.8.2 路线 B：NFS root

这条路线适合开发阶段。rootfs 放在 PC 上，修改文件不需要反复烧录板子。

PC 侧准备思路：

```text
mkdir -p /srv/nfs/rootfs              // 准备 NFS rootfs 目录
tar xf rootfs.tar -C /srv/nfs/rootfs  // 解包根文件系统
配置 /etc/exports                     // 导出 NFS 目录
exportfs -ra                          // 重新导出
```

`/etc/exports` 示例：

```text
/srv/nfs/rootfs *(rw,sync,no_root_squash,no_subtree_check) // 开发阶段方便调试，量产不这样放开
```

U-Boot 启动参数示例：

```text
setenv bootargs 'console=ttyS0,115200 root=/dev/nfs rw nfsroot=${serverip}:/srv/nfs/rootfs,v3,tcp ip=dhcp' // 指定 NFS root
tftp ${kernel_addr_r} zImage          // 加载内核
tftp ${fdt_addr_r} my_board.dtb       // 加载设备树
bootz ${kernel_addr_r} - ${fdt_addr_r} // rootfs 不由 U-Boot 加载，Linux 通过网络挂载
```

内核配置重点：

- 网卡驱动必须内建，不能只编成模块
- PHY、MDIO、MAC 相关驱动要能在 rootfs 前工作
- `CONFIG_IP_PNP`
- `CONFIG_IP_PNP_DHCP` 或静态 IP 参数
- `CONFIG_ROOT_NFS`
- `CONFIG_NFS_FS`

成功信号：

- 内核能识别网卡
- 能获取 IP 或使用静态 IP
- 能看到 NFS mount 成功
- 能执行 NFS rootfs 里的 `/sbin/init`

失败优先查：

- Linux 网卡驱动是否内建
- DTS 网口、PHY、MDIO、clock、reset 是否正确
- PC NFS 服务、导出路径、防火墙是否正确
- `nfsroot=` 路径和协议版本是否匹配

#### 3.8.3 路线 C：本地存储 rootfs

这条路线最接近产品落地。rootfs 放在 eMMC、SD、NAND、SPI flash 等本地介质上。

eMMC/SD + ext4 示例：

```text
setenv bootargs 'console=ttyS0,115200 root=/dev/mmcblk0p2 rootfstype=ext4 rw rootwait' // 等待 MMC 分区出现
fatload mmc 0:1 ${kernel_addr_r} zImage       // 从启动分区加载内核
fatload mmc 0:1 ${fdt_addr_r} my_board.dtb    // 从启动分区加载设备树
bootz ${kernel_addr_r} - ${fdt_addr_r}         // Linux 后续自己挂载 /dev/mmcblk0p2
```

NAND + UBI/UBIFS 示例：

```text
setenv bootargs 'console=ttyS0,115200 ubi.mtd=rootfs root=ubi0:rootfs rootfstype=ubifs rw' // 指向 UBI 卷
bootz ${kernel_addr_r} - ${fdt_addr_r}         // Linux 后续 attach UBI 并挂载 UBIFS
```

内核配置重点：

- 对应存储控制器驱动必须内建
- 分区解析支持要打开
- 对应文件系统要内建，例如 ext4、squashfs、ubifs、jffs2
- 如果 rootfs 在 UBI 上，MTD、UBI、UBIFS 都要配套

成功信号：

- 内核日志中出现目标块设备或 MTD/UBI 设备
- 能看到分区被识别
- `mount_root()` 成功
- `run_init_process()` 成功

失败优先查：

- `root=` 设备名是否写错
- 是否需要 `rootwait`
- 文件系统类型是否和分区实际格式一致
- 存储驱动是否内建而不是模块
- NAND 分区名、`ubi.mtd=`、UBI 卷名是否对应

### 3.9 推荐调试顺序

三条 rootfs 路线不是并列乱选，推荐顺序是：

```text
initramfs / initrd                    // 先证明内核能执行第一个用户态程序
-> NFS root                           // 再用可快速修改的完整 rootfs 做开发调试
-> 本地存储 rootfs                    // 最后切到接近量产的 eMMC/NAND/SD/SPI flash
```

这样每一步只新增一类变量。initramfs 阶段不碰存储，NFS 阶段重点验证网络和完整用户态，本地存储阶段再验证分区、文件系统和烧录。

## 4. 现实例子二：Zynq/PetaLinux 自定义 IP 的联调链

这个例子来自 FPGA 软硬件协同场景。它和普通 ARM 板移植的工具不同，但思路很有价值：硬件结构变化必须先进入硬件描述，再进入设备树和内核配置，最后才能在 Linux 里看到 driver/probe 或用户态节点。

完整链路可以这样看：

```text
修改 Vivado Block Design                    // 硬件结构变化的源头，例如新增 AXI 外设或 PWM
-> Generate Output Products                 // 让 Vivado 重新生成底层 HDL/BD 输出
-> Run Synthesis / Export XSA               // 生成软件侧能理解的硬件描述文件
-> petalinux-config --get-hw-description    // 把新的 XSA 导入 PetaLinux 工程
-> Device Tree Generator                    // 根据 XSA 生成 pcw.dtsi、pl.dtsi 等基础设备树
-> system-user.dtsi                         // 手动追加或覆盖自动生成层没有写完整的节点
-> petalinux-build -c device-tree -x cleansstate // 清理设备树缓存，避免旧 DTS 继续参与构建
-> petalinux-build -c u-boot / bootloader   // 重新生成引导相关产物
-> petalinux-build                          // 重新生成内核、设备树、rootfs 等产物
-> boot.scr / BOOT.BIN / image / dtb         // 准备实际启动材料
-> Linux dmesg / sysfs / driver probe        // 在系统里验证设备是否真的出现并绑定驱动
```

这个例子里最容易漏的是“硬件已经加了 IP，不代表 Linux 已经知道它”。Linux 至少还需要三类东西同时对上：

| 层级 | 要确认什么 | 如果漏了会怎样 |
| --- | --- | --- |
| 硬件描述 | XSA/HDF 里确实包含新的硬件结构 | PetaLinux 自动生成层看不到这个硬件 |
| 设备树 | `pl.dtsi` 或 `system-user.dtsi` 里有正确节点、地址、中断、clock 等属性 | Linux 不会创建对应 device，driver 不会 probe |
| 内核配置 | 对应 framework 和 driver 已经打开，例如 PWM、GPIO、I2C、SPI、UIO 等 | 有设备节点也可能没有驱动接管 |
| 启动材料 | 实际启动用的是新 `dtb`、新 boot script、新镜像 | 改了源码但板子仍在跑旧产物 |

如果目标是验证一个 PWM 类外设，可以按下面顺序排查：

```text
确认 Vivado 里确实有 PWM/IP              // 硬件源头先成立
-> 确认 XSA 已重新导入 PetaLinux          // 软件工程拿到新的硬件描述
-> 确认 pl.dtsi 或 system-user.dtsi 有节点 // Linux 需要从 DTS 创建设备对象
-> 确认内核打开 PWM framework/driver      // 没有驱动支持，probe 不会完成
-> 确认 boot.scr 或 U-Boot 实际加载新 dtb  // 避免板子仍启动旧设备树
-> 启动后看 dmesg                         // 观察 pinctrl、platform device、driver probe
-> 看 /sys/class/pwm 或对应 sysfs          // 从用户态确认设备是否出现
```

这里还有一个很实用的判断：日志最后停在某一行，例如 `zynq-pinctrl ... initialized`，不一定说明这一行对应的驱动坏了。它只说明“最后可见输出停在这里”。真正排查时要继续问：

- 后面是不是控制台没继续输出
- rootfs 是否正在等待块设备
- 目标外设是否缺 DTS 节点
- 目标外设的内核配置是否没打开
- 启动时加载的 `dtb` 是否还是旧文件

这个例子适合放在本章，是因为它把 Bring-up 的本质暴露得很清楚：硬件变化、设备树变化、内核配置变化、启动材料变化必须形成闭环，缺一层都会表现成“Linux 里看不到设备”。

## 5. 联调时最稳的分界线

### 5.1 `U-Boot` 阶段

只关心：

- 能否稳定出串口
- 能否稳定初始化 DDR
- 能否从存储读取镜像

### 5.2 内核阶段

只关心：

- 控制台是否正常
- 内存和中断是否稳定
- 关键总线和存储是否 probe

### 5.3 用户态阶段

只关心：

- 根文件系统是否挂载
- `init` 是否运行
- 系统服务和业务程序是否拉起

## 6. 全流程总结

```text
串口最早输出                          // 先建立可观察窗口，否则后续问题没有证据
-> DDR 稳定                           // 保证 U-Boot 重定位和内核运行不会随机异常
-> Linux 源码基线确认                  // 选定 hi3516cv610 的 defconfig、DTS 和驱动配置
-> U-Boot 存储或网络可读               // 能从 flash/eMMC/SD/TFTP 读取 image、dtb 或 initrd
-> bootcmd 加载 image/dtb/可选 initrd   // 把交接物放到不互相覆盖的内存地址
-> bootm / bootz 跳入 Linux            // U-Boot 阶段完成，控制权交给内核
-> Linux console 可见                  // 验证 console=、chosen、串口驱动和早期日志
-> 内存/时钟/中断/驱动模型稳定          // 内核底座可用，关键 initcall 和 probe 能跑
-> rootfs 路线选择                     // 根据阶段选择 initramfs、NFS root 或本地存储 rootfs
   -> 路线 A：initramfs / initrd        // 先用内存 rootfs 验证能执行第一个用户态程序
   -> 路线 B：NFS root                  // 再用网络 rootfs 提高开发调试效率
   -> 路线 C：本地存储 rootfs           // 最后验证 eMMC/SD/NAND/SPI flash 等产品化路径
-> rootfs 后端可用                     // root=、nfsroot=、ubi.mtd= 或 initramfs 内容能被内核识别
-> mount_root() 成功                   // 文件系统类型和 rootfs 内容可被挂载
-> run_init_process() 成功             // /sbin/init 或 init= 指定程序可执行
-> 关键外设逐个打开                    // 网络、USB、WiFi、显示等业务外设最后分层验证
```

## 7. 例子使用边界

本章两个例子的作用是提供真实可操作的排查模板，不是要求所有板子逐字照抄命令。

- HI3516CV610 Linux 5.10.y 例子的重点是“当前板子使用哪份 defconfig、哪份 DTS、哪些驱动配置，并如何被 U-Boot 实际加载”，其中 TFTP、MMC、`bootz` 命令只是验证手段，真实项目也可能从 eMMC、SD、NAND、SPI flash 或 USB 加载镜像
- rootfs 三条路线不是互相替代的死规则，而是调试阶段不同：initramfs/initrd 先降变量，NFS root 方便开发，本地存储 rootfs 接近量产
- `root=/dev/mmcblk0p2` 只是本地存储示例，真实 rootfs 也可能是 `root=/dev/nfs`、`rdinit=/init`、UBIFS、squashfs 或其他分区
- Zynq/PetaLinux 例子强调“硬件描述 -> 设备树 -> 内核配置 -> 启动材料 -> Linux 验证”的闭环，普通 Linux 板也有类似闭环，只是工具不一定是 Vivado/PetaLinux
- 如果实际板子不是 Zynq，不需要照搬 `petalinux-*` 命令，但要保留同样的检查思路：硬件是否存在、DTS 是否描述、driver 是否启用、启动是否加载了新产物

参考材料：

- 本地文档：`C:\Users\Administrator\Desktop\linux\Vivado自定义IP与设备.docx`
- 参考文章：<https://blog.csdn.net/pyh1322712308/article/details/145140845>

## 8. 一句话总结

Bring-up 的本质不是一次性把整板功能全做完，而是给启动链逐层建立稳定边界。
