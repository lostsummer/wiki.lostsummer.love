---
title: "InfluxDB + relay + keepalived  双机高可用配置"
date: 2017-05-12 16:05
---

[TOC]

# 硬件

内网互通 CentOS64 主机 x 2

添加 /etc/hosts

172.31.37.6

```
172.31.37.6     37D6.localhost 37D6
172.31.37.7     37D7
172.28.1.120    smtp.emoney.cn
```

172.31.37.7

```
172.31.37.7     37D7.localhost 37D7
172.31.37.6     37D6
172.28.1.120    smtp.emoney.cn
```

# InfluxDB + Relay 安装配置

安装参见 [InfluxDB 写高可用 Relay](http://wiki.lostsummer.love/InfluxDB/influxdb%E5%86%99%E9%AB%98%E5%8F%AF%E7%94%A8relay.html)

## 架构差异：

相比较上面官方给出的方案，取消了 LB，使用 Keepalived

![myrelay](http://140.143.250.15/wiki-img/myrelay.png)

## Relay 配置

- relay http://0.0.0.0:9096/write   -> InfluxDB http://0.0.0.0:8086/write
- relay udp 0.0.0.0:9096            -> InfluxDB udp 0.0.0.0:8089



172.31.37.6

```toml
[[http]]
name = "relay2-http"
bind-addr = "0.0.0.0:9096"
output = [
    { name="influxdb1", location = "http://37D7:8086/write" },
    { name="influxdb2", location = "http://37D6:8086/write" },
]
 
[[udp]]
name = "relay2-udp"
bind-addr = "0.0.0.0:9096"
read-buffer = 0 # default
output = [
    { name="influxdb1", location="37D7:8089", mtu=1024 },
    { name="influxdb2", location="37D6:8089", mtu=1024 },
]
```

172.31.37.7

```toml
[[http]]
name = "relay2-http"
bind-addr = "0.0.0.0:9096"
output = [
    { name="influxdb1", location = "http://37D7:8086/write" },
    { name="influxdb2", location = "http://37D6:8086/write" },
]
 
[[udp]]
name = "relay2-udp"
bind-addr = "0.0.0.0:9096"
read-buffer = 0 # default
output = [
    { name="influxdb1", location="37D7:8089", mtu=1024 },
    { name="influxdb2", location="37D6:8089", mtu=1024 },
]
```


# Keepalived 安装配置

yum 安装

## 配置

### 配置心跳检查

为 relay /write 和 influxdb /query 分别配置 virtual_server  

先在每台主机上获取diggest, /write 端 类似

```shell
genhash -s 172.31.37.7 -p 8086 -u /query
```

其他详细部分见后面完整配置文件

### 主备切换活性检查

以 relay /write 为主，暂时只检查它

```shell
curl http://localhost:9096/write && exit 0 || exit 1
```

其他详细部分见后面完整配置文件

### 完整配置文件

#### keepalived.conf on 172.31.37.6

```config
! Configuration File for keepalived

global_defs {
   notification_email {
     wangyuxin0623@emoney.cn
   }
   notification_email_from wangyuxin0623@emoney.cn
   smtp_server smtp.emoney.cn
   smtp_connect_timeout 30
   router_id LVS_DEVEL
}

vrrp_script chk_down {
    script "curl http://localhost:9096/write && exit 0 || exit 1"   # http response
    interval 1
    weight -2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass emon
    }
    virtual_ipaddress {
        172.31.37.201
    }
    track_interface {
        eth0
    }
    track_script {
        chk_down
    }
    notify_master "/etc/keepalived/sendmail.py master"
    notify_backup "/etc/keepalived/sendmail.py backup"
    notify_fault "/etc/keepalived/sendmail.py fault"
}

virtual_server 172.31.37.201 8086 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    nat_mask 255.255.255.0
    persistence_timeout 50
    protocol TCP

    real_server 172.31.37.6 8086 {
        weight 1
        HTTP_GET {
            url {
              path /query
              digest 9506e2817de5457b3ebe9bc36f33e444
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
    real_server 172.31.37.7 8086 {
        weight 1
        HTTP_GET {
            url {
              path /query
              digest 9506e2817de5457b3ebe9bc36f33e444
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}

virtual_server 172.31.37.201 9096 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    nat_mask 255.255.255.0
    persistence_timeout 50
    protocol TCP

    real_server 172.31.37.6 9096 {
        weight 1
        HTTP_GET {
            url {
              path /write
              digest 8fa2fa624e3dd3163f007e7acfab9d46
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
    real_server 172.31.37.7 9096 {
        weight 1
        HTTP_GET {
            url {
              path /write
              digest 8fa2fa624e3dd3163f007e7acfab9d46
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
```

#### keepalived.conf on 172.31.37.7 
只有 vrrp_instance VI_1 段两处不同

```
    state BACKUP
    ...
    priority 99
```

### 通知邮件脚本 sendmail.py 参考

```python
import smtplib
from email.mime.text import MIMEText
import os
import sys
import time

smtp_server = 'smtp.emoney.cn'
user = 'baoyifeng@emoney.cn'
password = '*******'
mail_from = 'baoyifeng@emoney.cn'
display_from = 'keepalive_influx<{0}>'.format(mail_from)
mail_to = [
    "wangyuxin0623@emoney.cn",
    "lostsummer@gmail.com"
]
vip = '172.31.37.201'

def send_mail(mail_to, subject, content):
    msg = MIMEText(content, _subtype='plain', _charset='utf-8')
    msg['Subject'] = subject
    msg['From'] = display_from

    try:
        smtp = smtplib.SMTP()
        smtp.connect(smtp_server)
        smtp.login(user, password)
        smtp.sendmail(mail_from, mail_to, msg.as_string())
        smtp.close()
        print('send ok')
        return True
    except Exception as e:
        senderr = str(e)
        print(senderr)
        return False


if __name__ == "__main__":
    if (len(sys.argv) < 2):
        print("less arg!")
        sys.exit(1)
    stat = sys.argv[1]
    if stat not in ["master", "backup", "fault"]:
        print("error arg!")
        sys.exit(2)

    hostip = os.popen(
        "ifconfig eth0 | awk '/inet /{print $2}'").readlines()[0].strip()

    timestamp = time.strftime('%Y-%m-%d %X', time.localtime())

    subject = "{0} is change to be {1}, {2} floating".format(
        hostip, stat, vip)

    content = "{0} :\nvrrp transition, {1} changed to be {2}".format(
        timestamp, hostip, stat)

    sys.exit(0) if send_mail(mail_to, subject, content) else sys.exit(3)
```


