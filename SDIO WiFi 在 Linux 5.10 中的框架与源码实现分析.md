# SDIO WiFi 在 Linux 5.10 中的框架与源码实现分析

## 1. 目标与结论

本文聚焦 **Linux 5.10 内核下 SDIO WiFi 的软件框架与主线源码实现方式**。这里的“SDIO WiFi”并不是单独一个子系统，而是由三层共同组成：

1. **MMC/SDIO 总线层**：负责枚举 SDIO function、设备匹配、寄存器/块读写、中断与电源管理。
2. **WiFi 芯片驱动层**：负责固件下载、命令/数据收发、设备状态机、异常恢复。
3. **802.11 无线协议接口层**：向 Linux 无线栈暴露能力，通常走 **cfg80211**；若是 SoftMAC 设备则再接 **mac80211**，若是 FullMAC 设备则驱动自己处理大部分 802.11 逻辑。

因此，分析 SDIO WiFi 时，最关键的不是只盯 `drivers/mmc/`，而是要沿着这条链路看：

**SDIO bus → 芯片驱动 probe → firmware/init → netdev/wiphy 注册 → TX/RX/IRQ/PM**

------

## 2. Linux 5.10 下的总体分层

### 2.1 总体架构图

```text
User Space
  ├─ wpa_supplicant / hostapd / NetworkManager
  ├─ nl80211
  └─ ip/ifconfig/iw

Kernel Wireless Stack
  ├─ cfg80211         <- 所有现代无线驱动都要对接
  └─ mac80211         <- 仅 SoftMAC 驱动使用

WiFi Driver
  ├─ FullMAC: brcmfmac, mwifiex(很多场景偏 FullMAC/firmware-heavy)
  └─ SoftMAC: 芯片较少直接走 SDIO + mac80211 的主线方案

SDIO/MMC Core
  ├─ drivers/mmc/core/
  ├─ sdio bus
  └─ host controller driver

Hardware
  ├─ SDIO host controller
  └─ WiFi chip (func0/func1 ...)
```

### 2.2 为什么要区分 FullMAC / SoftMAC

这是理解源码结构的关键：

- **FullMAC**：关联、认证、扫描、功耗控制等大量 802.11 逻辑在芯片固件里。Linux 驱动主要做“主机到固件”的命令与数据搬运、状态同步、事件上报。**brcmfmac** 很典型。
- **SoftMAC**：802.11 的管理/控制逻辑更多在 Linux `mac80211` 中，驱动提供硬件能力与收发钩子。SDIO WiFi 主线里常见程度低于 PCIe/USB 方案。

所以很多 SDIO WiFi 驱动源码看起来不像“纯网络驱动”，而更像“总线 + 固件协议 + 无线接口包装”。

------

## 3. MMC/SDIO 总线层：设备是怎样被发现并绑定到驱动的

### 3.1 入口位置

Linux 5.10 的 SDIO 支撑位于 MMC 子系统中，文档入口在内核 MMC/SD/SDIO 驱动实现说明。总线注册与 function device 的创建/匹配主要在 `drivers/mmc/core/sdio_bus.c`。

### 3.2 核心对象

#### (1) `struct mmc_host`

表示 SoC 上的 SD/MMC 主控制器。

#### (2) `struct mmc_card`

表示整张卡。

#### (3) `struct sdio_func`

表示 SDIO 卡上的某个 function。WiFi 常常工作在 **function 1**，function 0 用于公共 CIS/配置访问。

#### (4) `struct sdio_driver`

具体的 SDIO 设备驱动。驱动通过 ID table 与 function 进行匹配，并实现：

- `probe`
- `remove`
- `drv.pm` / suspend / resume
- 可选 `shutdown`

### 3.3 注册与匹配流程

总线代码中，`__sdio_register_driver()` 会把驱动注册到 `sdio_bus_type`。而 `sdio_bus_type` 提供 `.match/.probe/.remove/.shutdown/.pm` 等总线级回调。

这意味着 SDIO WiFi 驱动的起点通常是：

```c
static struct sdio_driver xxx_driver = {
    .name = "xxx_sdio",
    .id_table = xxx_ids,
    .probe = xxx_probe,
    .remove = xxx_remove,
    .drv = {
        .pm = &xxx_pm_ops,
    },
};
module_sdio_driver(xxx_driver);
```

