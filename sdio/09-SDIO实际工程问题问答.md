# SDIO 实际工程问题问答

## 学习目标

- 把 SDIO 实际故障现象映射到 DTS/host、card/core、bus/probe、I/O、IRQ、function driver 各层
- 建立从日志和用户现象反推内核调用链的排查方法
- 形成一套适合 HI3516CV610 + SDIO 模组调试的检查清单

## 导读

### 本章定位

这一章是 SDIO 笔记的工程问答章。前面章节已经讲清对象、枚举、probe、I/O、中断和板级落地，本章把这些内容重新组织成实际问题排查入口。

### 核心对象

- `mmc_host`
  - host 控制器、时钟、电压、总线宽度、中断能力
- `mmc_card`
  - 整卡识别、CCCR/CIS、function 数量
- `sdio_func`
  - function 设备、block size、I/O、IRQ handler
- `sdio_driver`
  - function driver 的 id 匹配、probe、remove
- `sdhci_host / sdhci_nebula`
  - HI3516CV610 板级 host 适配

### 关键函数

- `mmc_add_host()`
- `mmc_rescan()`
- `mmc_attach_sdio()`
- `mmc_sdio_init_card()`
- `sdio_init_func()`
- `sdio_add_func()`
- `sdio_bus_match()`
- `sdio_bus_probe()`
- `sdio_enable_func()`
- `sdio_set_block_size()`
- `sdio_claim_irq()`
- `sdio_memcpy_fromio() / sdio_memcpy_toio()`

### 主流程

现象 -> 判断层次 -> 找调用链断点 -> 检查对象字段和返回值 -> 再进入模组私有协议。

## 这一章按什么逻辑展开

这一章按“先给总排查链路，再按真实现象分问答”的逻辑展开。

这样拆的原因是：

- 工程问答章不能只列一堆问题和答案
- 先给总排查链路，后面的每个现象才知道自己落在哪一层、该沿哪条调用链回退

所以本章后面的结构是：

1. 先给 `DTS -> host -> card -> func -> probe -> I/O -> IRQ` 总排查链路
2. 再按具体现象拆分问答
3. 每个问答都回到层级、对象、调用链和检查点

## 1. 工程排查总链路

SDIO 问题按下面顺序分层：

```text
DTS / pinctrl / clock / power
-> host controller probe
-> mmc_add_host()
-> mmc_rescan()
-> mmc_attach_sdio()
-> mmc_sdio_init_card()
-> sdio_init_func()
-> sdio_add_func()
-> sdio_bus_match()
-> sdio_bus_probe()
-> sdio_driver.probe()
-> sdio_enable_func()
-> sdio_set_block_size()
-> sdio_claim_irq()
-> CMD52 / CMD53 I/O
-> function driver private protocol
```

这条总链更适合按五段读：

1. `DTS / pinctrl / clock / power -> host controller probe`
   - 这是板级和 host 准备阶段
   - 这里出问题时，后面不会有任何 card 或 function 痕迹
2. `mmc_add_host() -> mmc_rescan() -> mmc_attach_sdio() -> mmc_sdio_init_card()`
   - 这是 card/core 枚举阶段
   - 这里决定整张卡能不能被按 SDIO 路线识别出来
3. `sdio_init_func() -> sdio_add_func()`
   - 这是 function 对象创建和注册阶段
   - 这里决定 `/sys/bus/sdio/devices` 里是否会出现 function 设备
4. `sdio_bus_match() -> sdio_bus_probe() -> sdio_driver.probe()`
   - 这是总线匹配和 function driver 进入阶段
   - 这里决定具体芯片驱动能不能真正跑起来
5. `sdio_enable_func() -> sdio_set_block_size() -> sdio_claim_irq() -> CMD52 / CMD53 I/O -> function driver private protocol`
   - 这是 function 真正工作阶段
   - 这里出问题时，通常已经不是“设备没出来”，而是“设备出来了但不能正常工作”

