# 典型 Function 驱动例子

## 导读

### 本章定位

这一章用 `drivers/staging/wfx/bus_sdio.c` 作为 SDIO function driver 示例，把前面几章的匹配、probe、claim host、I/O、IRQ、remove 回滚流程落到一个具体驱动里。

### 核心对象

- `struct sdio_driver`
  - `wfx` SDIO 驱动注册对象
- `struct sdio_func` #sdio_func
  - probe 入口拿到的 function 设备
- `struct device_node`
  - 当前 `sdio_func` 对应的设备树 child node，后面会继续映射到 `compatible`、`reg` 和 `interrupts`
- `struct wfx_sdio_priv` #wfx_sdio_priv
  - `wfx` 的 SDIO 私有总线对象
- `sdio_set_drvdata() / sdio_get_drvdata()`
  - function 和驱动私有对象之间的绑定入口

### 关键函数

- `wfx_sdio_probe()`
- `wfx_sdio_remove()`
- `mmc_of_find_child_device()`
- `of_match_node()`
- `irq_of_parse_and_map()`
- `sdio_enable_func()`
- `sdio_set_block_size()`
- `sdio_memcpy_fromio() / sdio_memcpy_toio()`
- `sdio_claim_irq()`
- `devm_request_threaded_irq()`

### 主流程

sdio bus 匹配 -> `wfx_sdio_probe()` -> 保存私有对象 -> enable function -> 设置 block size -> 注册 IRQ -> 封装读写路径 -> 接入上层业务。

## 这一章按什么逻辑展开

这一章按“先看 driver 壳子，再按 `probe -> I/O -> IRQ -> remove` 这条实际驱动生命周期展开”的逻辑展开。

这样拆的原因是：

- 示例驱动章节的目标不是重新讲一遍框架概念
- 而是把前面枚举、probe、I/O、IRQ、回滚这些规则，按一个真实驱动的执行顺序重新落地

所以本章后面的结构是：

1. 先看为什么选这个例子
2. 再看 `sdio_driver` 壳子和 `id_table`
3. 再按 `probe()` 内部顺序展开初始化
4. 再看 I/O、IRQ、失败回滚和 `remove()` 怎样闭合

## 1. 为什么选 `wfx`

示例文件：

- `drivers/staging/wfx/bus_sdio.c`

选它的原因很简单：

- 文件短
- 结构清楚
- 把 `probe`、I/O、IRQ、block size、`drvdata` 都走了一遍

它比某些大型 WiFi 驱动更适合拿来做“第一次通读 SDIO function driver”的例子。

## 2. 它的 driver 壳子非常标准

文件末尾可以看到：

- `wfx_sdio_ids[]`
- `struct sdio_driver wfx_sdio_driver`

这正对应前面讲过的 `sdio bus` 匹配模型。

这里先区分两类对象：

- 前面章节已经讲过的通用对象
  - `struct sdio_func`：见 [[01-SDIO核心数据结构]]
  - `struct sdio_driver`：见 [[01-SDIO核心数据结构]] 和 [[03-SDIO总线匹配与probe]]
- 这一章新出现、而且后面会反复用到的私有对象
  - `struct wfx_sdio_priv`

也就是说，这一章真正需要新增展开的是把 `wfx` 自己的私有总线对象讲清楚。

### 2.1 `struct wfx_sdio_priv` 是什么

它定义在：

- `drivers/staging/wfx/bus_sdio.c`

>[!INFO]
```C
struct wfx_sdio_priv {
	struct sdio_func *func;
	struct wfx_dev *core;
	u8 buf_id_tx;
	u8 buf_id_rx;
	int of_irq;
};
```
这个结构体不是 SDIO core 的通用对象，而是 `wfx` 这个示例驱动自己的私有状态容器。

它的作用是：

- 把 `sdio_func`
- 上层 `wfx core`
- 发送/接收队列状态
- 可能存在的外部 IRQ

收拢到一个驱动私有对象里，后面 `probe()`、I/O、IRQ、remove 都围绕它展开。

### 2.2 这个私有结构里哪些字段最关键

- `struct sdio_func *func`
  - 当前 function driver 绑定到的 SDIO function
  - 后面所有 `sdio_*` API 基本都要通过它进入
- `struct wfx_dev *core`
  - 指向更上层的 `wfx` 公共核心对象
  - 说明 `bus_sdio.c` 这一层只是总线适配层，不是全部业务逻辑
