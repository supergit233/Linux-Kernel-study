可以。我们用一个**假设例子**讲：  
你有一块新板子，芯片是 **A100 SoC**，官方已经有一块参考板叫 **a100_evb**，你的新板叫 **myboard**。

你的目标是：

```text
让 U-Boot 在 myboard 上跑起来，并从 eMMC 启动 Linux
```

---

# 一、先找参考板

假设 U-Boot 里已经有官方参考板：

```text
configs/a100_evb_defconfig
board/vendor/a100_evb/
arch/arm/dts/a100_evb.dts
include/configs/a100_evb.h
```

你的新板和参考板用的是同一个 SoC，所以先复制：

```bash
cp configs/a100_evb_defconfig configs/myboard_defconfig

mkdir -p board/mycompany/myboard
cp board/vendor/a100_evb/* board/mycompany/myboard/

cp arch/arm/dts/a100_evb.dts arch/arm/dts/myboard.dts
cp include/configs/a100_evb.h include/configs/myboard.h
```

这一步的意思是：

```text
我不从零写 U-Boot，而是拿同芯片的参考板改。
```

---

# 二、改 defconfig

打开：

```bash
vim configs/myboard_defconfig
```

原来可能是：

```text
CONFIG_TARGET_A100_EVB=y
CONFIG_DEFAULT_DEVICE_TREE="a100_evb"
```

改成：

```text
CONFIG_TARGET_MYBOARD=y
CONFIG_DEFAULT_DEVICE_TREE="myboard"
CONFIG_SPL=y
CONFIG_DM_SERIAL=y
CONFIG_DM_MMC=y
CONFIG_CMD_MMC=y
```

意思是：

```text
我要编译 myboard 这个板子；
默认设备树用 myboard.dts；
打开 SPL；
打开串口；
打开 eMMC/SD 支持。
```

---

# 三、改 Kconfig，让 U-Boot 认识你的板子

在 `board/mycompany/myboard/Kconfig` 里写：

```kconfig
if TARGET_MYBOARD

config SYS_BOARD
    default "myboard"

config SYS_VENDOR
    default "mycompany"

config SYS_CONFIG_NAME
    default "myboard"

endif
```

这段的作用是告诉 U-Boot：

```text
TARGET_MYBOARD 对应的 board 目录是 board/mycompany/myboard/
配置头文件是 include/configs/myboard.h
```

还要在 SoC 对应的 Kconfig 里加一项：

```kconfig
config TARGET_MYBOARD
    bool "MyCompany MyBoard"
    select ARCH_A100
```

意思是：

```text
menuconfig 里多一个 myboard 选项。
```

---

# 四、改设备树 DTS

这是最重要的一步。

假设参考板 `a100_evb.dts` 里是这样：

```dts
/ {
    model = "A100 EVB";
    compatible = "vendor,a100-evb", "vendor,a100";

    chosen {
        stdout-path = "serial0:115200n8";
    };

    aliases {
        serial0 = &uart0;
    };

    memory {
        device_type = "memory";
        reg = <0x80000000 0x40000000>; // 1GB
    };
};

&uart0 {
    status = "okay";
};

&mmc0 {
    status = "okay";
    bus-width = <4>;
};
```

你的新板情况是：

```text
DDR 是 512MB
调试串口用 uart1
eMMC 挂在 mmc1 上，8bit 总线
```

那你就改成：

```dts
/ {
    model = "MyCompany MyBoard";
    compatible = "mycompany,myboard", "vendor,a100";

    chosen {
        stdout-path = "serial1:115200n8";
    };

    aliases {
        serial1 = &uart1;
    };

    memory {
        device_type = "memory";
        reg = <0x80000000 0x20000000>; // 512MB
    };
};

&uart1 {
    status = "okay";
};

&mmc1 {
    status = "okay";
    bus-width = <8>;
    non-removable;
};
```

这一步的意思是：

```text
告诉 U-Boot：
1. 我的调试串口不是 uart0，而是 uart1
2. 我的 DDR 是 512MB
3. 我的 eMMC 用的是 mmc1
```

---

# 五、改 board.c

假设你的板子上 eMMC 需要一个 GPIO 拉高供电，比如 GPIO23。

那就在：

```text
board/mycompany/myboard/board.c
```

里加：

```c
#include <common.h>
#include <dm.h>
#include <asm/gpio.h>

int board_init(void)
{
    /*
     * 假设 GPIO23 控制 eMMC 电源
     * 拉高后 eMMC 才能工作
     */
    gpio_request(23, "emmc_power");
    gpio_direction_output(23, 1);

    return 0;
}
```

这段代码的意思是：

```text
U-Boot 启动时，把 GPIO23 拉高，给 eMMC 上电。
```

但实际项目里，能写进设备树的就尽量写设备树。  
`board.c` 一般只放设备树不好描述的板级特殊逻辑。

---

# 六、改默认启动命令

在 `include/configs/myboard.h` 里可以写：