这条图解决的是“工程排查应该沿哪条主线逐层后退”的问题。

排查原则：

- `/sys/bus/sdio/devices` 没有设备时，优先看 host/card/core 层
- 有 `sdio_func` 但驱动 probe 不进时，优先看 bus match/id_table
- probe 进了但芯片不工作时，优先看 enable、block size、I/O、IRQ
- WiFi 接口不出现时，再继续看固件、cfg80211/netdev 等上层

## 2. 系统没有识别出任何 SDIO 设备

### 现象

- `/sys/bus/sdio/devices` 为空
- 日志里没有 `mmc_attach_sdio()` 相关输出
- WiFi/BT 模组完全没有枚举迹象

### 定位层

优先定位 DTS/host/card detect/供电，而不是 function driver。

### 常见检查点

- `sdio0` DTS 节点是否为 `okay`：节点未启用时 platform driver 不会创建 host，后续 `mmc_add_host()` 不会发生；相关链路：`DTS status -> platform probe -> mmc_add_host`。
>[!TIP]
>设备树节点在系统上是/sys/firmware/devicetree，
>总线设备驱动是在/sys/bus/
- `mmc0` 和 `sdio0` 是否配置冲突：HI3516CV610 上 `mmc0` 与 `sdio0` 共用 `0x10030000` 控制器，选错用途会导致实际 SDIO 控制器没有按预期工作；相关链路：`hi3516cv610-demb.dts -> mmc0/sdio0 status -> nebula host`。
- pinctrl / iomux 是否配置到 SDIO 功能：引脚仍处于 GPIO 或其他复用功能时，CMD/CLK/DATA 物理信号不通；相关链路：`DTS pinctrl -> host pins -> card response`。
- 模组电源、复位、WL_REG_ON 是否正确：模组未上电或复位未释放时，host 发 CMD5 不会得到响应；相关链路：`power/reset -> CMD5 -> mmc_attach_sdio`。
- `sdhci_nebula` 是否 probe 成功：host 没起来时 MMC core 没有控制器可 rescan；相关链路：`platform probe -> sdhci_nebula_add_host -> mmc_add_host`。

### 常见根因

- 板级 DTS 启用了错误控制器
- SDIO 引脚复用或上拉配置错误
- 模组供电/复位时序不满足
- host 时钟未打开
- 板级焊接或线序问题

### 核心判断

没有 `sdio_func` 设备时，function driver 还没有参与，问题通常在 DTS、host、供电或卡枚举之前。

## 3. host probe 成功，但 `mmc_attach_sdio()` 没走到

### 现象

- `sdhci_nebula` probe 有日志
- `mmc_host` 看起来已经注册
- 没有 SDIO 卡初始化日志

### 定位层

重点看 MMC rescan 和卡识别路径。

### 常见检查点

- `mmc_add_host()` 是否执行：只有 host 加入 MMC core 后，rescan 才有机会检测卡；相关链路：`sdhci_nebula_add_host -> sdhci_add_host -> mmc_add_host`。
- host 是否被标记为 non-removable 或具备 card detect：没有插拔检测或 non-removable 配置时，core 可能认为没有卡；相关链路：`host caps/DTS -> mmc_rescan -> card detect`。
- `no-sd / no-mmc / no-sdio` 类属性是否配置合理：错误禁用 SDIO 会让 core 不走 SDIO 识别路径；相关链路：`DTS capability flags -> mmc_rescan_try_freq -> attach path`。
- 时钟频率是否过高：初始阶段频率过高可能导致 CMD5 无响应；相关链路：`mmc_rescan -> init clock -> CMD5`。
- 供电电压是否在 OCR 支持范围内：电压选择失败会阻断 attach；相关链路：`CMD5 OCR -> mmc_select_voltage -> mmc_attach_sdio`。

### 常见根因

