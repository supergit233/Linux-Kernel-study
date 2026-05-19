# MPP 架构学习实录

这份记录用于梳理 Hi3516CV610 项目中 MPP 视频通路、编码通路、RTSP/MP4、MPU6050 数据采集、时间戳对齐之间的关系。当前实践项目是 `venc_rtsp_mp4_gyro`，它是在原有 RTSP/MP4 视频程序基础上增加 IMU 旁路采集和视频帧时间戳记录。

## 0. 当前工程环境与资料边界

当前真正用于编译、运行、验证的工程环境在远程 Linux 主机：

```text
192.168.1.10
/home/ipc/Hi3516/1.0.1.0/Hi3516CV610_SDK_V1.0.1.0
```

当前项目的代码修改、编译结论、能力判断，都必须以这套 `1.0.1.0` SDK 为准。

本机另外有一套 `1.0.2.0` SDK：

```text
E:\linux_share\驱动学习\HI3516CV610-SIM-开发板资料\原厂sdk\版本文件\1.0.2.0
```

它可以帮助查接口、对照结构体变化，但不能直接当作当前工程依据。实际已经发现：

```text
1.0.1.0 的 ot_dis_mode:
  4_DOF_GME / 6_DOF_GME / GYRO / GYRO_ADVANCE

1.0.2.0 参考头文件里还多了:
  HYBRID
```

这说明“看起来同名的模块”在不同 SDK 版本里也可能能力不同，后续设计不能只看本地参考版本。

本地资料包里有正式文档：

```text
ReleaseDoc/zh/01.software/board/MPP/MPP 媒体处理软件 V6.0 开发参考.pdf
ReleaseDoc/en/01.software/board/MPP/MPP Media Processing Software V6.0 Development Reference.pdf
```

后面关于 DIS 的正式说明，主要来自这份开发参考；而当前工程能否编译、有哪些字段，仍要回到远程 `1.0.1.0` 头文件确认。

## 1. MPP 是什么

MPP 可以理解为海思提供的一套媒体处理框架。应用程序不是直接操作 sensor、ISP、编码器硬件，而是通过一组 MPI 接口调用 MPP 模块。

典型模块包括：

```text
VI    Video Input，视频输入，负责 sensor 数据接入
ISP   Image Signal Processor，图像信号处理
VPSS  Video Process Sub-System，视频处理，例如缩放、裁剪、格式转换
VENC  Video Encoder，视频编码，例如 H.264/H.265
RGN   Region，OSD/区域叠加
DIS   Digital Image Stabilization，电子防抖
VB    Video Buffer，图像帧缓冲池
SYS   系统级初始化、PTS、绑定关系等
```

应用层看到的 `ss_mpi_xxx` 接口就是调用这些模块的入口，例如：

```c
ss_mpi_vi_get_chn_frame()
ss_mpi_vpss_get_chn_frame()
ss_mpi_venc_get_stream()
ss_mpi_venc_release_stream()
ss_mpi_sys_get_cur_pts()
```

## 2. 视频从 sensor 到编码的主流程

当前项目的视频主链路可以简化成：

```text
Sensor
  -> VI
  -> ISP
  -> VPSS
  -> VENC
  -> 应用层取码流
  -> RTSP / MP4
```

更细一点：

```text
1. Sensor 曝光，输出 RAW/MIPI 数据
2. VI 接收 sensor 数据
3. ISP 对图像做 3A、去噪、颜色、曝光等处理
4. VPSS 对图像做缩放、裁剪、格式转换等处理
5. VENC 把图像帧压缩成 H.264/H.265 码流
6. 应用层调用 ss_mpi_venc_get_stream() 取出编码后的码流
7. 应用层把 H.264 送 RTSP，把 H.265 写 MP4
8. 应用层调用 ss_mpi_venc_release_stream() 释放码流
```

注意：应用层拿到的不是图像帧，而是编码后的码流包。也就是说，到了 `ss_mpi_venc_get_stream()` 这一步，图像已经经过 VI/ISP/VPSS/VENC 的完整流水线。

### 2.1 先区分三类东西

刚开始跑样例时，容易把“视频”看成一个整体。后面要做防抖，就必须拆开看：

```text
图像帧
  还没编码的 YUV/RAW 画面，属于 VI/ISP/VPSS 这条链路

编码码流
  H.264/H.265 压缩后的 NALU，属于 VENC 输出

应用侧数据
  RTSP 包、MP4 文件、CSV、IMU 环形队列，这些由应用自己维护
```

这三者所在的阶段不同，能做的事也不同：

```text
要改画面几何形状，例如防抖、旋转、裁剪
  最适合在“图像帧阶段”做

要发送网络或保存文件
  才是在“编码码流阶段”做
```

这也是为什么普通 RTSP 项目只要能从 VENC 取到码流就算跑通，而一旦进入防抖问题，就必须重新回头理解 VI/VPSS、VB、PTS。

## 3. 当前项目里的两路编码

当前设计目标是：

```text
H.264 -> RTSP 实时预览
H.265 -> MP4 录像
```

也就是：

```text
Sensor/VI/ISP/VPSS
  -> VENC H.264 -> RTSP
  -> VENC H.265 -> MP4
```

两路编码共享前面的图像采集和处理链路，但后面的用途不同：

```text
H.264 低延迟、便于实时播放
H.265 压缩率更高，更适合录像保存
```

在应用里，取流线程会轮询多个 VENC 通道。每取到一帧编码码流后，按编码类型分发：

```text
H.264 -> rtsp_sever_tx_video()
H.265 -> sample_mp4_muxer_write_stream()
```

## 4. VB、wrap buffer 和 VENC stream buffer 的关系

MPP 里常见几个容易混的概念：

```text
VB pool
wrap buffer
VENC stream buffer
应用层缓存
```

