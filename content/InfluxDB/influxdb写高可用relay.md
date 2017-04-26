---
title: "InfluxDB 写高可用 Relay"
date: 2017-04-19 17:32
---

项目: [influxdata/influxdb-relay](https://github.com/influxdata/influxdb-relay)

[TOC]

# InfluxDB Relay

为 InfluxDB 提供基本的高可用层
依赖 go 1.5++

## 原理

只负责 write 的高可用，write 时根据配置的每个节点都要写一次，相当于多写
read 需要其他工具做 Load Balance

## 安装

从源码编译运行

```sh
$ # Install influxdb-relay to your $GOPATH/bin
$ go get -u github.com/influxdata/influxdb-relay
$ # Edit your configuration file
$ cp $GOPATH/src/github.com/influxdata/influxdb-relay/sample.toml ./relay.toml
$ vim relay.toml
$ # Start relay!
$ $GOPATH/bin/influxdb-relay -config relay.toml
```

## 配置

```toml
[[http]]
# Name of the HTTP server, used for display purposes only.
name = "example-http"

# TCP address to bind to, for HTTP server.
bind-addr = "127.0.0.1:9096"

# Enable HTTPS requests.
ssl-combined-pem = "/etc/ssl/influxdb-relay.pem"

# Array of InfluxDB instances to use as backends for Relay.
output = [
    # name: name of the backend, used for display purposes only.
    # location: full URL of the /write endpoint of the backend
    # timeout: Go-parseable time duration. Fail writes if incomplete in this time.
    # skip-tls-verification: skip verification for HTTPS location. WARNING: it's insecure. Don't use in production.
    { name="local1", location="http://127.0.0.1:8086/write", timeout="10s" },
    { name="local2", location="http://127.0.0.1:7086/write", timeout="10s" },
]

[[udp]]
# Name of the UDP server, used for display purposes only.
name = "example-udp"

# UDP address to bind to.
bind-addr = "127.0.0.1:9096"

# Socket buffer size for incoming connections.
read-buffer = 0 # default

# Precision to use for timestamps
precision = "n" # Can be n, u, ms, s, m, h

# Array of InfluxDB instances to use as backends for Relay.
output = [
    # name: name of the backend, used for display purposes only.
    # location: host and port of backend.
    # mtu: maximum output payload size
    { name="local1", location="127.0.0.1:8089", mtu=512 },
    { name="local2", location="127.0.0.1:7089", mtu=1024 },
]
```
 
## 描述

架构相当简单，由一个负载均衡器，两个以上 InfluxDB relay 进程和两个以上 InfluxDB 进程组成。负载均衡器要将 UDP 数据和发往路径 "/write" 的 HTTP POST 请求定向到两个 relay，将方法路径 "/query" 的 GET 请求定向到两个InfluxDB。

安装起来像下面这样：

```
        ┌─────────────────┐
        │writes & queries │
        └─────────────────┘
                 │
                 ▼
         ┌───────────────┐
         │               │
┌────────│ Load Balancer │─────────┐
│        │               │         │
│        └──────┬─┬──────┘         │
│               │ │                │
│               │ │                │
│        ┌──────┘ └────────┐       │
│        │ ┌─────────────┐ │       │┌──────┐
│        │ │/write or UDP│ │       ││/query│
│        ▼ └─────────────┘ ▼       │└──────┘
│  ┌──────────┐      ┌──────────┐  │
│  │ InfluxDB │      │ InfluxDB │  │
│  │ Relay    │      │ Relay    │  │
│  └──┬────┬──┘      └────┬──┬──┘  │
│     │    |              |  │     │
│     |  ┌─┼──────────────┘  |     │
│     │  │ └──────────────┐  │     │
│     ▼  ▼                ▼  ▼     │
│  ┌──────────┐      ┌──────────┐  │
│  │          │      │          │  │
└─▶│ InfluxDB │      │ InfluxDB │◀─┘
   │          │      │          │
   └──────────┘      └──────────┘
```

## 缓冲

relay 可以通过配置为 HTTP 后端缓冲错误请求
这个设计逻辑是为了减少在网络中断或周期性的网络问题发生时请求错误的数量

> 这个重试逻辑并 **不** 适合长时间中断的情况，因为所有数据都保存在内存中

选项 （每 HTTP 后端）

* buffer-size-mb -- 端点数据保存在内存中的上限 (MB)
* max-batch-kb -- 要发送的聚合批数据的最大尺寸 (in KB)
* max-delay-interval -- 和每个后端每次重传的最大延迟.
    初始值为500ms，每次失败后翻倍

如果缓冲满，会丢弃新的请求，并记录错误日志
进入缓冲的请求会不断重传直至写入成功

重传数据会被序列化到单独的后端。需要补充的是，写数据会被聚合成批数据，尺寸在配置的最大值范围内尽可能大。
如果缓冲的数据成功写入，后续的数据将无延迟尝试。

如果一个后端 down 掉而 relay 数据还没有写入, 那么这些没被写入的 relay 数据会在线驻留在缓冲中，直到缓冲被刷掉，这意味着不需要运维余外干涉去恢复数据，数据会自己聚合成批，按照接收顺序写到恢复数据库中。

*NOTE*: The limits for buffering are not hard limits on the memory usage of the application, and there will be additional overhead that would be much more challenging to account for. The limits listed are just for the amount of point line protocol (including any added timestamps, if applicable). Factors such as small incoming batch sizes and a smaller max batch size will increase the overhead in the buffer. There is also the general application memory overhead to account for. This means that a machine with 2GB of memory should not have buffers that sum up to _almost_ 2GB.

## 恢复

InfluxDB 把它的磁盘数据组织成时间逻辑块，叫做 shard，我们可以借此创建一个零故障时间的热备进程。

一个 shard 呈现在 InfluxDB 中的时间一般是1小时，1天，或7天，取决于数据保留时间，但是可以在创建数据保留策略时显式设置。我们的例子中，假设shard的保留期是1天。

比如说，一个 InfluxDB 服务器在 2016-03-10 down 掉一小时。一旦跨过午夜 UTC 时间，所有的 InfluxDB 进程正把 2016-03-11 的数据写入 shard，2016-03-10 的文件被冻结写操作。我么可以靠以下步骤复原数据：

1. 通知 LB 停止向 down 掉的服务器发送查询数据（需要一侦测到故障马上操作以阻止查询返回切片或不连续)
2. 从整天都运行的服务器上创建一个 2016-03-10 的备份
3. 在 down 掉的机器上用这个备份恢复 shard
4. 通知LB向之前 down 掉的服务器恢复发送查询

