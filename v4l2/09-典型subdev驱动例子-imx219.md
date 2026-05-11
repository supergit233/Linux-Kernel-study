# 典型 `subdev` 驱动例子：`imx219.c`

## 导读

### 本章定位

这一章用 `imx219.c` 作为标准 sensor `subdev` 样例，把 `08` 里的对象和流程真正落到驱动代码里，重点是看一个 sensor 驱动怎样初始化 `subdev`、controls、pad 和 async 注册。

### 核心对象

- `struct v4l2_subdev`
  - sensor 在 V4L2 里的核心对象
- `struct media_pad`
  - sensor 对外的 source pad
- `struct v4l2_ctrl_handler`
  - controls 管理对象
- `struct v4l2_subdev_ops`
  - sensor 各类回调的入口表

### 关键函数

- `v4l2_i2c_subdev_init()`
- `imx219_init_controls()`
- `media_entity_pads_init()`
- `v4l2_async_register_subdev_sensor_common()`
- `imx219_probe()`

### 主流程

分配私有结构 -> 初始化 I2C subdev -> 校验硬件配置 -> 初始化 controls -> 初始化 `entity/pad` -> async 注册

这一章放在 [[06-Media-Controller框架总览]]、[[07-entity-pad-link-pipeline主线]]、[[08-subdev与异步注册]] 之后。  
先把：

- `media_device`
- `media_entity`
- `media_pad`
- `media_link`
- `subdev`

这几层对象和主线理顺以后，再回到 `imx219`，就会更容易看清 sensor 驱动在整条管线里的职责边界。

## 1. 为什么选 `imx219.c`

示例文件：

- `drivers/media/i2c/imx219.c`

它是一个标准主线 camera sensor 驱动，适合拿来理解：

- `v4l2_subdev`
- `pad ops`
- control handler
- 异步注册
- sensor 驱动和 host 驱动的职责边界

## 2. 先看它的核心回调表

### 2.1 `v4l2_subdev_ops`

- `drivers/media/i2c/imx219.c:1211`

```c
static const struct v4l2_subdev_ops imx219_subdev_ops = {
    .core  = &imx219_core_ops,
    .video = &imx219_video_ops,
    .pad   = &imx219_pad_ops,
};
```

这表明它的关注点是：

- 核心控制
- 视频流开关
- pad 级格式协商

### 2.2 `v4l2_subdev_internal_ops`

- `drivers/media/i2c/imx219.c:1217`

它只实现了：

- `.open = imx219_open`

说明这个 sensor 需要在 subdev 文件句柄打开时准备默认格式之类的状态。

## 3. control 初始化是 sensor 驱动的重点

入口：

- `drivers/media/i2c/imx219.c:1222`
  `imx219_init_controls()`

第一步：

- `drivers/media/i2c/imx219.c:1232`
  `v4l2_ctrl_handler_init(ctrl_hdlr, 11)`

后面依次建立：

- `V4L2_CID_PIXEL_RATE`
- `V4L2_CID_VBLANK`
- `V4L2_CID_HBLANK`
- `V4L2_CID_EXPOSURE`
- `V4L2_CID_ANALOGUE_GAIN`
- `V4L2_CID_DIGITAL_GAIN`
- `V4L2_CID_HFLIP`
- `V4L2_CID_VFLIP`
- `V4L2_CID_TEST_PATTERN`

理解重点：

- sensor 驱动很多“用户可调参数”最终都映射成 V4L2 controls
- control handler 通常会挂到 `sd.ctrl_handler`

## 4. probe 主线怎么走

入口：

- `drivers/media/i2c/imx219.c:1385` 左右
  `imx219_probe()`

这一段可拆成 10 步。

### 4.1 分配私有结构

```c
imx219 = devm_kzalloc(...)
```

### 4.2 初始化为 I2C subdev

- `drivers/media/i2c/imx219.c:1394`

```c
v4l2_i2c_subdev_init(&imx219->sd, client, &imx219_subdev_ops);
```

这一步非常关键，它把 sensor 驱动正式放进 V4L2 `subdev` 模型。

### 4.3 检查硬件配置

`imx219_check_hwcfg()` 会解析 fwnode / DT endpoint，确认：

- 使用的是 `V4L2_MBUS_CSI2_DPHY`
- data lane 数匹配
- `link-frequency` 合法

这类检查在现代 sensor 驱动里非常常见。

