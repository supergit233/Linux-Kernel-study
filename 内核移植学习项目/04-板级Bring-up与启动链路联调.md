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

| 位置                                        | 作用                       | 本项目里重点看什么                                                                             |
| ----------------------------------------- | ------------------------ | ------------------------------------------------------------------------------------- |
| `arch/arm/configs/hi3516cv610*_defconfig` | 板级默认内核配置                 | 当前走 eMMC、flash、NAND、debug 还是 mini 路线                                                  |
| `.config`                                 | 实际参与本次编译的配置结果            | `CONFIG_ARCH_HI3516CV610`、MMC、MTD、UBIFS、NFS、initramfs 是否打开                            |
| `arch/arm/boot/dts/hi3516cv610.dtsi`      | SoC 公共硬件描述               | CPU、GIC、clock、reset、UART、MMC、SPI、GPIO 等基础控制器                                          |
| `arch/arm/boot/dts/hi3516cv610-demb*.dts` | 板级差异描述                   | memory、chosen、串口状态、eMMC/flash/NAND/SDIO 使能差异                                          |
| `arch/arm/boot/dts/Makefile`              | dtb 构建入口                 | `hi3516cv610-demb.dtb`、`hi3516cv610-demb-emmc.dtb`、`hi3516cv610-demb-flash.dtb` 是否被编译 |
| `drivers/vendor/mmc/`                     | HI3516 相关 MMC/SDIO 控制器驱动 | `sdhci_nebula.c`、`CONFIG_MMC_SDHCI_NEBULA`、SDIO/eMMC probe 路径                         |
| `drivers/mtd/`、`fs/ubifs/`                | flash/UBI/UBIFS 路线       | NAND/SPI flash、UBI attach、UBIFS rootfs                                                |
| `net/`、`fs/nfs/`                          | NFS root 路线              | 网卡驱动和 NFS client 必须在挂 rootfs 前可用                                                      |
| `init/`、`fs/`                             | rootfs 挂载主线              | `prepare_namespace()`、`mount_root()`、`run_init_process()`                             |

这份 Linux 源码里还会反复遇到几个核心对象：

| 对象 | 先这样理解 | 调试意义 |
| --- | --- | --- |
| `struct device_node` | 设备树节点在内核里的对象 | DTS 节点是否被 Linux 解析出来 |
| `struct platform_device` | 由设备树节点等来源生成的设备对象 | 节点是否变成可匹配的 Linux device |
| `struct platform_driver` | platform 总线上的驱动对象 | `of_match_table` 是否能匹配 `compatible` |
| `struct resource` | `reg`、`interrupts` 等资源的内核表示 | 地址、中断、内存资源是否解析正确 |
| `struct mmc_host` / `struct sdhci_host` | MMC/SDIO 控制器在内核里的核心对象 | eMMC、SD、SDIO 枚举失败时要回到这里 |
| `struct mtd_info` / UBI volume | flash 和 UBI/UBIFS 路线的核心对象 | NAND/SPI flash rootfs 失败时要看这一层 |

这一步的作用是先把“要改的地方”分层，而不是一上来就改文件。`defconfig` 解决“哪些内核能力被编进去”，DTS 解决“板上有什么硬件、资源在哪里”，驱动目录解决“DTS 节点最终由谁 probe”，`init/` 和 `fs/` 解决“内核后半段怎样进入 rootfs”。分清这些位置后，后面遇到现象就不会把配置问题、设备树问题和驱动问题混成一团。

判断这一步是否达标，可以看是否能回答三个问题：新板 `dtb` 从哪个 `.dts` 编出来，`.config` 从哪个 defconfig 来，某个 DTS 节点最终会被哪个驱动或子系统消费。

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

这一步解决的是“文件存在”和“构建系统知道它”之间的差异。仅仅复制出 `hi3516cv610-study-emmc.dts`，不会自动生成 `hi3516cv610-study-emmc.dtb`；只有把它加入 DTS Makefile，对应平台配置又打开时，`make dtbs` 才会生成这个新 `dtb`。

接下来要区分两种命令链，它们不是互相替代关系，而是处在不同阶段。

第一种是“已有新板 defconfig 后的可重复编译链路”。它适合 `hi3516cv610_study_emmc_defconfig` 已经整理好、只是要重新生成 image 和 `dtb` 的场景：

```text
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- hi3516cv610_study_emmc_defconfig // 载入新板默认配置
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- olddefconfig                     // 让 Kconfig 补齐新增或缺省选项
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- zImage dtbs -j8                  // 生成 kernel image 和所有 HI3516CV610 dtb
```

这条链路里，`olddefconfig` 是非交互式的。它不会让人选择功能，只会根据当前 defconfig 和 Kconfig 默认值补齐 `.config`，适合自动化构建和复现。

第二种是“生成或更新新板 defconfig 的配置适配链路”。它适合新增功能或改配置，例如 RTL8188EU WiFi 适配、打开某个文件系统、打开某个调试选项：

```text
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- distclean                 // 清理旧 .config 和生成文件，避免旧配置干扰
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- hi3516cv610_emmc_defconfig // 先加载最接近的参考底座
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- menuconfig                // 交互式增量修改，例如打开 R8188EU 或 NFS/ext4
保存并退出 menuconfig                                                             // 修改结果先落到当前 .config
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- savedefconfig             // 从当前 .config 生成最小化 defconfig
复制 defconfig 到 arch/arm/configs/hi3516cv610_study_emmc_defconfig              // 把配置结果沉淀成新板默认配置
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- zImage dtbs -j8           // 用沉淀后的配置重新编译验证
```

这条链路里，`menuconfig` 是“修改当前 `.config`”，`savedefconfig` 是“把当前 `.config` 压缩成可维护的最小 defconfig”。是否覆盖原厂 `hi3516cv610_emmc_defconfig` 取决于版本管理策略；学习项目更推荐保存成新板自己的 `hi3516cv610_study_emmc_defconfig`，避免把参考底座改乱。

以 RTL8188EU 为例，这份源码里的真实配置符号是 `CONFIG_R8188EU`，来源是 `drivers/staging/rtl8188eu/Kconfig`；它依赖 `WLAN`、`USB`、`CFG80211`，并且因为 `depends on m`，通常以模块形式生成 `r8188eu.ko`。所以 RTL8188EU 适配属于“配置适配链路”，不是每次普通编译都必须重新走 `menuconfig`。

这两条链路的选择标准很简单：如果只是复现一次已经确定的板级配置，走 `defconfig -> olddefconfig -> build`；如果要新增能力、调整驱动、打开 WiFi 或文件系统，再走 `defconfig -> menuconfig -> savedefconfig -> build`。前者强调可重复，后者强调把人工选择沉淀成后续可复用的默认配置。

如果本机实际安装的交叉编译器前缀不同，只替换 `CROSS_COMPILE=` 后面的前缀，不改变 `ARCH=arm`、defconfig 和 dtb 主线。

这一步完成后，U-Boot 后面要加载的主要产物是：

```text
arch/arm/boot/zImage                         // Linux kernel image
arch/arm/boot/dts/hi3516cv610-study-emmc.dtb // 新板设备树产物
```

配置检查示意：

```text
CONFIG_ARCH_HI3516CV610=y             // 目标 SoC 平台必须正确
CONFIG_MMC_SDHCI_NEBULA=y             // eMMC/SDIO 控制器驱动进入内核
CONFIG_MTD=y                          // flash/NAND/SPI flash 路线需要
CONFIG_EXT4_FS=y                      // eMMC/SD 分区如果使用 ext4 rootfs，需要打开
CONFIG_MTD_UBI=y                      // UBI 卷路线需要
CONFIG_UBIFS_FS=y                     // UBIFS rootfs 路线需要
CONFIG_NFS_FS=y                       // NFS root 路线需要
CONFIG_BLK_DEV_INITRD=y               // 外部 initrd/initramfs 路线需要
CONFIG_R8188EU=m                      // RTL8188EU USB WiFi 适配需要，通常生成 r8188eu.ko
```

这些配置不一定要在同一次最小化配置里全部打开。实际项目通常按 rootfs 路线选择：eMMC/ext4 看 `CONFIG_MMC_*` 和 `CONFIG_EXT4_FS`，NAND/UBIFS 看 `CONFIG_MTD`、`CONFIG_MTD_UBI`、`CONFIG_UBIFS_FS`，NFS root 看网卡驱动、IP 自动配置和 `CONFIG_NFS_FS`。