系统枚举到 SDIO function 后，会为其创建 `struct sdio_func` 设备，再通过 `sdio_bus_type.match` 匹配 `id_table`，命中后调用驱动的 `probe()`。

### 3.4 典型 SDIO 操作 API

驱动日常最常使用这些接口：

- `sdio_claim_host()` / `sdio_release_host()`：串行化访问 host
- `sdio_enable_func()` / `sdio_disable_func()`：使能/关闭 function
- `sdio_set_block_size()`：设置块传输大小
- `sdio_readb()` / `sdio_writeb()`：单字节寄存器访问
- `sdio_memcpy_toio()` / `sdio_memcpy_fromio()`：内存窗口传输
- `sdio_writesb()` / `sdio_readsb()`：块模式读写
- `sdio_claim_irq()` / `sdio_release_irq()`：中断注册
- `sdio_set_drvdata()` / `sdio_get_drvdata()`：私有数据绑定

### 3.5 为什么 `claim_host` 很重要

MMC/SDIO 控制器通常不能被多个线程随意并发访问。Linux 用 `sdio_claim_host()` 保护一次总线事务期间的主机独占，这也是你在很多源码中频繁看到“先 claim，再 read/write，最后 release”的原因。

------

## 4. 无线栈：cfg80211 / mac80211 在 SDIO WiFi 中扮演什么角色

### 4.1 cfg80211 是统一入口

在 Linux 5.10 中，现代无线驱动都应对接 **cfg80211**。它负责：

- 与用户态 `nl80211` 对接
- 设备能力、信道、扫描、连接、AP 等配置接口
- 监管域（regulatory）相关约束
- `wiphy` / `wireless_dev` 抽象

### 4.2 `wiphy` 与 netdev

无线设备最终要向系统暴露：

- 一个 `wiphy`：物理无线设备抽象
- 一个或多个 `net_device`
- 对于虚拟接口，关联 `struct wireless_dev`

对于 SDIO FullMAC 驱动，流程往往是：

1. SDIO `probe()` 成功
2. 与芯片固件建立控制通道
3. 获取芯片能力
4. 注册 `wiphy`
5. 创建/注册 `net_device`
6. 支持扫描、连接、收发数据

### 4.3 SoftMAC 才会重度依赖 mac80211

如果驱动是 SoftMAC，就会注册 `ieee80211_hw`，由 `mac80211` 统一处理大量 802.11 管理逻辑。

而大量 SDIO WiFi 主线驱动并不走这种模式，而是：

- 对外仍然接入 `cfg80211`
- 但内部通过固件协议完成扫描、认证、关联、漫游等操作

这就是为什么你分析 SDIO WiFi 时，经常发现 **真正复杂的代码在“bus + firmware protocol + event handling”**，而不是 `mac80211` 回调本身。

------

## 5. 典型源码样板：SDIO WiFi 驱动的 probe 主流程

无论是 Broadcom 还是 Marvell/NXP 一类 SDIO WiFi 驱动，`probe()` 大体都会走下面这条链：

```text
sdio bus match
  -> driver probe(func, id)
      -> 分配 private/card/adapter 结构
      -> sdio_enable_func(func)
      -> sdio_set_block_size(func, ...)
      -> sdio_claim_irq(func, irq_handler)
      -> 初始化寄存器/IO port
      -> 下载/唤醒 firmware
      -> 建立 command/data path
      -> 注册 cfg80211 / netdev
      -> 启动 TX/RX 线程/worker/DPC
```

典型失败回滚则相反：

```text
unregister netdev/wiphy
release irq
disable func
free private data
```

------

## 6. 典型实现一：mwifiex SDIO（更容易看清总线调用）

`drivers/net/wireless/marvell/mwifiex/sdio.c` 是很好的 SDIO WiFi 参考实现，因为总线动作比较直观。

### 6.1 probe 的核心动作

`mwifiex_sdio_probe()` 的职责非常标准：

- 分配 `struct sdio_mmc_card`
- 保存 `func`
- 设置 quirk / 固件信息 / 卡级能力
- `sdio_enable_func(func)`
- 后续继续设备初始化、注册逻辑接口

从源码风格能看出，它把 **“SDIO card 抽象”** 和 **“无线 adapter 抽象”** 分开：

- `sdio_mmc_card`：偏总线和卡级资源
- `mwifiex_adapter`：偏无线协议和设备状态

这是很多成熟 SDIO 驱动常见的分层方式。

### 6.2 寄存器与数据收发接口