- `u8 buf_id_tx`
  - 发送路径里的 queue/buffer 索引
  - 配合 `sdio_memcpy_toio()` 组织芯片私有发送窗口
- `u8 buf_id_rx`
  - 接收路径里的 queue/buffer 索引
  - 配合 `sdio_memcpy_fromio()` 组织芯片私有接收窗口
- `int of_irq`
  - 从设备树解析出来的外部 IRQ 号
  - `0` 表示没有可用的外部 IRQ，于是回退走标准 `sdio_claim_irq()` 路线
  - 非 `0` 表示当前板级给了额外 IRQ，于是走 `devm_request_threaded_irq()`

### 2.3 为什么这一章必须把它单独讲出来

因为后面很多看起来像“函数分支”的东西，其实都是围绕 `wfx_sdio_priv` 里的字段展开：

- `func`
  - 决定控制面、数据面、IRQ API 最终操作哪个 SDIO function
- `buf_id_tx / buf_id_rx`
  - 决定收发地址为什么会额外拼出 queue 偏移
- `of_irq`
  - 决定 IRQ 路径为什么分成标准 SDIO IRQ 和外部平台 IRQ 两种
- `core`
  - 决定 IRQ handler 为什么只是请求上层继续收包，而不是在这里直接做全部业务处理

### 2.4 这个例子对应的设备树节点长什么样

这一章虽然是 function driver 例子，但它并不是完全脱离设备树工作的。

从源码里已经能确认的事实有三条：

1. `wfx` 接受这两个 `compatible`
   - `silabs,wfx-sdio`
   - `silabs,wf200`
2. `wfx_sdio_probe()` 里会取：
   - `struct device_node *np = func->dev.of_node;`
3. `wfx_sdio_irq_subscribe()` 是否走外部 IRQ 路线，取决于：
   - `bus->of_irq`

按这些事实反推，最小可用的节点形态可以先理解成：

```dts
wifi@1 {
	compatible = "silabs,wfx-sdio";
	reg = <1>;
	interrupts = <...>;        /* 如果板级走外部 IRQ */
	interrupt-parent = <...>;  /* 如果平台需要显式指定 */
};
```

这里最重要的不是节点名 `wifi@1` 本身，而是下面这些字段：

- `compatible`
  - 要能被 `wfx_sdio_of_match[]` 命中
- `reg = <1>`
  - 要和当前 function 编号对应
- `interrupts`
  - 如果板级走外部 IRQ 路线，就要提供可解析的第一个 IRQ 资源

### 2.5 代码字段和设备树是怎么连起来的

这一层的连接关系不是 `wfx` 驱动自己随便定义的，而是 SDIO core 先帮它把 function 和 OF child node 对上。

这条链可以直接写成：

```text
host 的 OF 节点下面的 child node
-> mmc_of_find_child_device(host, func_num)
-> func->dev.of_node
-> wfx_sdio_probe() 里的 np
-> of_match_node()
-> irq_of_parse_and_map()
-> bus->of_irq
```

这里最关键的源码点有两处：

1. `sdio_add_func()` 里：
   - `func->dev.of_node = mmc_of_find_child_device(host, func->num);`
2. `mmc_of_find_child_device()` 里：
   - 逐个找 host 父节点下面的 child node
   - 比较的是 child node 的 `reg`
   - 谁的 `reg == func->num`，谁就会挂到这个 `sdio_func`

所以这条关系要分三步理解：

- `reg`
  - 决定这个 DT child node 属于哪个 function
- `compatible`
  - 决定 `wfx_sdio_of_match[]` 能不能匹配这个 node
- `interrupts`
  - 决定 `irq_of_parse_and_map(np, 0)` 能不能解析出 `bus->of_irq`

也就是说，前面在 `probe()` 里看到的这些字段：

- `func->num`
- `np = func->dev.of_node`
- `bus->of_irq`

其实都能回到设备树里找到对应来源：

- `func->num` <- `reg`
- `np` <- host 节点下面、且 `reg` 命中当前 function 的 child node
- `bus->of_irq` <- 这个 child node 的第一个 IRQ 资源

### 2.6 这和后面两条 IRQ 路径的关系

把设备树字段和后面的 IRQ 分支连起来看，就会更清楚：

