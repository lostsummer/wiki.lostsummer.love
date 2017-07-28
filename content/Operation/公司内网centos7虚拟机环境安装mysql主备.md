---
title: "公司内网CentOS7虚拟机环境安装MySql主备"
date: 2017-07-26 16:26
---

[TOC]

# 单机安装 & 基本配置

以下步骤主备环境都执行一遍

## 1. 包准备

````shell
mkdir -p /home/wangyx/packages
cd /home/wangyx/packages
scp root@192.168.240.28:/root/packages/mysql-5.7.18-1.el7.x86_64.rpm-bundle.tar .
````

解压后有多个rpm包，其实冗余。我们只需要

- mysql-community-common-5.7.18-1.el7.x86_64.rpm

- mysql-community-libs-5.7.18-1.el7.x86_64.rpm

- mysql-community-server-5.7.18-1.el7.x86_64.rpm

- mysql-community-client-5.7.18-1.el7.x86_64.rpm

  ​

## 2. 冲突预处理

系统已安装的 mariadb 相关组件会冲突，先卸载

```shell
[root@localhost packages]# rpm -qa|grep mariadb
mariadb-libs-5.5.44-2.el7.centos.x86_64
[root@localhost packages]# rpm -e --nodeps mariadb-libs
```

修改 SELinux 配置

```
vi /etc/selinux/config

SELINUX=permissive           # 原为enforce
```

配置生效需要reboot重启系统

免重启，可以使用shell命令临时修改

```
setenforce 0
```

firewalld 开放 3306 端口

```
firewall-cmd --zone=public --add-port=3306/tcp --permanent
firewall-cmd --reload
```



## 3. 依次安装 rpm 包

```shell
rpm -ivh  mysql-community-common-5.7.18-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.18-1.el7.x86_64.rpm                  # 依赖于common
rpm -ivh mysql-community-client-5.7.18-1.el7.x86_64.rpm                # 依赖于libs
rpm -ivh mysql-community-server-5.7.18-1.el7.x86_64.rpm                # 依赖于client, common
```



## 4. 修改配置文件

修改数据文件路径、数据库默认字符集

字符集使用utf8mb4原因详解：http://blog.csdn.net/woslx/article/details/49685111



```shell
vi /etc/my.cnf
```

```ini
[mysqld]
datadir=/data/mysql                             # 修改
socket=/data/mysql/mysql.sock                   # 修改
character-set-client-handshake = FALSE          # 添加
character-set-server = utf8mb4                  # 添加
collation-server = utf8mb4_unicode_ci           # 添加
init_connect='SET NAMES utf8mb4'                # 添加
[mysql]                                         # [mysql]标签及以下为添加
socket=/data/mysql/mysql.sock
default-character-set=utf8mb4
[client]                                        # [client]标签及以下为添加
default-character-set=utf8mb4
[mysqladmin]                                    # [mysqladmin]标签及以下为添加
socket=/data/mysql/mysql.sock
```



## 5. 创建数据目录

```
mkdir -p /data/mysql
chown mysql:mysql /data/mysql -R 
```



## 6. 初始化 & 启动

```
mysqld --initialize
systemctl start mysqld.service
```



## 7. 查看临时密码 & 测试登录

```
grep 'A temporary password' /var/log/mysqld.log
mysql -uroot -p
```

输入密码测试可以成功登录。



## 8. 改密码

```
mysqladmin -u root -p password xxxxxx

```

上面 xxxxxx 为新密码，出现 password: 提示符后输入老密码后确认回车，改密成功



## 9. 允许root用户远程登录

mysql -uroot -p 进入 mysql

```mysql
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'mypassword' WITH GRANT OPTION;
mysql> FLUSH PRIVILEGES;
```



## 10. 查看数据库默认字符集

mysql -uroot -p 进入 mysql

