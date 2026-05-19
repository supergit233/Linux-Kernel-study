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
- `struct v4l2_ctrl`
  - 单个曝光、增益、翻转等参数对象
- `struct v4l2_ctrl_ops`
  - control 框架回调表，最终把参数落到 sensor 驱动
- `struct v4l2_subdev_ops`
  - sensor 各类回调的入口表

### 关键函数

- `v4l2_i2c_subdev_init()`
- `imx219_init_controls()`
- `v4l2_s_ctrl()`
- `imx219_set_ctrl()`
- `__v4l2_ctrl_handler_setup()`
- `imx219_write_reg()`
- `media_entity_pads_init()`
- `v4l2_async_register_subdev_sensor_common()`
- `imx219_probe()`

### 主流程

分配私有结构 -> 初始化 I2C subdev -> 校验硬件配置 -> 初始化 controls -> 初始化 `entity/pad` -> async 注册 -> 用户态调参进入 control 框架 -> `s_ctrl` 写 sensor 寄存器

这一章放在 [[06-Media-Controller框架总览]]、[[07-entity-pad-link-pipeline主线]]、[[08-subdev与异步注册]] 之后。  
先把：

- `media_device`
- `media_entity`
- `media_pad`
- `media_link`
- `subdev`

这几层对象和主线理顺以后，再回到 `imx219`，就会更容易看清 sensor 驱动在整条管线里的职责边界。

本章按“先看 sensor 建模，再看参数控制，再回到 probe 和起流”的顺序展开。这样拆分，是因为 sensor 不只负责注册成 `subdev`，还要负责把曝光、增益、翻转等运行时参数真正落到寄存器；只看 probe 而不看 control 路径，会漏掉 sensor 驱动最常被业务层触碰的一半职责。

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

## 3. control 框架是 sensor 参数调节的主线

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

### 3.1 先把三个 control 对象分开

这一层最容易混在一起的，不是函数，而是对象。

| 对象 | `imx219` 中从哪里来 | 负责什么 | 调试时先看什么 |
| --- | --- | --- | --- |
| `struct v4l2_ctrl_handler` | `imx219->ctrl_handler`，由 `v4l2_ctrl_handler_init()` 初始化，随后挂到 `imx219->sd.ctrl_handler` | 保存这一组 controls、引用关系和锁，是 control 框架查找参数的入口 | `sd.ctrl_handler` 是否已挂好，handler 是否有 `error` |
| `struct v4l2_ctrl` | 由 `v4l2_ctrl_new_std()` / `v4l2_ctrl_new_std_menu_items()` 创建 | 表示一个具体参数，例如 `V4L2_CID_EXPOSURE`，保存 `id`、范围、当前值和所属 handler | 目标 control 是否存在，`id/min/max/step/val` 是否符合预期 |
| `struct v4l2_ctrl_ops` | `imx219_ctrl_ops` | 保存 `s_ctrl` 等回调，决定参数变化后如何落到驱动 | `.s_ctrl` 是否指向 `imx219_set_ctrl()` |

因此 `imx219_init_controls()` 不只是“创建一串参数名”，而是在建立下面这层关系：

```text
imx219 private struct
-> ctrl_handler                         // 管一组 control
   -> exposure / analogue_gain / ...    // 每个具体 control
      -> imx219_ctrl_ops.s_ctrl          // 参数变化后回调到 sensor 驱动
```

`v4l2_ctrl_handler` 是容器，`v4l2_ctrl` 是参数对象，`v4l2_ctrl_ops` 是回调表。  
这三者正好对应“参数归谁管、参数本身是什么、参数变化后谁执行”三个问题。

### 3.2 `imx219` 里哪些 control 会真正写寄存器

`imx219_set_ctrl()` 的源码入口是：

- `drivers/media/i2c/imx219.c:621`

这里已经由源码确认的典型映射关系是：

