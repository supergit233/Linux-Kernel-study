# 典型 `video_device` 驱动例子：`sh_vou.c`

## 导读

### 本章定位

这一章用 `sh_vou.c` 把前面 `02-04` 的单节点主线全部落到具体驱动，实现从 `probe`、`open`、`ioctl`、`vb2` 到 IRQ 完成路径的一次闭环。

### 核心对象

- `sh_vou_video_template`
  - 这个驱动对外暴露的 `video_device` 模板
- `sh_vou_fops`
  - 文件操作表
- `sh_vou_ioctl_ops`
  - `VIDIOC_*` 回调表
- `sh_vou_qops`
  - 驱动自己的 `vb2_ops`

### 关键函数

- `sh_vou_probe()`
- `sh_vou_open() / sh_vou_release()`
- `sh_vou_queue_setup() / sh_vou_buf_queue()`
- `sh_vou_start_streaming() / sh_vou_stop_streaming()`
- 中断完成路径里的 `vb2_buffer_done()`

### 主流程

probe 建节点 -> open/release 落驱动 -> `REQBUFS/QBUF/STREAMON` 进入 `vb2` -> IRQ 完成回 buffer -> `DQBUF/STREAMOFF` 收尾

## 1. 为什么选 `sh_vou.c`

示例文件：

- `drivers/media/platform/sh_vou.c`

这个驱动很适合拿来当 V4L2 教学样例，原因是它结构清楚：

- 有完整的 `v4l2_file_operations`
- 有完整的 `v4l2_ioctl_ops`
- 有完整的 `vb2_queue`
- 有实际中断完成路径
- 逻辑不像大型 camera host 那么绕

虽然它是 **video output** 驱动，不是 capture，但对理解 V4L2 框架非常友好。

## 2. 先看它的三张表

### 2.1 `v4l2_ioctl_ops`

- `drivers/media/platform/sh_vou.c:1173`

这里定义了：

- 能力查询
- 格式枚举/设置
- output 相关操作
- 一整套 `vb2_ioctl_*`

这代表它是一个标准的流式 V4L2 设备节点。

### 2.2 `v4l2_file_operations`

- `drivers/media/platform/sh_vou.c:1198`

核心成员：

```c
.open           = sh_vou_open,
.release        = sh_vou_release,
.unlocked_ioctl = video_ioctl2,
.mmap           = vb2_fop_mmap,
.poll           = vb2_fop_poll,
.write          = vb2_fop_write,
```

这个组合非常典型：

- open/release 由驱动做硬件电源与句柄管理
- ioctl 交给 `video_ioctl2`
- mmap/poll/write 交给 vb2 helper

### 2.3 `video_device` 模板

- `drivers/media/platform/sh_vou.c:1208`

关键字段：

- `name = "sh_vou"`
- `fops = &sh_vou_fops`
- `ioctl_ops = &sh_vou_ioctl_ops`
- `vfl_dir = VFL_DIR_TX`
- `device_caps = V4L2_CAP_VIDEO_OUTPUT | V4L2_CAP_READWRITE | V4L2_CAP_STREAMING`

这就是一个“面向用户态”的 V4L2 视频节点定义。

## 3. probe 主线非常标准

入口：

- `drivers/media/platform/sh_vou.c:1217`
  `sh_vou_probe()`

它的主线可以拆成下面几步。

### 3.1 准备私有结构和缺省格式

包括：

- 分配 `vou_dev`
- 初始化锁、链表
- 填默认分辨率和像素格式

这一段本质是在准备驱动自己的上下文。

### 3.2 建立 `v4l2_device`

- `drivers/media/platform/sh_vou.c:1275`

```c
ret = v4l2_device_register(&pdev->dev, &vou_dev->v4l2_dev);
```

这一步把 `vou_dev->v4l2_dev` 建起来。

### 3.3 填 `video_device`

```c
vdev = &vou_dev->vdev;
*vdev = sh_vou_video_template;
vdev->v4l2_dev = &vou_dev->v4l2_dev;
vdev->release = video_device_release_empty;
vdev->lock = &vou_dev->fop_lock;
video_set_drvdata(vdev, vou_dev);
```

重点：

- `v4l2_dev` 绑上
- `lock` 绑上
- 用 `video_set_drvdata()` 反挂驱动私有数据