整个过程Relay需要向每台服务器发送写数据，包括之前down掉过的。

## Sharding

可以在这种安装方式之上为 shard 数据再添加一层。根据需要你可以根据 measurement 名称或特定的 tag，比如 customer_id，来建立 shard。sharding 层既可以为查询服务业可以为数据写入服务。

因为 relay 不处理查询，所以它不实现任何 sharding 逻辑。sharding 必须在 relay 层外部做。

## 忠告

虽然 relay 提供了一些层次的高可用特性，但仍有一些场景需要说明下：

- influxdb-relay 并不处理 /query 端，包括数据库 schema 的修改（create, drop，etc）。 这就要求在写库数据进入前必须预先创建好数据库。
- 连续查询（continuous query）仍然只在本地写查询结果。如果一台服务器 down 掉，数据恢复之后连续查询需要自己回填数据。
- 重写（overwrite）数据有潜在的不可预期性。比如，两台数据库 A 和 B，如果 B down 掉，数据 X （我们记为 X1）恰好在 B 恢复上线前写入，这个写操作在队列中排在 B 下线时的其他写操作之后。一旦 B 重新上线，第一个缓存的数据写入成功，所有新的写入数据被放行。这时（X1 写入 B 之前）, X 又向 A 和 B 被写了一遍（此时值为 X2），当 relay 到达 B 的写缓冲底部时，它会把 X （值为 X1）写入 B …… 此时 A 中有 X2，B 中有 X1.
	- 如果可以最好避免重写数据点，否则就要注意重写一个数据点的同一个值段会导致数据不一致。
	- 可以这样规避，等待缓冲区数据全部刷出后，在开放数据写入。

## 构建

推荐使用 docker 方式编译，Dockerfile (Dockerfile_build_ubuntu64) 提供了包含所有依赖包的编译环境。

构建 docker 镜像：

```shell
docker build -f Dockerfile_build_ubuntu64 -t influxdb-relay-builder:latest .
```

编译构建工程：

```shell
docker run --rm -v $(pwd):/root/go/src/github.com/influxdata/influxdb-relay influxdb-relay-builder
```

调用的是 build.py 脚本，构建输出在 ./build 目录。查看构建可以命令列表，用 --help 选项

```shell
docker run -v $(pwd):/root/go/src/github.com/influxdata/influxdb-relay influxdb-relay-builder --help
```

## 打包

构建各种 Linux 系统包（deb，rpm，etc），使用 --package 选项:

```shell
docker run -v $(pwd):/root/go/src/github.com/influxdata/influxdb-relay influxdb-relay-builder --package
```




## 搭建集群

### 节点准备

node_relay 安装好
node_a, node_b 按标准安装好

```
node_relay: 192.168.1.112
node_a: 192.168.1.12
node_b: 192.168.1.15
```

### 生成 http使用的pem 

密码: influxdb123

openssl genrsa -des3 -out /etc/influxdb/influxdb-relay.pem 2048

### relay 配置

vim /etc/influxdb/relay.toml

```toml
[[http]]
name = "influxdb-http-dba"
bind-addr = "0.0.0.0:9096"
output = [
    { name="http-12", location = "http://192.168.1.12:8086/write", timeout="10s", buffer-size-mb = 100, max-batch-kb = 50, max-delay-interval = "5s" },
    { name="http-15", location = "http://192.168.1.15:8086/write", timeout="10s", buffer-size-mb = 100, max-batch-kb = 50, max-delay-interval = "5s" },
]

[[udp]]
name = "influxdb-udp-dba"
bind-addr = "0.0.0.0:9099"
read-buffer = 0
precision = "n"
output = [
    { name="udp-12", location="192.168.1.12:8089", mtu=512 },
    { name="udp-15", location="192.168.1.15:8089", mtu=1024 },
]
```

## 启动

```shell
nohup $GOPATH/bin/influxdb-relay -config /etc/influxdb/relay.toml >/tmp/influxdb_relay.log 2>/dev/null &
```
