---
title: "基于Docker的内网集成环境搭建"
date: 2017-07-28 19:08
---

[TOC]

# 部署计划

## 主机及用途

| 编号   | IP            | 用途                             |
| ---- | ------------- | ------------------------------ |
| 1    | 192.168.42.73 | GitLab服务器                      |
| 2    | 192.168.42.74 | 数据备份（GitLab 及 Docker Registry） |
| 3    | 192.168.42.75 | CI Runner服务器                   |
| 4    | 192.168.42.76 | Docker镜像仓库                     |

## 部署程序

### 1. GitLab 服务器

| 顺序编号 | 安装程序                                | 备注   |
| ---- | ----------------------------------- | ---- |
| 1    | MySql                               | 已装   |
| 2    | Docker                              |      |
| 3    | Git                                 |      |
| 4    | docker-compose 工具                   |      |
| 5    | Gitlab, Redis 容器 （docker-compose编排） |      |

### 2. 数据备份服务器

| 顺序编号 | 安装程序   | 备注   |
| ---- | ------ | ---- |
| 1    | MySql  | 已装   |
| 2    | Docker | 备用   |
| 3    | Git    | 备用   |

### 3. CI Runner 服务器

| 顺序编号 | 安装程序         | 备注   |
| ---- | ------------ | ---- |
| 1    | Docker       |      |
| 2    | Git          |      |
| 3    | CI Runner 容器 |      |

### 4. Docker镜像仓库

| 顺序编号 | 安装程序                   | 备注   |
| ---- | ---------------------- | ---- |
| 1    | Docker                 |      |
| 2    | Git                    |      |
| 3    | docker-compose 工具      |      |
| 4    | Docker Registry 容器     |      |
| 5    | openssl, openssl-devel |      |



## 部署后工作

1. 从测试gitlab服务器192.168.240.28导入所有repo和用户数据
2. 完成gitlab和registry数据定时备份脚本并启动任务



# Docker 安装

1-4四台服务器部署方式及目录相同，都需要将数据配置到 /data 以下。

## 删除旧版

避免冲突，删除系统已安装的旧版。

命令

```
yum remove docker \
                  docker-common \
                  docker-selinux \
                  docker-engine
                  
rm -rf /var/lib/docker
```



## 通过线上脚本安装

前提：主机访问外网权限，已安装yum

```
curl -sSL https://get.docker.com/ | bash -x
```

## 改变数据目录配置和存储驱动[^1]

```
vi /etc/docker/daemon.json                      # 新文件
```

```json
{
  "graph": "/data/docker",
  "storage-driver": "devicemapper"              # 存储驱动类型
}
```

## firewalld 开放端口

```
firewall-cmd --permanent --zone=public --add-interface=docker0
firewall-cmd --permanent --zone=public --add-port=2377/tcp
firewall-cmd --permanent --zone=public --add-port=7946/tcp
firewall-cmd --permanent --zone=public --add-port=7046/udp
firewall-cmd --permanent --zone=public --add-port=4789/tcp
firewall-cmd --permanent --zone=public --add-port=4789/udp
firewall-cmd --reload
```

>**注意：** 当 firewalld 启动或者重启的时候，将会从 iptables 中移除 DOCKER 的规则，从而影响了 Docker 的正常工作。
>当你使用的是 Systemd 的时候， firewalld 会在 Docker 之前启动，但是如果你在 Docker 启动之后再启动 或者重启 firewalld ，你就需要重启 Docker 进程了。

## 启动 Docker

```
systemctl start docker
```

验证

```
systemctl status docker
ls -l /data/docker
```



# Docker-compose 工具安装

在 1， 4 服务器上安装 docker-compose  工具

```
yum install python-pip
pip install docker-compose
```



# GitLab 安装

数据库

```
CREATE USER 'gitlab'@'%' IDENTIFIED BY 'password';
CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;
GRANT ALL PRIVILEGES ON `gitlabhq_production`.* TO 'gitlab'@'%';
```

docker

```
git clone ssh://git@192.168.240.28:2222/research/gitlab-docker-compose.git /data/gitlab
cd /data/gitlab
docker-compose up -d
```



# GitLab 数据导入

## 前提

数据导出方和导入方使用的容器中gitlab版本一致（借助镜像tag可以区分），否则不过校验。

## 导出

```
cd /data/gitlab
./cr_full_bk.sh
```

/data/gitlab/data/backups 下会生成一个tar文件，备用

```
cd /data/gitlab/data
tar cvf ssh.tar ssh/ .ssh/
```

得到文件 /data/gitlab/data/ssh.tar，备用

## 导入

目标服务器

```
cd /data/gitlab/data
mv ssh ssh_bak
mv .ssh _ssh_bak
```

"导出"步骤中得到的ssh.tar放到 /data/gitlab/data

```
tar xvf ssh.tar
rm ssh.tar
```

"导出"步骤中的到的/data/gitlab/data/backups/xxxxxx.tar 放到目标服务器的 /data/gitlab/data/backup/ 下，注意保持属主（如使用cp命令时加 -p）

运行导入脚本

```
cd /data/gitlab
./rst_full_bk.sh
```

过程中需要你输入列出的tar文件名， 几个yes等，不详述。



# Docker Registry 安装

在 4 服务器上安装 docker registry

## firewalld 开放端口

```
firewall-cmd --permanent --zone=public --add-port=5000/tcp
firewall-cmd --permanent --zone=public --add-port=443/tcp
firewall-cmd --permanent --zone=public --add-port=80/tcp
firewall-cmd --reload
```

## git clone 脚本[^2]


```
git clone ssh://git@192.168.240.28:2222/research/emregistry-docker-compose.git /data/emregistry
```

## 运行（首次运行会pull镜像）

```
cd /data/emregistr
docker-compose up -d
```



# Docker Registry客户配置和测试

在 1-4 服务器上

```
vi /etc/hosts
```

添加

```
192.168.42.76	emregistry.com
```

 下载信任证书和添加脚本，并执行生效

```
git clone ssh://git@192.168.240.28:2222/research/emregistry-docker-compose.git /data/emregistry
cd /data/emregistry
./add_trust_ca.sh
systemctl restart docker
```

测试登录

```
docker login emregistry.com
Username: emoney
Password:
Login Succeeded
```



[^1]: 这一步是避免在centos7上默认配置安装最新的docker创建容器可能失败的问题，参见 https://stackoverflow.com/questions/42248571/cannt-run-or-build-docker-images-on-centos-7
[^2]: 从测试环境老gitlab