这一步通过后，内核侧至少已经准备好两类交接物：一个是 kernel image，另一个是新板 `dtb`。后续 U-Boot 阶段不再关心这些文件是怎么编出来的，只关心能否把它们加载到正确内存地址并传给 Linux。

如果这一步失败，优先看：

- 是否选错了 `hi3516cv610`、`hi3516cv610_emmc`、`hi3516cv610_nand_mini` 等 defconfig
- `arch/arm/boot/dts/Makefile` 里是否包含目标 `.dtb`
- 新 DTS 是否正确 `#include "hi3516cv610.dtsi"` 或参考板 DTS
- `CONFIG_ARCH_HI3516CV610`、存储驱动、rootfs 文件系统是否真的进了 `.config`
- 交叉编译器前缀是否正确

### 3.4 实际修改新板 DTS：先改能影响启动链的差异

复制出 `hi3516cv610-study-emmc.dts` 后，第一轮不要急着把所有外设都改完。更稳的做法是先只改和“能启动、能出日志、能挂 rootfs”直接相关的节点。

可以按下面顺序理解新板 DTS：

```text
/ { model / compatible / memory }      // 先描述这块板是谁、内存在哪里
-> #include "hi3516cv610.dtsi"         // 复用 SoC 级 CPU、GIC、clock、UART、MMC 等公共控制器
-> &uart0 status                       // 先保证 Linux console 有输出
-> &mmc0 status                        // eMMC rootfs 路线需要先让 MMC 控制器 probe
-> &sdio0 status                       // 如果不用 SDIO/WiFi，先禁用减少变量
-> &sfc / &snfc                        // SPI NOR/NAND 或 UBI/UBIFS 路线再打开
-> &mdio0 / &femac0                    // NFS root 路线需要网口、PHY、MDIO 对齐
```

第一轮可重点检查这些字段：

| DTS 位置 | 新板移植时先看什么 | 后续影响 |
| --- | --- | --- |
| `/model` | 板子名称是否已经换成新板名 | 方便从启动日志确认实际加载的是新 `dtb` |
| `/compatible` | 是否仍能匹配当前 SoC 平台 | 影响平台级初始化和部分驱动匹配 |
| `/memory/reg` | DDR 起始地址和大小是否和板子一致 | 错了可能 early panic、随机死机或内存越界 |
| `#include "hi3516cv610.dtsi"` | 是否复用正确 SoC 公共描述 | 少了会丢 CPU、GIC、clock、UART、MMC 等基础节点 |
| `&uart0 { status = "okay"; }` | console 使用的 UART 是否打开 | `console=` 正确但串口仍无输出时要查这里 |
| `&mmc0 { status = "okay"; }` | eMMC 控制器是否启用 | 本地 eMMC rootfs 依赖它 probe 成功 |
| `&sdio0 { status = "disabled"; }` | 暂时不用的 SDIO 是否禁用 | 减少枚举、供电、WiFi 模块等额外变量 |
| `&sfc` / `&snfc` | SPI NOR/NAND 是否和启动介质一致 | flash、UBI、UBIFS 路线依赖这些节点 |
| `&mdio0` / `&femac0` | 网口、PHY 地址、reset 时序是否正确 | NFS root 路线依赖网卡在挂 rootfs 前可用 |

因此，新板 DTS 第一轮修改可以长这样：

```text
/ {                                                     // 板级根节点
    model = "HI3516CV610 Study eMMC Board";             // 启动日志中用于确认加载了新板 dtb
    compatible = "hisilicon,hi3516cv610";               // 继续匹配 HI3516CV610 SoC 平台

    memory {                                            // DDR 描述，必须和真实板级内存一致
        device_type = "memory";                         // 标准 memory 节点类型
        reg = <0x40000000 0xC0000000>;                  // 示例沿用参考板，真实新板要按硬件修改
    };
};

#include "hi3516cv610.dtsi"                             // 复用 SoC 公共控制器
#include "hi3516cv610_family_usb.dtsi"                  // 如果 USB 资源沿用参考板，可先保留

&uart0 {
    status = "okay";                                    // 第一阶段先保证 Linux console
};

&mmc0 {
    status = "okay";                                    // eMMC rootfs 路线需要打开
};

&sdio0 {
    status = "disabled";                                // 暂不验证 SDIO/WiFi 时先关掉
};
```

这一轮的目标不是让整板所有外设都好，而是先让 Linux 能用新 `dtb` 出 console，并能识别后续 rootfs 所需的最小存储或网络后端。

### 3.5 先认识 U-Boot 文件结构：它负责上一层交接

Linux 的 `zImage` 和 `dtb` 编出来以后，还需要 U-Boot 把它们从存储介质或网络读到 DDR，并把 `bootargs` 交给 Linux。因此在看 U-Boot 串口、DDR、加载命令之前，需要先知道 U-Boot 源码里哪些文件负责这些事。

当前资料里可以作为只读参考的 U-Boot 源码路径是：

```text
E:\linux_share\驱动学习\HI3516CV610-SIM-开发板资料\应用程序\system\u-boot\u-boot-2022.07 // U-Boot 2022.07 参考源码
```

这里有一个容易混淆的地方：Linux 内核和 U-Boot 里都有名为 `hi3516cv610_emmc_defconfig` 的文件，但它们不在同一棵源码树里，也不是给同一个构建系统消费。

| 名称 | 实际路径 | 消费者 | 主要决定什么 |
| --- | --- | --- | --- |
| Linux 内核 defconfig | `E:\linux_share\驱动学习\HI3516CV610-SIM-开发板资料\linux-5.10.y\arch\arm\configs\hi3516cv610_emmc_defconfig` | Linux Kconfig | Linux 编译哪些驱动、文件系统、网络协议和板级内核能力 |
| U-Boot defconfig | `E:\linux_share\驱动学习\HI3516CV610-SIM-开发板资料\应用程序\system\u-boot\u-boot-2022.07\configs\hi3516cv610_emmc_defconfig` | U-Boot Kconfig | U-Boot 目标板、加载地址、默认 `bootargs`、环境变量保存位置、存储和文件系统命令能力 |

所以第 3.1 到第 3.4 节讨论的是 Linux 内核侧 `arch/arm/configs/` 和 DTS；从本节开始讨论的是 U-Boot 侧 `configs/`、`board/`、`arch/`、`drivers/`、`env/` 和 `cmd/boot*`。如果拿 Linux 内核 defconfig 去查 `CONFIG_TARGET_HI3516CV610`、`CONFIG_SYS_TEXT_BASE`、`CONFIG_ENV_IS_IN_MMC`，对不上是正常现象。

这份 U-Boot 树里，HI3516CV610 相关文件可以这样分层看：

| U-Boot 位置                                | 作用                             | 和内核移植的关系                                       |
| ---------------------------------------- | ------------------------------ | ---------------------------------------------- |
| `configs/hi3516cv610_emmc_defconfig`     | U-Boot 的板级默认配置                 | 决定目标板、串口、MMC、FAT、环境变量存储位置、默认 `bootargs` 等      |
| `board/vendor/hi3516cv610/`              | 板级私有初始化目录                      | 放板级初始化代码、板级 Kconfig/Makefile，是新板适配时最像“板级入口”的位置 |
| `arch/arm/cpu/armv7/hi3516cv610/`        | SoC 级早期代码                      | 和 CPU、时钟、低级初始化、重定位前后流程相关                       |
| `arch/arm/include/asm/arch-hi3516cv610/` | SoC 级寄存器和头文件                   | 板级代码和驱动访问 SoC 寄存器时会引用                          |
| `include/configs/hi3516cv610.h`          | 传统板级宏和环境相关配置                   | 可看到环境变量、启动参数、存储环境等传统配置痕迹                       |
| `drivers/mmc/bsp_hi3516cv610.c`          | U-Boot 侧 MMC 相关 BSP 代码         | 影响 U-Boot 能否从 eMMC/SD 读取 `zImage` 和 `dtb`      |
| `drivers/vendor/mmc/platform/`           | U-Boot 侧 vendor MMC/SDHCI 平台代码 | 和 `CONFIG_MMC_SDHCI_NEBULA`、MMC 控制器初始化相关       |
| `drivers/mtd/fmc_hi3516cv610.c`          | U-Boot 侧 flash/FMC 代码          | 影响 SPI NOR/NAND 等 flash 启动或读取能力                |
| `u-boot-hi3516cv610.bin`                 | 已生成的 U-Boot 镜像产物               | 烧写或启动时真正运行的 U-Boot 产物之一                        |

