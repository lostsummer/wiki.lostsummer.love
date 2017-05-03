---
title: "Oracle 数据库审计配置及使用"
date: 2017-04-25 15:54
---

## 详细说明

### 概述

__审计类型__：语句审计、权限审计、模式对象审计、细粒度的审计  

* 语句审计：按照语句类型审计SQL语句，而不论访问何种特定的模式对象。也可以在数据库中指定一个或多个用户，针对特定的语句审计这些用户  
* 权限审计：审计系统权限，例如CREATE TABLE或ALTER INDEX。和语句审计一样，权限审计可以指定一个或多个特定的用户作为审计的目标  
* 模式对象审计：审计特定模式对象上运行的特定语句(例如，DEPARTMENTS表上的UPDATE语句)。模式对象审计总是应用于数据库中的所有用户  
* 细粒度的审计：根据访问对象的内容来审计表访问和权限。使用程序包DBMS_FGA来建立特定表上的策略 

### 开启审计  

管理员权限登录查看

    sqlplus / as sysdba

和审计相关的几个参数

    SQL>show parameter audit
    audit_file_dest
    audit_sys_operations
    audit_trail

__audit\_sys\_operations__：  
默认为false，当设置为true时，所有sys用户（包括以sysdba,sysoper身份登录的用户）的操作都会被记录，audit trail不会写在 aud\$ 表中，这个很好理解，如果数据库还未启动  aud\$ 不可用，那么像conn /as sysdba这样的连接信息，只能记录在其它地方。如果是windows平台，audti trail会记录在windows的事件管理中，如果是linux/unix平台则会记录在audit_file\_dest参数指定的文件中。

__audit\_trail__：  
None：是默认值，不做审计；
DB：将audit trail 记录在数据库的审计相关表中，如aud$，审计的结果只有连接信息；
DB,Extended：这样审计结果里面除了连接信息还包含了当时执行的具体语句；
OS：将audit trail 记录在操作系统文件中，文件名由audit_file_dest参数指定；

__audit\_file\_dest__：Audit_trail=OS时 文件位置

__注__：参数AUDIT\_TRAIL不是动态的，为了使AUDIT\_TRAIL参数中的改动生效，必须关闭数据库并重新启动。在对 SYS.AUD\$ 表进行审计时，应该注意监控该表的大小，以避免影响SYS表空间中其他对象的空间需求。推荐周期性归档 SYS.AUD\$ 中的行，并且截取该表。Oracle提供了角色 DELETE\_CATALOG_ROLE，和批处理作业中的特殊账户一起使用，用于归档和截取审计表。