# `subdev` 与异步注册

## 导读

### 本章定位

这一章是 `09-典型subdev驱动例子-imx219` 的前置知识章，先把 `subdev` 自身讲完整，再进入具体 sensor 驱动。  
前一章的 `entity/pad/link` 负责建图，本章继续回答四个问题：

- `subdev` 作为对象本身包含什么
- 它什么时候只是管线内部模块，什么时候又会变成 `/dev/v4l-subdevX`
- 用户态访问 `v4l-subdevX` 时，`open/ioctl` 怎样进入 `subdev` 回调
- `async notifier` 又怎样把外部 sensor/bridge 挂进 host 管理

这样到 `09` 章再看 `imx219` 时，就可以把注意力放在“这个 sensor 怎样填写这些框架接口”，而不是一边读示例驱动一边补框架地基。

### 核心对象

- `struct v4l2_subdev`
  - 管线内部功能模块对象
- `struct v4l2_subdev_ops`
  - `subdev` 对外业务回调表，按 `core/video/pad/...` 分组
- `struct v4l2_subdev_internal_ops`
  - `subdev` 节点注册、打开、关闭等内部生命周期回调表
- `struct v4l2_subdev_fh`
  - `/dev/v4l-subdevX` 每个文件句柄对应的上下文对象
- `struct video_device`
  - 当 `subdev` 暴露成 `/dev/v4l-subdevX` 时，外层仍由它承载字符设备节点
- `struct v4l2_ctrl_handler`
  - 一组 V4L2 controls 的管理对象，`subdev` 和后续 devnode 都会通过它找到控制项
- `struct v4l2_ctrl`
  - 单个控制项，例如曝光、增益、翻转
- `struct v4l2_ctrl_ops`
  - control 值变化后回调到驱动的操作表
- `struct v4l2_async_subdev`
  - async 框架里的匹配描述对象
- `struct v4l2_async_notifier`
  - host 侧等待并绑定外部 subdev 的对象
- `sd.entity / media_pad`
  - `subdev` 进入 Media Controller 图模型时依赖的对象

### 关键函数

- `v4l2_subdev_init()`
- `v4l2_i2c_subdev_init()`
- `__v4l2_device_register_subdev_nodes()`
- `subdev_open()`
- `subdev_ioctl()`
- `subdev_do_ioctl()`
- `v4l2_subdev_call()`
- `v4l2_async_register_subdev()`
- `v4l2_async_register_subdev_sensor_common()`
- `v4l2_device_register_subdev()`

### 主流程

注册期主线：初始化 `subdev` -> 初始化 `entity/pad` -> async 注册 -> host notifier 等待 -> 匹配成功 -> 挂到 `v4l2_dev->subdevs` -> 可选暴露 `/dev/v4l-subdevX`

使用期主线：`open("/dev/v4l-subdevX")` -> `subdev_open()` -> `ioctl()` -> `subdev_ioctl()` -> `subdev_do_ioctl()` -> `pad/video/core/control` 回调

## 1. 为什么 V4L2 要有 `subdev`

如果一个媒体设备只有一个简单 `/dev/videoX` 节点，那只靠 `video_device` 就够了。  
但真实相机链路往往是：

- sensor
- MIPI CSI-2 接收器
- ISP
- scaler
- capture DMA

这些模块经常不是一个驱动就能包完，所以 V4L2 引入了 `subdev` 模型。
#subdev

它的目标是：

- 把“媒体管线内部零件”抽象出来
- 让 host 驱动和 sensor/bridge 驱动解耦
- 允许异步绑定，而不是强依赖 probe 顺序

前面的 `01-04` 和 [[05-典型video节点驱动例子-sh_vou]]，主要是在单节点 `video_device` 语境下讲主线。  
进入这一章以后，开始正式切到：

- Media Controller
- `subdev`
- 异步绑定

