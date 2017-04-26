---
title: "使用 CollectD + InfuxDB + Grafana 监控主机"
date: 2017-04-19 15:55
---

原文： [Monitoring hosts with CollectD, InfluxDB and Grafana](http://jansipke.nl/monitoring-hosts-with-collectd-influxdb-and-grafana/ "原文地址")

本篇文章将介绍一种监控主机的解决方案 —— 在 CentOS 7 系统上搭建 CollectD, InfluxDB 和 Grafana 的组合。

[TOC] 

# 安装配置 CollectD

使用 yum 安装

     yum -y install epel-release
     yum -y install collectd

本篇写作之时，以上命令会在系统上安装 CollectD 5.5.0 版本，如果你使用老版本的 CentOS 系统，请确认安装的 Collectd 版本不是 4 及 4 以下版本。

配置文件 /etc/collectd.conf (debian 8 上是 /etc/collectd/collectd.conf) 应配置以下内容：

    FQDNLookup  true
    BaseDir     "/var/lib/collectd"
    PIDFile     "/var/run/collectd.pid"
    PluginDir   "/usr/lib64/collectd"
    TypesDB     "/usr/share/collectd/types.db"
    LoadPlugin  syslog
    LoadPlugin  interface
    LoadPlugin  load
    LoadPlugin  network
    <Plugin interface>
    Interface "eth0"
    IgnoreSelected false
    </Plugin>
    <Plugin load>
        ReportRelative true
    </Plugin>
    <Plugin network>
        Server "127.0.0.1" "25826"
    </Plugin>

最关键的是 “network” plugin，这部分设定监控信息将要发送到 127.0.0.1 的 25826 端口。稍后我们将要指定 InfluxDB 监听此端口以接收 CollectD 的发包。

现在启动这个守护程序，并且将它加入到开机自启动中。

    systemctl start collectd.service
    systemctl enable collectd.service

# 安装配置 InfluxDB 

可以这样安装 InfluxDB

    yum -y install http://influxdb.s3.amazonaws.com/influxdb-0.9.4.2-1.x86_64.rpm

编辑 InfluxDB 的配置文件 /etc/opt/influxdb.conf，将 [collectd] 标签下的内容修改为：

    [collectd]
        enabled = true
        bind-address = "127.0.0.1:25826"
        database = "collectd"
        typesdb = "/usr/share/collectd/types.db"

启动守护程序 

    systemctl start influxdb.service   

你可以验证下 InfluxDB 是否已经收到了 CollectD 发送的监控信息：

    # /opt/influxdb/influx
    Connected to http://localhost:8086 version 0.9.4.2
    InfluxDB shell 0.9.4.2
    > use collectd
    Using database collectd
    > show measurements
    name: measurements
    ------------------
    name
    interface_rx
    interface_tx
    load_longterm
    load_midterm
    load_shortterm

# 安装使用 Grafana

安装和启动

    yum -y install https://grafanarel.s3.amazonaws.com/builds/grafana-2.5.0-1.x86_64.rpm
    systemctl start grafana-server.service
    systemctl enable grafana-server.service

打开浏览器： http://localhost:3000 使用用户名/密码 admin/admin 登录后看到页面
 
![gf1](http://jansipke.nl/wp-content/uploads/grafana1.png)

点击 Data Source 然后 Add new，填写如下配置项

![gf2](http://jansipke.nl/wp-content/uploads/grafana2.png)

InfluxDB 默认的用户名和密码都是 root，测试下连接 (Test the connection) 然后保存 (save)

点击右上角图标，选择 Dashboards / New， 

New Dashboard 右边掩藏一个绿色小矩形， 鼠标移动上去，点击， 选择 Add Panel / Graph

![gf3](http://jansipke.nl/wp-content/uploads/grafana4.png)

点击 Graph 的标题， 选择点击 Edit， 选择 collectd 作为数据源 (data source)

![gf4](http://jansipke.nl/wp-content/uploads/grafana5.png)

编辑窗口里你可以改变 graph 的外观和数据。General 标签下，把标题改为 Load。在 Metrics 标签下，点击 "selecct measurements" 选择 "load_longterm"。 
点击 "WHERE" 后的 "+"，选择 "host"，点击随后出现的 "select tag value"，选择你想要检测的主机名。最后把  "ALIAS BY" 的值改为 "Long" ：

![gf5](http://jansipke.nl/wp-content/uploads/grafana6.png)

点击 " + Query"  添加 mid term 和 short term load 的值。最后点击屏幕顶部的 save 按钮。graph 将呈现像下面这样：

![gf6](http://jansipke.nl/wp-content/uploads/grafana7.png)

我们再添加一个新的 graph 来显示网络速率。点击 "ADD ROW"，然后再次从绿色小矩形上选择 Add Panel / Graph。选择 collectd datasource. 这次我们不使用可视化查询编辑器。interface_rx 和 interface_tx 的统计数据是流经网络接口的总包数或总字节数，但我们需要的是速率，也就是单位时间的字节数。选择右边的三条水平线样式的按钮，选择 "Switch editor mode"：

![gf7](http://jansipke.nl/wp-content/uploads/grafana8.png)

输入框内填入如下查询语句：

    SELECT derivative("value") AS "value" FROM "interface_rx" WHERE "host" = 'test' AND "type" = 'if_octets' AND "instance" = 'eth0'

test  是你的主机名，确保替换正确，确保单引号和双引号输入正确。在 "Axes & Grid tab" 标签下，修改 Y 轴 ( Left Y ) 单位  ( unit ) 为 "bytes per second"：

![gf8](http://jansipke.nl/wp-content/uploads/grafana9.png)

点击 "+ Query" 添加 interface_tx 的查询，方法同上。为两个查询添加别名 ( Alias )： interface_rx : Received (接收), interface_tx : Transmitted (输送)。 graph 将变成类似下面的样子：

![gf9](http://jansipke.nl/wp-content/uploads/grafana10.png)