| control | sensor 寄存器 | 写入宽度 | 含义 |
| --- | --- | --- | --- |
| `V4L2_CID_EXPOSURE` | `IMX219_REG_EXPOSURE = 0x015a` | 16 bit | 曝光 |
| `V4L2_CID_ANALOGUE_GAIN` | `IMX219_REG_ANALOG_GAIN = 0x0157` | 8 bit | 模拟增益 |
| `V4L2_CID_DIGITAL_GAIN` | `IMX219_REG_DIGITAL_GAIN = 0x0158` | 16 bit | 数字增益 |
| `V4L2_CID_VBLANK` | `IMX219_REG_VTS` | 16 bit | 帧长相关控制 |
| `V4L2_CID_HFLIP` / `V4L2_CID_VFLIP` | `IMX219_REG_ORIENTATION` | 8 bit | 方向翻转 |

这一步把“V4L2 参数名”落回了“真实 sensor 寄存器”。例如曝光并不是一个抽象配置最终停在框架里，而是会写到 `0x015a`。

### 3.3 从用户态到 `imx219_set_ctrl()` 的两种入口

参数调整可以从两类节点进入，但两者不是同一条路。

#### 3.3.1 直接对 `/dev/v4l-subdevX` 设置 sensor control

如果 host 为 sensor 注册了 subdev devnode，路径最直接：

```text
用户态 ioctl(fd, VIDIOC_S_CTRL, V4L2_CID_EXPOSURE) // 用户提交曝光值
-> v4l2_subdev_fops.unlocked_ioctl = subdev_ioctl  // subdev 节点接住 ioctl
-> video_usercopy(..., subdev_do_ioctl_lock)       // 统一做用户参数拷贝
-> subdev_do_ioctl()                               // 识别 VIDIOC_S_CTRL
-> v4l2_s_ctrl(vfh, vfh->ctrl_handler, arg)        // 进入 control core
-> v4l2_ctrl_find() / set_ctrl_lock()              // 找 control、加锁、准备新值
-> try_or_set_cluster()                            // 校验 cluster 并决定是否真正设置
-> imx219_ctrl_ops.s_ctrl = imx219_set_ctrl()      // 回调 sensor 驱动
-> imx219_write_reg()                              // 组装寄存器地址和值
-> i2c_master_send()                               // 通过 I2C 写入 IMX219
```

这条链已经由源码确认：

- `v4l2_subdev_fops`：`drivers/media/v4l2-core/v4l2-subdev.c:732`
- `VIDIOC_S_CTRL -> v4l2_s_ctrl()`：`drivers/media/v4l2-core/v4l2-subdev.c:402`
- `v4l2_s_ctrl()`：`drivers/media/v4l2-core/v4l2-ctrls.c:4287`
- `try_or_set_cluster()`：`drivers/media/v4l2-core/v4l2-ctrls.c:3914`
- `imx219_set_ctrl()`：`drivers/media/i2c/imx219.c:621`
- `imx219_write_reg()`：`drivers/media/i2c/imx219.c:518`

subdev devnode 之所以能直接找到 sensor 的 control handler，是因为 `__v4l2_device_register_subdev_nodes()` 创建 `v4l-subdevX` 时，会把：

```c
vdev->ctrl_handler = sd->ctrl_handler;
```

挂到 subdev 对应的 `video_device` 上。随后 `v4l2_fh_init()` 再让文件句柄继承这个 handler。

#### 3.3.2 通过 `/dev/videoX` 设置时，前面还隔着 host

如果用户对 `/dev/videoX` 调 `VIDIOC_S_CTRL`，前半段会走 [[03-ioctl派发与v4l2_ioctl_ops]] 讲过的 video node 路径：

```text
用户态 ioctl(fd, VIDIOC_S_CTRL, ...)
-> v4l2_ioctl()
-> video_ioctl2()
-> __video_do_ioctl()
-> v4l_s_ctrl()
-> v4l2_s_ctrl(...)
```

但这条路径能不能最终调到 `imx219_set_ctrl()`，取决于 host 是否把 sensor controls 暴露到 video node：

- `v4l2_device_register_subdev()` 会尝试把 `sd->ctrl_handler` 通过 `v4l2_ctrl_add_handler()` 合并到 `v4l2_dev->ctrl_handler`
- `__video_register_device()` 在 `vdev->ctrl_handler` 为空时，会让 video node 继承 `vdev->v4l2_dev->ctrl_handler`
- 因此，只有 host 建好了自己的 handler 关系，`/dev/videoX` 上的 control ioctl 才可能继续命中 sensor controls

所以不能简单记成：

