# SDIO 总线匹配与 probe

## 导读

### 本章定位

这一章讲 `sdio_func` 设备进入 Linux driver model 之后，如何通过 `sdio bus` 匹配到具体 `sdio_driver`，并最终进入 function driver 的 `probe()`。

### 核心对象

- `struct sdio_func`
  - sdio bus 上的 device
- `struct sdio_driver`
  - sdio bus 上的 driver
- `struct sdio_device_id`
  - function 和 driver 的匹配表项
- `sdio_bus_type`
  - 负责 match/probe/remove 的总线对象

### 关键函数

- `module_sdio_driver()`
- `sdio_register_driver()`
- `sdio_unregister_driver()`
- `MODULE_DEVICE_TABLE()`
- `sdio_match_one()`
- `sdio_match_device()`
- `sdio_bus_match()`
- `sdio_bus_probe()`
- `sdio_bus_remove()`

### 主流程

function 设备注册 -> sdio bus match -> 命中 `id_table` -> `sdio_bus_probe()` 做公共准备 -> 调用具体 `sdio_driver.probe()`。

## 这一章按什么逻辑展开

这一章按“先解释总线模型，再解释匹配，再解释 probe 前的公共准备，最后落到 function driver”的逻辑展开。

这样拆的原因是：

- `probe()` 能不能进入，前面其实已经经过了 device、driver、bus 三层关系
- 如果不先把 `sdio bus` 的撮合逻辑讲清楚，后面的 `sdio_bus_probe()` 和 `driver.probe()` 容易看成一层

所以本章后面的结构是：

1. 先把 device 注册链和 driver 注册链放到同一张图里
2. 再看 `sdio_driver` 怎么注册
3. 再看 `sdio bus` 是什么
4. 然后看 match 怎样命中 `id_table`
5. 再用一个最小注册模板把 `id_table`、`MODULE_DEVICE_TABLE()`、`probe/remove` 和 `module_sdio_driver()` 串起来
6. 最后看 `sdio_bus_probe()` 做完公共准备后，如何进入真正的 function driver `probe()`

## 0. 入口从哪里来：device 注册和 driver 注册在哪里汇合

`sdio_driver.probe()` 不是驱动自己直接调用的，它是 Linux driver model 在 `sdio bus` 上把 device 和 driver 撮合成功后回调的。

这里要先分清两条入口链。

第一条是枚举侧，也就是上一章创建出来的 function device：

```text
mmc_sdio_init_card()                 // 整卡初始化完成后，已经知道 function 数量
-> sdio_init_func()                  // 为某个 function 创建 struct sdio_func
-> mmc_add_card()                    // 先把整张 card 注册到设备模型
-> sdio_add_func()                   // 再把单个 function 注册成 sdio bus device
-> device_add(&func->dev)            // driver model 开始为这个 device 找 driver
-> sdio_bus_match()                  // sdio bus 对比 func 身份和 driver id_table
```

第二条是驱动侧，也就是模块加载或内核内建驱动注册：

```text
module_sdio_driver(xxx_driver)       // function driver 的模块入口宏
-> sdio_register_driver()            // 把 struct sdio_driver 注册到 sdio bus
-> driver_register(&drv->drv)        // driver model 开始为这个 driver 找 device
-> sdio_bus_match()                  // 同样走 sdio bus 的 match 函数
```

所以不管是“设备先出现，驱动后加载”，还是“驱动先注册，设备后枚举”，最终都会汇合到同一个位置：

```text
sdio_func device                     // 枚举阶段创建出的真实 function 设备
+ sdio_driver                        // 驱动声明自己能支持哪些 function
-> sdio_bus_match()                  // sdio bus 负责判断两者是否匹配
-> sdio_bus_probe()                  // 匹配成功后进入 bus 层 probe 包装
-> drv->probe(func, id)              // 最后才进入具体 function driver
```

## 1. `sdio_driver` 是怎么注册进去的

入口定义在：

- `include/linux/mmc/sdio_func.h`

常见注册方式有两种：

