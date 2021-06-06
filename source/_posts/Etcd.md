---
title: 使用 etcd 作为服务配置中心
date: 2020-12-12 09:30:18
categories: 微服务
tags: etcd
---

### etcd 简单介绍
etcd 是一个高可用的分布式 key-value 数据存储系统，内部采用 `Raft` 协议作为一致性算法，基于 Go 语言实现。

主要特点：
- 简单：提供明确的定义、面向用户的 API（gRPC）
- 安全：支持 TLS客户端证书认证
- 快速：基准测试每秒 10000 写入
- 可靠：使用 Raft 算法保证一致性

<!--more-->

### etcd 应用场景

#### 服务发现
服务发现要解决的也是分布式系统中最常见的问题之一，即在同一个分布式集群中的进程或服务，要如何才能找到对方并建立连接。本质上来说，服务发现就是想要了解集群中是否有进程在监听 `udp` 或 `tcp` 端口，并且通过名字就可以查找和连接。

<center>![服务发现](/static/etcd/01.png)</center>

#### 配置中心
将配置进行集中管理。一般应用在启动时会主动从 etcd 中获取配置信息，同时在节点上注册一个 `Watcher` 监控，每当配置有更新时，etcd 都会实时通知订阅者，以此获取最新的配置。

#### 分布式锁
使用 `Raft` 算法保持数据的一致性，每当数据储存到集群中的值必然是全局一致的，所以很容易实现分布式锁。所有获取锁的用户最终只有一个可以得到，为此提供了分布式锁原子操作 CAS（CompareAndSwap）的 API。通过设置prevExist值，可以保证在多个节点同时去创建某个目录时，只有一个成功。而创建成功的用户就可以认为是获得了锁。

