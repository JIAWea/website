---
title: 数据结构之链表
date: 2019-12-10 12:01:36
categories: 数据结构与算法
tags: [数据结构,链表]
---

## 单向链表
单向链表也叫单链表，是链表中最简单的一种形式，它的每个节点包含两个域，一个信息域（元素域）和一个链接域。这个链接指向链表中的下一个节点，而最后一个节点的链接域则指向一个空值。

<center>![](https://img-blog.csdnimg.cn/20191111093914440.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0pJQVdFQQ==,size_16,color_FFFFFF,t_70)</center>

<!--more-->

- 表元素域elem用来存放具体的数据。
- 链接域next用来存放下一个节点的位置（python中的标识）
- 变量p指向链表的头节点（首节点）的位置，从p出发能找到表中的任意节点。

**下面引用算法图解一书中的链表：**
链表中的元素可存储在内存的任何地方。
<center>![](https://img-blog.csdnimg.cn/20191111094236680.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0pJQVdFQQ==,size_16,color_FFFFFF,t_70)</center>

链表的每个元素都存储了下一个元素的地址，从而使一系列随机的内存地址串在一起。

<center>![](https://img-blog.csdnimg.cn/20191111094413665.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0pJQVdFQQ==,size_16,color_FFFFFF,t_70)</center>

这犹如寻宝游戏。你前往第一个地址，那里有一张纸条写着“下一个元素的地址为123”。因此，你前往地址123，那里又有一张纸条写着“下一个元素的地址为668”。在链表中添加元素很容易：只需将其放入内存，并将其地址存储到前一个元素中。

## 链表的查找

在查找元素时，链表不想数组那样可以通过下标快速定位，只能从头节点向后一个一个节点逐一查找。


## 链表的更新
如果不考虑查找节点的话，链表的更新过程会想数组那样简单，直接把旧数据替换成新数据

<center>![链表更新](/static/data_struct/lianbiao_update.png)</center>

## 链表的插入
与数组类似，分为三种：
- 头部插入
- 中间插入
- 尾部插入

### 1).头部插入
1. 把新节点的`next`指向原先的头节点。
2. 把新节点变为链表的头节点

<center>![](/static/data_struct/link_list_add.png)</center>

### 2).中间插入
1. 把新节点的`next`指向插入位置的节点。
2. 把插入位置前置节点的`next`指向新节点

<center>![](/static/data_struct/link_list_add_2.png)</center>

### 3).尾部插入
尾部插入最简单，把最后一个节点的`next`指向新插入的节点

<center>![](/static/data_struct/link_list_add_3.png)</center>

## 链表的删除
也分为三种：头部删除、中间删除、尾部删除。跟上面插入相似，这里将不啰嗦了。

## 链表与顺序表的对比
链表失去了顺序表随机读取的优点，同时链表由于增加了结点的指针域，空间开销比较大，但对存储空间的使用要相对灵活。

链表与顺序表的各种操作复杂度如下所示：
<center>![](/static/data_struct/list_vs_link_list.png)</center>

链表和顺序表在插入和删除时进行的是完全不同的操作。
- 链表的主要耗时操作是遍历查找，删除和插入操作本身的复杂度是O(1)。
- 顺序表查找很快，主要耗时的操作是拷贝覆盖。因为除了目标元素在尾部的特殊情况，顺序表进行插入和删除时需要对操作点之后的所有元素进行前后移位操作，只能通过拷贝和覆盖的方法进行。

## 总结
1. 在读操作比较多、写操作比较少的情况下用数组比较合适。
2. 在读操作比较少，需要频繁地插入和删除操作比较多的情况下，使用链表更合适一些。
