---
layout: post
title: "扒一扒 Objective-C Runtime Weak 源码"
description: ""
categories: iOS
tags: [Runtime]
---

## 背景

现在对于一个 iOS 开发人员来说如果不看一些 Runtime 的源码都不好意思出去面试. 那么 Runtime 中经常被问到的除了 Method Swizzling 大概就是 Weak 的实现了吧. 网上搜一下, 所讲的内容基本上都是大同小异, 那么抛开面试的角度, 再去扒一扒 Runtime 中 Weak 的相关源码, 我们又能学到什么新鲜的内容呢? 这里我将尝试从其他角度对源码进行解读, 欢迎大家一起探讨.

## 准备工作

本文采用的 Runtime 源码版本是 750, [官方版本](https://opensource.apple.com/tarballs/objc4/)在官网上可以下载到, 本文使用的是 GitHub 热心网友提供的[可编译版本](https://github.com/RetVal/objc-runtime), 使用 Xcode 10.1 即可编译运行. 当然 Runtime 的源码都是 C/C++ 写的, 所以熟悉一些 C++ 编程知识还是很有必要的.

## 管中窥豹

### 从注释说起

为什么从注释说起? 注释也是代码的重要组成部分, 特别是对于一些逻辑较为复杂的代码, 重要的核心代码, '神奇'代码, 对外暴露的接口等, 注释是很好的补充, 对于研究和接手代码的人来说, 是你留下的宝贵财产. 我们知道 Apple 的 Runtime 源码是开源的, 那么开源代码是怎么写注释的呢? 对于我们又有什么借鉴意义呢?  
打开 `objc-weak.h` 头文件, 我们来看几个例子:

#### 设计说明

```c++
/*
The weak table is a hash table governed by a single spin lock.
An allocated blob of memory, most often an object, but under GC any such 
allocation, may have its address stored in a __weak marked storage location 
through use of compiler generated write-barriers or hand coded uses of the 
register weak primitive. Associated with the registration can be a callback 
block for the case when one of the allocated chunks of memory is reclaimed. 
The table is hashed on the address of the allocated memory.  When __weak 
marked memory changes its reference, we count on the fact that we can still 
see its previous reference.

So, in the hash table, indexed by the weakly referenced item, is a list of 
all locations where this address is currently being stored.
 
For ARC, we also keep track of whether an arbitrary object is being 
deallocated by briefly placing it in the table just prior to invoking 
dealloc, and removing it via objc_clear_deallocating just prior to memory 
reclamation.

*/
```

尝试翻译下: 弱引用表是由单个自旋锁控制的哈希表(线程安全). 一个已分配的内存块(通常是一个对象, GC 下可以是任何开辟的内存)，通过使用编译器生成的 `write-barriers` 或手动编码注册弱`原语`可以将其地址存储在 `__weak` 标记的存储位置...

#### 类型说明

```c++
// The address of a __weak variable.
// These pointers are stored disguised so memory analysis tools
// don't see lots of interior pointers from the weak table into objects.
typedef DisguisedPtr<objc_object *> weak_referrer_t;
```

这里很好的解释了为什么要对 `objc_object *` 包一层的原因, 大意就是内存分析工具只能要看到值而看不到内部结构, 因为通过指针是能直接访问到该内存区域的.

#### 类/结构体说明

```c++
/**
 * The global weak references table. Stores object ids as keys,
 * and weak_entry_t structs as their values.
 */
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};
```

这里解释了 `weak_table_t` 这个结构体的作用.

#### 常量说明

```c++
// out_of_line_ness field overlaps with the low two bits of inline_referrers[1].
// inline_referrers[1] is a DisguisedPtr of a pointer-aligned address.
// The low two bits of a pointer-aligned DisguisedPtr will always be 0b00
// (disguised nil or 0x80..00) or 0b11 (any other address).
// Therefore out_of_line_ness == 0b10 is used to mark the out-of-line state.
#define REFERRERS_OUT_OF_LINE 2
```

这里把常量定义为 2 的来龙去脉都解释清楚了.

#### 接口说明

```c++
/// Adds an (object, weak pointer) pair to the weak table.
id weak_register_no_lock(weak_table_t *weak_table, id referent, 
                         id *referrer, bool crashIfDeallocating);

```

这里描述的也比较清晰, 将对象和弱指针对添加到弱引用表中. 这里有两个细节, `_no_lock` 后缀说明这个函数没有加锁操作是线程不安全的, 调用的时候需要注意是否要进行加锁处理. 另外 `crashIfDeallocating` 这个参数说明了对象释放的过程中调用 `weak_register_no_lock` 函数是否会触发 crash.

#### 小结

我们写业务代码的时候一般不太会重视注释, 包括一些提供给其他团队的 SDK 里面的注释有的也不是很全, 这里通过对开源代码的学习可以很好的借鉴下良好的注释应该怎么去编写和维护. 这样后续维护业务代码也会清晰很多, 我们也会少一些"技术客服"工作.  
另外, 之前很少去研读注释, 而是直接看代码. 仔细读了下注释之后, 会对代码的前世今生, 心路历程会有更好的了解, 说不定还有意外的收获.
回过头去 GitHub 上扫了眼常用的三方库, 发现那些受欢迎的开源库都对注释不惜笔墨.

### union 使用

#### 源码

```c++
struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };
    // ...
};
```

`weak_entry_t` 结构体中使用了 `union`, 此处为何使用 `union`? 另外 `struct` 中为何又使用了位域? 下面会逐个分析下.

#### union 的介绍

什么 union? 翻译过来叫共用体或者联合体, c/c++ 里面用的比较多, iOS 开发的时候用的比较少. 所以对于我个人来说还是比较陌生的. 那么它有什么好处? 我们先用一个例子来说明下:

```c++
struct s1 {
    char mark;
    int num;
    double score;
};
union u1 {
    char mark;
    int num;
    double score;
};
```

这里定义了一个结构体 s1, 一个联合体 u1. 在 Mac (x86_64) 上面我们分别 `sizeof(struct s1)` `sizeof(union u1)` 一下得到的结果是 16 和 8.  
对于 `s1` 在 x86_64 结构计算机上面, char 占 1 个字节, int 占 4 个字节, double 占 8 个字节, 因为 `struct` 会进行内存对齐, 所以 char 会向 int 对齐, 整个就是 4 + 4 + 8 = 16.
对于 `u1` 在 x86_64 结构计算机上面, 会以最宽的 `double` 作为所占大小就是 8.
也就是说 `union` 直观的一个好处就是省内存, 对于 `weak` 这种较为频繁的操作来说, 也是个不小的优化. 

但是使用 `union` 需要注意的是:  

1. 同一个内存段可以用来存放几种不同类型的成员, 但在每一个时刻只能存在其中一种, 而不是同时存在几种. 也就是说, 每一瞬间只有一个成员起作用, 其它的成员不起作用, 即不是同时都存在和起作用.
2. 共用体变量中起作用的成员是最后一个存放的成员, 在存入一个新的成员后, 原有的成员就失去作用.

所以使用 `union` 需要格外的小心, 不然比较容易出错.  

#### weak_entry_t 中 union 的分析

我们再来看看, `weak_entry_t` 中 `union` 的用法. 我们可以运行 Runtime 的工程源码, 断点查看.

```shell
(lldb) po sizeof(weak_entry_t)
40
```

我们来分析下:

- `DisguisedPtr<objc_object>` 占 8 个字节.
- `union` 中第一个 `struct` 中 `referrers` 占 8 个字节, `out_of_line_ness` 占 2 位, `num_refs` 占 62 位, 那么 `out_of_line_ness` 和 `num_refs` 加起来占 8 个字节, `mask` 占 8 个字节, `max_hash_displacement` 占 8 个字节. 所以总的占 8 * 4 = 32 个字节.
- `union` 中第二个 `struct` 中 `WEAK_INLINE_COUNT` 为 4, 那么 `inline_referrers` 占 32 个字节. 
- 两个 `struct` 都占 32 个字节, 所以 8 + 32 = 40 个字节. 

#### 位域的使用

我们看到源码中还有一个技巧就是位域的使用:

```c++
uintptr_t        out_of_line_ness : 2;
uintptr_t        num_refs : PTR_MINUS_2;
```

这样做的好处, 当然也是节约内存了, 因为 `out_of_line_ness` 本身只是一个标志位, 不需要存大数, 所以两位就够了, 我们从注释中就可以看出来:

```c++
// out_of_line_ness field overlaps with the low two bits of inline_referrers[1].
// inline_referrers[1] is a DisguisedPtr of a pointer-aligned address.
// The low two bits of a pointer-aligned DisguisedPtr will always be 0b00
// (disguised nil or 0x80..00) or 0b11 (any other address).
// Therefore out_of_line_ness == 0b10 is used to mark the out-of-line state.
#define REFERRERS_OUT_OF_LINE 2
```

也就是代码中只用到了 `out_of_line_ness == REFERRERS_OUT_OF_LINE` 进行判断, 两位就够了.
而对于 `num_refs` 引用计数来说用剩余的 62 位也够了. 对于内存的节约, 也是做到了极致了.

#### 设计合理性

我们知道 `union` 中同一时刻只有一个 `struct` 生效, 这里先看一下代码:

```objective_c
id obj = [[NSObject alloc] init];
// 前4个存在 `inline_referrers` 中, 下面的 `struct` 生效
__weak id weakObj1 = obj;
__weak id weakObj2 = obj;
__weak id weakObj3 = obj;
__weak id weakObj4 = obj;
// 第5个超限, 进行扩容并重新将这5个存在 `referrers` 中, 上面的 `struct` 生效
__weak id weakObj5 = obj;
```

所以, 如果一个对象被弱引用的次数较少(<=4), 那么直接存在数组里面, 数据在栈中操作相对快些.
如果被弱引用的次数较多, 那么会在堆上重新开辟内存进行扩容存储, 而且需要手动管理内存, 操作相对慢些, 处理逻辑上也要复杂很多.
所以使用 `weak` 的代价还是有的, 大部分对象被弱引用的次数还是不超过阈值的, 能够平衡内存和性能.

## Design HashMap

### 热身

先来热个身, 我们来看一道 LeetCode 的原题 [Design HashMap](https://leetcode.com/problems/design-hashmap/), 这题难度为 Easy. 如果不考虑内存上的优化的话, 直接使用 1000000 大小数组进行实现即可, 而这个 HashMap 的 key 就是值本身, value 也是值本身.
但是如果考虑上内存优化, 那么一个快速的 hash 函数和 hash 冲突的处理都是必不可少的. 关于这题的解我用 golang 实现了一遍. [golang 版本解](https://github.com/cxjwin/go-algocasts/blob/master/algo/design_hashmap.go), 目前该解已经通过 LeetCode 的单元测试.

### weak_table_t 的处理

`weak_table_t` 本质上也是一个 Hash Map, 上题中的解也是参考了 `weak_table_t` 的部分实现, `weak_table_t` 实现上还是有很多值得思考和学习的地方.

#### fast hash 函数

hash 函数的重要性不必多说了, 因为我们最终需要把 key 映射成下标然后存到数组里面去, 那么一个快速的 hash 函数能够保证频繁操作时的效率, 同时这个 hash 函数计算出来的值又要足够的均匀和随机, 这样能够减少散列冲突的概率, 保证存储的高效. 

那么 Runtime 源码中是怎么实现的呢, 我们看一下代码:

```c++

// ...
size_t begin = w_hash_pointer(new_referrer) & (entry->mask);
// ...

/** 
 * Unique hash function for weak object pointers only.
 * 
 * @param key The weak object pointer. 
 * 
 * @return Size unrestricted hash of pointer.
 */
static inline uintptr_t w_hash_pointer(objc_object **key) {
    return ptr_hash((uintptr_t)key);
}

// Pointer hash function.
// This is not a terrific hash, but it is fast 
// and not outrageously flawed for our purposes.

// Based on principles from http://locklessinc.com/articles/fast_hash/
// and evaluation ideas from http://floodyberry.com/noncryptohashzoo/
#if __LP64__
static inline uint32_t ptr_hash(uint64_t key)
{
    key ^= key >> 4;
    key *= 0x8a970be7488fda55;
    key ^= __builtin_bswap64(key);
    return (uint32_t)key;
}
#else
static inline uint32_t ptr_hash(uint32_t key)
{
    key ^= key >> 4;
    key *= 0x5052acdb;
    key ^= __builtin_bswap32(key);
    return key;
}
#endif

```

其中 `ptr_hash` 就是 fast hash 函数了, 当然这段函数的实现逻辑确实没太看懂, 我们可以去 http://locklessinc.com/articles/fast_hash/ 这个网站上参考下, 具体就不深究了, 博客里面的内容还是蛮复杂的, 甚至用到了汇编.
正如注释中描述的那样, ptr_hash 函数不够完美(但符合目标)且足够快, 至于 Apple 做了多少尝试和测试我们就无从考证了. 不过下面注释的代码可以看出, 这个简简单单的函数还是经过深思熟虑的.

```c++
/*
  Higher-quality hash function. This is measurably slower in some workloads.
#if __LP64__
 uint32_t ptr_hash(uint64_t key)
{
    key -= __builtin_bswap64(key);
    key *= 0x8a970be7488fda55;
    key ^= __builtin_bswap64(key);
    key *= 0x8a970be7488fda55;
    key ^= __builtin_bswap64(key);
    return (uint32_t)key;
}
#else
static uint32_t ptr_hash(uint32_t key)
{
    key -= __builtin_bswap32(key);
    key *= 0x5052acdb;
    key ^= __builtin_bswap32(key);
    key *= 0x5052acdb;
    key ^= __builtin_bswap32(key);
    return key;
}
#endif
*/
```

源码中还有一处细节值得学习

```c++
size_t begin = w_hash_pointer(new_referrer) & (entry->mask);
```

注意此处用到了位与(&), 而不是通常使用的取余(%)操作, 为什么呢? 因为 weak table 总是以 2 的 N 次幂进行扩容的, 所以我们可以通过位运算进行取余操作, 但是需要注意的是这种方式只是适合于求一个数除以 2 的 N 次冥. 当然位运算肯定比常规取余要快, 这也算一个小技巧吧.

#### 散列冲突

上文提到, `ptr_hash` 函数并非完美的 hash 函数(目前为止并没有完美的 hash 函数), 那么散列冲突再所难免, 一般的处理散列冲突有两种主要的方式 -- 开放寻址法和链表法.
weak table 中使用了开放寻址法, 下面我们就通过源码一探究竟:

```c++
if (entry->num_refs >= TABLE_SIZE(entry) * 3/4) {
	return grow_refs_and_insert(entry, new_referrer);
}
size_t begin = w_hash_pointer(new_referrer) & (entry->mask);
size_t index = begin;
size_t hash_displacement = 0;
while (entry->referrers[index] != nil) {
	hash_displacement++;
	index = (index+1) & entry->mask;
	if (index == begin) bad_weak_table(entry);
}
if (hash_displacement > entry->max_hash_displacement) {
	entry->max_hash_displacement = hash_displacement;
}
weak_referrer_t &ref = entry->referrers[index];
ref = new_referrer;
entry->num_refs++;
```

这里通过 hash 函数和取余计算出散列值(数组下标), 如果该下标中已经有值, 那么递增1重新取余计算下标, 直到找到一个没有存值的位置. 当然这里还有一些其他边界条件的判断. 
这里使用的是线性探测法, 除了这个方法以外还有另外两种比较经典的方法 -- 二次探测和双重散列, 这里暂时就不介绍了, 有兴趣的同学可以左转 Google 下.
当然线性探测法有一个比较大的缺陷. 当散列表中插入的数据越来越多, 散列表冲突的可能性越大, 空闲位置越来越少, 那么线性探测的时间也会越来越长.
这里我们看到为了减少此类情况的发生, 在开始位置会检查下当前 weak table 的容量, 如果已经达到总容量的 3/4 就会进行扩容.

代码虽短, 设计还是很周到的, 需要细细阅读, 细细体会.

#### 删除操作

对于线性探测发来说, 删除操作也需要特殊注意下, 不能简单的就把该位置的元素置空. 因为查找操作时, 如果找到一个置空的位置, 我们就认为其是有效位置, 如果这个位置是我们后来删除并置空的, 那么原先查找算法就会失效, 本来存在的数据就会认为不存在, 那么这个问题该如何解呢? 我们还是直接来看源码:

```c++
/**
 * Remove entry from the zone's table of weak references.
 */
static void weak_entry_remove(weak_table_t *weak_table, weak_entry_t *entry)
{
    // remove entry
    if (entry->out_of_line()) free(entry->referrers);
    bzero(entry, sizeof(*entry));

    weak_table->num_entries--;

    weak_compact_maybe(weak_table);
}
```

这里奇怪的地方是, `free(entry->referrers);` 之后并没有将 `entry` 置空, 而且使用 `bzero` 对内存存储的数据进行了擦除. 
那么问题来了, 如果不置空的话, 那么随着不断插入和删除, 原来的被删除的元素还是占着这位置, 势必会造成较多的浪费. 所以 Apple 在最后调用 `weak_compact_maybe` 检查下, 冗余空间达到一定阈值就进行压缩. 

#### 读取操作

因为存在散列冲突, 所以读取操作多了一步校验工作:

```c++
/** 
 * Return the weak reference table entry for the given referent. 
 * If there is no entry for referent, return NULL. 
 * Performs a lookup.
 *
 * @param weak_table 
 * @param referent The object. Must not be nil.
 * 
 * @return The table of weak referrers to this object. 
 */
static weak_entry_t *
weak_entry_for_referent(weak_table_t *weak_table, objc_object *referent)
{
    assert(referent);

    weak_entry_t *weak_entries = weak_table->weak_entries;

    if (!weak_entries) return nil;

    size_t begin = hash_pointer(referent) & weak_table->mask;
    size_t index = begin;
    size_t hash_displacement = 0;
    // 当存在散列冲突时, hash 函数计算的下标所取出的值可能不是正确值. 
    // 这里需要进行遍历, 直到找到正确的值或者超限返回空值. 
    // 所以当散列冲突较多时, 数据存取性能都会大幅下降.
    while (weak_table->weak_entries[index].referent != referent) {
        index = (index+1) & weak_table->mask;
        if (index == begin) bad_weak_table(weak_table->weak_entries);
        hash_displacement++;
        if (hash_displacement > weak_table->max_hash_displacement) {
            return nil;
        }
    }

    return &weak_table->weak_entries[index];
}
```

#### 扩容和压缩处理

扩容和压缩的处理没有特别的地方, 主要是一些阈值的设定和判断.

```c++
// Grow the given zone's table of weak references if it is full.
static void weak_grow_maybe(weak_table_t *weak_table)
{
    size_t old_size = TABLE_SIZE(weak_table);

    // Grow if at least 3/4 full.
    if (weak_table->num_entries >= old_size * 3 / 4) {
        weak_resize(weak_table, old_size ? old_size*2 : 64);
    }
}

// Shrink the table if it is mostly empty.
static void weak_compact_maybe(weak_table_t *weak_table)
{
    size_t old_size = TABLE_SIZE(weak_table);

    // Shrink if larger than 1024 buckets and at most 1/16 full.
    if (old_size >= 1024  && old_size / 16 >= weak_table->num_entries) {
        weak_resize(weak_table, old_size / 8);
        // leaves new table no more than 1/2 full
    }
}
```

当存储容量已经大于等于当前总容量的 3/4 时就要进行扩容.
当总容量大于等于 1024 且存储容量不足 1/16 时, 就需要压缩.
通过这两个动态的内存空间处理, 保证 weak table 处于一个可控的内存占用状态.

## 周边

以上是对 weak table 大致分析, 其实它的"周边"也有不少有意思的地方, 下面我们继续扒一扒.

### SideTable

#### 线程安全

上面提到了, `objc-weak.h` 里面暴露出来的函数基本都是 `_no_lock` 结尾的, 也就是说 `__weak` 线程安全的问题交给了调用者去处理. 那么究竟是谁再保证线程安全呢? 答案是 `SideTable`, 我们简单看一下代码:

```c++
struct SideTable {
    spinlock_t slock;
    RefcountMap refcnts;
    weak_table_t weak_table;

    SideTable() {
        memset(&weak_table, 0, sizeof(weak_table));
    }

    ~SideTable() {
        _objc_fatal("Do not delete SideTable.");
    }

    void lock() { slock.lock(); }
    void unlock() { slock.unlock(); }
    void forceReset() { slock.forceReset(); }

    // Address-ordered lock discipline for a pair of side tables.

    template<HaveOld, HaveNew>
    static void lockTwo(SideTable *lock1, SideTable *lock2);
    template<HaveOld, HaveNew>
    static void unlockTwo(SideTable *lock1, SideTable *lock2);
};
```

`SideTable` 里面有 `spinlock_t` 类型的变量 `slock`, 还有 `weak_table_t` 类型的变量 `weak_table`, 同时还有一些锁方法, 这里暂时不深究. 加锁的处理一般有两种方式:

```c++
// 调用静态函数锁住两个 SideTable
SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);
//...
SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

// 单个 SideTable 调用
table.lock();
//...
table.unlock();
```

#### 全局变量设计

通过上文得知 weak table 是 "The global weak references table". 那么这个 "global" 是怎么实现的呢? 通过上面的代码来看, `weak_table` 是被 `SideTable` 持有的, 而 `SideTable` 是被全局的 `SideTables` 持有的. 那么为什么需要这么设计呢? 在我看来, 应该是为了存储和查找的效率考虑吧. 毕竟如果把所有的 weak 变量都存在一个 weak table 中, 那么这个 weak table 的负担会有些重.
我们先看看 `SideTables` 是怎么实现的:

```c++
// We cannot use a C++ static initializer to initialize SideTables because
// libc calls us before our C++ initializers run. We also don't want a global 
// pointer to this struct because of the extra indirection.
// Do it the hard way.
alignas(StripedMap<SideTable>) static uint8_t 
    SideTableBuf[sizeof(StripedMap<SideTable>)];

static void SideTableInit() {
    new (SideTableBuf) StripedMap<SideTable>();
}

static StripedMap<SideTable>& SideTables() {
    return *reinterpret_cast<StripedMap<SideTable>*>(SideTableBuf);
}
```

首先用到 `alignas` 这里有 [alignas](http://www.enseignement.polytechnique.fr/informatique/INF478/docs/Cpp/en/cpp/language/alignas.html) 的说明. 另外初始化函数也比较特殊, 这里注释上也给出了详细解释, 原来库的加载顺序还会影响代码的实现. 这里又接触到了一个新的类型 `StripedMap`. 这又是个什么数据类型呢? 你暂时可以理解为是个简版的 Hash Map.
`StripedMap` 的详细实现, 这里就不展开了, 大家可以移步到 `objc-private.h` 看一下. 这里说一个细节吧:

```c++
enum { CacheLineSize = 64 };

// StripedMap<T> is a map of void* -> T, sized appropriately 
// for cache-friendly lock striping. 
// For example, this may be used as StripedMap<spinlock_t>
// or as StripedMap<SomeStruct> where SomeStruct stores a spin lock.
template<typename T>
class StripedMap {
#if TARGET_OS_IPHONE && !TARGET_OS_SIMULATOR
    enum { StripeCount = 8 };
#else
    enum { StripeCount = 64 };
#endif

    struct PaddedT {
        T value alignas(CacheLineSize);
    };
    // 此处省略N行代码 ...
}
```

目前来看 iPhone 真机上的大小为 8 个 CacheLineSize, SideTable 就分散存储在不同的区域上. 那么怎么判断一个对象最终落到那个区域呢? 还是通过位运算和取余的方式:

```c++
static unsigned int indexForPointer(const void *p) {
	uintptr_t addr = reinterpret_cast<uintptr_t>(p);
	return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
}
```

我们看一下怎么调用的:

```c++
// 从 SideTables 中获取一个 SideTable
table = &SideTables()[obj];

template<typename T>
class StripedMap {
  // 此处省略N行代码 ...
  // 实际调用代码
 public:
    T& operator[] (const void *p) { 
        return array[indexForPointer(p)].value; 
    }
    const T& operator[] (const void *p) const { 
        return const_cast<StripedMap<T>>(this)[p]; 
    }
  // 此处省略N行代码 ...
};
```

这里我们对一个全局存储的数据结构有了一些认识, 后续也可以借鉴类似的实现, 比如线程安全的设计, 数据存储结构的设计等等.

## 总结

当然, 这里只是分析总结了 weak 实现相关的一些源码, 整个 `NSObject.mm` 文件里面的实现还是很复杂的, 包含很多的 C/C++ 的实现技巧(比如模板的使用场景还是比较多的, 后续可以仔细学习下), 也有很多精妙的设计. 后续应该花更多的时间去专研下, 而不仅仅是为了应付面试. 另外在学习源码的时候, 每读一次都会有不同的收获, 也可以抛开语言层面去想想通用的设计, 都是很有意思的.
