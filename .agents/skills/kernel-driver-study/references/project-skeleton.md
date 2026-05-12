# Project Skeleton

## 1. Scope

Use this structure for:

- bus driver study projects such as `SDIO`, `USB`, `PCIe`, `I2C`, `SPI`
- subsystem driver study projects such as `V4L2`, `DRM`, `ALSA ASoC`, `Input`, `netdev`
- board bring-up projects such as `MMC host`, `clock`, `reset`, `GPIO`, `IRQ controller`, `pinctrl`, `power management`
- mixed projects such as `bus + function driver + board-level landing`
- engineering issue projects where the real goal is driver localization and debugging

Default project location:

- `调试问题/<项目目录名>/`

Directory naming:

- use the subsystem or driver family name directly when possible
- use an explicit topic name for mixed projects

## 2. Default Chapter Set

Every new driver study project should start with the `00-09` minimum set:

1. `00-<项目>源码解析总览.md`
2. `01-<项目>核心对象.md`
3. `02-<项目>初始化或枚举主线.md`
4. `03-<项目>总线匹配与probe.md`
5. `04-<项目>数据通路与常用API.md`
6. `05-<项目>中断或完成路径.md`
7. `06-<项目>板级落地或平台接入.md`
8. `07-典型驱动例子.md`
9. `08-<项目>调试与阅读建议.md`
10. `09-<项目>实际工程问题问答.md`

Optional expansion for complex projects:

11. `10-<项目>生命周期.md`
12. `11-<项目>对象关系图.md`
13. `12-<项目>回调关系.md`
14. `13-<项目>资源模型.md`
15. `14-<项目>上下文与并发模型.md`
16. `15-<项目>用户态到内核路径.md`

Add specialized chapters when needed:

- topology, graph building, pipeline start for graph or pipeline projects
- UAPI and user-tool chapters when there is a strong user interface layer
- staged boot-chain chapters for bootloader, kernel, and rootfs projects

## 3. Per-Chapter Structure

### 3.1 Chapter `00`

`00` should contain:

- `## 学习目标`
- `## 导读`
  - `### 本章定位`
  - `### 核心对象`
  - `### 关键函数`
  - `### 主流程`
- `## 推荐源码入口`
- `## 阅读顺序`
- `## 最重要的一条主线`
- `## 对象关系总图`
- `## 生命周期总览`
- `## 这套笔记回答的问题`

### 3.2 Ordinary chapters

Ordinary chapters should contain:

- `## 导读`
  - `### 本章定位`
  - `### 核心对象`
  - `### 关键函数`
  - `### 主流程`
- opening explanation of why the chapter follows this order or grouping
- object carry-forward rule: already-covered core objects should link back to earlier chapters; newly introduced private or example-specific objects should be explained in place
- chapter body
- explanation sections after diagrams or code samples when they appear
- `## 本章边界`
- `## 一句话总结` or an equivalent closing section

If a chapter uses a flow chart, relationship chart, or code skeleton, do not stop at the chart or sample itself.
Also do not jump straight into subpoints without first explaining why the chapter is split that way.
Add explanation that:

- states whether the chapter is organized by lifecycle, main-flow stage, object layer, upstream-downstream boundary, or engineering path
- explains why that organization is used before entering the subpoints
- maps the figure or sample back to the chapter's core objects
- explains each stage or key line in plain study-note wording
- shows how it connects to previous sections or the previous chapter
- clarifies the boundary to the next chapter when the chapter is part of a larger path

### 3.3 Closing chapters

Debug and engineering-Q&A chapters should contain:

- `## 学习目标`
- `## 导读`
  - `### 本章定位`
  - `### 核心对象`
  - `### 关键函数`
  - `### 主流程`
- problem layering or fixed debug order
- symptom-to-layer decision table
- `## 回答的问题`

## 4. New Project Initialization Steps

Use this order:

1. create the project directory
2. create `00`
3. create the `01-09` minimum set
4. decide whether `10-15` are needed
5. straighten the reading order and the main project flow in `00`
6. add recommended source entry points
7. add an object relationship diagram
8. fill the field-level object chapter
9. continue through enumerate, probe, I/O, IRQ, and example chapters
10. add lifecycle, callback, resource, and context chapters when needed
11. finish with debug and engineering-Q&A chapters

## 5. Completion Criteria

A project is structurally complete when it has:

- a total overview chapter
- recommended source entry points
- field-level core object notes
- an object relationship diagram
- a complete main flow
- lifecycle coverage
- callback relationship coverage
- resource model coverage
- context or concurrency coverage
- a user-to-kernel path when there is user space interaction
- a board or platform landing chapter
- at least one real driver example
- a debugging chapter
- an engineering-Q&A chapter
- note wording kept in learning-note style
