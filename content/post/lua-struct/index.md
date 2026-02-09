---
title: "Lua 数据类型与GC对象"
date: 2024-02-05T18:00:00+08:00
math: true
license: true
hidden: false
comments: true
draft: false
categories:
    - "lua"
tags:
    - "lua"
---
# Lua 源码手记：数据类型与隐藏的 GC 对象

最近在啃 Lua 的源码，主要想搞清楚一个问题：当我们写下 `local a = 1` 随后又写 `a = "hello"` 时，Lua 虚拟机底层到底发生了什么？

作为一个 Lua 开发者，我们习惯了“变量无类型，值有类型”的自由。但在 C 语言编写的虚拟机底层，内存是严苛且静态的。Lua 是如何在一个静态的 C 结构体中塞进千变万化的动态类型的？哪些数据是“活”在栈上的，哪些又是需要垃圾回收器（GC）费心照顾的？

本文将带你通过“显微镜”视角，从底层的 `TValue` 结构开始，重新审视 Lua 的数据类型系统与内存管理策略。

## 一、 源码揭秘：万物之容器 TValue

在 Lua 虚拟机中，并没有“整数变量”或“字符串变量”的概念。无论是全局变量、局部变量，还是 Table 中的一个槽位，在底层 C 源码中，它们都由一个统一的数据结构表示：**`TValue`**。

`TValue` 的设计完美体现了 Lua 的核心哲学：**Tagged Union（带标签的联合体）**。

### 1.1 核心结构定义

我们可以将 `TValue` 想象成一个标准化的“通用快递盒”，它由两部分组成：

1. **标签 (Tag)**：说明书，标记盒子里装的是什么类型。
2. **值 (Value)**：实际货物，或者指向货物的取货单（指针）。

让我们看看 Lua 5.3+ 源码中的核心定义（位于 `lobject.h`）：

```
/* 通用值联合体：同一时间只能存一种数据，保证内存紧凑 */
typedef union Value {
  GCObject *gc;    /* collectable objects: 复杂对象（Table, String...）存指针 -> 指向堆内存 */
  void *p;         /* light userdata: 存纯指针 -> 直接存地址 */
  int b;           /* boolean: 存 0 或 1 -> 直接存整数 */
  lua_CFunction f; /* light C functions: 轻量级 C 函数 */
  lua_Integer i;   /* integer numbers: 整数 */
  lua_Number n;    /* float numbers: 浮点数 */
} Value;

/* Lua 中的通用值结构 */
#define TValuefields  Value value_; int tt_

typedef struct TValue {
  TValuefields;
} TValue;
```

### 1.2 它是如何工作的？

当你写 `local a = 10` 时，Lua 虚拟机做的事情是：

1. 找到 `a` 对应的栈槽位（一个 `TValue` 内存块）。
2. 将 `tt_` (标签) 设为 `LUA_TNUMINT`。
3. 将 `value_.i` (数据) 设为 `10`。

当你随后写 `a = {}` 时：

1. 虚拟机在**堆内存**中创建一个新的 Table 对象。
2. 将 `a` 的 `tt_` 更新为 `LUA_TTABLE`。
3. 将 `value_.gc` 更新为指向那个 Table 的指针。

这种设计使得 Lua 的变量可以承载任何类型，因为它们本质上只是一个携带了类型标签的容器。

## 二、 Lua 的 8 种基本数据类型：内存视角

基于 `TValue` 的存储方式，我们可以将 Lua 暴露给开发者的 8 种数据类型分为三大类。这直接关系到赋值操作的性能和 GC 的压力。

### 1. 值类型 (Value Types) —— 栈上的原住民

这些类型的数据非常小，直接存储在 `TValue` 结构体内部。

- **nil (空)**：表示“无效值”。全局变量默认值。赋值 `nil` 等同于删除。
- **boolean (布尔)**：只有 `true` 和 `false`。
- *注意*：Lua 中只有 `false` 和 `nil` 为假，数字 `0` 是真。
- **number (数值)**：
- Lua 5.3+ 分为 Integer (64位整型) 和 Float (双精度浮点)。
- VM 会自动处理转换，通常无需感知。
- **light userdata (轻量用户数据)**：
- **特例**：它本质上是一个纯粹的 C 指针 (`void *`)。Lua 把它当作一个数字处理，**不管理**它的生命周期（生死由 C 端代码负责）。这是高性能交互的利器。

### 2. GC 类型 (GCObjects) —— 堆上的住客

这些类型数据量大且结构复杂，`TValue` 只存指针（`value_.gc`）。