U-Boot 文件结构和启动链可以这样连起来：

```text
U-Boot configs/hi3516cv610_emmc_defconfig // 选择 HI3516CV610 eMMC 版 U-Boot 能力
-> board/vendor/hi3516cv610/          // 进入板级初始化代码
-> arch/arm/cpu/armv7/hi3516cv610/    // 进入 SoC 级早期初始化和运行环境建立
-> drivers/mmc / drivers/mtd / net    // 初始化能读取镜像的存储或网络后端
-> env / include/configs/hi3516cv610.h // 准备 bootargs、环境变量和默认启动行为
-> cmd/boot* / boot/                  // 执行 bootz/bootm，把 image、dtb、initrd 交给 Linux
```

这一步解决的是“U-Boot 要改哪里、查哪里”的问题。Linux 侧 DTS 描述的是 Linux 如何认识硬件；U-Boot 侧 `configs/`、`board/`、`arch/`、`drivers/`、`env/` 则决定 U-Boot 自己能否初始化串口、DDR、MMC/flash/network，并把 Linux 需要的文件读出来。

结合当前 U-Boot 侧 `configs/hi3516cv610_emmc_defconfig`，可以先关注这些已经由源码确认的配置：

```text
CONFIG_TARGET_HI3516CV610=y           // U-Boot 目标板是 HI3516CV610
CONFIG_SYS_TEXT_BASE=0x41800000       // U-Boot 链接/运行相关基地址
CONFIG_SYS_LOAD_ADDR=0x42080000       // U-Boot 默认 loadaddr
CONFIG_BOOTARGS="mem=256M console=ttyAMA0,115200n8" // 默认传给 Linux 的启动参数
CONFIG_ENV_IS_IN_MMC=y                // U-Boot 环境变量保存在 MMC
CONFIG_MMC_SDHCI_NEBULA=y             // U-Boot 侧 MMC/SDHCI 控制器能力
CONFIG_FS_FAT=y                       // U-Boot 能读 FAT 分区里的 zImage/dtb
```

这些配置说明：U-Boot 阶段并不是只执行几条命令，它自己也有板级配置、板级代码、控制器驱动和环境变量。后面看到 `printenv`、`fatload`、`tftp`、`bootz` 时，要能反向知道它们分别和 `env/`、`drivers/`、`cmd/boot*`、`boot/` 等 U-Boot 目录有关。类似 `CONFIG_MMC_SDHCI_NEBULA` 这种名字可能在 U-Boot 和 Linux 两侧都出现，但控制的不是同一个编译目标：U-Boot 侧影响启动阶段读取镜像，Linux 侧影响内核启动后自己的 MMC/SDHCI 驱动。

#### 3.5.1 U-Boot 适配目标：先完成“能交接 Linux”的闭环

U-Boot 适配不是一开始就追求所有外设可用，而是先让启动链能稳定走到 Linux。最小目标可以拆成六个检查点：U-Boot 能编译出独立镜像，串口能输出，DDR 能稳定使用，启动介质或下载介质能读取文件，`bootargs` 能传给 Linux，`bootz/bootm` 能把 kernel image、`dtb`、可选 initrd 交给内核。

```text
U-Boot defconfig                    // 选择目标板、命令、环境变量和启动介质能力
-> Kconfig target                    // 把 CONFIG_TARGET_* 映射到具体 board 目录和配置头文件
-> board init callbacks              // 执行 board_init、dram_init、board_mmc_init 等板级入口
-> serial/mmc/mtd/net drivers        // 提供串口输出、存储读取、网络下载等控制器能力
-> env / include/configs/*.h         // 形成 bootargs、loadaddr、boot targets 等默认行为
-> bootz / bootm                     // 把 image、dtb、initrd 交给 Linux
```

同 SoC、同启动介质的新板，优先走“最小派生路线”：先复制 U-Boot defconfig，继续复用 `board/vendor/hi3516cv610/`、`arch/arm/cpu/armv7/hi3516cv610/` 和 `include/configs/hi3516cv610.h`，只改启动介质、命令能力、默认启动参数和少量板级差异。等串口、DDR、eMMC/TFTP、`bootz` 全部稳定后，再决定是否拆出独立 board 目录。

真正差异很大的新板，才走“独立新板路线”：复制 board 目录、配置头文件、defconfig，并新增 Kconfig target。它的改动更多，但好处是新板和参考板可以长期分开维护。

#### 3.5.2 当前源码确认的 U-Boot 构建接入链

这份 U-Boot 2022.07 里，`hi3516cv610_emmc_defconfig` 不是孤立文件。它通过 Kconfig 逐级选到板级目录和板级代码：

```text
configs/hi3516cv610_emmc_defconfig       // make xxx_defconfig 时加载的 U-Boot 默认配置
-> CONFIG_TARGET_HI3516CV610=y           // arch/arm/mach-vendor/Kconfig 里的目标板开关
-> CONFIG_TARGET_HI3516CV610_FAMILY=y    // 选中 HI3516CV610 family 公共能力
-> board/vendor/hi3516cv610/Kconfig      // 设置 SYS_VENDOR、SYS_SOC、SYS_BOARD、SYS_CONFIG_NAME
-> board/vendor/hi3516cv610/Makefile     // obj-$(CONFIG_TARGET_HI3516CV610_FAMILY) := hi3516cv610.o
-> board/vendor/hi3516cv610/hi3516cv610.c // 编进板级初始化、存储、网络、GPIO 等回调
-> include/configs/hi3516cv610.h         // 提供内存基址、启动参数地址、boot targets 等传统配置
```

这条链路说明：只复制 `configs/*.defconfig` 只能得到一个新的配置入口；如果板级代码、配置头文件、Kconfig target 仍然指向 `hi3516cv610`，实际运行的板级初始化仍然是参考板那一套。学习阶段这样做是合理的，因为它能减少变量；真实新板长期维护时，需要评估是否拆出独立 board 目录。

#### 3.5.3 最小派生路线：同 SoC、eMMC 启动先复制 U-Boot defconfig

当前完整移植例子假设新板仍然是 HI3516CV610，并且先走 eMMC 启动路线。U-Boot 侧可以先复制一个新 defconfig，避免直接改乱参考配置：

```text
cd E:\linux_share\驱动学习\HI3516CV610-SIM-开发板资料\应用程序\system\u-boot\u-boot-2022.07 // 进入 U-Boot 参考源码
cp configs/hi3516cv610_emmc_defconfig configs/hi3516cv610_study_emmc_defconfig // 复制新板 U-Boot 默认配置
修改 configs/hi3516cv610_study_emmc_defconfig // 调整启动介质、命令能力、bootargs、加载地址等
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- hi3516cv610_study_emmc_defconfig // 生成 U-Boot .config
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- olddefconfig // 让 U-Boot Kconfig 补齐缺省项
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- -j8 // 编译 U-Boot 镜像
```

这一步和 Linux 内核的 defconfig 操作很像，但消费它的是 U-Boot Kconfig，不是 Linux Kconfig。产物也不是 `zImage` 或 `dtb`，而是 U-Boot 自己的镜像，例如 `u-boot.bin`、板厂命名的 `u-boot-hi3516cv610.bin` 或构建脚本生成的烧写产物。

这套链路可以这样理解：

```text
defconfig
-> Kconfig
-> SYS_VENDOR / SYS_BOARD / SYS_CONFIG_NAME
-> Makefile
-> board.c
-> DTS
-> 启动命令 bootcmd / bootargs
-> 编译生成 U-Boot
-> 烧录运行
-> U-Boot 加载 Linux image + Linux dtb
```

更准确地分成两条线。

**一、构建时链路：决定 U-Boot 编译哪些东西**