- 缺少 non-removable 或 card detect 配置不符合硬件
- 电源未稳定时就开始 rescan
- 初始时钟或电压不匹配
- DTS capability 属性误配

### 核心判断

host probe 成功只说明控制器驱动起来了，不代表 SDIO 卡已经被 core 识别。

## 4. `mmc_attach_sdio()` 失败

### 现象

- 日志能看到进入 SDIO attach
- 但没有生成 `mmc_card` 或 `sdio_func`
- CMD5、CCCR、CIS 相关步骤报错

### 定位层

重点看整卡初始化和 SDIO 公共信息读取。

### 常见检查点

- `mmc_send_io_op_cond()` 是否成功：CMD5 是 SDIO 识别入口，失败说明卡没有响应 SDIO OCR；相关链路：`mmc_attach_sdio -> CMD5 -> OCR`。
- `mmc_select_voltage()` 是否成功：电压选择失败会导致后续无法可靠通信；相关链路：`CMD5 OCR -> mmc_select_voltage -> card init`。
- `mmc_sdio_init_card()` 是否创建 `mmc_card`：整卡对象创建失败会阻断所有 function 初始化；相关链路：`mmc_attach_sdio -> mmc_sdio_init_card -> mmc_card`。
- `sdio_read_cccr()` 是否成功：CCCR 决定卡级能力，读失败通常说明基础 CMD52 通路不可靠；相关链路：`mmc_sdio_init_card -> sdio_read_cccr -> CMD52`。
- `sdio_read_common_cis()` 是否成功：CIS 读取失败会影响厂商、功能等描述信息；相关链路：`mmc_sdio_init_card -> sdio_read_common_cis`。

### 常见根因

- CMD/DATA 线硬件问题
- 模组未出复位
- 电压不匹配
- host 时序或频率不兼容
- SDIO 卡本身需要额外上电 GPIO 或 firmware strap

### 核心判断

`mmc_attach_sdio()` 失败时，function driver 还没出现，重点查 CMD5、CMD52、CCCR、CIS 和基础硬件通信。

## 5. 能识别卡，但没有 function 设备

### 现象

- 能看到 `mmc_card`
- 但 `/sys/bus/sdio/devices` 没有具体 function
- function driver probe 不可能进入

### 定位层

重点看 function 数量解析和 `sdio_init_func()`。

### 常见检查点

- OCR 中 function 数量是否正确：core 根据 OCR 判断有几个 function，数量为 0 时不会创建 `sdio_func`；相关链路：`CMD5 OCR -> funcs -> sdio_init_func`。
- `sdio_init_func()` 是否成功：该函数分配并填充 `struct sdio_func`，失败会导致对应 function 不存在；相关链路：`mmc_attach_sdio -> sdio_init_func(fn)`。
- FBR 是否读取成功：FBR 提供 function class 等信息，读失败会阻断 function 初始化；相关链路：`sdio_init_func -> read FBR -> func->class`。
- function CIS 是否读取成功：function 私有 CIS 填充 vendor/device 等信息，读失败会影响后续匹配；相关链路：`sdio_init_func -> sdio_read_func_cis -> func ids`。
- `sdio_add_func()` 是否执行：只有 add 后 function 才进入 driver model；相关链路：`sdio_init_func -> mmc_add_card -> sdio_add_func`。

### 常见根因

- OCR function 数量异常
- FBR/CIS 读失败
- 内存分配失败
- function 初始化失败后被清理

### 核心判断

`sdio_init_func()` 解决“对象创建”，`sdio_add_func()` 解决“设备注册”。没有 function 设备时，probe 层还没有开始。

## 6. function 设备存在，但驱动 probe 不进

### 现象

- `/sys/bus/sdio/devices` 能看到 function
- 具体驱动的 `probe()` 没日志
- 手动加载驱动后仍不绑定

### 定位层

重点看 sdio bus match 和 `id_table`。

### 常见检查点

