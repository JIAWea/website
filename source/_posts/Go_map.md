---
title: Golang map实现原理
date: 2020-11-22 20:00:54
categories: 后端
tags: [源码,Golang,map]
---

### map数据结构
在介绍`map`之前，先来看看它的数据结构，`map`实际上就是一个散列表，指向一堆桶`buckets`。通过哈希函数计算出`key`的值，然后再存储到每个桶`bmap`中，每个`bmap`可以存储 8 组键值对。为了加速`hash`定位桶，`bmap`里记录了`tophash`数组，记录了`key`的哈希高 8 位，通过比较不同键的哈希的高 8 位可以减少访问键值对的次数。

内部结构图：

<center>![结构图](/static/go_map/data_struct.png)</center>
<!--more-->

源码：

```
type hmap struct {
    count     int   
    flags     uint8             // 状态标示：iterator  = 1，oldIterator = 2，hashWriting = 4，sameSizeGrow = 8；
    B         uint8             // 桶的个数 = 2^B
    noverflow uint16            // 溢出桶的个数，B<16时为精确值，否则为近似值；
    hash0     uint32 

    buckets    unsafe.Pointer   // 当前桶，长度为（0-2^B）
    oldbuckets unsafe.Pointer   // 扩容时保存的旧桶；
    nevacuate  uintptr          // 迁移数，下一个迁移的桶；

    extra *mapextra             // 溢出桶
}

// 额外记录overflow桶信息
type mapextra struct {
    overflow     *[]*bmap       // 没有指针时，溢出桶挂在这；
    oldoverflow  *[]*bmap
    nextOverflow *bmap          // map初始化时，预分配了一些溢出桶；
}

// 桶结构 （字段会根据key和elem类型动态生成，见下边bmap）
type bmap struct {
  // 记录桶内8个单元的高8位hash值，或标记空桶状态，用于快速定位key
  // emptyRest      = 0 // 此单元为空，且更高索引的单元也为空
  // emptyOne       = 1 // 此单元为空
  // evacuatedX     = 2 // 用于表示扩容迁移到新桶前半段区间
  // evacuatedY     = 3 // 用于表示扩容迁移到新桶后半段区间
  // evacuatedEmpty = 4 // 用于表示此单元已迁移
  // minTopHash     = 5 // 最小的空桶标记值，小于其则是空桶标志
  tophash [bucketCnt]uint8
}

// cmd/compile/internal/gc/reflect.go
type bmap struct {
    topbits  [8]uint8           // 每个key哈希值的高8位，加速访问；
    keys     [8]keytype         // 先key后value，节省内存；
    elems    [8]elemtype
    // overflow 溢出桶
    // otyp 类型为指针*Type,
    // 若keytype及elemtype不含指针，则为uintptr
    // 使bmap整体不含指针,避免gc去scan此类map
    overflow otyp
}
```

### hash方式
哈希表是计算机科学中最重要的数据结构之一，读写性能很高，时间复杂度是`O(1)`。
#### 哈希函数
哈希函数的选择在很大程度上能够决定哈希表的读写性能，尽可能的让哈希函数的结构均匀分布，会减少哈希冲突导致的读写性能变差的情况。

#### 哈希冲突
解决哈希冲突一般有两种：

**开放寻址法**

开放寻址法核心思想是对数组中的元素依次探测和比较以判断目标键值对是否存在于哈希表中，即遇到哈希冲突时，就会将键值对写入到下一个不为空的位置。查找对应key会继续查找后面的元素，直到内存为空或者找到目标元素。

**拉链法**

拉链法一般会使用数组加上链表，不过有一些语言会在拉链法的哈希中引入红黑树以优化性能，拉链法会使用链表数组作为哈希底层的数据结构。
当我们需要将一个键值对，如 (Key6, Value6) 写入哈希表时，键值对中的键 Key6 都会先经过一个哈希函数，哈希函数返回的哈希会帮助我们选择一个桶，和开放地址法一样，选择桶的方式就是直接对哈希返回的结果取模：
```
mod := hash("Key6") % array.len
```
选择了桶后就会遍历当前桶中的链表了，在遍历链表的过程中会遇到以下两种情况：
1. 找到键相同的键值对 —— 更新键对应的值；
2. 没有找到键相同的键值对 —— 在链表的末尾追加新键值对；

### 扩容
随着哈希表中元素的增加，会逐渐影响哈希的性能，所以我们需要更多的桶和更大的内容保证哈希的读写性能。

```
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	...

    // If we hit the max load factor or we have too many overflow buckets,
	// and we're not already in the middle of growing, start growing.
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again
	}
	...
}
```

扩容条件：
1. 装载因子`overLoadFactor`大于 6.5；
2. 哈希使用了太多溢出桶，次数会等量扩容，垃圾回收会清理老的溢出桶并释放内存，使 `bmap` 中的键值对更加紧凑，提高性能。

```
装载因子 := 元素数量 / 桶数量
```

一般当`overLoadFactor`大于 6.5 时，会扩容一倍。在扩容的过程中不会一下子把数据全部迁移至新的桶中，采用**渐近式扩容**，`buckets`指向新桶，`oldbuckets`指向要迁移的桶，`nevacuate`表示下一个要迁移的桶，每当对数据进行读写时，都会看是否需要进行部分迁移，以防全部迁移时产生的瞬间抖动。


### 总结
- Go 语言使用拉链法来解决哈希碰撞的问题实现了哈希表，它的访问、写入和删除等操作都在编译期间转换成了运行时的函数或者方法。
- 哈希在每一个桶中存储键对应哈希的前 8 位，当对哈希进行操作时，这些 tophash 就成为了一级缓存帮助哈希快速遍历桶中元素，每一个桶都只能存储 8 个键值对，一旦当前哈希的某个桶超出 8 个，新的键值对就会被存储到哈希的溢出桶中。
- 随着键值对数量的增加，溢出桶的数量和哈希的装载因子也会逐渐升高，超过一定范围就会触发扩容，扩容会将桶的数量翻倍，元素再分配的过程也是在调用写操作时增量进行的，不会造成性能的瞬时巨大抖动。

##### 参考
- [Golang源码](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L115:6)
- [Go语言设计与实现](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/)
- [Dig101:Go之读懂map的底层设计](http://blog.newbmiao.com/2020/02/04/dig101-golang-map.html)
