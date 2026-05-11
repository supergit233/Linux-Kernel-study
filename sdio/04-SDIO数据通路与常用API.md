# SDIO 数据通路与常用 API

## 导读

### 本章定位

这一章讲 function driver 进入 `probe()` 之后，如何通过 SDIO core 提供的 API 完成 function 使能、block size 配置、寄存器访问和批量数据传输。

### 核心对象

- `struct sdio_func`
  - 所有 SDIO I/O API 的目标对象
- `struct mmc_host`
  - I/O 事务实际占用的 host 控制器
- `func->cur_blksize`
  - 当前 function 的 block size
- host claim 状态
  - 保护一次 SDIO 总线事务的串行访问条件

### 关键函数

- `sdio_claim_host()`
- `sdio_release_host()`
- `sdio_enable_func()`
- `sdio_disable_func()`
- `sdio_set_block_size()`
- `sdio_readb() / sdio_writeb()`
- `sdio_memcpy_fromio() / sdio_memcpy_toio()`
- `sdio_readsb() / sdio_writesb()`
- `sdio_align_size()`

### 主流程

claim host -> enable function -> 设置 block size -> 读写寄存器或搬运数据 -> release host。

## 这一章按什么逻辑展开

这一章按“先控制面 API，再数据面 API，再访问约束，最后落到真实驱动”的逻辑展开。

这样拆的原因是：

- 只按函数名平铺，容易看到一堆 `sdio_*` API，却不知道哪些是打开 function、哪些是配置、哪些是搬数据、哪些是访问约束
- 本章真正要建立的是 function driver 进入 `probe()` 之后，怎样把这些 API 组织成一条可运行的数据路径

所以本章后面的结构是：

1. 先看控制面 API：`enable`、`disable`、`set_block_size`
2. 再看数据面 API：寄存器访问和批量传输
3. 再看 `claim/release` 这条硬约束
4. 最后用示例驱动把初始化、收发、IRQ 接缝和退出路径串起来

## 1. 这一层看什么

如果说 `sdio.c` 解决的是“设备怎么出来”，那么 `sdio_io.c` 解决的就是：

- function 怎么上电
- block size 怎么配置
- 寄存器怎么读写
- 数据怎么搬运

核心文件：

- `drivers/mmc/core/sdio_io.c`

## 2. 最常见的控制面 API

### 2.1 `sdio_enable_func()`

作用：

- 通过 `CCCR_IOEx` 打开某个 function
- 轮询 `CCCR_IORx` 等待 function ready

直觉理解：

- 没有 `enable`，后面的 function 寄存器访问通常都不成立

### 2.2 `sdio_disable_func()`

作用：

- 清掉 `CCCR_IOEx` 对应 bit

它是 `enable` 的对偶操作，通常在 `remove()` 或异常回滚路径里出现。

### 2.3 `sdio_set_block_size()`

作用：

- 设置 function 的块大小寄存器
- 更新 `func->cur_blksize`

默认策略：

- 如果传 `0`，core 会取 host 和 function 都支持的合理默认值
- 上限通常不会超过 `512`

这就是为什么很多 function driver 一开始会先接受 core 的默认值，然后再按芯片特性改成 `64`、`128`、`256` 或 `512`

## 3. 常见数据访问 API

### 3.1 单字节寄存器访问

- `sdio_readb()`
- `sdio_writeb()`
- `sdio_writeb_readb()`

适合：

- 控制寄存器
- 状态寄存器
- 小量配置项

### 3.2 批量数据传输

- `sdio_memcpy_fromio()`
- `sdio_memcpy_toio()`
- `sdio_readsb()`
- `sdio_writesb()`

适合：

- FIFO
- packet buffer
- mailbox
- window memory

大多数 WiFi/BT SDIO 驱动最核心的收发路径都在这里。

## 4. `sdio_align_size()` 为什么有用

作用：

- 把一次传输长度调整为 host/card 更容易接受的尺寸

它不是功能正确性的前提，但常常能改善：

- 传输效率
- block mode 利用率
- 某些 host/controller 的兼容性

网络类驱动常把它作为发包前的一步预处理。

## 5. 使用这些 API 的一个硬规则

绝大多数时候，调用前都要满足下面两个条件之一：

1. 已经 `sdio_claim_host()`
2. 当前处于 SDIO IRQ handler 上下文，host 已由 core 代为持有

这也是很多驱动里反复出现下面写法的原因：

