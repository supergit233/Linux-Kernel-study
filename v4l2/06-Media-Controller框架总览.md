# Media Controller 框架总览

## 导读

### 本章定位

这一章是进入管线型模型前的对象总览，负责先说明 Media Controller 为什么存在、`/dev/mediaX` 是怎么来的，以及 `media_device / entity / pad / link` 这四层对象各自承担什么职责。

### 核心对象

- `struct media_device`
  - 媒体图的总容器
- `struct media_entity`
  - 媒体图里的节点
- `struct media_pad`
  - 节点上的输入/输出端口
- `struct media_link`
  - pad-to-pad 连接对象

### 关键函数

- `media_device_init()`
- `media_device_register()`
- `media_device_register_entity()`
- `media_create_pad_link()`

### 主流程

建立 `media_device` -> 注册 `/dev/mediaX` -> 注册 entity/pad/link 对象 -> 把 V4L2 对象和媒体图模型挂接起来

## 1. 先把边界说清

前面的 V4L2 笔记主要回答的是：

- `/dev/videoX` 怎么出来
- `ioctl` 怎么派发
- `vb2` 怎么组织 buffer
- `subdev` 怎么异步绑定

但在 camera/ISP 这类复杂媒体设备里，只知道这些还不够。  
因为真实硬件往往不是一个节点，而是一张“媒体拓扑图”。

这时就轮到 **Media Controller** 出场了。

## 2. Media Controller 解决什么问题

它解决的是“设备内部拓扑描述”和“链路控制”。

也就是把这些问题标准化：

1. 系统里有哪些媒体实体 `entity`
2. 每个实体有哪些输入/输出端 `pad`
3. 哪些 pad 之间存在连接 `link`
4. 某条链路当前是否启用
5. 当前 streaming 的 pipeline 覆盖了哪些实体

所以它的关注点不是 `VIDIOC_QBUF`，而是：

- 拓扑
- 路由
- 链接状态
- pipeline 合法性

## 3. V4L2 和 Media Controller 的关系

两者不是替代关系，而是上下层协作关系。

### 3.1 V4L2

更偏向：

- `/dev/videoX`
- `VIDIOC_*`
- buffer streaming

### 3.2 Media Controller

更偏向：

- `/dev/mediaX`
- entity/pad/link 拓扑
- pipeline 校验

### 3.3关键结构体

media_device
#media_device
```c fold："media_device"
struct media_device {
	/* dev->driver_data points to this struct. */
	struct device *dev;
	struct media_devnode *devnode;

	char model[32];
	char driver_name[32];
	char serial[40];
	char bus_info[32];
	u32 hw_revision;

	u64 topology_version;

	u32 id;
	struct ida entity_internal_idx;
	int entity_internal_idx_max;

	struct list_head entities;
	struct list_head interfaces;
	struct list_head pads;
	struct list_head links;

	/* notify callback list invoked when a new entity is registered */
	struct list_head entity_notify;

	/* Serializes graph operations. */
	struct mutex graph_mutex;
	struct media_graph pm_count_walk;

	void *source_priv;
	int (*enable_source)(struct media_entity *entity,
			     struct media_pipeline *pipe);
	void (*disable_source)(struct media_entity *entity);

	const struct media_device_ops *ops;

	struct mutex req_queue_mutex;
	atomic_t request_id;
};
```


## 4. 关键源码目录

Linux 5.10 主线里，Media Controller 的核心代码主要在：

- `drivers/media/mc/mc-device.c`
- `drivers/media/mc/mc-entity.c`
- `drivers/media/mc/mc-devnode.c`

关键头文件在：

- `include/media/media-device.h`
- `include/media/media-entity.h`
- `include/media/media-devnode.h`
- `include/uapi/linux/media.h`

## 5. `/dev/mediaX` 是怎么来的

这条链路的核心在：

- `drivers/media/mc/mc-device.c:693`
  `media_device_init()`
- `drivers/media/mc/mc-device.c:732`
  `__media_device_register()`
- `drivers/media/mc/mc-devnode.c:211`
  `media_devnode_register()`

典型驱动写法通常是：

1. 填 `struct media_device`
2. `media_device_init(&mdev)`
3. 把 `v4l2_dev.mdev = &mdev`
4. 所有 entity/link 建好以后
5. `media_device_register(&mdev)`

这一步注册完成后，用户态才能看到 `/dev/mediaX`。