### 3.4 初始化 `vb2_queue`

- `drivers/media/platform/sh_vou.c:1293` 到 `1303`

绑定了：

- queue 类型
- IO 模式
- `drv_priv`
- `vb2_ops`
- `mem_ops`
- 锁

然后：

```c
ret = vb2_queue_init(q);
```

### 3.5 初始化下游 subdev

```c
subdev = v4l2_i2c_new_subdev_board(...)
```

说明这个 output 驱动不是纯粹孤立节点，它还会带一个下游编码器/显示相关子设备。

### 3.6 最后注册 `/dev/videoX`

- `drivers/media/platform/sh_vou.c:1330`

```c
ret = video_register_device(vdev, VFL_TYPE_VIDEO, -1);
```

这是 probe 的收尾动作，也是对用户态真正可见的时刻。

## 4. 它的 `open/release` 很值得看

### 4.1 `sh_vou_open()`

- `drivers/media/platform/sh_vou.c:1107`

核心动作：

1. `v4l2_fh_open(file)`
2. 如果是第一次打开，就拉起 runtime PM
3. 调 `sh_vou_hw_init()` 初始化硬件
4. 设备状态从 `INITIALISING` 切到 `IDLE`

这说明它把“硬件真正启用”延后到了第一次 open，而不是 probe 时就彻底跑起来。

### 4.2 `sh_vou_release()`

- `drivers/media/platform/sh_vou.c:1131`

核心动作：

1. 判断是不是最后一个 fd
2. `_vb2_fop_release(file, NULL)`
3. 最后一个 fd 关闭时，关输出、`pm_runtime_put()`

这是一种很常见的生命周期管理模式。

## 5. 它的 `vb2_ops` 非常适合入门

### 5.1 `queue_setup`

- `drivers/media/platform/sh_vou.c:237`

根据当前分辨率/格式计算单 plane buffer 大小。

### 5.2 `buf_prepare`

- `drivers/media/platform/sh_vou.c:254`

检查用户 buffer 是否足够大，并设置 payload。

### 5.3 `buf_queue`

- `drivers/media/platform/sh_vou.c:275`

把 buffer 放到驱动自己的 `buf_list`。

### 5.4 `start_streaming`

- `drivers/media/platform/sh_vou.c:287`

关键动作：

- `v4l2_device_call_until_err(..., video, s_stream, 1)`
- 选首 buffer
- 写硬件寄存器
- 开中断
- 设备进入 `RUNNING`

### 5.5 `stop_streaming`

- `drivers/media/platform/sh_vou.c:335`

关键动作：

- 下发 `s_stream(0)`
- 停输出
- 清理所有待处理 buffer
- 用 `VB2_BUF_STATE_ERROR` 回退未完成 buffer

### 5.6 用 5 条关键链把 `vb2` 走通