```text
/dev/videoX 的 VIDIOC_S_CTRL 一定直接进 sensor
```

更准确的说法是：

```text
imx219 必然拥有自己的 sd.ctrl_handler；
video node 是否也能操作这些 control，取决于 host 有没有把这组 handler 合并或转发出去。
```

这一点和 [[09-1-参数调整附录]] 的整体思路一致，但主章需要把“直接 subdev 路径”和“经 host 暴露的 video node 路径”明确分开。

### 3.4 为什么设置成功后不一定立刻写硬件

`imx219_set_ctrl()` 里有一个很关键的判断：

```c
if (pm_runtime_get_if_in_use(&client->dev) == 0)
    return 0;
```

它表示：

- 如果 sensor 当前没处于上电使用状态，control 框架先保存新值
- 此时不强行唤醒硬件，也不立刻写寄存器
- 等后面真正起流时，再统一把已经保存的 control 值写入硬件

后半段由 `imx219_start_streaming()` 完成：

```text
imx219_set_stream(1)                         // host 请求 sensor 起流
-> imx219_start_streaming()                  // 写 mode 表和当前格式
-> __v4l2_ctrl_handler_setup(sd.ctrl_handler) // 重新把已保存 controls 回放给驱动
-> imx219_set_ctrl()                         // 按当前值写曝光、增益等寄存器
-> IMX219_REG_MODE_SELECT = STREAMING       // sensor 正式出流
```

这解释了一个常见现象：

- 用户态先设置曝光，ioctl 返回成功
- 但在 sensor 尚未 streaming 时，I2C 总线上不一定马上看到曝光寄存器写入
- 真正起流时，`__v4l2_ctrl_handler_setup()` 会把这些值重新应用到硬件

把这点补上以后，control 路径才算从“参数保存”讲到了“硬件生效”。

## 4. probe 主线怎么走

入口：

- `drivers/media/i2c/imx219.c:1385` 左右
  `imx219_probe()`

这一节按 `probe` 的真实职责展开：先从总线匹配进入驱动，再拿板级资源并确认硬件真实存在，之后把 sensor 建模成 V4L2 `subdev`，最后挂入 Media Controller / async 框架。  
`probe()` 的目标不是开始采图，而是让系统确认“这个 sensor 存在、资源可用、模型已建立、后续可由 host 启动”。

### 4.1 先把 `probe()` 总线压平

```text
I2C match -> imx219_probe()                                  // I2C core 匹配到设备和驱动后进入 sensor probe
-> devm_kzalloc(sizeof(*imx219))                             // 分配驱动私有结构，保存 sd、controls、mode、资源句柄
-> v4l2_i2c_subdev_init(&imx219->sd, client, &ops)           // 把私有结构里的 sd 初始化成 I2C subdev
-> imx219_check_hwcfg(dev)                                   // 解析并校验 DT/fwnode endpoint 描述
-> devm_clk_get(dev, NULL) / clk_get_rate()                  // 获取并校验 xclk，要求 24MHz
-> imx219_get_regulators(imx219)                             // 获取 VANA/VDIG/VDDL 三路 regulator
-> devm_gpiod_get_optional(dev, "reset", GPIOD_OUT_HIGH)     // 获取可选 reset GPIO
-> imx219_power_on(dev)                                      // 打开电源、时钟并释放 reset，使 I2C 可访问
-> imx219_identify_module(imx219)                            // 读取 CHIP_ID，确认真实硬件是 IMX219
-> imx219->mode = &supported_modes[0]                        // 选择缺省 mode 作为后续 controls/format 的基准
-> IMX219_MODE_STREAMING -> IMX219_MODE_STANDBY              // 为 LP-11 状态做一次模式切换，最终仍回到 standby
-> imx219_init_controls(imx219)                              // 基于当前 mode 建立曝光、增益、blanking 等 controls
-> sd.internal_ops / sd.flags / sd.entity.function           // 补齐 subdev 内部回调、devnode 标志和 entity 类型
-> imx219->pad.flags = MEDIA_PAD_FL_SOURCE                   // sensor 只有一个 source pad
-> imx219_set_default_format(imx219)                         // 设置默认 mbus format、宽高和 colorspace
-> media_entity_pads_init(&sd.entity, 1, &imx219->pad)       // 建立 Media Controller entity/pad 关系
-> v4l2_async_register_subdev_sensor_common(&sd)             // 把 sensor subdev 交给 async 框架等待 host 绑定
-> pm_runtime_set_active() / pm_runtime_enable() / idle()    // 开启 runtime PM，probe 结束后允许设备空闲下电
```

