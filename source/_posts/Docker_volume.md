---
title: Docker之数据卷volume
date: 2019-12-10 11:48:02
categories: Docker
tags: [Docker,volume]
---

**数据卷**
默认情况下，在容器内创建的所有文件都存储在可写容器层上。这意味着：
- 当该容器不再存在时，数据将不会持久保存，并且如果另一个进程需要它，则可能很难从容器中取出数据。
- 容器的可写层与运行容器的主机紧密耦合。您不能轻易地将数据移动到其他地方。
- 写入容器的可写层需要 存储驱动程序来管理文件系统。存储驱动程序使用Linux内核提供联合文件系统。与使用直接写入主机文件系统的数据卷相比，这种额外的抽象降低了性能 。

<center>![avatar](/static/blogImg/DockerVolume.jpg)</center>

<!--more-->

Docker为容器提供了两个选项来将文件存储在主机中，以便即使容器停止后文件也可以持久存储：
- bind mount（绑定安装）
- volume（数据卷）

### volume：
是一个可供一个或多个容器使用的特殊目录，它绕过 UFS，可以提供很多有用的特性：
- 数据卷 可以在容器之间共享和重用
- 对 数据卷 的修改会立马生效
- 对 数据卷 的更新，不会影响镜像
- 数据卷 默认会一直存在，即使容器被删除

### bind mount：
- 文件或目录由主机上的完整路径引用。
- 该文件或目录不需要在Docker主机上已经存在。如果尚不存在，则按需创建。
- 性能非常好，但是它们依赖于具有特定目录结构的主机文件系统。

**注意：可以通过容器中运行的进程来更改主机文件系统 ，包括创建，修改或删除重要的系统文件或目录，存在安全隐患**

## 创建数据卷
```
$ docker volume create my-vol
```

查看所有的 数据卷
```
$ docker volume ls
local               my-vol
```
在主机里使用以下命令可以查看指定 数据卷 的信息
```
$ docker volume inspect my-vol

[
    {
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/my-vol/_data",
        "Name": "my-vol",
        "Options": {},
        "Scope": "local"
    }
]
```

### 查看数据卷的具体信息
在主机里使用以下命令可以查看 web 容器的信息
```
$ docker inspect web
数据卷 信息在 "Mounts" Key 下面

"Mounts": [
    {
        "Type": "volume",
        "Name": "my-vol",
        "Source": "/var/lib/docker/volumes/my-vol/_data",
        "Destination": "/app",
        "Driver": "local",
        "Mode": "",
        "RW": true,
        "Propagation": ""
    }
]
```

### 启动一个挂载数据卷的容器
在用 docker run 命令的时候，使用 `--mount` 标记或者直接使用`-v`来将 数据卷 挂载到容器里。在一次 docker run 中可以挂载多个 数据卷。

下面创建一个名为 web 的容器，并加载一个 数据卷 到容器的 /webapp 目录。
```
$ docker run -d -P \
    --name web \
    # -v my-vol:/wepapp \
    --mount source=my-vol,target=/webapp \
    training/webapp \
    python app.py
```

### 删除数据卷
```
$ docker volume rm my-vol
```
数据卷 是被设计用来持久化数据的，它的生命周期独立于容器，Docker 不会在容器被删除后自动删除 数据卷，并且也不存在垃圾回收这样的机制来处理没有任何容器引用的 数据卷。如果需要在删除容器的同时移除数据卷。可以在删除容器的时候使用`docker rm -v` 这个命令。

无主的数据卷可能会占据很多空间，要清理请使用以下命令
```
$ docker volume prune
```

在删除容器的时候也可使用`-v`来删除数据卷或者在启动容器是加上`--rm`，如：
- `docker rm -v 容器 `
- `docker run --rm`