- `sdio_register_driver(&drv)`
- `module_sdio_driver(drv)`

实现落在：

- `drivers/mmc/core/sdio_bus.c`

核心函数：

- `sdio_register_driver()`
- `sdio_unregister_driver()`

### 1.1 `module_sdio_driver()` 到底省掉了什么

很多入门文章会直接给出一个 `module_sdio_driver(xxx_driver)` 模板。它不是新的 SDIO 匹配机制，而是把普通模块入口压缩成一个宏。

源码里可以看到它最终还是落到：

```text
module_sdio_driver(xxx_driver)      // SDIO function driver 最常见的模块入口宏
-> module_driver()                  // 通用模块注册/注销辅助宏
-> sdio_register_driver()           // 模块加载时把 sdio_driver 注册到 sdio bus
-> driver_register(&drv->drv)       // 进入 Linux driver model 的 driver 注册入口
```

模块卸载时则是反向路径：

```text
module_sdio_driver(xxx_driver)      // 同一个宏也生成模块退出路径
-> sdio_unregister_driver()         // 模块卸载时从 sdio bus 注销 driver
-> driver_unregister(&drv->drv)     // 从 Linux driver model 移除这个 driver
```

所以学习时不要把 `module_sdio_driver()` 看成“probe 入口”。它只是注册入口，真正决定 `probe()` 能不能进入的仍然是：

```text
sdio_driver 注册成功                // driver 侧已经告诉 sdio bus 自己存在
+ sdio_func 设备已经注册             // device 侧已经由枚举流程创建出来
+ id_table 匹配成功                  // bus 能把 device 和 driver 撮合起来
-> sdio_bus_probe()                 // 匹配成功后进入 bus 层 probe 包装
-> drv->probe(func, id)             // 最后才回调具体 function driver 的 probe
```

## 2. `sdio bus` 的本质

`sdio_bus.c` 注册了一个 `bus_type`：

- 名字就是 `sdio`

它负责：

- 匹配
- uevent
- probe
- remove

所以从驱动模型视角看：

- `sdio_func` 是 device
- `sdio_driver` 是 driver
- `sdio bus` 负责把两者撮合起来

```c
static struct bus_type sdio_bus_type = {
	.name		= "sdio",
	.dev_groups	= sdio_dev_groups,
	.match		= sdio_bus_match,
	.uevent		= sdio_bus_uevent,
	.probe		= sdio_bus_probe,
	.remove		= sdio_bus_remove,
	.pm		= &sdio_bus_pm_ops,
};
```
## 3. 匹配逻辑很直接

关键函数：

- `sdio_match_one()`
- `sdio_match_device()`
- `sdio_bus_match()`

匹配条件来自 `struct sdio_device_id`：


- `class`
- `vendor`
- `device`

只要 `id_table` 中有一项和当前 `sdio_func` 对上，`sdio_bus_match()` 就返回成功。

这里要把“设备侧字段”和“驱动侧匹配表字段”分开看：

| 对象 | 字段来源 | 后续被谁消费 |
| --- | --- | --- |
| `func->class` | function 级 CIS 解析得到，表示这个 function 的标准功能类别 | 被 `sdio_match_one()` 拿来和 `id->class` 对比 |
| `func->vendor` / `func->device` | common CIS 或 function CIS 解析得到，表示卡/function 报告的厂商和设备身份 | 被 `sdio_match_one()` 拿来和 `id->vendor` / `id->device` 对比 |
| `sdio_device_id.class/vendor/device` | function driver 在 `id_table` 里声明，表示驱动愿意匹配哪些设备 | 被 `sdio_bus_match()` 间接消费，决定 probe 能不能进入 |
| `SDIO_ANY_ID` | driver 侧通配宏，表示这一项不限制对应字段 | 让驱动可以按 class 匹配一类设备，或按 vendor/device 精确匹配 |
| `driver_data` | driver 侧自定义匹配附加值 | 匹配成功后跟随 `id` 传给 `probe(func, id)`，常用于区分芯片版本或能力表 |
| `id` 参数 | `sdio_match_device()` 返回的命中表项 | `sdio_bus_probe()` 最终传给具体 `drv->probe(func, id)` |

