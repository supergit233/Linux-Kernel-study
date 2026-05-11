# 内核适配rtl8188eu

## 交叉编译链地址

提供了一个命令source  /etc/profile（具体见测试文档）提供了编译链的配置路径，让环境可以找到编译链但不是生效

./install_gcc_toolchain.sh这个脚本文件有写（赞美AI）

## 步骤流程

在嵌入式交叉编译的场景下，完整且安全的操作流必须包含**环境清理**和**变量声明**。以下是进入菜单前完整的逻辑链路：

### 1. 前置步骤一：深度清理构建环境（排雷）

在修改任何内核配置之前，必须确保当前代码树是绝对干净的，没有任何之前编译残留的中间文件（`.o`）、依赖关系文件或旧的 `.config`。

- **执行命令**：

  Bash

  ```
  make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- distclean
  ```

- **原理解析**：`distclean` 比常规的 `clean` 更彻底。它会删除所有编译生成的文件、配置文件（包括隐藏的 `.config`）以及自动生成的头文件。如果不执行这一步，旧配置的残留极易导致新配置产生无法预料的依赖冲突。

### 2. 前置步骤二：声明架构与交叉编译工具链（定基调）

内核源码默认是为当前运行的主机（通常是 x86_64）配置和编译的。要为目标板编译，必须**在每一次执行 make 命令时**明确告知构建系统你的目标架构和所用的编译器前缀。

- **核心参数**：
  - `ARCH=arm`：锁定目标硬件体系结构，决定了系统会去 `arch/arm/` 目录下寻找规则和配置。
  - `CROSS_COMPILE=arm-v01c02-linux-musleabi-`：指定工具链前缀。系统在调用 gcc 时，会自动拼接成 `arm-v01c02-linux-musleabi-gcc`。
- **防错提醒**：这两个参数可以作为环境变量全局 `export`，但在实际工程中，为了避免污染其他项目，通常建议像上面那样直接跟在 `make` 命令后面，作为命令行变量传入。

### 3. 前置步骤三：主机侧依赖库确认（防报错）

`menuconfig` 是一个基于 ncurses 库的图形化字符界面。如果你的 Ubuntu 宿主机是刚初始化的环境，直接运行 `make menuconfig` 会因为缺少底层库而直接报错退出。

- **检查与安装**：需确保主机已安装了对应的开发包。对于基于 Debian/Ubuntu 的系统：

  Bash

  ```
  sudo apt-get update
  sudo apt-get install libncurses5-dev libncursesw5-dev build-essential flex bison
  ```

  *(注：`flex` 和 `bison` 是内核 Kconfig 语法解析器 `mconf` 编译时强依赖的词法/语法分析工具。)*

------

### 完整的标准执行流闭环

明确了前置步骤后，标准的内核配置链路应该是这样的严格顺序：

1. **清理环境**：make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- distclean
2. **加载底座**：make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- hi3516cv610_defconfig (在此处生成初始 `.config`)
3. **进入菜单**：make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- menuconfig (读取刚才的 `.config` 并允许你进行增量修改，如开启 WiFi 模块)
4. **保存配置**：退出并保存。
5. **覆盖原厂**：make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- savedefconfig 并替换掉 `arch/arm/configs/` 下的默认文件（可选，取决于你的版本管理策略）。

## 1.启动图形配置界面

进入open_source/linux/linux-5.10.y目录（目录是源码位置，不固定），执行make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- menuconfig，

## 2.配置路径与内容

![image-20260409215331377](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20260409215331377.png)



### Networking support无线网配置界面

注意：选中的选项最后是以<\*>编译的

![image-20260409222641074](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20260409222641074.png)

这张图展示的是 Linux 内核源码配置（`make menuconfig`）中的“无线网络”（Wireless）子系统配置界面。

在解析具体功能前，必须先明确界面中尖括号与方括号的底层编译逻辑：

- **`<\*>`**：静态编译进内核映像（如 zImage/bzImage）。随内核启动直接加载入内存，执行效率最高，但会增加内核文件体积。
- **`<M>`**：编译为独立的动态可加载模块（.ko 文件）。内核启动时不加载，由系统或用户脚本按需动态挂载，灵活性高。
- **`[ ]` 与 `[\*]`**：通常代表依附于某个主模块的子功能布尔开关（关闭/开启）。

以下是对图中具体配置项的技术解剖与现状分析：

#### 1. 核心支撑层

- **`cfg80211 - wireless configuration API` `<\*>`**
  - **定义**：Linux 现代无线网络设备的统一配置管理 API。它是连接底层网卡驱动与用户层应用（如 `wpa_supplicant` 或 `NetworkManager`）的核心桥梁。
  - **分析**：当前被配置为静态内建 `<*>`。这是绝对合理的。作为整个无线子系统的地基，将其做成模块化意义不大，缺少它后续任何现代 Wi-Fi 驱动都无法初始化。