`mwifiex` 中非常典型的几个函数：

- `mwifiex_write_reg()` / `mwifiex_read_reg()`
- `mwifiex_write_data_sync()`
- `mwifiex_read_data_sync()`
- `mwifiex_sdio_read_fw_status()`
- `mwifiex_check_fw_status()`

这些函数揭示了 SDIO 驱动的核心事实：

1. **控制面**：小寄存器读写，用 `sdio_readb/writeb`
2. **数据面**：大块数据搬运，用 `sdio_readsb/writesb`
3. **状态机**：通过若干状态寄存器轮询或中断判断 firmware ready、端口可用、异常状态

### 6.3 读写路径的本质

`mwifiex_write_data_sync()`/`read_data_sync()` 一类函数通常会：

- 根据端口/长度决定 **byte mode** 还是 **block mode**
- 计算 block size / block count
- 组装 IO port
- `sdio_claim_host()`
- `sdio_writesb()` 或 `sdio_readsb()`
- `sdio_release_host()`

也就是说，**SDIO WiFi 的“发包/收包”从总线角度看就是“往某个 function 的某个 port 搬运一块共享内存/窗口数据”**，而具体这是命令还是网络帧，要由上层固件协议格式决定。

### 6.4 firmware ready 检查

很多 SDIO WiFi 芯片上电后并不是立即可用，驱动常要：

- 下载固件
- 等待 firmware ready 标志
- 必要时增加额外 delay

`mwifiex_check_fw_status()` 这类函数就是典型例子：读取状态寄存器并轮询，直到 firmware 进入 ready 状态。这个阶段如果失败，驱动 probe 往往整体失败。

### 6.5 中断与工作队列

SDIO 中断通常只做轻量级处理：

- 读取中断状态
- 屏蔽/确认中断
- 调度 worker/tasklet/thread

后续再在 worker 中：

- 拉取 RX 包
- 读取事件
- 更新电源/睡眠状态
- 继续下发 TX

这是典型“上半部快，重活下半部做”的设计。

------

## 7. 典型实现二：brcmfmac SDIO（FullMAC 的代表）

Broadcom/Cypress 的 `brcmfmac` 是 Linux 主线里极典型的 **SDIO + FullMAC** 实现。其 SDIO 代码主要在：

- `drivers/net/wireless/broadcom/brcm80211/brcmfmac/sdio.c`
- 配套公共层如 `core.c`、`common.c`、`firmware.c`、`bcdc.c`

### 7.1 从头文件依赖就能看出分层

在 `sdio.c` 中可以看到它同时包含：

- `linux/mmc/sdio.h`
- `linux/mmc/sdio_func.h`
- `linux/mmc/card.h`
- `firmware.h`
- `core.h`
- `common.h`
- `bcdc.h`

这已经非常说明问题：

1. 下面接 SDIO/MMC
2. 上面接 firmware 下载
3. 中间有 core/common
4. 再上一层是 Broadcom 的 BCDC 协议

### 7.2 brcmfmac 的核心思想

它不是把 802.11 管理逻辑放在内核里自己跑，而是：

- host 驱动负责和 dongle firmware 说话
- 控制命令走 BCDC / dongle protocol
- 数据帧通过 SDIO 管道传输
- 固件把扫描/关联/漫游/功耗等复杂逻辑做掉
- 驱动将结果映射到 cfg80211 接口

因此从代码阅读角度，`brcmfmac/sdio.c` 更像：

**总线适配器 + 固件 transport 层**

### 7.3 你在 brcmfmac 里要重点盯的对象

读这类驱动，建议优先找到这些概念：

- `sdiodev` / bus 对象
- frame/packet queue
- 中断处理与 DPC/worker
- control path（dcmd）
- data path（TX/RX）
- glom/聚合、block transfer
- firmware 下载与 NVRAM 配置

### 7.4 brcmfmac 的典型分层路径

可粗略理解为：

```text
cfg80211 request
  -> brcmfmac core
      -> BCDC/control command
          -> SDIO transport
              -> sdio read/write/block transfer
                  -> chip firmware
```

而收到事件/数据时则反向：

```text
SDIO IRQ / polling / DPC
  -> 从卡上取回 frame/event
  -> 解析 BCDC / firmware event
  -> 上报 cfg80211 / netif_rx
```

### 7.5 brcmfmac 为什么常见线程/worker/DPC

因为 SDIO 设备在中断上下文里不适合做太重的事情，且很多芯片还有：

