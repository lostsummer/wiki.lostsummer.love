---
title: "Swarm集群上部署portainer监控"
date: 2018-01-17 15:19
---

[TOC]

# 准备

## 1. 配置好每个节点主机名

初装系统没有设置hostname的话，默认都是localhost.localdomain，
无法辨识在centos7上最快捷的方法的是
hostnamectl --static set-hostname HOSTNAME 
假定四个节点主机名为 WG-DOCKER1, WG-DOCKER2, WG-DOCKER3, WG-DOCKER4


## 2. 每个节点安装docker

略


## 3. 暴露每个节点的Remote API

```
vim /usr/lib/systemd/system/docker.service
# 修改
ExecStart=/usr/bin/dockerd -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2375
# 重启服务
systemctl daemon-reload
systemctl restart docker
```


## 4. 四个节点组成swarm

    略


# 部署&运行

## 1. 在管理节点上运行 portainer 容器
```
docker pull portainer/portainer
docker service create \
--name portainer \
--publish 9000:9000 \
--constraint 'node.role == manager' \
--mount type=bind,src=//var/run/docker.sock,dst=/var/run/docker.sock \
--constraint node.hostname==WG-DOCKER1 \
--container-label com.docker.stack.namespace=portainer \
portainer/portainer \
-H unix:///var/run/docker.sock
```
注:
--constraint node.hostname==WG-DOCKER1 指定容器运行在管理节点WG-DOCKER1上.
--container-label com.docker.stack.namespace=portainer 是为了prometheus+grafana能通过stack标签查看监控数据，建议今后不以stack方式部署的服务都加上com.docker.stack.namespace标签。


## 2. 管理员首次登录

浏览器上访问 http://节点IP:9000/ 打开portainer面板，首次设置管理员密码


## 3. 配置端点

面板左侧Endpoints选项下添加其他节点，EndpointURL 使用IP:2375