这条线说明：`probe()` 是“建模和接入阶段”，不是“持续出流阶段”。真正输出图像要等 host/capture 驱动在用户态 `STREAMON` 后调用 `s_stream(1)`。

### 4.2 设备树资源是怎么拿的

`imx219_probe()` 不是用一个函数一次性把所有设备树资源拿完，而是按资源类型分散在几个阶段处理：

| 资源类型            | 源码入口                                         | 来自哪里                               | 起什么作用                                                     |
| --------------- | -------------------------------------------- | ---------------------------------- | --------------------------------------------------------- |
| endpoint / 总线连接 | `imx219_check_hwcfg()`                       | fwnode / DT graph endpoint         | 描述 sensor 输出连接到哪类总线、几条 MIPI data lanes、link frequency 是多少 |
| 外部时钟 `xclk`     | `devm_clk_get(dev, NULL)`                    | clock provider / DT clocks         | 给 sensor 提供工作时钟，本驱动要求 `IMX219_XCLK_FREQ = 24000000`       |
| 电源 regulator    | `imx219_get_regulators()`                    | regulator provider / DT supplies   | 获取 `VANA`、`VDIG`、`VDDL` 三路电源                              |
| reset GPIO      | `devm_gpiod_get_optional(dev, "reset", ...)` | GPIO provider / DT reset-gpios     | 控制 sensor 复位脚，可选存在                                        |
| camera 方向属性     | `v4l2_fwnode_device_parse()`                 | fwnode 中可选的 V4L2 device properties | 如果平台描述提供 orientation / rotation 这类属性，可转换成 controls        |

这几个资源的层次不同：

- clock、regulator、GPIO 是“板级硬件资源”
- endpoint 是“媒体链路拓扑资源”
- orientation / rotation 是“设备属性资源”，如果平台描述提供，最终可被转成 V4L2 controls

所以看到 `devm_clk_get()`、`devm_regulator_bulk_get()`、`fwnode_graph_get_next_endpoint()` 时，不能把它们都混成“读设备树”。更准确地说，是 Linux 不同子系统从同一个固件描述入口消费不同类型的信息。

把 `Documentation/devicetree/bindings/media/i2c/imx219.yaml` 里的例子压成学习版，大致是这个形状：

```dts
imx219: sensor@10 {
        compatible = "sony,imx219";                 // OF/I2C 匹配入口，命中 imx219_i2c_driver
        reg = <0x10>;                               // I2C 地址，生成 i2c_client->addr
        clocks = <&imx219_clk>;                     // devm_clk_get(dev, NULL) 获取 xclk
        VANA-supply = <&imx219_vana>;               // imx219_get_regulators() 获取 2.8V 模拟电源
        VDIG-supply = <&imx219_vdig>;               // imx219_get_regulators() 获取 1.8V I/O 电源
        VDDL-supply = <&imx219_vddl>;               // imx219_get_regulators() 获取 1.2V core 电源
        reset-gpios = <&gpioX Y GPIO_ACTIVE_LOW>;   // devm_gpiod_get_optional(dev, "reset", ...)

        port {
                imx219_out: endpoint {
                        remote-endpoint = <&csi_in>;                // 指向 host/CSI 接收端 endpoint
                        data-lanes = <1 2>;                         // imx219_check_hwcfg() 校验为 2 lanes
                        clock-noncontinuous;                        // 描述 MIPI CSI-2 clock 行为
                        link-frequencies = /bits/ 64 <456000000>;   // 校验为 IMX219_DEFAULT_LINK_FREQ
                };
        };
};
```

这段 DTS 不是让 `probe()` 一次性“读完整个节点”，而是被分阶段消费：