它们不是同一个东西。

### 4.1 三种 buffer 分别是什么形态，为什么能起作用

#### 4.1.1 VB pool：按整帧复用的固定块池

官方文档对 VB pool 的定义很直接：

```text
A VB pool consists of physically continuous equally-sized buffers.
```

也就是：

```text
一组物理连续、大小相同的 buffer block
```

在代码里也能看到这种形态：

```c
vb_cfg.common_pool[i].blk_size = ...
vb_cfg.common_pool[i].blk_cnt = ...
```

它不是“按需 malloc 一大块图像内存”，而是启动时就按业务规格预先准备好：

```text
这一池每块多大
这一池一共有几块
```

例如图像帧的块大小可由分辨率、像素格式、压缩格式计算出来。  
只要某一路输出的图像规格固定，整帧 buffer 的大小也就基本固定。

因此 VB pool 很适合保存：

```text
RAW 整帧
YUV 整帧
VPSS 输出整帧
```

它为什么能起作用：

```text
1. 预分配
   启动时就把大块物理内存准备好，避免运行中频繁申请大内存

2. 等大小复用
   一帧用完后把 block 归还池中，下一帧直接复用

3. 传引用，不搬整帧
   VI、VPSS、VENC 之间可以传 block 句柄/地址，而不是反复 memcpy 一整帧

4. 适合 offline
   只要中间要落一张完整 RAW/YUV 帧，就需要某种“整帧容器”，VB block 正好承担这个角色
```

所以 VB pool 更像：

```text
一摞可重复使用的整帧托盘
```

#### 4.1.2 wrap buffer：在线流水线里的环形行缓冲

wrap buffer 和普通 VB block 最大的区别是：

```text
它不是为了先完整保存一整帧
而是为了让上下游在线流转时，保留一小段尚未被消费的图像行
```

官方接口里不是让你配置“多少整帧”，而是围绕：

```text
buf_line
```

也就是“需要多少行像素的 wrapping buffer”。当前样例启动日志里也会看到类似：

```text
buf line is 192
```

这说明它关心的是：

```text
保留多少行
而不是保留多少完整帧
```

可以把它想成这样：

```text
生产者不断把新行写进一段环形内存
消费者在后面不断把这些行读走
写指针到末尾后回到开头继续写
```

只要满足：

```text
消费者在写指针追上来之前，已经把旧数据读走
```

那就没有必要准备完整一帧大小的中间 buffer。

它为什么能起作用：

```text
1. online 链路本来就是边来边处理
   下游不必等整帧全部到齐才开始工作

2. 只保留“处理延迟差”所需的若干行
   不是保留全帧，所以内存占用显著更小

3. 环形复用
   已被消费的旧行可以立即被新行覆盖，不需要再申请新块

4. 低延迟
   没有“先写完整帧到 DDR，再整帧读出”的等待过程
```

代价也很明确：

```text
wrap buffer 依赖上下游跟得上
如果下游太慢，或者中断响应拖延，所需 buf_line 就会变大
大到接近半帧甚至更多时，wrap 的收益就开始失去意义
```

官方文档也专门说明，`buf_line` 会受这些因素影响：

```text
sensor timing
ISP 中断配置时间
VPSS 性能
VENC 性能
CPU 中断响应延迟
```

当前 sample 里也能看出它和普通整帧池不是一回事：

```c
vb_cfg->common_pool[i].blk_cnt = 1;
vb_cfg->common_pool[i].blk_size = wrap_size;
vpss_cfg->wrap_attr[0].enable = TD_TRUE;
```

也就是专门准备一个 wrap 用的 buffer，而不是准备多块完整帧托盘。

所以 wrap buffer 更像：

```text
在线流水线上的环形传送带
```

#### 4.1.3 VENC stream buffer：编码器自己的压缩字节环

VENC stream buffer 保存的不是图像帧，而是：

```text
H.264 / H.265 编码后的压缩字节流
```

它的对象已经不是“二维图像矩阵”，而是：

```text
长度会变化的 NALU / bitstream 字节
```

因此它不适合继续沿用“固定大小整帧 block”的思路。  
官方文档给出的结构也是：

```text
每个 VENC 通道有自己的 stream buffer
应用通过 ss_mpi_venc_get_stream() 取出码流描述
用完后必须 ss_mpi_venc_release_stream()
```

它的实现形态可以理解为：

```text
编码器持有的一段码流缓冲区
内部按字节顺序写入压缩结果
读到末尾后也可能发生 wrap
```

文档里甚至给了检查 stream buffer 是否发生 wrap 的说明，说明它本身就是会环绕使用的一段码流空间。

它为什么能起作用：

```text
1. 编码输出大小天然不固定
   I 帧大，P 帧小，NALU 数量也会变
   用字节流缓冲比用整帧块更合适

2. 解耦编码器和应用层
   编码器先把码流写进去
   应用稍后再 get_stream 取走

3. 通过 get/release 管理占用
   get_stream 只是把这段码流交给应用使用
   release_stream 才表示这段空间可以回收

4. 能吸收短时处理抖动
   应用偶尔晚一点取流，前提是 stream buffer 还没满，编码器还能继续往后写
```

如果应用长期不取流或取了不释放，官方文档也明确说：

```text
stream buffer 会满
buffer 满后，编码会暂停，直到流被取走并释放
```

当前项目里的代码正是这套生产消费关系：

```c
ss_mpi_venc_get_stream(...)
  -> RTSP / MP4 使用
ss_mpi_venc_release_stream(...)
```

所以 VENC stream buffer 更像：

```text
编码器和应用之间的压缩字节环
```

#### 4.1.4 三者放在一张表里看