- mailbox/doorbell 机制
- flow control
- frame aggregation
- 信用额度/队列水位

所以常见设计是：

- IRQ 中只记录“有事了”
- 真正的读卡、取包、处理事件放到 DPC/kthread/workqueue

这也是很多 FullMAC SDIO 驱动源码“看起来比较绕”的原因：它们不是单纯 `ndo_start_xmit()` → DMA → 完事，而是多级状态机与异步 worker 协作。

------

## 8. SDIO WiFi 的收发主链路

## 8.1 TX 发送链路

发送路径通常可概括为：

```text
Socket
 -> netdev xmit
 -> driver tx enqueue
 -> 封装 firmware/device header
 -> 选择 TX port / queue
 -> sdio_claim_host()
 -> sdio_writesb() / memcpy_toio()
 -> sdio_release_host()
 -> 等待 firmware credit / tx complete / flow control update
```

关键点：

- **不是直接发 802.11 空口帧**，很多 FullMAC 驱动发的是“host-to-firmware 协议帧”
- 常需要附加长度、优先级、端口号、序号、对齐填充
- 往往有 **flow control / credit**，否则 host 发太快会淹没 chip FIFO

## 8.2 RX 接收链路

接收路径通常是：

```text
chip raises SDIO interrupt
 -> irq handler / wake thread
 -> 读取中断状态寄存器
 -> 判断是 data / event / mailbox
 -> 从 RX port 读出一个或多个 frame
 -> 解析协议头
 -> 区分 control event / data frame
 -> 上报 cfg80211 或 netif_rx/netif_receive_skb
```

关键点：

- RX 不一定一中断一包，可能一次拉多包
- 常见聚合读取，减少 SDIO 事务次数
- 某些驱动会区分 event packet 与 data packet

------

## 9. 中断、轮询、线程化处理

### 9.1 为什么 SDIO WiFi 特别依赖异步处理

SDIO 总线吞吐与时延都不如 PCIe；同时寄存器/数据搬运有较强事务性。因此驱动通常会把以下工作异步化：

- RX 拉包
- TX 队列推进
- firmware event 处理
- 睡眠唤醒状态切换
- 错误恢复

### 9.2 常见模式

#### 模式 A：IRQ + worker

中断里只标记事件，worker 中完成大量 I/O。

#### 模式 B：IRQ + 线程/DPC

类似网络驱动里的 deferred procedure call 概念。

#### 模式 C：轮询辅助

某些异常或初始化阶段不完全依赖 IRQ，而是轮询状态寄存器。

------

## 10. 固件下载与 NVRAM/校准

这是 SDIO WiFi 驱动区别于很多普通网卡驱动的地方。

### 10.1 为什么需要 firmware

大量 SDIO WiFi 芯片上电后只是一个“半初始化执行体”，需要：

- 下载主 firmware
- 提供 NVRAM / board data / 校准参数
- 之后芯片才真正能扫描、连网、发包

### 10.2 典型流程

```text
probe
 -> chip identify
 -> request_firmware()
 -> write firmware to device memory/port
 -> 启动 firmware
 -> 等待 ready 标志
 -> 下载 NVRAM / board settings
 -> 拉起 network interface
```

### 10.3 阅读源码时的重点

重点找这些内容：

- `request_firmware()`
- firmware image / nvram parsing
- boot status / ready status
- timeout / retry / fallback
- reset / watchdog / trap dump

很多“驱动起不来”的根因，其实不在 SDIO 总线本身，而在：

- 固件文件找不到
- board 文件不匹配
- block size / clock / timing 不稳定
- 上电时序错误
- 中断或唤醒 GPIO 配置错误

------

## 11. 电源管理：suspend / resume / host sleep

SDIO WiFi 在嵌入式系统里几乎一定涉及电源管理。

### 11.1 Linux 层的 PM

驱动会实现：

- system suspend / resume
- runtime PM（部分场景）
- WOWLAN / host sleep 相关逻辑

### 11.2 SDIO 特有注意点

- suspend 时可能需要告诉 firmware 进入 host sleep
- resume 时先恢复 SDIO function，再取消 host sleep
- 某些平台 resume 后需要重新切总线宽度/时钟/调谐
- 中断唤醒能力要结合 host controller 能力与板级 GPIO

### 11.3 为什么 5.10 上经常看到 SDIO WiFi PM bug

因为它跨了三层：

