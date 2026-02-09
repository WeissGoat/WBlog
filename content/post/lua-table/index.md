---
title: "Lua Table"
date: 2024-02-05T18:00:00+08:00
math: true
license: true
hidden: false
comments: true
draft: false
categories:
    - "Lua"
tags:
    - "Lua"
---
# Lua Table 源码剖析：混合数据结构的艺术

在 Lua 的设计哲学中，Table 是唯一的复合数据结构。它既要扮演数组（Array）的角色，又要胜任哈希表（Hash Map）的工作。这种“万金油”的特性背后，隐藏着非常精妙的设计思路。

最近在研究 Lua 源码（基于 Lua 5.3/5.4 逻辑），整理了一些关于 Table 内部实现的笔记，在此记录并分享一下关于其数据结构、内存布局、冲突解决以及 Rehash 机制的探索。

## 1. 核心数据结构：混合存储的基石

Lua Table 的高效由其底层的 `struct Table` 决定。从源码中我们可以清晰地看到，它并非单纯的哈希表，而是**数组**与**哈希表**的结合体。

### 1.1 Table 结构体定义

```
typedef struct Table {
    CommonHeader;
    lu_byte flags;      /* 1<<p means tagmethod(p) is not present (元方法缓存) */
    lu_byte lsizenode;  /* 哈希表长度的对数，即 hash_size = 2^lsizenode */
    unsigned int alimit;/* 数组部分的“逻辑”大小 (即 sizearray) */
    TValue *array;      /* 数组部分指针 */
    Node *node;         /* 哈希桶数组起始指针 */
    Node *lastfree;     /* 指向哈希部分最后一个空闲位置，用于快速插入 */
    struct Table *metatable; /* 元表 */
    GCObject *gclist;
} Table;
```

**设计解读：**

- **双重存储**：`array` 指针管理连续内存，用于存放整数键（1..n）；`node` 指针管理哈希桶，用于存放其他键或离散的整数键。
- **空间换时间**：`lastfree` 指针的设计非常巧妙，它避免了每次插入时线性扫描寻找空位，将查找空闲位置的复杂度尽量降低。
- **lsizenode**：哈希表的大小始终保持为 2 的幂次，这里存储的是幂次（log2），省去了存储实际大小的内存，计算时位移即可。

### 1.2 节点（Node）与键（Key）

哈希部分的节点结构如下：

```
typedef union TKey {
    struct {
        TValuefields;
        struct Node *next; /* 链表法解决冲突，指向下一个冲突节点 */
    } nk;
    TValue tvk;
} TKey;

typedef struct Node {
    TValue i_val;
    TKey i_key;
} Node;
```

这里 `Node` 包含了 Key 和 Value。Key 是一个 union，为了内存对齐和访问效率，当发生哈希冲突时，通过 `next` 指针形成单向链表。

## 2. 键的归属：数组还是哈希？

当我们执行 `t[k] = v` 时，Lua 是如何决定把数据存在 `array` 还是 `node` 里的？

### 2.1 整数键的“VIP 通道”

一般情况下，正整数键（如 `t[1]`, `t[100]`）会优先尝试放入数组部分。但并非所有整数都会进入数组，必须满足**利用率原则**。

> **核心规则**：包含当前 key 的区间 $[1, 2^n]$ 内，非空元素数量必须大于 $2^{n-1}$（即利用率 > 50%）。

**源码佐证 (`ltable.c`)**： Lua 通过 `computesizes` 函数来计算数组的最佳大小，代码清晰地展示了“> 50%”的判断逻辑：

```
/* 计算数组部分的最佳大小 */
static unsigned int computesizes (unsigned int nums[], unsigned int *narray) {
    int i;
    unsigned int twotoi;  /* 2^i */
    unsigned int a = 0;   /* 统计小于 2^i 的元素个数 */
    unsigned int na = 0;  /* 最终落入数组部分的元素个数 */
    unsigned int optimal = 0;  /* 最佳数组大小 */

    /* 循环遍历每个 2^i 区间 */
    for (i = 0, twotoi = 1; twotoi/2 < *narray; i++, twotoi *= 2) {
        if (nums[i] > 0) {
            a += nums[i];
            if (a > twotoi/2) {  /* 核心判断：利用率 > 50% */
                optimal = twotoi;
                na = a;
            }
        }
    }
    *narray = optimal;
    return na;
}
```