- **string (字符串)**：
- 不可变字符序列。
- **String Interning (字符串驻留)**：Lua 会对短字符串进行唯一化处理，相同内容的短字符串在全局只有一份内存，比较时只需比较指针地址，速度极快。
- **table (表)**：
- Lua 唯一的数据结构。
- 既是数组（Array，下标从1开始），也是字典（Hash Map）。
- **function (函数)**：
- 第一类值（First-class）。
- 包含 C 函数和 Lua 函数（闭包）。
- **thread (线程)**：
- 协程 (Coroutine) 的实现基础。
- **userdata (完全用户数据)**：
- 由 Lua 虚拟机分配内存的一块原始内存区域，支持 `__gc` 元方法析构。

## 三、 谁在参与 GC？（内存管理揭秘）

在 Lua 中，并不是所有变量都需要垃圾回收器（Garbage Collection, GC）操心。

### 3.1 GCObject 的秘密

所有需要被 GC 管理的对象，在 C 层面都有一个共同的头部，叫做 `CommonHeader`。

```
/* 所有 GC 对象通用的头部 */
#define CommonHeader	GCObject *next; lu_byte tt; lu_byte marked

struct GCObject {
  CommonHeader;  /* 包含：链表指针、类型标签、GC 颜色标记 */
};
```

这就解释了为什么 Lua 可以统一管理不同类型的对象：因为它们强转为 `GCObject*` 后，头部结构是一样的，GC 只需要扫描这个头部链表即可。

### 3.2 区分标准

- **不参与 GC**：`nil`, `boolean`, `number`, `light userdata`。它们随栈帧生灭，无需 GC 介入。
- **参与 GC**：`string`, `table`, `function`, `userdata`, `thread`。它们在堆上分配，需要通过标记-清除（Mark-and-Sweep）算法回收。

## 四、 隐形的 GC 对象：Proto 与 UpVal

除了上述我们在 Lua 代码中能直接操作的类型外，Lua 虚拟机底层还有两个非常关键的内部对象也是由 GC 管理的。**理解它们，是掌握 Lua 闭包 (Closure) 机制的钥匙。**

### 1. Proto (函数原型) —— 静态的蓝图

你可以把 `Proto` 看作是**“函数的编译模具”**。

- **它是什么**：当 Lua 编译器编译一段代码时，生成的字节码（Bytecode）、常量表（Constants）、调试信息等**静态数据**，都被打包存储在一个 `Proto` 对象中。
- **生命周期**：虽然它是静态的，但当你使用 `load` 加载一段代码块时，系统会生成对应的 `Proto`。如果这个模块被卸载，或者没有任何闭包再使用这个模具，`Proto` 就需要被回收。

### 2. UpVal (上值对象) —— 闭包的灵魂

这是 Lua 闭包机制最精妙的地方。`UpVal` 是独立于函数栈帧存在的对象，用于存储闭包捕获的外部局部变量。

- **为什么需要它？** 当一个函数返回时，它的栈帧会被销毁，原本存在栈上的局部变量也就消失了。如果内部函数（闭包）还引用了这个变量，怎么办？
- **Open vs Closed 状态**：
- **Open (开放)**：当父函数还在运行时，被捕获的变量还在栈上。此时 `UpVal` 只是一个指针，指向栈上的那个位置。
- **Closed (关闭)**：当父函数返回，栈即将销毁时，Lua 会将变量的值**复制**到 `UpVal` 对象内部（堆内存），并将 `UpVal` 指针指向自己。这就实现了“变量逃逸”。

**公式类比：**

> **Lua Function (运行时闭包) = Proto (静态代码逻辑) + UpVals (动态环境上下文)**

这就是为什么两个闭包可以使用同一份代码（同一个 Proto），却拥有不同的状态（不同的 UpVals）。

## 五、 总结与性能建议

通过重新审视 `TValue` 和 GC 对象，我们可以得出一些写出高性能 Lua 代码的原则：

1. **赋值开销**：
- `number/boolean` 赋值是内存拷贝（极快）。
- `table/function` 赋值是指针拷贝（极快）。
- **但是**，创建 `table` (`{}`) 或闭包 (`function() end`) 是堆内存分配，相对较慢。
2. **Light Userdata 的妙用**： 如果你只需要在 Lua 里持有一个 C++ 对象的句柄，且生命周期由 C++ 管理，请务必使用 `light userdata`。它可以完全避开 GC 的扫描和追踪开销。
3. **闭包的代价**： 每创建一个新的闭包，不仅会创建一个 `Function` 对象，如果涉及新的外部变量捕获，还可能创建新的 `UpVal` 对象。在极高频调用的场景下（如游戏每帧 Update），应尽量避免在循环内创建匿名函数。
4. **GC 压力**： 字符串拼接（产生新 String 对象）和表的频繁创建销毁是 GC 卡顿的两大元凶。在性能敏感区，复用 Table（Object Pooling）永远是王道。