#### 2. 调试与测试选项（NEW 标识表示此配置中新出现的选项）

- **`nl80211 testmode command` `[ ]`**
- **`enable developer warnings` `[ ]`**
- **`cfg80211 certification onus` `[ ]`**
  - **定义**：这三项分别是：射频底层测试模式、驱动开发者警告信息开关、以及将合规认证校验责任转移到用户空间的选项。
  - **分析**：当前全部留空（关闭）。逻辑清晰且正确。除非你正在实验室对网卡进行 FCC/CE 射频合规测试，或是开发底层驱动，否则开启这些选项毫无意义，只会徒增系统开销、制造大量无用的 dmesg 垃圾日志。

#### 3. 电源与合规策略

- **`enable powersave by default` `[\*]`**
  - **定义**：默认允许 Wi-Fi 芯片进入休眠节电状态。
  - **批判性评估**：当前状态为开启。这里存在环境上下文的矛盾。如果该内核用于笔记本等电池供电设备，开启是必须的；但如果你的目标设备是常插电运行的嵌入式开发板、服务器或需要极低网络延迟的工控机，**默认开启此项是一个常见的坑**，极易导致 SSH 连接频繁假死或网络吞吐量周期性暴跌。
- **`support CRDA` `[\*]`**
  - **定义**：中央监管域代理支持。用于读取不同国家的无线电频率管制数据库。
  - **分析**：必须开启。它约束网卡的发射频率和信道必须符合当地法律法规（例如避让军用或气象雷达频段）。关闭会导致合规性问题或部分信道无法扫描。

#### 4. 兼容性配置

- **`cfg80211 wireless extensions compatibility` `[\*]`**
  - **定义**：对已经淘汰的 `Wireless Extensions` (wext) 接口的后向兼容。
  - **分析**：当前为开启。这暴露出一种对老旧环境的妥协。如果你的系统镜像非常老旧，还在依赖十几年前的 `iwconfig` 等工具配置网络，开启它是必须的。但如果是现代 Linux 发行版（全面使用 `iw` 工具），这个选项纯属历史包袱，徒增内核代码体积，理应剔除。

#### 5. 协议栈实现

- **`Generic IEEE 802.11 Networking Stack (mac80211)` `<M>`**（当前高亮行）
  - **定义**：基于软件实现 MAC 层功能的标准 802.11 协议栈。市面上绝大多数不带独立处理器的普通 Wi-Fi 网卡（如典型的 Intel、Atheros 芯片）都强依赖此协议栈处理底层数据帧。
  - **分析**：当前配置为 `<M>`（模块化）。这展示了一种典型的系统工程权衡：将地基（cfg80211）打入内核，但将体积庞大且复杂的实际协议栈（mac80211）作为模块剥离。这有助于加快内核第一阶段的引导速度，后续再由 `udev` 探测到物理网卡时动态加载该协议栈。
- **`Minstrel` `[\*]`**
  - **定义**：mac80211 协议栈内置的最经典速率控制算法。
  - **分析**：负责根据空间信号衰减和丢包率，动态升降 Wi-Fi 传输速率（例如从 300Mbps 降档到 54Mbps 以保稳定）。必须开启，没有它网卡无法自适应复杂的电磁环境。

### 设备无线驱动配置界面

#### 网络设备支持

![image-20260409222520617](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20260409222520617.png)

这三项是一个整体的父子依赖关系，全部属于一个极其老旧的无线驱动体系：**Host AP 驱动**。

##### 1. 核心父选项

- **`IEEE 802.11 for Host AP (Prism2/2.5/3 and WEP/TKIP/CCMP) <\*>`**
  - **技术定义**：这是专门针对 **Intersil Prism2/2.5/3** 芯片组的古董级驱动程序（`hostap`）。在早期（2000年代初，802.11b时代），只有特定硬件支持“软AP”模式，这个驱动就是让这类老网卡能作为无线路由器（Access Point）使用的核心代码。
  - **配置现状与批判**：当前被配置为 `<*>`（静态编译进内核）。**这是一个严重的配置冗余。** 除非你的目标设备插着20多年前的 PCMCIA 或老式 PCI 接口的 Prism 网卡，否则将其硬编码进内核毫无意义，纯粹是在浪费内核内存空间、拖慢启动速度。现代 Linux 的 AP 功能早已由 `mac80211` 协议栈配合用户态的 `hostapd` 接管，支持绝大多数现代芯片。

##### 2. 子选项一：内存级固件加载

- **`Support downloading firmware images with Host AP driver [\*]`**
  - **技术定义**：允许该驱动程序在网卡初始化时，将固件映像（Firmware）从主机的根文件系统加载到 Prism 网卡的 **RAM（易失性内存）** 中。
  - **逻辑分析**：因为许多早期的 Prism 网卡自身没有搭载足够大的 Flash 芯片，或者出厂固件有严重 Bug，必须在每次系统上电时由宿主机重新“喂”给它运行代码。如果确实在使用这种网卡，此项必须开启，否则网卡无法工作。

