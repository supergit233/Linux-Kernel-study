# 通用字符设备注册与节点暴露调用链

这个补充文件的目的只有一个：

- 先把 `Linux 普通 cdev 字符设备` 自己的注册、节点暴露、`open` 调用链讲清楚
- 再回头看 `V4L2` 时，就知道哪些是 `V4L2` 自己的逻辑，哪些只是复用了通用字符设备框架

## 1. 驱动侧最常见的写法

先不看 `V4L2`，一个最常见的字符设备驱动通常会这样写：

1. 分配设备号
2. 初始化 `struct cdev`
3. `cdev_add()`
4. 建 `class`
5. `device_create()`
6. 用户看到 `/dev/xxx`

最常见的一组 API 是：

- `alloc_chrdev_region()`
- `cdev_init()`
- `cdev_add()`
- `class_create()`
- `device_create()`

## 2. 一段最小代码骨架

下面这段代码更接近“教学版最小骨架”：

```c
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/fs.h>
#include <linux/module.h>
#include <linux/uaccess.h>

static dev_t demo_devt;
static struct cdev demo_cdev;
static struct class *demo_class;

static int demo_open(struct inode *inode, struct file *filp)
{
	return 0;
}

static int demo_release(struct inode *inode, struct file *filp)
{
	return 0;
}

static ssize_t demo_read(struct file *filp, char __user *buf,
			 size_t count, loff_t *ppos)
{
	return 0;
}

static const struct file_operations demo_fops = {
	.owner = THIS_MODULE,
	.open = demo_open,
	.release = demo_release,
	.read = demo_read,
};

static int __init demo_init(void)
{
	int ret;

	ret = alloc_chrdev_region(&demo_devt, 0, 1, "demo_char");
	if (ret)
		return ret;

	cdev_init(&demo_cdev, &demo_fops);
	demo_cdev.owner = THIS_MODULE;

	ret = cdev_add(&demo_cdev, demo_devt, 1);
	if (ret)
		goto err_unregister_chrdev;

	demo_class = class_create(THIS_MODULE, "demo_char");
	if (IS_ERR(demo_class)) {
		ret = PTR_ERR(demo_class);
		goto err_cdev_del;
	}

	if (IS_ERR(device_create(demo_class, NULL, demo_devt, NULL,
				 "demo_char0"))) {
		ret = -EINVAL;
		goto err_class_destroy;
	}

	return 0;

err_class_destroy:
	class_destroy(demo_class);
err_cdev_del:
	cdev_del(&demo_cdev);
err_unregister_chrdev:
	unregister_chrdev_region(demo_devt, 1);
	return ret;
}

static void __exit demo_exit(void)
{
	device_destroy(demo_class, demo_devt);
	class_destroy(demo_class);
	cdev_del(&demo_cdev);
	unregister_chrdev_region(demo_devt, 1);
}

module_init(demo_init);
module_exit(demo_exit);
MODULE_LICENSE("GPL");
```

## 3. 这段代码到底各管什么

### 3.1 `alloc_chrdev_region()`

它负责分配：

- `major`
- `minor`
- 对应的 `dev_t`

也就是说，它先解决“这个字符设备以后用哪个设备号”。

### 3.2 `cdev_init()` / `cdev_add()`

这两步负责把：

- 这个设备号
- 这套真正的 `file_operations`

挂到内核的字符设备映射表里。

最关键的是 `cdev_add()` 之后，内核已经知道：

- 某个 `major + minor`
- 对应哪一个 `struct cdev`
- 而这个 `struct cdev`
- 又对应哪一套真正的 `file_operations`

也就是说，`open` 真正分发时最核心依赖的是这一层。

### 3.3 `class_create()` / `device_create()`

这两步主要是为了让设备进入设备模型，并在用户态形成常见的设备节点表现，例如：

- `/dev/demo_char0`

所以可以把这两层职责分开记：

- `cdev_add()` 解决“字符设备分发”
- `device_create()` 解决“节点怎么在 `/dev` 里出现”

这里很容易和 `device_register()` 混掉，要单独分清：

- 很多通用字符设备教程代码里，确实看不到显式的 `device_register()`
- 因为它们用的是更上层的 `device_create()`
- 在这份 `Linux 5.10` 源码里，`device_create()` 往下走的是 `device_initialize()` + `device_add()`
- 而 [core.c:3085](/E:/linux_share/驱动学习/HI3516CV610-SIM-开发板资料/linux-5.10.y/drivers/base/core.c:3085) 里的 `device_register()` 本身也只是：
  - `device_initialize(dev);`
  - `return device_add(dev);`

所以从语义上看：

- `device_register()` 对应“已有现成的 `struct device`，再把它注册进设备模型”
- `device_create()` 对应“分配一个 `struct device`，填好 class / devt / 名字，再把它加进设备模型”

也就是说，很多教程没有写出 `device_register()` 这个名字，不等于它没走“设备进入设备模型”这层；只是它用了更高一层的包装 API。

## 4. 用户 `open("/dev/xxx")` 之后，内核怎么走

这一段是最关键的，也是后面理解 `V4L2 open` 的基础。

### 4.1 inode 最初不是直接挂驱动私有的 `demo_fops`

字符设备 inode 初始化时，先挂的是默认字符设备操作集：

- [inode.c:2127](/E:/linux_share/驱动学习/HI3516CV610-SIM-开发板资料/linux-5.10.y/fs/inode.c:2127)
  `init_special_inode()`
