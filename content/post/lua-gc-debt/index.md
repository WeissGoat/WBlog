---
date: '2026-02-21T11:04:22+08:00'
draft: false
image: ""
math: true
license: true
hidden: false
comments: true
title: "Lua 增量GC 演进：从阈值模型到 Debt (债务) 模型"
categories:
    - "Lua"
tags:
  - "Lua"
  - "GC"
---

从 Lua 5.1 到 Lua 5.4，GC 的核心驱动力发生了一次重要的转变：从原先的**“阈值模型”（Threshold）**转向了**“债务模型”（Debt）**。

这是为了解决增量 GC 在实际运行中遇到的步长控制和内存飙升问题。

## 1. 之前的版本（Lua 5.1）是如何工作的？——“阈值模型”

在引入 Debt 之前（主要是 Lua 5.1），GC 的触发逻辑是基于**阈值（Threshold）**的。

* **核心逻辑**：设定一个内存阈值（通常是当前使用内存的一定倍数，比如 2 倍）。
* **触发条件**：当 `totalbytes`（当前总分配内存） > `GCthreshold`（设定的阈值）时，触发 GC 工作。

### 步长（Step）的计算方式
在旧版中，GC 每次“步进”的工作量是一个相对固定的常数，与程序刚才到底分配了多少内存没有直接联系。
* **基准步长**：源码中硬编码了一个宏 `LUA_GCSTEPSIZE`，通常定义为 1KB (1024 字节)。
* **倍率调整**：工作量会乘以 `stepmul`（默认 200%）。
* **最终工作量**：`StepWork = 1KB * 2 = 2KB`

> ```c
> if (g->totalbytes >= g->GCthreshold) {
>   luaC_step(L);
> }
> 
> // lgc.c 中的单步工作量计算
> #define GCSTEPSIZE 1024u
> // ...
> l_mem debt = (cast(l_mem, GCSTEPSIZE) * g->gcstepmul) / 100;
> ```

### 阈值模型带来的问题
1.  反应滞后，容易导致内存突然冲高, 高负载下容易飙升，GC 往往追不上分配速度。
2.  粒度太粗，在一次回收周期中, 每次 Step 时间固定，但为了追回内存可能触发得更频繁; 没到阈值则完全停着。

---

## 2. 什么是 "GC Debt"（GC 债务）？

从 Lua 5.2 开始，直到 Lua 5.4，GC 的核心驱动力变成了 **Debt（债务）**。

* **核心逻辑**：你每分配 1 字节的内存，Debt 就会增加 1。这就叫“按需计费”（Pay-as-you-go）。
* **触发条件**：
    * `GCdebt > 0`：GC 正在运行（欠债了，快干活）。
    * `GCdebt < 0`：GC 处于暂停/睡眠状态（预存了额度，可以先不干活）。

当 GC 完成一轮完整的回收（Full Cycle）后，它会计算存活对象的真实大小，并结合 `pause` 参数，将 `GCestimate` 设置得比当前实际内存大很多。此时，`GCdebt` 就变成了一个巨大的**负数**。随着程序继续分配内存，`totalbytes` 不断增加，`GCdebt` 慢慢从负数增长，直到突破 0，此时 GC 自动“醒来”开始工作。

### 步长（Step）的计算方式
新版的步长不再是固定死的大小，而是与你欠的“债”息息相关。它同样受 `stepmul`（默认 200%）影响：
`需要回收的工作量 = debt * (stepmul / 100)`

这意味着一种非常直观的对应关系：**“程序每分配 1KB 内存，GC 就应该扫描 2KB 的数据”**。

### 核心公式

`GCdebt = totalbytes - GCestimate`

* `totalbytes`: 当前实际分配的内存总数。
* `GCestimate`: GC 认为“应该”有的内存基准线（一轮回收结束后的存活数据大小预测值）。

> ```c
> void luaC_step (lua_State *L) {
>   global_State *g = G(L);
>   l_mem debt = g->GCdebt;  // 获取当前欠债
>   if (debt > 0) {
>     // 根据债务和步进倍率计算当前需要执行的回收工作量
>     l_mem work = (debt * g->gcstepmul) / 100;
>     // 执行实际的标记/清扫工作，并扣除已完成的工作量...
>   }
> }
> ```
**核心思想**：每当你分配 1 Byte 的内存，你就欠了 GC 1 Byte 的“债”。GC 的工作就是不断地运行，通过扫描和清理对象来“还债”，直到债务归零（或变为负数继续休眠）。

### 优势
* 让“内存分配速率”和“GC 回收速率”在微观上实现了数学级别的精确挂钩。分配得越快，欠债越多，当次 GC step 的工作量就越大。
* 对 Spike（突发分配）的反应极其灵敏，提供实时反馈，立刻加大 GC 工作量应对。

---