## 6. `media_device_init()` 做了什么

源码：

- `drivers/media/mc/mc-device.c:693`
>[!INFO]
```C fold:"media_device_init"
void media_device_init(struct media_device *mdev)
{
	INIT_LIST_HEAD(&mdev->entities);
	INIT_LIST_HEAD(&mdev->interfaces);
	INIT_LIST_HEAD(&mdev->pads);
	INIT_LIST_HEAD(&mdev->links);
	INIT_LIST_HEAD(&mdev->entity_notify);

	mutex_init(&mdev->req_queue_mutex);
	mutex_init(&mdev->graph_mutex);
	ida_init(&mdev->entity_internal_idx);

	atomic_set(&mdev->request_id, 0);

	dev_dbg(mdev->dev, "Media device initialized\n");
}
```

它主要初始化：

- `entities`
- `interfaces`
- `pads`
- `links`
- `entity_notify`
- `graph_mutex`
- `req_queue_mutex`
- `entity_internal_idx`

这里先把“媒体图容器”搭起来。

## 7. `media_device_register()` 做了什么

真正重活在：

- `drivers/media/mc/mc-device.c:732`
  `__media_device_register()`
>[!INFO]
```C
#define media_device_register(mdev) __media_device_register(mdev, THIS_MODULE)
```
```C {20} fold:"__media_device_register"
int __must_check __media_device_register(struct media_device *mdev,
					 struct module *owner)
{
	struct media_devnode *devnode;
	int ret;

	devnode = kzalloc(sizeof(*devnode), GFP_KERNEL);
	if (!devnode)
		return -ENOMEM;

	/* Register the device node. */
	mdev->devnode = devnode;
	devnode->fops = &media_device_fops;
	devnode->parent = mdev->dev;
	devnode->release = media_device_release;

	/* Set version 0 to indicate user-space that the graph is static */
	mdev->topology_version = 0;

	ret = media_devnode_register(mdev, devnode, owner);
	if (ret < 0) {
		/* devnode free is handled in media_devnode_*() */
		mdev->devnode = NULL;
		return ret;
	}

	ret = device_create_file(&devnode->dev, &dev_attr_model);
	if (ret < 0) {
		/* devnode free is handled in media_devnode_*() */
		mdev->devnode = NULL;
		media_devnode_unregister_prepare(devnode);
		media_devnode_unregister(devnode);
		return ret;
	}

	dev_dbg(mdev->dev, "Media device registered\n");

	return 0;
}
```

关键动作：

1. 分配 `media_devnode`
2. 绑定 `media_device_fops`
3. 初始化 `topology_version`
4. 调 `media_devnode_register()`

这意味着：

- `/dev/mediaX` 本质上也是一个字符设备节点
- 只是它服务的不是 `VIDIOC_*`，而是 media graph 相关 ioctl

## 8. `media_device` 和 `v4l2_device` 怎么挂起来

在复杂 V4L2 驱动里，经常能看到：

```c
camss->v4l2_dev.mdev = &camss->media_dev;
ret = v4l2_device_register(camss->dev, &camss->v4l2_dev);
```

例如：

- `drivers/media/platform/qcom/camss/camss.c:882`
  `media_device_init(&camss->media_dev)`
- `drivers/media/platform/qcom/camss/camss.c:893` 左右
  `camss->v4l2_dev.mdev = &camss->media_dev`