### Raft 简单介绍
`Raft` 是一种协议，集群节点可以使用该协议维护一个复制的状态机，状态机与复制的日志保存同步。具体详情可看[Raft.pdf](https://Raft.github.io/Raft.pdf)。`Raft` 被广泛地使用在许多产品中，其中包括 etcd, Kubernetes, Docker Swarm, Cloud Foundry Diego, CockroachDB, TiDB, Project Calico, Flannel, Hyperledger 等。

相关特性：
- 领导人选举
- 日志复制
- 日志压缩
- 会员变更
- 领导转移扩展
- ····

### etcd 与其他常见服务发现框架对比

| 名称  |   优点 | 缺点   | 接口 | 一致性算法 |
| :---: | ---  |---  |:---: | :---: |
|zookeeper|1.功能强大，不仅仅只是服务发现<br>2.提供 watcher 机制能实时获取服务提供者的状态<br/>3.dubbo 等框架支持| 1.没有健康检查<br>2.需在服务中集成 sdk，复杂度高<br/>3.不支持多数据中心| sdk | Paxos |
|consul|1.简单易用，不需要集成 sdk<br>2.自带健康检查<br/>3.支持多数据中心<br>4.提供 web 管理界面<br/>| 不能实时获取服务信息的变化通知| http/dns	 | Raft |
|etcd|1.简单易用，不需要集成 sdk<br>可配置性强<br/>|1.没有健康检查<br>2.需配合第三方工具一起完成服务发现<br/>3.不支持多数据中心| http | Raft |


### etcd 单机部署

部署环境：Ubuntu-16.04

#### etcd Server 端

下载：
```
ETCD_VER=v3.4.14

# choose either URL
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}

rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

/tmp/etcd-download-test/etcd --version
/tmp/etcd-download-test/etcdctl version
```

启动：
```
# start a local etcd server
/tmp/etcd-download-test/etcd
```

启动信息：
```
[WARNING] Deprecated '--logger=capnslog' flag is set; use '--logger=zap' flag instead
2020-12-10 17:07:49.921382 I | etcdmain: etcd Version: 3.4.14
2020-12-10 17:07:49.924906 I | etcdmain: Git SHA: 8a03d2e96
2020-12-10 17:07:49.925297 I | etcdmain: Go Version: go1.12.17
2020-12-10 17:07:49.927355 I | etcdmain: Go OS/Arch: linux/amd64
2020-12-10 17:07:49.928312 I | etcdmain: setting maximum number of CPUs to 2, total number of available CPUs is 2
2020-12-10 17:07:49.928478 W | etcdmain: no data-dir provided, using default data-dir ./default.etcd
[WARNING] Deprecated '--logger=capnslog' flag is set; use '--logger=zap' flag instead
2020-12-10 17:07:49.930599 I | embed: name = default
2020-12-10 17:07:49.930678 I | embed: data dir = default.etcd
2020-12-10 17:07:49.930739 I | embed: member dir = default.etcd/member
2020-12-10 17:07:49.930810 I | embed: heartbeat = 100ms
2020-12-10 17:07:49.930885 I | embed: election = 1000ms
2020-12-10 17:07:49.930953 I | embed: snapshot count = 100000
2020-12-10 17:07:49.931005 I | embed: advertise client URLs = http://localhost:2379
2020-12-10 17:07:49.974825 I | etcdserver: starting member 8e9e05c52164694d in cluster cdf818194e3a8c32
Raft2020/12/10 17:07:49 INFO: 8e9e05c52164694d switched to configuration voters=()
Raft2020/12/10 17:07:49 INFO: 8e9e05c52164694d became follower at term 0
Raft2020/12/10 17:07:49 INFO: newRaft 8e9e05c52164694d [peers: [], term: 0, commit: 0, applied: 0, lastindex: 0, lastterm: 0]
Raft2020/12/10 17:07:49 INFO: 8e9e05c52164694d became follower at term 1
Raft2020/12/10 17:07:49 INFO: 8e9e05c52164694d switched to configuration voters=(10276657743932975437)
2020-12-10 17:07:49.997394 W | auth: simple token is not cryptographically signed
2020-12-10 17:07:50.011174 I | etcdserver: starting server... [version: 3.4.14, cluster version: to_be_decided]
2020-12-10 17:07:50.014616 I | etcdserver: 8e9e05c52164694d as single-node; fast-forwarding 9 ticks (election ticks 10)
Raft2020/12/10 17:07:50 INFO: 8e9e05c52164694d switched to configuration voters=(10276657743932975437)
2020-12-10 17:07:50.018429 I | etcdserver/membership: added member 8e9e05c52164694d [http://localhost:2380] to cluster cdf818194e3a8c32
2020-12-10 17:07:50.020840 I | embed: listening for peers on 127.0.0.1:2380
Raft2020/12/10 17:07:50 INFO: 8e9e05c52164694d is starting a new election at term 1
Raft2020/12/10 17:07:50 INFO: 8e9e05c52164694d became candidate at term 2
Raft2020/12/10 17:07:50 INFO: 8e9e05c52164694d received MsgVoteResp from 8e9e05c52164694d at term 2
Raft2020/12/10 17:07:50 INFO: 8e9e05c52164694d became leader at term 2
Raft2020/12/10 17:07:50 INFO: Raft.node: 8e9e05c52164694d elected leader 8e9e05c52164694d at term 2
2020-12-10 17:07:50.788838 I | etcdserver: setting up the initial cluster version to 3.4
2020-12-10 17:07:50.794910 N | etcdserver/membership: set the initial cluster version to 3.4
2020-12-10 17:07:50.795922 I | etcdserver/api: enabled capabilities for version 3.4
2020-12-10 17:07:50.796055 I | etcdserver: published {Name:default ClientURLs:[http://localhost:2379]} to cluster cdf818194e3a8c32
2020-12-10 17:07:50.797360 I | embed: ready to serve client requests
2020-12-10 17:07:50.803625 N | embed: serving insecure client requests on 127.0.0.1:2379, this is strongly discouraged!
```

相关信息：
- name 表示节点名称，默认为 default。
- data-dir 保存日志和快照的目录，默认为当前工作目录default.etcd/目录下。
- 在 http://localhost:2380 和集群中其他节点通信。
- 在 http://localhost:2379 提供 HTTP API 服务，供客户端交互。
- heartbeat 为 100ms，该参数的作用是 leader 多久发送一次心跳到
- followers，默认值是100ms。
- election 为 1000ms，该参数的作用是重新投票的超时时间，如果 follow 在该 + 时间间隔没有收到心跳包，会触发重新投票，默认为 1000ms。
- snapshot count 为 10000，该参数的作用是指定有多少事务被提交时，触发 + 截取快照保存到磁盘。
- 集群和每个节点都会生成一个 uuid。
- 启动的时候会运行 Raft，选举出 leader。


#### etcd Client 端

##### 命令行客户端
etcd 提供一个命令行客户端，这里简单看一下

```
# write,read to etcd
/tmp/etcd-download-test/etcdctl --endpoints=localhost:2379 put foo bar
/tmp/etcd-download-test/etcdctl --endpoints=localhost:2379 get foo
```

```
ray@ray-virtual-machine:~$ /tmp/etcd-download-test/etcdctl -h
NAME:
        etcdctl - A simple command line client for etcd3.

USAGE:
        etcdctl [flags]

VERSION:
        3.4.14

API VERSION:
        3.4

COMMANDS:
        alarm disarm            Disarms all alarms
        alarm list              Lists all alarms
        auth disable            Disables authentication
        auth enable             Enables authentication
        check datascale         Check the memory usage of holding data for different workloads on a given server endpoint.
        check perf              Check the performance of the etcd cluster
        compaction              Compacts the event history in etcd
        defrag                  Defragments the storage of the etcd members with given endpoints
        del                     Removes the specified key or range of keys [key, range_end)
        elect                   Observes and participates in leader election
        endpoint hashkv         Prints the KV history hash for each endpoint in --endpoints
        endpoint health         Checks the healthiness of endpoints specified in `--endpoints` flag
        endpoint status         Prints out the status of endpoints specified in `--endpoints` flag
        get                     Gets the key or a range of keys
        help                    Help about any command
        lease grant             Creates leases
        lease keep-alive        Keeps leases alive (renew)
        lease list              List all active leases
        lease revoke            Revokes leases
        lease timetolive        Get lease information
        lock                    Acquires a named lock
        make-mirror             Makes a mirror at the destination etcd cluster
        member add              Adds a member into the cluster
        member list             Lists all members in the cluster
        member promote          Promotes a non-voting member in the cluster
        member remove           Removes a member from the cluster
        member update           Updates a member in the cluster
        migrate                 Migrates keys in a v2 store to a mvcc store
        move-leader             Transfers leadership to another etcd cluster member.
        put                     Puts the given key into the store
        role add                Adds a new role
        role delete             Deletes a role
        role get                Gets detailed information of a role
        role grant-permission   Grants a key to a role
        role list               Lists all roles
        role revoke-permission  Revokes a key from a role
        snapshot restore        Restores an etcd member snapshot to an etcd directory
        snapshot save           Stores an etcd node backend snapshot to a given file
        snapshot status         Gets backend snapshot status of a given file
        txn                     Txn processes all the requests in one transaction
        user add                Adds a new user
        user delete             Deletes a user
        user get                Gets detailed information of a user
        user grant-role         Grants a role to a user
        user list               Lists all users
        user passwd             Changes password of user
        user revoke-role        Revokes a role from a user
        version                 Prints the version of etcdctl
        watch                   Watches events stream on keys or prefixes

OPTIONS:
      --cacert=""                               verify certificates of TLS-enabled secure servers using this CA bundle
      --cert=""                                 identify secure client using this TLS certificate file
      --command-timeout=5s                      timeout for short running command (excluding dial timeout)
      --debug[=false]                           enable client-side debug logging
      --dial-timeout=2s                         dial timeout for client connections
  -d, --discovery-srv=""                        domain name to query for SRV records describing cluster endpoints
      --discovery-srv-name=""                   service name to query when using DNS discovery
      --endpoints=[127.0.0.1:2379]              gRPC endpoints
  -h, --help[=false]                            help for etcdctl
      --hex[=false]                             print byte strings as hex encoded strings
      --insecure-discovery[=true]               accept insecure SRV records describing cluster endpoints
      --insecure-skip-tls-verify[=false]        skip server certificate verification (CAUTION: this option should be enabled only for testing purposes)
      --insecure-transport[=true]               disable transport security for client connections
      --keepalive-time=2s                       keepalive time for client connections
      --keepalive-timeout=6s                    keepalive timeout for client connections
      --key=""                                  identify secure client using this TLS key file
      --password=""                             password for authentication (if this option is used, --user option shouldn't include password)
      --user=""                                 username[:password] for authentication (prompt if password is not supplied)
  -w, --write-out="simple"                      set the output format (fields, json, protobuf, simple, table)
```

##### 客户端包

> 安装

```
go get go.etcd.io/etcd/clientv3
```

> put、get

```
package main

import (
	"context"
	"fmt"
	"time"

	"go.etcd.io/etcd/clientv3"
)

// etcd client put/get demo
// use etcd/clientv3

func main() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		// handle error!
		fmt.Printf("connect to etcd failed, err:%v\n", err)
		return
	}
    fmt.Println("connect to etcd success")
	defer cli.Close()
	// put
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	_, err = cli.Put(ctx, "q1mi", "dsb")
	cancel()
	if err != nil {
		fmt.Printf("put to etcd failed, err:%v\n", err)
		return
	}
	// get
	ctx, cancel = context.WithTimeout(context.Background(), time.Second)
	resp, err := cli.Get(ctx, "q1mi")
	cancel()
	if err != nil {
		fmt.Printf("get from etcd failed, err:%v\n", err)
		return
	}
	for _, ev := range resp.Kvs {
		fmt.Printf("%s:%s\n", ev.Key, ev.Value)
	}
}
```

> watch

```
package main

import (
	"context"
	"fmt"
	"time"

	"go.etcd.io/etcd/clientv3"
)

// watch demo

func main() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		fmt.Printf("connect to etcd failed, err:%v\n", err)
		return
	}
	fmt.Println("connect to etcd success")
	defer cli.Close()
	// watch key:q1mi change
	rch := cli.Watch(context.Background(), "q1mi") // <-chan WatchResponse
	for wresp := range rch {
		for _, ev := range wresp.Events {
			fmt.Printf("Type: %s Key:%s Value:%s\n", ev.Type, ev.Kv.Key, ev.Kv.Value)
		}
	}
}
```

> lease 租约

```
package main

// etcd lease

import (
	"fmt"
	"time"
	"context"
	"log"

	"go.etcd.io/etcd/clientv3"
)

func main() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: time.Second * 5,
	})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("connect to etcd success.")
	defer cli.Close()

	// 创建一个5秒的租约
	resp, err := cli.Grant(context.TODO(), 5)
	if err != nil {
		log.Fatal(err)
	}

	// 5秒钟之后, /nazha/ 这个key就会被移除
	_, err = cli.Put(context.TODO(), "/nazha/", "dsb", clientv3.WithLease(resp.ID))
	if err != nil {
		log.Fatal(err)
	}
}
```

> keepAlive

```
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"go.etcd.io/etcd/clientv3"
)

// etcd keepAlive

func main() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: time.Second * 5,
	})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("connect to etcd success.")
	defer cli.Close()

	resp, err := cli.Grant(context.TODO(), 5)
	if err != nil {
		log.Fatal(err)
	}

	_, err = cli.Put(context.TODO(), "/nazha/", "dsb", clientv3.WithLease(resp.ID))
	if err != nil {
		log.Fatal(err)
	}

	// the key 'foo' will be kept forever
	ch, kaerr := cli.KeepAlive(context.TODO(), resp.ID)
	if kaerr != nil {
		log.Fatal(kaerr)
	}
	for {
		ka := <-ch
		fmt.Println("ttl:", ka.TTL)
	}
}
```

> 分布式锁

```
package main

import (
	"fmt"
	"log"
	"go.etcd.io/etcd/clientv3/concurrency"
)

func main() {
	cli, err := clientv3.New(clientv3.Config{Endpoints: endpoints})
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	// 创建两个单独的会话用来演示锁竞争
	s1, err := concurrency.NewSession(cli)
	if err != nil {
		log.Fatal(err)
	}
	defer s1.Close()
	m1 := concurrency.NewMutex(s1, "/my-lock/")

	s2, err := concurrency.NewSession(cli)
	if err != nil {
		log.Fatal(err)
	}
	defer s2.Close()
	m2 := concurrency.NewMutex(s2, "/my-lock/")

	// 会话s1获取锁
	if err := m1.Lock(context.TODO()); err != nil {
		log.Fatal(err)
	}
	fmt.Println("acquired lock for s1")

	m2Locked := make(chan struct{})
	go func() {
		defer close(m2Locked)
		// 等待直到会话s1释放了/my-lock/的锁
		if err := m2.Lock(context.TODO()); err != nil {
			log.Fatal(err)
		}
	}()

	if err := m1.Unlock(context.TODO()); err != nil {
		log.Fatal(err)
	}
	fmt.Println("released lock for s1")

	<-m2Locked
	fmt.Println("acquired lock for s2")
}

```

输出:
```
acquired lock for s1
released lock for s1
acquired lock for s2
```

> 参考文献
- [https://etcd.io/docs/v3.4.0/demo/](https://etcd.io/docs/v3.4.0/demo/)
- [https://github.com/etcd-io/etcd/releases](https://github.com/etcd-io/etcd/releases)