```text
make myboard_defconfig
    // 载入 configs/myboard_defconfig

-> configs/myboard_defconfig
    // 里面写 CONFIG_TARGET_MYBOARD=y、CONFIG_CMD_MMC=y、CONFIG_DM_SERIAL=y 等

-> 某个 Kconfig 里定义 config TARGET_MYBOARD
    // 例如 arch/arm/mach-xxx/Kconfig 或 board/xxx/yyy/Kconfig
    // 这里定义“这个 target 是谁”，并 select SoC family、CPU、基础能力

-> source "board/mycompany/myboard/Kconfig"
    // 把板级 Kconfig 加入配置系统

-> board/mycompany/myboard/Kconfig
    // 设置 SYS_VENDOR="mycompany"
    // 设置 SYS_BOARD="myboard"
    // 设置 SYS_CONFIG_NAME="myboard"

-> U-Boot 得到 board 路径和配置头文件
    // board 路径 = board/mycompany/myboard/
    // 配置头文件 = include/configs/myboard.h

-> board/mycompany/myboard/Makefile
    // 根据 CONFIG_TARGET_MYBOARD 或其他 CONFIG_* 编译 board.c

-> include/configs/myboard.h
    // 提供传统板级宏、默认环境变量、boot target、内存参数等

-> drivers/*/Makefile
    // 根据 CONFIG_MMC、CONFIG_NET、CONFIG_SERIAL 等决定哪些驱动编进来

-> make -j8
    // 生成 u-boot.bin、u-boot.img、SPL 等产物
```

一句话：**defconfig 选 target，Kconfig 建立 target 和 board 的关系，Makefile 根据配置把对应代码编进去。**

**二、运行时链路：决定 U-Boot 在板子上干什么**

```text
BootROM / SPL
    // 芯片最早期启动，可能先初始化 SRAM、DDR，加载完整 U-Boot

-> U-Boot early init
    // 早期串口、时钟、pinmux、DDR、重定位

-> board.c 回调
    // board_early_init_f()
    // dram_init()
    // board_init()
    // board_mmc_init()
    // board_eth_init()
    // misc_init_r()

-> U-Boot prompt / autoboot
    // 可以 printenv、bdinfo、mmc list、fatload

-> bootcmd
    // 自动执行加载命令

-> bootargs
    // 传给 Linux 的命令行参数

-> bootz / bootm / booti
    // 把 kernel image、dtb、initrd 交给 Linux
```

`DTS` 要单独看，分两种情况：

```text
主线 U-Boot 常见情况：
U-Boot 自己有 arch/arm/dts/myboard.dts
CONFIG_DEFAULT_DEVICE_TREE="myboard"
这个 DTS 给 U-Boot 自己的 driver model 用

当前 HI3516CV610 这份源码：
没有发现 U-Boot 侧 arch/arm/dts/*3516cv610*
所以不要硬套 U-Boot DTS 流程
Linux dtb 在 linux-5.10.y 里编译
U-Boot 只是把 hi3516cv610-myboard.dtb 加载到 DDR，再交给 Linux
```

以 CV610 最小派生路线看，就是：

```text
configs/hi3516cv610_myboard_emmc_defconfig
    // 新板 U-Boot 配置入口

-> CONFIG_TARGET_HI3516CV610=y
    // 先复用已有 CV610 target

-> arch/arm/mach-vendor/Kconfig
    // 定义 TARGET_HI3516CV610

-> board/vendor/hi3516cv610/Kconfig
    // SYS_VENDOR=vendor
    // SYS_BOARD=hi3516cv610
    // SYS_CONFIG_NAME=hi3516cv610

-> board/vendor/hi3516cv610/Makefile
    // 编译 hi3516cv610.c

-> board/vendor/hi3516cv610/hi3516cv610.c
    // board_init、dram_init、board_mmc_init 等实际运行

-> include/configs/hi3516cv610.h
    // DDR、boot 参数地址、boot target、传统环境配置

-> bootcmd
    // fatload zImage
    // fatload hi3516cv610-myboard.dtb
    // bootz
```

所以完整适配时，脑子里可以记成：

```text
defconfig：我要编译哪个板子、哪些功能
Kconfig：这个板子是谁，依赖哪些 SoC/CPU/公共能力
board Kconfig：board 目录和配置头文件叫什么
Makefile：哪些 .c 文件会被编进去
board.c：U-Boot 运行时怎么初始化板子
DTS：如果 U-Boot 使用 OF_CONTROL，就给 U-Boot 描述硬件；CV610 当前主要是 Linux dtb
bootargs：告诉 Linux console、rootfs、init 等参数
bootcmd：告诉 U-Boot 怎样加载 kernel/dtb/initrd 并跳内核
make：把上面的配置和代码变成 U-Boot 镜像
```


#### 3.5.4 U-Boot defconfig 里优先改哪些配置

U-Boot defconfig 的重点不是打开 Linux 驱动，而是让 U-Boot 自己具备“初始化基础硬件、读取文件、传参、跳内核”的能力。

| 配置项 | 当前源码状态 | 适配时先看什么 |
| --- | --- | --- |
| `CONFIG_TARGET_HI3516CV610=y` | 已打开 | 同 SoC 最小派生时保持不变；独立 target 路线才新增自己的 `CONFIG_TARGET_*` |
| `CONFIG_SYS_TEXT_BASE=0x41800000` | 已设置 | 需要和 BootROM/ATF/DDR 布局匹配；不了解启动镜像布局时不要随意改 |
| `CONFIG_SYS_LOAD_ADDR=0x42080000` | 已设置 | 这是 U-Boot 默认 `loadaddr`，后续还要结合 `kernel_addr_r`、`fdt_addr_r`、`ramdisk_addr_r` 规划 |
| `CONFIG_BOOTARGS="mem=256M console=ttyAMA0,115200n8"` | 已设置 | 早期先保留 `console=` 和最小 `mem=`，rootfs 路线确定后再补 `root=`、`rootfstype=`、`rootwait`、`nfsroot=` 等 |
| `CONFIG_ENV_IS_IN_MMC=y`、`CONFIG_ENV_OFFSET`、`CONFIG_ENV_SIZE` | 已设置 | 决定环境变量保存到 eMMC 的哪个位置；分区或烧录布局变化时必须重新核对 |
| `CONFIG_MMC_SDHCI_NEBULA=y`、`CONFIG_FS_FAT=y` | 已打开 | 支持 eMMC/SDHCI 控制器和 FAT 分区读取，适合先用 `fatload` 加载 `zImage` 和 `dtb` |
| `# CONFIG_CMD_NET is not set` | 当前未打开 | 如果要用 `ping`、`tftp` 调试，必须在 U-Boot 配置中打开网络命令和对应网卡能力 |
| `# CONFIG_CMD_FDT is not set` | 当前未打开 | 如果要在 U-Boot 命令行执行 `fdt addr`、`fdt print` 检查 `dtb`，必须打开 FDT 命令 |
| `# CONFIG_CMD_IMI is not set` | 当前未打开 | 只有需要检查 `uImage` 头时才考虑；`zImage` 路线通常不依赖 `iminfo` |

因此，若坚持使用当前 eMMC defconfig 不改命令能力，最稳的验证方式是先走 `mmc list`、`fatls`、`fatload`、`bootz`。若选择 TFTP 或需要在 U-Boot 命令行检查 `dtb`，要先把 `CONFIG_CMD_NET`、`CONFIG_CMD_FDT` 之类能力沉淀进新板 U-Boot defconfig。

#### 3.5.5 板级 C 文件按什么顺序改

`board/vendor/hi3516cv610/hi3516cv610.c` 是这份 U-Boot 中最直接的板级入口文件。适配时不要从业务外设开始，而是按启动链顺序看这些函数：

| 函数或位置 | 当前源码中的角色 | 适配关注点 |
| --- | --- | --- |
| `board_early_init_f()` | 早期板级初始化入口，当前实现很轻 | 如果新板串口 pinmux、早期 GPIO 或时钟需要更早处理，优先从这里或更早的 SoC 低级代码排查 |
| `board_init()` | 设置 `bi_arch_number`、`bi_boot_params`，并调用 `boot_flag_init()` | 确认启动参数地址、启动介质识别和板级基础信息正确 |
| `dram_init()` | 把 `gd->ram_size` 设置为 `PHYS_SDRAM_1_SIZE` | DDR 容量变化时，要和 `include/configs/hi3516cv610.h`、Linux DTS `memory` 节点、`bootargs mem=` 一起核对 |
| `misc_init_r()` | 重定位后杂项初始化，包含环境变量、自动升级、GPIO/IR-cut 等板厂逻辑 | 新板不需要的板厂副作用要特别小心，避免启动阶段误动 GPIO、电源或升级流程 |
| `board_mmc_init()` | 初始化 eMMC/SD 控制器，调用 `ext_sdhci_add_port()` 和 `mmc_flash_init()` | eMMC/SD 端口、控制器基地址、pinmux、时钟或启动介质变化时重点检查 |
| `board_eth_init()` | 初始化网卡，当前和 `CONFIG_SFV300_ETH` 相关 | TFTP、NFS root 调试依赖它；只走本地 eMMC 时可以先不把网络作为第一目标 |
| `include/configs/hi3516cv610.h` | 保存 DDR 基址、`CFG_BOOT_PARAMS`、`BOOT_TARGET_DEVICES` 等传统配置 | U-Boot 加载地址、环境变量、boot target、DDR 范围和 Linux 交接参数都要从这里反推 |