### 4.4 获取时钟、电源、GPIO

包括：

- `xclk`
- regulators
- reset GPIO

这一步还是普通硬件驱动的基本功。

### 4.5 上电并识别芯片

probe 里会：

- `imx219_power_on()`
- `imx219_identify_module()`

这说明 sensor 驱动很多时候在 probe 就要和硬件真实通信。

### 4.6 选择缺省 mode

```c
imx219->mode = &supported_modes[0];
```

后面的 controls 和默认 format 都基于当前 mode。

### 4.7 初始化 controls

- `drivers/media/i2c/imx219.c`
  `imx219_init_controls(imx219)`

### 4.8 初始化 subdev 属性

```c
imx219->sd.internal_ops = &imx219_internal_ops;
imx219->sd.flags |= V4L2_SUBDEV_FL_HAS_DEVNODE;
imx219->sd.entity.function = MEDIA_ENT_F_CAM_SENSOR;
```

意义分别是：

- 绑定 internal ops
- 允许后续生成 `v4l-subdevX`
- 把 entity 标成 camera sensor

### 4.9 初始化 pad 和默认格式

```c
imx219->pad.flags = MEDIA_PAD_FL_SOURCE;
imx219_set_default_format(imx219);
media_entity_pads_init(&imx219->sd.entity, 1, &imx219->pad);
```

这说明它是一个只有 **一个 source pad** 的 sensor。

### 4.10 异步注册

- `drivers/media/i2c/imx219.c:1477`

```c
ret = v4l2_async_register_subdev_sensor_common(&imx219->sd);
```

这里就是 sensor 驱动和 host 驱动真正接轨的地方。

## 5. 它为什么不直接注册 `/dev/videoX`

这是看 sensor 驱动时最常见的疑问。

答案是：

- `imx219` 只是图像源头
- 它不负责 DMA capture
- 它不负责最终用户态视频节点

所以它只注册 `subdev`，把：

- 时序
- 格式
- controls
- `s_stream`

这些能力提供给上游 host/capture 驱动。

真正的 `/dev/videoX` 往往由：

- CSI host
- ISP capture
- DMA engine

那一端的驱动注册。

## 6. `pad ops` 才是这个驱动的关键接口之一

`imx219_pad_ops` 里包含：

- `enum_mbus_code`
- `get_fmt`
- `set_fmt`
- `get_selection`
- `enum_frame_size`

这反映出 sensor 驱动的核心职责：

- 向 host 报告支持哪些总线格式
- 支持哪些分辨率
- 当前 pad 格式是什么

而不是直接对用户态处理 `VIDIOC_S_FMT`。

## 7. `s_stream` 的意义

虽然这里没逐行展开 `imx219_video_ops`，但对 sensor 驱动来说：

- `video.s_stream(1)`
  代表开始出流
- `video.s_stream(0)`
  代表停止出流

通常是由 host 驱动在 `STREAMON/STREAMOFF` 前后调用的。

sensor `subdev` 的定位可以收成：

- 自己不直接管理用户态 buffer
- 但负责让硬件“开始吐像素”或“停止吐像素”

## 8. remove 路径也很标准

- `drivers/media/i2c/imx219.c:1504` 左右
  `imx219_remove()`

它会：

1. `v4l2_async_unregister_subdev(sd)`
2. `media_entity_cleanup(&sd->entity)`
3. 释放 controls
4. 关闭 runtime PM
5. 必要时断电

这条链体现了 `subdev` 驱动的拆卸顺序。

## 9. 用这个例子能学到什么

### 9.1 sensor 驱动的职责边界

它负责：

- 时钟/电源/GPIO
- mode 选择
- controls
- pad format
- `s_stream`

它不负责：

- `/dev/videoX`
- buffer 队列
- `REQBUFS/QBUF/DQBUF`

### 9.2 为什么 camera 驱动总是分层

因为 sensor 和 capture 天然就不是同一个层次的问题。

### 9.3 异步注册为什么是标配

没有 notifier 机制，sensor 和 host 的 probe 顺序会变成系统稳定性的雷区。

## 10. 一句话总结

`imx219.c` 代表的是 **“一个标准 V4L2 sensor subdev 怎么写”**。  
如果把 `sh_vou.c` 对应为 `/dev/videoX` 这一层，`imx219.c` 对应的就是媒体管线内部“源头设备”这一层。