- [inode.c:2131](/E:/linux_share/驱动学习/HI3516CV610-SIM-开发板资料/linux-5.10.y/fs/inode.c:2131)
  `inode->i_fop = &def_chr_fops;`

也就是说，用户第一次 `open("/dev/demo_char0")` 时，并不是直接进入驱动私有的 `demo_open()`。

### 4.2 先进入 `chrdev_open()`

默认字符设备操作集定义在：

- [char_dev.c:452](/E:/linux_share/驱动学习/HI3516CV610-SIM-开发板资料/linux-5.10.y/fs/char_dev.c:452)
  `const struct file_operations def_chr_fops`

其中：

- [char_dev.c:453](/E:/linux_share/驱动学习/HI3516CV610-SIM-开发板资料/linux-5.10.y/fs/char_dev.c:453)
  `.open = chrdev_open`

所以用户 `open("/dev/demo_char0")` 后，会先进入：

- [char_dev.c:373](/E:/linux_share/驱动学习/HI3516CV610-SIM-开发板资料/linux-5.10.y/fs/char_dev.c:373)
  `chrdev_open()`

### 4.3 `chrdev_open()` 做了什么

它的关键动作可以压成 4 步：

1. 从 `inode->i_rdev` 取出 `major + minor`
2. 按这个设备号找到对应的 `struct cdev`
3. 取出 `cdev->ops`
4. `replace_fops(filp, fops)`，把当前 `file` 的操作表替换成真正设备自己的操作表

然后它会继续调用新的：

- `filp->f_op->open()`

这时才真正进入驱动里注册的那套 `demo_fops.open`，也就是：

- `demo_open()`

### 4.4 用一条链把它串起来

主线可压成：

`open("/dev/demo_char0")`
-> `def_chr_fops.open`
-> `chrdev_open()`
-> 按 `inode->i_rdev` 找 `cdev`
-> `replace_fops(filp, cdev->ops)`
-> `demo_open()`

这就是普通 `cdev` 字符设备最标准的打开路径。

## 5. 和 `V4L2` 的关系

回到 `V4L2`：

- `V4L2` 没有发明新的字符设备 `open` 机制
- 它只是把自己的 `&v4l2_fops` 填进了 `vdev->cdev->ops`
- 所以通用字符设备路径在 `replace_fops()` 之后，才切进 `V4L2`

但这里要特别注意：  
和“普通字符设备直接把 `cdev->ops` 指到驱动自己的 `demo_fops`”相比，`V4L2` 确实多了一层框架分发。

### 5.1 普通 `cdev` 的打开链

普通字符设备通常是：

- `cdev->ops = &demo_fops`

所以它的主线可以压成：

`open("/dev/demo_char0")`
-> `def_chr_fops.open`
-> `chrdev_open()`
-> `replace_fops(filp, cdev->ops)`
-> `demo_open()`

也就是说，`chrdev_open()` 切过去以后，基本就直接到驱动自己的 `struct file_operations` 了。

### 5.2 `V4L2` 的打开链比它多一层

`V4L2` 不是把 `cdev->ops` 直接指到驱动私有的 `open`，而是：

- `vdev->cdev->ops = &v4l2_fops`

然后 `v4l2_fops.open` 对应的是 `v4l2_open()`，而 `v4l2_open()` 再继续分发到：

- `vdev->fops->open()`

这里的 `vdev->fops` 不是 Linux 原生 `struct file_operations`，而是 `V4L2` 自己的：

- `struct v4l2_file_operations`

也就是说，`V4L2` 对应的是下面这段：

`open("/dev/videoX")`
-> `def_chr_fops.open`
-> `chrdev_open()`
-> `replace_fops(filp, cdev->ops)`
-> `cdev->ops == v4l2_fops`
-> `v4l2_open()`
-> `vdev->fops->open()`

按“层”来记，可写成：

- `def_chr_fops`
- `v4l2_fops`
- `vdev->fops`

只是要注意，严格说真正的“函数调用链”里，中间还夹着：

- `chrdev_open()`
- `replace_fops()`
- `v4l2_open()`

对应展开见 [[02-video_device注册与open链路#6. `open()` 链路怎么走]]。

## 6. 为什么 `V4L2` 的注册代码看起来不完全像教程

很多字符设备教程会写：

- `alloc_chrdev_region()`
- `cdev_add()`
- `class_create()`
- `device_create()`

而 `V4L2` 在 `video_register_device()` 里更像：

- 先分配固定主设备号体系下的 `minor`
- `vdev->cdev->ops = &v4l2_fops`
- `cdev_add()`
- `device_register(&vdev->dev)`

所以要注意：

- 底层“字符设备分发”的思想是一样的
- 只是“节点进入设备模型”的包装形式不一定完全一样

通用教程更喜欢 `class_create() + device_create()`，而 `V4L2` 这种子系统更常见的是自己维护 `struct device`，再走 `device_register()`。

## 7. 一句话总结

普通字符设备的主线不是“用户直接进驱动自己的 `open`”，而是：

- 先由 `def_chr_fops` 接住
- 再由 `chrdev_open()` 按设备号找到对应的 `cdev`
- 然后把 `file_operations` 切换成真正设备自己的 `cdev->ops`
- 最后才进入驱动自己的 `open`

掌握这条线后，再看 `V4L2`、串口、GPIO、misc device 这类字符设备时，主框架会更清晰。