- `sdio_driver` 是否注册成功：驱动没有注册到 sdio bus 时，设备没有可匹配的 driver；相关链路：`module_sdio_driver -> sdio_register_driver -> sdio_bus_type`。
- `id_table` 是否包含当前 vendor/device/class：match 只按表项比较，ID 不匹配就不会 probe；相关链路：`sdio_bus_match -> sdio_match_device -> id_table`。
- `MODULE_DEVICE_TABLE(sdio, ids)` 是否存在：模块自动加载依赖该表导出 modalias，缺失会导致设备存在但驱动不自动加载；相关链路：`sdio uevent -> modalias -> module autoload`。
- `func->vendor/device/class` 是否符合预期：CIS 解析出的 ID 不对时，驱动表项看起来正确也匹配不上；相关链路：`sdio_init_func -> func ids -> sdio_bus_match`。
- `sdio_bus_probe()` 是否被调用：match 成功后总线层会先进通用 probe，再调驱动 probe；相关链路：`match ok -> sdio_bus_probe -> drv->probe`。

### 常见根因

- 驱动 ID 表漏项
- 模块没有加载
- `MODULE_DEVICE_TABLE` 缺失
- 芯片实际 ID 和预期不一致
- function 编号不是驱动预期的编号

### 核心判断

function 设备存在但 probe 不进，优先查 `sdio_device_id` 匹配，而不是查 I/O 读写。

## 7. probe 进入后 `sdio_enable_func()` 失败

### 现象

- 驱动 `probe()` 已经进入
- `sdio_enable_func()` 返回错误
- 后续寄存器访问或固件下载无法继续

### 定位层

重点看 function 使能和 CCCR I/O ready。

### 常见检查点

- 调用前是否 `sdio_claim_host()`：enable 操作需要独占 host，缺少 claim 会造成并发和协议状态错误；相关链路：`probe -> sdio_claim_host -> sdio_enable_func`。
- `CCCR_IOEx` 写入是否成功：enable 本质是打开 function enable bit，写失败说明 CMD52 写路径异常；相关链路：`sdio_enable_func -> write CCCR_IOEx`。
- `CCCR_IORx` 是否 ready：enable 后 core 会轮询 ready bit，ready 不到说明 function 没真正启动；相关链路：`write IOEx -> poll IORx -> function ready`。
- 模组电源/固件前置条件是否满足：部分芯片需要额外 GPIO、firmware strap 或时钟条件，ready 才会置位；相关链路：`power/reset -> enable_func -> IORx ready`。

### 常见根因

- 未 claim host
- 模组 function 未 ready
- CMD52 写读异常
- 电源/复位/时钟时序不满足

### 核心判断

`sdio_enable_func()` 是 function driver 真正接管芯片前的第一道门，失败时不应继续做私有寄存器访问。

## 8. CMD52 小寄存器读写失败

### 现象

- `sdio_readb()` / `sdio_writeb()` 返回错误
- 读寄存器全 0 或全 0xff
- probe 中读取芯片 ID 失败

### 定位层

重点看 host claim、function enable、地址和基础 CMD52 通信。

### 常见检查点

- 是否已经 `sdio_enable_func()`：function 未启用时访问 function 寄存器通常无效；相关链路：`probe -> enable_func -> sdio_readb/writeb`。
- 是否在 claim host 后访问：普通上下文必须先 claim host，否则可能和其他 SDIO 事务冲突；相关链路：`sdio_claim_host -> sdio_readb/writeb -> sdio_release_host`。
- 地址是否属于目标 function 的寄存器空间：地址错会读到无意义值或触发错误；相关链路：`driver register map -> CMD52 address`。
- 是否误把 func0 和 func1 寄存器混在一起：CCCR/FBR 属于 func0 访问语义，芯片私有寄存器通常在具体 function 空间；相关链路：`func0 common regs / func1 private regs`。

### 常见根因

- function 未 enable
- 地址偏移错误
- claim host 缺失
- 模组还在复位或休眠
- 读写函数使用的 `func` 对象不是目标 function