因此，`sdio_device_id` 不是“被 probe 的设备”，而是 driver 写给 bus 的匹配声明；真正被 probe 的对象仍然是 `struct sdio_func`。

```c
static int sdio_bus_match(struct device *dev, struct device_driver *drv)
{
	struct sdio_func *func = dev_to_sdio_func(dev);
	struct sdio_driver *sdrv = to_sdio_driver(drv);

	if (sdio_match_device(func, sdrv))
		return 1;

	return 0;
}



static const struct sdio_device_id *sdio_match_device(struct sdio_func *func,
	struct sdio_driver *sdrv)
{
	const struct sdio_device_id *ids;

	ids = sdrv->id_table;

	if (ids) {
		while (ids->class || ids->vendor || ids->device) {
			if (sdio_match_one(func, ids))
				return ids;
			ids++;
		}
	}

	return NULL;
}
```
## 4. `sdio_bus_probe()` 做了哪些通用准备

这是所有 function driver 的真正前置入口。

关键动作如下：

1. 再次确认 `id_table` 匹配成功
2. `dev_pm_domain_attach()`
3. 增加 `sdio_funcs_probed`
4. 如果支持 runtime PM，就先把设备拉到 active
5. `sdio_claim_host()`
6. `sdio_set_block_size(func, 0)` 设置默认块大小
7. `sdio_release_host()`
8. 调用具体驱动的 `drv->probe(func, id)`

这里有一个很容易忽略的点：

- function driver 进入 `probe()` 之前，core 已经设置过一次默认 block size

但很多驱动还是会在自己的 `probe()` 里重新设置更适合芯片的数据块大小，这并不冲突。

### 4.1 这次默认 block size 真正是在哪里设进去的

调用点在：

- `drivers/mmc/core/sdio_bus.c`
- `sdio_bus_probe()`

这里做的是：

- `sdio_claim_host()`
- `sdio_set_block_size(func, 0)`
- `sdio_release_host()`

其中 `0` 不是“块大小为 0”，而是“让 core 选择默认值”。

真正计算默认值的地方在：

- `drivers/mmc/core/sdio_io.c`
- `sdio_set_block_size()`

这一层的逻辑是：

1. 如果 `blksz != 0`，就直接按调用者给的值设置
2. 如果 `blksz == 0`，就走 core 的默认策略
3. 默认值取下面三者中的最小值
   - `func->max_blksize`
   - `func->card->host->max_blk_size`
   - `512`
4. 写 FBR 里的 block size 寄存器
5. 把结果保存到 `func->cur_blksize`

所以这里要区分两个字段：

- `max_blksize`
  - 表示这一个 function 理论上支持的最大块大小
- `cur_blksize`
  - 表示当前真正生效、后面 CMD53 等数据传输实际使用的块大小

也就是说：

- `sdio_bus_probe()` 负责在 function driver 进入 `probe()` 之前触发一次“默认块大小设置”
- `sdio_set_block_size()` 负责真正算出默认值并写入寄存器

这也是很多驱动后面还会再次调用 `sdio_set_block_size(func, xxx)` 的原因：

- core 先给一个保底且通用的默认值
- 具体 function driver 再按芯片吞吐、协议帧长度或固件要求改成更合适的值

## 5. `sdio_bus_remove()` 做了什么

退出路径同样统一在 `sdio_bus.c` 里。

主要步骤：

1. 确保卡是上电状态
2. 调具体驱动的 `remove(func)`
3. 减少 `sdio_funcs_probed`
4. 检查驱动是否忘记释放 IRQ
5. 回收 runtime PM 状态

这里那个告警很有价值：

- 如果驱动 `remove()` 结束时 `func->irq_handler` 还在，core 会直接报警

这通常意味着：

- 驱动漏掉了 `sdio_release_irq()`

## 6. 最小 SDIO driver 注册模板

