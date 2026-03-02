---
title: "Lua 数据类型与GC对象"
date: 2026-02-05T18:00:00+08:00
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

## TValue

在 Lua 虚拟机中，并没有“整数变量”或“字符串变量”的概念。无论是全局变量、局部变量，还是 Table 中的一个槽位，在底层 C 源码中，它们都由一个统一的数据结构表示：**`TValue`**。

### 核心结构定义

 `TValue` 由两部分组成：

1. **标签 (Tag)**
2. **值 (Value)**

Lua 5.3+ 源码中的核心定义（ `lobject.h`）：

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

### 它是如何工作的？

当你写 `local a = 10` 时，Lua 虚拟机做的事情是：

1. 找到 `a` 对应的栈槽位（一个 `TValue` 内存块）。
2. 将 `tt_` (标签) 设为 `LUA_TNUMINT`。
3. 将 `value_.i` (数据) 设为 `10`。

当你随后写 `a = {}` 时：

1. 虚拟机在**堆内存**中创建一个新的 Table 对象。
2. 将 `a` 的 `tt_` 更新为 `LUA_TTABLE`。
3. 将 `value_.gc` 更新为指向那个 Table 的指针。

这种设计使得 Lua 的变量可以承载任何类型，因为它们本质上只是一个携带了类型标签的容器。

## Lua 的 8 种基本数据类型

### 值类型 (Value Types) —— 栈上

这些类型的数据非常小，直接存储在 `TValue` 结构体内部。

- **nil (空)**：表示“无效值”。全局变量默认值。赋值 `nil` 等同于删除。
- **boolean (布尔)**：只有 `true` 和 `false`。
  - *注意*：Lua 中只有 `false` 和 `nil` 为假，数字 `0` 是真。
- **number (数值)**：
- **light userdata**：
  - 它本质上是一个纯粹的 C 指针 (`void *`)。Lua 把它当作一个数字处理，**不管理**它的生命周期（由 C 代码负责）。

### GC 类型 (GCObjects) —— 堆上

这些类型数据量大且结构复杂，`TValue` 只存指针（`value_.gc`）。

- **string (字符串)**：
  - Lua 会对短字符串进行唯一化处理，相同内容的短字符串在全局只有一份内存，比较时只需比较指针地址，速度极快。
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

## 谁在参与 GC？

### GCObject

所有需要被 GC 管理的对象，在 C 层面都有一个共同的头部，叫做 `CommonHeader`。

```
/* 所有 GC 对象通用的头部 */
#define CommonHeader	GCObject *next; lu_byte tt; lu_byte marked

struct GCObject {
  CommonHeader;  /* 包含：链表指针、类型标签、GC 颜色标记 */
};
```

这就解释了为什么 Lua 可以统一管理不同类型的对象：因为它们强转为 `GCObject*` 后，头部结构是一样的，GC 只需要扫描这个头部链表即可。


- **不参与 GC**：`nil`, `boolean`, `number`, `light userdata`。它们随栈帧，无需 GC 介入。
- **参与 GC**：`string`, `table`, `function`, `userdata`, `thread`。它们在堆上分配，需要通过标记-清除（Mark-and-Sweep）算法回收。

## 额外的 GC 对象：Closure, Proto 与 UpVal

### Closure (闭包对象)

这是我们在 Lua 运行时真正在操作的“函数实体”。无论是全局函数、局部函数还是匿名函数，在 Lua 虚拟机眼中全都是 `Closure`（闭包）。

- **职责**：它是连接静态代码和动态运行环境的桥梁。为了直观理解它的本质，我们可以看看 Lua 源码（C语言）中对于 Lua 闭包（`LClosure`）的结构定义：

```c
typedef struct LClosure {
    ClosureHeader;         // GC 对象的公共头部信息
    struct Proto *p;       // 指向函数原型 (静态的字节码、常量等)
    UpVal *upvals[1];      // 指向 UpVal 对象的指针数组 (动态的环境上下文)
} LClosure;

typedef union Closure {
    CClosure c;            // C 语言闭包
    LClosure l;            // Lua 语言闭包
} Closure;
```

从源码中可以清晰地看到，一个 `Closure` 对象内部最核心的就是两个指针：一个指向该函数的 `Proto`，另一个（数组）指向它所捕获的一个或多个 `UpVal`。
- **生命周期**：每次代码执行到 `function() ... end` 时，都会在堆上动态分配一个新的 `Closure` 对象。当没有任何变量引用这个闭包时，它会被 GC 回收。



### Proto (函数原型)

当 Lua 编译器编译一段代码时，生成的字节码（Bytecode）、常量表（Constants）、局部变量名表等**静态数据**，都被打包存储在一个 `Proto` 对象中。

- **职责**：提供执行指令。无论基于这个原型创建了多少个 `Closure` 实例，它们在内存中都共享同一份 `Proto`，以极大地节省内存。
- **生命周期**：虽然它是静态数据的载体，但它本身也是 GC 对象（例如使用 `load` 动态加载的代码块会生成对应的 `Proto`）。只有当所有引用该 `Proto` 的闭包都被销毁，且模块被卸载时，`Proto` 才会被回收。

### UpVal (上值对象)

`UpVal` 是独立于函数栈帧存在的对象，专门用于存储闭包捕获的外部局部变量，让局部变量的生命周期得以延续。

看一个经典的例子：

```lua
function create_counter()
    local count = 0  -- 这是一个局部变量
    
    -- 闭包 A (f1)
    local function inc()
        count = count + 1
        return count
    end
    
    -- 闭包 B (f2)
    local function get()
        return count
    end
    
    return inc, get
end

local f1, f2 = create_counter()
```

在这个例子中：
* `count` 本来是分配在 `create_counter` 的栈（Stack）上的。
* `f1` (inc) 和 `f2` (get) 都是运行时生成的 `Closure` 对象，它们都需要访问同一个 `count`。如果 `f1` 修改了它，`f2` 必须能看到变化。

**Lua 是怎么做到的？**
Lua 会在堆上创建一个 `UpVal` 对象来管理这个跨越作用域的变量，它有两种状态转换：
* **Open**： 当 `create_counter` 还在运行时，`count` 还在栈上。此时的 `UpVal` 对象仅仅是一个指针，直接指向栈上的 `count` 内存位置。
* **Closed**： 当 `create_counter` 执行完毕返回时，栈帧即将被销毁，此时 Lua 虚拟机会触发 "Close" 操作：把栈上 `count` 的值复制到堆上的 `UpVal` 对象内部，并将指针指回自己。

多个闭包可以共享同一个 `UpVal` 对象（如上例的 f1 和 f2）。

> **Closure (运行时闭包实例) = Proto (共享的静态代码逻辑) + UpVals (独占/共享的动态环境上下文)**


但相对的，每次执行到函数定义处，都会在堆上分配一个新的 `Closure` 对象；如果涉及捕获新的外部变量，还会触发新 `UpVal` 对象的创建。因此，在极高频调用的场景下（例如游戏每帧的 `Update` 循环），应尽量避免在循环内部频繁定义匿名函数，以减轻 GC 的内存分配与扫描压力。