1. host controller resume 是否可靠
2. SDIO card/function 状态是否保留
3. WiFi firmware 是否真的醒来并与 host 重新同步

只要任一层有瑕疵，就会表现为：

- resume 后扫描失败
- first packet timeout
- CMD52/CMD53 错误
- IRQ 丢失
- 总线重新调谐失败

------

## 12. Device Tree / 板级资源在 SDIO WiFi 中的作用

很多嵌入式平台上，SDIO WiFi 不只是“插卡即用”，而是焊死在板上，因此还需要板级配合：

- `vmmc` / `vqmmc` 供电
- `reset-gpios`
- `interrupts` / `wakeup-gpios`
- non-removable
- keep-power-in-suspend
- mmc-pwrseq
- bus-width
- max-frequency

驱动本身可能也会从 OF 节点中读兼容串和补充配置。`mwifiex` 就有 `of_match_table` 与 OF probe 逻辑。

这解释了为什么实际调试时，很多问题虽然表现为 WiFi 不工作，但根因可能是 **DTS、regulator、pwrseq、GPIO 时序**。

------

## 13. Linux 5.10 下阅读源码的推荐路径

如果你要真正“啃源码”，建议按下面顺序读，而不是一上来就钻某个 4000 行 `sdio.c`：

### 第一步：先看 SDIO bus

- `drivers/mmc/core/sdio_bus.c`
- 关注：`__sdio_register_driver()`、`sdio_add_func()`、bus `probe/remove`

目标：理解设备为何会进入 WiFi 驱动的 `probe()`。

### 第二步：看目标驱动的 `probe/remove`

例如：

- `drivers/net/wireless/marvell/mwifiex/sdio.c`
- `drivers/net/wireless/broadcom/brcm80211/brcmfmac/sdio.c`

目标：弄清私有对象、block size、irq、firmware、netdev/wiphy 初始化顺序。

### 第三步：看控制面

重点找：

- firmware ready
- command send / response wait
- event handling

目标：知道“连接/扫描命令”是怎么下到芯片里的。

### 第四步：看数据面

重点找：

- xmit 入口
- sdio_writesb/readsb
- RX worker
- flow control
- skb 生命周期

目标：知道包是怎么从 Linux 网络栈进出芯片的。

### 第五步：看 cfg80211 注册

重点找：

- `wiphy` 注册
- scan/connect/disconnect 回调
- station/AP 模式能力

目标：把“总线代码”和“无线功能”连起来。

------

## 14. 调试 SDIO WiFi 的常见问题定位思路

### 14.1 probe 失败

优先查：

- `sdio_enable_func()` 是否成功
- block size 是否设置成功
- 中断是否申请成功
- firmware/NVRAM 是否找到
- DTS 中 regulator / pwrseq / reset 是否正确

### 14.2 发包失败

优先查：

- TX port/queue 是否耗尽
- firmware credit 是否回收
- CMD53 传输错误
- block size/align/padding 是否正确
- suspend 状态是否误判

### 14.3 收包异常

优先查：

- IRQ 是否真的来
- 中断状态寄存器是否被正确清除
- RX 聚合包解析是否正确
- event/data 包区分是否出错
- resume 后 SDIO timing 是否失效

### 14.4 suspend/resume 后死机或断网

优先查：

- host sleep 协议
- wake IRQ/GPIO
- keep-power-in-suspend
- host controller retune / bus width / clock restore
- firmware resume 超时

------

## 15. 一个“源码认知模型”

为了避免看源码时迷路，你可以把 SDIO WiFi 驱动抽象成 5 个模块：

```text
[1] Bus binding
    probe/remove/suspend/resume

[2] Bus I/O
    readb/writeb/readsb/writesb/claim_host

[3] Firmware transport
    cmd/event/data protocol

[4] Driver core
    queues, state machine, workqueue, flow control

[5] Wireless integration
    cfg80211/mac80211, netdev, wiphy
```

以后无论你看到 brcmfmac、mwifiex 还是厂商私有 out-of-tree 驱动，都可以先问自己：

- 这个函数属于哪一层？
- 它是在做 bus I/O，还是在做 firmware protocol？
- 它返回的是 net stack 事件，还是 dongle 事件？

这样会清晰很多。

------

## 16. 基于 5.10 的源码结论总结

### 16.1 结论一：SDIO WiFi 的核心不在“网卡”，而在“总线 + 固件协议”