```text
compatible / reg                               // 先由 OF/I2C core 创建并匹配 i2c_client
-> imx219_probe(client)                        // 匹配成功后进入 sensor 驱动
-> fwnode_graph_get_next_endpoint()            // sensor 驱动取 graph endpoint
-> v4l2_fwnode_endpoint_alloc_parse()          // V4L2 fwnode 层解析 lanes、link-frequency 等总线参数
-> devm_clk_get() / clk_get_rate()             // clock 框架解析 clocks，并校验 24MHz
-> devm_regulator_bulk_get()                   // regulator 框架解析 VANA/VDIG/VDDL supply
-> devm_gpiod_get_optional()                   // GPIO 描述符框架解析 reset-gpios
```

### 4.3 endpoint 是什么，怎么起作用

`endpoint` 是设备树 graph 里的端点描述，用来表达媒体链路两端怎么连接。  
对 sensor 来说，它描述的是“sensor 输出端怎样接到 CSI receiver / host”。

`imx219_check_hwcfg()` 的源码逻辑是：

```text
fwnode_graph_get_next_endpoint(dev_fwnode(dev), NULL)     // 找到 sensor 节点下的第一个 endpoint
-> v4l2_fwnode_endpoint_alloc_parse(endpoint, &ep_cfg)   // 解析 V4L2 fwnode endpoint 信息
-> ep_cfg.bus_type = V4L2_MBUS_CSI2_DPHY                 // 期望这是 MIPI CSI-2 D-PHY 总线
-> ep_cfg.bus.mipi_csi2.num_data_lanes == 2              // 只接受 2 条 data lanes
-> ep_cfg.nr_of_link_frequencies == 1                    // 必须提供 link-frequency
-> ep_cfg.link_frequencies[0] == IMX219_DEFAULT_LINK_FREQ // 只接受 456MHz link frequency
```

这一步的作用不是直接创建 `/dev/videoX`，也不是直接连好所有 media links。  
它先做 sensor 侧的硬件配置校验：设备树描述的链路能力必须和这个驱动支持的能力一致，否则 `probe()` 直接失败。

需要特别区分两个动作：

| 动作 | 谁做 | 作用 |
| --- | --- | --- |
| 校验本地 endpoint | `imx219_check_hwcfg()` | 确认 sensor 侧 lanes 和 link frequency 能被本驱动支持 |
| 根据 endpoint 建立跨设备关系 | host / bridge 驱动的 async notifier | 根据 fwnode / endpoint 关系把 sensor subdev 绑定进 host 管线 |

后续 host 驱动通常也会解析自己的 endpoint，并通过 async notifier 把两侧 `subdev` 绑定起来。  
因此 endpoint 的价值是：为 sensor 和 host 提供共同的链路描述基础。

### 4.4 怎么检查 sensor 是否真实存在

设备树里有节点，只能说明“板级描述声称这里有一个 IMX219”。  
真实芯片是否存在，要靠上电后通过 I2C 读寄存器确认。

`probe()` 里的硬件确认分两步：

```text
imx219_power_on(dev)                              // regulator_bulk_enable + clk_prepare_enable + reset_gpio 拉起
-> imx219_identify_module(imx219)                 // 通过 I2C 读 CHIP_ID
-> imx219_read_reg(IMX219_REG_CHIP_ID, 16bit)     // 读取 sensor ID 寄存器
-> val == IMX219_CHIP_ID                          // ID 匹配才认为真实 sensor 存在
```

如果读寄存器失败，说明 I2C 通信、电源、时钟、reset 或硬件连接可能有问题。  
如果读到的 `CHIP_ID` 不等于 `IMX219_CHIP_ID`，说明设备树和实际硬件不匹配，驱动返回错误。

这也是为什么 `imx219_identify_module()` 前面必须先 `imx219_power_on()`：芯片未上电或未出 reset 时，I2C 寄存器通常不可读。

### 4.5 怎么初始化 V4L2 subdev

`imx219` 的 subdev 初始化分成两个阶段。

第一阶段是把 I2C 设备变成 V4L2 subdev：

```c
v4l2_i2c_subdev_init(&imx219->sd, client, &imx219_subdev_ops);
```

这一步由 V4L2 helper 完成，核心效果是：

- 调 `v4l2_subdev_init()` 绑定 `imx219_subdev_ops`
- 设置 `V4L2_SUBDEV_FL_IS_I2C`
- 绑定 `sd->dev = &client->dev`
- 建立 `i2c_client <-> v4l2_subdev` 的双向关联
- 设置 subdev 名字和 owner