**举个例子**： 如果你创建了一个表 `t = {[1]=1, [1000]=1}`。

- `1` 在数组部分。
- `1000` 虽然是整数，但如果强行扩容数组到 1000，中间会有 998 个空洞，不满足 `a > twotoi/2`。因此，`1000` 会被当作普通 Key 扔进哈希部分。

### 2.2 哈希部分的“收容所”

以下情况 Key 会进入哈希表：

1. **非整数类型**：字符串、Table、Userdata 等。
2. **越界的整数**：数值极大，或者像上面提到的过于稀疏的整数。
3. **负数**。

## 3. 冲突解决：Brent's Method 的变种

这是 Lua Table 实现中最“骚”的操作之一。

通常哈希表解决冲突用的是**开链法（Open Addressing with Chaining）**，但 Lua 为了节省内存，并没有在这个链表之外单独分配内存，而是直接利用哈希桶数组中未被使用的槽位。

### 3.1 插入逻辑（Insert）

当我们要插入一个新的 Key (`new_key`)，算出它应该在的位置 `main_pos`，却发现那里已经有“人”了 (`old_node`)，Lua 会根据 `old_node` 的身份采取不同策略。

**源码佐证 (`luaH_newkey`)**：

```
/* 插入新键的核心逻辑 (简化版) */
TValue *luaH_newkey (lua_State *L, Table *t, const TValue *key) {
    Node *mp;
    if (ttisnil(key)) luaG_runerror(L, "table index is nil");

    /* 1. 计算主位置 (Main Position) */
    mp = mainposition(t, key);

    /* 2. 如果主位置已被占用 */
    if (!ttisnil(gval(mp)) || mp == dummynode) {
        Node *othern;
        /* 获取空闲位置 (lastfree) */
        Node *f = getfreepos(t);
        if (f == NULL) { /* 没位置了？扩容 */
            rehash(L, t, key);
            return luaH_set(L, t, key);
        }

        /* 计算占位者(old_node)原本该在的主位置 */
        othern = mainposition(t, keyfromval(mp));

        if (othern != mp) {
            /* Case A: 鸠占鹊巢 -> 把 old_node 赶到 free pos */
            while (othern + gnext(othern) != mp)
                othern += gnext(othern); /* 找到指向 mp 的前驱 */
            
            gnext(othern) = cast_int(f - othern); /* 修正链表指向新位置 f */
            *f = *mp; /* 搬迁数据 */
            
            /* 腾出 mp 给新键，因为 mp 是新键的主位置 */
            gnext(mp) = 0;
            setnilvalue(gval(mp));
        }
        else {
            /* Case B: 原住民冲突 -> 新键只能去 free pos */
            if (gnext(mp) != 0)
                gnext(f) = cast_int((mp + gnext(mp)) - f); /* 头插法：链在 mp 后面 */
            else gnext(f) = 0;
            
            gnext(mp) = cast_int(f - mp);
            mp = f;
        }
    }
    
    setnodekey(L, &mp->i_key, key);
    return gval(mp);
}
```

- **情况 A：鸠占鹊巢**
- 如果 `old_node` 是被挤到这里的（即 `othern != mp`）。
- **操作**：Lua 把它挪到 `lastfree`，把 `main_pos` 抢回来给 `new_key`。
- **目的**：保证 `new_key` 这种“原住民”能直接找到位置，减少后续查找的开销。
- **情况 B：原住民冲突**
- 如果 `old_node` 本来就该待在这里。
- **操作**：`new_key` 去 `lastfree`，并链接在 `old_node` 后面。

这种机制被称为 **Chained Scatter Table**，它充分利用了数组空间，避免了由于大量微小对象分配导致的内存碎片。

## 4. 动态扩容：Rehash 的艺术

Table 的空间不是固定的。当空间不足或利用率失衡时，会触发 Rehash。

