---
title: "配置 InfluxDB 鉴权及 HTTP API 写数据的方法"
date: 2017-04-19 16:02
---

本文简要描述如何为 InfluxDB 开启鉴权和配置用户管理权限（安装后默认不需要登录），以及开启鉴权后如何使用 HTTP API 写数据。

[TOC]

<br />
# 创建 InfluxDB 管理员账号

__创建 admin 帐号密码并赋予所有数据库权限__

1. 创建

        CREATE USER admin WITH PASSWORD 'admin'

2. 赋权
 
        GRANT ALL PRIVILEGES TO admin

__其他命令__

- 修改用户(密码)
 
        SET PASSWORD FOR admin = 'admin'
 
- 删除用户
 
        DROP USER admin

- 撤消权限
 
        REVOKE ALL  ON  mydb   FROM admin

- 查看权限

        SHOW GRANTS FOR admin

<br />
# 打开认证

    vi /etc/influxdb/influxdb.conf

把 [http] 标签下的 auth-enabled 选项值改为 true

    [http]  
    enabled = true  
    bind-address = ":8086"  
    auth-enabled = true 
    log-enabled = true  
    write-tracing = false  
    pprof-enabled = false  
    https-enabled = false  
    https-certificate = "/etc/ssl/influxdb.pem"

重启 InfluxDB

__命令行 CLI 登录__

    $ influx -username admin -password admin
    Connected to http://localhost:8086 version 0.13.x
    InfluxDB shell 0.13.x
    >

或者

    $ influx
    Connected to http://localhost:8086 version 0.13.x
    InfluxDB shell 0.13.x
    > auth
    username: admin
    password:
    >

__HTTP API 查询的方式变为__

    curl -G "http://localhost:8086/query" -u admin:admin --data-urlencode "q=SHOW DATABASES"
    curl -G "http://localhost:8086/query" --data-urlencode "u=admin" --data-urlencode "p=admin" --data-urlencode "q=SHOW DATABASES"
    curl -G "http://localhost:8086/query?u=admin&p=admin&q=SHOW+DATABASES"

<br />
# 开启鉴权后如何写数据

使用insert命令行写数据：

    > INSERT cpu_load_short,host=server01,region=us-west value=0.64,value2=0.86 1434055562000000000
 
使用HTTP API写数据：

- 用户名密码写在 URL 中
 
        curl -i -X POST "http://localhost:8086/write?db=mydb&u=admin&p=admin" --data-binary "cpu_load_short,host=server01,region=us-west value=0.64,value2=0.86 1434055562000000000"

- 用户名密码写在 HTTP 头 Authorization 选项

		curl -i -X POST "http://localhost:8086/write?db=mydb" -u admin:admin --data-binary "cpu_load_short,host=server01,region=us-west value=0.64,value2=0.86 1434055562000000000" 
 
除了命令关键字不同，数据部分基本相同，仅观察一下数据部分：
 
    cpu_load_short,host=server01,region=us-west value=0.64,value2=0.86 1434055562000000000
 
这行数据含有两个空格，将之分为三个部分：
 
- 第一部分： cpu_load_short,host=server01,region=us-west

  第一部分称为key，key中包含了measurement name和tags（tags又分为tag key和tag value，tags可以有多个），例子中的“cpu_load_short”就是measurement name，“host”和“reginon”是tag key，“server01”和“us-west”是对应的tag value。在我看来，measurement name和MySQL数据库的表名是等同的。另外，在tag value中的空格应以“/”加上空格表示，若有多个tag value，可以使用逗号隔开，但是应以“/”加逗号表示。
 
- 第二部分：value=0.64,value2=0.86

  第二部分称为Field，同样和tags的形式相同，都是键值对的形式，但是tags中的值必须是string类型，而Field中的值可以为Integer、float、Boolean、string类型，若为Integer类型，则值后必须加“i”，否则该值为float类型，比如value=23意味着这个值23是float类型，而value=23i，意味着值23是Integer类型。Boolean类型的值的表示方式有很多，直接写成：t, T, true, TRUE, f, F, false或 FALSE都可以。

- 第三部分（可选）：1434055562000000000

  第三部分称为Timestamp，是时间戳，如果该部分省略，则默认将当前时间的时间戳插入数据库，否则按照用户输入的时间戳插入。


第一种写数据方式
产生 HTTP 包(使用 Chrome 插件 Postman 生成)

    POST /write?db=mydb HTTP/1.1
    Host: 192.168.6.93:8086
    Authorization: Basic YWRtaW46YWRtaW4=
    Cache-Control: no-cache
    Postman-Token: d3ce583a-1e11-63ec-a818-034aeaa91f10
 
    cpu_load_short,host=server01,region=us-west value=0.64,value2=0.86 1434055565000000000


第二种写数据方式
产生 HTTP 包

    POST /write?db=mydb&amp;u=admin&amp;p=admin HTTP/1.1
    Host: 192.168.6.93:8086
    Cache-Control: no-cache
    Postman-Token: 6773f66a-de4c-435d-6469-0a1194752a86
 
    cpu_load_short,host=server01,region=us-west value=0.64,value2=0.86 1434055565000000000


参考： [访问需要HTTP Basic Authentication认证的资源的各种语言的实现](http://www.cnblogs.com/QLeelulu/archive/2009/11/22/1607898.html)