第二阶段是补齐 IMX219 作为 sensor 的属性：

```text
imx219->sd.internal_ops = &imx219_internal_ops       // subdev devnode open 时进入 imx219_open()
-> imx219->sd.flags |= V4L2_SUBDEV_FL_HAS_DEVNODE   // 允许后续创建 /dev/v4l-subdevX
-> imx219->sd.entity.function = MEDIA_ENT_F_CAM_SENSOR // 告诉 MC 这是 camera sensor entity
```

这两阶段的区别是：

- `v4l2_i2c_subdev_init()` 解决“这是一个 V4L2 I2C subdev”
- 后面的 `internal_ops / flags / entity.function` 解决“这个 subdev 在媒体框架里怎样被访问和识别”

### 4.6 缺省 mode 是什么，怎么起作用

`mode` 是 sensor 驱动内部对“一组分辨率、裁剪、时序和寄存器表”的描述。  
`imx219_mode` 里包含：

| 字段 | 含义 | 后续谁会用 |
| --- | --- | --- |
| `width / height` | 输出帧尺寸 | 默认 format、`set_fmt()`、枚举分辨率 |
| `crop` | analog crop 矩形 | `get_selection()`、裁剪能力描述 |
| `vts_def` | 默认垂直时序 | `vblank`、`exposure` 范围计算 |
| `reg_list` | 该 mode 对应的寄存器表 | `imx219_start_streaming()` 起流时写入 sensor |
| `binning` | 是否使用 2x2 binning | 起流时设置 binning |

probe 中选择：

```text
imx219->mode = &supported_modes[0]      // 选择 3280x2464 15fps 作为缺省模式
```

它会影响三个后续动作：

```text
imx219_init_controls()                  // 用 mode->height 和 mode->vts_def 算 vblank/exposure 范围
imx219_set_default_format()             // 用 supported_modes[0] 设置默认 width/height
imx219_start_streaming()                // 起流时写 imx219->mode->reg_list
```

如果用户后续通过 `VIDIOC_SUBDEV_S_FMT` 选择其他尺寸，`imx219_set_pad_format()` 会用 `v4l2_find_nearest_size()` 重新选择最接近的 mode，并同步调整 `vblank`、`hblank`、`exposure` 的范围。

所以缺省 mode 不是“永久固定模式”，而是 probe 后系统尚未收到格式请求时的初始基准。

### 4.7 controls 是怎么注册并起作用的

`imx219_init_controls()` 负责把 sensor 可调参数注册到 V4L2 control 框架。  
源码入口：

- `drivers/media/i2c/imx219.c:1222`
  `imx219_init_controls()`

这里的 `controls` 不是设备树里的一个单独节点，而是驱动在 probe 阶段创建出来的“参数对象集合”。每个 control 至少包含：

| 内容 | 含义 |
| --- | --- |
| control id | 例如 `V4L2_CID_EXPOSURE`、`V4L2_CID_ANALOGUE_GAIN` |
| 范围和步进 | 最小值、最大值、step、default value |
| 当前值 | 用户态设置后先保存在 control core 中 |
| 回调 | 值真正提交时进入 `imx219_set_ctrl()` |

对 IMX219 来说，probe 阶段注册 controls 的目的不是马上调参，而是先把“这个 sensor 支持哪些参数、每个参数范围是多少、以后怎么写寄存器”告诉 V4L2 core。

核心链路是：

