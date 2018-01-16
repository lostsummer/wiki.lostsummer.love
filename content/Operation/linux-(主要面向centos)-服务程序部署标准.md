---
title: "Linux (主要面向CentOS) 服务程序部署标准"
date: 2017-05-15 12:18
---

[TOC]

# 目的

1. 服务器磁盘分区空间的合理使用
2. 统一服务程序的配置、日志和数据存放位置，编译查找维护

# 原则

## 1. 大文件存放在数据分区

大文件包括数据库文件，大量日志文件的目录等

目前运维提供的大容量数据卷挂载在 /data，所以程序的大文件应该配置在 /data/服务名/ 目录下

## 2. /emoney 下统一放置安装程序的配置、日志和数据文件

逻辑上提供访问性即可，可以是连接。一个服务在 /emoney 下的具体各子目录如下

```
/emoney/service_name/conf
/emoney/service_name/data
/emoney/service_name/log
```

# 实例：CentOS7 部署 InfluxDB 程序

## 1. rpm 包安装
从官方地址下载 rpm 包，并使用 yum 安装
```
wget https://dl.influxdata.com/influxdb/releases/influxdb-1.2.2.x86_64.rpm
sudo yum localinstall influxdb-1.2.2.x86_64.rpm
```
## 2. 配置数据目录
把数据文件目录配置到 /data 下
修改配置文件
```
[root@37D6 influxdb]# vi /etc/influxdb/influxdb.conf
```
把
```
[meta]
  # Where the metadata/raft database is stored
  dir = "/var/lib/influxdb/meta"
 
  # Automatically create a default retention policy when creating a database.
  # retention-autocreate = true
 
  # If log messages are printed for the meta service
  # logging-enabled = true
 
###
### [data]
###
### Controls where the actual shard data for InfluxDB lives and how it is
### flushed from the WAL. "dir" may need to be changed to a suitable place
### for your system, but the WAL settings are an advanced configuration. The
### defaults should work for most systems.
###
 
[data]
  # The directory where the TSM storage engine stores TSM files.
  dir = "/var/lib/influxdb/data"
 
  # The directory where the TSM storage engine stores WAL files.
  wal-dir = "/var/lib/influxdb/wal"
```
改成
```
[meta]
  # Where the metadata/raft database is stored
  dir = "/data/influxdb/meta"
 
  # Automatically create a default retention policy when creating a database.
  # retention-autocreate = true
 
  # If log messages are printed for the meta service
  # logging-enabled = true
 
###
### [data]
###
### Controls where the actual shard data for InfluxDB lives and how it is
### flushed from the WAL. "dir" may need to be changed to a suitable place
### for your system, but the WAL settings are an advanced configuration. The
### defaults should work for most systems.
###
 
[data]
  # The directory where the TSM storage engine stores TSM files.
  dir = "/data/influxdb/data"
 
  # The directory where the TSM storage engine stores WAL files.
  wal-dir = "/data/influxdb/wal"
```
## 3. /emoney 下创建链接
```
[root@37D6 influxdb]# ln -s /etc/influxdb /emoney/influxdb/conf
[root@37D6 influxdb]# ln -s /var/log/influxdb /emoney/influxdb/log
[root@37D6 influxdb]# ln -s /data/influxdb /emoney/influxdb/data
```

