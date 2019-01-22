---
title: "基于rsync的数据异地备份方案"
date: 2017-07-31 16:18
---

# 基于 rsync 的数据异地备份方案

[TOC]

## 备份方案总揽

### 0. 说明

本方案备份数据包含gitlab 和 registry 指定的数据卷全部数据

gitlab的mysql数据不在方案内，备份服务器已经安装了mysql，作为gitlab服务器上mysql的slave在运行。但容器本身每周日生成一份全备文件，在数据卷/backups目录下（/data/gitlab/data/backups），也包含了全部gitlab在mysql上的数据。

### 1. 涉及服务器

| 编号   | 名称       | IP            |
| ---- | -------- | ------------- |
| 1    | GitLab   | 192.168.42.73 |
| 2    | Registry | 192.168.42.76 |
| 3    | Backup   | 192.168.42.74 |



### 2. 数据流向

概览

![](http://140.143.250.15/wiki-img/内网gitlab和registry备份.png)

rsync 推送目录

| 源                         | 目的                             |
| ------------------------- | ------------------------------ |
| GitLab:/data/gitlab       | Backup:/data/backup/gitlab     |
| Registry:/data/emregistry | Backup:/data/backup/emregistry |

备份服务器Backup归档路径

| 源                       | 目的                                       |
| ----------------------- | ---------------------------------------- |
| /data/backup/gitlab     | /data/backup/archive/gitlab_xxxx-xx-xx.tgz |
| /data/backup/emregistry | /data/backup/archive/emregistry_xxxx-xx-xx.tgz |



## 配置 rsync 服务 

Backup 服务器上

- 服务配置文件：/etc/rysncd.conf

```
motd file = /etc/rsyncd.motd
transfer logging = yes
log file = /data/log/rsyncd.log
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsync.lock
port = 873
address = 192.168.42.74
uid = root
gid = root
read only = no
write only = yes
use chroot = no
max connections = 10
[gitlab]
comment = gitlab data
path = /data/backup/gitlab
auth users = gitlab
secrets file = /etc/rsyncd.secrets
hosts allow = 192.168.42.73
hosts deny = *
[emregistry]
comment = emregistry data
path = /data/backup/emregistry
auth users = emregistry
secrets file = /etc/rsyncd.secrets
hosts allow = 192.168.42.76
hosts deny = *
```

- motd 文件：/etc/rsyncd.motd

  该文件为登录rsync服务器时的欢迎提示文本，自行根据需要配置

- 用户口令文件：/etc/rsyncd.secrets

  ```
  gitlab:xxxxxx
  ```
  ```
  emregsitry:xxxxxx
  ```



## 开启 rsyncd 服务

Backup 服务器上

```
systemctl start rsyncd
```



## rsync客户端同步命令

在 GitLab 和 Registry 上

首先为了方便脚本化，避免交互式输入密码，需要

```
echo 'xxxxxx' > /data/rsync.pas
chmod 600 rsync.pas
```

- GitLab 上同步推数据的命令

  ```
  rsync --password-file=/data/rsync.pas -avzP /data/gitlab/ gitlab@192.168.42.74::gitlab
  ```

- Registry 上同步推数据的命令

  ```
  rsync --password-file=/data/rsync.pas -avzP /data/emregistry/ emregistry@192.168.42.74::emregistry
  ```

  ​

## 脚本化同步&归档命令

### 1. GitLab

同步推数据脚本 /data/rsync_push.sh，内容见上一节“rsync客户端同步命令”

定时任务，每周二~周六 0 点执行

```
crontab -e
 0 0 * * 2-6 sh /data/rsync_push.sh
```

### 2. Registry

同 1.

### 3. Backup 服务器归档脚本

/data/backup/mk_tgz.sh

```bash
#!/bin/bash

DIR=/data/backup/archive/

tar cvzf $DIR/gitlab_$(date +%Y-%m-%d).tgz gitlab/
tar cvzf $DIR/emregistry_$(date +%Y-%m-%d).tgz emregistry/

# delete old more than 10 days ago tgz files when size > 500G
if [ $(du -sm $DIR |cut -f1) -gt 500000 ]
then
        find /data/backup/archive -name "*.tgz" -mtime +10 | xargs rm -f
fi
```

定时任务，每周二~周六1点执行

```
crontab -e
 0 01 * * 2-6 sh /data/backup/mk_tgz.sh
```