### 4.1 触发时机

- **插入新键且空间已满**：当我们赋值一个新键，而 Table 既没有数组空间，也没有哈希空位时。
- **注意**：将某个 Key 置为 `nil` **不会**立即触发 Rehash（为了防止频繁的内存抖动）。只有在空间不足需要重新评估时，才会回收 `nil` 占用的空间。

### 4.2 Rehash 的三部曲

Rehash 是一个相对昂贵的操作，Lua 必须一次性重新计算数组和哈希两部分的最佳大小。

1. **统计（Counting）**： 遍历整个 Table（包括现有的 Array 和 Hash 部分），统计所有正整数键的数量，并按 $2^n$ 的区间（1-2, 3-4, 5-8...）进行分桶统计。同时统计非整数键的总数。
2. **定界（Sizing）**： 计算数组部分的最佳大小 `sizearray`。
- 算法：找到最大的 $N$（$2$ 的幂），使得 $[1, N]$ 区间内的整数键数量 $\ge N/2$。
- 凡是小于等于 $N$ 的整数放入新数组，其余放入新哈希表。
3. **搬迁（Relocation）**：
- 申请新的 Array 和 Hash 内存块。
- 将数据搬运过去（重新计算哈希值）。
- 释放旧内存。

这解释了为什么 Lua Table 既能像 C 数组一样紧凑高效，又能像 Python Dict 一样灵活。

## 5. 迭代器的秘密：pairs 与 ipairs

我们在写 Lua 时常用的两种遍历方式，其底层性能差异巨大。

### 5.1 ipairs：序列专用

`ipairs` 是为**序列（Sequence）**量身定做的。

- **逻辑**：维护一个内部计数器 `i`，从 1 开始递增。
- **寻址**：Lua 内部会先判断 `i` 是否在 `array` 的范围内。

**源码佐证 (`luaH_getint`)**： 这是整数查找的入口函数，清晰展示了“先查数组，再查哈希”的逻辑：

```
const TValue *luaH_getint (Table *t, lua_Integer key) {
    /* 1. 数组部分快速查找 (Fast Path) */
    /* 这里的 alimit 就是数组的逻辑大小 */
    if (l_castS2U(key) - 1 < t->alimit)
        return &t->array[key - 1];
    else {
        /* 2. 哈希部分查找 (Slow Path) */
        Node *n = hashint(t, key);
        for (;;) {
            if (ttisinteger(gkey(n)) && ivalue(gkey(n)) == key)
                return gval(n); /* 找到了 */
            else {
                int nx = gnext(n);
                if (nx == 0) break;
                n += nx; /* 沿着冲突链查找 */
            }
        }
        return luaO_nilobject;
    }
}
```

- **在范围内**：直接通过指针偏移 `array[i]` 读取，这几乎就是 C 语言访问数组的速度，**极快**。
- **不在范围内**：才去查 Hash 表。

### 5.2 pairs：全量遍历

`pairs` 使用的是 `lua_next` 函数。

- **流程**：
1. **先遍历数组部分**：从下标 0 到 `sizearray` 线性扫描。
2. **再遍历哈希部分**：从哈希桶的第一个位置扫描到最后一个位置。
- **顺序问题**：
- 对于**纯数组**表（如 `{10, 20, 30}`），因为只扫描数组部分，输出看起来是有序的。
- 一旦涉及哈希部分，顺序就是**乱序**的。这是因为 Key 在哈希表中的物理位置取决于 Hash 算法和冲突时的插入顺序，与逻辑顺序无关。

**避坑指南**：

> 不要依赖 `pairs` 的遍历顺序，除非你确认表是纯数组且尚未发生过复杂的 Rehash。

## 6. 总结

Lua Table 的源码实现体现了极强的**工程实用主义**：

1. **混合结构**平衡了数组的 $O(1)$ 访问和哈希表的稀疏灵活性。
2. **50% 利用率原则**保证了数组空间不会被过度浪费。
3. **lastfree 冲突解决**最大化利用了预分配内存，减少了 `malloc` 调用的开销。
