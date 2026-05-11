# Engineering Debug and Q&A

## 1. Layered Debugging Rule

Debug chapters should not be written as miscellaneous notes. Use layered localization.

At minimum, split issues into:

- board or host layer
- DTS, ACPI, or firmware layer
- bus or enumeration layer
- core or framework layer
- match or probe layer
- private initialization layer
- resource-acquisition layer
- IRQ, DMA, or completion layer
- user-space or upper subsystem layer

For each layer, answer:

- the common failures in this layer
- the first checkpoints in this layer
- whether the next step should go upward or downward
- whether related objects have been created
- whether related callbacks have been registered
- whether related resources have been acquired successfully

Recommended debug order:

```text
现象
↓
判断用户态入口是否存在
↓
判断 framework 对象是否注册
↓
判断 probe 是否执行
↓
判断 match 是否成功
↓
判断设备是否枚举 / DTS 是否生效
↓
判断资源是否申请成功
↓
判断数据路径 / IRQ / DMA 是否运行
```

## 2. Engineering Q&A Rule

Organize engineering Q&A by symptom, not by concepts.

Each item should follow:

- `### 现象`
- `### 定位层`
- `### 现象对应的主流程阶段`
- `### 常见检查点`
- `### 常见根因`
- `### 核心判断`
- `### 下一步动作`

### 2.1 Checkpoint format

Each checkpoint should contain:

1. what to inspect
2. why it matters
3. the related call path
4. the expected result
5. what an abnormal result implies

Recommended format:

```md
- `检查项`：为什么这个点会导致当前现象；相关链路：`调用链`；预期结果：...；异常结果：...
```

### 2.2 Minimum issue coverage

At minimum, cover:

- device does not appear
- device node does not appear
- `probe()` never runs
- `probe()` fails
- initialization failure
- open failure
- ioctl failure
- I/O read or write failure
- interrupt never arrives
- DMA never completes
- buffer flow stops
- stop or unload cleanup failure
- suspend or resume failure
- upper-layer function never appears

## 3. Engineering Localization Rule

Before diving into function details, decide whether the symptom is caused by:

```text
对象没建立
match 没成功
probe 没执行
资源申请失败
回调没有注册
用户态没有触发
数据链路没有打通
中断没有完成
释放路径异常
```

Use this fixed order:

1. decide the layer
2. decide the key object
3. decide where the main path stopped
4. decide whether resources are usable
5. decide whether callbacks are firing
6. then analyze function-level details

## 4. Learning Priority

### 4.1 First layer

- core objects
- object relationships
- main flow
- probe
- data path
- user-space entry

### 4.2 Second layer

- lifecycle
- callback relationships
- resource model
- buffer model
- DMA model
- completion or IRQ path

### 4.3 Third layer

- concurrency context
- locking
- runtime PM
- suspend and resume
- error handling
- release path
- race conditions

## 5. Wording Rule

Generated notes should use learning-note wording:

- no person-centered phrasing
- no chatty reassurance
- no large code dumps
- no concept piles without object and path context
- keep the style review-friendly and engineering-oriented

Prefer:

- object duty
- function role
- main flow
- boundary
- core judgment

## 6. Embedded BusyBox or musl Constraints

Do not assume a full desktop Linux environment in embedded board work.

Typical constraints:

- target may be BusyBox-based
- target may use musl libc
- `systemd` may be absent
- `modprobe` may be absent
- full `iproute2` may be absent
- `udevadm` may be absent
- `lsusb`, `lspci`, `ethtool`, `v4l2-ctl` may be absent
- only `dmesg`, `cat /proc/*`, `cat /sys/*`, `ifconfig`, `mount`, `insmod`, `rmmod` may be available

When suggesting debug commands, prefer two variants:

```text
完整 Linux 环境命令
BusyBox / 最小 rootfs 替代命令
```

## 7. Response Templates

### 7.1 Explaining a driver concept

```md
## 1. 它解决什么问题
## 2. 它在内核里的位置
## 3. 核心对象
## 4. 关键函数
## 5. 主流程
## 6. 生命周期
## 7. 回调关系
## 8. 常见工程现象
## 9. 调试入口
## 10. 一句话总结
```

### 7.2 Analyzing a source file

```md
## 1. 文件定位
## 2. 这个文件属于哪一层
## 3. 核心对象
## 4. 关键函数
## 5. 入口函数
## 6. 主调用链
## 7. 重要回调
## 8. 资源申请与释放
## 9. 常见失败点
## 10. 阅读顺序
```

### 7.3 Debugging a real engineering issue

```md
## 1. 现象
## 2. 初步定位层级
## 3. 相关主流程
## 4. 必查对象
## 5. 必查资源
## 6. 必查回调
## 7. 推荐检查顺序
## 8. 可能根因
## 9. 下一步验证命令或打点位置
## 10. 核心判断
```