```c
static const struct sdio_device_id xxx_ids[] = {
	{ SDIO_DEVICE(vendor, device) },
	{ }
};
MODULE_DEVICE_TABLE(sdio, xxx_ids);

static int xxx_probe(struct sdio_func *func,
		     const struct sdio_device_id *id)
{
	...
}

static void xxx_remove(struct sdio_func *func)
{
	...
}

static struct sdio_driver xxx_driver = {
	.name = "xxx",
	.id_table = xxx_ids,
	.probe = xxx_probe,
	.remove = xxx_remove,
};

module_sdio_driver(xxx_driver);
```

这个模板适合作为入门文章里的最小 SDIO function driver 骨架来看，但要注意它只解决“驱动如何注册、如何被匹配、匹配后从哪里进入”这几个问题，还没有包含真实芯片初始化、I/O、IRQ 和上层子系统接入。

### 6.1 这几个成员分别对应哪一层

- `xxx_ids`
  - 给 `sdio_bus_match()` 使用
  - 负责回答“当前这个 `sdio_func` 是否应该交给这个驱动”
- `MODULE_DEVICE_TABLE(sdio, xxx_ids)`
  - 把 SDIO 匹配表导出给模块设备表
  - 负责让用户空间模块自动加载机制知道这个模块支持哪些 SDIO 设备
- `xxx_probe()`
  - 给 `sdio_bus_probe()` 在公共准备之后回调
  - 负责进入芯片私有初始化
- `xxx_remove()`
  - 给 `sdio_bus_remove()` 回调
  - 负责 function driver 的退出清理
- `xxx_driver`
  - 是这整个 function driver 注册给 `sdio bus` 的总入口对象
- `module_sdio_driver(xxx_driver)`
  - 生成模块加载/卸载入口
  - 加载时调用 `sdio_register_driver()`，卸载时调用 `sdio_unregister_driver()`

所以从驱动模型视角看：

- `module_sdio_driver()` 决定这个 driver 怎么注册进内核
- `id_table` 决定这个 driver 能不能匹配当前 `sdio_func`
- `MODULE_DEVICE_TABLE()` 决定模块自动加载能不能根据 SDIO modalias 找到这个模块
- `probe` 决定匹配成功后做什么
- `remove` 决定退出时怎么收尾

### 6.2 这套模板的完整触发关系

把模板里的几个点按运行顺序串起来，就是下面这条线：

```text
module_sdio_driver(xxx_driver)      // 模块入口宏，替代手写 module_init/module_exit
-> sdio_register_driver()           // 模块加载时把 xxx_driver 注册到 sdio bus
-> driver_register()                // Linux driver model 开始为这个 driver 找 device
-> xxx_ids / id_table               // sdio bus 用这张表判断支持哪些 function
-> MODULE_DEVICE_TABLE(sdio, ...)   // 导出模块匹配信息，服务自动加载
-> sdio_bus_match()                 // 对比 func 的 class/vendor/device 和 id_table
-> sdio_bus_probe()                 // 匹配成功后做 SDIO bus 级公共准备
-> xxx_probe(func, id)              // 进入具体 function driver 的私有初始化
-> xxx_remove(func)                 // 设备解绑或模块卸载时做反向清理
```

这条链里最容易混淆的是：

- `module_sdio_driver()` 是 driver 注册入口，不是设备 probe 入口
- `MODULE_DEVICE_TABLE()` 是模块匹配信息导出，不参与内核里的逐项比较逻辑
- `id_table` 才是 `sdio_bus_match()` 真正遍历的匹配表
- `probe(func, id)` 里的 `id` 是命中的表项，`func` 才是真正被绑定的设备对象

### 6.3 `probe()` 真正开始之前，core 已经做完了什么

这一点和上面的模板要连起来看。

当 `drv->probe(func, id)` 被调用时，前面的 `sdio core` 已经完成了这些准备：

1. `sdio_func` 设备已经创建并注册
2. `sdio_bus_match()` 已经根据 `id_table` 找到命中的驱动
3. `sdio_bus_probe()` 已经再次确认匹配成功
4. runtime PM 相关的公共准备已经做过一次
5. core 已经先设置过一次默认 `block size`