这一侧的管线型模型。  
单节点主线可回看 [[02-video_device注册与open链路#2. 驱动侧通常怎么走]]。  
本章放在 [[06-Media-Controller框架总览]] 和 [[07-entity-pad-link-pipeline主线]] 之后。
因为 `subdev` 放到复杂管线里理解时，通常要先知道：

- `media_device`
- `media_entity`
- `media_pad`
- `media_link`

这些对象在 [[07-entity-pad-link-pipeline主线]] 里介绍。

### 1.1 本章核心对象

先把这一章真正依赖的三个对象分开：

- `struct v4l2_subdev`
  - 真正的模块对象
  - 关键看 `ops / internal_ops / flags / name / v4l2_dev / host_priv / entity`
- `struct v4l2_async_subdev`
  - async 匹配描述对象
  - 关键看“匹配谁”的那组 match 信息，例如 fwnode 匹配键
- `struct v4l2_async_notifier`
  - host 侧管理等待列表和完成回调的对象
  - 关键看“等谁”“匹配成功后做什么”

这一章和上一章的联动点在于：

- `v4l2_subdev`
  - 解决模块对象和 host 归属关系
- `sd.entity / pad`
  - 解决这个模块进入媒体图以后怎么表示和怎么连线

### 1.2 三个对象的职责分工

这一章最容易混的地方，不是函数调用本身，而是三个对象各自站在哪一层：

- `v4l2_subdev`
  - 真正的模块对象
  - 代表 sensor / bridge / CSI / ISP 这类管线内部模块
  - 最终会挂到某个 `v4l2_device->subdevs`
- `v4l2_async_subdev`
  - 匹配描述对象
  - 不代表一个真正工作的模块，而是代表“host 现在等的是谁”
  - 它通常挂在某个 notifier 下面，保存 fwnode / I2C / devname 这类匹配键
- `v4l2_async_notifier`
  - host 侧匹配管理对象
  - 负责维护等待项、调用 `bound()`、在全部匹配完成后调用 `complete()`

可以先把三者关系压成下面这条线：

```text
host 先准备 notifier
-> notifier 里挂多个 async_subdev 匹配描述
-> subdev 驱动把真正的 sd 注册进 async 框架
-> 框架拿 sd 去和 notifier 里的 async_subdev 做匹配
-> 匹配成功后再把 sd 挂进 host 的 v4l2_dev->subdevs
```

### 1.3 `v4l2_subdev` 在异步路径里重点看哪些成员

这一章不需要把 `struct v4l2_subdev` 所有字段再展开一遍，但异步路径里有几项需要特别记住：

- `ops`
  - 模块本身的操作集
- `dev`
  - 对应的底层设备，常见就是 `&client->dev`
- `fwnode`
  - async 匹配时常用的设备描述入口
- `v4l2_dev`
  - 当前归属的 host，初始化时为空，匹配成功后才指向目标 `v4l2_device`
- `list`
  - 挂到 `v4l2_dev->subdevs` 的链表节点
- `async_list`
  - async 框架内部等待和流转用的链表节点
- `asd`
  - 当前匹配到的那个 `v4l2_async_subdev`
- `subdev_notifier`
  - 当本身还要继续管理下级 subdev 时使用
- `entity`
  - 进入 Media Controller 图模型时对应的 entity

这里可以先把两条链表关系分开：

- `list`
  - 说明这个 `subdev` 已经正式归某个 host 管理
- `async_list`
  - 说明这个 `subdev` 还在 async 框架里等待匹配或流转

### 1.4 `v4l2_async_subdev` 在异步路径里重点看哪些成员

`v4l2_async_subdev` 的定位不是“模块对象”，而是“匹配模板”。

这一章理解它，重点只需要抓住三类信息：

- 匹配类型
  - 例如按 fwnode、I2C、devname 这类方式匹配
- 匹配键
  - 例如 `match.fwnode`
  - 这决定 notifier 在等谁
- 链表归属
  - 它本身挂在 notifier 的等待集合里
  - 匹配成功后，`sd->asd` 会反向指到它

所以 `async_subdev` 回答的问题是：

- host 现在在等哪一个外部模块
- 用什么键去认这个模块

而不回答：

- 这个模块本身的寄存器、格式、streaming 怎么做

那些仍然属于 `v4l2_subdev` 自己。

### 1.5 `v4l2_async_notifier` 在异步路径里重点看哪些成员

`v4l2_async_notifier` 是 host 侧真正的“协调者”。

这一章理解它，重点抓住下面几项就够用：

- 等待集合
  - host 当前期待绑定的 `async_subdev` 集合
- `bound()`
  - 某个 `subdev` 匹配成功后立即回调
- `complete()`
  - 当前 notifier 下的等待项全部匹配完成后回调
- `unbind()`
  - 解绑和清理时回调
- 归属关系
  - 它最终会挂到某个 `v4l2_device`

所以 notifier 回答的问题是：

- host 在等谁
- 谁先到达时先记下来
- 全部到齐后什么时候进入建图阶段

## 2. `v4l2_subdev_init()`

源码：

- `drivers/media/v4l2-core/v4l2-subdev.c:868`

>[!INFO]
```C fold:"v4l2_subdev_init"
void v4l2_subdev_init(struct v4l2_subdev *sd, const struct v4l2_subdev_ops *ops)
{
	INIT_LIST_HEAD(&sd->list);
	BUG_ON(!ops);
	sd->ops = ops;
	sd->v4l2_dev = NULL;
	sd->flags = 0;
	sd->name[0] = '\0';
	sd->grp_id = 0;
	sd->dev_priv = NULL;
	sd->host_priv = NULL;
#if defined(CONFIG_MEDIA_CONTROLLER)
	sd->entity.name = sd->name;
	sd->entity.obj_type = MEDIA_ENTITY_TYPE_V4L2_SUBDEV;
	sd->entity.function = MEDIA_ENT_F_V4L2_SUBDEV_UNKNOWN;
#endif
}
```

它做的事情很朴素：

- 初始化链表头`struct list_head list`，
- 绑定 `ops`
- 清空 `v4l2_dev`，- 表示该 subdev 当前尚未归属任何 host，但`v4l2_dev`本身是还在的
- 清 flags/name/priv
- 初始化 media entity 的默认属性

可以把它理解成：

**把一个普通结构体，初始化成 V4L2 subdev 对象。**
#subdev
[[01-V4L2核心对象与驱动模型#4. `struct v4l2_subdev`]]
#结构体填充流程 
[[v4l2驱动总结#驱动初始化与退出函数]]
## 3. `v4l2_i2c_subdev_init()` 的角色

很多 sensor 是 I2C 设备，所以驱动 probe 时常见这句：

- `drivers/media/i2c/imx219.c:1394`
  `v4l2_i2c_subdev_init(&imx219->sd, client, &imx219_subdev_ops)`

```c
void v4l2_i2c_subdev_init(struct v4l2_subdev *sd, struct i2c_client *client,
			  const struct v4l2_subdev_ops *ops)
{
	v4l2_subdev_init(sd, ops);
	sd->flags |= V4L2_SUBDEV_FL_IS_I2C;
	/* the owner is the same as the i2c_client's driver owner */
	sd->owner = client->dev.driver->owner;
	sd->dev = &client->dev;
	/* i2c_client and v4l2_subdev point to one another */
	v4l2_set_subdevdata(sd, client);
	i2c_set_clientdata(client, sd);
	v4l2_i2c_subdev_set_name(sd, client, NULL, NULL);
}
```
这一步一般会同时做：

- `v4l2_subdev_init()`
- 绑定 `sd->dev`
- 绑定 `i2c_client`
- 设置名字和 owner

所以对 sensor 驱动来说，它通常是最常见的 subdev 初始化入口。

## 4. `struct v4l2_subdev_ops`

定义位置：

- `include/media/v4l2-subdev.h:749`
>[!INFOG]
```c
struct v4l2_subdev_ops {
	const struct v4l2_subdev_core_ops	*core;
	const struct v4l2_subdev_tuner_ops	*tuner;
	const struct v4l2_subdev_audio_ops	*audio;
	const struct v4l2_subdev_video_ops	*video;
	const struct v4l2_subdev_vbi_ops	*vbi;
	const struct v4l2_subdev_ir_ops		*ir;
	const struct v4l2_subdev_sensor_ops	*sensor;
	const struct v4l2_subdev_pad_ops	*pad;
};
```

这是 `subdev` 的回调表，总体按功能分组：

- `core`
- `tuner`
- `audio`
- `video`
- `vbi`
- `ir`
- `sensor`
- `pad`

对 camera sensor 来说，最常用的是：

- `core`
- `video`
- `pad`

比如 `imx219.c:1211`：

```c
static const struct v4l2_subdev_ops imx219_subdev_ops = {
    .core  = &imx219_core_ops,
    .video = &imx219_video_ops,
    .pad   = &imx219_pad_ops,
};
```

## 5. 为什么 sensor 驱动特别重视 `pad ops`

因为 sensor 往往不直接面向应用层 `VIDIOC_S_FMT`，而是通过媒体管线协商：

- 总线格式 `mbus code`
- 帧尺寸
- crop / selection
- sink/source pad format

所以 `pad ops` 往往比传统 `video ioctl ops` 更重要。

`imx219_pad_ops`：

- `drivers/media/i2c/imx219.c:1203` 左右
-
```c
static const struct v4l2_subdev_pad_ops imx219_pad_ops = {
	.enum_mbus_code = imx219_enum_mbus_code,
	.get_fmt = imx219_get_pad_format,
	.set_fmt = imx219_set_pad_format,
	.get_selection = imx219_get_selection,
	.enum_frame_size = imx219_enum_frame_size,
};
```

关键成员：

- `enum_mbus_code`
- `get_fmt`
- `set_fmt`
- `get_selection`
- `enum_frame_size`

## 6. `media_entity_pads_init()` 的意义

对于参与媒体拓扑的 subdev，通常会看到：

```c
media_entity_pads_init(&sd->entity, npads, pads);
```
比如 `imx219.c:1471` 附近。

>[!INFO]
```c fold："media_entity_pads_init"
int media_entity_pads_init(struct media_entity *entity, u16 num_pads,
			   struct media_pad *pads)
{
	struct media_device *mdev = entity->graph_obj.mdev;
	unsigned int i;

	if (num_pads >= MEDIA_ENTITY_MAX_PADS)
		return -E2BIG;

	entity->num_pads = num_pads;
	entity->pads = pads;

	if (mdev)
		mutex_lock(&mdev->graph_mutex);

	for (i = 0; i < num_pads; i++) {
		pads[i].entity = entity;
		pads[i].index = i;
		if (mdev)
			media_gobj_create(mdev, MEDIA_GRAPH_PAD,
					&entity->pads[i].graph_obj);
	}

	if (mdev)
		mutex_unlock(&mdev->graph_mutex);

	return 0;
}
```

它的作用是：

- 告诉 media framework 这个 subdev 有几个 pad
- 每个 pad 是 `SOURCE` 还是 `SINK`
- 后续媒体链路如何连接

如果没有 pad/entity，很多媒体管线信息就没法建立起来。

## 7. 先补一个后面会反复出现的对象：`ctrl_handler`

`ctrl` 可以先理解成“用户或框架可调的一个参数”，例如：

- 曝光
- 模拟增益
- 数字增益
- 水平翻转
- 垂直翻转

这些参数不能散落在驱动里各管各的，所以 V4L2 control 框架会再提供一个上层容器：

- `struct v4l2_ctrl_handler`

源码注释对它的定义很直接：

- `include/media/v4l2-ctrls.h:331`
  - 它负责跟踪一组 controls，既包括自己拥有的，也包括从其他 handler 继承进来的

### 7.1 三个 control 对象先分开

| 对象 | 最小理解 | 解决的问题 |
| --- | --- | --- |
| `struct v4l2_ctrl_handler` | 一组 controls 的管理器 | 这些参数归谁统一管理、怎样被查找 |
| `struct v4l2_ctrl` | 一个具体控制项 | 这个参数本身是什么、当前值和范围是什么 |
| `struct v4l2_ctrl_ops` | control 的回调表 | 参数变化后谁真正执行 |

可以先压成这条关系：

```text
driver private struct
-> ctrl_handler                         // 管一组 control
   -> exposure / gain / flip / ...      // 每个具体 control
      -> ctrl_ops.s_ctrl                // 参数变化后回调到驱动
```

### 7.2 `ctrl_handler` 里大致管什么

当前阶段不需要把 `struct v4l2_ctrl_handler` 的所有字段都背下来，但至少要知道它管理四类东西：

| 成员类别 | 代表字段 | 作用 |
| --- | --- | --- |
| 自己拥有的 controls | `ctrls` | 保存本 handler 创建出的控制项 |
| 可查询的 control 引用 | `ctrl_refs`、`buckets`、`cached` | 让框架按 control id 快速找到目标 control，也支持合并别的 handler 的 controls |
| 并发保护 | `lock` | 设置 control 时保护这一组参数 |
| 初始化错误状态 | `error` | 任一 control 创建失败时先记住错误，便于 probe 统一检查 |

因此它不是“某一个曝光值”，而更像“这台设备的一整组参数管理入口”。

### 7.3 为什么 `subdev` 里要挂 `sd.ctrl_handler`

因为 `subdev` 也可能直接承载 controls。  
sensor 驱动通常会在 probe 阶段：

```text
v4l2_ctrl_handler_init()                 // 先建立 handler
-> v4l2_ctrl_new_std()                   // 往 handler 里注册曝光、增益等 ctrl
-> sd.ctrl_handler = &driver_ctrl_hdlr  // 把这组 controls 挂到 subdev
```

以后如果这个 `subdev` 被暴露成 `/dev/v4l-subdevX`：

- `__v4l2_device_register_subdev_nodes()`
  - 会让外层 `video_device` 继承 `sd->ctrl_handler`
- `v4l2_fh_init()`
  - 再让文件句柄继承 `vdev->ctrl_handler`
- `subdev_do_ioctl()`
  - 处理 `VIDIOC_S_CTRL` 时通过 `vfh->ctrl_handler` 找到具体 control

所以后面看到：

```text
vdev->ctrl_handler = sd->ctrl_handler
vfh->ctrl_handler
v4l2_s_ctrl(vfh, vfh->ctrl_handler, ...)
```

时，它们不是三个不同的参数系统，而是同一组 controls 沿着：

```text
subdev -> devnode -> file handle
```

一路传下来的结果。

### 7.4 和 `09` 章怎么分工

本章只先回答“`ctrl_handler` 是什么，为什么 `subdev` 需要它”。  
到了 `09` 章，再用 `imx219` 具体展开：

- 它怎样创建曝光、增益、翻转这些 control
- `V4L2_CID_EXPOSURE` 怎样落到真实寄存器
- `VIDIOC_S_CTRL` 怎样从用户态一路回调到 `imx219_set_ctrl()`

## 8. `subdev` 什么时候只是内部对象，什么时候会变成 `/dev/v4l-subdevX`

`subdev` 的本体始终是 `struct v4l2_subdev`，它先解决的是“管线内部模块”这个问题。  
是否额外暴露成 `/dev/v4l-subdevX`，取决于后续是否满足节点创建条件，而不是 `subdev` 一初始化就天然有用户态节点。

### 8.1 `V4L2_SUBDEV_FL_HAS_DEVNODE` 只是声明“需要节点”

以 `imx219` 为例，probe 里会设置：

```c
imx219->sd.flags |= V4L2_SUBDEV_FL_HAS_DEVNODE;
```

这一步只表达：

- 这个 `subdev` 希望后续拥有自己的 subdev devnode
- 允许用户态直接通过 `/dev/v4l-subdevX` 操作它

但它本身还不会立刻创建字符设备节点。  
真正创建节点的是 host 侧后续调用：

- `drivers/media/v4l2-core/v4l2-device.c:189`
  `__v4l2_device_register_subdev_nodes()`

### 8.2 节点暴露时，外面仍然包了一层 `video_device`

源码已经明确说明，`v4l-subdevX` 不是绕开 `video_device` 单独注册的另一套字符设备。  
它的创建路径是：

```text
sd.flags |= V4L2_SUBDEV_FL_HAS_DEVNODE                  // subdev 声明希望暴露 devnode
-> v4l2_device_register_subdev(v4l2_dev, sd)            // 先把真实 sd 挂到 host 管理下
-> __v4l2_device_register_subdev_nodes(v4l2_dev, ...)   // host 主动为 eligible subdev 建节点
-> kzalloc(struct video_device)                         // 为这个 subdev 包一层 vdev
-> video_set_drvdata(vdev, sd)                          // 让外层 vdev 反向找到真实 sd
-> vdev->fops = &v4l2_subdev_fops                       // 指定 subdev 专用文件操作
-> vdev->ctrl_handler = sd->ctrl_handler                // 节点直接继承该 subdev 的 controls
-> __video_register_device(..., VFL_TYPE_SUBDEV, ...)   // 最终生成 /dev/v4l-subdevX
-> sd->devnode = vdev                                   // sd 记录自己的外层 devnode
```

这一段来自：

- `drivers/media/v4l2-core/v4l2-device.c:189`
  `__v4l2_device_register_subdev_nodes()`

因此这里要同时记住两层对象：

| 层次 | 对象 | 负责什么 |
| --- | --- | --- |
| 真实模块层 | `struct v4l2_subdev` | 代表 sensor/bridge/ISP 等内部功能模块 |
| 字符设备承载层 | `struct video_device` | 负责把它包装成 `/dev/v4l-subdevX` |

### 8.3 与 `/dev/videoX` 的根本差异

`/dev/videoX` 和 `/dev/v4l-subdevX` 都可能由 `video_device` 承载，但它们表达的对象不同：

| 节点 | 背后的核心对象 | 主要面向谁 | 典型职责 |
| --- | --- | --- | --- |
| `/dev/videoX` | capture/output 一侧的 `video_device` | 普通应用程序 | buffer、streaming、最终图像输入输出 |
| `/dev/v4l-subdevX` | 管线内部 `v4l2_subdev` 的 devnode 包装 | 调试工具、media 管线配置工具、专业应用 | pad format、selection、frame interval、直接 control |

因此：

- sensor 常常不注册 `/dev/videoX`
- 但 sensor 可以拥有 `/dev/v4l-subdevX`
- 两者不是“谁替代谁”，而是分属不同职责层

还要补一个边界：这套用户态 subdev devnode API 受 `CONFIG_VIDEO_V4L2_SUBDEV_API` 控制。  
如果内核没有打开它，源码中的 `subdev_open()` / `subdev_ioctl()` 会退化成直接返回 `-ENODEV` 的桩函数；也就是说，`subdev` 对象本身仍可存在，但用户态不一定能直接打开它。

## 9. `/dev/v4l-subdevX` 的 `open` 与文件句柄模型

### 9.1 `subdev` 节点的 `open` 主线

`/dev/v4l-subdevX` 仍然复用 V4L2 通用字符设备入口，真正分叉发生在第二层 `fops`：

```text
open("/dev/v4l-subdevX")                      // 用户打开 subdev devnode
-> v4l2_fops.open = v4l2_open()               // V4L2 字符设备统一入口
-> vdev->fops->open = subdev_open()           // subdev 节点专用 open
-> subdev_fh_init()                           // 为该文件句柄准备 TRY pad 配置
-> v4l2_fh_init()                             // 建立通用 v4l2_fh，并继承 ctrl_handler
-> file->private_data = &subdev_fh->vfh       // 后续 ioctl 从 file 找回当前句柄
-> sd->internal_ops->open(sd, subdev_fh)      // 如驱动实现，则进入私有 open
```

关键源码：

- `drivers/media/v4l2-core/v4l2-subdev.c:44`
  `subdev_open()`
- `drivers/media/v4l2-core/v4l2-fh.c:21`
  `v4l2_fh_init()`

### 9.2 为什么还要有 `struct v4l2_subdev_fh`

普通 `video_device` 的文件句柄重点常放在：

- 权限
- controls
- 事件
- queue 访问

而 `subdev` 节点还额外需要保存“每个文件句柄自己的临时 pad 状态”。  
这就是：

- `struct v4l2_subdev_fh`
  - 内嵌一个通用 `struct v4l2_fh`
  - 还带一个 `struct v4l2_subdev_pad_config *pad`

源码位置：

- `include/media/v4l2-subdev.h:922`
  `struct v4l2_subdev_fh`

当 `sd->entity.num_pads` 非零时，`subdev_fh_init()` 会调用：

```c
fh->pad = v4l2_subdev_alloc_pad_config(sd);
```

这块 per-file pad 配置，正是后续 `TRY` format 的落点。  
所以在 sensor 驱动里看到：

- `internal_ops->open`
- `v4l2_subdev_get_try_format()`
- `V4L2_SUBDEV_FORMAT_TRY`

时，应把它们和 `v4l2_subdev_fh->pad` 连起来看。  
这也是 `09` 章里 `imx219_open()` 能为每个打开者准备默认 TRY format 的前提。

### 9.3 `internal_ops` 和 `ops` 不是一回事

这两个表都挂在 `struct v4l2_subdev` 上，但职责完全不同：

| 对象 | 关注阶段 | 典型成员 | 谁来调用 |
| --- | --- | --- | --- |
| `v4l2_subdev_internal_ops` | 对象生命周期、devnode 生命周期 | `registered`、`unregistered`、`open`、`close` | V4L2 core |
| `v4l2_subdev_ops` | 业务操作 | `core`、`video`、`pad`、`sensor` | host、subdev ioctl、其他框架代码 |

以 `imx219` 为例：

- `imx219_internal_ops.open = imx219_open`
  - 处理 subdev 文件句柄打开时的私有准备
- `imx219_subdev_ops.pad.set_fmt = imx219_set_pad_format`
  - 处理 pad 格式协商

两者不能混成同一类“回调”。

## 10. `/dev/v4l-subdevX` 的 `ioctl` 主线

### 10.1 先把完整调用链压平

对于普通 `video_device`，`03` 章已经建立了这条主线：

```text
v4l2_fops.unlocked_ioctl
-> v4l2_ioctl()
-> vdev->fops->unlocked_ioctl
-> video_ioctl2()
-> video_usercopy()
-> __video_do_ioctl()
-> v4l2_ioctl_ops->vidioc_*
```

`subdev` 前半段一致，真正分叉点在第二层 `unlocked_ioctl`：

```text
v4l2_fops.unlocked_ioctl                          // 所有 V4L2 字符设备共用的 VFS 入口
-> v4l2_ioctl()                                   // 取出 video_device，转交给 vdev->fops
-> vdev->fops->unlocked_ioctl                     // 对 subdev devnode，实际是 v4l2_subdev_fops.unlocked_ioctl
-> subdev_ioctl()                                 // subdev 节点专用 ioctl 入口
-> video_usercopy(..., subdev_do_ioctl_lock)      // 仍复用通用用户态参数拷贝框架
-> subdev_do_ioctl_lock()                         // 加锁，并确认节点仍处于 registered 状态
-> subdev_do_ioctl()                              // subdev 专用派发核心
```

关键源码：

- `drivers/media/v4l2-core/v4l2-dev.c:353`
  `v4l2_ioctl()`
- `drivers/media/v4l2-core/v4l2-subdev.c:682`
  `subdev_ioctl()`
- `drivers/media/v4l2-core/v4l2-subdev.c:667`
  `subdev_do_ioctl_lock()`
- `drivers/media/v4l2-core/v4l2-subdev.c:354`
  `subdev_do_ioctl()`
- `drivers/media/v4l2-core/v4l2-subdev.c:732`
  `v4l2_subdev_fops`

### 10.2 `subdev_do_ioctl()` 不是 `__video_do_ioctl()` 的换名版本

两者都位于 ioctl 派发末端，但职责不同：

| 派发核心 | 面向对象 | 主要分发到哪里 |
| --- | --- | --- |
| `__video_do_ioctl()` | 普通 video node | `v4l2_ioctl_ops->vidioc_*` |
| `subdev_do_ioctl()` | subdev devnode | `v4l2_subdev_call()` 或 control 框架 |

因此不能把 `subdev` 路径记成：

```text
subdev_ioctl()
-> __video_do_ioctl()
-> v4l2_ioctl_ops
```

源码里并不是这样。  
`subdev` 节点没有走 `video_ioctl2() -> __video_do_ioctl()` 这一套，而是走自己的 `subdev_do_ioctl()`。

### 10.3 `subdev_do_ioctl()` 里常见命令会落到哪里

| 用户态 ioctl | `subdev_do_ioctl()` 中的下游 | 最终回调类型 | 典型含义 |
| --- | --- | --- | --- |
| `VIDIOC_SUBDEV_G_FMT` | `v4l2_subdev_call(sd, pad, get_fmt, ...)` | `sd->ops->pad->get_fmt` | 读取 pad 格式 |
| `VIDIOC_SUBDEV_S_FMT` | `v4l2_subdev_call(sd, pad, set_fmt, ...)` | `sd->ops->pad->set_fmt` | 设置 pad 格式 |
| `VIDIOC_SUBDEV_G_FRAME_INTERVAL` | `v4l2_subdev_call(sd, video, g_frame_interval, ...)` | `sd->ops->video->g_frame_interval` | 读取帧间隔 |
| `VIDIOC_SUBDEV_S_FRAME_INTERVAL` | `v4l2_subdev_call(sd, video, s_frame_interval, ...)` | `sd->ops->video->s_frame_interval` | 设置帧间隔 |
| `VIDIOC_S_CTRL` | `v4l2_s_ctrl(vfh, vfh->ctrl_handler, ...)` | `ctrl->ops->s_ctrl` | 设置 control |
| `VIDIOC_SUBSCRIBE_EVENT` | `v4l2_subdev_call(sd, core, subscribe_event, ...)` | `sd->ops->core->subscribe_event` | 订阅事件 |
| 未被框架单独识别的命令 | `v4l2_subdev_call(sd, core, ioctl, cmd, arg)` | `sd->ops->core->ioctl` | 私有或扩展 ioctl 兜底 |

这张表把 `subdev` 的四类入口分开了：

- `pad`
  - 处理格式、裁剪、selection、mbus code 等 pad 级协商
- `video`
  - 处理帧间隔、标准、时序等视频语义
- `core`
  - 处理事件、调试、私有扩展
- `control`
  - 走通用 control 框架，不直接归到 `subdev_ops` 某个分组里

### 10.4 `v4l2_subdev_call()` 是什么

`v4l2_subdev_call(sd, group, func, ...)` 是 `subdev` 框架统一调业务回调的宏。  
它会先检查：

1. `sd` 是否为空
2. `sd->ops->group` 是否存在
3. 目标回调 `func` 是否存在

若目标回调不存在，通常返回 `-ENOIOCTLCMD`；存在时才真正调用：

```text
v4l2_subdev_call(sd, pad, set_fmt, ...)        // 先检查 pad ops 是否存在
-> wrapper 或 sd->ops->pad->set_fmt(sd, ...)    // 再进入包装层或驱动实现
```

源码位置：

- `include/media/v4l2-subdev.h:1140`
  `v4l2_subdev_call`

这就是 host 驱动在运行期调用：

```c
v4l2_subdev_call(sd, video, s_stream, 1);
```

以及 subdev devnode ioctl 最终能落进同一套 `ops` 的原因。

### 10.5 和 `09` 章的连接点

到这里，`09` 章里的几个动作就都有了前置解释：

- `imx219_subdev_ops.pad.set_fmt`
  - 对应 `VIDIOC_SUBDEV_S_FMT`
- `imx219_internal_ops.open`
  - 对应 `subdev_open()` 最后的私有 open 回调
- `imx219->sd.ctrl_handler`
  - 会在创建 `/dev/v4l-subdevX` 时挂到外层 `video_device`
- `VIDIOC_S_CTRL`
  - 会从 `subdev_do_ioctl()` 进入 control 框架，再落到 sensor 驱动的 `s_ctrl`

## 11. 为什么需要异步注册

在真实系统里：

- sensor 可能先 probe
- CSI host 可能后 probe
- 也可能反过来

如果强依赖 probe 顺序，驱动就会很脆弱。  
所以 V4L2 用 notifier 机制做异步匹配。

### 11.1 host 侧 notifier 一般先做什么

异步模型不是只在 sensor 侧调用一个 `v4l2_async_register_subdev()` 就结束。  
真正完整的异步模型有两条线同时进行：

- `subdev` 侧
  - 初始化 `sd`
  - 注册 `sd`
- host 侧
  - 初始化 notifier
  - 往 notifier 里加入等待项
  - 注册 notifier

host 侧最常见的三步是：

1. `v4l2_async_notifier_init()`
   - 把 notifier 初始化成可用对象
2. `v4l2_async_notifier_add_fwnode_subdev()` 或解析 DT helper
   - 往 notifier 里加入一个或多个等待项
   - 这些等待项本质上就是 `v4l2_async_subdev`
3. `v4l2_async_notifier_register()`
   - 把 notifier 正式交给 async 框架

到这里，host 侧才算真正进入“等待外部 subdev 到来”的状态。

### 11.2 这一章真正的两条并行主线

把 `subdev` 和 notifier 放到同一张时间线上，更容易看清后面的匹配点：

```text
subdev 侧:
初始化 sd
-> 初始化 entity/pad
-> v4l2_async_register_subdev(sd)

host 侧:
初始化 notifier
-> 添加 async_subdev 等待项
-> v4l2_async_notifier_register()

匹配点:
find_match()
-> match_notify()
-> bound()
-> try_complete()
-> complete()
```

## 12. `v4l2_async_register_subdev()`

源码：

- `drivers/media/v4l2-core/v4l2-async.c:750`

>[[!info]]
```C fold:"v4l2_async_register_subdev"
int v4l2_async_register_subdev(struct v4l2_subdev *sd)
{
	struct v4l2_async_notifier *subdev_notifier;
	struct v4l2_async_notifier *notifier;
	int ret;

	/*
	 * No reference taken. The reference is held by the device
	 * (struct v4l2_subdev.dev), and async sub-device does not
	 * exist independently of the device at any point of time.
	 */
	if (!sd->fwnode && sd->dev)
		sd->fwnode = dev_fwnode(sd->dev);

	mutex_lock(&list_lock);

	INIT_LIST_HEAD(&sd->async_list);

	list_for_each_entry(notifier, &notifier_list, list) {
		struct v4l2_device *v4l2_dev =
			v4l2_async_notifier_find_v4l2_dev(notifier);
		struct v4l2_async_subdev *asd;

		if (!v4l2_dev)
			continue;

		asd = v4l2_async_find_match(notifier, sd);
		if (!asd)
			continue;

		ret = v4l2_async_match_notify(notifier, v4l2_dev, sd, asd);
		if (ret)
			goto err_unbind;

		ret = v4l2_async_notifier_try_complete(notifier);
		if (ret)
			goto err_unbind;

		goto out_unlock;
	}

	/* None matched, wait for hot-plugging */
	list_add(&sd->async_list, &subdev_list);

out_unlock:
	mutex_unlock(&list_lock);

	return 0;

err_unbind:
	/*
	 * Complete failed. Unbind the sub-devices bound through registering
	 * this async sub-device.
	 */
	subdev_notifier = v4l2_async_find_subdev_notifier(sd);
	if (subdev_notifier)
		v4l2_async_notifier_unbind_all_subdevs(subdev_notifier);

	if (sd->asd)
		v4l2_async_notifier_call_unbind(notifier, sd, sd->asd);
	v4l2_async_cleanup(sd);

	mutex_unlock(&list_lock);

	return ret;
}
```
它的大致逻辑是：

1. 如果 `sd->dev` 存在，补上 `sd->fwnode`
2. 遍历当前系统中已经注册的 notifier
3. 查找有没有能匹配这个 subdev 的等待项
4. 匹配到了就调用 `bound/complete`
5. 如果暂时没人匹配，就把 subdev 挂到等待`subdevice_list`链表里

也就是说，这一步不是“简单登记一下”，而是：

- 尝试立即绑定
- 绑定不了就排队等待

常见链表[[v4l2驱动总结#核心队列（链表）的**总结日志**]]

### 12.1 `v4l2_async_subdev` 在这里怎么参与匹配

`v4l2_async_register_subdev(sd)` 这一条路径里，真正被拿来做比较的是：

- 当前到达的 `sd`
- notifier 里提前挂好的 `asd`

匹配点就是：

- `v4l2_async_find_match(notifier, sd)`

这一句的含义不是：

- 再创建一个新的 `async_subdev`

而是：

- 在 host 现有 notifier 的等待项里，找有没有哪个 `asd` 能和当前 `sd` 对上

如果匹配成功，就得到：

- 这个真实到达的模块是 `sd`
- host 原先等待它的模板是 `asd`

后面框架会把这组关系记下来，于是：

- `sd->asd`
  - 可以反向指到这次匹配到的 `asd`

这样 `subdev` 和“等待模板”之间就建立了关联。

### 12.2 `v4l2_async_notifier` 在这里做什么

在 `v4l2_async_register_subdev(sd)` 这条路径里，notifier 主要承担三件事：

1. 提供等待集合
   - 让框架知道 host 正在等哪些 `asd`
2. 提供回调入口
   - 匹配成功后要调哪个 `bound()`
   - 全部匹配完成后要调哪个 `complete()`
3. 提供归属上下文
   - 让框架知道匹配成功后该把 `sd` 挂到哪个 `v4l2_device`

所以 `notifier` 在这一章里的作用，不是“又一个等待链表”，而是：

- host 侧的匹配管理器

### 12.3 匹配成功后真正发生了什么

这一段是最容易被一笔带过的地方。

`v4l2_async_match_notify()` 做的不是单纯“通知一下 host”，而是先把管理关系补齐，再回调 host：

1. `v4l2_device_register_subdev(v4l2_dev, sd)`
   - 先把 `sd` 正式挂进 host 的 `v4l2_dev->subdevs`
2. 记录 `sd <-> asd` 的匹配关系
3. 调 notifier 的 `bound()`
4. 再尝试 `v4l2_async_notifier_try_complete()`
5. 如果当前 notifier 下所有等待项都匹配完成，再调 `complete()`

所以这里的顺序要特别记住：

- 不是先 `bound()` 再注册 `subdev`
- 而是先把 `sd` 纳入 host 管理，再进入 `bound()/complete()`

### 12.4 这一章里三者到底怎么协作

把 `subdev`、`async_subdev`、`notifier` 再压成一句最核心的话：

- `subdev`
  - 真正到达系统里的模块对象
- `async_subdev`
  - host 预先声明的匹配模板
- `notifier`
  - 维护模板、发起匹配、触发 `bound/complete` 的 host 管理器

三者一起工作，才形成这一章的完整异步模型。

## 13. `v4l2_async_register_subdev_sensor_common()`

源码：

- `drivers/media/v4l2-core/v4l2-fwnode.c:1340`
>[!INFO]
```c {20,24,33} fold:"v4l2_async_register_subdev_sensor_common"
int v4l2_async_register_subdev_sensor_common(struct v4l2_subdev *sd)
{
	struct v4l2_async_notifier *notifier;
	int ret;

	if (WARN_ON(!sd->dev))
		return -ENODEV;

	notifier = kzalloc(sizeof(*notifier), GFP_KERNEL);
	if (!notifier)
		return -ENOMEM;

	v4l2_async_notifier_init(notifier);

	ret = v4l2_async_notifier_parse_fwnode_sensor_common(sd->dev,
							     notifier);
	if (ret < 0)
		goto out_cleanup;

	ret = v4l2_async_subdev_notifier_register(sd, notifier); //链接-常见链表的notifier_list链表的入队函数
	if (ret < 0)
		goto out_cleanup;

	ret = v4l2_async_register_subdev(sd); //链接-常见链表的subdevice_list链表的入队函数
	if (ret < 0)
		goto out_unregister;

	sd->subdev_notifier = notifier;

	return 0;

out_unregister:
	v4l2_async_notifier_unregister(notifier); //链接-常见链表的subdevice_list链表的出队函数

out_cleanup:
	v4l2_async_notifier_cleanup(notifier);
	kfree(notifier);

	return ret;
}
```

这个 helper 很适合 camera sensor，用途是：

1. 分配 notifier
2. 从 fwnode / device tree 解析 sensor 常见连接信息
3. 注册 subdev notifier
4. 最后再调用 `v4l2_async_register_subdev(sd)`

所以 `imx219.c:1477` 直接调用它，能省掉一大段样板代码。


## 14. `imx219.c` 的 subdev 初始化流程
>[!TIP]
>总结：


>[!INFO]
```C {11,14,73,79,78,80,83,88} fold:"imx219_probe"
static int imx219_probe(struct i2c_client *client)
{
	struct device *dev = &client->dev;
	struct imx219 *imx219;
	int ret;

	imx219 = devm_kzalloc(&client->dev, sizeof(*imx219), GFP_KERNEL);
	if (!imx219)
		return -ENOMEM;

	v4l2_i2c_subdev_init(&imx219->sd, client, &imx219_subdev_ops);

	/* Check the hardware configuration in device tree */
	if (imx219_check_hwcfg(dev))
		return -EINVAL;

	/* Get system clock (xclk) */
	imx219->xclk = devm_clk_get(dev, NULL);
	if (IS_ERR(imx219->xclk)) {
		dev_err(dev, "failed to get xclk\n");
		return PTR_ERR(imx219->xclk);
	}

	imx219->xclk_freq = clk_get_rate(imx219->xclk);
	if (imx219->xclk_freq != IMX219_XCLK_FREQ) {
		dev_err(dev, "xclk frequency not supported: %d Hz\n",
			imx219->xclk_freq);
		return -EINVAL;
	}

	ret = imx219_get_regulators(imx219);
	if (ret) {
		dev_err(dev, "failed to get regulators\n");
		return ret;
	}

	/* Request optional enable pin */
	imx219->reset_gpio = devm_gpiod_get_optional(dev, "reset",
						     GPIOD_OUT_HIGH);

	/*
	 * The sensor must be powered for imx219_identify_module()
	 * to be able to read the CHIP_ID register
	 */
	ret = imx219_power_on(dev);
	if (ret)
		return ret;

	ret = imx219_identify_module(imx219);
	if (ret)
		goto error_power_off;

	/* Set default mode to max resolution */
	imx219->mode = &supported_modes[0];

	/* sensor doesn't enter LP-11 state upon power up until and unless
	 * streaming is started, so upon power up switch the modes to:
	 * streaming -> standby
	 */
	ret = imx219_write_reg(imx219, IMX219_REG_MODE_SELECT,
			       IMX219_REG_VALUE_08BIT, IMX219_MODE_STREAMING);
	if (ret < 0)
		goto error_power_off;
	usleep_range(100, 110);

	/* put sensor back to standby mode */
	ret = imx219_write_reg(imx219, IMX219_REG_MODE_SELECT,
			       IMX219_REG_VALUE_08BIT, IMX219_MODE_STANDBY);
	if (ret < 0)
		goto error_power_off;
	usleep_range(100, 110);

	ret = imx219_init_controls(imx219);
	if (ret)
		goto error_power_off;

	/* Initialize subdev */
	imx219->sd.internal_ops = &imx219_internal_ops;
	imx219->sd.flags |= V4L2_SUBDEV_FL_HAS_DEVNODE;
	imx219->sd.entity.function = MEDIA_ENT_F_CAM_SENSOR;

	/* Initialize source pad */
	imx219->pad.flags = MEDIA_PAD_FL_SOURCE;

	/* Initialize default format */
	imx219_set_default_format(imx219);

	ret = media_entity_pads_init(&imx219->sd.entity, 1, &imx219->pad);
	if (ret) {
		dev_err(dev, "failed to init entity pads: %d\n", ret);
		goto error_handler_free;
	}

	ret = v4l2_async_register_subdev_sensor_common(&imx219->sd);
	if (ret < 0) {
		dev_err(dev, "failed to register sensor sub-device: %d\n", ret);
		goto error_media_entity;
	}

	/* Enable runtime PM and turn off the device */
	pm_runtime_set_active(dev);
	pm_runtime_enable(dev);
	pm_runtime_idle(dev);

	return 0;

error_media_entity:
	media_entity_cleanup(&imx219->sd.entity);

error_handler_free:
	imx219_free_controls(imx219);

error_power_off:
	imx219_power_off(dev);

	return ret;
}
```

它非常值得当模板看：

1. `v4l2_i2c_subdev_init()`，主要是调用v4l2_i2c_subdev_init，多了一个i2c的绑定  #L11
2. 检查硬件配置 `imx219_check_hwcfg()` #L14
3. 上电、读 chip id
4. 初始化 controls   #L73
5. 设 `sd.internal_ops` #L78
6. 设 `sd.flags |= V4L2_SUBDEV_FL_HAS_DEVNODE` #L79
7. 设 `sd.entity.function = MEDIA_ENT_F_CAM_SENSOR` #L80
8. 设 source pad #L83
9. `media_entity_pads_init()` #L88
10. `v4l2_async_register_subdev_sensor_common()`

这条链说明：

- sensor 先把自己建模成一个可被绑定的 subdev
- 真正什么时候接到 host 上，由异步匹配决定
## 15. 主机侧一般怎么和 subdev 交互

常见方式有两种：

### 15.1 注册时绑定

host 通过 notifier 的 `bound()` 回调拿到 subdev 指针。

### 15.2 运行时调用

host 用：

```c
v4l2_subdev_call(sd, video, s_stream, 1);
```

或者：

```c
v4l2_device_call_until_err(&v4l2_dev, 0, video, s_stream, 1);
```

去通知整个管线开始或停止流。

## 16. 全流程总结

```text
sensor probe()                                              // 底层总线匹配后进入具体 sensor 驱动
-> v4l2_i2c_subdev_init()                                   // 把私有结构中的 sd 初始化成真实 subdev 对象
-> sd.ops / sd.internal_ops / sd.ctrl_handler               // 填好业务回调、内部回调和 controls 入口
-> media_entity_pads_init()                                 // 建立 entity/pad，准备进入 media graph
-> v4l2_async_register_subdev_sensor_common()               // 把 sensor 交给 async 框架
-> host notifier 匹配成功                                   // 找到等待它的 async_subdev 描述
-> v4l2_device_register_subdev()                            // 把 sd 正式挂到 host 的 v4l2_dev->subdevs
-> __v4l2_device_register_subdev_nodes()                    // host 可选地为 sd 建立 devnode
-> /dev/v4l-subdevX                                         // 用户态可以直接访问这个内部模块节点
-> open() -> subdev_open()                                  // 建立 v4l2_subdev_fh 与 TRY pad 配置
-> ioctl() -> subdev_ioctl()                                // 进入 subdev 专用 ioctl 入口
-> video_usercopy() -> subdev_do_ioctl_lock()               // 统一搬运用户参数并做串行化
-> subdev_do_ioctl()                                        // 按命令类型继续分发
-> pad/video/core 回调，或 v4l2_ctrl 框架                   // 落到 set_fmt、s_stream、s_ctrl 等具体能力
```

这一章的闭环可以按四段看：

- `对象建立`
  - `subdev` 先成为真实模块对象
- `图与归属`
  - `entity/pad` 让它进入媒体图，async 框架让它归到 host
- `节点暴露`
  - host 可选把它包装成 `/dev/v4l-subdevX`
- `用户态使用`
  - `open/ioctl` 再把请求导向 `internal_ops`、`subdev_ops` 或 control 框架

读完这条总线，再进入 `09` 看 `imx219`，就能把示例驱动中的每个填写点放回对应阶段，而不是把它们看成一串互不相干的 API。

## 17. 异步模型的核心价值

```mermaid
flowchart TD
    A["sensor probe"] --> B["v4l2_async_register_subdev()"]
    C["host probe"] --> D["register notifier"]
    B --> E["匹配 fwnode/I2C/devname"]
    D --> E
    E --> F["bound()"]
    F --> G["complete()"]
    G --> H["形成完整媒体管线"]
```

它把“谁先 probe”这个问题从驱动逻辑里拿掉了。

## 18. 最容易出问题的地方

### 18.1 DT/fwnode 信息不一致

匹配条件不对时，subdev 会一直留在等待链表里。

### 18.2 pad/entity 没初始化完整

后面媒体链路创建和格式协商会很别扭。

### 18.3 以为 sensor 自己会注册 `/dev/videoX`

大多数 sensor 驱动不会这么做，它通常只注册 `subdev`。

## 19. 一句话总结

`subdev` 先描述媒体管线里的内部组件，再按需暴露为 `/dev/v4l-subdevX`，由 `subdev_open()`、`subdev_do_ioctl()` 和 `v4l2_subdev_call()` 把用户请求导向对应回调；`async notifier` 则负责把这些组件在运行时拼回 host 管理。  
这也是为什么 camera 类 V4L2 驱动看起来总比普通字符设备驱动更“分层”。
