# SDIO 调试与阅读建议

## 学习目标

- 把 SDIO 问题按 DTS/host、card/core、bus/probe、I/O、IRQ 分层定位
- 建立稳定的源码阅读顺序
- 把常见故障现象映射回对应对象和函数

## 导读

### 本章定位

这一章是 SDIO 主线学习后的收束章，负责把对象、枚举、probe、I/O、中断、板级落地重新压回调试视角。

### 核心对象

- `mmc_host`
  - 控制器和 DTS/host 问题的核心对象
- `mmc_card`
  - 卡识别和 CCCR/CIS 问题的核心对象
- `sdio_func`
  - function 设备、I/O、IRQ 问题的核心对象
- `sdio_driver`
  - probe/remove、芯片初始化问题的核心对象

### 关键函数

- `mmc_attach_sdio()`
- `mmc_sdio_init_card()`
- `sdio_add_func()`
- `sdio_bus_probe()`
- `sdio_enable_func()`
- `sdio_claim_irq()`
- `sdio_memcpy_fromio() / sdio_memcpy_toio()`

### 主流程

先分层 -> 再看枚举 -> 再看 probe -> 再看 I/O 和 IRQ -> 最后回到具体模组私有协议。

## 这一章按什么逻辑展开

这一章按“先分层，再给阅读顺序，再给工程定位入口”的逻辑展开。

这样拆的原因是：

- 调试章的重点不是增加新框架知识
- 而是把前面已经学过的对象和主流程重新压回排查视角
- 如果不先分层，后面很容易一上来就钻寄存器或某个私有协议细节

所以本章后面的结构是：

1. 先判断问题落在哪一层
2. 再给不同现象下的源码阅读切入顺序
3. 最后把当前板级和常见现象对应起来

## 1. 先分层定位，不要一上来就钻寄存器

先判断问题落在哪一层：

### 1.1 DTS/host 层

- 控制器节点有没有启用
- `mmc0` / `sdio0` / `sdio1` 有没有选错
- 时钟、复位、bus-width、max-frequency 是否合理
- `nebula,sdhci` 有没有 probe 成功

### 1.2 card/core 层