### 核心判断

CMD52 失败通常说明基础控制通路有问题，先不要进入大块数据或固件协议分析。

## 9. CMD53 批量读写超时或数据错乱

### 现象

- `sdio_memcpy_toio()` / `sdio_memcpy_fromio()` 失败
- 大包传输超时
- 小寄存器读写正常，但数据路径不稳定

### 定位层

重点看 block size、长度对齐、地址窗口和 host 时序。

### 常见检查点

- `sdio_set_block_size()` 是否成功：CMD53 块传输依赖当前 block size，设置不合理会导致大包和尾包异常；相关链路：`probe -> sdio_set_block_size -> CMD53 transfer`。
- 传输长度是否经过 `sdio_align_size()` 或芯片要求的对齐：未对齐可能让 host/card 在 block mode 和 byte mode 间切换异常；相关链路：`tx/rx len -> sdio_align_size -> sdio_memcpy_*io`。
- FIFO/window 地址是否正确：CMD53 访问地址错会造成数据错位或读写到错误区域；相关链路：`driver port/window -> sdio_memcpy_toio/fromio`。
- host `max_blk_count/max_req_size` 是否限制传输：超过 host 能力时 core 可能拆分或失败；相关链路：`sdio_io_rw_ext_helper -> mmc_io_rw_extended -> host limits`。
- 时钟频率是否过高：高频下边沿、走线、上拉不稳定会表现为大包偶发错误；相关链路：`max-frequency -> mmc_set_clock -> CMD53`。

### 常见根因

- block size 与芯片协议不匹配
- 长度未按芯片要求对齐
- SDIO 时钟过高
- FIFO 地址或 packet port 选择错误
- host/controller DMA 限制未处理

### 核心判断

CMD52 正常但 CMD53 异常时，重点从 block size、长度对齐、port 地址、host 传输限制四个方向排查。

## 10. SDIO 中断不来

### 现象

- `sdio_claim_irq()` 成功
- function driver 的 irq handler 不触发
- 数据接收依赖轮询才有反应

### 定位层

重点看 function IENx、host SDIO IRQ 能力和 core 分发线程。

### 常见检查点

- `sdio_claim_irq()` 是否成功：它会记录 `func->irq_handler` 并打开 function interrupt enable；相关链路：`probe -> sdio_claim_irq -> func->irq_handler`。
- `CCCR_IENx` master bit 和 function bit 是否打开：没有打开中断使能时，卡不会把 pending 交给 host；相关链路：`sdio_claim_irq -> write IENx -> card interrupt`。
- host 是否有 `MMC_CAP_SDIO_IRQ`：host 不声明能力时 core 无法按 SDIO IRQ 路线工作；相关链路：`host caps -> sdio_card_irq_get -> irq thread`。
- `host->ops->enable_sdio_irq` 是否实现：host 需要能打开底层 SDIO IRQ 检测；相关链路：`sdio_irq_thread -> enable_sdio_irq(host, 1)`。
- `mmc_signal_sdio_irq()` 是否被 host 调用：host 感知到卡中断后要通知 core，否则 handler 不会被分发；相关链路：`host IRQ -> mmc_signal_sdio_irq -> sdio_irq_thread`。

### 常见根因

- 驱动没有成功 claim IRQ
- IENx 未打开
- host 不支持或未使能 SDIO IRQ
- 使用 out-of-band IRQ 但 DTS 中断配置错误
- handler 里错误地再次 claim host 导致卡死

### 核心判断

SDIO IRQ 是 `function driver -> SDIO core -> host` 共同完成的机制，不能只看 function driver 的 handler。

## 11. IRQ handler 里卡死

### 现象

- 中断触发后系统卡住
- handler 中读写 SDIO 后无返回
- 日志停在 IRQ 回调附近

### 定位层

重点看 IRQ handler 上下文里的 host claim 规则。

