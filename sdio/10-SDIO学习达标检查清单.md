# SDIO 学习达标检查清单

## 学习目标

- 明确学完 SDIO 项目后应该达到的理解效果
- 把 SDIO 知识拆成对象、主流程、源码入口、板级落地、I/O、IRQ 和工程排查几个可检查维度
- 提供一份可反复复盘的自检清单，用来判断当前理解是否已经从“看过”变成“能定位问题”

## 导读

### 本章定位

这一章是 SDIO 学习项目的收口章，不继续展开新的源码细节，而是把前面 `00-09` 的内容压成达标标准。

学习完成后的目标不是背下每个函数，而是形成三种能力：

- 看到 SDIO 设备枚举问题时，能判断问题落在哪一层
- 读一个 SDIO function driver 时，能把私有代码放回 Linux SDIO 主线里
- 调试 HI3516CV610 板级 SDIO 问题时，能从 DTS/host 一路追到 I/O/IRQ

### 核心对象

- `mmc_host`
  - 控制器和板级资源的入口对象
- `mmc_card`
  - 整张 SDIO 卡的身份和能力对象
- `sdio_func`
  - function 设备和具体 function driver 之间的桥
- `sdio_driver`
  - 具体芯片驱动绑定 SDIO function 的入口

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

学习达标的核心判断标准是：能把下面这条链路从对象、源码、日志、板级配置和故障现象五个角度讲清楚。

```text
DTS / pinctrl / clock / power
-> host driver probe
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

## 这一章按什么逻辑展开

这一章按“结果目标 -> 能力清单 -> 自检题 -> 不达标信号”的逻辑展开。

这样拆的原因是：

- 学习完成度不能只用“看完几章”判断
- SDIO 的真正难点在于把对象、调用链、板级资源和故障现象连起来
- 自检清单需要能直接暴露薄弱点，方便回到对应章节补读

所以本章后面的结构是：

1. 先定义学习完成后的效果
2. 再按知识层级列出必须掌握的内容
3. 再给源码和工程自检题
4. 最后列出需要回炉补读的典型信号

## 1. 学完这个项目后应该达到的效果

### 1.1 能建立 SDIO 的分层模型

学完后应能明确区分：

- host controller 驱动负责把 SoC 控制器接入 MMC core
- MMC/SDIO core 负责识别卡、读取 CCCR/CIS、创建 `mmc_card` 和 `sdio_func`
- SDIO bus 负责把 `sdio_func` 和 `sdio_driver` 匹配起来
- function driver 负责芯片私有初始化、寄存器访问、数据收发、中断处理和上层子系统接入

达标表现：

- 能解释为什么 SDIO WiFi 驱动的 `probe()` 不是插卡后第一个执行的函数
- 能说明 `mmc_card` 和 `sdio_func` 的区别
- 能说出一个问题是 host 层、card/core 层、bus/probe 层，还是 function driver 层

### 1.2 能画出完整枚举到 probe 的主线

学完后应能不看笔记画出这条主线：

```text
host probe
-> mmc_add_host()
-> mmc_rescan()
-> mmc_attach_sdio()
   -> mmc_sdio_init_card()
      -> mmc_send_relative_addr()
      -> mmc_select_card()
      -> sdio_read_cccr()
      -> sdio_read_common_cis()
   -> sdio_init_func()
   -> mmc_add_card()
   -> sdio_add_func()