和 PCIe NIC 相比，SDIO WiFi 驱动更多是在解决：

- 怎样稳定访问总线
- 怎样跟固件通信
- 怎样把异步事件映射回 Linux 无线栈

### 16.2 结论二：主线驱动大多直接对接 cfg80211，是否用 mac80211 取决于芯片形态

- FullMAC：驱动更像 transport + cfg80211 bridge
- SoftMAC：驱动更像硬件抽象层 + mac80211 ops

### 16.3 结论三：源码阅读要抓住 4 条主线

1. `probe/init`
2. `TX`
3. `RX/IRQ`
4. `suspend/resume`

只要把这 4 条线串起来，整个 SDIO WiFi 驱动就基本读通了。

------

## 17. 推荐你下一步继续深挖的源码点

如果你要继续做更深入的源码分析，建议下一步直接指定一种驱动，我可以继续往下拆：

1. **brcmfmac SDIO**：适合看 FullMAC + firmware transport
2. **mwifiex SDIO**：适合看较清晰的 SDIO probe / I/O / FW ready 逻辑
3. **某个具体芯片厂商驱动**（例如 RTL/MTK/AIC 等）

最有价值的继续拆法是：

- 逐函数分析 `probe()`
- 逐函数分析 TX/RX 主路径
- 结合 DTS 与启动日志解释初始化时序
- 画出从 `iw dev wlan0 scan` 到 SDIO 命令下发的调用链

------

## 18. 参考依据

- Linux 5.10 内核文档中的 MMC/SD/SDIO 驱动文档
- Linux 5.10 无线栈中的 cfg80211 文档
- 主线源码中的 `drivers/mmc/core/sdio_bus.c`
- 主线源码中的 `drivers/net/wireless/marvell/mwifiex/sdio.c`
- 主线源码中的 `drivers/net/wireless/broadcom/brcm80211/brcmfmac/sdio.c`



