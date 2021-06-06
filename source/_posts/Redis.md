---
title: Redis类型及应用场景
date: 2020-05-24 12:26:00
categories: 数据库
tags: [redis,NoSQL]
---

NoSQL非关系型数据库。
Redis支持5种数据类型：string(字符串), hash(哈希), list(双向列表), set(无序集合), zset(有序集合)

### 1、string
**介绍**
string数据类型是最常用的、简单的key-value类型，普通的key/value存储都可以归为此类。value也可以是数字。因为是二进制安全的，所有可以将一个图片文件的内容作为string来存储。redis的string完全实现目前`memcached`的功能，并且效率更高。除了提供与memcached一样的`get`,`set`,`incr`,`decr`等操作外，还提供了下面的操作：

<!--more-->

1. 获取字符串长度
2. 往字符串append内容
3. 设置和获取字符串的某一段内容
4. 设置和获取字符串的某一位(bit)
5. 批量设置一系列字符串的内容

**应用场景**
1. 应用memcached和ckv的所有场景。字符串和数字之间存取。结构化数据需要先序列化，再set到value，取出时也需要反序列化。
2. 可以利用INCR, INCRBY, DECR, DECRBY等指令来实现原子计数的效果。即可用来实现业务上的统计计数需求。
3. 存放session key，实现一个分布式session系统。Redis的key可以方便地设置过期时间，用于实现session key的自动过期。验证skey时先根据uid路由到对应的redis，如取不到skey，则表示skey已过期，需要重新登录；如取到skey且校验通过则升级此skey的过期时间即可。
4. Set nx或SetNx，仅当key不存在时才Set。可以用来选举Master或实现分布式锁：所有Client不断尝试使用SetNx master myName抢注Master，成功的那位不断使用Expire刷新它的过期时间。如果Master挂掉了key就会失效，剩下的节点又会发生新一轮抢夺。
5. 借助redis2.6开始支持的lua脚本，可以实现更安全的2种分布式锁：一种适用于各进程竞争但总是单个进程获取锁并处理的场景。除非原处理进程挂掉因而锁过期才会被其它进程获取到锁。无须主动解锁。通过get、expire/pexpire、setnx ex| px的lua脚本实现；一种适用于各进程竞争获取锁并处理的场景。通过set nx ex| px获取锁，用完需要通过先get判断再del释放锁，否则在锁过期之前不能获取到锁。
6. GetSet， 设置新值，返回旧值。比如实现一个计数器，可以用GetSet获取计数并重置为0。
7. GetBit/SetBit/BitOp/BitCount， BitMap的玩法，比如统计今天的独立访问用户数时，每个注册用户都有一个offset，他今天进来的话就把他那个位设为1，用BitCount就可以得出今天的总人数。
8. Append/SetRange/GetRange/StrLen，对文本进行扩展、替换、截取和求长度，对特定数据格式非常有用。

**实现方式**
String在redis内部存储默认就是一个字符串，被redisObject所引用，当遇到incr,decr等操作时会转成数值型进行计算，此时`redisObject`的encoding字段为`int`。


### 2. Hash
**介绍**
Hash存的是字符串和字符串值之间的映射。Hash将对象的各个属性存入Map里，可以只读取/更新对象的某些属性。

常用命令: `hget`, `hset`, `hgetall`等。

**应用场景**
1. 存放结构化数据，比如用户信息。如果存放string的方式的话，修改用户信息中某一项时，需要取出后反序列化后修改完，再序列化回去。这样不仅增大了开销，也不适用于一些可能并发操作的场合。
2. 需要注意的是，Redis提供的`hgetall`可以直接取到全部的属性数据，但是数据过多的话，那么涉及到遍历整个内部的map操作，由于Redis是`单线程`模型，这个操作可能会比较耗时，而对其他客户端的请求完全不响应。

**实现方式**
Hash对应的value内部实际就是一个`HashMap`，这里会有2种不同实现，这个Hash的成员比较少时Redis为了节省内存会采用类似一堆数组的方式来紧凑存储，而不会采用真正的HashMap结构，对应的`value redisObject`的encoding为`zipmap`，当成员数量增大时会自动转成真正的HashMap，此时的encoding为`ht`。


### 3. List
**介绍**
List是一个双向链表，支持双向的pop、push，一般是FIFO(First In First Out)。`Blocking`版本的BLPop/BRPop，客户端可以阻塞在那里直到有消息到来。

按值的操作：LRem(按值删除元素)、Linsert(在某个元素插入)，复杂度是O(N)，因为List的值不唯一，所以要遍历全部元素，而Set只要O(log(N))。

按下标操作：LSet、LIndex、LRange，LTrim可限制长度，比如只保留最新N条消息。复杂度O(N),因为List是链表而不是数组，所以按下标访问其实要遍历链表，除非下标刚好是队头和队尾。


常用命令：`lpush`, `rpush`, `lpop`, `lrange`等。 

**应用场景**
1. 各自列表。比如twitter的关注列表、粉丝列表等，最新消息排行、每篇文章的评论等也可以用List结构实现。
2. 消息队列，将任务push到List中，然后工作线程再用pop操作把任务取走。这里的消息队列并没有`ACK`机制，如果消费者把任务pop走了没处理完就死机了怎么办？解决方法之一就是多加一个`sorted set`，分发的时候同时发到`List`与`sorted set`，以分发时间为`score`，消费者把任务做完之后通过`ZREM`删除`sorted set`里的job，并且定时取出超时未完成的任务，重新放回`List`。
3. 利用LRange可以方便的实现List内容分页的功能。

**实现方式**
List是一个双向链表，即可以支持方向查找和遍历等操作，不过带来了部分额外的内存开销，Redis内部的很多实现，包括发送缓冲队列等也是用这个数据结构。


### 4. Set
**介绍**
Set是一种无序的集合，集合中的元素没有先后顺序，不重复。将重复的元素放入Set会自动去重。

**应用场景**
1. 某些需要去重的列表，并且set提供了判断某个元素是否存在一个set集合内的接口
2. 存储一些集合性的数据，比如在微博中，可以将一个用户的所有关注人存在一个集合中，并将所有粉丝也存入一个集合中。Redis还提供了集合求交集、并集、差集操作。可以非常方便的实现如共同关注、共同喜好。

**实现方式**
set的内部实现是一个value用户为`null`的HashMap，实际就是通过计算hash的方式来快速排重，这也是set能提供判断一个成员是否在这个集合内的原因。


### 5. Sorted Set
**介绍**
有序集合，相比Set，元素放入集合时还要提供该元素的分数，可根据分数自动排序。

**应用场景**
1. 存放一个有序的并且不重复的集合列表，比如twitter的pubilc timeline可以以发表时间作为score来存储，这样获取时就是自动按时间排好序的。
2. 可以做带权重的队列，比如普通消息的`score`为1，重要消息的score为2，然后工作线程可以按score的倒序来获取工作任务，然重要的先执行。

**实现方式**
sorted set的内部使用HashMap和跳跃表(SkipList)来保证数据的存储和有序，HashMap里放的是成员到score的映射，而跳跃表里存放的是所有的成员，排序依据是HashMap里存的score，使用跳跃表的结构可以获取比较高的查找效率。