```c
sdio_claim_host(func);
... sdio_enable_func / sdio_set_block_size / sdio_memcpy_toio ...
sdio_release_host(func);
```

不要把它当模板噪音，它其实是在保护同一个 host 上的串行访问。

## 6. 一个常见的初始化顺序

典型顺序通常是：

1. `sdio_claim_host(func)`
2. `sdio_enable_func(func)`
3. `sdio_set_block_size(func, xxx)`
4. `sdio_release_host(func)`

后面真正收发数据时，再按需要：

1. `sdio_claim_host(func)`
2. `sdio_memcpy_toio()` 或 `sdio_memcpy_fromio()`
3. `sdio_release_host(func)`

## 7. 为什么“块大小”特别值得关注

因为它直接影响：

- 每次传输走 block mode 还是 byte mode
- 一次命令能搬多少数据
- 某些芯片是否出现“长度对不上”“尾包异常”“偶发超时”

如果驱动跑不稳定，块大小常常是第一批要核对的参数。

## 8. 和示例驱动的对应关系
[[06-典型Function驱动例子]]
`drivers/staging/wfx/bus_sdio.c` 是一个很好的观察点：

这一节按“为什么选这个例子 -> 初始化怎么接前文 -> 收发路径怎么落地 -> IRQ 路径怎么配合 -> 退出路径怎么闭合”展开。

这样拆的原因是：

- 前面的 `2-4` 节讲的是 API 本身
- `5-7` 节讲的是调用这些 API 时的约束条件
- 这一节则把前面这些规则真正落到一个 function driver 里

如果只贴代码而不按这条顺序解释，就容易看到很多 `sdio_*` API，却不知道它们分别处于哪一段流程。

### 8.1 为什么这个例子适合放在这一章

`wfx` 这个例子覆盖了本章最重要的几类 API：

- 初始化期的 `sdio_enable_func()`
- 初始化期的 `sdio_set_block_size()`
- 数据面的 `sdio_memcpy_fromio()` / `sdio_memcpy_toio()`
- 互斥访问的 `sdio_claim_host()` / `sdio_release_host()`
- 中断相关的 `sdio_claim_irq()` / `sdio_release_irq()`
- 长度处理的 `sdio_align_size()`

也就是说，本章前面讲到的大多数常用 API，都能在这个驱动里找到真实落点。