- `mmc_attach_sdio()` 有没有走到    [[02-SDIO卡枚举与初始化#2. `mmc_attach_sdio()` 做了什么]]
- `mmc_sdio_init_card()` 有没有报错   [[02-SDIO卡枚举与初始化#3. `mmc_sdio_init_card()` 是整条链最关键的函数]]
- `sdio_read_cccr()` / `sdio_read_common_cis()` 是否成功    [[02-SDIO卡枚举与初始化#4. `sdio_read_cccr()` 到底读了什么]]
- function 数量是否解析正确

### 1.3 function driver 层

- `sdio_add_func()` 后设备是否出现   [[02-SDIO卡枚举与初始化#6. `sdio_add_func()` 才是真正把 function 暴露给驱动]]
- `sdio_driver.probe()` 是否进入    [[03-SDIO总线匹配与probe#4. `sdio_bus_probe()` 做了哪些通用准备]]
- `sdio_enable_func()`、`sdio_set_block_size()` 是否成功   [[04-SDIO数据通路与常用API#2.1 `sdio_enable_func()`]]
- `sdio_claim_irq()` 是否成功  [[05-SDIO中断机制#3. 先建立 IRQ 通路：`sdio_claim_irq()`]]

## 2. 对 HI3516CV610 当前板级，第一眼先看什么

针对当前板型，第一眼先确认：

- `hi3516cv610-demb.dts` 中 `sdio0` 是否为 `okay`
- `mmc0` 是否被关掉
- `sdio1` 是否被关掉

因为 `mmc0` 和 `sdio0` 共用 `0x10030000`，这里一旦配错，后面所有上层分析都会偏。

## 3. 阅读源码时的最佳切入顺序

### 3.1 想搞清“为什么 probe 没进”

读：

1. `sdio_add_func()`
2. `sdio_bus_match()`
3. `sdio_bus_probe()`
4. 具体驱动的 `id_table`

### 3.2 想搞清“为什么卡没起来”

读：

1. `mmc_attach_sdio()`
2. `mmc_sdio_init_card()`
3. `sdio_read_cccr()`
4. `sdio_init_func()`

### 3.3 想搞清“为什么收发异常”

读：

1. `sdio_set_block_size()`
2. `sdio_memcpy_toio()`
3. `sdio_memcpy_fromio()`
4. 具体驱动里的长度对齐和 buffer 组织

### 3.4 想搞清“为什么中断不通”

读：

1. `sdio_claim_irq()`
2. `mmc_signal_sdio_irq()`
3. `sdio_irq_thread()`
4. `process_sdio_pending_irqs()`
5. host 的 `enable_sdio_irq`

## 4. 最常见的几个坑

### 4.1 忘记 claim host

表现：

- 偶发错误
- 并发场景下行为混乱

### 4.2 `enable_func` 成功前就访问 function 寄存器

表现：

- `CMD52/CMD53` 报错
- 读写全零或超时

### 4.3 block size 设得不合适

表现：

- 大包异常
- 尾包错位
- 吞吐异常

### 4.4 IRQ handler 里重复 claim host

表现：

- 死锁
- 卡死

原因：

- `sdio_claim_irq()` 的 handler 被调用时，host 通常已经由 core 持有

### 4.5 remove 路径忘记 `sdio_release_irq()`

表现：

- 卸载时告警
- 下次加载异常

## 5. 面向实战的排查顺序

调试真实模组时，固定按这个顺序展开：

1. 看 DTS 是否启用了对的控制器
2. 看 `sdhci_nebula` 是否 probe 成功
3. 看 `mmc_attach_sdio()` 是否成功识别出卡
4. 看 function 数量和 `sdio_add_func()` 是否正常
5. 看 function driver `probe()` 是否进入
6. 看 `enable/block size/IRQ` 三件套是否成功
7. 再看具体业务收发

这个顺序的好处是：

- 每一步都能明确卡在“更下层”还是“更上层”
- 不会一开始就陷进某个模组私有寄存器里

更细的工程问题拆解见 [[09-SDIO实际工程问题问答]]。

## 6. 最后的阅读方法

SDIO 源码不要按文件名横着扫，最好按调用链竖着看：

```text
mmc_attach_sdio
  -> mmc_sdio_init_card
  -> sdio_init_func
  -> sdio_add_func
  -> sdio_bus_probe
  -> your_sdio_driver_probe
```

这条调用链更适合按三段读：

1. `mmc_attach_sdio -> mmc_sdio_init_card`
   - 这是整卡初始化阶段
   - 先确认卡按 SDIO 路线工作，再把 `mmc_card` 站稳
2. `sdio_init_func -> sdio_add_func`
   - 这是 function 创建和注册阶段
   - 先把 `sdio_func` 创建出来，再交给 `sdio bus`
3. `sdio_bus_probe -> your_sdio_driver_probe`
   - 这是 bus 匹配和具体驱动进入阶段
   - 说明 function driver 的 `probe()` 已经是这条链的后半段

这条图解决的是“源码阅读时该沿哪一条竖向主线往下走”的问题。

把它和本章前面的调试分层对应起来，就是：

- 前半段出问题，优先看 host/card/core
- 中间段出问题，优先看 `sdio_func` 创建和注册
- 后半段出问题，优先看 bus 匹配和 function driver `probe()`

这条调用链跑顺以后，无论是看 WiFi、BT，还是写一个简单 SDIO function driver，都能先判断问题卡在哪一层。

## 7. 回答的问题

- SDIO 问题应该先从哪一层开始定位
- 卡没起来、probe 没进、读写异常、中断不通分别应该看哪些函数
- HI3516CV610 当前板级为什么要先确认 `sdio0/mmc0/sdio1` 的启用状态
- SDIO 源码为什么适合按调用链竖着读