**基于 Linux 5.10，SDIO WiFi 的主线框架本质上是：`MMC/SDIO 总线层 + WiFi 芯片驱动层 + cfg80211/mac80211 无线栈接口层`。**
其中，SDIO function 设备由 MMC/SDIO 子系统创建并挂到 `sdio` 总线上，驱动通过 `sdio_driver` 的 `probe/remove/pm` 完成绑定；而现代 Linux 无线驱动需要接入 `cfg80211`，`cfg80211` 负责把驱动能力通过 `nl80211` 暴露给用户态。([Linux内核归档](https://www.kernel.org/doc/html/v5.10//driver-api/mmc/index.html))

我已经把一份系统化分析整理到旁边文档里，里面按 **框架分层、probe 初始化、TX/RX/IRQ、firmware、PM、电源与 DTS、源码阅读路径** 全部拆开了。这里先把最核心的源码理解给你压缩成几个重点：

**1）总线入口在 MMC/SDIO，不在无线栈。**
Linux 5.10 的 MMC/SDIO 文档明确把 MMC/SD/SDIO 支撑放在驱动实现文档中；而 `drivers/mmc/core/sdio_bus.c` 里可以看到 `sdio_bus_type` 提供了 `.match/.probe/.remove/.shutdown/.pm`，以及 `__sdio_register_driver()` 用来把具体 SDIO 驱动注册到总线，`sdio_add_func()` 用来把枚举出的 function 注册成设备。也就是说，任何 SDIO WiFi 驱动，第一现场都是 `sdio_bus.c`。([Linux内核归档](https://www.kernel.org/doc/html/v5.10//driver-api/mmc/index.html))

**2）无线统一接口是 cfg80211，是否用 mac80211 要看芯片是 FullMAC 还是 SoftMAC。**
Linux 5.10 的 `cfg80211` 文档说明，现代无线驱动都应直接或间接通过 `cfg80211` 接入系统，并以 `wiphy` 作为物理无线设备抽象，关联 `wireless_dev` 和网络接口。很多 SDIO WiFi，尤其是 Broadcom/Cypress 一类，是 **FullMAC**：扫描、关联、漫游、功耗等逻辑主要在固件里，内核驱动更像“host 到 firmware 的 transport 层”；这类驱动一般不重度依赖 `mac80211`。([Infradead](https://www.infradead.org/~mchehab/kernel_docs/driver-api/80211/cfg80211.html))

**3）典型 probe 调用链是：枚举 function → 使能 function → 配 block size/IRQ → 固件初始化 → 注册 wiphy/netdev。**
这在 `mwifiex` 的 SDIO 实现里非常清楚。搜索结果直接展示了 `mwifiex_sdio_probe()`：它先分配 `sdio_mmc_card`，保存 `func`，设置卡级能力与固件信息，然后 `sdio_enable_func(func)`，再继续设备初始化与逻辑接口注册。这里能明显看出主线 SDIO WiFi 驱动常见的两层私有对象：一层偏总线/卡资源，一层偏 adapter/无线协议状态。([GitHub](https://github.com/torvalds/linux/blob/master/drivers/net/wireless/marvell/mwifiex/sdio.c?utm_source=chatgpt.com))

**4）SDIO WiFi 的数据面本质是“往 function 的 port 做块读写”，不是传统 PCIe 网卡那种 DMA ring。**
`mwifiex` 的源码片段里直接能看到：寄存器访问用 `sdio_readb()/sdio_writeb()`，数据读写用 `sdio_readsb()/sdio_writesb()`，外围再由 `sdio_claim_host()/sdio_release_host()` 保护总线访问。`mwifiex_write_data_sync()` 和 `mwifiex_read_data_sync()` 还会根据端口和长度选择 byte mode 或 block mode，并计算块大小、块数和 I/O port。这个模式就是 SDIO WiFi 的关键实现特征。([GitHub](https://github.com/torvalds/linux/blob/master/drivers/net/wireless/marvell/mwifiex/sdio.c?utm_source=chatgpt.com))

**5）固件状态机是 SDIO WiFi 源码的核心复杂度来源之一。**
`mwifiex` 的 `mwifiex_sdio_read_fw_status()` / `mwifiex_check_fw_status()` 说明驱动会通过状态寄存器轮询 firmware ready，并在必要时增加额外 delay；若固件没真正 ready，probe 往往直接失败。也就是说，很多“WiFi 驱动问题”实际上不是网络栈问题，而是 **firmware、上电时序、状态寄存器同步** 问题。([GitHub](https://github.com/torvalds/linux/blob/master/drivers/net/wireless/marvell/mwifiex/sdio.c?utm_source=chatgpt.com))

**6）如果你想看 FullMAC 的代表实现，brcmfmac 是最典型样本。**
`brcmfmac` 的 `sdio.c` 在包含关系上就已经把层次暴露得很清楚：它既包含 `linux/mmc/sdio*.h`，也包含 `firmware.h`、`core.h`、`common.h`、`bcdc.h`。这说明它的核心不是单纯“发网包”，而是：**SDIO 传输层 + 固件下载/控制通道 + Broadcom 自己的 host/firmware 协议（如 BCDC）**。这正是 FullMAC SDIO 驱动在 5.10 主线中的典型写法。([GitHub](https://github.com/torvalds/linux/blob/master/drivers/net/wireless/broadcom/brcm80211/brcmfmac/sdio.c))

你如果要继续往下深挖，最建议的阅读顺序是：

先看 `drivers/mmc/core/sdio_bus.c`，把 `sdio_driver` 怎么注册、`sdio_func` 怎么创建设备、怎么触发 `probe()` 搞清楚。([GitHub](https://github.com/torvalds/linux/blob/master/drivers/mmc/core/sdio_bus.c?utm_source=chatgpt.com))

再看一个具体驱动的 `probe/remove`。入门上，`mwifiex/sdio.c` 比较直观；要看 FullMAC 思路，就看 `brcmfmac/sdio.c`。([GitHub](https://github.com/torvalds/linux/blob/master/drivers/net/wireless/marvell/mwifiex/sdio.c?utm_source=chatgpt.com))

之后再沿四条线读：`probe/init`、`TX`、`RX/IRQ`、`suspend/resume`。这四条线串起来，整个 SDIO WiFi 驱动基本就读通了。相关 PM 钩子在 `sdio_bus.c` 总线侧也能看到，而 `mwifiex` 搜索结果里也直接展示了 `resume/suspend` 相关实现。([GitHub](https://github.com/torvalds/linux/blob/master/drivers/mmc/core/sdio_bus.c?utm_source=chatgpt.com))

你要是愿意，我下一条可以直接继续做两种之一：
**A：按 5.10 源码把 `mwifiex_sdio_probe()` 逐函数带你读一遍**，或者
**B：把 `brcmfmac SDIO` 的初始化、收发、IRQ、firmware 下载调用链画成一张“源码调用图”给你。**