| buffer | 缓存对象 | 典型形态 | 主要解决什么问题 |
| --- | --- | --- | --- |
| `VB pool` | RAW/YUV 整帧 | 多个固定大小、物理连续的整帧 block | 整帧复用、跨模块传递、offline 落帧 |
| `wrap buffer` | 在线流转中的若干图像行 | 按行数设计的环形缓冲 | online 低延迟、少占内存、减少整帧 DDR 周转 |
| `VENC stream buffer` | H.264/H.265 压缩码流 | 编码器持有的字节流缓冲，可环绕 | 吸收码流大小波动，解耦编码器与应用取流 |

有了这张表，再看下面几句话就不会混了。

VB pool 主要用于图像帧： 

```text
VI/VPSS 输出 YUV 图像帧时，需要从 VB 池里拿 buffer
```

wrap buffer 主要用于低延迟在线流水线的图像周转。  
当前项目里实际启用的是 **VPSS-VENC wrapping** 这一类：

```text
VPSS 输出侧不按“整帧块”交给 VENC
而是用环形行缓冲减少整帧周转和内存占用
```

VENC stream buffer 用于编码后的码流：

```text
H.264/H.265 编码后的 NALU 数据不是 YUV 图像，而是压缩码流
VENC 有自己的码流 buffer
应用层通过 ss_mpi_venc_get_stream() 拿到指向这些码流包的结构
```

应用层缓存是我们自己设计的数据结构：

```text
比如预录环、CSV、MP4 writer、IMU 环形缓冲
```

因此，历史录像缓存不应该放在 wrap buffer。wrap buffer 是给实时图像流水线周转用的，不适合作为“最近一分钟历史录像”的存储区。

### 4.2 online、offline 到底指哪一段

`online/offline` 不是整个系统只有一个总开关，而是描述两段边界各自怎么传数据。

官方开发参考把这两段定义得很明确：

```text
VI online/offline
  指 VI_CAP 和 VI_PROC 之间的传输方式

VPSS online/offline
  指 VI_PROC 和 VPSS 之间的传输方式
```

先把这几个名字翻成更容易理解的话：

```text
VI_CAP
  采集前端，接 sensor 数据

VI_PROC
  VI 管线的处理侧，和 ISP 这段 RAW -> YUV 的处理链路放在一起理解

VPSS
  后续视频处理模块，继续做缩放、几何处理等
```

这里要特别避免一个误解：

```text
VI_PROC 不是“ISP 后面又额外多出来的一个普通模块”
```

更接近真实的数据流理解应该是：

```text
Sensor/MIPI
  -> VI_CAP
  -> VI_PROC / ISP 处理侧
  -> VPSS
  -> VENC
```

其中：

```text
VI_CAP
  负责把 sensor 侧的 RAW 接进来

ISP
  负责 RAW 域到可用图像的处理能力，例如去马赛克、曝光、白平衡等

VI_PROC
  是 MPP 在数据流上描述这段“处理侧”的节点名
  官方文档用它来划分 VI online/offline 和 VPSS online/offline 的边界
```

因此：

```text
VI_CAP -> VI_PROC
  讨论的是 RAW 这一段是否落 DDR

VI_PROC -> VPSS
  讨论的是处理后的 YUV 是否落 DDR
```

因此所谓 online/offline，本质上问的是：

```text
两个模块之间是直接流过去，
还是先把完整中间结果写到 DDR，再由下一级读回来？
```

### 4.3 四种 VI/VPSS 工作模式

官方文档给出了四种组合：

```text
OT_VI_ONLINE_VPSS_ONLINE
OT_VI_ONLINE_VPSS_OFFLINE
OT_VI_OFFLINE_VPSS_ONLINE
OT_VI_OFFLINE_VPSS_OFFLINE
```

可以把它们画成下面这样。

#### 1. VI online + VPSS online

```text
Sensor
  -> VI_CAP
  => 直传
  -> VI_PROC
  => 直传
  -> VPSS
```

含义：

```text
VI_CAP 不把 RAW 写到 DDR
VI_PROC 不把 YUV 写到 DDR
两段都尽量直接流转
```

特点：

```text
延迟低
DDR 读写少
对整帧 VB 依赖小
链路更紧，很多需要完整帧落地或额外处理的功能会受限
```

#### 2. VI online + VPSS offline

```text
Sensor
  -> VI_CAP
  => 直传
  -> VI_PROC
  -> YUV 写 DDR
  -> VPSS 再从 DDR 读 YUV
```

含义：

```text
前半段 RAW 不落 DDR
后半段 YUV 落 DDR
```

特点：

```text
比全 online 多一个 YUV 全帧 buffer 边界
延迟和带宽会上升
但 VPSS 前有了完整 YUV 帧，后续处理更灵活
```

#### 3. VI offline + VPSS online

```text
Sensor
  -> VI_CAP
  -> RAW 写 DDR
  -> VI_PROC 再从 DDR 读 RAW
  => 直传
  -> VPSS
```

含义：

```text
前半段 RAW 落 DDR
后半段 YUV 不落 DDR
```

特点：

```text
适合需要前端 RAW 落地的场景
但 VPSS 前仍然是 online，很多 VI 通道后处理接口依然受限
```

#### 4. VI offline + VPSS offline

```text
Sensor
  -> VI_CAP
  -> RAW 写 DDR
  -> VI_PROC 再从 DDR 读 RAW
  -> YUV 写 DDR
  -> VPSS 再从 DDR 读 YUV
```

含义：

```text
RAW 和 YUV 两段都落 DDR
```

特点：

```text
最灵活
最容易插入完整帧级处理
但 VB、DDR 带宽、延迟开销都最大
```

### 4.4 buffer 在这四种模式里的作用

一旦把路径拆开，就能看清 buffer 为什么会影响能力。

```text
online
  重点不是“完全没有 buffer”，而是中间不必先保存完整帧再交给下一级
  更像流水线边生产边消费

offline
  中间必须有完整结果落到 DDR
  下一级再从 DDR 读取
  这就需要 VB block 承接完整帧
```