>[!INFO]
```C fold:"bus_sdio.c"
// SPDX-License-Identifier: GPL-2.0-only
/*
 * SDIO interface.
 *
 * Copyright (c) 2017-2020, Silicon Laboratories, Inc.
 * Copyright (c) 2010, ST-Ericsson
 */
#include <linux/module.h>
#include <linux/mmc/sdio.h>
#include <linux/mmc/sdio_func.h>
#include <linux/mmc/card.h>
#include <linux/interrupt.h>
#include <linux/of_irq.h>
#include <linux/irq.h>

#include "bus.h"
#include "wfx.h"
#include "hwio.h"
#include "main.h"
#include "bh.h"

static const struct wfx_platform_data wfx_sdio_pdata = {
	.file_fw = "wfm_wf200",
	.file_pds = "wf200.pds",
};

struct wfx_sdio_priv {
	struct sdio_func *func;
	struct wfx_dev *core;
	u8 buf_id_tx;
	u8 buf_id_rx;
	int of_irq;
};

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

static void wfx_sdio_lock(void *priv)
{
	struct wfx_sdio_priv *bus = priv;

	sdio_claim_host(bus->func);
}

static void wfx_sdio_unlock(void *priv)
{
	struct wfx_sdio_priv *bus = priv;

	sdio_release_host(bus->func);
}

static void wfx_sdio_irq_handler(struct sdio_func *func)
{
	struct wfx_sdio_priv *bus = sdio_get_drvdata(func);

	wfx_bh_request_rx(bus->core);
}

static irqreturn_t wfx_sdio_irq_handler_ext(int irq, void *priv)
{
	struct wfx_sdio_priv *bus = priv;

	sdio_claim_host(bus->func);
	wfx_bh_request_rx(bus->core);
	sdio_release_host(bus->func);
	return IRQ_HANDLED;
}

static int wfx_sdio_irq_subscribe(void *priv)
{
	struct wfx_sdio_priv *bus = priv;
	u32 flags;
	int ret;
	u8 cccr;

	if (!bus->of_irq) {
		sdio_claim_host(bus->func);
		ret = sdio_claim_irq(bus->func, wfx_sdio_irq_handler);
		sdio_release_host(bus->func);
		return ret;
	}

	flags = irq_get_trigger_type(bus->of_irq);
	if (!flags)
		flags = IRQF_TRIGGER_HIGH;
	flags |= IRQF_ONESHOT;
	ret = devm_request_threaded_irq(&bus->func->dev, bus->of_irq, NULL,
					wfx_sdio_irq_handler_ext, flags,
					"wfx", bus);
	if (ret)
		return ret;
	sdio_claim_host(bus->func);
	cccr = sdio_f0_readb(bus->func, SDIO_CCCR_IENx, NULL);
	cccr |= BIT(0);
	cccr |= BIT(bus->func->num);
	sdio_f0_writeb(bus->func, cccr, SDIO_CCCR_IENx, NULL);
	sdio_release_host(bus->func);
	return 0;
}

static int wfx_sdio_irq_unsubscribe(void *priv)
{
	struct wfx_sdio_priv *bus = priv;
	int ret;

	if (bus->of_irq)
		devm_free_irq(&bus->func->dev, bus->of_irq, bus);
	sdio_claim_host(bus->func);
	ret = sdio_release_irq(bus->func);
	sdio_release_host(bus->func);
	return ret;
}

static size_t wfx_sdio_align_size(void *priv, size_t size)
{
	struct wfx_sdio_priv *bus = priv;

	return sdio_align_size(bus->func, size);
}

static const struct hwbus_ops wfx_sdio_hwbus_ops = {
	.copy_from_io = wfx_sdio_copy_from_io,
	.copy_to_io = wfx_sdio_copy_to_io,
	.irq_subscribe = wfx_sdio_irq_subscribe,
	.irq_unsubscribe = wfx_sdio_irq_unsubscribe,
	.lock			= wfx_sdio_lock,
	.unlock			= wfx_sdio_unlock,
	.align_size		= wfx_sdio_align_size,
};

static const struct of_device_id wfx_sdio_of_match[] = {
	{ .compatible = "silabs,wfx-sdio" },
	{ .compatible = "silabs,wf200" },
	{ },
};
MODULE_DEVICE_TABLE(of, wfx_sdio_of_match);

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

static void wfx_sdio_remove(struct sdio_func *func)
{
	struct wfx_sdio_priv *bus = sdio_get_drvdata(func);

	wfx_release(bus->core);
	sdio_claim_host(func);
	sdio_disable_func(func);
	sdio_release_host(func);
}

#define SDIO_VENDOR_ID_SILABS        0x0000
#define SDIO_DEVICE_ID_SILABS_WF200  0x1000
static const struct sdio_device_id wfx_sdio_ids[] = {
	{ SDIO_DEVICE(SDIO_VENDOR_ID_SILABS, SDIO_DEVICE_ID_SILABS_WF200) },
	// FIXME: ignore VID/PID and only rely on device tree
	// { SDIO_DEVICE(SDIO_ANY_ID, SDIO_ANY_ID) },
	{ },
};
MODULE_DEVICE_TABLE(sdio, wfx_sdio_ids);

struct sdio_driver wfx_sdio_driver = {
	.name = "wfx-sdio",
	.id_table = wfx_sdio_ids,
	.probe = wfx_sdio_probe,
	.remove = wfx_sdio_remove,
	.drv = {
		.owner = THIS_MODULE,
		.of_match_table = wfx_sdio_of_match,
	}
};

```

### 8.2 这个例子和前面初始化顺序怎么对应

前面第 `6` 节给出的常见初始化顺序是：

1. `sdio_claim_host(func)`
2. `sdio_enable_func(func)`
3. `sdio_set_block_size(func, xxx)`
4. `sdio_release_host(func)`

`wfx_sdio_probe()` 里正好就是这一条主线：

- `sdio_claim_host(func)`
  - 先拿到 host 使用权
- `sdio_enable_func(func)`
  - 把 function 真正打开
- `sdio_set_block_size(func, 64)`
  - 按芯片特性把块大小改成 `64`
- `sdio_release_host(func)`
  - 初始化完成后把 host 释放回去

这正好印证了前面第 `5` 节的硬规则：这些控制面 API 不是随手就能调，而是通常要放在 `claim/release` 保护区间内。

这里还值得注意一个细节：

- 这个驱动没有接受默认块大小不管
- 而是主动把块大小设成 `64`

这和前面第 `7` 节对应：块大小直接影响 block mode/byte mode 的使用方式，也影响小包场景下的效率和稳定性。

### 8.3 收发路径怎么把数据面 API 用起来