所以 function driver 的 `probe()` 不是从“空白卡状态”开始，而是从“已经被 core 枚举、已经进入 driver model、已经做过一轮公共准备”的状态开始。

这也是为什么前一章最后停在：

- `sdio_add_func()`

而这一章的真正起点是：

- `sdio_bus_probe()`
- `drv->probe(func, id)`

### 6.4 真正复杂的部分在 `probe()` 内部顺序

代码壳子本身很简单，真正容易出错的是 `probe()` 里的动作顺序。

常见 function driver 在 `probe()` 里会继续处理：

- host claim/release
- `sdio_enable_func()`
- `sdio_set_block_size()`
- `sdio_claim_irq()`
- I/O 寄存器访问

这些调用顺序之所以重要，是因为：

- `sdio_enable_func()` 决定 function 是否真的进入工作状态
- `sdio_set_block_size()` 决定后面 CMD53 的传输粒度
- `sdio_claim_irq()` 决定中断路径是否建立
- 寄存器访问通常依赖前面 enable 和 block size 已经正确设置

这部分 `probe()` 内部的 function 级操作，在 [[04-SDIO数据通路与常用API]] 章节继续展开。

### 6.5 `remove()` 为什么也要和这段骨架一起看

`probe()` 建起来的东西，后面基本都要由 `remove()` 对应收回。

至少要对应：

- 释放 IRQ
- 停用 function
- 回收驱动私有数据

前面第 `5` 节已经提到：

- 如果 `remove()` 结束时 `func->irq_handler` 还在，core 会报警

所以这段最小骨架虽然看起来简单，但实际上已经把：

- 匹配入口
- 私有初始化入口
- 退出清理入口

三条主线固定下来了。

## 7. 这一层和上一层的边界

边界可以画得很清楚：

- `sdio.c`：把卡和 function 枚举出来
- `sdio_bus.c`：把 function 交给驱动模型
- 具体 `sdio_driver`：开始进入芯片私有逻辑

## 8. 源码里最值得重点盯的四行

看一眼这四个点，整体就通了：

- `sdio_register_driver()`
- `sdio_bus_match()`
- `sdio_bus_probe()`
- `drv->probe(func, id)`

顺着这四个点往下走，就能找到一个 SDIO function 驱动是怎么被拉起来的。

## 9. 全流程总结：从 function device 到 driver probe

这一章跨过了 device 创建、driver 注册、bus match、bus probe 和具体 driver probe 几个阶段，最后可以压成下面这条闭环。

```text
mmc_sdio_init_card()                 // 枚举阶段已经完成整卡初始化，知道有几个 SDIO function
-> sdio_init_func()                  // 为每个 function 创建 struct sdio_func
-> sdio_add_func()                   // 把 function 注册到 sdio bus
-> device_add(&func->dev)            // 设备模型开始寻找可匹配的 driver
-> module_sdio_driver()              // function driver 模块侧生成注册/注销入口
-> sdio_register_driver()            // function driver 另一侧注册到 sdio bus
-> MODULE_DEVICE_TABLE(sdio, ids)     // 导出模块匹配信息，支持自动加载
-> sdio_bus_match()                  // bus 对比 func 身份字段和 driver id_table
-> sdio_match_device()               // 遍历 driver 声明的 sdio_device_id 表
-> sdio_match_one()                  // 逐项比较 class/vendor/device，支持 SDIO_ANY_ID 通配
-> sdio_bus_probe()                  // bus 层做 PM、默认 block size 等公共准备
-> drv->probe(func, id)              // 进入具体 function driver，开始芯片私有初始化
```

读这条链时，最重要的是别把 `sdio_add_func()` 和 `sdio_bus_probe()` 看成同一件事。

- `sdio_add_func()` 负责把 function 设备交给 driver model
- `sdio_bus_match()` 负责判断哪个 driver 能绑定这个 function
- `sdio_bus_probe()` 是匹配成功后的 bus 层包装入口
- `drv->probe(func, id)` 才是具体芯片驱动真正开始工作的地方