如果再按“哪一段需要完整中间帧”来对照，会更直观：

```text
VI online + VPSS online
  RAW 不落 DDR
  YUV 不落 DDR
  中间主要依赖在线周转，最省内存和带宽

VI online + VPSS offline
  RAW 不落 DDR
  YUV 落 DDR
  VPSS 前有完整 YUV 帧

VI offline + VPSS online
  RAW 落 DDR
  YUV 不落 DDR
  VI 前后处理之间有完整 RAW 帧

VI offline + VPSS offline
  RAW 落 DDR
  YUV 落 DDR
  两段都有完整中间帧，能力最灵活，代价也最大
```

所以：

```text
全 online
  更省内存、更省带宽、更低延迟
  但中间没有那么自由的完整帧边界可供各种处理复用

含 offline
  更吃内存、更吃带宽
  但模块边界清楚，完整帧更容易被裁剪、取出、附加处理
```

这件事是后面理解 DIS 的钥匙：  
**很多高级处理不是单纯“有没有算法”，而是它需要在哪个边界拿到什么样的帧和附加信息。**

#### 4.4.1 四种模式分别到底用了哪些 buffer

前面只说“哪一段 online、哪一段 offline”还不够。  
真正把模式和 buffer 扣起来，应该这样看：

| 模式                          | `VI_CAP -> VI_PROC` 这一段 | `VI_PROC -> VPSS` 这一段 | RAW 是否落 DDR | YUV 是否落 DDR | 主要中间 buffer                                          |
| --------------------------- | ----------------------- | --------------------- | ----------- | ----------- | ---------------------------------------------------- |
| `VI online + VPSS online`   | 直接流转                    | 直接流转                  | 否           | 否           | 中间不需要 RAW/YUV 整帧 VB；若启用 wrap，则靠 `wrap buffer` 做在线行周转 |
| `VI online + VPSS offline`  | 直接流转                    | 先写 DDR，再读回            | 否           | 是           | `YUV 整帧 VB block`                                    |
| `VI offline + VPSS online`  | 先写 DDR，再读回              | 直接流转                  | 是           | 否           | `RAW 整帧 VB block`                                    |
| `VI offline + VPSS offline` | 先写 DDR，再读回              | 先写 DDR，再读回            | 是           | 是           | `RAW 整帧 VB block + YUV 整帧 VB block`                  |

这张表可以再翻译成更口语的话：

```text
online
  不是“没有 buffer”
  而是这个模块边界上不需要先攒出一张完整中间帧

offline
  不是“只是慢一点”
  而是这个模块边界上真的要有一张完整中间帧落到 DDR
  所以要由 VB block 这种整帧容器承接
```

因此四种模式下，buffer 的真实分工是：

```text
1. VI online + VPSS online
   RAW 不需要整帧 VB
   YUV 不需要整帧 VB
   若项目开启 wrap，在线链路中会用 wrap buffer 保留若干图像行
   VENC 之后仍然有自己的 VENC stream buffer

2. VI online + VPSS offline
   RAW 仍不需要整帧 VB
   但 YUV 必须先落一张完整帧到 DDR
   所以 VI_PROC 和 VPSS 之间需要 YUV 整帧 VB

3. VI offline + VPSS online
   RAW 必须先落一张完整帧到 DDR
   所以 VI_CAP 和 VI_PROC 之间需要 RAW 整帧 VB
   但 VI_PROC 到 VPSS 仍是在线直传，不需要 YUV 整帧 VB

4. VI offline + VPSS offline
   RAW 先落整帧 VB
   YUV 再落整帧 VB
   两个边界都存在完整帧容器
```

这里还要把一个容易混淆的点单独拎出来：

```text
VENC stream buffer 不属于上面这两个 online/offline 边界
```

它永远在编码器输出侧，保存的是 H.264/H.265 压缩码流。  
所以无论前面是：

```text
全 online
半 offline
全 offline
```

只要最后进了 VENC，编码后的字节流都还会进入 `VENC stream buffer`，再由应用层 `get_stream / release_stream` 取走和归还。  
这也是为什么：

```text
VI/VPSS 的 online/offline
  决定的是图像域中间帧怎么走

VENC stream buffer
  决定的是编码后码流怎么暂存
```

它们不是一层东西。

上面这张表只是在回答：

```text
online/offline 所定义的这两个模块边界上，是否需要完整中间帧 buffer
```

它不等价于：

```text
全 online 时，整套系统里就完全没有 VB
```

因为在这两个边界之外，仍然可能存在：

```text
VPSS 某个输出通道自己的输出帧 buffer
应用主动取帧时使用的帧 buffer
其他模块为了交付最终图像结果而持有的 buffer
```

所以更准确的说法是：

```text
全 online
  省掉的是 VI_CAP -> VI_PROC 的 RAW 中间整帧 VB
  以及 VI_PROC -> VPSS 的 YUV 中间整帧 VB

它不是把所有图像 buffer 都清空
```

#### 4.4.2 用“数据在什么地方停下来”再记一遍

如果还觉得抽象，可以只问一句：

```text
这一段数据有没有在 DDR 里停成一张完整帧？
```

对应关系就是：

```text
没有停成完整帧
  -> online
  -> 可能只需要 wrap 这类小范围在线周转

停成 RAW 完整帧
  -> VI offline
  -> 需要 RAW 整帧 VB

停成 YUV 完整帧
  -> VPSS offline
  -> 需要 YUV 整帧 VB
```

再把整个视频链路从左到右画出来：

