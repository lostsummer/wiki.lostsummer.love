---
title: "Debian 配置软阵列（以 SSD x 2 raid0 为例）"
date: 2017-04-25 17:08
---



## 准备
__1. 安装软阵列管理工具 mdadm__


    apt-get install mdadm


__2. 给两块SSD 建立分区__


    fdisk /dev/sdb
    fdisk /dev/sdc
    # 具体交互略，全盘分配sdb1 和 sdc1，确保容量相同 ，不必格式化


## 建立
    mdadm -C /dev/md0 -l 0 -n 2 -c 128 /dev/sdb1 /dev/sdc1
    # 等同:  mdadm --create /dev/md0 --level=0 --raid-devices=2 --chunck=128 /dev/sdb1 /dev/sdc1


##  开关
__1. 启用__


    mdadm -A -s /dev/md0
__2. 停用__


    mdadm --stop -s 


## 格式化


    mkfs -t ext4 /dev/md0


## 运行状态查看
 查看所有MD设备信息
     # 直接查看 /proc/mdstat  
     cat /proc/mdstat  


查看指定MD设备详细运行信息


     mdadm --detail /dev/md0


##  自动挂载配置
编辑 /etc/fstab ， 添加一行


    /dev/md0                /home/oracle/ssd                auto    defaults        0 0


## mdadm  配置文件
 mdadm 不采用 /etc/mdadm.conf 作为主要配置文件，它可以完全不依赖该文件而不会影响阵列的正常工作。
  该配置文件的主要作用是方便跟踪软 RAID 的配置。对该配置文件进行配置是有好处的，但不是必须的。推荐对该文件进行配置。


格式：


    DEVICE  参与阵列的设备
    ARRAY  阵列的描述


  通常可以这样来建立：


    echo DEVICE /dev/sd{b,c,d}1 >> /etc/mdadm.conf
    mdadm --detail --scan >> /etc/mdadm.conf


## 参考资料


http://www.ibm.com/developerworks/cn/linux/l-cn-raid/