`wfx_sdio_copy_from_io()` 和 `wfx_sdio_copy_to_io()` 对应的是本章第 `3.2` 节的批量数据 API：

- `sdio_memcpy_fromio()`
- `sdio_memcpy_toio()`

这里可以看到几个真实驱动里很常见的动作：

1. 先把逻辑寄存器号转换成 SDIO 地址
   - `unsigned int sdio_addr = reg_id << 2;`
2. 再根据 queue/buffer 模式补上地址偏移
3. 最后调用批量数据 API 完成真正搬运

这一段说明，前面讲的 `memcpy_toio/fromio` 不只是“搬一段内存”，而是通常会和：

- 芯片私有地址映射
- FIFO/queue 编号
- buffer 索引推进

一起出现。

所以真实工程里看这类 API，不能只盯函数名本身，还要同时看：

- 地址是怎么换算出来的
- 长度有没有对齐约束
- 传输完成后驱动私有状态有没有更新

### 8.4 `claim/release` 在这个例子里落在哪

前面第 `5` 节说过，host 串行访问是硬规则。这个例子里对应得很直接：

- `wfx_sdio_lock()` -> `sdio_claim_host(bus->func)`
- `wfx_sdio_unlock()` -> `sdio_release_host(bus->func)`

这说明该驱动把“host 串行访问”进一步包装成了自己的总线操作。

这样后面更上层的 `wfx` 公共逻辑就不需要每次直接写：

```c
sdio_claim_host(func);
...
sdio_release_host(func);
```

而是通过 bus 抽象把这条规则收进去。

这一点很重要，因为它说明：

- `sdio_*` API 是底层能力
- 真实驱动往往还会再包一层自己的 bus ops
- 这样上层收发逻辑就不会被 SDIO 细节淹没

### 8.5 IRQ 路径和本章数据面有什么关系

虽然 IRQ 主线会在下一章展开，但这个例子里已经能看到它和数据面的接缝：

- `wfx_sdio_irq_subscribe()`
  - 调 `sdio_claim_irq()` 或平台 IRQ 注册
- `wfx_sdio_irq_unsubscribe()`
  - 调 `sdio_release_irq()`
- `wfx_sdio_irq_handler()`
  - 中断来了以后请求上层收包

这一段放在本章里，重点不是展开 IRQ 机制本身，而是说明：

- function driver 的数据通路并不是孤立的
- 收发 API 和 IRQ 路径通常会一起构成完整的设备通信模型

也就是说：

- `sdio_memcpy_fromio()/toio()` 负责真正搬数据
- `sdio_claim_irq()` 负责在合适的时机触发“该收数据了”

这样下一章讲 IRQ 生命周期时，就能直接和这里的示例驱动接起来。

### 8.6 退出路径怎么闭合

`wfx_sdio_remove()` 和 `probe()` 形成了完整对偶：

- `probe()` 里：
  - `sdio_enable_func()`
  - `sdio_set_block_size()`
- `remove()` 里：
  - `sdio_disable_func()`

而在 `probe()` 失败回滚路径里，也能看到同样的闭合动作：

- 如果后续初始化失败
- 就重新 `claim_host`
- `sdio_disable_func(func)`
- 再 `release_host`

这一点说明，本章的控制面 API 不能只看“怎么打开”，还要同步看：

- 怎么回滚
- 怎么退出
- 哪些调用是成对出现的

### 8.7 用一条小流程把这个例子串起来

这个驱动大致可以压成下面这条线：

1. `probe()` 里 `claim_host`
2. `sdio_enable_func()`
3. `sdio_set_block_size(func, 64)`
4. 建立私有 bus ops
5. 后续收发路径里通过 `sdio_memcpy_fromio()/toio()` 搬数据
6. IRQ 来时通过 `sdio_claim_irq()` 这条线触发收包
7. `remove()` 或失败回滚时 `sdio_disable_func()`

这样回头再看本章前面各节，关系就很清楚了：

- 第 `2` 节：控制面 API
- 第 `3` 节：数据面 API
- 第 `4` 节：长度调整
- 第 `5` 节：claim/release 约束
- 第 `6` 节：初始化顺序
- 第 `7` 节：块大小的重要性
- 第 `8` 节：这些规则在真实驱动里的完整落点

## 9. 这一章最该记住的一句话

`sdio_io.c` 这一层本质上就是把 `CMD52/CMD53` 包成了 function driver 好用的 API，而 host lock 是它们正确工作的前提条件之一。

