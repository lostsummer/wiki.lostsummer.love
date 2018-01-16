---
title: "Gitlab灾难恢复"
date: 2018-01-16 09:42
---

# Gitlab 灾难恢复

## Docker数据卷损坏

git.emoney.cn 容器的运行环境在 192.168.42.73:/data/gitlab
其中 /data/gitlab/data 为gitlab容器数据卷
如果此中文件遭到破坏，造成gitlab容器无法正确运行，请按以下步骤操作

1. 确定数据库连接正常

    - DB_HOST=192.168.42.73
    - DB_USER=gitlab
    - DB_PASS=empassword
    - DB_NAME=gitlabhq_production

2. 从备份服务器得到备份文件，恢复最近正常的gitlab目录

    ```
    cd /data
    mv gitlab gitlab_bak
    scp root@192.168.42.74:/data/backup/archive/gitlab_2018-01-13.tgz .      # 具体时间视当时环境而定
    tar xvzf gitlab_2018-01-13.tgz
    ```

3. 重启容器组

    ```
    cd gitlab
    docker-compose rm -f             # 删除之前有问题的容器
    docker-compose up -d             # 启动
    ```

## MySQL 数据库损坏

如果数据库损坏并无法恢复数据，可从gitlab自身每日备份的tar包中复原，按以下步骤操作


1. 重新安装数据库

2. MySQL数据库中创建用户及库

    ```
    CREATE USER 'gitlab'@'%' IDENTIFIED BY 'password';
    CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;
    GRANT ALL PRIVILEGES ON `gitlabhq_production`.* TO 'gitlab'@'%';
    ```

3. 停止容器，如果数据库连接参数有变，编辑容器编排配置

    ```
    cd /data/gitlab
    docker-compose stop
    vim docker-compose.yml
    ```

4. 运行数据恢复脚本

    ```
    ./rst_full_bk.sh
    ...               # 按提示选择唯一的备份文件
    ```

5. 再次启动容器

    ```
    docker-compose up -d
    ```