```text
Sensor
  -> VI_CAP
     [若 VI offline：这里需要 RAW 整帧 VB]
  -> VI_PROC / ISP
     [若 VPSS offline：这里需要 YUV 整帧 VB]
  -> VPSS
     [若在线链路启用 wrap：这里可能使用 wrap buffer 做行级周转]
  -> VENC
     [这里始终进入 VENC stream buffer，保存压缩码流]
  -> 应用层
```

所以以后讨论某个功能能不能插进去，不要只问：

```text
它支不支持 online？
```

还要继续问：

```text
它需要拿到 RAW 完整帧，还是 YUV 完整帧？
这个边界当前有没有整帧 VB？
如果没有，能不能接受把这段改成 offline？
```
 
### 4.5 wrap buffer 为什么通常跟 online 放在一起讲

wrap buffer 不是录像缓存，也不是普通意义上的完整帧队列。

它更接近：

```text
在线链路里的环形行缓冲
```

也就是下游还在持续消费，上游继续往后写，通过环形复用一段较小的 buffer，而不是每次都准备一整帧大小的中间块。

当前项目的代码就明确写了：

```c
sys_cfg->mode_type = OT_VI_ONLINE_VPSS_ONLINE;
sys_cfg->vpss_wrap_en = TD_TRUE;
vpss_cfg->wrap_attr[0].enable = TD_TRUE;
```

这说明当前默认路线就是：

```text
VI online + VPSS online + VPSS wrap
```

它非常适合当前最初的目标：

```text
低延迟 RTSP
少占内存
快速把 sensor 图像送去编码
```

但这也同时意味着：

```text
当前链路并不是为了“在中间随便插一个完整帧级处理模块”设计的
```

### 4.6 为什么官方 DIS 不能直接塞进当前默认链路

官方文档对 DIS 的限制不是随手写的，而是和上面的数据流结构一致。

DIS 相关接口明确不支持：

```text
VI online + VPSS online
VI offline + VPSS online
```

把这两个模式放一起看，会发现它们共同点是：

```text
VPSS online
```

而开发参考又明确说明：

```text
DIS is implemented in the VPSS,
and takes effect only after data is output from the VPSS.
```

也就是说，官方 DIS 虽然从 VI 通道接口配置，但真正实现落在 VPSS；当 VPSS 仍是 online 直传时，这条官方 DIS 路径不满足要求。

因此可以得到当前项目的直接结论：

```text
当前默认链路:
  OT_VI_ONLINE_VPSS_ONLINE + wrap

官方 Gyro DIS:
  不能在这条默认链路上直接打开

若要走官方 DIS:
  至少要重新评估为 VPSS offline 的模式
```

这不是说“防抖一定做不了”，而是说：

```text
默认全 online 链路
  和
官方 DIS 所要求的处理条件

不是一套前提
```

后面如果不用官方 DIS，也仍然要回答同一个问题：

```text
你的自研补偿到底在哪个边界拿到完整帧、在哪个模块完成几何变换？
```

这里先只讨论“链路形态是否满足条件”。  
对 `Hi3516CV610` 这颗芯片，后面还要再叠加一层更靠前的判断：**芯片本身是否支持 DIS/GDC。**

## 5. VENC get/release 的意义

正常编码取流流程是：

```text
ss_mpi_venc_query_status()
ss_mpi_venc_get_stream()
读取/发送/保存码流
ss_mpi_venc_release_stream()
```

`get_stream` 的意思是：应用层拿到了 VENC 已经编码好的码流包。

`release_stream` 的意思是：应用层告诉 MPP，这批码流已经用完，可以释放底层 buffer。

如果应用层处理太慢，或者长时间不 release，VENC 的码流 buffer 可能被占满，最终影响编码输出。所以实时 RTSP 链路里必须尽快 release。

## 6. 为什么需要时间戳

做普通 RTSP/录像时，只要视频能正常出流就行。但做 IMU、姿态、防抖时，必须知道：

```text
某一帧画面发生在什么时候
这个时间附近设备的角速度是多少
```

所以当前项目记录了两条时间线：

```text
mpu6050_imu.csv   传感器时间线
video_pts.csv     视频帧时间线
```

`mpu6050_imu.csv` 记录：

```text
IMU 采样序号
IMU 采样 monotonic 时间
三轴加速度
三轴角速度
温度
```

`video_pts.csv` 记录：

```text
视频帧序号
VENC 通道
编码类型
VENC PTS
应用层拿到该帧时的 monotonic 时间
是否关键帧
码流大小
```

这两个 CSV 的核心对应关系是：

```text
video_pts.csv 的 monotonic_ns_snapshot
    对齐
mpu6050_imu.csv 的 monotonic_ns
```

这样可以先做离线验证：某一帧视频附近有哪些 IMU 样本。

## 7. 当前 video_pts.csv 的局限

当前 `video_pts.csv` 里的 `monotonic_ns_snapshot` 是应用层调用 `ss_mpi_venc_get_stream()` 后记录的时间。

它代表：

```text
应用层拿到编码码流的时间
```

它不代表：

```text
sensor 曝光开始时间
sensor 曝光结束时间
VI 收到这一帧的时间
```

中间存在一段流水线延迟：

```text
Sensor 曝光
  -> VI
  -> ISP
  -> VPSS
  -> VENC 编码
  -> 应用层 get_stream
```

所以它适合做第一阶段验证：

```text
视频是否稳定出帧
IMU 是否稳定采样
两条时间线能否大致对齐
延迟是否稳定
```

但如果要做严格 Gyro EIS，必须进一步靠近真实帧时间。

## 8. 更好的 Hi3516 时间戳设计

Hi3516 MPP 里视频帧结构 `ot_video_frame_info` 包含：

```c
frame_info.video_frame.pts
frame_info.video_frame.time_ref
```

VENC 码流包里也有：

```c
stream.pack[i].pts
```

此外 SYS 模块提供：