##### 3. 子选项二：非易失性闪存烧写

- **`Support for non-volatile firmware download [\*]`**
  - **技术定义**：允许驱动程序不仅将固件加载到网卡的 RAM，还允许直接将其烧写（Flash）到网卡的 **非易失性存储器（如 EEPROM 或 Flash 芯片）** 中。
  - **配置现状与批判**：当前配置为开启。**这是一个带有一定危险性的底层功能选项。** 这相当于在内核驱动里内嵌了一个“硬件 BIOS 刷写工具”。一旦开启，用户空间程序可以通过特定接口修改网卡的底层固件。如果刷写过程中断电或输入了错误的固件映像，会**直接导致网卡变砖（永久性硬件损坏）**。在现代工程实践中，除非你正在对这些古董网卡进行硬件维修或固件逆向开发，否则绝不应该在生产或常规编译环境中开启此类带有硬件破坏风险的选项。

### usb网络支持

![image-20260409221018361](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20260409221018361.png)

#### USB Wireless Device Management support

- **本质**：这是一个属于 USB 通信设备类（CDC）的子驱动。
- **作用**：它专门用于为基于 USB 接口的**蜂窝网络模块**（即 3G/4G/5G 上网卡、LTE 通信模块等）提供**控制与管理通道**。
- **底层逻辑**：在现代蜂窝通信中，数据业务（上网流量）和控制业务（拨号、查信号、收发短信）是物理隔离的。`cdc-wdm` 不负责传输你的上网数据，它的职责是暴露出一个字符设备节点（通常是 `/dev/cdc-wdmX`），让用户空间的管理程序（如 `ModemManager`、`libqmi`、`libmbim`）能够通过高通 QMI 或 MBIM 协议与基带芯片进行指令交互。没有它，你的 4G/5G 模块将无法完成拨号注册网络的动作。

### 驱动编译Staging drivers

![image-20260409221644353](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20260409221644353.png)

这是**`Staging drivers`**（暂存驱动/实验性驱动）子系统，非内核主线的驱动（厂商驱动）会放在这里，但不同于自己编译驱动，这里的驱动是在内核官方源码树里。

高亮行下方紧跟的 `[*] Realtek RTL8188EU AP mode` 表明开启了该芯片的 SoftAP（软路由热点）支持。这在无屏物联网设备中是刚需，通常用于设备出厂后的首次手机直连配网（扫码配置 SSID 和密码）。由于是在 `Staging` 驱动上开启 AP 模式，其多带机量和长期吞吐稳定性通常极差，仅适合作为短时的配置通道。

#### 主要区别如下：

- **Staging 驱动**： 跟随内核版本一同发布。如果 Linux 内核修改了某个底层 API（比如电源管理或网络队列 API），内核维护者会使用自动化工具或手动顺便把 Staging 里的驱动也改掉。**它永远能和当前内核成功编译。**
- **树外驱动（OOT）**： 严重依赖特定版本的内核。原厂发布这包代码时，可能只针对 Linux 4.19 或 5.10 测试过 。如果你未来想把系统内核升级到 Linux 6.x，这个 OOT 驱动大概率会因为大量 API 废弃而面临**编译彻底崩溃**。你需要自己一行行去修 C 语言报错，这在嵌入式产品维护后期是巨大的噩梦。

### 开启SDIO驱动

![image-20260409222031017](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20260409222031017.png)

确保开启sdio驱动支持



### 开启4G支持

![image-20260425154301269](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20260425154301269.png)



这个功能是开始usb转串口功能

但编译过程报下面错误，缺少依赖库

```
fatal error: gmp.h: No such file or directory
```

可以执行下面命令

```
sudo apt-get update
sudo apt-get install libgmp-dev libmpc-dev libmpfr-dev
```



## 3.生成defconfig文件

当在图形界面选完并按下 `< Save >` 时，图形界面即刻死亡。它唯一的遗产就是会在源码根目录生成一个隐藏文本文件 `.config`

通过这个包含内核编译选项的文本文件生成编译选项指导文件defconfig

make ARCH=arm CROSS_COMPILE=arm-v01c02-linux-musleabi- savedefconfig





## 4.编译内核

将生成的defconfig文件拷贝为新的deconfig文件，执行cp defconfig arch/arm/configs/hi3516cv610_new_defconfig

这里换名是为了匹配后续编译，可以不改但后续编译的脚本文件里要改那里**hi3516cv610_new_defconfig**的文件名

进入bsp目录，重新执行make DEBUG=1 KERNEL_CFG=hi3516cv610_new_defconfig kernel (注：eMMC 介质需加上 BOOT_MEDIA=emmc 参数)