### 常见检查点

- handler 中是否重复 `sdio_claim_host()`：`sdio_claim_irq()` 注册的 handler 被调用时，host 通常已经由 core claim，重复 claim 可能死锁；相关链路：`sdio_irq_thread -> claim host -> func->irq_handler`。
- handler 是否执行耗时私有协议：长时间处理会阻塞 SDIO IRQ 分发线程，影响其他 function 或后续中断；相关链路：`func->irq_handler -> long operation -> irq thread blocked`。
- handler 是否只做唤醒 worker：复杂协议适合放到工作线程里处理，IRQ handler 保持短路径；相关链路：`irq_handler -> schedule_work/thread -> SDIO I/O`。

### 常见根因

- IRQ handler 里重复 claim host
- handler 里等待另一个需要 host 的线程
- 在 SDIO IRQ 分发线程里执行过重任务

### 核心判断

标准 SDIO IRQ handler 不是裸硬中断，但它运行时 host 通常已被 core 持有，因此 claim 规则和普通上下文不同。

## 12. 卸载驱动或 remove 时报警

### 现象

- rmmod 或设备移除时出现 IRQ 相关告警
- 下次加载驱动异常
- function 资源没有清干净

### 定位层

重点看 `sdio_bus_remove()` 对 function driver 的清理检查。

### 常见检查点

- `remove()` 是否调用 `sdio_release_irq()`：core 会检查 `func->irq_handler` 是否还存在，未释放会报警；相关链路：`sdio_bus_remove -> drv->remove -> func->irq_handler check`。
- 是否在 claim host 后调用 `sdio_disable_func()`：disable function 需要走 SDIO 控制通路，缺少 claim 会破坏总线事务规则；相关链路：`remove -> sdio_claim_host -> sdio_disable_func`。
- `sdio_set_drvdata(func, NULL)` 或私有对象释放是否完整：私有指针残留会影响后续 probe 或错误路径；相关链路：`probe set_drvdata -> remove cleanup`。
- 上层 netdev/wiphy/worker 是否先注销或停止：上层还在发起 SDIO I/O 时释放 function 会造成 use-after-free；相关链路：`stop upper layers -> release irq -> disable func -> free private`。

### 常见根因

- 漏掉 `sdio_release_irq()`
- disable function 顺序错误
- worker/thread 未停止
- 私有数据生命周期不匹配

### 核心判断

remove 路径要按 probe 的反向顺序收尾，尤其要先停上层业务，再释放 IRQ，最后 disable function。

## 13. WiFi SDIO probe 成功，但没有网络接口

### 现象

- SDIO function driver probe 已进入
- `sdio_enable_func()` 成功
- 没有 `wlan0` 或无线设备

### 定位层

SDIO 层已经基本打通，问题进入 WiFi 驱动私有初始化、固件、cfg80211/netdev 注册层。

### 常见检查点

- 固件下载是否成功：FullMAC SDIO WiFi 通常依赖 firmware，固件失败会阻断 wiphy/netdev 注册；相关链路：`sdio probe -> firmware download -> cfg80211/netdev register`。
- 芯片命令通道是否建立：驱动需要通过 SDIO I/O 和芯片固件通信，命令通道失败时上层无线能力拿不到；相关链路：`sdio CMD53 -> firmware command -> chip response`。
- `wiphy` 是否注册成功：现代 WiFi 驱动通常先注册无线物理设备，失败时用户态看不到无线能力；相关链路：`driver init -> wiphy_register -> nl80211`。
- `net_device` 是否注册成功：没有 netdev 时不会出现 `wlan0` 一类网络接口；相关链路：`driver init -> register_netdev -> ip link`。
- regulatory 或 mac 地址是否有效：部分驱动在能力或地址不合法时会拒绝继续注册；相关链路：`firmware/chip info -> cfg80211/netdev setup`。

### 常见根因