```c
ss_mpi_sys_get_cur_pts()
ss_mpi_sys_init_pts_base()
ss_mpi_sys_sync_pts()
```

更好的设计是建立三条时间关系：

```text
IMU monotonic 时间
MPP 当前 PTS
VENC/VI frame PTS
```

推荐新增一份映射日志：

```csv
mpp_pts_map.csv
seq,mpp_cur_pts,monotonic_ns,realtime_ms
```

它用于建立：

```text
MPP PTS 时间域 <-> CLOCK_MONOTONIC 时间域
```

有了这个映射，后续可以把 `video_pts.csv` 里的 `venc_pts` 换算到 `monotonic_ns` 附近，再去匹配 IMU 数据。

### 8.1 为什么防抖比普通录像更在意时间

普通录像只需要“帧顺序对”；防抖需要“这一帧曝光时设备怎么动”。

下面三个时间不是一回事：

```text
sensor 曝光时间
  画面真正被采下来的时间

frame pts
  MPP 给帧打的媒体时间戳

get_stream 时间
  应用层拿到编码码流的时间
```

应用层 `get_stream` 已经在 VENC 之后，里面混入了整条流水线延迟。离线验证阶段可以先拿它看趋势，但要做真正的 Gyro EIS，最终目标一定是靠近 `frame pts`，最好还能进一步理解曝光窗口。

## 9. VI/VPSS 帧 PTS 的验证思路

MPP 提供 VI/VPSS 取帧接口：

```c
ss_mpi_vi_get_chn_frame()
ss_mpi_vi_release_chn_frame()
ss_mpi_vpss_get_chn_frame()
ss_mpi_vpss_release_chn_frame()
```

可以做一个实验版本：

```text
从 VI 或 VPSS 临时取帧
读取 frame_info.video_frame.pts
立即 release
和 VENC pack pts 做比较
```

如果发现：

```text
VI/VPSS frame pts 与 VENC pack pts 一致或稳定偏移
```

说明 VENC PTS 可以作为视频帧主时间。这样后续不必长期从 VI/VPSS 额外取帧，避免影响主链路。

## 10. DIS/Gyro DIS 与 IMU 的关系

SDK 里有 DIS/Gyro DIS 相关接口：

```c
ss_mpi_vi_set_chn_dis_cfg()
ss_mpi_vi_set_chn_dis_attr()
ss_mpi_vi_set_chn_dis_param()
ss_mpi_vi_set_chn_dis_alg_attr()
```

远程实际工程 `1.0.1.0` 中，DIS 模式包括：

```c
OT_DIS_MODE_4_DOF_GME
OT_DIS_MODE_6_DOF_GME
OT_DIS_MODE_GYRO
OT_DIS_MODE_GYRO_ADVANCE
```

开发参考对 DIS 的正式解释是：

```text
DIS 会比较当前图像与前两帧图像，
用不同自由度的 DIS 算法计算当前图像各轴向的抖动偏移，
再对当前图像做校正。
```

这句话里有两个重点：

```text
1. DIS 不只是“拿一个角速度做点运算”，它本身是一个图像稳定处理链路
2. GME DIS 这一类模式会利用图像帧之间的运动关系
```

而 `GYRO` / `GYRO_ADVANCE` 则说明官方同时提供了接入陀螺仪的路线。

### 10.1 DIS 在 MPP 里的真实位置

这点很容易误解。接口名长这样：

```c
ss_mpi_vi_set_chn_dis_cfg()
ss_mpi_vi_set_chn_dis_attr()
```

看上去像是“DIS 在 VI 里执行”。但开发参考的正式说明是：

```text
The DIS is implemented in the VPSS,
and takes effect only after data is output from the VPSS.
```

所以更准确的关系是：

```text
VI 负责配置入口
VPSS 内部真正执行 DIS
VPSS 输出后，稳定后的画面才真正生效
VENC 再去编码这份画面
```

可以画成：

```text
VI 配置 DIS
  -> VPSS 内部执行 DIS/GDC
  -> VPSS 输出稳定画面
  -> VENC H.264/H.265
```

这也解释了为什么 `ot_common_vpss.h` 里会有：

```c
is_dis_gyro_support
is_dis_ref_support
```

它们不是和 VI 接口矛盾，而是说明 DIS 的执行资源和 VPSS 组能力有关。

### 10.2 官方 DIS 不是“开关一开就完事”

开发参考里还明确了很多限制：

```text
1. 使用 DIS 时，要给 VB 开 motion supplement:
   OT_VB_SUPPLEMENT_MOTION_DATA_MASK

2. 一个 pipe 只能启用一个 DIS 通道

3. 使用 DIS 时，VI_CHN 压缩格式必须满足要求

4. Gyro DIS 与 VPSS 的 PMF/FOV/spread/rotation/LDC/stitch/fisheye/ZME 等能力互斥

5. 这套 DIS 接口不支持 VI online + VPSS online
```

这说明官方 DIS 不是单独加一个 API 就结束，而是会反过来约束：

```text
当前 VI/VPSS 模式
VB 补充信息
VPSS 组能力
后续还能不能缩放、旋转、LDC
```

其中 `ot_dis_attr` 里还有一个关键字段：

```c
td_s32 timelag;
```

注释含义是：

```text
Timestamp delay between Gyro and Frame PTS
```

这说明海思 DIS 设计里本来就考虑了：

```text
gyro 时间戳和视频帧 PTS 之间存在延迟
```

因此真正接入 Gyro DIS 时，不能简单拿“当前 IMU 数据”去补偿“当前帧”，而是应该根据：

```text
frame_pts + timelag
```

去 IMU 缓冲区里查找对应时间窗口内的 gyro 数据。

### 10.3 crop_ratio 为什么必不可少

防抖不是“把画面固定住”这么简单。画面一旦做反向平移或反向旋转，边缘就会露出无效区域，所以必须预留裁剪余量。