**KERNEL_CFG=hi3516cv610_new_defconfig**指定编译选项

## 5.开始编译



## menuconfig搜索出来的结果跳转

出现数字按数字跳转





## wifi驱动程序安装

| 功能                 | 需要    |
| -------------------- | ------- |
| 普通 WiFi（PSK）     | 基础    |
| 新驱动（nl80211）    | libnl   |
| 企业 WiFi（EAP/TLS） | OpenSSL |
| 老驱动（wext）       | WEXT    |



### 设备识别

需要先把wifi驱动执行把板子改到host模式

```
echo host > /proc/10300000.usb20drd/mode
```

**详情见接USB-4G的准备工作.md**



### openssl的编译

1.打开openssl-1.1.1w文件

2.执行config修改

| 参数                   | 作用       |
| ---------------------- | ---------- |
| linux-armv4            | 目标平台   |
| no-shared              | 静态库     |
| no-async               | 关闭异步   |
| --prefix               | 安装路径   |
| --cross-compile-prefix | 交叉编译器 |

```C
./Configure linux-armv4 no-shared no-async \
--prefix=$PWD/output \
--cross-compile-prefix=arm-v01c02-linux-musleabi-
```

3.make

4.make install



### libnl编译

1.打开文件

2.执行config修改

```C
./configure --host=arm-v01c02-linux-musleabi --prefix=$PWD/output --disable-shared
```

3.make

4.make install



### wpa_supplicant编译

1.wpa_supplicant没有提供.configure半⾃动⽣成Makefile⾸先需要复制其提供的defconfig为.config

```
cp defconfig .config
```

2.在.config中指定交叉编译器



### udhpcp用户态程序编译

可以使用busybox提供的包

```C
//在 open_source/busybox 目录下通过 make menuconfig 开启 udhcpc 功能。
Networking Utilites->udhcpc
```

```C
可以有两种方法
//一种是直接去下载编译好的静态编译包
//一种是对busybox进行简化只开启udhpcp功能，将其作为一个程序包使用，但这里还要开启大文件支持Support large files
//先关闭所有功能
make allnoconfig
//然后
Networking Utilites->udhcpc
Settings  ---> Support files (> 2 GB)
//因为在较新的 musl 库设计中，为了彻底解决 2038 年时间戳溢出问题，它强制规定 32 位系统下的文件偏移量类型（off_t）必须是 64位（8字节）。因为 BusyBox 的大文件支持被关了，它自己定义无符号偏移量（uoff_t）时用了 32位（4字节）。接着它在 libbb.h 里做了一个安全检查：sizeof(off_t) == sizeof(uoff_t)。结果 8 不等于 4，条件为假，试图声明一个大小为 -1 的数组，编译器当场崩溃
```





## wifi固件

```C
//放到以下的目录
/lib/firmware # find / -name "rtl8188eufw*"
/lib/firmware/rtlwifi/rtl8188eufw.bin

rtl8188eufw.bin
```



## wifi与网线切换

### 标准操作步骤

为了确保路由表不冲突，并在 32MB 内存里做到最干净的切换，请按以下顺序敲命令：

#### **1. 彻底关闭无线网络（释放内存）**

Bash

```
killall wpa_supplicant
ifconfig wlan0 down
```

#### **2. 激活有线网络并获取 IP** 插上网线，然后执行：

Bash

```
ifconfig eth0 up
udhcpc -i eth0
```

*（注意观察获取到的 IP，假设这次 eth0 获取到的是 `172.29.214.80`，此时板子和电脑都在同一个网段下了）*



#### **3.搜索 wifi**

```C
//动态库路径指定，也可放在系统库目录，/lib/
export LD_LIBRARY_PATH=/home/r8188eu/iwlist
//搜索 wifi
./iwlist wlan0 scanning
```

错误

```C
/home/r8188eu/iwlist # ./iwlist wlan0 scanning
Error loading shared library libiw.so.29: No such file or directory (needed by ./iwlist)
Error relocating ./iwlist: iw_enum_devices: symbol not found
Error relocating ./iwlist: iw_print_stats: symbol not found
Error relocating ./iwlist: iw_sockets_open: symbol not found
Error relocating ./iwlist: iw_freq_to_channel: symbol not found
Error relocating ./iwlist: iw_mwatt2dbm: symbol not found
Error relocating ./iwlist: iw_extract_event_stream: symbol not found
Error relocating ./iwlist: iw_print_pm_mode: symbol not found
Error relocating ./iwlist: iw_print_key: symbol not found
Error relocating ./iwlist: iw_ether_ntop: symbol not found
Error relocating ./iwlist: iw_print_bitrate: symbol not found
Error relocating ./iwlist: iw_dbm2mwatt: symbol not found
//缺少动态库libiw.so.29
//一种是编译时用静态编译就像openssl，libnl，wpa_supplicant编译时一样
//一种是上面的路径指定，并把libiw.so.29文件放进来
```



