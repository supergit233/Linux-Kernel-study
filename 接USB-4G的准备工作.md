# 接USB-4G的准备工作

可以，我们就只讲 `USB host` 这件事本身。

## **最基础的概念**

USB 里有两种角色：

- `host`：主机，负责“供电 + 发现设备 + 分配地址 + 管理通信”
- `device`：设备，负责“被主机发现和使用”

像 `USB WiFi 网卡`、`EC20 4G 模块` 这种插在板子上的外设，本质上都是 `USB device`。  
所以你的板子如果想“带着它们跑”，板子自己就必须当 `USB host`。

这也是为什么你这里一定要切到 `host`。  
如果板子还处在 `device` 模式，它更像“U 盘/摄像头/网卡那种被电脑识别的对象”，而不是去识别别人。

**你这颗 USB 控制器为什么能切换**

你这套 DTS 里，这个节点本身就是一颗“可切换角色”的控制器：

- 节点在 [hi3516cv610_family_usb.dtsi](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/arch/arm/boot/dts/hi3516cv610_family_usb.dtsi#L27)
- `compatible = "wing-usb,drd"`，见 [hi3516cv610_family_usb.dtsi](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/arch/arm/boot/dts/hi3516cv610_family_usb.dtsi#L28)
- `support-drd;`，见 [hi3516cv610_family_usb.dtsi](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/arch/arm/boot/dts/hi3516cv610_family_usb.dtsi#L32)
- `init_mode="device"`，见 [hi3516cv610_family_usb.dtsi](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/arch/arm/boot/dts/hi3516cv610_family_usb.dtsi#L40)
- 子节点 `dwc3` 里是 `dr_mode = "otg"` 和 `usb-role-switch`，见 [hi3516cv610_family_usb.dtsi](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/arch/arm/boot/dts/hi3516cv610_family_usb.dtsi#L51)

这几句合起来的意思就是：

“这不是一颗只能固定当 host 或固定当 device 的 USB 控制器，而是一颗 `DRD/OTG` 控制器，默认先按 `device` 起，但运行时可以切换。”

**真正是谁在切换**

不是 DTS 自己切，也不是 shell 命令直接改寄存器。  
真正干活的是厂商 USB 驱动 [wing_usb.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/wing_usb.c)。

它启动时会做几件底层准备：

- 读设备树属性，拿到 `support-drd`、`is-usb2`、时钟、PHY、寄存器基址，见 [wing_usb.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/wing_usb.c#L388)
- 打开时钟和 PHY，见 [wing_usb.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/wing_usb.c#L781)
- 做 USB2/阈值/省电相关配置，最后会写一次模式位，见 [wing_usb.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/wing_usb.c#L559)
- 如果支持 DRD，就创建 proc 接口并初始化切换状态，见 [wing_usb.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/wing_usb.c#L694) 和 [wing_usb.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/wing_usb.c#L843)

**为什么 `echo host > /proc/10300000.usb20drd/mode` 会生效**

```
echo host > /proc/10300000.usb20drd/mode
```

因为这个 `/proc` 文件就是驱动自己创建的，不是系统天然就有。

创建逻辑在 [proc.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/proc.c#L102)：

- `proc_mkdir(dev_name(dev), NULL)` 创建目录
- `proc_create_data("mode", ...)` 创建 `mode` 文件

所以最终就有了：
`/proc/10300000.usb20drd/mode`

当你写入 `host` 时，流程是：

1. `proc.c` 读到字符串 `"host"`，见 [proc.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/proc.c#L18)
2. 它把这个字符串翻译成一个事件 `SWITCH_TO_HOST`，见 [proc.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/proc.c#L35)
3. 然后调用 `wing_usb_queue_event()` 把事件塞进队列，见 [proc.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/proc.c#L43)
4. 工作线程再异步处理这个事件，见 [wing_usb.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/wing_usb.c#L302)
5. 事件类型是 `SWITCH_TO_HOST` 时，进入 `wing_usb_switch_to_host()`，见 [wing_usb.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/wing_usb.c#L224)

这一步是整件事最核心的地方。

**切到 host 时，底层到底做了什么**

这不是简单改一个变量，而是至少做了两层动作。

第一层是厂商胶水层 `wing_usb`：

- 先停掉 `device`，见 [wing_usb.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/wing_usb.c#L236)
- 再启动 `host`，见 [wing_usb.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/wing_usb.c#L242)
- 启动 host 时，会把 extcon 状态设成 `EXTCON_USB_HOST`，见 [wing_usb.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/wing_usb.c#L175)
- 然后通过 `usb_role_switch_set_role(..., USB_ROLE_HOST)` 告诉标准 DWC3 子驱动切角色，见 [wing_usb.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/vendor/usb/wing_usb.c#L189)

第二层是标准 DWC3 dual-role 框架：

- `dwc3_usb_role_switch_set()` 收到 `USB_ROLE_HOST` 后，把模式翻译成 `DWC3_GCTL_PRTCAP_HOST`，见 [drd.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/usb/dwc3/drd.c#L481)
- 然后调用 `dwc3_set_mode()` 真正切换，见 [drd.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/usb/dwc3/drd.c#L502)

进入 host 模式后，DWC3 核心会继续做 host 初始化：

- 把端口能力设成 `HOST`，见 [core.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/usb/dwc3/core.c#L1205)
- 把 PHY 切成 `USB host` 模式，见 [core.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/usb/dwc3/core.c#L1210)
- 打开 VBUS 相关控制，见 [core.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/usb/dwc3/core.c#L1208)
- OTG host 流程里还会设置 `Port Power`，见 [drd.c](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/linux/linux-5.10.y/drivers/usb/dwc3/drd.c#L237)

所以你可以把它理解成：

`echo host`
-> 写 proc
-> 厂商 DRD 驱动收到“切到主机”事件
-> 标准 USB role switch 把 DWC3 切到 host
-> PHY/端口供电/主机控制器一起进入 host 工作状态

**为什么还要有板级 GPIO 操作**

USB host 不只是“协议角色变了”，还常常牵涉到板级电源开关、USB 口复用、ID/供电使能这类硬件动作。  
你这套板级代码里就有这个味道，在 [init.sh](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/rootfs_3516cv610/rootfs/rootfs_debug_musl_arm/root/init.sh#L36) 附近配置了 `GPIO5_6`，并在 [init.sh](/E:/linux_share/驱动学习/hi3516cv610-wifi-4G模块/system_wifi_4g/rootfs_3516cv610/rootfs/rootfs_debug_musl_arm/root/init.sh#L40) 写了：
`echo host > /proc/10300000.usb20drd/mode`

这说明在你这块板子上，`USB host` 实际上有两层：

- SoC 内部 USB 控制器角色切换
- 板级外部口线/供电状态准备

前者是协议和控制器层面，后者是硬件连线层面。少一个都可能不行。

**你可以怎么验证自己的理解**

先别急着跑 WiFi/4G，先只观察 USB 角色本身：

```sh
cat /proc/10300000.usb20drd/mode
echo host > /proc/10300000.usb20drd/mode
cat /proc/10300000.usb20drd/mode
dmesg | tail -n 50
```

如果切成功，再插 USB 外设时你会看到“主机枚举设备”的行为，比如出现 `wlan0` 或 `/dev/ttyUSB*`。  
这时才说明“host 角色 + 枚举链路”真的起来了。

一句话总结：

`USB host` 在你这套代码里，本质上就是“把一颗默认 device/otg 的 DWC3 双角色控制器，通过厂商 DRD 驱动和标准 role-switch 框架，切成主机模式，并配合板级电源/引脚，让它能去枚举 USB WiFi 和 EC20 这种外设。”

如果你愿意，我下一步可以继续只讲一个点：
“`从 echo host 开始，到为什么会出现 /dev/ttyUSB2` 这条 4G 枚举链怎么走。”