```text
ctrl_hdlr = &imx219->ctrl_handler                         // 使用私有结构中的 ctrl_handler
-> v4l2_ctrl_handler_init(ctrl_hdlr, 11)                  // 初始化 handler，预计容纳一组 controls
-> ctrl_hdlr->lock = &imx219->mutex                       // 让 control 设置和 format/streaming 共用互斥保护
-> v4l2_ctrl_new_std(..., V4L2_CID_PIXEL_RATE, ...)       // 注册 pixel rate
-> v4l2_ctrl_new_std(..., V4L2_CID_VBLANK, ...)           // 注册 vblank，范围基于当前 mode
-> v4l2_ctrl_new_std(..., V4L2_CID_HBLANK, ...)           // 注册 hblank
-> v4l2_ctrl_new_std(..., V4L2_CID_EXPOSURE, ...)         // 注册曝光，最大值基于 mode->vts_def
-> v4l2_ctrl_new_std(..., V4L2_CID_ANALOGUE_GAIN, ...)    // 注册模拟增益
-> v4l2_ctrl_new_std(..., V4L2_CID_DIGITAL_GAIN, ...)     // 注册数字增益
-> v4l2_ctrl_new_std(..., V4L2_CID_HFLIP / VFLIP, ...)    // 注册翻转控制
-> v4l2_ctrl_new_std_menu_items(..., TEST_PATTERN, ...)   // 注册测试图案
-> v4l2_fwnode_device_parse()                             // 解析 fwnode 设备属性
-> v4l2_ctrl_new_fwnode_properties()                      // 把 orientation/rotation 等属性转成 controls
-> imx219->sd.ctrl_handler = ctrl_hdlr                    // 把这组 controls 挂到 subdev
```

这些 controls 的作用分两层：

- 对用户态
  - `VIDIOC_QUERYCTRL` / `VIDIOC_S_CTRL` 可以看到并设置曝光、增益、翻转等参数
- 对驱动内部
  - 参数变化最终进入 `imx219_ctrl_ops.s_ctrl = imx219_set_ctrl()`，再写寄存器或更新相关范围

需要注意，control 设置成功不等于一定立刻写硬件。  
如果 sensor 当前未 runtime active，`imx219_set_ctrl()` 会先保存值，真正起流时由 `__v4l2_ctrl_handler_setup(imx219->sd.ctrl_handler)` 统一回放到硬件。

### 4.8 怎么和 Media Controller 建立关系

```c
imx219->sd.entity.function = MEDIA_ENT_F_CAM_SENSOR;
imx219->pad.flags = MEDIA_PAD_FL_SOURCE;
media_entity_pads_init(&imx219->sd.entity, 1, &imx219->pad);
```

这三句解决的是 Media Controller 图模型里的表示问题：

- `sd.entity.function = MEDIA_ENT_F_CAM_SENSOR`
  - 表示这个 entity 是 camera sensor
- `imx219->pad.flags = MEDIA_PAD_FL_SOURCE`
  - 表示 sensor 对外只有一个 source pad，像素流从这里输出
- `media_entity_pads_init()`
  - 把这个 pad 正式挂到 `sd.entity` 上，后续 `/dev/mediaX` 才能看到 entity/pad 关系

随后：

```text
v4l2_async_register_subdev_sensor_common(&imx219->sd) // 把 sensor subdev 注册进 async 框架
-> host notifier 匹配 sensor endpoint / fwnode         // host 侧把 sensor 绑定进自己的管线
-> host 后续可注册 subdev nodes 和 video nodes         // 生成 /dev/v4l-subdevX 或 /dev/videoX
```

所以 `media_entity_pads_init()` 建的是“图里的节点和 pad”，`v4l2_async_register_subdev_sensor_common()` 解决的是“这个节点什么时候归 host 管理”。  
两者都不是直接注册 `/dev/videoX`。

### 4.9 `probe()` 里会不会直接开始出图

不会直接进入正式采集出图。

源码中确实有一段：

```text
IMX219_REG_MODE_SELECT = IMX219_MODE_STREAMING    // 短暂切到 streaming
-> usleep_range(100, 110)
-> IMX219_REG_MODE_SELECT = IMX219_MODE_STANDBY   // 立刻切回 standby
```

但这不是用户态意义上的“开始采图”。  
源码注释说明，这段是为了让 sensor 在上电后进入 LP-11 相关状态，最后仍然回到 standby。

正式出图发生在运行期：

```text
用户态 STREAMON
-> host/capture 驱动启动 pipeline
-> v4l2_subdev_call(sd, video, s_stream, 1)
-> imx219_set_stream(1)
-> imx219_start_streaming()
-> 写 common regs / mode reg_list / frame format / binning
-> __v4l2_ctrl_handler_setup(sd.ctrl_handler)
-> IMX219_REG_MODE_SELECT = IMX219_MODE_STREAMING
```

`probe()` 不直接出图的原因有三个：

- sensor 驱动不负责 DMA buffer，也没有 `/dev/videoX` 的 `REQBUFS/QBUF/DQBUF`
- host/capture 管线此时还不一定完成 async 绑定和 media link 配置
- 用户态还没有选择格式、申请 buffer、执行 `STREAMON`