-> sdio_bus_match()
-> sdio_bus_probe()
-> sdio_driver.probe()
```

达标表现：

- 能解释每一步创建或注册了什么对象
- 能说明 `sdio_init_func()` 和 `sdio_add_func()` 的区别
- 能说明 `sdio_bus_match()` 依赖 `sdio_device_id` 做什么
- 能判断 `probe()` 没进时应该从哪几个断点回查

### 1.3 能把 SDIO I/O 和 IRQ 放回对象模型里

学完后应能理解：

- CMD52 通常对应小寄存器访问
- CMD53 通常对应批量数据传输
- `sdio_claim_host()` 保护对同一个 host 的访问
- `sdio_enable_func()` 是 function 正式访问前的重要步骤
- `sdio_set_block_size()` 会影响批量传输行为
- `sdio_claim_irq()` 把 function IRQ handler 注册给 SDIO core

达标表现：

- 能解释为什么访问 function 寄存器前通常要先 `sdio_claim_host()`
- 能说明 block size 设置不合适可能造成什么现象
- 能讲清 SDIO 中断不是直接跳到 function driver，而是先经过 host/core 分发

### 1.4 能把 HI3516CV610 板级落地和 Linux 标准主线接起来

学完后应能说明：

- `hi3516cv610-demb.dts` 当前启用的是 `sdio0`
- `mmc0`、`sdio0`、`sdio1` 的启用状态会影响后续所有 SDIO 分析
- `nebula,sdhci` 的核心作用是把 HI3516CV610 控制器接到标准 `sdhci + mmc core`
- 上层 SDIO core 逻辑仍然主要沿 Linux 5.10 标准路径运行

达标表现：

- 能先从 DTS 判断当前板子到底启用了哪个 SDIO host
- 能找到 `sdhci_nebula.c` 里 host 注册到 MMC core 的关键位置
- 能说明板级问题和 function driver 私有问题的边界

### 1.5 能按现象反推排查路径

学完后应能按下面方式定位：

- `/sys/bus/sdio/devices` 没有设备：优先看 DTS/host/card/core
- 有 `sdio_func` 但驱动不 probe：优先看 `sdio_bus_match()`、`id_table`、模块加载
- probe 进了但芯片不可用：优先看 `sdio_enable_func()`、block size、CMD52/CMD53、IRQ
- WiFi 接口没有出现：继续看固件加载、cfg80211、netdev 注册和芯片私有协议

达标表现：

- 能把一个现象映射到排查层级
- 能列出下一步应该加日志或断点的位置
- 能区分“设备没枚举出来”和“设备枚举了但业务层没起来”
[[10-SDIO学习达标检查清单#8. 内容清单参考答案]]
## 2. 必须知道的内容清单

### 2.1 对象清单

- [ ] `mmc_host` 表示什么，它由谁创建，什么时候注册给 MMC core
- [ ] `mmc_card` 表示什么，它在 SDIO 卡初始化中保存哪些整卡级信息
- [ ] `sdio_func` 表示什么，它为什么是 function driver 的核心入口对象
- [ ] `sdio_driver` 如何通过 `id_table` 和 `sdio_func` 匹配
- [ ] `sdio_device_id` 里的 vendor/device/class 字段如何影响匹配
- [ ] 能区分 `sdio_func` 的真实身份字段和 `sdio_device_id` 的驱动侧匹配声明
- [ ] host、card、func、driver 四类对象之间是什么关系

### 2.2 枚举和 probe 清单

- [ ] `mmc_add_host()` 之后为什么会触发 rescan
- [ ] `mmc_rescan()` 如何进入不同卡类型的识别路径
- [ ] `mmc_attach_sdio()` 为什么是 SDIO 主线入口
- [ ] `mmc_sdio_init_card()` 为什么是整卡初始化核心函数
- [ ] `sdio_read_cccr()` 和 `sdio_read_common_cis()` 解决什么问题
- [ ] `sdio_init_func()` 什么时候为每个 function 创建对象
- [ ] `sdio_add_func()` 如何把 function 暴露给 driver model
- [ ] function device 注册链和 function driver 注册链如何在 `sdio_bus_match()` 汇合
- [ ] `module_sdio_driver()`、`sdio_register_driver()`、`MODULE_DEVICE_TABLE()`、`id_table`、`probe/remove` 分别负责什么
- [ ] `sdio_bus_match()` 如何决定哪个 `sdio_driver` 可以绑定
- [ ] `sdio_bus_probe()` 在调用具体 driver probe 前做了什么准备
- [ ] 具体 `sdio_driver.probe()` 进入后通常做哪些私有初始化

### 2.3 I/O 清单

- [ ] `sdio_claim_host()` 和 `sdio_release_host()` 的作用
- [ ] `sdio_enable_func()` 为什么通常是访问 function 前的必经步骤
- [ ] `sdio_set_block_size()` 对 CMD53 传输有什么影响
- [ ] `sdio_readb()` / `sdio_writeb()` 通常对应哪类访问
- [ ] `sdio_memcpy_fromio()` / `sdio_memcpy_toio()` 通常对应哪类访问
- [ ] `sdio_readb()` / `sdio_writeb()` 如何经过 CMD52、MMC request、`host->ops->request`
- [ ] `sdio_memcpy_fromio()` / `sdio_memcpy_toio()` 如何经过 CMD53、MMC request、`host->ops->request`
- [ ] 教学型字符设备读写例子为什么适合练习 SDIO API，但不能代表真实 WiFi/BT function driver 的完整结构
- [ ] 真实 WiFi/BT function driver 为什么常把 SDIO I/O 封装成 bus ops 或 transport 层
- [ ] CMD52 和 CMD53 的工程现象差异
- [ ] I/O 超时、全零、错位、吞吐异常分别可能优先怀疑哪里

### 2.4 IRQ 清单

- [ ] `sdio_claim_irq()` 注册的是哪一层的回调
- [ ] host 层 SDIO IRQ 如何通知 MMC/SDIO core
- [ ] `sdio_irq_thread()` 和 `process_sdio_pending_irqs()` 的分发职责
- [ ] 能区分 host 控制器 IRQ、标准 SDIO in-band IRQ、function child node 外部 IRQ
- [ ] 外部平台 IRQ handler 内访问 SDIO 时，为什么通常要自己 `sdio_claim_host()`
- [ ] function driver 的 IRQ handler 应该避免哪些阻塞或重入问题
- [ ] 中断不来时如何区分 host 没上报、core 没分发、function 没产生中断

### 2.5 HI3516CV610 板级清单

- [ ] 当前板级 DTS 中 `mmc0`、`sdio0`、`sdio1` 的 `status`
- [ ] `sdio0` 的 `compatible`、`reg`、`interrupts`、clock、reset、`crg_regmap`、`iocfg_regmap`、bus-width、max-frequency 等资源如何被内核消费
- [ ] `nebula,sdhci` host 驱动的 probe 入口
- [ ] host 驱动如何注册 `mmc_host`
- [ ] 板级 host 初始化成功后，后续如何回到标准 MMC/SDIO core
- [ ] host 节点 `interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>` 和 function child node `interrupts` 的区别
- [ ] 如果补了 `wifi@1` 这类 child node，`reg = <1>` 和 `func->dev.of_node` 如何对应
- [ ] 板级电源、复位、时钟、管脚配置错误时，日志通常停在哪一层

### 2.6 工程排查清单

- [ ] 系统完全没有 SDIO 设备时，能按 DTS/host/card/core 顺序排查
- [ ] function 设备存在但驱动不绑定时，能检查 `modalias`、`id_table`、模块加载和匹配函数
- [ ] probe 失败时，能定位失败点在 enable、block size、firmware、I/O 还是 IRQ
- [ ] CMD52 失败时，能优先检查 function enable、地址、claim host 和返回值
- [ ] CMD53 失败时，能优先检查 block size、长度对齐、buffer、host 能力和超时
- [ ] 中断不来时，能先分清 host 控制器 IRQ、SDIO in-band IRQ、out-of-band IRQ，再分层检查
- [ ] WiFi 网络接口不出现时，能继续向 cfg80211/netdev/firmware 层追
- [ ] 能区分“SDIO function driver 已经 probe 成功”和“上层 WiFi/BT 子系统已经注册成功”

## 3. 源码阅读达标题

### 3.1 基础达标

- 能在源码树中找到 `drivers/mmc/core/sdio.c`
- 能在源码树中找到 `drivers/mmc/core/sdio_bus.c`
- 能在源码树中找到 `drivers/mmc/core/sdio_io.c`
- 能在源码树中找到 `drivers/mmc/core/sdio_irq.c`
- 能在源码树中找到 `include/linux/mmc/sdio_func.h`
- 能在源码树中找到 HI3516CV610 的 DTS 和 `sdhci_nebula.c`

### 3.2 主线达标

- 能从 `mmc_attach_sdio()` 追到 `mmc_sdio_init_card()`
- 能从 `mmc_sdio_init_card()` 追到 `sdio_init_func()`
- 能从 `sdio_add_func()` 追到 driver model 设备注册
- 能从 `sdio_bus_probe()` 追到具体 `sdio_driver.probe()`
- 能从 `sdio_readb()` / `sdio_writeb()` 追到 CMD52 和 `host->ops->request`
- 能从 `sdio_memcpy_fromio()` / `sdio_memcpy_toio()` 追到 CMD53 和 `host->ops->request`
- 能从 `sdio_claim_irq()` 追到 SDIO IRQ 分发线程

### 3.3 工程达标

- 给出“probe 不进”的排查路径，不直接跳到芯片私有代码
- 给出“能 probe 但 I/O 超时”的排查路径，不直接怀疑上层网络协议
- 给出“中断不来”的排查路径，能区分 host/core/function driver 三层
- 给出“HI3516CV610 板上 SDIO 完全没枚举”的排查路径，先回到 DTS/host

## 4. 能独立回答的问题

学完后应能独立回答下面这些问题：

- SDIO 和普通 platform driver 的 probe 入口有什么不同
- `sdio_driver.probe()` 之前，内核已经做完了哪些事情
- 一张 SDIO 卡为什么可能有多个 `sdio_func`
- `mmc_card` 和 `sdio_func` 谁先出现，分别代表什么
- `sdio_add_func()` 和 `sdio_bus_probe()` 为什么不是一回事
- `id_table` 匹配失败时，为什么 function 设备可能存在但 driver probe 不进
- `module_sdio_driver()`、`MODULE_DEVICE_TABLE()` 和 `id_table` 三者分别解决什么问题
- `sdio_enable_func()` 失败一般说明问题处在哪个阶段
- `sdio_readb()` 最终是如何落到 host/controller driver 的
- `sdio_memcpy_fromio()` 和 `sdio_readb()` 在协议命令路径上有什么区别
- 教学型字符设备 SDIO 例子和真实 WiFi/BT function driver 的区别是什么
- CMD52 和 CMD53 问题在排查上有什么区别
- SDIO IRQ 为什么要先看 host 是否支持和上报
- 标准 SDIO in-band IRQ 和 out-of-band IRQ 的入口、DTS 依赖和 handler 归属有什么不同
- `sdio0` host 节点里的 `interrupts` 为什么不是 WiFi function 的外部中断
- HI3516CV610 上为什么要先确认 `sdio0`、`mmc0`、`sdio1` 的启用关系
- SDIO WiFi 模组没有网络接口时，为什么不能只看 SDIO core

## 5. 不达标信号

出现下面情况时，说明还需要回到对应章节补读：

- 只知道 `probe()`，说不清 `probe()` 之前的 host/card/function 创建过程
- 能说出 `sdio_func`，但说不清它和 `mmc_card` 的边界
- 遇到设备没出现时，直接去看 WiFi 私有驱动，而不是先看 DTS/host/core
- 看到 `sdio_claim_host()` 只知道“加锁”，但说不清它保护的是 host 访问权
- 看到 CMD52/CMD53 只知道是读写命令，但说不清 API 如何封装 request 并落到 `host->ops->request`
- 中断不来时，只看 function driver handler，不先区分 host 控制器 IRQ、SDIO in-band IRQ 和 out-of-band IRQ
- 板级 DTS 里 `mmc0`、`sdio0`、`sdio1` 的关系还不能快速说清
- 看到 DTS 里的 `interrupts` 就默认是 function 中断，不能判断它属于 host 节点还是 child node
- 不能根据日志判断当前停在 host、card/core、bus/probe、I/O 还是 IRQ 层

## 6. 达标复盘方式

复盘时按下面顺序检查：

1. 先不看笔记画出 SDIO 主线
2. 再打开 [[00-SDIO源码解析总览]] 对照主线是否完整
3. 用 [[01-SDIO核心数据结构]] 检查对象边界是否清楚
4. 用 [[02-SDIO卡枚举与初始化]] 和 [[03-SDIO总线匹配与probe]] 检查枚举到 probe 是否能串起来
5. 用 [[04-SDIO数据通路与常用API]] 和 [[05-SDIO中断机制]] 检查工作阶段是否理解
6. 用 [[07-HI3516CV610板级落地]] 检查板级资源是否能接回主线
7. 用 [[09-SDIO实际工程问题问答]] 随机挑三个现象，按层级写出排查路径

## 7. 一句话标准

真正达标的标准是：看到一个 SDIO 现象时，能先判断它卡在 `DTS/host -> card/core -> func/bus -> driver probe -> I/O -> IRQ -> 上层业务` 的哪一段，再决定看哪份源码、加哪条日志、验证哪个对象。

## 8. 内容清单参考答案

本节用于学完后对答案。不要一开始就背答案，更适合先按前面的清单自己写一遍，再用这里校准是否漏了层级、对象或调用链。

### 8.1 对象清单参考答案

- `mmc_host` 表示 SoC 侧一套 MMC/SD/SDIO 控制器。它由 host controller 驱动创建并注册给 MMC core；对 HI3516CV610 来说，`nebula,sdhci` 先接入 `sdhci_host`，再向上形成 `mmc_host`。
- `mmc_card` 表示 host 上被识别出来的一张卡。SDIO 初始化阶段会把 OCR、CCCR、CIS、function 数量、速度能力、总线宽度等整卡级信息挂到它上面。
- `sdio_func` 表示 SDIO 卡上的一个 function。function driver 的 `probe()` 看到的核心对象就是它，I/O API、IRQ、drvdata、block size 都围绕它展开。
- `sdio_driver` 是绑定到 `sdio_func` 的驱动对象。它通过 `id_table` 告诉 SDIO bus 自己能支持哪些 vendor/device/class 组合。
- `sdio_device_id` 里的 vendor/device/class 用来做 SDIO bus 匹配。`func->vendor` / `func->device` / `func->class` 是枚举阶段从 CIS 等信息填出来的真实设备身份，`sdio_driver.id_table[]` 里的 `sdio_device_id` 是 driver 声明自己能匹配哪些 function。匹配成功后，bus 层才会调用具体 `sdio_driver.probe()`。
- `driver_data` 属于 driver 侧匹配表附加信息，不是设备自己上报的字段；命中后会跟着 `id` 参数传给 `probe(func, id)`，常用来区分芯片版本或能力表。
- host、card、func、driver 的关系是：`mmc_host` 发现并承载 `mmc_card`，`mmc_card` 下创建多个 `sdio_func`，`sdio_func` 注册到 `sdio bus` 后匹配 `sdio_driver`。

### 8.2 枚举和 probe 清单参考答案

- `mmc_add_host()` 把 host 交给 MMC core，core 才能对这套 host 做 rescan，检测上面是否有 MMC、SD 或 SDIO 设备。
- `mmc_rescan()` 会按不同卡类型尝试识别路径。SDIO 路径会走到 `mmc_attach_sdio()`，SD/MMC 则走各自的 attach 路线。
- `mmc_attach_sdio()` 是 SDIO 主线入口，因为它先用 CMD5 探测 SDIO OCR，然后选择电压、初始化 card、创建 function、注册 card/function。
- `mmc_sdio_init_card()` 是整卡初始化核心函数，因为它创建/填充 `mmc_card`，读取 card 级能力，并完成速度、总线宽度等整卡准备。
- `sdio_read_cccr()` 读取 SDIO 公共控制寄存器，确认基础协议能力；`sdio_read_common_cis()` 读取 card 级 CIS，补齐 vendor/device、block size 等公共描述。
- `sdio_init_func()` 在 `mmc_sdio_init_card()` 成功后，根据 function 数量逐个创建 `struct sdio_func`，但这时还没有注册到 driver model。
- `sdio_add_func()` 调 `device_add()`，把 `sdio_func` 变成 SDIO bus 上的 device，之后才可能触发匹配和 probe。
- function device 注册链来自 `mmc_sdio_init_card() -> sdio_init_func() -> sdio_add_func() -> device_add()`；function driver 注册链来自 `module_sdio_driver() -> sdio_register_driver() -> driver_register()`。两条链最终都在 driver model 的 `sdio_bus_match()` 汇合。
- `module_sdio_driver()` 是模块入口辅助宏，加载时调用 `sdio_register_driver()`，卸载时调用 `sdio_unregister_driver()`；`sdio_register_driver()` 把 `struct sdio_driver` 注册到 `sdio bus`；`MODULE_DEVICE_TABLE(sdio, ids)` 导出模块匹配信息，服务自动加载；`id_table` 是内核匹配时真正遍历的表；`probe/remove` 分别是绑定成功后的初始化入口和解绑/卸载时的清理入口。
- `sdio_bus_match()` 用 function 的 vendor/device/class 和 driver 的 `sdio_device_id` 表做匹配，决定哪个 `sdio_driver` 能绑定。
- `sdio_bus_probe()` 是 SDIO bus 的 probe 包装层，做 runtime PM、power domain 等总线级准备，再调用具体 driver 的 `probe()`。
- 具体 `sdio_driver.probe()` 进入后通常会分配私有结构、保存 drvdata、claim host、enable function、设置 block size、初始化芯片、注册 IRQ、接入上层子系统。

### 8.3 I/O 清单参考答案

- `sdio_claim_host()` / `sdio_release_host()` 用来独占和释放 `mmc_host` 访问权，避免多个上下文同时往同一条 SDIO 总线发命令。
- `sdio_enable_func()` 会打开当前 function 的使能位，通常在访问 function 私有寄存器前必须成功。
- `sdio_set_block_size()` 设置当前 function 的块大小，直接影响 CMD53 block mode 的单块长度、传输效率和某些芯片的兼容性。
- `sdio_readb()` / `sdio_writeb()` 通常对应 CMD52 小寄存器读写，适合状态寄存器、控制寄存器、少量配置项。
- `sdio_memcpy_fromio()` / `sdio_memcpy_toio()` 通常对应 CMD53 批量传输，适合 FIFO、mailbox、packet buffer、窗口内存等数据路径。
- `sdio_readb()` / `sdio_writeb()` 的典型路径是 `sdio_readb/writeb -> mmc_io_rw_direct() -> CMD52 -> mmc_wait_for_cmd()/mmc_wait_for_req() -> host->ops->request() -> host/controller 发命令 -> R5 response`。
- `sdio_memcpy_fromio()` / `sdio_memcpy_toio()` 的典型路径是 `sdio_memcpy_fromio/toio -> sdio_io_rw_ext_helper() -> mmc_io_rw_extended() -> CMD53 -> mmc_wait_for_req() -> host->ops->request() -> data complete/error`。
- 教学型字符设备读写例子通常用 `file_operations.read/write/ioctl` 直接调用 `sdio_*` API，适合练习 CMD52/CMD53、claim host、block size 和返回值，但它只是 API 放大镜，不代表真实 WiFi/BT 驱动一定会暴露 `/dev/xxx` 给用户态。
- 真实 WiFi/BT function driver 通常接入网络、蓝牙或其他内核子系统，上层业务路径会先经过 driver 私有 core，再通过 bus ops/transport 调到底层 `sdio_readb/writeb` 或 `sdio_memcpy_fromio/toio`。
- CMD52 问题多表现为寄存器访问失败、状态不对、全零或返回错误；CMD53 问题更常表现为大包超时、数据错位、尾包异常、吞吐异常。
- I/O 超时优先看 host 时钟/电源、function 是否 enable、claim host、block size、长度对齐；全零优先看 function 是否 ready、地址是否正确；错位优先看 block size、长度和芯片协议；吞吐异常优先看 block mode、host 能力、DMA/PIO 和对齐。

### 8.4 IRQ 清单参考答案

- `sdio_claim_irq()` 注册的是 function 级 in-band SDIO IRQ 回调，由 SDIO core 负责后续分发。
- host 层检测到 card 侧 SDIO IRQ 后，通过 `mmc_signal_sdio_irq()` 通知 MMC/SDIO core，core 再唤醒 IRQ 线程或工作队列处理。
- `sdio_irq_thread()` 是 SDIO core 的 IRQ 处理线程；`process_sdio_pending_irqs()` 负责检查哪些 function 有 pending IRQ，并调用对应 function handler。
- host 控制器 IRQ 是 `sdhci`/host 层用来处理命令完成、数据完成、错误和 SDIO IRQ 上报的硬件中断；标准 SDIO in-band IRQ 是 card 通过 SDIO 协议路径上报，再由 core 分发到 `func->irq_handler`；out-of-band IRQ 是模组额外 GPIO/GIC 中断线，由 function driver 从 child node 解析并自己注册 Linux IRQ。
- 外部平台 IRQ handler 不由 SDIO core 自动 claim host；如果 handler 或线程里要访问 SDIO 寄存器或数据路径，通常需要自己 `sdio_claim_host()` / `sdio_release_host()` 包住 CMD52/CMD53 访问。
- function driver 的 IRQ handler 应避免长时间阻塞、重复 claim 已由 core 持有的 host、做复杂睡眠路径或和工作线程形成死锁。
- 中断不来时先分层：host 是否支持并上报 SDIO IRQ，core 是否进入 `sdio_irq_thread()` / pending 分发，function 是否真正产生并使能中断，driver handler 是否注册成功。

### 8.5 HI3516CV610 板级清单参考答案

- 当前板级 DTS 里需要先确认 `mmc0`、`sdio0`、`sdio1` 的 `status`。当前学习主线按 `sdio0 = okay`、`mmc0 = disabled`、`sdio1 = disabled` 理解。
- `sdio0` 重点资源包括 `compatible = "nebula,sdhci"`、`reg = <0x10030000 0x1000>`、host 控制器中断 `interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>`、`clock-names`、`resets`、`crg_regmap`、`iocfg_regmap`、`bus-width = <4>`、`max-frequency = <50000000>` 等。它们分别被 platform/OF、resource、IRQ、clock/reset、regmap、MMC core 或 host driver 消费。
- `sdio0` host 节点里的 `interrupts` 描述的是 host controller 中断，不是 WiFi/BT function 的外部中断。function 的 out-of-band IRQ 应写在 `wifi@1` 这类 child node 下面，再由 function driver 解析成普通 Linux IRQ。
- 如果补了 child node，`reg = <1>` 表示这个 node 对应 SDIO function 1；`mmc_of_find_child_device()` 会根据 function number 把它挂到对应 `func->dev.of_node`。
- `nebula,sdhci` host 驱动的 probe 入口来自 platform driver 对 `compatible = "nebula,sdhci"` 的匹配，后续走 `sdhci_pltfm_init()` 和 `sdhci_nebula_add_host()`。
- host 驱动通过 SDHCI 平台化流程建立 `sdhci_host`，再通过 `sdhci_add_host()` / `mmc_add_host()` 把标准 `mmc_host` 注册给 MMC core。
- 板级 host 初始化成功后，后续回到标准 MMC/SDIO core：`mmc_rescan()` -> `mmc_attach_sdio()` -> `mmc_sdio_init_card()` -> `sdio_init_func()` -> `mmc_add_card()` -> `sdio_add_func()`。
- 板级电源、复位、时钟、管脚配置错误时，日志通常停在 host probe、host 初始化、CMD5 无响应、card/core 枚举失败这些阶段，还不会进入具体 function driver probe。

### 8.6 工程排查清单参考答案

- 系统完全没有 SDIO 设备时，先查 DTS 是否启用正确 host，再查 host driver probe、clock/reset/pinctrl/power、`mmc_add_host()`、CMD5、`mmc_attach_sdio()`。
- function 设备存在但驱动不绑定时，查 `/sys/bus/sdio/devices`、modalias、driver `id_table`、模块是否加载、`sdio_bus_match()` 是否匹配。
- probe 失败时，先看返回值停在哪一步：`sdio_enable_func()`、`sdio_set_block_size()`、固件加载、寄存器访问、IRQ 注册、上层注册。
- CMD52 失败时，优先看 function 是否 enable、地址是否越界、function number 是否正确、是否 claim host、R5 response 是否有错误位。
- CMD53 失败时，优先看 block size、传输长度、byte/block mode、buffer 对齐、host 最大传输能力、DMA/PIO、超时和芯片协议窗口。
- 中断不来时，先分清是 host 控制器 IRQ、标准 SDIO in-band IRQ，还是外部 out-of-band IRQ。host 控制器 IRQ 先查 DTS host `interrupts` 和 SDHCI 中断；in-band IRQ 查 host 上报和 SDIO core 分发；out-of-band IRQ 查 DTS child node、GPIO/GIC IRQ、`devm_request_threaded_irq()`。
- WiFi 网络接口不出现时，SDIO 层只保证 function driver 能绑定和 I/O 可用；还要继续查固件、芯片私有初始化、cfg80211、mac80211/netdev 注册和用户态配置。
- `sdio_driver.probe()` 成功只说明 function driver 已经绑定并完成基础初始化；上层 WiFi/BT 子系统是否可用，还要看 transport ops 是否建立、固件是否启动、IRQ/I/O 是否正常，以及 `ieee80211_register_hw()`、HCI 注册或其他子系统注册是否成功。

## 9. 独立回答问题参考答案

- SDIO 和普通 platform driver 的 probe 入口不同：platform driver 直接由平台设备匹配进入 probe；SDIO function driver 要先经过 host 注册、card 枚举、function 创建、sdio bus 注册和匹配，最后才进入 `sdio_driver.probe()`。
- `sdio_driver.probe()` 之前，内核已经完成 host 初始化、CMD5/OCR 探测、整卡初始化、CCCR/CIS 读取、function 对象创建、card 注册、function 注册和 bus match。
- 一张 SDIO 卡可能有多个 `sdio_func`，因为 SDIO 规范允许一张卡暴露多个功能单元，例如 WiFi、BT、GNSS 或厂商私有 function。
- `mmc_card` 先出现，代表整张卡；`sdio_func` 后出现，代表卡上的某一个 function。function driver 面对的是 `sdio_func`，但它可以通过 `func->card` 回到整卡对象。
- `sdio_add_func()` 和 `sdio_bus_probe()` 不是一回事：前者把 function 注册成 device，后者是在 device 和 driver 匹配成功后，由 bus 层调用具体 driver probe 前的包装入口。
- `id_table` 匹配失败时，function 设备仍然可能存在，因为设备注册由 `sdio_add_func()` 完成；但没有 driver 匹配，所以不会进入具体 `probe()`。
- `module_sdio_driver()` 解决模块加载/卸载时如何注册和注销 `sdio_driver`；`MODULE_DEVICE_TABLE()` 解决模块自动加载所需的设备表导出；`id_table` 解决内核里 `sdio_bus_match()` 如何判断 driver 是否支持当前 `sdio_func`。
- `sdio_enable_func()` 失败通常说明 function 工作阶段还没打开，可能是 card/function 状态、CMD52 访问、host 通信、电源/时钟或芯片响应问题。
- `sdio_readb()` 最终会由 SDIO core 封装成 CMD52，再经 MMC core 的 request 路径交给 `host->ops->request()`，最后由 host/controller driver 真正发命令并从 R5 response 里取回状态或数据。
- `sdio_memcpy_fromio()` 和 `sdio_readb()` 的区别是：前者通常走 CMD53 extended I/O，带 data 阶段，面向 FIFO/window/buffer 批量搬运；后者通常走 CMD52 direct I/O，偏单字节寄存器读写。
- 教学型字符设备 SDIO 例子通常把用户态 `read/write/ioctl` 直接连到 `sdio_*` API，方便学习和调试；真实 WiFi/BT function driver 通常不直接暴露字符设备，而是接入 mac80211、蓝牙 HCI 或其他内核子系统，并通过私有 bus ops/transport 间接使用 SDIO API。
- CMD52 和 CMD53 的排查区别：CMD52 偏寄存器和控制面，重点看地址、function、R5 状态、enable；CMD53 偏批量数据面，重点看 block size、长度、buffer、DMA/PIO、超时和芯片协议。
- SDIO IRQ 先看 host 是否支持和上报，因为标准 in-band SDIO IRQ 是先由 host 感知，再通知 MMC/SDIO core 分发，不是 function driver handler 自己凭空触发。
- 标准 SDIO in-band IRQ 由 `sdio_claim_irq()` 建立，依赖 CCCR 中断使能、host SDIO IRQ 上报和 `sdio_irq_thread()` 分发；out-of-band IRQ 来自额外 GPIO/GIC 中断线，通常写在 function child node 里，由 function driver 自己解析并注册普通 Linux IRQ。
- `sdio0` host 节点里的 `interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>` 是 host controller 的中断资源，供 SDHCI host 处理命令完成、数据完成、错误和 SDIO IRQ 上报；它不是 WiFi function 的外部中断。WiFi function 的 out-of-band IRQ 应写在 `wifi@1` 这类 child node 下面。
- HI3516CV610 上要先确认 `sdio0`、`mmc0`、`sdio1` 的启用关系，因为 `mmc0` 和 `sdio0` 共享控制器地址，DTS 选错会让后面所有 SDIO 分析都建立在错误 host 上。
- SDIO WiFi 模组没有网络接口时不能只看 SDIO core，因为 SDIO core 只负责枚举、匹配和 I/O/IRQ 通道；网络接口还依赖 WiFi 驱动私有初始化、固件、cfg80211/mac80211 或 netdev 注册。
