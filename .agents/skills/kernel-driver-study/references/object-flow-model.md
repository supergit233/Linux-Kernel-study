# Object and Flow Model

## 1. Source Entry Rules

Identify source entry points before deep code reading.

At minimum, locate:

- user-space entry
- driver registration entry
- bus match and probe entry
- data-path entry
- interrupt entry
- resource-acquisition entry
- lifecycle-exit entry

Recommended structure:

```md
## 推荐源码入口

### 用户态入口
- `open()` / `read()` / `write()` / `ioctl()` / `mmap()`
- `sysfs` / `procfs` / `debugfs`
- user tools such as `v4l2-ctl`, `ip`, `ethtool`, `arecord`, `modetest`

### 驱动注册入口
- `module_init()`
- `xxx_driver_register()`
- `platform_driver_register()`
- `usb_register()`
- `pci_register_driver()`

### 总线匹配入口
- `probe()`
- `of_match_table`
- `id_table`

### 数据路径入口
- `file_operations`
- `net_device_ops`
- `vb2_ops`
- `snd_soc_ops`
- `drm_driver`

### 中断入口
- `request_irq()`
- `irq_handler_t`
- threaded irq handler

### 释放入口
- `remove()`
- `shutdown()`
- `release()`
- `module_exit()`
```

## 2. Core Object Notes

Do not list struct names only. Write field-level notes.

For each key struct, explain:

1. what the object is
2. what it does in the main path
3. why the selected fields matter

Recommended structure:

```md
### `struct xxx`

职责：
- ...

在主线中的位置：
- ...

关键成员：
- `field_a`
  - ...
- `field_b`
  - ...

字段理解：
- ...

相关对象：
- 上层对象：...
- 下层对象：...
- 回调对象：...
```

Selection rule:

- do not copy every field in the struct
- only expand fields required to understand the current chapter
- make ownership, registration, callback, and private-data relationships explicit
- if the object was already explained in the project core-object chapter, add a chapter link instead of restating everything
- if the object is newly introduced by an example driver or a bridge layer private struct, explain it in place at field level before relying on it in later subpoints

## 3. Object Relationship Diagrams

Build relationship diagrams before long function call descriptions.

Each diagram should show:

- who owns whom
- who registers whom
- who calls back whom
- which object is the container
- which object is the core or framework object
- which object is the device
- which object is the driver
- which object holds private data

Examples:

```text
platform_device
    ↓ dev_get_drvdata()
xxx_priv
    ├── reg base
    ├── irq
    ├── clk
    └── framework object
```

```text
usb_device
 └── usb_interface
       └── usb_driver
             └── probe()
                   └── driver private data
```

```text
platform_device
 └── xxx_dev
       ├── v4l2_device
       ├── video_device
       ├── media_device
       └── vb2_queue
```

Do not leave diagrams unexplained.

After each important diagram, add explanation that:

- states what problem the diagram solves
- explains each stage or major node in the diagram
- maps the diagram back to the previously introduced objects or functions
- states where the next chapter or next layer begins

## 4. Function Notes

Do not only say what the function calls. Explain:

1. the function position in the main path
2. the problem it solves
3. the core object it operates on
4. the boundary to the next layer
5. the execution context
6. the engineering symptom if it fails

Recommended structure:

```md
## `xxx_function()`

源码：
- `path/to/file.c:line`

主链位置：
- ...

作用：
- ...

操作对象：
- ...

运行上下文：
- process / IRQ / workqueue / kthread / timer

主逻辑：
1. ...
2. ...
3. ...

这一层和下一层的边界：
- ...

失败影响：
- ...
```

When a chapter includes a code skeleton or callback table example, add a follow-up explanation section that:

- states which layer the sample belongs to
- explains what each key member or callback slot means
- separates the shell structure from the real complexity inside `probe()`, `start()`, `ioctl()`, or equivalent paths
- links the sample back to earlier object and main-flow sections

## 5. Main Flow Notes

Every project needs a complete main flow.

Build it in three steps:

1. a single top-level flow line
2. explain why the chapter follows this split order
3. staged explanation

For each stage, explain:

- why this stage is placed here in the reading order
- what it solves
- the entry function
- the involved core objects
- how it connects to the next stage
- what failure looks like

Example stage layout:

```text
阶段一：对象建立
阶段二：资源获取
阶段三：设备注册
阶段四：总线匹配
阶段五：私有初始化
阶段六：数据/I/O/中断路径
阶段七：停止与清理
```

If a chapter uses a summary flow chart, add a second layer of explanation after the chart.
That explanation should:

- first explain why the chart is split into these phases
- split the chart into readable phases
- explain what each phase establishes
- tie each phase back to the sections already covered in the same chapter
- tie the last phase forward to the next chapter when the path continues there

When there is user space or a system-level boot chain, also add:

- user-space symptom to kernel path
- boot-chain handoff path
- device-node to registration path
- log to failure-stage mapping

## 6. Lifecycle Notes

Every driver project should explain lifecycle:

- create
- register
- match
- initialize
- run
- suspend
- resume
- stop
- unregister
- release

Always answer:

- when each object is created
- who owns it
- when it is registered to the framework
- when it becomes visible to user space
- when it stops working
- when it is released
- which resources come in pairs
- which resources use refcounts
- which resources are devm-managed

## 7. Callback Notes

Always distinguish:

- direct calls
- framework callbacks invoked after registration

For callback tables such as `file_operations`, `net_device_ops`, `v4l2_ioctl_ops`, `vb2_ops`,
`snd_soc_ops`, `drm_driver`, `usb_driver`, `pci_driver`, `platform_driver`, explain:

- who registers the callback
- where it is stored
- who triggers it
- when it runs
- what context it runs in
- what breaks if it never triggers

## 8. Resource Model Notes

For any hardware-facing driver, cover resources such as:

- register or regmap
- IRQ
- DMA
- clock
- reset
- regulator
- GPIO
- pinctrl
- memory or reserved memory
- IOMMU
- firmware
- device tree property

For each resource, explain:

- where it comes from
- who parses it
- who requests it
- who enables it
- who disables it
- who releases it
- whether it is shared
- whether it is refcounted
- whether it depends on device tree
- what symptom appears when it fails

## 9. Context and Concurrency Notes

When IRQ, DMA, buffers, locking, or deferred execution are involved, explain the execution context.

At minimum, distinguish:

- process context
- hard IRQ context
- softirq or tasklet
- workqueue
- kthread
- timer
- completion or waitqueue paths

For each critical function, answer:

- whether sleep is allowed
- whether blocking is allowed
- whether mutex is legal
- whether copy_to_user or copy_from_user is legal
- whether bus access may sleep
- whether spinlock or mutex is required
- whether atomic variables or completions are needed

## 10. User-to-Kernel Path

For any project with a user-space interface, add a user-to-kernel path.

Recommended structure:

```text
用户态工具 / 应用
↓
system call / ioctl / netlink / sysfs
↓
VFS / UAPI layer
↓
framework core
↓
driver callback
↓
hardware resource
```

Always answer:

- which UAPI corresponds to the user action
- which framework core receives it
- which driver callback is reached last
- which layer to revisit when user space reports an error
- which registration path to revisit when a device node never appears