[[02-video_device注册与open链路#3. `v4l2_device_register()` 做了什么]]
[[01-V4L2核心对象与驱动模型#2. `struct v4l2_device`]]
这说明：

- `v4l2_device` 管 V4L2 子设备和 `/dev/videoX`
- `media_device` 管整体媒体拓扑和 `/dev/mediaX`

两者经常在同一个驱动里同时存在，是挂在v4l2_dev.mdev。

## 9. Media Controller 的核心对象

这层最关键的对象有四个：
#media_device #media_entity  #media_pad  #media_link
1. `media_device` ^02e8b1
>[!INFO]  
```C fold:"media_device"
struct media_device {
	/* dev->driver_data points to this struct. */
	struct device *dev;
	struct media_devnode *devnode;

	char model[32];
	char driver_name[32];
	char serial[40];
	char bus_info[32];
	u32 hw_revision;

	u64 topology_version;

	u32 id;
	struct ida entity_internal_idx;
	int entity_internal_idx_max;

	struct list_head entities;
	struct list_head interfaces;
	struct list_head pads;
	struct list_head links;

	/* notify callback list invoked when a new entity is registered */
	struct list_head entity_notify;

	/* Serializes graph operations. */
	struct mutex graph_mutex;
	struct media_graph pm_count_walk;

	void *source_priv;
	int (*enable_source)(struct media_entity *entity,
			     struct media_pipeline *pipe);
	void (*disable_source)(struct media_entity *entity);

	const struct media_device_ops *ops;

	struct mutex req_queue_mutex;
	atomic_t request_id;
};
```

2. `media_entity`
>[!INFO]
```c fold:"media_entity"
struct media_entity {
	struct media_gobj graph_obj;	/* must be first field in struct */
	const char *name;
	enum media_entity_type obj_type;
	u32 function;
	unsigned long flags;

	u16 num_pads;
	u16 num_links;
	u16 num_backlinks;
	int internal_idx;

	struct media_pad *pads;
	struct list_head links;

	const struct media_entity_operations *ops;

	int stream_count;
	int use_count;

	struct media_pipeline *pipe;

	union {
		struct {
			u32 major;
			u32 minor;
		} dev;
	} info;
};
```

3. `media_pad`
>[!INFO]
```C fold:"media_pad"
struct media_pad {
	struct media_gobj graph_obj;	/* must be first field in struct */
	struct media_entity *entity;
	u16 index;
	enum media_pad_signal_type sig_type;
	unsigned long flags;
};
```

4. `media_link`
>[!INFO]
```C fold:"media_link"
struct media_link {
	struct media_gobj graph_obj;
	struct list_head list;
	union {
		struct media_gobj *gobj0;
		struct media_pad *source;
		struct media_interface *intf;
	};
	union {
		struct media_gobj *gobj1;
		struct media_pad *sink;
		struct media_entity *entity;
	};
	struct media_link *reverse;
	unsigned long flags;
	bool is_backlink;
};
```


这一层可先画成：

```mermaid
flowchart LR
    A["media_device"] --> B["media_entity"]
    B --> C["media_pad(source/sink)"]
    C --> D["media_link"]
```

如果只记名字，很容易知道“这层有这四个结构体”，但还不容易形成对象模型。  
更顺的理解方式是：

- `media_device`
  - 整张媒体图的容器
- `media_entity`
  - 图里的节点
- `media_pad`
  - 节点上的输入/输出端口
- `media_link`
  - 两个 pad 之间的连接

### 9.1 `media_device`

`media_device` 适合先理解成“整张媒体图的总容器”。
[[06-Media-Controller框架总览#9. Media Controller 的核心对象]]
它主要回答的是：

- 这张图归谁管理
- 这张图里有哪些 entity / pad / link
- `/dev/mediaX` 对外应该暴露哪张拓扑

从结构职责上看，它更偏全局管理层：

- 管 entity 集合
- 管 interfaces / pads / links 这些 graph object
- 持有 `graph_mutex`
- 持有 `topology_version`
- 最终对应 `/dev/mediaX`

先看这些关键字段：

- `dev`
  - 这张媒体图归属哪个底层 `struct device`
- `devnode`
  - `/dev/mediaX` 对应的 `media_devnode`
- `model / driver_name / serial / bus_info / hw_revision`
  - 这张媒体图对用户态暴露的设备身份信息
- `topology_version`
  - 当前媒体图拓扑版本号
- `entities / interfaces / pads / links`
  - 这张图里所有 graph object 的全局集合
- `entity_notify`
  - 新 entity 注册时的通知链表
- `graph_mutex`
  - 串行化媒体图操作
- `req_queue_mutex`
  - request 与部分 streaming 路径配合时使用的锁
- `ops`
  - 这张图在 source enable/disable 等场景下的全局操作入口

所以它不是媒体图里的一个“节点”，而是**整张媒体图的容器和入口**。

这也是为什么：

- `media_device_init()`
  是先把“图容器”搭起来
- `media_device_register()`
  是最后把 `/dev/mediaX` 暴露给用户态

### 9.2 `media_entity`
[[06-Media-Controller框架总览#9. Media Controller 的核心对象]]
`media_entity` 是媒体图里的节点。  
它代表的不是“某种固定设备类型”，而是“图上的一个功能块”。

常见对应物包括：

- 一个 `v4l2_subdev`
- 一个 capture/output `video_device`
- 一个 ISP/CSI/sensor/bridge 功能块

所以在 Media Controller 眼里，重点不是“它是不是 sensor”，而是：

- 它是不是一个独立功能块
- 它有哪些输入/输出端
- 它和谁相连

读 `media_entity` 时，最值得先抓住的是这些语义：

- `pads`
  - 这个节点有哪些端口，链接media_pad结构体
- `num_pads`
  - 一共有多少端口
- `links`
  - 和别人的连接关系
- `num_links`
  - 当前连接数量
- `function`
  - 这个节点在图里的功能角色
- `ops`
  - 这个节点在 link validate、link setup 等场景下如何参与框架
- `stream_count`
  - 当前是否已经参与某条正在运行的 pipeline
- `pipe`
  - 当前归属的 pipeline

按字段继续往下拆，最值得先抓住的是：

- `graph_obj`
  - 这个 entity 作为 graph object 挂进整张媒体图的基础对象头
- `name`
  - 节点名字，用户态打印拓扑时也会看到
- `obj_type`
  - 这个 graph object 的对象类型
- `function`
  - 节点在图里的功能角色，例如 sensor、video node、decoder、bridge
- `flags`
  - entity 自身的状态与能力标志
- `pads`
  - 指向该节点所有 pad 的数组
- `num_pads`
  - 这个节点一共有多少个 pad
- `links`
  - 从这个节点视角维护的连接链表
- `num_links / num_backlinks`
  - 正向 link 与 backlink 的数量
- `ops`
  - 这个节点参与 `link_setup`、`link_validate`、`has_pad_interdep` 等操作时的回调
- `stream_count`
  - 当前有多少条 streaming pipeline 正在经过它
- `use_count`
  - 当前有多少地方正在使用这个 entity
- `pipe`
  - 当前归属的 `media_pipeline`
- `info.dev.major / minor`
  - 如果这个 entity 背后对应的是 devnode，这里会记录设备号信息

所以 `entity` 的本质是：

**媒体图里的功能节点，而不是用户态设备节点本身。**

### 9.3 `media_pad`
[[06-Media-Controller框架总览#9. Media Controller 的核心对象]]
`media_pad` 不是节点，而是节点上的端口。  
一个 entity 没有 pad，就很难描述“数据从哪里进、从哪里出”。

它主要回答的是：

- 这个端口属于哪个 entity
- 这是该 entity 的第几个端口
- 这是输入端还是输出端

所以读 `media_pad` 时，最关键的不是字段数量，而是三层语义：

- `pad->entity`
  - 这个 pad 属于谁
- `pad->index`
  - 这是这个 entity 的第几个 pad
- `pad->flags`
  - 这是 `source` 还是 `sink`

从结构体字段看，关键点可以继续收成：

- `graph_obj`
  - 这个 pad 作为 graph object 挂进媒体图的对象头
- `entity`
  - 这个 pad 属于哪个 `media_entity`
- `index`
  - 这是所属 entity 的第几个 pad
- `sig_type`
  - 这个 pad 传的是什么类型的信号
- `flags`
  - 这个 pad 的方向和约束，最常见的是 `SOURCE` 和 `SINK`

这也是为什么：

- `media_entity_pads_init()`
  不是“可选修饰动作”
- 而是在正式建图前，把一个节点的输入/输出端口建模出来

没有 `pad`，就只有“节点”；  
有了 `pad`，媒体图才开始有“方向”和“连接点”。

### 9.4 `media_link`
[[06-Media-Controller框架总览#9. Media Controller 的核心对象]]
`media_link` 描述的是两个 pad 之间的连接关系。

它回答的是：

- 哪个 source pad 连到哪个 sink pad
- 这条连接当前是否启用
- 这条连接是否允许运行时修改

所以 `link` 不是抽象“线条”，而是带状态、带属性的连接对象。

最关键的语义通常是：

- `source`
  - 从哪个输出 pad 发出
- `sink`
  - 连到哪个输入 pad
- `flags`
  - 是否 `ENABLED`
  - 是否 `IMMUTABLE`

再对应到结构体字段，可直接看成：

- `graph_obj`
  - 这个 link 作为 graph object 挂进媒体图的对象头
- `list`
  - 挂到 entity 链表里的 link 节点
- `source`
  - 正常 pad-to-pad 链路里的源 pad
- `sink`
  - 正常 pad-to-pad 链路里的宿 pad
- `intf / entity`
  - 某些 interface 相关 link 会复用这组 union 字段
- `reverse`
  - 这条 link 对应的反向 link
- `flags`
  - 当前 link 的状态和属性，例如 `ENABLED`、`IMMUTABLE`、`DYNAMIC`
- `is_backlink`
  - 这条 link 是否是内部维护的 backlink

这里还要补一个容易忽略的点：

- 用户态通常只看到正向 link
  - `source pad -> sink pad`
- 内核内部还会维护 backlink
  - 主要为了图遍历和管理方便

所以 `media_link` 不是简单地“两个指针连一下”，而是整个媒体图遍历和 link 管理的基础对象。

### 9.5 这四个对象怎么连成一张图

先建立下面这条对象直觉：

```text
media_device
  -> 管整张图
  -> 图里有多个 media_entity
       -> 每个 entity 有多个 media_pad
            -> pad 和 pad 之间用 media_link 连接
```

也就是说：

- `media_device` 是容器
- `media_entity` 是节点
- `media_pad` 是节点端口
- `media_link` 是端口之间的连接

这四层一旦对上，后面再看：

- `media_device_register_entity()`
- `media_entity_pads_init()`
- `media_create_pad_link()`
- `media_pipeline_start()`

就会更顺，因为这些函数本质上都是在操作这四类对象。

### 9.6 和 `v4l2_device` / `video_device` / `subdev` 的关系

这里也很容易混。

这一组关系可先记成：

- `v4l2_device`
  - 是 V4L2 侧管理对象
- `media_device`
  - 是 MC 侧拓扑容器
- `video_device`
  - 往往会在 MC 图里对应一个 entity
- `v4l2_subdev`
  - 往往也会在 MC 图里对应一个 entity

所以：

- `media_entity` 不是和 `video_device/subdev` 平行竞争的概念
- 而更像是它们在媒体图层里的“图节点表现形式”

在复杂驱动里，经常是：

- `video_device`
  既是 V4L2 数据节点
  又在 MC 图里对应一个 entity
- `v4l2_subdev`
  既是内部模块对象
  又在 MC 图里对应一个 entity

这样一来，V4L2 侧负责：

- `/dev/videoX`
- `VIDIOC_*`
- buffer streaming

MC 侧负责：

- `/dev/mediaX`
- entity/pad/link 拓扑
- pipeline 校验

## 10. 一张最小的媒体图

以最简单的 camera 管线为例：

```mermaid
flowchart LR
    A["sensor subdev"] -->|"source pad -> sink pad"| B["CSI/ISP subdev"]
    B -->|"source pad -> sink pad"| C["capture video_device"]
```

在 Media Controller 眼里，真正的重点不是“这是 sensor 还是 video 节点”，而是：

- 谁是 entity
- 谁有哪些 pad
- 哪两个 pad 通过 link 连起来

## 11. 为什么 `media-ctl -p` 能看到整张图

因为 `/dev/mediaX` 支持一组专门的 media ioctls，用户态工具正是通过这些 ioctl 去枚举：

- entity
- pad
- link
- topology

相关 UAPI 定义在：

- `include/uapi/linux/media.h`

>[!TIP]
>暴露给用户是通过`media_device_register`->`__media_device_register`->`media_devnode_register()`
## 12. 一个典型 host 驱动怎么收尾

以 `xilinx-vipp.c` 为例，它在异步绑定完成后会：

1. 为 subdev 之间创建 pad link
2. 为 DMA video 节点创建 pad link
3. `v4l2_device_register_subdev_nodes(&xdev->v4l2_dev)`
4. `media_device_register(&xdev->media_dev)`

对应位置：

- `drivers/media/platform/xilinx/xilinx-vipp.c:306`
- `drivers/media/platform/xilinx/xilinx-vipp.c:310`

也就是说，**媒体图先构建完整，再把 `/dev/mediaX` 暴露给用户态**。

## 13. 这一层最容易误解的点

### 13.1 `/dev/mediaX` 不是数据面节点

它不是拿来做采集数据收发的，而是拿来管理拓扑和链路的。

### 13.2 `media_device_register()` 不等于 `/dev/videoX`

它对应的是 `/dev/mediaX`。

### 13.3 不是所有 V4L2 驱动都必须重度依赖 MC

简单单节点驱动可以几乎不使用 Media Controller。  
但 camera/ISP/bridge 这类复杂拓扑设备，通常离不开它。