按这个顺序改的好处是问题边界清楚：串口不通先看早期初始化和串口配置，DDR 不稳先看 `dram_init()` 与内存宏，eMMC 不读文件先看 `board_mmc_init()` 和 MMC 配置，TFTP 不通再看 `board_eth_init()`、网络命令和环境变量。

#### 3.5.6 独立新板路线：差异很大时才复制 board 目录

如果真实新板不只是改启动参数，而是 DDR、GPIO、电源、启动介质、网口、默认环境和板厂逻辑都和参考板明显不同，才适合拆出独立 board 目录。此时不能只复制目录，还要把 Kconfig、Makefile、配置头文件和 defconfig 一起接上：

```text
cp -r board/vendor/hi3516cv610 board/vendor/hi3516cv610_study // 复制板级目录，保留参考板作为对照
cp include/configs/hi3516cv610.h include/configs/hi3516cv610_study.h // 复制传统板级配置头文件
cp configs/hi3516cv610_emmc_defconfig configs/hi3516cv610_study_emmc_defconfig // 复制 U-Boot 默认配置
修改 arch/arm/mach-vendor/Kconfig // 增加 CONFIG_TARGET_HI3516CV610_STUDY，并 source 新 board Kconfig
修改 board/vendor/hi3516cv610_study/Kconfig // 设置 SYS_VENDOR、SYS_SOC、SYS_BOARD、SYS_CONFIG_NAME
修改 board/vendor/hi3516cv610_study/Makefile // 让新 target 编译自己的板级对象
修改 configs/hi3516cv610_study_emmc_defconfig // 选择新 CONFIG_TARGET_* 和新板命令能力
```

独立路线的判断标准是“是否需要长期让两块板共存”。如果只是学习或同 SoC 同 eMMC 的小差异，最小派生路线更稳；如果参考板逻辑已经会干扰新板启动，独立路线更适合。

#### 3.5.7 U-Boot 适配完成后的验收方式

U-Boot 适配完成不等于 Linux 已经成功启动，它只表示交接前的基础能力可用。建议按下面顺序验收：

```text
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- hi3516cv610_study_emmc_defconfig // 确认新 defconfig 能被 U-Boot 构建系统识别
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- olddefconfig                     // 确认 Kconfig 能补齐配置且没有符号冲突
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- -j8                              // 确认能编出 U-Boot 镜像
串口上电有 U-Boot 日志                    // 确认早期串口和基础启动链可见
printenv bootargs loadaddr kernel_addr_r fdt_addr_r ramdisk_addr_r // 确认环境变量和加载地址
bdinfo                                    // 确认 DDR 起始、大小、重定位位置和保留区域
mmc list                                  // 确认 U-Boot 能看到 eMMC/SD 控制器
fatls mmc 0:1                             // 确认 U-Boot 能读启动分区里的文件
fatload mmc 0:1 ${kernel_addr_r} zImage   // 确认 Linux 镜像能加载到 DDR
fatload mmc 0:1 ${fdt_addr_r} hi3516cv610-study-emmc.dtb // 确认新板 dtb 能加载到 DDR
bootz ${kernel_addr_r} - ${fdt_addr_r}    // 确认 U-Boot 能把控制权交给 Linux
```

如果使用 TFTP 路线，还要额外验证 `help tftp`、`ping ${serverip}`、`tftp ${kernel_addr_r} zImage`；如果需要在 U-Boot 阶段检查 `dtb` 内容，还要额外验证 `help fdt`、`fdt addr ${fdt_addr_r}`、`fdt print /model`。这些命令能否使用，首先取决于 U-Boot defconfig 是否打开了对应 `CONFIG_CMD_*`。

#### 3.5.8 可操作样例：基于 HI3516CV610 派生一块 myboard

假设新板叫 `hi3516cv610_myboard`，目标是让 U-Boot 在新板上跑起来，并从 eMMC 启动 Linux。这个例子先选“最小派生路线”：SoC 仍然是 HI3516CV610，调试串口先沿用 `ttyAMA0`，eMMC 先沿用当前参考板的 eMMC 路线，U-Boot 先不拆独立 board 目录。

```text
目标 SoC: HI3516CV610                       // 和参考板同 SoC
参考 U-Boot 配置: configs/hi3516cv610_emmc_defconfig // 当前源码真实存在
新板 U-Boot 配置: configs/hi3516cv610_myboard_emmc_defconfig // 新增学习用配置入口
Linux 镜像: zImage                          // Linux 5.10.y 编译产物
Linux dtb: hi3516cv610-myboard.dtb           // Linux 内核 DTS 编译产物，由 U-Boot 加载
rootfs: /dev/mmcblk0p2                       // 示例：eMMC 第二分区作为 ext4 rootfs
```

这份 CV610 U-Boot 源码没有发现 `arch/arm/dts/*3516cv610*`，`hi3516cv610_emmc_defconfig` 里也没有 `CONFIG_DEFAULT_DEVICE_TREE`。因此本例不要照主线 U-Boot 的新板模板去复制 U-Boot DTS。Linux 使用的 `hi3516cv610-myboard.dtb` 在 Linux 内核源码中维护，U-Boot 阶段只负责把它从 eMMC 读到 DDR，再通过 `bootz` 交给 Linux。

第一步，先复制一个新板 U-Boot defconfig：

```text
cd E:\linux_share\驱动学习\HI3516CV610-SIM-开发板资料\应用程序\system\u-boot\u-boot-2022.07 // 进入 U-Boot 源码
cp configs/hi3516cv610_emmc_defconfig configs/hi3516cv610_myboard_emmc_defconfig // 从 eMMC 参考配置派生新板配置
```

这一步的意思是：不从零写 U-Boot，也不先复制整套 board 目录，而是先拿同 SoC、同 eMMC 路线的配置作为底座。这样后续如果串口、DDR、eMMC 失败，优先排查真实硬件差异，而不是同时怀疑新 Kconfig、新 Makefile、新 board 目录都写错。

第二步，检查并修改 `configs/hi3516cv610_myboard_emmc_defconfig`。最小派生阶段可以继续保持参考 target：

```text
CONFIG_TARGET_HI3516CV610=y           // 仍然使用 HI3516CV610 参考板级代码
CONFIG_SYS_TEXT_BASE=0x41800000       // U-Boot 运行/链接基址，先不要随意改
CONFIG_SYS_LOAD_ADDR=0x42080000       // 默认 loadaddr，后续还要规划 kernel_addr_r 等地址
CONFIG_USE_BOOTARGS=y                 // 使用 U-Boot 默认 bootargs
CONFIG_BOOTARGS="mem=256M console=ttyAMA0,115200n8 root=/dev/mmcblk0p2 rootfstype=ext4 rw rootwait" // 示例：先让 Linux console 和本地 rootfs 路线明确
CONFIG_ENV_IS_IN_MMC=y                // 环境变量仍保存在 MMC
CONFIG_MMC_SDHCI_NEBULA=y             // 保留 CV610 eMMC/SDHCI 控制器能力
CONFIG_FS_FAT=y                       // 保留 FAT 读取能力，便于 fatload zImage 和 dtb
```

如果需要在 U-Boot 命令行检查 `dtb`，再打开 FDT 命令；如果需要 TFTP 下载，再打开网络命令。当前参考配置里这些命令默认关闭：

```text
CONFIG_CMD_FDT=y                      // 可选：提供 fdt addr、fdt print 等命令
CONFIG_CMD_NET=y                      // 可选：提供网络命令入口
CONFIG_CMD_PING=y                     // 可选：用于 ping serverip
CONFIG_CMD_TFTPBOOT=y                 // 可选：用于 tftp 下载镜像
```

更稳的操作方式是先执行 `make ... menuconfig` 打开这些命令，再用 `savedefconfig` 沉淀，避免手写配置项漏掉依赖：

