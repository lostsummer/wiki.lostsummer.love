---
title: "使用 docker 搭建 gitlab 服务器"
date: 2017-05-31 16:45
---

[TOC]

# 0. 背景

快速部署，使用 docker，但数据库要求实体机服务。

# 1. MySQL 安装

参见[CentOS7 64位 上安装 MySql Community 以及配置主备高可用](http://wiki.lostsummer.love/Operation/centos7-64%E4%BD%8D-%E4%B8%8A%E5%AE%89%E8%A3%85-mysql-community-%E4%BB%A5%E5%8F%8A%E9%85%8D%E7%BD%AE%E4%B8%BB%E5%A4%87%E9%AB%98%E5%8F%AF%E7%94%A8.html)

# 2. 创建数据库和用户

```
CREATE USER 'gitlab'@'%' IDENTIFIED BY 'password';
CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;
GRANT ALL PRIVILEGES ON `gitlabhq_production`.* TO 'gitlab'@'%';
```

# 3. 安装 docker, docker-compose

外网，yum 确认联通，以下命令安装最新的社区版
```
curl -sSL https://get.docker.com/ | sh
```

通过 pip 安装 docker 编排工具 docker-compose
```
pip install -U docker-compose
```

# 4. docker-compose 编排

```
mkdir gitlab
cd gitlab
vi docker-compose.yml
```

```yml
version: "3"
services:
  gitlab:
    image: sameersbn/gitlab
    ports:
      - "2222:2222"
      - "80:80"
    depends_on:
      - redis
    environment:
      - DEBUG=false
 
      - DB_ADAPTER=mysql2
      - DB_HOST=192.168.240.28
      - DB_USER=gitlab
      - DB_PASS=password
      - DB_NAME=gitlabhq_production
 
      - REDIS_HOST=redis
      - REDIS_PORT=6379
 
      - GITLAB_HOST=192.168.240.28
      - GITLAB_PORT=80
      - GITLAB_SSH_PORT=2222
      - GITLAB_SECRETS_DB_KEY_BASE=long-and-random-alpha-numeric-string
      - GITLAB_SECRETS_SECRET_KEY_BASE=long-and-random-alphanumeric-string
      - GITLAB_SECRETS_OTP_KEY_BASE=long-and-random-alphanumeric-string
 
    restart: always
 
  redis:
    image: sameersbn/redis
    restart: always
```

启动
```
docker-compose up -d
```