```c
/home/r8188eu/iwlist # ./iwlist wlan0 scanning
wlan0     No scan results
//没扫到wifi，检查时硬件连接问题
```



```
Successfully initialized wpa_supplicant
rfkill: Cannot get wiphy information
ioctl[SIOCSIWAP]: Operation not permitted
//本身不是大问题，怀疑是不支持工具但发现是硬件连接
//ai建议是驱动不支持，但其实不是
```



#### **4.启动wifi**

通过wpa_supplicant用户态程序启动，需要事先安装编译

**执行**

```C
./wpa_supplicant -D wext -i wlan0 -c /etc/wpa_supplicant.conf &
//通过wext去连接，这个接口在wifi驱动程序安装有介绍，要加&进行后台执行不然会占据当前命令行
//etc/wpa_supplicant为配置路径
wpa_supplicant -D nl80211 -i wlan0 -c /etc/wpa_supplicant.conf &
//编译过openssl跟libnl就可以用nl80211
```

wpa_supplicant内容

```
ctrl_interface=/var/run/wpa_supplicant
update_config=1

network={
    ssid="你的WiFi名"
    psk="你的WiFi密码"
    key_mgmt=WPA-PSK
}
```

防止musl系统下没有这个目录

```
mkdir -p /var/run/wpa_supplicant
```

`ctrl_interface` = 给 wpa_supplicant 开一个“控制接口”，让 wpa_cli 或其他程序能操作它

```C
wpa_cli -i wlan0 status
//可以查看状态
```

#### **5.获取ip**

```
./udhcpc -i wlan0
```

这个工具是udhcpc用户态程序，可以在busybox里获取并编译

但还需要一个脚本simple.script，目录在**busybox/busybox-1.34.1/examples/udhcp**