```c
#define CFG_EXTRA_ENV_SETTINGS \
    "kernel_addr_r=0x82000000\0" \
    "fdt_addr_r=0x83000000\0" \
    "bootargs=console=ttyS1,115200 root=/dev/mmcblk1p2 rw rootwait\0" \
    "bootcmd=mmc dev 1; ext4load mmc 1:1 ${kernel_addr_r} Image; ext4load mmc 1:1 ${fdt_addr_r} myboard.dtb; booti ${kernel_addr_r} - ${fdt_addr_r}\0"
```

这里每一项什么意思？

```text
kernel_addr_r=0x82000000
    内核加载到 DDR 的 0x82000000 地址

fdt_addr_r=0x83000000
    设备树加载到 DDR 的 0x83000000 地址

bootargs=console=ttyS1,115200 root=/dev/mmcblk1p2 rw rootwait
    Linux 启动参数
    console=ttyS1 表示 Linux 串口打印用 ttyS1
    root=/dev/mmcblk1p2 表示根文件系统在 eMMC 的第 2 分区

bootcmd=...
    U-Boot 默认执行的启动命令
```

展开看就是：

```bash
mmc dev 1
ext4load mmc 1:1 0x82000000 Image
ext4load mmc 1:1 0x83000000 myboard.dtb
booti 0x82000000 - 0x83000000
```

意思是：

```text
选择 eMMC 设备 1
从第 1 分区加载 Image 到内存
从第 1 分区加载 myboard.dtb 到内存
启动 Linux
```

---

# 七、编译

```bash
export ARCH=arm
export CROSS_COMPILE=aarch64-linux-gnu-

make myboard_defconfig
make -j8
```

生成的文件可能有：

```text
spl/u-boot-spl.bin
u-boot.img
u-boot.itb
u-boot.bin
```

具体烧哪个，看芯片手册。

---

# 八、烧录后调试流程

## 1. 最理想的启动日志

串口看到：

```text
U-Boot SPL 2024.xx

DRAM: 512 MiB
Trying to boot from MMC1

U-Boot 2024.xx

Model: MyCompany MyBoard
DRAM: 512 MiB
MMC: mmc@xxx: 1
Hit any key to stop autoboot: 0
```

这说明：

```text
SPL 启动了
DDR 初始化成功
SPL 成功从 eMMC 加载了完整 U-Boot
U-Boot proper 起来了
```

---

## 2. 如果完全没打印

重点查：

```text
烧录位置对不对
启动模式拨码对不对
串口接的是不是 uart1
波特率是不是 115200
DEBUG_UART_BASE 配得对不对
pinmux 有没有配置
```

---

## 3. 如果 SPL 有打印，但 U-Boot 起不来

例如只看到：

```text
U-Boot SPL 2024.xx
DRAM: 512 MiB
Trying to boot from MMC1
```

然后没了。

重点查：

```text
DDR 初始化是否真的稳定
mmc1 是否能读到 u-boot.img
u-boot.img 烧录位置是否正确
SPL 加载地址是否正确
```

---

## 4. 如果 U-Boot 起了，但找不到 eMMC

输入：

```bash
mmc list
```

如果没有 mmc1，重点查：

```text
设备树 &mmc1 status 是否 okay
pinctrl 是否配置
eMMC 电源 GPIO 是否拉高
时钟是否打开
复位脚是否释放
bus-width 是否正确
```

---

# 九、把这个例子总结成一句话

假设新板叫 `myboard`，适配 U-Boot 就是：

```text
复制同 SoC 的参考板
    ↓
改 defconfig，让 U-Boot 编译你的板子
    ↓
改 Kconfig/Makefile，让 U-Boot 找到你的 board 目录
    ↓
改 DTS，告诉 U-Boot 你的串口、DDR、eMMC、GPIO 怎么接
    ↓
改 board.c，处理特殊 GPIO、电源、复位
    ↓
改 bootcmd/bootargs，让它能加载 Linux
    ↓
编译、烧录、看串口日志一步步调
```

你可以先把它理解成：

```text
defconfig：我要编译哪些功能
DTS：我的硬件怎么接
board.c：板子上特殊动作怎么做
bootcmd：怎么启动 Linux
SPL：先初始化 DDR，再加载完整 U-Boot
```

面试回答时可以这样说：

> 比如我适配一块同 A100 SoC 的新板 myboard，我会先复制官方 a100_evb 的 defconfig、board 目录和 dts。然后修改 defconfig 里的 TARGET 和 DEFAULT_DEVICE_TREE，新增 TARGET_MYBOARD 的 Kconfig。接着根据原理图修改 myboard.dts，比如调试串口从 uart0 改成 uart1，DDR 容量改成 512MB，eMMC 从 mmc0 改成 mmc1，并配置 bus-width、pinctrl、电源 GPIO。SPL 阶段重点保证串口打印和 DDR 初始化成功，再让 SPL 从 eMMC 加载 u-boot.img。U-Boot proper 启动后，用 mmc list、ext4load 验证存储，最后配置 bootcmd 和 bootargs 加载 Image 和 dtb 启动 Linux。