- 如果这个 child node 没有可用 IRQ 资源
  - `bus->of_irq` 就是 `0`
  - 驱动回退走标准 `sdio_claim_irq()` 路线
- 如果这个 child node 提供了可解析的 IRQ
  - `bus->of_irq != 0`
  - 驱动就走 `devm_request_threaded_irq()` 这条外部 IRQ 路线

所以后面第 `5` 节里两种 IRQ 路径的分支，并不是凭空出现的，而是前面这个设备树节点是否提供额外 IRQ 资源的直接结果。

## 3. 它的 `probe()` 值得怎么读

主入口：

- `wfx_sdio_probe(struct sdio_func *func, const struct sdio_device_id *id)`

>[!INFO]
```C {8,31,32,33,37-41,} fold:"wfx_sdio_probe"
static int wfx_sdio_probe(struct sdio_func *func,
			  const struct sdio_device_id *id)
{
	struct device_node *np = func->dev.of_node;
	struct wfx_sdio_priv *bus;
	int ret;

	if (func->num != 1) {
		dev_err(&func->dev, "SDIO function number is %d while it should always be 1 (unsupported chip?)\n",
			func->num);
		return -ENODEV;
	}

	bus = devm_kzalloc(&func->dev, sizeof(*bus), GFP_KERNEL);
	if (!bus)
		return -ENOMEM;

	if (np) {
		if (!of_match_node(wfx_sdio_of_match, np)) {
			dev_warn(&func->dev, "no compatible device found in DT\n");
			return -ENODEV;
		}
		bus->of_irq = irq_of_parse_and_map(np, 0);
	} else {
		dev_warn(&func->dev,
			 "device is not declared in DT, features will be limited\n");
		// FIXME: ignore VID/PID and only rely on device tree
		// return -ENODEV;
	}

	bus->func = func;
	sdio_set_drvdata(func, bus);
	func->card->quirks |= MMC_QUIRK_LENIENT_FN0 |
			      MMC_QUIRK_BLKSZ_FOR_BYTE_MODE |
			      MMC_QUIRK_BROKEN_BYTE_MODE_512;

	sdio_claim_host(func);
	ret = sdio_enable_func(func);
	// Block of 64 bytes is more efficient than 512B for frame sizes < 4k
	sdio_set_block_size(func, 64);
	sdio_release_host(func);
	if (ret)
		goto err0;

	bus->core = wfx_init_common(&func->dev, &wfx_sdio_pdata,
				    &wfx_sdio_hwbus_ops, bus);
	if (!bus->core) {
		ret = -EIO;
		goto err1;
	}

	ret = wfx_probe(bus->core);
	if (ret)
		goto err1;

	return 0;

err1:
	sdio_claim_host(func);
	sdio_disable_func(func);
	sdio_release_host(func);
err0:
	return ret;
}
```

