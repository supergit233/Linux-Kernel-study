# 离线 OpenCV 实验

## 导读

### 本章定位

这一章设计一个低风险的离线实验路径。先用录制视频和 gyro CSV 跑通防抖闭环，再进入嵌入式实时链路，会更容易定位问题。

### 核心对象

- video file
- gyro CSV
- calibration config
- Python/OpenCV script
- output video
- debug plot

### 关键函数

- `cv2.VideoCapture`
- `cv2.VideoWriter`
- `cv2.warpPerspective()` / `cv2.warpAffine()`
- NumPy interpolation
- gyro integration
- matplotlib plot

### 主流程

```text
录制原始视频                         // 保留明显手抖和转动
-> 同步保存 IMU CSV                   // 至少包含 timestamp,accel,gyro,temp
-> 离线读取视频帧                     // 获取 frame index 和 frame timestamp
-> 读取 IMU 样本                      // 检查采样率、单位、时间范围和三类数据是否合理
-> 做时间对齐和零偏修正                // 先把最基础输入修正好
-> 计算每帧补偿                       // 用简化积分和平滑算法
-> OpenCV warp 输出                   // 生成稳定后视频
-> 对比效果和曲线                     // 观察相位、方向、漂移和黑边
```

## 这一章按什么逻辑展开

本章按“数据准备 -> 最小算法 -> 可视化验证”的顺序展开。离线实验的价值在于把驱动实时性、ISP 性能和算法正确性暂时拆开。

## 1. 数据格式建议

IMU CSV 建议至少包含：

```text
timestamp_ns,ax_g,ay_g,az_g,gx_rad_s,gy_rad_s,gz_rad_s,temp_c
```

如果当前程序已经输出 `gyro=(...)dps`，建议离线转换后再保存一份 `rad/s`，避免算法里反复混单位。转换关系是：

```text
rad/s = deg/s * 0.01745329252
```

如果还想保留排查能力，可以额外保存 raw 值：

```text
timestamp_ns,raw_ax,raw_ay,raw_az,raw_temp,raw_gx,raw_gy,raw_gz,ax_g,ay_g,az_g,gx_dps,gy_dps,gz_dps,temp_c
```

第一版算法可以只读取 `gx_rad_s/gy_rad_s/gz_rad_s`，但 `ax_g/ay_g/az_g/temp_c` 不建议丢掉：

- `accel` 用来检查静止段重力是否合理，辅助确认板子姿态和轴向。
- `gyro` 用来做零偏估计、角速度积分和防抖补偿。
- `temp` 用来观察热机过程中 gyro 零偏是否漂移。

视频侧至少需要：

- 原始视频文件
- 帧率
- 分辨率
- 每帧 timestamp

如果没有真实 frame timestamp，第一版可以用固定帧率生成估算时间，但要明确这是简化模型。

## 2. 最小离线防抖版本

第一版不要追求复杂：

```text
读取 IMU CSV                          // 得到时间序列 accel/gyro/temp
-> 静止段估计 bias                     // 简单求均值
-> gx/gy/gz 减 bias                    // 去零偏
-> 按帧 timestamp 采样或积分             // 得到每帧旋转量
-> 对旋转轨迹做滑动平均                  // 得到平滑轨迹
-> correction = smooth - raw            // 得到补偿量
-> cv2.warpAffine/warpPerspective       // 应用到每帧图像
```

最初可以只补偿一个轴，例如绕光轴旋转，先验证方向、时间和 crop 逻辑。

## 3. 必须画出来的调试曲线

- gyro 三轴原始曲线
- gyro 去零偏后的曲线
- accel 三轴曲线和合成值，静止段应接近 `1g`
- temp 曲线，观察温度变化时 gyro bias 是否也变化
- frame timestamp 与 gyro timestamp 覆盖范围
- raw rotation path
- smooth rotation path
- correction
- crop 使用量

这些曲线能快速暴露：

- 时间戳错位
- 轴方向反了
- 零偏没去干净
- 平滑窗口过大
- 补偿量超过 crop margin

## 4. 离线实验的成功标准

第一版成功不要求画面完美，只要求：

- 静止段不明显漂移
- 单方向小幅转动时补偿方向正确
- 调整 time offset 会按预期改变效果
- 平滑窗口变大时画面更稳但裁剪压力更大
- 输出视频没有明显黑边或撕裂

## 本章边界

离线 OpenCV 实验不代表最终嵌入式实现。它忽略了实时延迟、内存带宽、ISP metadata、硬件 warp 能力等问题，但非常适合验证算法方向。

## 一句话总结

离线实验的目标是先证明“gyro 数据能正确变成图像补偿”，不要把第一版卡在实时系统复杂度里。