因此 probe 的终点是：

```text
硬件可识别 + subdev 已注册 + media entity/pad 已建立 + controls 已准备 + runtime PM 可管理
```

而不是：

```text
sensor 已经持续输出图像并被系统采集
```

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

## 10. 全流程总结

```text
I2C match -> imx219_probe()                              // I2C 设备匹配后进入 sensor probe
-> v4l2_i2c_subdev_init()                                // 建立真实 sensor subdev 对象
-> imx219_check_hwcfg()                                  // 解析 endpoint，校验 CSI-2 lanes 和 link frequency
-> devm_clk_get() / imx219_get_regulators() / reset GPIO // 获取时钟、电源和复位 GPIO
-> imx219_power_on() -> imx219_identify_module()         // 上电后读取 CHIP_ID，确认真实 sensor 存在
-> imx219->mode = &supported_modes[0]                    // 选择缺省 mode，作为 controls 和默认 format 的基准
-> streaming -> standby                                  // 为 LP-11 状态做一次短暂模式切换，最终仍不正式出图
-> imx219_init_controls()                                // 基于 mode 建立 ctrl_handler、controls 和 ctrl_ops
-> sd.internal_ops / HAS_DEVNODE / MEDIA_ENT_F_CAM_SENSOR // 补齐 subdev 生命周期回调、devnode 标志和 entity 类型
-> media_entity_pads_init()                              // 建立 sensor source pad 和 media entity 关系
-> v4l2_async_register_subdev_sensor_common()            // 交给 host 侧异步绑定
-> host 完成绑定 / 可选注册 v4l-subdevX                  // sensor 进入媒体管线并可被用户态访问
-> ioctl(VIDIOC_S_CTRL, V4L2_CID_EXPOSURE)               // 用户发起参数调整
-> v4l2_s_ctrl() -> try_or_set_cluster()                 // control core 查找、校验并提交新值
-> imx219_set_ctrl()                                     // sensor 驱动接住具体 control
-> imx219_write_reg() -> i2c_master_send()               // 当前已上电时直接写曝光等寄存器
-> 用户态 STREAMON / host 启动 pipeline                  // 真正采集由 host/capture 节点触发
-> imx219_start_streaming()                              // 起流阶段准备 mode 和格式
-> __v4l2_ctrl_handler_setup(sd.ctrl_handler)            // 若此前未上电，则把已保存 controls 回放到硬件
-> IMX219_REG_MODE_SELECT = STREAMING                    // sensor 开始输出 MIPI 像素流
```

这条闭环可以按四段理解：

- `probe` 段确认设备树配置、硬件存在、资源可用，并建立 sensor 自身对象和参数模型
- `async` 段把 sensor 接入 host 管线
- `control` 段把用户态曝光/增益请求送到 sensor 驱动
- `streaming` 段保证尚未落硬件的 controls 在真正出流前被统一应用

这样再回看 `imx219`，它就不只是“一个能注册的 subdev”，而是一个从对象建立、参数承载、异步接入到寄存器生效都闭合的 sensor 驱动。

## 11. 本章边界

本章只展开 sensor 自身这一侧：

- `subdev` 怎么建立
- controls 怎么注册
- 用户态参数怎样落到 sensor 寄存器
- `pad format` 和 `s_stream` 在 sensor 层各自负责什么

下面这些内容不在本章展开：

- host 怎样把多路 subdev 拼成完整 graph，见 [[11-典型host驱动链路-camss]]
- control ioctl 的通用 video node 派发细节，见 [[03-ioctl派发与v4l2_ioctl_ops]]
- 更长的参数调整复盘说明，见 [[09-1-参数调整附录]]

## 12. 一句话总结

`imx219.c` 代表的是 **“一个标准 V4L2 sensor subdev 怎么写”**。  
如果把 `sh_vou.c` 对应为 `/dev/videoX` 这一层，`imx219.c` 对应的就是媒体管线内部“源头设备”这一层；它不仅负责 `pad format` 和 `s_stream`，还负责把用户态 exposure/gain 等 controls 最终写回 sensor 寄存器。

扩展复盘可接 [[09-1-参数调整附录]]。