udhcpc的[CSDN参考文献](https://blog.csdn.net/guoyaoyao1990/article/details/12624439?ops_request_misc=elastic_search_misc&request_id=b67b3c4a4d797c7b74d8ef0ebc39aadb&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-4-12624439-null-null.142^v102^pc_search_result_base5&utm_term=UDHCPC%E5%AE%89%E8%A3%85&spm=1018.2226.3001.4187)



##### 报错

```

~ # ps | grep udhcpc
  685 root      0:00 udhcpc -i wlan0 -b -p /var/run/netfailover/udhcpc.wlan0.pid -s /usr/share/udhcpc/netfailover.script
  704 root      0:00 udhcpc -i eth0 -b -p /var/run/netfailover/udhcpc.eth0.pid -s /usr/share/udhcpc/netfailover.script
 1109 root      0:00 grep udhcpc
~ # cat /var/run/netfailover/lease.eth0
cat: can't open '/var/run/netfailover/lease.eth0': No such file or directory
~ # /usr/sbin/udhcpc -i eth0
udhcpc: started, v1.34.1
Clearing IP addresses on eth0, upping it
udhcpc: broadcasting discover
udhcpc: broadcasting discover
udhcpc: broadcasting discover

```

你现在有 eth0 的 DHCP 进程：

```
udhcpc -i eth0 ... 
```

但没有：

```
/var/run/netfailover/lease.eth0 
```

手动跑也一直是：

```
udhcpc: broadcasting discover 
```

说明 eth0 发出了 DHCP Discover，但没有收到 DHCP Offer。

所以不是 netfailover 优先级问题，而是：

```
eth0 所在网络没有 DHCP 响应 
```

常见原因：

1. eth0 接到电脑直连，电脑没开 DHCP 服务；
2. eth0 接到交换机，但交换机上游没有路由器/DHCP；
3. 网线物理 Link Up，但对端不分配 IP；
4. eth0 所在网络需要静态 IP；
5. 网口接的是调试口/隔离网段，没有 DHCP。



#### **5. 重新尝试 TFTP 传输** 此时底层通道变成了极其稳定的以太网，可以直接再次执行你的传输命令：

Bash

```
tftp -g -r rtsp 172.29.214.5
```



## 用户视频接收

FFmpeg 拉取 / 推流 RTSP 常用命令给你整理一下（直接能用👇）

------

## 🎥 1. 拉取 RTSP 流（播放 / 保存）

### 👉 直接播放（用 ffplay）

```bash
ffplay rtsp://username:password@ip:port/path
ffplay rtsp://板子IP/test
ffplay rtsp://10.188.138.79/test
ffplay rtsp://192.168.1.80/test
ffmpeg -rtsp_transport tcp -i rtsp://10.188.138.79/test
```

------

### 👉 保存为文件

```bash
ffmpeg -i rtsp://username:password@ip:port/path -c copy output.mp4
```

👉 说明：

- `-c copy`：不重新编码，直接拷贝流（最省 CPU）

------

### 👉 指定 TCP（很常用，避免丢包）

```bash
ffmpeg -rtsp_transport tcp -i rtsp://username:password@ip:port/path -c copy output.mp4
```

### 👉 推本地文件到 RTSP 服务端

```bash
ffmpeg -re -i input.mp4 -c copy -f rtsp rtsp://ip:port/live/stream
```

👉 说明：

- `-re`：按实时速度读取（很关键）
- `-f rtsp`：指定输出格式

------

### 👉 推摄像头

```bash
ffmpeg -f v4l2 -i /dev/video0 -c:v libx264 -f rtsp rtsp://ip:port/live/stream
```

## 

### 👉 降低延迟（常用）

```bash
ffmpeg -rtsp_transport tcp -fflags nobuffer -flags low_delay -i rtsp://xxx
```

------

### 👉 指定 UDP

```bash
ffmpeg -rtsp_transport udp -i rtsp://xxx
```

------

## 👍 最常用一条（直接记这个）

```bash
ffmpeg -rtsp_transport tcp -i rtsp://xxx -c copy output.mp4
```

------

如果你是用在你那个 Hi3516 板子上，我可以帮你写一条**最适配你环境的命令（比如推流到 NVR / VLC / 自建 RTSP server）**。





## 网络管理

需要四个文件实现网络管理

```C
/etc/netfailover.conf
/usr/share/udhcpc/netfailover.script
/usr/sbin/netfailover.sh
/etc/init.d/S81netfailover
```



```
chmod +x /usr/sbin/netfailover.sh
chmod +x /usr/share/udhcpc/netfailover.script
chmod +x /etc/init.d/S81netfailover
```

这 4 个文件分工是这样的：

### **1. `/etc/netfailover.conf`**

配置文件。

它负责告诉脚本：

```text
哪个是主网卡：wlan0
哪个是备用网卡：eth0
WiFi 配置文件在哪里
检测间隔是多少
DNS 用什么
要不要 ping 外网检测
```

比如这里：

```sh
WIFI_IF=wlan0
BACKUP_IF=eth0
WIFI_PRIORITY=10
ETH_PRIORITY=20
WIFI_CONF=/etc/wpa.conf
```

意思就是：

```text
优先用 wlan0
wlan0 不可用再用 eth0
```

以后你想改优先级，不用改脚本，只改这个配置文件。

---

### **2. `/usr/share/udhcpc/netfailover.script`**

DHCP 回调脚本。

`udhcpc` 获取 IP 的时候会调用它。

它主要做两件事：

```text
1. 给网卡配置 DHCP 拿到的 IP
2. 把 DHCP 拿到的网关和 DNS 记录到 /var/run/netfailover/lease.xxx
```

比如 wlan0 拿到 IP 后，它会生成：

```text
/var/run/netfailover/lease.wlan0
```

里面类似：

```sh
IP='10.188.138.79'
SUBNET='255.255.255.0'
ROUTER='10.188.138.1'
DNS='10.188.138.1'
```

注意：这个脚本 **不负责切默认路由**。  
这样做是为了避免 `wlan0` 和 `eth0` 两个 DHCP 同时抢默认路由。

---

### **3. `/usr/sbin/netfailover.sh`**

核心主脚本。

它是真正负责“自动切换”的守护脚本。

它循环做这些事：

```text
1. 启动 wlan0 的 wpa_supplicant
2. 启动 wlan0 的 udhcpc
3. 启动 eth0 的 udhcpc
4. 检查 wlan0 是否连上 WiFi 并拿到 IP
5. 检查 eth0 是否插线并拿到 IP
6. wlan0 可用就切到 wlan0
7. wlan0 不可用才切到 eth0
8. wlan0 恢复后再切回 wlan0
```

它切换网络的关键动作是改默认路由：

```sh
ip route replace default via 网关 dev wlan0 metric 10
```

或者：

```sh
ip route replace default via 网关 dev eth0 metric 20
```

它还会记录当前网络状态：

```text
/var/run/netfailover/current
```

内容类似：

```text
iface=wlan0
ip=10.188.138.79
gateway=10.188.138.1
metric=10
```

你可以用这个看当前走哪个网：

```sh
cat /var/run/netfailover/current
```

---

### **4. `/etc/init.d/S81netfailover`**

开机启动脚本。

你的系统启动时一般会执行：

```text
/etc/init.d/S00xxx
/etc/init.d/S80network
/etc/init.d/S81netfailover
...
```

这个文件的作用就是开机自动启动主脚本：

```sh
/usr/sbin/netfailover.sh &
```

所以你不用每次手动运行。

它也支持：

```sh
/etc/init.d/S81netfailover start
/etc/init.d/S81netfailover stop
/etc/init.d/S81netfailover restart
```

---

### 整体关系是：

```text
S81netfailover
      |
      v
netfailover.sh
      |
      +--> 启动 wpa_supplicant 连接 WiFi
      |
      +--> 启动 udhcpc 获取 wlan0/eth0 IP
                 |
                 v
          netfailover.script 记录 IP/网关/DNS
      |
      v
根据优先级切默认路由
```

简单说：

```text
/etc/netfailover.conf                  配置优先级
/usr/share/udhcpc/netfailover.script   DHCP 拿 IP、记录网关
/usr/sbin/netfailover.sh               判断哪个网能用、切路由
/etc/init.d/S81netfailover             开机自动启动
```

你实际最常看的就是：

```sh
cat /var/run/netfailover/current
ip route
ifconfig wlan0
ifconfig eth0
```

最常改的只有：

```sh
/etc/netfailover.conf
```





## WiFi自动启动脚本

只执行到启动wifi

```
/usr/sbin/start_usb_wifi.sh
```

```C
//默认路由现在走谁
ip route
```



# 4G

#### 看是否入网

```
echo -e "AT+CGATT?\r\n" > /dev/ttyUSB3
```

------

#### 看运营商

```
echo -e "AT+COPS?\r\n" > /dev/ttyUSB3
```

------

#### 看信号

```
echo -e "AT+CSQ\r\n" > /dev/ttyUSB3
```

cat /dev/ttyUSB3 &

可以。你现在是 **EC20 + usb0 网卡模式**，需要执行的命令分成 4 类：**清理串口、查模块状态、拨号联网、查 Linux 网络**。

------

## 1. 先清理后台 cat

因为你之前执行了：

```bash
cat /dev/ttyUSB3 &
```

它会一直占用/读取串口，先停掉：

```bash
jobs
kill %1
```

如果不确定，就用：

```bash
killall cat
```

作用：
停止后台读取串口，避免后面 AT 命令输出被搅乱。

------

## 2. 确认 AT 口是否正常

你现在看起来 `/dev/ttyUSB3` 可以回 AT，所以先用它。

```bash
echo -e "AT\r" > /dev/ttyUSB3
sleep 1
cat /dev/ttyUSB3
```

作用：
确认这个串口能和 EC20 通信。

正常返回：

```text
OK
```

如果 `cat` 不退出，就按：

```text
Ctrl+C
```

------

## 3. 查询 SIM 卡状态

```bash
echo -e "AT+CPIN?\r" > /dev/ttyUSB3
sleep 1
cat /dev/ttyUSB3
```

作用：
看 SIM 卡是否可用。

正常返回：

```text
+CPIN: READY
OK
```

如果不是 `READY`，说明 SIM 卡、卡槽、PIN 锁有问题。

------

## 4. 查询信号强度

```bash
echo -e "AT+CSQ\r" > /dev/ttyUSB3
sleep 1
cat /dev/ttyUSB3
```

作用：
查看信号强度。

你之前返回：

```text
+CSQ: 16,99
OK
```

这个信号可以，不算差。

一般判断：

```text
+CSQ: 0~5     信号很差
+CSQ: 10~31   可用
+CSQ: 99      未知/异常
```

------

## 5. 查询是否注册到运营商

```bash
echo -e "AT+COPS?\r" > /dev/ttyUSB3
sleep 1
cat /dev/ttyUSB3
```

作用：
查看是否选上运营商。

比较正常的返回类似：

```text
+COPS: 0,0,"CHINA MOBILE",7
OK
```

你之前返回：

```text
+COPS: 0
OK
```

这个不太完整，说明可能还没正常选网，或者返回被你前面的 `cat &` 搅乱了，需要重新查。

------

## 6. 查询是否附着到数据网络

```bash
echo -e "AT+CGATT?\r" > /dev/ttyUSB3
sleep 1
cat /dev/ttyUSB3
```

作用：
看模块是否附着到分组数据网络。

正常：

```text
+CGATT: 1
OK
```

异常：

```text
+CGATT: 0
OK
```

如果是 `0`，模块还没真正接入 4G 数据网络，后面 ping 外网肯定不通。

------

## 7. 设置 APN

按你的 SIM 卡运营商选一个。

中国移动：

```bash
echo -e 'AT+CGDCONT=1,"IP","CMNET"\r' > /dev/ttyUSB3
sleep 1
cat /dev/ttyUSB3
```

中国联通：

```bash
echo -e 'AT+CGDCONT=1,"IP","3GNET"\r' > /dev/ttyUSB3
sleep 1
cat /dev/ttyUSB3
```

中国电信：

```bash
echo -e 'AT+CGDCONT=1,"IP","CTNET"\r' > /dev/ttyUSB3
sleep 1
cat /dev/ttyUSB3
```

作用：
告诉 EC20 用哪个 APN 建立数据连接。

正常返回：

```text
OK
```

------

## 8. 激活 PDP 数据连接

```bash
echo -e "AT+CGACT=1,1\r" > /dev/ttyUSB3
sleep 2
cat /dev/ttyUSB3
```

作用：
激活第 1 路数据连接。

正常返回：

```text
OK
```

------

## 9. EC20 网卡模式启动联网

```bash
echo -e "AT+QNETDEVCTL=1,1,1\r" > /dev/ttyUSB3
sleep 3
cat /dev/ttyUSB3
```

作用：
EC20 专用命令，用来启动 USB 网卡模式下的数据连接。

正常返回：

```text
OK
```

------

## 10. 查询 EC20 数据连接状态

```bash
echo -e "AT+QIACT?\r" > /dev/ttyUSB3
sleep 1
cat /dev/ttyUSB3
```

作用：
查看 EC20 当前 PDP 是否已经激活，以及是否拿到公网/运营商侧 IP。

正常类似：

```text
+QIACT: 1,1,1,"10.xxx.xxx.xxx"
OK
```

如果没有 `+QIACT` 或显示未激活，说明模块内部还没拨号成功。

------

## 11. 重新获取 usb0 IP

```bash
ifconfig usb0 up
udhcpc -i usb0
```

作用：
让 Linux 这边从 EC20 的虚拟网卡重新获取 IP、网关、DNS。

------

## 12. 检查路由

```bash
route -n
```

作用：
确认默认路由是否走 `usb0`。

正常应该有：

```text
0.0.0.0    192.168.225.1    0.0.0.0    UG    usb0
```

你这里已经有了。

------

## 13. 测试连通性

先测模块网关：

```bash
ping 192.168.225.1
```

作用：
确认板子到 EC20 网卡侧通不通。你这里已经能通。

再测外网 IP：

```bash
ping 8.8.8.8
```

作用：
确认 4G 数据是否真正出网。

最后测域名：

```bash
ping www.baidu.com
```

作用：
确认 DNS 是否正常。

如果 `8.8.8.8` 能通，但域名不通，补 DNS：

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

------



 # 4g网络排查步骤

1.进入AT命令模式

```C
///dev/ttyUSB3是AT控制端口
microcom -s 115200 /dev/ttyUSB3
```

2.查看是否识别

```
//当前是否被识别
AT+CPIN?
+CPIN: READY
//运营商信息
AT+CGDCONT?
```

3.手动识别

```
AT+CFUN=4
```

这会让模块进入飞行模式。基带处理器正常工作，USB 正常工作，但底层的射频接收和发射电路被彻底强制切断电源。

```
请保持移动卡插在里面的状态，依次在串口输入以下指令：

第一步：强制重启协议栈与 SIM 卡接口

输入：AT+CFUN=0

(期望结果：返回 OK。此时模块彻底休眠，SIM 卡掉电。)

输入：AT+CFUN=4

(期望结果：返回 OK。此时模块唤醒，重新给 SIM 卡上电并读取，但严禁射频发射。)
```

4.重启模块

```C
//前面的搜网模式改完以后，要让模块重新启动，再按新的方式搜网。
//这里注意：AT+CFUN=4禁用飞行模式后卡成功识别+CPIN: READY，但执行重启后断开，可以怀疑是供电不足，供电线的问题
AT+CFUN=1
//，1是重启
AT+CFUN=1,1
```

5.配置运营商商

```
//移动
AT+CGDCONT=1,"IP","CMNET"
//电信
AT+CGDCONT=1,"IP","ctnet"
```

6.信息查询

```
AT+QNWINFO  
看当前网络信息。  
一般会告诉你它现在是不是在 LTE，上了哪个运营商，频点大概是什么。

AT+CSQ
信号强度

AT+QENG="servingcell"
看当前驻留小区信息。  
这个很关键，它能看出模块有没有搜到基站、信号强不强、驻留在哪个小区。  
如果这里基本没内容，通常就是没搜到网或天线/频段有问题。

`AT+QCFG="band"`  
看当前启用了哪些频段。  
如果频段被改乱了，比如把国内能用的 LTE band 关掉了，就会一直搜不上。

AT+CEREG?  
看 LTE 网络注册状态。  
常见值意思是：

- `0,2`：正在搜网
- `0,1`：已注册，本地网
- `0,5`：已注册，漫游网
- `0,3`：注册被拒绝

你现在一直是 `0,2`，所以模块还没真正上网。

AT+CGATT? 
看数据业务是否附着。  
`1` 表示已经附着，可以走数据。  
`0` 表示还没附着。  
你前面是 `0`，这就是为什么上不了网。

AT+CGPADDR
看 PDP 地址，也就是模块真正拿到的数据 IP。  
如果是 `0.0.0.0`，说明还没拿到运营商分配的地址。  
如果这里出来一个正常 IP，才说明蜂窝数据链路真的起来了。

你可以把它们理解成一条链：

1. 先搜到网：`CEREG`
2. 再附着数据：`CGATT`
3. 再拿到数据 IP：`CGPADDR`
4. 最后 Linux 才可能真的上网
```