[[04-vb2缓冲队列机制#10. `vb2` 和 `v4l2_ioctl_ops` 的关系]]

`sh_vou` 里，`q->ops` 和 `q->mem_ops` 在 `probe` 填 `vb2_queue` 时就已经固定好：

- `q->ops = &sh_vou_qops`
- `q->mem_ops = &vb2_dma_contig_memops`

所以后面的 `REQBUFS/QBUF/DQBUF/STREAMON/STREAMOFF`，并不是临时去搜索有哪些 `vb2_ops` 要执行，而是顺着这条已经配置好的 `vb2_queue` 往下走。

这 5 条链可以分别对应到：

- `REQBUFS`：`queue_setup` 和 `mem_ops`
- `QBUF`：`buf_prepare` 和 `buf_queue`
- `STREAMON`：`start_streaming`
- `DQBUF`：`wait_prepare` / `wait_finish` / `vb2_buffer_done`
- `STREAMOFF`：`stop_streaming`

#### 从 REQBUFS 到 sh_vou_queue_setup 的实际调用链

```text
VIDIOC_REQBUFS
-> v4l2_ioctl_ops->vidioc_reqbufs
-> vb2_ioctl_reqbufs
-> vb2_core_reqbufs
-> call_qop(q, queue_setup, ...)
-> sh_vou_queue_setup()
```

这里的 `call_qop(q, queue_setup, ...)` 本质上就是对 `q->ops->queue_setup(...)` 的宏包装，所以源码里经常直接搜不到完整的 `q->ops->queue_setup`。

`sh_vou_queue_setup()` 只负责谈 buffer 规格：

- 确定 plane 数量
- 计算 `sizes[0]`

这一步还不会真正进入硬件队列。后续 buffer 背后的内存后端准备，继续由 `q->mem_ops` 参与；真正开始进入驱动私有队列和硬件启动，要到后面的 `QBUF` 和 `STREAMON`。

#### 从 QBUF 到 sh_vou_buf_queue 的实际调用链

```text
VIDIOC_QBUF
-> v4l2_ioctl_ops->vidioc_qbuf
-> vb2_ioctl_qbuf
-> vb2_core_qbuf
-> sh_vou_buf_prepare()
-> sh_vou_buf_queue()
```

`QBUF` 的核心不是启动硬件，而是把一个用户 buffer 送进 `vb2` 队列。

其中：

- `sh_vou_buf_prepare()` 负责检查 buffer 是否足够大，并设置 payload
- `sh_vou_buf_queue()` 负责把 buffer 挂到驱动自己的 `buf_list`

所以 `QBUF` 结束后，buffer 已经进入驱动私有等待队列，但硬件未必已经开始消费它。

#### 从 STREAMON 到 sh_vou_start_streaming 的实际调用链

```text
VIDIOC_STREAMON
-> v4l2_ioctl_ops->vidioc_streamon
-> vb2_ioctl_streamon
-> vb2_core_streamon
-> sh_vou_start_streaming()
```

`STREAMON` 才是真正让队列开始跑起来的时刻。

在 `sh_vou_start_streaming()` 里，关键动作是：

- 对下游 subdev 下发 `s_stream(1)`
- 选出首个 buffer
- 写硬件寄存器
- 开中断

所以 `STREAMON` 之前更像是在做排队准备，`STREAMON` 之后才真正进入硬件输出状态。

#### 从 DQBUF 到 vb2_buffer_done 的实际调用链

```text
VIDIOC_DQBUF
-> v4l2_ioctl_ops->vidioc_dqbuf
-> vb2_ioctl_dqbuf
-> vb2_core_dqbuf
-> wait_prepare()
-> 睡眠等待
-> 中断里 vb2_buffer_done()
-> wait_finish()
-> 返回完成 buffer
```

`DQBUF` 的关键点是：如果当前还没有完成的 buffer，就会进入阻塞等待路径。

这时：

- `wait_prepare = vb2_ops_wait_prepare`
- `wait_finish = vb2_ops_wait_finish`

它们负责等待前后的通用锁处理。  
真正把一个 buffer 标成“已经完成”的关键动作，不在 `DQBUF` 本身，而在中断完成路径里的 `vb2_buffer_done()`。

#### 从 STREAMOFF 到 sh_vou_stop_streaming 的实际调用链

```text
VIDIOC_STREAMOFF
-> v4l2_ioctl_ops->vidioc_streamoff
-> vb2_ioctl_streamoff
-> vb2_core_streamoff
-> sh_vou_stop_streaming()
```

`STREAMOFF` 的重点不是“简单停一个标志位”，而是要把队列完整收尾。

在 `sh_vou_stop_streaming()` 里，关键动作是：

- 对下游 subdev 下发 `s_stream(0)`
- 停输出
- 清理驱动私有链表
- 把还没正常完成的 buffer 以 `VB2_BUF_STATE_ERROR` 回退

职责可以压成三句：

- `queue_setup`：谈 buffer 规格
- `mem_ops`：准备 buffer 背后的内存后端
- `buf_queue/start_streaming/stop_streaming`：把通用 `vb2` 队列真正接到驱动私有队列和硬件生命周期上

## 6. 中断完成路径也很标准

前面的 `REQBUFS/QBUF/STREAMON` 主要是在讲：

- buffer 怎么建立
- buffer 怎么入队
- 硬件怎么启动

但一个完整的 streaming 驱动还必须有一条“完成路径”。  
否则 buffer 只能送进硬件，不能从硬件正确回到 `vb2`，用户态的 `DQBUF` 也就不会真正返回。

对单节点驱动来说，完整主线至少要包含两条路径：

- 提交路径
  - `QBUF -> buf_prepare -> buf_queue`
  - `STREAMON -> start_streaming -> 硬件开始工作`
- 完成路径
  - `硬件完成中断 -> 驱动 IRQ handler -> vb2_buffer_done() -> 用户态 DQBUF 返回`

所以中断完成路径不是附属细节，而是单节点 `video_device + vb2_queue` 模型的另一半。

### 6.1 `sh_vou` 里的完成路径主线

`sh_vou` 的完成路径可以收成：

```text
硬件输出完成中断
-> 找当前 active buffer
-> 写 timestamp / sequence
-> vb2_buffer_done(..., VB2_BUF_STATE_DONE)
-> 切到下一块 buffer
-> 唤醒 DQBUF 等待方
```

这里最关键的动作是 `vb2_buffer_done()`。  
它的意义不是“顺手通知一下”，而是正式告诉 `vb2`：

- 这块 buffer 已经处理完成
- 状态可以从驱动持有转为可返回用户态

如果少了这一步：

- buffer 会一直停留在驱动或 `vb2` 的进行中状态
- 用户态 `DQBUF` 就可能一直阻塞

### 6.2 为什么 `DQBUF` 一定要和中断完成路径连起来看

前面 `5.6` 里已经把 `DQBUF` 收成一条链：

```text
VIDIOC_DQBUF
-> vb2_ioctl_dqbuf
-> vb2_core_dqbuf
-> wait_prepare()
-> 睡眠等待
-> 中断里 vb2_buffer_done()
-> wait_finish()
-> 返回完成 buffer
```

这条链说明：

- `DQBUF` 本身不负责“制造完成 buffer”
- `DQBUF` 只是到 `vb2` 里取“已经完成”的 buffer
- 如果当前还没有完成的 buffer，它就只能睡眠等待
- 真正把 buffer 变成“可出队”的关键动作，在中断完成路径里的 `vb2_buffer_done()`

所以：

- `QBUF/STREAMON` 解决的是“怎么把 buffer 送进硬件”
- `IRQ/vb2_buffer_done/DQBUF` 解决的是“怎么把完成后的 buffer 还给用户”

### 6.3 用 `sh_vou` 看单节点 streaming 驱动的完整闭环

如果把前面几节和这里合起来，`sh_vou` 的完整闭环可以压成：

```text
REQBUFS
-> 建 queue 和 buffer 地基

QBUF
-> buffer 进入驱动私有等待队列

STREAMON
-> 启动硬件和中断

硬件处理中
-> 中断完成
-> vb2_buffer_done()

DQBUF
-> 用户态取回已完成 buffer

QBUF
-> 再次把 buffer 送回队列

STREAMOFF
-> 停硬件并清理未完成 buffer
```

这正是学习“一个单节点 V4L2 streaming 驱动如何完成整条 buffer 生命周期”的好样板。

## 7. 用它能学到什么

### 7.1 一个独立 `video_device` 驱动的最小骨架

- `v4l2_device`
- `video_device`
- `vb2_queue`
- IRQ 完成路径

### 7.2 `ioctl_ops` 和 `vb2` 怎么配合

格式/能力相关 ioctl 由驱动自己处理，缓冲队列 ioctl 大量复用 `vb2_ioctl_*`。

### 7.3 open/release 和 runtime PM 怎么衔接

这对很多真实硬件驱动都非常有参考价值。

## 8. 阅读建议

顺着源码阅读时，可按下面顺序展开：

1. `sh_vou_video_template`
2. `sh_vou_fops`
3. `sh_vou_ioctl_ops`
4. `sh_vou_probe()`
5. `sh_vou_qops`
6. 中断函数里的 `vb2_buffer_done()`

本章放在 [[04-vb2缓冲队列机制]] 之后。  
因为前四章先把：

- `video_device`
- `open/ioctl`
- `vb2`

这些基础对象和主线搭起来以后，再看 `sh_vou`，单节点闭环会非常清楚。  
这一章读完后，再进入 [[06-Media-Controller框架总览]] 和 [[07-entity-pad-link-pipeline主线]]，节奏会更顺。

## 9. 这个例子的局限

它是一个 output 驱动，不是 camera capture host。  
所以它不会覆盖：

- sensor 异步匹配
- CSI/ISP 多 subdev 管线
- 复杂媒体拓扑协商

但如果目标是先看懂 V4L2 驱动主骨架，它反而比大型 capture 驱动更合适。
