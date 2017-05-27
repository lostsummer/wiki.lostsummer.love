---
title: "CentOS7 64位 上安装 MySql Community 以及配置主备高可用"
date: 2017-05-27 11:41
---

[TOC]

# 安装 (RPM)

## 1. 包准备
下载 [mysql-5.7.18-1.el7.x86_64.rpm-bundle.tar](https://dev.mysql.com/downloads/file/?id=469456)

解压后有多个rpm包，其实冗余。我们只需要

    - mysql-community-common-5.7.18-1.el7.x86_64.rpm
    - mysql-community-libs-5.7.18-1.el7.x86_64.rpm
    - mysql-community-server-5.7.18-1.el7.x86_64.rpm 
    - mysql-community-client-5.7.18-1.el7.x86_64.rpm

你也可以在官网分别单独下载它们

## 2. 冲突预处理 

系统已安装的 mariadb 相关组件会冲突，先卸载
```
[root@localhost packages]# rpm -qa|grep mariadb
mariadb-libs-5.5.44-2.el7.centos.x86_64
[root@localhost packages]# rpm -e --nodeps mariadb-libs
```

## 3. 依次安装 rpm 包

```bash
rpm -ivh  mysql-community-common-5.7.18-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.18-1.el7.x86_64.rpm                      # 依赖于common
rpm -ivh mysql-community-client-5.7.18-1.el7.x86_64.rpm                    # 依赖于libs
rpm -ivh mysql-community-server-5.7.18-1.el7.x86_64.rpm                    # 依赖于client, common
```

## 4. 初始化

```
mysqld --initialize
```

## 5. 改变属主 & 启动数据库

```
chown mysql:mysql /var/lib/mysql -R  
systemctl start mysqld.service
```

## 6. 查看临时密码 & 测试登录

```
grep 'A temporary password' /var/log/mysqld.log
mysql -uroot -p
```

输入密码测试可以成功登录。

## 7. 改密码

```
mysqladmin -u root -p password xxxxxx
```

上面 xxxxxx 为新密码，出现 password: 提示符后输入老密码后确认回车，改密成功

# 配置主备

主机地址：

- A 主机： 192.168.240.28
- B 主机：192.168.240.29

## 1. 修改配置文件：

vi /etc/my.conf

A 主机
```
log-bin=mysql-bin
server-id=28
```

B 主机
```
log-bin=mysql-bin
server-id=29
```

## 2. 创建Replication用户

A主机上执行如下命令：

```
create user 'repl'@'%' identified by '12345678';
 
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
 
grant replication client,replication slave on *.* to 'repl'@'192.168.240.29' identified by '12345678';
```

B主机上执行如下命令：

```
create user 'repl'@'%' identified by '12345678';
 
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
 
grant replication client,replication slave on *.* to 'repl'@'192.168.240.28' identified by '12345678';
```

## 3. 查看两台主机的mysql bin log位置

首先将两台主机mysql中的表锁定

```
FLUSH TABLES WITH READ LOCK;
```
FLUSH TABLES WITH READ LOCK; 代表锁定表，禁止所有操作。防止bin log位置发生变化。
查看A主机bin log位置。

```
SHOW MASTER STATUS;
```

A主机结果
```
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```

B主机结果
```
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |      911 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+

```

记录下A主机结果，和B主机结果
然后再解除两台主机mysql table的锁定
```
unlock tables;
```

## 4. 开始设置 Slave Replication

A主机执行如下命令：
```
STOP SLAVE;
CHANGE MASTER TO MASTER_HOST = '192.168.240.29', MASTER_USER = 'repl',
MASTER_PASSWORD = '12345678', MASTER_LOG_FILE = 'mysql-bin.000002',
MASTER_LOG_POS = 911;
START SLAVE;
```

B主机执行如下命令：
```
STOP SLAVE;
CHANGE MASTER TO MASTER_HOST = '192.168.240.28', MASTER_USER = 'repl',
MASTER_PASSWORD = '12345678', MASTER_LOG_FILE = 'mysql-bin.000002',
MASTER_LOG_POS = 154;
START SLAVE;
```
## 5. 查看两台主机是否设置成功

```
mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.240.29
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 1343
               Relay_Log_File: localhost-relay-bin.000006
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
...
```

如果两台主机Slave_IO_Running 和Slave_SQL_Running都为YES代表设置成功。可以进行数据库操作了。


# 开启半同步复制

要想使用半同步复制，必须满足以下几个条件（按照之前的说明安装就会都满足）：
 
1. MySQL 5.5及以上版本
2. 变量have_dynamic_loading为YES
3. 异步复制已经存在

## 1. 加载插件

两台服务器上都运行
```
mysql> INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';        # 作为主
mysql> INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';          # 作为从
```

## 2. 检查插件是否加载成功

```
mysql> show plugins \G;
...
*************************** 45. row ***************************
   Name: rpl_semi_sync_master
Status: ACTIVE
   Type: REPLICATION
Library: semisync_master.so
License: GPL
*************************** 46. row ***************************
   Name: rpl_semi_sync_slave
Status: ACTIVE
   Type: REPLICATION
Library: semisync_slave.so
License: GPL
```

## 3. 启动半同步复制

两台服务器上
```
mysql> SET GLOBAL rpl_semi_sync_master_enabled = 1;
mysql> SET GLOBAL rpl_semi_sync_slave_enabled = 1;
```

## 4. 重启slave上的IO线程

两台服务器上
```
mysql> STOP SLAVE IO_THREAD;
mysql> START SLAVE IO_THREAD;
```

## 5. 查看半同步是否在运行

两台服务器上都执行下面的验证

验证 master
```
mysql>  show status like 'Rpl_semi_sync_master_status';
+-----------------------------+-------+
| Variable_name               | Value |
+-----------------------------+-------+
| Rpl_semi_sync_master_status | ON    |
+-----------------------------+-------+
```

验证 slave
```
mysql> show status like 'Rpl_semi_sync_slave_status';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Rpl_semi_sync_slave_status | ON    |
+----------------------------+-------+
```

这两个变量常用来监控主从是否运行在半同步复制模式下。
至此，MySQL半同步复制搭建完毕。