```mysql
mysql> SHOW VARIABLES WHERE Variable_name LIKE 'character_set_%' OR Variable_name LIKE 'collation%';
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8mb4                    |
| character_set_connection | utf8mb4                    |
| character_set_database   | utf8mb4                    |
| character_set_filesystem | binary                     |
| character_set_results    | utf8mb4                    |
| character_set_server     | utf8mb4                    |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
| collation_connection     | utf8mb4_unicode_ci         |
| collation_database       | utf8mb4_unicode_ci         |
| collation_server         | utf8mb4_unicode_ci         |
+--------------------------+----------------------------+

```



# 主备配置



## 1. 配置准备

主机地址

| 编号   | 角色     | IP            |
| ---- | ------ | ------------- |
| A    | master | 192.168.42.73 |
| B    | slave  | 192.168.42.74 |

两台主机的配置文件 /etc/my.cnf 的 [mysqld] 添加下列项目

- A

  ```ini
  log-bin=mysql-bin
  server-id=73
  ```

- B

  ```ini
  log-bin=mysql-bin
  server-id=74
  ```




## 2. 创建Replication用户

A主机上执行如下命令：

```mysql
create user 'repl'@'%' identified by 'mypassword';

GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';

grant replication client,replication slave on *.* to 'repl'@'192.168.42.74' identified by 'mypassword';
```



## 3. 查看master的mysql bin log位置

A主机上，首先将mysql中的表锁定

```mysql
mysql> FLUSH TABLES WITH READ LOCK;
```

FLUSH TABLES WITH READ LOCK; 代表锁定表，禁止所有操作。防止bin log位置发生变化。 查看A主机bin log位置。

```mysql
mysql> SHOW MASTER STATUS
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |     1062 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```



## 4. 开始设置 Slave Replication

B主机执行如下命令：

```
STOP SLAVE;
CHANGE MASTER TO MASTER_HOST = '192.168.42.73', MASTER_USER = 'repl',
MASTER_PASSWORD = 'emoney.cn', MASTER_LOG_FILE = 'mysql-bin.000002',
MASTER_LOG_POS = 1062;
START SLAVE;
```

 查看是否设置成功

```
mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.42.73
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

如果lave_IO_Running 和Slave_SQL_Running都为YES代表设置成功。可以进行数据库操作了。





# 开启半同步复制

要想使用半同步复制，必须满足以下几个条件（按照之前的说明安装就会都满足）：

1. MySQL 5.5及以上版本
2. 变量have_dynamic_loading为YES
3. 异步复制已经存在

## 1. 加载插件

### master

```mysql
mysql> INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';   
```

 检查插件是否加载成功

```mysql
mysql> show plugins \G;
...
*************************** 45. row ***************************
   Name: rpl_semi_sync_master
Status: ACTIVE
   Type: REPLICATION
Library: semisync_master.so
License: GPL
```

###  slave

```mysql
mysql> INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
```

检查插件是否加载成功

```mysql
mysql> show plugins \G;
...
*************************** 45. row ***************************
   Name: rpl_semi_sync_slave
 Status: ACTIVE
   Type: REPLICATION
Library: semisync_slave.so
License: GPL
```



## 2. 启动半同步复制

### master

```mysql
mysql> SET GLOBAL rpl_semi_sync_master_enabled = 1;
```

### slave

```mysql
mysql> SET GLOBAL rpl_semi_sync_slave_enabled = 1;
```

## 3. 重启slave上的IO线程

### slave

```mysql
mysql> STOP SLAVE IO_THREAD;
mysql> START SLAVE IO_THREAD;
```

## 4. 查看半同步是否在运行

两台服务器上执行下面的验证

### master

```
mysql>  show status like 'Rpl_semi_sync_master_status';
+-----------------------------+-------+
| Variable_name               | Value |
+-----------------------------+-------+
| Rpl_semi_sync_master_status | ON    |
+-----------------------------+-------+
```

### slave

```
mysql> show status like 'Rpl_semi_sync_slave_status';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Rpl_semi_sync_slave_status | ON    |
+----------------------------+-------+
```

这两个变量常用来监控主从是否运行在半同步复制模式下。 至此，MySQL半同步复制搭建完毕。
