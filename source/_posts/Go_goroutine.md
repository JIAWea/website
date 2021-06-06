---
title: Golang并发之Goroutine
date: 2020-11-21 10:44:18
categories: 后端
tags: [Golang,Goroutine]
---

### 1.介绍
Golang 中的并发指的是一个函数拥有独立于其他函数运行的能力，当创建一个`goroutine`时，Go 会将其视为一个独立的工作单元，然后会被调度到可用的逻辑处理器上执行。 Go 语言运行时的调度器能够管理所有的`goroutine`并为其分配执行时间。调度器是在操作系统之上的，将操作系统的线程与程序运行时的逻辑处理器绑定，并在逻辑处理器上运行`goroutine`。调度器在任何给定的时间，都会控制哪个`goroutine`在哪个逻辑处理器上执行。

Golang 的并发同步模型来自一个叫做**通讯顺序程序（Communicating Sequential Processes, CSP）**的范型。`CSP`是一种消息传递模型。在`goroutine`之间同步和传递数据是通过通道`channel`实现的，而不是通过对数据加锁来实现同步访问。

<!--more-->

### 进程与线程
什么是进程(process)和线程(thread)？
当应用程序在运行时，操作系统就会为其启动一个进程。可以将这个进程看成作一个包含了应用程序在运行中需要用到和维护的各种资源的容器。

下图展示了一个包含所有可能分配的常用资源的进程。这些资源包括但不限于内存地址空间、文件和设备的句柄以及线程。一个线程是一个**执行空间**，这个空间会被操作系统调度至物理处理器上来运行函数中的代码。每个进程至少包含一个线程，每个进程的初始化线程被称为**主线程**，一个进程里的所有线程共享所有资源，当主线程终止时，应用程序也会停止。

<center>![进程与线程](/static/goroutine/process.png)</center>

### 并发
操作系统会在物理处理器上调度线程来运行，而 Go 语言的运行时会在逻辑处理器上调度`goroutine`来运行。每个逻辑处理器都分别绑定到单个操作系统线程。

在下图中可以看到操作系统线程、逻辑处理器和本地运行队列之间的关系。如果创建一个`goroutine`并准备运行，这个`goroutine`就会被放到调度器的全局运行队列中。之后调度器就会将这些队列中的`goroutine`分配给一个逻辑处理器（单进程），并放到逻辑处理器对应的本地运行队列中。本地运行队列中的`goroutine`会一直等待直到自己被分配的逻辑处理器执行。

<center>![GPM](/static/goroutine/gpm-1.png)</center>

有时正在运行的`goroutine`需要执行一个阻塞的系统调用时，如打开一个文件。当这类调用发生时，线程和`goroutine`会从逻辑处理器上分离，该线程会继续阻塞，等待系统调用的返回。此时逻辑处理器就失去了用来运行的线程，所以调度器会创建一个新的线程并绑定到该逻辑处理器上。然后调度器会从本地运行队列里选择另外一个`goroutine`来运行，一旦被阻塞的系统调用执行完成并返回，其`goroutine`会被放回本地运行队列，而之前的线程会被保留，以便之后使用。

如果一个`goroutine`需要做一个网络`I/O`调用，流程上会有些不一样。`goroutine`会和逻辑处理器分离，并移动到集成了**网络轮训器**，一旦该轮训器指示某个网络读写操作就绪，对应的`goroutine`就会重新分配到逻辑处理器上完成操作。

### 并行
并发（concurrency）不是并行（parallelism）。并行是让多个代码片段同时在不同的物理处理器上执行。并行的关键是同时做很多事情，而并发是同时管理很多事情，这些事可能只做了一半就暂停去做别的事情了。

如果希望`goroutine`并行，必须使用多于一个逻辑处理器。当有多个逻辑处理器时，调度器会将`goroutine`平等分配到每个逻辑处理器上。这会让`goroutine`在不同的线程上运行。当然运行的机器要有多个物理处理器，否则达不到并行的效果。

<center>![GPM](/static/goroutine/gpm-2.png)</center>

### 示例
1. 分配一个逻辑处理器，启动两个`goroutine`，分别打印出5000以内的素数。

```
// 这个示例程序展示goroutine调度器是如何在单个线程上
// 切分时间片的
package main

import (
	"fmt"
	"runtime"
	"sync"
)

//   wg用来等待程序完成
var wg sync.WaitGroup

func main() {
	// 分配一个逻辑处理器给调度器使用
	runtime.GOMAXPROCS(1)
	wg.Add(2)
	fmt.Println("Create Goroutines")

	go printPrime("A")
	go printPrime("B")

	fmt.Println("Waiting To   Finish")
	wg.Wait()
	fmt.Println("Terminating Program")
}

// printPrime 显示5000以内的素数值(大于1，除了1和本身外不能被整除)
func printPrime(prefix string) {
	defer wg.Done()
next:
	for outer := 2; outer < 5000; outer++ {
		for inner := 2; inner < outer; inner++ {
			if outer%inner == 0 {
				continue next
			}
		}
		fmt.Printf("%s:%d\n", prefix, outer)
	}

	fmt.Println("Completed", prefix)
}

```

执行完上面的代码后，你会发现在并发情况下，有可能 A 先执行完，也有可能 B 先执行完，因为当 goroutine 被调度器分配给逻辑处理器执行是有执行时间的，当 A 或 B 没执行完时，有可能被切换到另一个 goroutine 执行了，所以你看到的结果会出现：A 和 B 都是各自打印了一段最终才是结束的。

如果把`runtime.GOMAXPROCS(1)`改成 `2` 呢？
这时候 A 和 B是并行执行的，而不是并发了。


##### 参考书籍：
- Go语言实战