```text
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- hi3516cv610_myboard_emmc_defconfig // 载入新板 U-Boot 配置
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- menuconfig                         // 交互打开 FDT、NET、TFTP 等命令
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- savedefconfig                      // 生成最小 defconfig
cp defconfig configs/hi3516cv610_myboard_emmc_defconfig                                   // 把配置沉淀回新板 defconfig
```

第三步，先不改 Kconfig/board 目录，确认当前构建接入链仍然成立：

```text
configs/hi3516cv610_myboard_emmc_defconfig // 新板配置入口
-> CONFIG_TARGET_HI3516CV610=y             // 仍然选中已有 target
-> arch/arm/mach-vendor/Kconfig            // 定义 TARGET_HI3516CV610 和 family
-> board/vendor/hi3516cv610/Kconfig        // 指向 SYS_VENDOR=vendor、SYS_BOARD=hi3516cv610
-> board/vendor/hi3516cv610/Makefile       // 编译 hi3516cv610.o
-> board/vendor/hi3516cv610/hi3516cv610.c  // 实际板级初始化代码仍来自参考板
```

这一步的意思是：`hi3516cv610_myboard_emmc_defconfig` 只是一个新配置名，不代表已经有独立 `board/mycompany/myboard/`。最小派生路线先接受这一点，目的是快速验证现有 U-Boot 在新板上能否跑起来。

第四步，按启动链检查 `board/vendor/hi3516cv610/hi3516cv610.c` 是否需要改。假设新板硬件差异如下：

```text
DDR 容量仍按 256MB 先验证              // `CONFIG_BOOTARGS` 中 mem=256M 暂时不改大
调试串口仍接 UART0                     // Linux console 继续用 ttyAMA0
eMMC 仍接当前参考 eMMC 控制器端口       // 先复用 board_mmc_init() 里的 ext_sdhci_add_port(0, EMMC_BASE_REG)
eMMC 电源/复位 GPIO 与参考板不同        // 这是最可能需要补板级动作的位置
```

如果 eMMC 电源或复位脚需要 U-Boot 提前处理，优先在重定位后杂项初始化或 MMC 初始化前处理。当前源码里可重点看这些位置：

```text
misc_init_r()                         // 重定位后杂项初始化，适合处理部分板级 GPIO/电源副作用
board_mmc_init()                      // MMC 初始化入口，eMMC 控制器端口、基地址、初始化失败时重点看这里
gpio_clock_config() / gpio_mux_config() / gpio_dir_config() / gpio_output_set() // 当前板级文件已有 GPIO 辅助函数
```

示例逻辑可以这样理解，真实 GPIO 编号、寄存器和电平必须以原理图和芯片手册为准：

```c
int misc_init_r(void)
{
    /* 原有参考板逻辑仍然保留，先不要大面积删除 */

    /* 示例：如果新板 eMMC 需要提前释放电源或复位，就在这里做板级动作 */
    gpio_clock_config(/* emmc_power_gpio */, 1);   // 打开 GPIO 时钟
    gpio_mux_config(/* emmc_power_mux_reg */, /* gpio_func */); // 把引脚切到 GPIO 功能
    gpio_dir_config(/* gpio_base */, /* gpio_offset */, 1); // 配置为输出
    gpio_output_set(/* gpio_base */, /* gpio_offset */, 1); // 拉高电源或释放复位

    return 0;
}
```

这段代码只表示适配位置和动作类型，不表示可以直接复制编译。真实项目需要把占位参数替换成 CV610 手册和新板原理图里的寄存器、GPIO offset 和有效电平。

第五步，规划 U-Boot 环境变量和启动命令。学习阶段建议先在 U-Boot prompt 里临时设置，验证成功后再固化到默认环境或保存到 MMC 环境区：

```text
setenv kernel_addr_r 0x42000000       // 示例：zImage 加载地址，必须位于 DDR 空闲区域
setenv fdt_addr_r 0x48000000          // 示例：Linux dtb 加载地址，和 kernel image 拉开距离
setenv bootargs 'mem=256M console=ttyAMA0,115200n8 root=/dev/mmcblk0p2 rootfstype=ext4 rw rootwait' // Linux 启动参数
setenv bootcmd 'mmc dev 0; fatload mmc 0:1 ${kernel_addr_r} zImage; fatload mmc 0:1 ${fdt_addr_r} hi3516cv610-myboard.dtb; bootz ${kernel_addr_r} - ${fdt_addr_r}' // 默认启动命令
saveenv                                // 如果环境变量保存位置已确认正确，再保存到 MMC
```

展开后的启动动作是：

```text
mmc dev 0                              // 选择 eMMC/SD 设备 0
fatload mmc 0:1 0x42000000 zImage      // 从第 1 分区读取 Linux zImage
fatload mmc 0:1 0x48000000 hi3516cv610-myboard.dtb // 从第 1 分区读取 Linux 设备树
bootz 0x42000000 - 0x48000000          // 不带 initrd，直接把 kernel 和 dtb 交给 Linux
```

这里的 `hi3516cv610-myboard.dtb` 来自 Linux 内核源码 `arch/arm/boot/dts/` 的编译产物。U-Boot 不负责解析这份 DTS 来初始化自己的 eMMC；它只是把这个二进制 `dtb` 交给 Linux，Linux 后续用它匹配串口、MMC、GPIO、clock、reset 等内核驱动资源。

第六步，编译新板 U-Boot：

```text
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- distclean                         // 清理旧配置和旧产物
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- hi3516cv610_myboard_emmc_defconfig // 加载新板 U-Boot 配置
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- olddefconfig                      // 补齐缺省配置
make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- -j8                               // 编译 U-Boot
```

生成产物名称以当前板厂构建脚本为准，常见检查对象包括：

```text
u-boot.bin                            // U-Boot 基础镜像
u-boot-hi3516cv610.bin                // 当前源码可能生成的板厂命名镜像
spl/ 或相关打包产物                   // 是否存在取决于当前配置和板厂脚本
```

第七步，烧录后按现象调试：

```text
完全无串口输出                         // 先查烧录位置、启动模式、串口线、波特率、UART pinmux、早期串口
有 U-Boot prompt，但 bdinfo 内存异常    // 查 DDR 初始化、PHYS_SDRAM_1_SIZE、mem= 参数和 Linux memory 节点
mmc list 看不到设备                    // 查 board_mmc_init()、eMMC 电源/复位、pinmux、clock、控制器端口
fatls mmc 0:1 失败                     // 查分区格式、分区号、FAT 支持、eMMC 是否枚举成功
fatload 成功但 bootz 无 Linux 日志       // 查 zImage/dtb 地址覆盖、console=、Linux dtb 是否正确、kernel 是否可执行
Linux 有日志但挂 rootfs 失败            // U-Boot 交接基本成立，转入 root=、rootfstype、rootwait、MMC 驱动和 rootfs 内容排查
```

这个 CV610 例子可以总结成：

```text
复制 hi3516cv610_emmc_defconfig       // 先得到新板 U-Boot 配置入口
-> 保持 CONFIG_TARGET_HI3516CV610      // 最小派生阶段继续复用参考板级代码
-> 调整 bootargs / 命令能力 / 环境变量  // 让 U-Boot 能加载并传参
-> 必要时改 hi3516cv610.c              // 处理新板 eMMC 电源、复位、GPIO 等特殊动作
-> 编译并烧录 U-Boot                   // 生成可运行启动镜像
-> 用串口、bdinfo、mmc、fatload、bootz 分阶段验收 // 一步一步确认交接链路
```

如果后续确认新板与参考板差异很大，再把这条最小派生路线升级为独立 board 路线：复制 `board/vendor/hi3516cv610/`、复制 `include/configs/hi3516cv610.h`、新增 `CONFIG_TARGET_HI3516CV610_MYBOARD`、修改 `arch/arm/mach-vendor/Kconfig` 和新 board `Kconfig/Makefile`。这一步适合正式产品长期维护，不适合作为第一轮 bring-up 的起点。
[[04-1-uboot适配举例]]