可以把它想成：

```text
原图范围更大
输出只取中间一块
当画面抖动时，在原图范围内把输出窗口挪回去
```

这就是为什么官方结构里会有：

```c
crop_ratio
still_crop
hor_limit
ver_limit
```

离线 OpenCV 脚本里的 `crop_scale=1.08`，本质上也是同一个道理。

### 10.4 当前芯片的支持边界，比链路条件更靠前

上面 10.1 到 10.3 讲的是“如果一颗芯片支持 DIS，它在 MPP 里大致怎么工作”。  
但看 `Hi3516CV610` 时，不能只看到头文件里有 `ss_mpi_vi_set_chn_dis_*()` 就默认它能用。

官方开发参考里有两类更强的限制：

```text
1. DIS 相关接口页多次直接标注：
   Hi3516CV610 does not support this MPI.

2. proc 调试章节的芯片差异说明写明：
   Hi3516CV610 and Hi3516CV608 do not support ... DIS ... and GDC.
```

这说明：

```text
共享头文件存在
  不等于
当前芯片已经实现对应模块
```

因此，对当前项目更稳妥的判断顺序应该是：

```text
第一步：先确认芯片支持矩阵
第二步：再看 online/offline 链路是否满足接入条件
第三步：最后才讨论具体 API、gyro 数据和调参
```

按目前这份正式开发参考，`Hi3516CV610` 的官方 DIS/GDC 路线应先视为：

```text
理解对象
而不是默认可落地方案
```

除非后续在板端的厂商包、私有扩展或实机能力验证里拿到相反证据，否则设计实时防抖时不能把“官方 DIS 一定可用”当作前提。

## 11. 讨论防抖方案前，必须先补齐的前置知识

这里不预设一定要走官方 DIS。官方 DIS 只是候选路线之一。  
但无论最后走哪条路线，下面这些知识都绕不过去。

### 11.1 第一层：先弄懂 online/offline 与 buffer

这是当前最优先要补的，不然后面讨论方案会一直停在“接口看起来有”这一层。

需要先能回答：

```text
1. 当前项目为什么默认选全 online
2. 全 online 省掉了哪些 DDR 落地
3. offline 多出来的 RAW/YUV 完整帧 buffer 在哪里
4. wrap buffer 和普通完整帧 VB block 有什么不同
5. 为什么 VPSS online 会让一批 VI 通道后处理接口不可用
6. 如果改成 VPSS offline，成本会落在内存、带宽还是延迟上
```

这几题答清楚后，DIS 的条件限制就不再像“SDK 硬规定”，而是能看出它背后的数据流原因。

### 11.2 第二层：先确认芯片支持矩阵

在 MPP 里，接口名、头文件、结构体通常是“家族共享”的。  
真正判断能不能用，还要再看：

```text
芯片差异表
接口页里的 support note
当前 SDK 版本的实际库和 proc 能力
```

对当前项目尤其要先确认：

```text
1. Hi3516CV610 是否真的支持 DIS
2. GDC 是否可用
3. 当前 1.0.1.0 板端包里，是否存在厂商私有替代路径
```

否则很容易出现：

```text
头文件能编译
结构体也能看懂
但模块在这颗芯片上根本不可用
```

### 11.3 第三层：再决定在哪个数据域做防抖

```text
编码前图像域
  在 VI/VPSS/GDC 一侧改画面
  优点：一次处理，RTSP 和录像都受益
  适合实时防抖

编码后码流域
  已经是 H.264/H.265
  想改画面就得解码、变换、再编码
  对板子实时链路基本不合适

离线后处理
  文件先录下来，后面再在 PC 上处理
  适合验证算法，不适合设备端实时预览
```

因此，如果目标是：

```text
RTSP 实时预览更稳
录像文件也更稳
```

那么无论是不是官方 DIS，最终都应优先考虑**编码前图像域**。

### 11.4 第四层：要知道当前链路有没有插入空间

同样是 `VI -> VPSS -> VENC`，不同工作模式的可插拔能力并不一样。

需要搞清：

```text
当前是 online 还是 offline
是否使用 wrap
VI 和 VPSS 如何绑定
是否允许额外取帧、送帧
是否能启用 VPSS/GDC 类能力
改变模式后，DDR 带宽和延迟会涨多少
```

这决定了三种路线能不能走：

```text
官方 DIS
自研但走 MPP 图像处理能力
完全应用层自处理
```

### 11.5 第五层：要知道 buffer 在谁手里

防抖会碰到的 buffer 至少有四类：

```text
VB 图像帧 buffer
wrap buffer
VENC stream buffer
应用层自己的 IMU/历史数据环
```

它们的生命周期不同。  
如果不清楚谁申请、谁释放、什么时候归还，就容易出现：

```text
图像链路没数据
VENC 取流卡住
缓存以为还在，底层其实已复用
```

### 11.6 第六层：要知道“画面时间”和“应用拿到时间”不是一回事

防抖最终要处理的是：

```text
这一帧曝光期间，相机转了多少角度
```

所以至少要理解：

```text
IMU monotonic 时间
MPP PTS
VI/VPSS frame pts
VENC pack pts
应用层 get_stream 时间
```

如果这几条时间线混着用，算法可能“看起来有补偿”，但实际上永远慢半拍。

### 11.7 第七层：要知道相机和 IMU 不是天然同一坐标系

陀螺仪的：

```text
x / y / z
```

不一定天然对应画面的：

```text
pitch / yaw / roll
```

还要考虑：

```text
传感器安装方向
镜头朝向
图像是否翻转
焦距换算
坐标正负号
rolling shutter
```

这也是为什么当前离线结果里：

```text
只做 z 轴旋转补偿，比 x/y 平移 + z 旋转更稳
```

