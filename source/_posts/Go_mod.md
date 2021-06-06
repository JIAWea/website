---
title: 使用go mod，告别GOPATH
date: 2019-09-25 09:47:23
categories: 后端
tags: [Golang,mod]
---

我们都知道在使用Golang时开发程序时都需要在 `GOPATH` 下面，这就非常不方便。如果你想放在磁盘上的其他地方，那么go mod将是你的“好伙伴”。

<!--more-->

关于 go mod 的说明，可以参考：
* [Introduction to Go Modules](https://roberto.selbach.ca/intro-to-go-modules/)
* [Go 1.11 Modules 官方说明文档](https://github.com/golang/go/wiki/Modules)

### 命令行说明
```
➜  ~ go mod
Go mod provides access to operations on modules.
Note that support for modules is built into all the go commands,
not just 'go mod'. For example, day-to-day adding, removing, upgrading,
and downgrading of dependencies should be done using 'go get'.
See 'go help modules' for an overview of module functionality.
Usage:
	go mod <command> [arguments]
The commands are:
	download    download modules to local cache		
	edit        edit go.mod from tools or scripts		
	graph       print module requirement graph		
	init        initialize new module in current directory	
	tidy        add missing and remove unused modules 
	vendor      make vendored copy of dependencies
	verify      verify dependencies have expected content  
	why         explain why packages or modules are needed 
Use "go help mod <command>" for more information about a command.
```

* go mod download: 下载依赖的 module 到本地 cache
* go mod edit: 编辑 go.mod
* go mod graph: 打印模块依赖图
* go mod init: 在当前目录下初始化 go.mod(就是会新建一个 go.mod 文件)
* go mod tidy: 整理依赖关系，会添加丢失的 module，删除不需要的 module
* go mod vender: 将依赖复制到 vendor 下
* go mod verify: 校验依赖
* go mod why: 解释为什么需要依赖

执行命令`go mod verify`命令来检查当前模块的依赖是否全部下载下来，是否下载下来被修改过。如果所有的模块都没有被修改过，那么执行这条命令之后，会打印`all modules verified`。

### 如果在项目中使用
1. 版本：首先将你的Go版本更新到(>=1.11)，这里将不介绍怎么更新
2. 设置环境变量(1.12默认)：在你的项目目录下使用`set GO111MODULE=ON`
3. 执行`go mod init`在当前目录下生成一个`go.mod`文件，如果之前有生成过需要删除再初始化

执行完上面步骤基本就完成了，运行下程序你会发现目录下多了一个`go.sum`文件，是用来记录所依赖的版本的锁定

### 总结
使用`go mod`后你会发现在`GOPATH`下面的`pkg`目录会有一个`mod`目录，里面包含了项目需要的依赖包，这也是为什么不需要再`GOPATH`中开发程序也能使用的原因