按下面顺序读：
[[06-典型Function驱动例子#3. 它的 `probe()` 值得怎么读]]
### 3.1 先检查 function 编号

它一开始就判断：

- `func->num != 1` 直接报错

这说明对这个芯片来说，驱动明确预期功能挂在 function 1。

### 3.2 分配私有结构并保存到 `drvdata`

它分配了：

- `struct wfx_sdio_priv`

然后：

- `sdio_set_drvdata(func, bus)`

这就是 function driver 的典型写法。

### 3.3 设置 quirks

它会改：

- `func->card->quirks`

这说明某些芯片的行为，确实会逼着 function driver 去给 core 打补丁式地开 quirk。

### 3.4 在 claim host 后打开 function

它走的顺序是：

1. `sdio_claim_host(func)`
2. `sdio_enable_func(func)`
3. `sdio_set_block_size(func, 64)`
4. `sdio_release_host(func)`

这正好是前面说的标准控制面初始化顺序。

## 4. 它的数据收发路径怎么写

### 4.1 读数据

- `wfx_sdio_copy_from_io()`
- 内部调用 `sdio_memcpy_fromio()`
>[!INFO]
```C {15,17} fold:"wfx_sdio_copy_from_io"
static int wfx_sdio_copy_from_io(void *priv, unsigned int reg_id,
				 void *dst, size_t count)
{
	struct wfx_sdio_priv *bus = priv;
	unsigned int sdio_addr = reg_id << 2;
	int ret;

	WARN(reg_id > 7, "chip only has 7 registers");
	WARN(((uintptr_t)dst) & 3, "unaligned buffer size");
	WARN(count & 3, "unaligned buffer address");

	/* Use queue mode buffers */
	if (reg_id == WFX_REG_IN_OUT_QUEUE)
		sdio_addr |= (bus->buf_id_rx + 1) << 7;
	ret = sdio_memcpy_fromio(bus->func, dst, sdio_addr, count);
	if (!ret && reg_id == WFX_REG_IN_OUT_QUEUE)
		bus->buf_id_rx = (bus->buf_id_rx + 1) % 4;

	return ret;
}
```
### 4.2 写数据

- `wfx_sdio_copy_to_io()`
- 内部调用 `sdio_memcpy_toio()`
>[!INFO]
```C {16,18} fold:"wfx_sdio_copy_to_io"
static int wfx_sdio_copy_to_io(void *priv, unsigned int reg_id,
			       const void *src, size_t count)
{
	struct wfx_sdio_priv *bus = priv;
	unsigned int sdio_addr = reg_id << 2;
	int ret;

	WARN(reg_id > 7, "chip only has 7 registers");
	WARN(((uintptr_t)src) & 3, "unaligned buffer size");
	WARN(count & 3, "unaligned buffer address");

	/* Use queue mode buffers */
	if (reg_id == WFX_REG_IN_OUT_QUEUE)
		sdio_addr |= bus->buf_id_tx << 7;
	// FIXME: discards 'const' qualifier for src
	ret = sdio_memcpy_toio(bus->func, sdio_addr, (void *)src, count);
	if (!ret && reg_id == WFX_REG_IN_OUT_QUEUE)
		bus->buf_id_tx = (bus->buf_id_tx + 1) % 32;

	return ret;
}
```

它还额外维护了：

- `buf_id_tx`
- `buf_id_rx`

这属于芯片私有的 queue/buffer 组织方式，但底层 API 仍然是标准 SDIO I/O。

## 5. 它的 IRQ 路径有两个版本

这一节按“先看分支条件，再分别看两条 IRQ 路径，最后做并排对照”的逻辑展开。

这样拆的原因是：

- 这不是两个互不相干的中断实现
- 而是同一颗 SDIO 芯片在两种板级接法下的两种 IRQ 接入方式

分支点就在 `wfx_sdio_irq_subscribe()`：

- `bus->of_irq == 0`
  - 走标准 SDIO in-band IRQ
- `bus->of_irq != 0`
  - 走设备树提供的外部 IRQ

### 5.1 纯 SDIO in-band IRQ
[[05-SDIO中断机制#4.解决“host 报告有中断”，mmc_signal_sdio_irq()`]]
如果没有外部 IRQ：

- `sdio_claim_irq(bus->func, wfx_sdio_irq_handler)`

对应回调是：

- `wfx_sdio_irq_handler(struct sdio_func *func)`

这条路径的完整链路是：

```text
wfx_sdio_irq_subscribe()
-> sdio_claim_irq()
-> SDIO core 打开 IENx
-> host 感知 card 侧 SDIO IRQ
-> mmc_signal_sdio_irq()
-> sdio_irq_thread()
-> process_sdio_pending_irqs()
-> wfx_sdio_irq_handler(func)
```

这条路径的几个关键点是：

- 这是标准 SDIO IRQ 机制
  - 中断分发由 SDIO core 完成
- 回调参数是 `struct sdio_func *`
  - 说明它是 function 级中断回调
- handler 运行时 host 已经由 core claim
  - 所以 `wfx_sdio_irq_handler()` 里不需要再 `sdio_claim_host()`

这也是为什么这个 handler 很短：

- 先 `sdio_get_drvdata(func)`
- 再请求上层去做收包处理

它的重点不是在 IRQ 回调里直接搬数据，而是把“该收包了”这件事交给后续路径。

### 5.2 设备树外部 IRQ
[[07-HI3516CV610板级落地#3. `sdio0` 节点当前配置了什么]]
如果 DTS 提供了外部 IRQ：

- 用 `devm_request_threaded_irq()`
- 然后手动改 `CCCR_IENx`

对应回调是：

- `wfx_sdio_irq_handler_ext(int irq, void *priv)`

这条路径的完整链路是：

```text
DTS 解析出 bus->of_irq
-> devm_request_threaded_irq()
-> 平台 IRQ 触发
-> wfx_sdio_irq_handler_ext()
```

而在注册这条 IRQ 之后，驱动又手动做了：

- `sdio_f0_readb(..., SDIO_CCCR_IENx, ...)`
- 置 `BIT(0)` 和 `BIT(func->num)`
- `sdio_f0_writeb(..., SDIO_CCCR_IENx, ...)`

这说明虽然中断入口换成了外部平台 IRQ，但 card 内部对应 function 的中断使能仍然要打开。

这条路径的几个关键点是：

- 入口来自 Linux 通用 IRQ 子系统
  - 不再是 SDIO core 的 `sdio_irq_thread()` 分发
- 回调参数变成 `(irq, priv)`
  - 这是标准平台 IRQ handler 形式
- handler 里需要自己 `sdio_claim_host()`
  - 因为这里不是在 SDIO core 已 claim host 的上下文里

这也是 `wfx_sdio_irq_handler_ext()` 和前一个 handler 最大的区别：

- 它进入后先 `sdio_claim_host(bus->func)`
- 再请求上层做收包
- 最后 `sdio_release_host(bus->func)`

也就是说：

- 标准 SDIO IRQ 路径里，host claim 由 core 代管
- 外部 IRQ 路径里，host claim 由驱动自己负责

### 5.3 这两条路径并排看，差别到底在哪

可以直接压成下面这组对照：

- `sdio_claim_irq()`
  - 标准 SDIO in-band IRQ
  - 由 SDIO core 分发
  - 回调参数是 `struct sdio_func *`
  - handler 运行时 host 已由 core claim

- `devm_request_threaded_irq()`
  - 外部平台 IRQ
  - 由 Linux 通用 IRQ 子系统分发
  - 回调参数是 `(irq, priv)`
  - handler 里要自己 `sdio_claim_host()`

### 5.4 为什么同一个驱动要支持两种 IRQ

因为同一颗 SDIO 芯片在不同板级设计下，可能有两种接法：

1. 纯标准 SDIO IRQ
   - 不依赖额外 GPIO/平台中断线
2. 芯片额外拉出一个外部 IRQ pin
   - 板级通过 DTS 把这个 IRQ 交给平台 IRQ 子系统

所以这不是“重复实现了两套无关中断代码”，而是：

- 同一套 function driver
- 兼容两种板级中断接入方式

这样回头再看这一节，关键就不是“有两个 handler”，而是：

- 标准 SDIO IRQ 走哪条链
- 外部 IRQ 走哪条链
- 哪一条路径由 core claim host
- 哪一条路径由驱动自己 claim host

这个例子很有价值，因为它说明：

- 一些芯片既能走标准 SDIO IRQ
- 也可能结合 out-of-band 中断设计

## 6. 这个驱动能说明什么
[[04-SDIO数据通路与常用API#8. 和示例驱动的对应关系]]
### 6.1 最小可用框架

它几乎完整展示了一个 SDIO function driver 的骨架：

- `probe/remove`
- `drvdata`
- host claim/release
- function enable/disable
- block size
- IRQ
- I/O 访问

### 6.2 什么时候应该包一层总线抽象

`wfx` 没有让业务层直接到处调用 `sdio_*` API，而是封装成：

- `copy_from_io`
- `copy_to_io`
- `irq_subscribe`
- `irq_unsubscribe`
- `lock/unlock`

这对后续维护很有帮助，尤其是一个芯片同时支持 SDIO/SPI 等多种总线时。

## 7. 编写 SDIO function driver 的最小顺序

可以先照着这条顺序搭起来：

1. 建私有结构
2. `sdio_set_drvdata()`
3. claim host
4. `sdio_enable_func()`
5. `sdio_set_block_size()`
6. release host
7. 初始化芯片寄存器
8. 注册 IRQ
9. 接入上层子系统

退出时做反向清理：

1. 注销上层子系统
2. 释放 IRQ
3. claim host
4. `sdio_disable_func()`
5. release host

## 8. 为什么这一章和 HI3516CV610 有关

虽然 `wfx` 不是 HI3516CV610 私有驱动，但它所在的上层框架和当前板级一致：

- 下面还是 `sdhci/nebula`
- 上面还是 `sdio core`
- function driver 看到的仍然是 `struct sdio_func` 和同一组 `sdio_*` API

因此这个例子适合用来分析后续挂在 `sdio0` 上的实际模组。