完整修改链路是[[04-1-uboot适配举例#九、把这个例子总结成一句话]]
在当前 CV610 U-Boot 里，kconfig链路是：
```
configs/hi3516cv610_emmc_defconfig
    |
    | CONFIG_TARGET_HI3516CV610=y
    v
arch/arm/mach-vendor/Kconfig
    |
    | config TARGET_HI3516CV610
    |     select TARGET_HI3516CV610_FAMILY
    |
    | source "board/vendor/hi3516cv610/Kconfig"
    v
board/vendor/hi3516cv610/Kconfig
    |
    | config SYS_VENDOR
    |     default "vendor"
    |
    | config SYS_SOC
    |     default "hi3516cv610"
    |
    | config SYS_BOARD
    |     default "hi3516cv610"
    |
    | config SYS_CONFIG_NAME
    |     default "hi3516cv610"
    v
U-Boot 得到：
    board 目录 = board/vendor/hi3516cv610/
    SoC 目录/名字 = hi3516cv610
    配置头文件 = include/configs/hi3516cv610.h
```
>[!TIP]
>arch/arm/mach-vendor/Kconfig  这个路径是通过搜索config TARGET_HI3516CV610得到的，是在soc下的kconfig

参考链路
```
configs/myboard_defconfig
    |
    | 里面写 CONFIG_TARGET_MYBOARD=y
    v
某个 SoC Kconfig
    |
    | 定义 config TARGET_MYBOARD
    | 并 source "board/mycompany/myboard/Kconfig"
    v
board/mycompany/myboard/Kconfig
    |
    | 设置 SYS_VENDOR="mycompany"
    | 设置 SYS_BOARD="myboard"
    | 设置 SYS_CONFIG_NAME="myboard"
    v
U-Boot 知道：
    board 目录 = board/mycompany/myboard/
    配置头文件 = include/configs/myboard.h
```

先查询找到soc的kconfig，看下deconfig添加的target会指向哪里，在对应的kconfig下查看配置路径，对同一款soc，从这里开始修改，
### 3.6 第一目标：先让 U-Boot 出串口

串口是第一条生命线。没有串口日志，后面 DDR、存储、Linux 都会变成盲调。

这一步仍然属于 U-Boot 阶段，目标不是验证 Linux 串口驱动，而是确认板子上电后至少有一个可观察窗口。U-Boot 串口正常，才能继续观察 DDR 初始化、存储命令、加载地址和 `bootargs`；如果这里没有输出，后面 Linux 的 DTS、rootfs、驱动配置都暂时没有排查意义。

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

这一步通过的标志是能稳定看到 U-Boot prompt，并能执行 `printenv`、`bdinfo` 等命令。通过以后，才进入 DDR 稳定性和镜像加载验证。

### 3.7 第二目标：DDR 稳定和 U-Boot 重定位

U-Boot 早期可能先在 SRAM 中运行，但完整 U-Boot 和 Linux 都需要 DDR。DDR 不稳时，现象会非常随机。

这一步必须放在加载 Linux 之前，因为 kernel image、`dtb`、initrd 都会被 U-Boot 放进 DDR。DDR 不稳定时，表现可能是 `tftp` 下载异常、`bootz` 随机卡死、Linux 解压崩溃，甚至同一条命令每次现象都不同。因此 DDR 先过关，后面的问题才有稳定复现价值。

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

这一步通过的标志是 U-Boot 能稳定重定位到 DDR，`bdinfo` 中 DRAM base/size 合理，简单内存测试不报错。通过以后，才能把 kernel image 和 `dtb` 放到 DDR 中验证加载链路。

### 3.8 第三目标：让 U-Boot 能读 Linux 镜像和 dtb

读镜像可以先选一种最容易的方式，常见是 TFTP、SD/eMMC、NAND/SPI flash。

这一步的核心不是“Linux 已经能启动”，而是验证 U-Boot 能把上一阶段编出来的 `zImage` 和 `hi3516cv610-study-emmc.dtb` 读到 DDR，并且地址没有互相覆盖。此时 rootfs 可以先不考虑，先把 image 和 `dtb` 两个交接物准备正确。

#### 3.8.1 加载地址从哪里来

`kernel_addr_r`、`fdt_addr_r`、`ramdisk_addr_r` 不是 Linux 内核里的变量，而是 U-Boot 环境变量。它们表示“把文件读到 DDR 的哪个地址”。这些地址通常有三种来源：

| 来源 | 怎么看 | 说明 |
| --- | --- | --- |
| U-Boot 当前环境变量 | `printenv kernel_addr_r fdt_addr_r ramdisk_addr_r loadaddr` | 最优先相信当前板子 U-Boot 已经定义好的地址 |
| U-Boot 板级默认环境 | U-Boot 源码里的 `CONFIG_EXTRA_ENV_SETTINGS`、板级头文件或默认环境 | 适合查“这些变量默认为什么是这个值” |
| 手动规划 DDR 空闲区 | `bdinfo`、DTS `/memory/reg`、芯片手册 | 适合 U-Boot 没有预设变量或要临时验证时使用 |

先在 U-Boot prompt 里查：

```text
printenv kernel_addr_r fdt_addr_r ramdisk_addr_r loadaddr // 看 U-Boot 是否已有推荐加载地址
bdinfo                                                    // 看 DRAM 起始地址、大小、U-Boot 重定位位置
fdt addr ${fdtcontroladdr}                                // 需要 CONFIG_CMD_FDT；如果 U-Boot 自己有控制用 dtb，可先观察当前 FDT 地址
```

如果这些变量不存在，可以临时设置一组地址，但必须满足三个条件：

- 地址必须落在 DDR 范围内
- 不能覆盖 U-Boot 自身、栈、malloc 区、环境区
- kernel image、`dtb`、initrd/rootfs.cpio.gz 之间要留足间隔

以当前参考 DTS 中的内存描述为例，`/memory/reg = <0x40000000 0xC0000000>` 表示 DDR 从 `0x40000000` 开始。实际调板时仍以 `bdinfo` 为准，可以先用下面这种“留出间隔”的示例地址：

```text
setenv kernel_addr_r 0x42000000        // 示例：kernel image 放到 DDR 起始地址后 32MB
setenv fdt_addr_r    0x48000000        // 示例：dtb 很小，但仍和 kernel 拉开距离
setenv ramdisk_addr_r 0x50000000       // 示例：initrd/rootfs.cpio.gz 可能较大，单独留更远空间
```

这组地址不是固定答案，只是说明选址思路。真实项目要结合 `bdinfo`、镜像大小、U-Boot 内存布局和板级启动习惯确定。

#### 3.8.2 读取 image 和 dtb

当前 U-Boot eMMC defconfig 已确认 `CONFIG_FS_FAT=y`，适合先用 FAT 分区和 `fatload` 验证本地启动介质。`tftp`、`fdt print`、`iminfo` 属于可选调试命令，当前源码里 `CONFIG_CMD_NET`、`CONFIG_CMD_FDT`、`CONFIG_CMD_IMI` 默认没有打开；如果要使用下面对应命令，需要先在新板 U-Boot defconfig 中打开相关 `CONFIG_CMD_*`。

TFTP 验证示例：

```text
ping ${serverip}                      // 需要 CONFIG_CMD_NET；先确认网络链路通
tftp ${kernel_addr_r} zImage          // 需要 CONFIG_CMD_NET；把 Linux 镜像读到 DDR
echo ${filesize}                      // 查看刚下载的 zImage 大小，用来判断是否会覆盖后续地址
tftp ${fdt_addr_r} hi3516cv610-study-emmc.dtb // 需要 CONFIG_CMD_NET；把新板 dtb 读到 DDR
echo ${filesize}                      // 查看 dtb 大小，dtb 通常远小于 kernel image
iminfo ${kernel_addr_r}               // 需要 CONFIG_CMD_IMI；仅 uImage 可用，zImage 路线通常跳过
fdt addr ${fdt_addr_r}                // 需要 CONFIG_CMD_FDT；让 U-Boot 指向当前 dtb
fdt print /chosen                     // 需要 CONFIG_CMD_FDT；检查 chosen、bootargs 等信息
fdt print /model                      // 需要 CONFIG_CMD_FDT；确认当前 dtb 是新板 model
```

本地存储验证示例：

```text
mmc list                              // 看 U-Boot 是否识别 MMC/SD 控制器
mmc dev 0                             // 选择设备
fatls mmc 0:1                         // 看分区里是否有镜像文件
fatload mmc 0:1 ${kernel_addr_r} zImage // 从 FAT 分区加载内核
echo ${filesize}                      // 查看 zImage 大小
fatload mmc 0:1 ${fdt_addr_r} hi3516cv610-study-emmc.dtb // 从 FAT 分区加载新板 dtb
echo ${filesize}                      // 查看 dtb 大小
```

这一阶段只要求 U-Boot 能把 image 和 `dtb` 放进 DDR，还不要求 rootfs 已经最终确定。

这一步通过的标志是 U-Boot 能稳定读到 `zImage` 和新板 `dtb`；如果已打开 `CONFIG_CMD_FDT`，`fdt print /model` 还应该能看到新板名称。通过以后，才执行 `bootz` 验证 Linux 是否能接管控制权。

### 3.9 第四目标：Linux 内核能出 console

U-Boot 能加载 image 和 `dtb` 后，先让 Linux 出串口。

这一步开始进入 Linux 阶段，但仍然只验证“内核入口和控制台”。先不要把 rootfs、WiFi、业务外设混进来，否则一旦卡住，很难判断是内核没起来、控制台没输出，还是 rootfs 没挂上。

```text
setenv bootargs 'console=ttyAMA0,115200 earlycon' // HI3516 的 PL011 串口通常使用 ttyAMA0，先把控制台打通
bootz ${kernel_addr_r} - ${fdt_addr_r}          // 无 initramfs 时，用 - 表示没有 ramdisk
```

成功信号：

- 串口出现 `Booting Linux on physical CPU`
- 出现 `Linux version ...`
- 出现 `Kernel command line: ...`
- 能看到 `start_kernel()` 后续初始化日志

失败优先查：

- `console=` 串口名是否正确；HI3516 的 PL011 串口通常先检查 `ttyAMA0`，其他平台可能是 `ttyS0` 或 `ttyPS0`
- `dtb` 是否为当前板子
- `chosen/stdout-path` 是否正确
- kernel image 地址、`dtb` 地址是否互相覆盖

这一步通过的标志是能看到 `Linux version`、`Kernel command line` 和后续 `start_kernel()` 相关日志。通过以后，说明 U-Boot 到 Linux 的 image、`dtb`、`bootargs` 交接基本成立，可以进入 rootfs 路线选择。

### 3.10 第五目标：rootfs 分三条路线走

Linux 能出 console 后，后半段不要混着讲。按 rootfs 形态分成三条路线最清楚。

这一步解决的是“内核起来以后，怎样进入第一个用户态进程”。三条路线的差异不在于 U-Boot 是否能 `bootz`，而在于 Linux 后半段从哪里得到 `/`：内存里的 initramfs、网络上的 NFS 目录，还是本地 eMMC/NAND/SPI flash 分区。

| 路线 | 适合阶段 | U-Boot 需要加载什么 | Linux 依赖什么 |
| --- | --- | --- | --- |
| initramfs / initrd | 最早期验证“能进用户态” | kernel、`dtb`、可选 initrd | initramfs 支持、`/init`、基础命令 |
| NFS root | 开发调试 rootfs | kernel、`dtb` | 网卡驱动、IP autoconfig、NFS client |
| 本地存储 rootfs | 产品落地 | kernel、`dtb` | eMMC/SD/NAND/SPI 驱动、分区、文件系统 |

#### 3.10.1 路线 A：initramfs / initrd

这条路线适合最早期 Bring-up。它的好处是先绕开本地存储驱动和分区问题，只验证内核能否启动第一个用户态程序。

这一路线引入的变量最少：不依赖 eMMC 分区、不依赖 NAND/UBI、不依赖网卡和 NFS 服务。只要 kernel、`dtb`、initramfs 格式和 `/init` 正确，就能验证 `run_init_process()` 是否能进入用户态。

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
tftp ${fdt_addr_r} hi3516cv610-study-emmc.dtb // 加载新板设备树
tftp ${ramdisk_addr_r} rootfs.cpio.gz // 加载 initrd/initramfs 到 DDR
setenv ramdisk_size ${filesize}       // 记录 ramdisk 大小
setenv bootargs 'console=ttyAMA0,115200 rdinit=/init' // 指定内存 rootfs 的 init
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

#### 3.10.2 路线 B：NFS root

这条路线适合开发阶段。rootfs 放在 PC 上，修改文件不需要反复烧录板子。

这一路线比 initramfs 多了网络变量，但调试效率高。驱动、脚本、库文件、业务程序都可以直接改 PC 上的 NFS 目录，适合内核已经能出 console 后做完整用户态验证。

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
setenv bootargs 'console=ttyAMA0,115200 root=/dev/nfs rw nfsroot=${serverip}:/srv/nfs/rootfs,v3,tcp ip=dhcp' // 指定 NFS root
tftp ${kernel_addr_r} zImage          // 加载内核
tftp ${fdt_addr_r} hi3516cv610-study-emmc.dtb // 加载新板设备树
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

#### 3.10.3 路线 C：本地存储 rootfs

这条路线最接近产品落地。rootfs 放在 eMMC、SD、NAND、SPI flash 等本地介质上。

这一路线变量最多，但最接近最终启动方式。它不仅要求内核支持对应存储控制器，还要求分区表、文件系统、`root=`、`rootfstype=`、`rootwait` 或 `ubi.mtd=` 全部对齐。

eMMC/SD + ext4 示例：

```text
setenv bootargs 'console=ttyAMA0,115200 root=/dev/mmcblk0p2 rootfstype=ext4 rw rootwait' // 等待 MMC 分区出现
fatload mmc 0:1 ${kernel_addr_r} zImage       // 从启动分区加载内核
fatload mmc 0:1 ${fdt_addr_r} hi3516cv610-study-emmc.dtb // 从启动分区加载新板设备树
bootz ${kernel_addr_r} - ${fdt_addr_r}         // Linux 后续自己挂载 /dev/mmcblk0p2
```

NAND + UBI/UBIFS 示例：

```text
setenv bootargs 'console=ttyAMA0,115200 ubi.mtd=rootfs root=ubi0:rootfs rootfstype=ubifs rw' // 指向 UBI 卷
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

### 3.11 推荐调试顺序

三条 rootfs 路线不是并列乱选，推荐顺序是：

```text
initramfs / initrd                    // 先证明内核能执行第一个用户态程序
-> NFS root                           // 再用可快速修改的完整 rootfs 做开发调试
-> 本地存储 rootfs                    // 最后切到接近量产的 eMMC/NAND/SD/SPI flash
```

这样每一步只新增一类变量。initramfs 阶段不碰存储，NFS 阶段重点验证网络和完整用户态，本地存储阶段再验证分区、文件系统和烧录。

这个顺序的意义是把复杂问题拆成可验证的台阶。若 initramfs 已经能进 shell，本地 rootfs 失败时就不应怀疑 `start_kernel()` 主线；若 NFS root 能跑完整用户态，本地存储失败时就应优先查块设备、MTD/UBI、分区和文件系统；若本地存储也能跑通，才继续把注意力放到业务外设、性能和稳定性。

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
-> 参考板派生新板配置                  // 从 hi3516cv610-demb-emmc 派生新 DTS 和 defconfig
-> Linux 源码基线确认                  // 新板 dtb 加入 Makefile，驱动和 rootfs 配置进入 .config
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

- HI3516CV610 Linux 5.10.y 例子的重点是“如何从真实存在的参考板 DTS/defconfig 派生新板，并确认新 `dtb`、驱动配置、rootfs 参数会被 U-Boot 实际加载”，其中 TFTP、MMC、`bootz` 命令只是验证手段，真实项目也可能从 eMMC、SD、NAND、SPI flash 或 USB 加载镜像
- rootfs 三条路线不是互相替代的死规则，而是调试阶段不同：initramfs/initrd 先降变量，NFS root 方便开发，本地存储 rootfs 接近量产
- `root=/dev/mmcblk0p2` 只是本地存储示例，真实 rootfs 也可能是 `root=/dev/nfs`、`rdinit=/init`、UBIFS、squashfs 或其他分区
- Zynq/PetaLinux 例子强调“硬件描述 -> 设备树 -> 内核配置 -> 启动材料 -> Linux 验证”的闭环，普通 Linux 板也有类似闭环，只是工具不一定是 Vivado/PetaLinux
- 如果实际板子不是 Zynq，不需要照搬 `petalinux-*` 命令，但要保留同样的检查思路：硬件是否存在、DTS 是否描述、driver 是否启用、启动是否加载了新产物

参考材料：

- 本地文档：`C:\Users\Administrator\Desktop\linux\Vivado自定义IP与设备.docx`
- 参考文章：<https://blog.csdn.net/pyh1322712308/article/details/145140845>

## 8. 一句话总结

Bring-up 的本质不是一次性把整板功能全做完，而是给启动链逐层建立稳定边界。