- 固件文件缺失或版本不匹配
- SDIO 数据通道可用但芯片命令协议失败
- cfg80211 注册失败
- MAC 地址读取失败
- 驱动私有初始化错误

### 核心判断

SDIO probe 成功只说明总线层绑定完成。网络接口是否出现，还取决于 WiFi 驱动私有协议、固件和无线栈注册。

## 14. 工程排查固定清单

1. DTS 节点
   - 原因：节点启用、pinctrl、电源、复位决定 host 和模组是否具备基本工作条件。
   - 相关链路：`DTS -> platform probe -> host/card detect`
2. `sdhci_nebula` probe
   - 原因：host 驱动没起来时，MMC core 没有控制器可用于 SDIO 枚举。
   - 相关链路：`platform probe -> sdhci_nebula_add_host -> mmc_add_host`
3. `mmc_attach_sdio()`
   - 原因：这是 SDIO 卡识别入口，未进入说明卡还没被按 SDIO 路线处理。
   - 相关链路：`mmc_rescan -> CMD5 -> mmc_attach_sdio`
4. `mmc_sdio_init_card()`
   - 原因：整卡初始化失败时，`mmc_card` 和 CCCR/CIS 信息不完整。
   - 相关链路：`mmc_attach_sdio -> mmc_sdio_init_card`
5. `sdio_init_func()`
   - 原因：它创建 `sdio_func`，失败会导致 function 设备不存在。
   - 相关链路：`function count -> sdio_init_func -> sdio_func`
6. `sdio_add_func()`
   - 原因：它把 function 注册进 driver model，之后才可能触发 bus match。
   - 相关链路：`sdio_func -> device_add -> sdio bus`
7. `sdio_bus_match()`
   - 原因：probe 不进时，最常见断点就是 ID 表未命中。
   - 相关链路：`sdio_func ids -> sdio_driver.id_table -> match`
8. `sdio_bus_probe()`
   - 原因：总线层会做默认 block size、PM 等公共准备，然后才调用驱动 probe。
   - 相关链路：`match ok -> sdio_bus_probe -> drv->probe`
9. `sdio_enable_func()`
   - 原因：function 未 enable 时，后续私有寄存器和数据通道通常不可用。
   - 相关链路：`drv->probe -> enable_func -> IORx ready`
10. `sdio_set_block_size()`
   - 原因：block size 直接影响 CMD53 数据通路稳定性和性能。
   - 相关链路：`probe -> block size -> sdio_memcpy_*io`
11. `sdio_claim_irq()`
   - 原因：SDIO IRQ handler 只有注册并使能 IENx 后才可能被 core 调用。
   - 相关链路：`claim_irq -> IENx -> sdio_irq_thread -> handler`
12. CMD52/CMD53 I/O
   - 原因：小寄存器和批量数据分别验证控制通路和数据通路。
   - 相关链路：`sdio_readb/writeb -> sdio_memcpy_*io -> chip protocol`
13. 上层无线或业务注册
   - 原因：SDIO 层成功不等于 WiFi/netdev 已经完成，需要继续看固件和上层子系统。
   - 相关链路：`SDIO probe -> firmware -> wiphy/netdev`

## 15. 一句话总结

SDIO 工程问题的关键是先判断断点在 host/card/core/bus/function driver 哪一层，再进入芯片私有协议；不能在 function probe 还没进入时就开始分析 WiFi 固件，也不能在 CMD52 都失败时直接追上层网络接口。

## 16. 回答的问题

- 没有 SDIO 设备时应该先看 DTS、host 还是 function driver
- `sdio_func` 是什么时候创建并注册到 sdio bus 的
- function 设备存在但 probe 不进时该怎样查 `id_table`
- `sdio_enable_func()`、CMD52、CMD53 分别代表哪一层通路
- SDIO IRQ 不来时应如何从 `claim_irq` 追到 host
- WiFi SDIO probe 成功但没有网络接口时为什么要继续看固件和 cfg80211/netdev

