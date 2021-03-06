---
title: 算法之时间复杂度
date: 2019-11-06 10:25:36
categories: 数据结构与算法
tags: [算法,时间复杂度]
---

## 在了解时间复杂度之前，让我们来了解一下什么是算法？
算法（Algorithm）是指解题方案的准确而完整的描述，是一系列解决问题的清晰指令（我的理解是一系列解决问题的步骤），算法代表着用系统的方法描述解决问题的策略机制。也就是说，能够对一定规范的输入，在有限时间内获得所要求的输出。不同的算法可能用不同的时间、空间或效率来完成同样的任务。

一个算法的优劣可以用空间复杂度与时间复杂度来衡量。

<!--more-->

### 算法的五大特性：
输入：算法具有0个或多个输入
输出: 算法至少有1个或多个输出
有穷性: 算法在有限的步骤之后会自动结束而不会无限循环，并且每一个步骤可以在可接受的时间内完成
确定性：算法中的每一步都有确定的含义，不会出现二义性
可行性：算法的每一步都是可行的，也就是说每一步都能够执行有限的次数完成
 

不同的算法对执行程序的结果也许是一样的，但是执行时间和效率却有着很大的区别，下面让我们来看个例子：

问题：a + b + c = 100, a^2  + b^2 = c^2,请计算出所有符合的a, b, c结果。
```
import time
 
start_time = time.time()
 
# 注意是三重循环
for a in range(0, 1001):
    for b in range(0, 1001):
        for c in range(0, 1001):
            if a**2 + b**2 == c**2 and a+b+c == 1000:
                print("a, b, c: %d, %d, %d" % (a, b, c))
 
end_time = time.time()
print("time: %f" % (end_time - start_time))
print("finished!")
```

结果：
```
a, b, c: 0, 500, 500
a, b, c: 200, 375, 425
a, b, c: 375, 200, 425
a, b, c: 500, 0, 500
time: 214.583347
finished!
```

程序执行完总共花费了214秒。我们再来看一个例子：
```
import time
 
start_time = time.time()
 
# 注意是两重循环
for a in range(0, 1001):
    for b in range(0, 1001-a):
        c = 1000 - a - b
        if a**2 + b**2 == c**2:
            print("a, b, c: %d, %d, %d" % (a, b, c))
 
end_time = time.time()
print("time: %f" % (end_time - start_time))
print("finished!")
```

结果：
```
a, b, c: 0, 500, 500
a, b, c: 200, 375, 425
a, b, c: 375, 200, 425
a, b, c: 500, 0, 500
time: 0.182897
finished!
```

程序执行完总共花费了0.18秒，这对于第一种方法来说，无疑是一种很大的提升。所以说，算法的好坏对于程序来说是非常重要的。

**因此，我们可以得出一个结论：实现算法程序的执行时间可以反应出算法的效率，即算法的优劣。**


## 单靠时间值绝对可信吗？
假设我们将第二次尝试的算法程序运行在一台配置古老性能低下的计算机中，情况会如何？很可能运行的时间并不会比在我们的电脑中运行算法一的214.583347秒快多少。

单纯依靠运行的时间来比较算法的优劣并不一定是客观准确的！

程序的运行离不开计算机环境（包括硬件和操作系统），这些客观原因会影响程序运行的速度并反应在程序的执行时间上。那么如何才能客观的评判一个算法的优劣呢？

 

## 时间复杂度与“大O记法”
我们假定计算机执行算法每一个基本操作的时间是固定的一个时间单位，那么有多少个基本操作就代表会花费多少时间单位。算然对于不同的机器环境而言，确切的单位时间是不同的，但是对于算法进行多少个基本操作（即花费多少时间单位）在规模数量级上却是相同的，由此可以忽略机器环境的影响而客观的反应算法的时间效率。

对于算法的时间效率，我们可以用“大O记法”来表示。

“大O记法”：对于单调的整数函数f，如果存在一个整数函数g和实常数c>0，使得对于充分大的n总有f(n)<=c*g(n)，就说函数g是f的一个渐近函数（忽略常数），记为f(n)=O(g(n))。也就是说，在趋向无穷的极限意义下，函数f的增长速度受到函数g的约束，亦即函数f与函数g的特征相似。


**时间复杂度：假设存在函数g，使得算法A处理规模为n的问题示例所用时间为T(n)=O(g(n))，则称O(g(n))为算法A的渐近时间复杂度，简称时间复杂度，记为T(n)**


## 最坏时间复杂度
分析算法时，存在几种可能的考虑：

算法完成工作最少需要多少基本操作，即最优时间复杂度
算法完成工作最多需要多少基本操作，即最坏时间复杂度
算法完成工作平均需要多少基本操作，即平均时间复杂度
对于最优时间复杂度，其价值不大，因为它没有提供什么有用信息，其反映的只是最乐观最理想的情况，没有参考价值。

对于最坏时间复杂度，提供了一种保证，表明算法在此种程度的基本操作中一定能完成工作。

对于平均时间复杂度，是对算法的一个全面评价，因此它完整全面的反映了这个算法的性质。但另一方面，这种衡量并没有保证，不是每个计算都能在这个基本操作内完成。而且，对于平均情况的计算，也会因为应用算法的实例分布可能并不均匀而难以计算。

因此，我们主要关注算法的最坏情况，亦即最坏时间复杂度。

 
## 算法分析
1）上面第一个列子核心部分：
```
for a in range(0, 1001):
    for b in range(0, 1001):
        for c in range(0, 1001):
            if a**2 + b**2 == c**2 and a+b+c == 1000:
                print("a, b, c: %d, %d, %d" % (a, b, c))
```
时间复杂度：
`T(n) = O(n*n*n) = O(n3)`

 

2）第二个例子核心部分：
```
for a in range(0, 1001):
    for b in range(0, 1001-a):
        c = 1000 - a - b
        if a**2 + b**2 == c**2:
            print("a, b, c: %d, %d, %d" % (a, b, c))
```
时间复杂度：
`T(n) = O(n*n*(1+1)) = O(n*n) = O(n2)`

由此可见，我们尝试的第二种算法要比第一种算法的时间复杂度好多的。
 

### 常见时间复杂度
执行次数函数举例 | 阶 | 非正式术语
:-: 			| :-: 		| :-:
12 				| O(1) 		| 常数阶
2n+3 			| O(n)		| 线性阶
3n^2+2n+1 		| O(n^2)	| 平方阶
5log2n+20 		| O(logn)	| 对数阶
2n+3nlog2n+19 	| O(nlogn)	| nlogn阶
6n^3+2n^2+3n+4 	| O(n^3)	| 立方阶
2^n	O(2n) 		| O(2^n)	| 指数阶



### 常见时间复杂度的关系
<center>![avatar](/static/blogImg/algorithm_1.png)</center>

所消耗的时间从小到大
O(1) < O(logn) < O(n) < O(nlogn) < O(n2) < O(n3) < O(2n) < O(n!) < O(nn)