它说明当前 `z -> roll` 关系比较可靠，而 `x/y -> 画面平移` 还没有标定好。

### 11.8 第八层：要把“稳定效果”和“代价”一起看

任何实时防抖都要付代价：

```text
裁剪带来的视场损失
额外缓存带来的延迟
offline 模式带来的带宽开销
更复杂算法带来的算力开销
黑边、果冻效应、过补偿风险
```

所以防抖方案不是“哪个最强就选哪个”，而是要结合：

```text
实时性
功耗
画质
稳定程度
当前 MPP 模式
产品目标
```

一起判断。

## 12. 防抖可行性方案，不止官方 DIS 一条路

当前可以先把路线分成三类。

### 12.1 官方 DIS / Gyro DIS

```text
优点：
  硬件链路内完成
  在编码前处理
  RTSP 和录像天然复用

难点：
  按当前正式开发参考，Hi3516CV610 标准支持层面已标注不支持 DIS/GDC
  即便先忽略芯片支持问题，当前 online/wrap 链路也不能直接套用官方 DIS
  还要继续查当前 1.0.1.0 板端包是否存在私有扩展或不同于公共文档的实现
```

### 12.2 自研实时 EIS，但仍放在编码前

思路是：

```text
IMU 环形队列
  -> 实时算每帧补偿
  -> 借助 MPP 可用的图像几何处理能力，或单独的图像处理链路
  -> 再交给 VENC
```

```text
优点：
  算法可控
  可以先从 rotation-only 做起

难点：
  必须找到当前芯片上可承受的实时图像变换位置
  很可能要改变当前链路
  不能把 PC 上的 OpenCV 思路原样搬到板子上
```

### 12.3 离线后处理

```text
优点：
  最容易验证算法
  目前已经打通

缺点：
  不改善设备实时预览
  不属于真正的设备端防抖
```

因此，当前最合理的研究顺序不是马上押注某一条路线，而是先把下面三个问题查实：

```text
1. 当前正式文档里的 CV610 不支持 DIS/GDC，是否被当前板端包或厂商扩展推翻
2. 当前项目如果退出 VI/VPSS online/wrap，代价有多大
3. 如果不用官方 DIS，Hi3516CV610 上还有没有适合的编码前几何处理落点
```

## 13. 防抖最终需要的数据

真正的 Gyro EIS/DIS 需要回答：

```text
某一帧曝光期间，设备转了多少角度？
```

所以最终数据流应该是：

```text
MPU6050 高频采样
  -> IMU 环形缓冲
  -> 根据 frame_pts 查找曝光时间窗口内的 gyro 数据
  -> 对 gyro 角速度积分
  -> 得到 delta_angle_x/y/z
  -> 送入 DIS/Gyro DIS 或自研 EIS 算法
```

其中积分关系是：

```text
角速度 * 时间 = 角度变化
```

如果只用应用层 `get_stream` 时间去找 IMU，会把编码延迟也混进去，防抖补偿会滞后。因此后续必须逐步转向 MPP PTS 和真实帧时间。

## 14. 当前项目阶段定位

当前 `venc_rtsp_mp4_gyro` 所处阶段：

```text
阶段 1：视频链路不动，增加 IMU 采集和视频时间戳日志
```

已经完成：

```text
H.264 RTSP 正常
H.265 MP4 正常
MPU6050 I2C 采样正常
mpu6050_imu.csv 正常
video_pts.csv 正常
mpp_pts_map.csv 已加入，用于记录 MPP 当前 PTS 和 CLOCK_MONOTONIC 的映射
CSV 默认写入 /mnt，也就是 NFS 主机共享目录
MPU6050 芯片内部已显式配置 100Hz；当前板子用户态 10000us sleep 实测约 50Hz，5000us sleep 实测接近 100Hz，因此应用默认轮询间隔使用 5000us
离线 frame_motion.csv 已完成，可按帧输出 gyro 积分角位移
第一版离线 EIS 已完成，可读取 MP4 + frame_motion.csv 输出 stabilized.mp4；当前默认采用 rotation-only，只启用 z 轴旋转补偿，x/y 平移补偿默认关闭
```

下一步建议：

```text
1. 补齐 VI/VPSS online/offline、VB、绑定关系、GDC 的理解
2. 验证 VI/VPSS frame pts 与 VENC pack pts 的关系
3. 继续做轴向标定，先把 rotation-only 的实时可行性看稳
4. 先确认 `Hi3516CV610` 官方 DIS/GDC 的不支持结论，是否在当前板端包上仍然成立
5. 同时评估不用官方 DIS 时，编码前还能落在哪个模块做几何补偿
6. 再决定最终实时方案是自研编码前 EIS、厂商私有路径，还是先停留在离线后处理
```

这条路线比直接上防抖更稳，因为它先把时间基准搞清楚。对 Gyro EIS 来说，时间对齐是地基，地基不稳，后面算法调参都会变成猜。

## 15. 目前还需要继续查实的点

这份笔记现在已经足够支撑“开始判断路线”，但还不足以直接下最终实现结论。后续至少还要继续查清三件事：

```text
1. 当前 sample_venc 的真实 VI/VPSS 绑定与工作模式
   不能只从函数名猜，要把实际配置值、bind 关系和 wrap 配置都列出来

2. `Hi3516CV610` 在当前板端包里的实际支持边界
   开发参考写明不支持 DIS/GDC；要先确认是否存在厂商私有扩展，还是应直接放弃官方 DIS 路线

3. 如果不用官方 DIS，Hi3516CV610 上还能在哪个编码前位置做几何补偿
   需要继续研究 VPSS/GDC 可用能力、模式限制和实时开销
```

这三件事查清后，才适合把方案真正收敛成：

```text
自研实时 EIS
厂商私有实时防抖路径
或保留离线后处理
```
