---
title: "SQL Server 2008 审计功能配置及使用"
date: 2017-04-25 16:02
---

 
## 详细说明
  
  
### 概述
 
创建数据库审计的步骤主要包括：__创建审计对象__，__创建审计规范并绑定审计对象__，__查看审计事件__
 
审计级别：__服务器级别__，__数据库级别__
 
### 审计对象
 
步骤一：创建审计对象，审计对象是跟保存路径关联的，所以如果你需要把审计操作日志保存到不同的路径就需要创建不同的审计对象
 
我们把审计操作日志保存在文件系统里，在创建之前我们还要在相关路径先创建好保存的文件夹，我们在D盘先创建sqlaudits文件夹，然后执行下面语句
 
    --创建审计对象之前需要切换到master数据库
    USE [master]
    GO
    CREATE SERVER AUDIT MyFileAudit TO FILE(
    FILEPATH='D:\sqlaudits',
    MAXSIZE=4GB,
    MAX_ROLLOVER_FILES=6)
    WITH (
    ON_FAILURE=CONTINUE,
    QUEUE_DELAY=1000);
 
    ALTER SERVER AUDIT MyFileAudit WITH(STATE =ON)
 
 
把日志放在磁盘的好处是可以使用新增的TVF:sys.[fn\_get_audit\_file] 来过滤和排序审计数据，如果把审计数据保存在 Windows 事件日志里查询起来非常麻烦。
 
我们在创建审计对象的同时可以指定审计选项
 
__MAXSIZE__：指明每个审计日志文件的最大大小是4GB
 
__MAX\_ROLLOVER\_FILES__：指明滚动文件数目，类似于SQL ERRORLOG，达到多少个文件之后删除前面的历史文件，这里是6个文件
 
__ON\_FAILURE__：指明当审计数据发生错误时的操作，这里是继续进行审计，如果指定shutdown，那么将会shutdown整个实例
 
__queue\_delay__：指明审计数据写入的延迟时间，这里是1秒，最小值也是1秒，如果指定0表示是实时写入，当然性能也有一些影响
 
__STATE__：指明启动审计功能，STATE这个选项不能跟其他选项共用，所以只能单独一句
 
在修改审计选项的时候，需要先禁用审计，再开启审计
 
    ALTER SERVER AUDIT MyFileAudit WITH(STATE =OFF)
    ALTER SERVER AUDIT MyFileAudit WITH(QUEUE_DELAY =1000)
    ALTER SERVER AUDIT MyFileAudit WITH(STATE =ON)
 
### 审计规范
 
在SQLSERVER审计里面有审计规范的概念，一个审计对象只能绑定一个审计规范，而一个审计规范可以绑定到多个审计对象
 
    CREATE SERVER AUDIT SPECIFICATION CaptureLoginsToFile
    FOR SERVER AUDIT MyFileAudit
    ADD (failed_login_group),
    ADD (successful_login_group)
    WITH (STATE=ON)
    GO
 
    CREATE SERVER AUDIT MyAppAudit TO APPLICATION_LOG
    GO
    ALTER SERVER AUDIT MyAppAudit WITH(STATE =ON)
    ALTER SERVER AUDIT SPECIFICATION CaptureLoginsToFile WITH (STATE=OFF)
    GO
 
    ALTER SERVER AUDIT SPECIFICATION CaptureLoginsToFile
    FOR SERVER AUDIT MyAppAudit
    ADD (failed_login_group),
    ADD (successful_login_group)
    WITH (STATE=ON)
    GO
 
我们创建一个服务器级别的审计规范CaptureLoginsToFile，然后再创建多一个审计对象MyAppAudit ，这个审计对象会把审计日志保存到Windows事件日志的应用程序日志里
 
我们禁用审计规范CaptureLoginsToFile，修改审计规范CaptureLoginsToFile属于审计对象MyAppAudit ，修改成功
 
这里要说一下 ：审计对象和审计规范的修改 ，无论是审计对象还是审计规范，在修改他们的相关参数之前，他必须要先禁用，后修改，再启用
 
 
    --禁用审计对象
    ALTER SERVER AUDIT MyFileAudit WITH(STATE =OFF)
    --禁用服务器级审计规范
    ALTER SERVER AUDIT SPECIFICATION CaptureLoginsToFile WITH (STATE=OFF)
    GO
    --禁用数据库级审计规范
    ALTER DATABASE AUDIT SPECIFICATION CaptureDBLoginsToFile WITH (STATE=OFF)
    GO
 
    --相关修改选项操作
 
 
    --启用审计对象
    ALTER SERVER AUDIT MyFileAudit WITH(STATE =ON)
    --启用服务器级审计规范
    ALTER SERVER AUDIT SPECIFICATION CaptureLoginsToFile WITH (STATE=ON)
    GO
    --启用数据库级审计规范
    ALTER DATABASE AUDIT SPECIFICATION CaptureDBLoginsToFile WITH (STATE=ON)
 
### 审计服务器级别事件
 
审计服务级别事件，我们一般用得最多的就是审计登录失败的事件，下面的脚本就是审计登录成功事件和登录失败事件
 
    CREATE SERVER AUDIT SPECIFICATION CaptureLoginsToFile
    FOR SERVER AUDIT MyFileAudit
    ADD (failed_login_group),
    ADD (successful_login_group)
    WITH (STATE=ON)
    GO
 
修改审计规范
 
    --跟审计对象一样，更改审计规范时必须将其禁用
    ALTER SERVER AUDIT SPECIFICATION CaptureLoginsToFile WITH (STATE =OFF)
    ALTER SERVER AUDIT SPECIFICATION CaptureLoginsToFile
    ADD (login_change_password_gourp),
    DROP (successful_login_group)
    ALTER SERVER AUDIT SPECIFICATION CaptureLoginsToFile WITH (STATE =ON)
    GO
 
### 审计操作组
 
每个审计操作组对应一种操作，在SQLSERVER2008里一共有35个操作组，包括备份和还原操作，数据库所有权的更改，从服务器和数据库角色中添加或删除登录用户
添加审计操作组的只需在审计规范里使用ADD，下面语句添加了登录用户修改密码操作的操作组
 
    ADD (login_change_password_gourp)
 
### 审计数据库级别事件
 
数据库审计规范存在于他们的数据库中，不能审计tempdb中的数据库操作
 
CREATE DATABASE AUDIT SPECIFICATION和ALTER DATABASE AUDIT SPECIFICATION
 
工作方式跟服务器审计规范一样
 
在SQLSERVER2008里一共有15个数据库级别的操作组
7个数据库级别的审计操作是：select ,insert,update,delete,execute,receive,references
 
相关脚本如下：
 
    --创建审计对象
    USE [master]
    GO
    CREATE SERVER AUDIT MyDBFileAudit TO FILE(FILEPATH='D:\sqldbaudits')
    GO
    ALTER  SERVER AUDIT  MyDBFileAudit WITH (STATE=ON)
    GO
 
    --创建数据库级别审计规范
    USE [sss]
    GO
    CREATE DATABASE AUDIT SPECIFICATION CaptureDBActionToEventLog
    FOR SERVER AUDIT MyDBFileAudit
    ADD (database_object_change_group),
    ADD (SELECT ,INSERT,UPDATE,DELETE ON schema::dbo   BY PUBLIC)
    WITH (STATE =ON)
 
我们先在D盘创建sqldbaudits文件夹
 
第一个操作组对数据库中所有对象的DDL语句create，alter，drop等进行记录
第二个语句监视由任何public用户（也就是所有用户）对dbo架构的任何对象所做的DML操作
创建完毕之后可以在 SQL Server Management Studio 里看到相关的审计。
 
### 查看审计事件
 
被记录到文件系统的审计文件不是存储在可以利用记事本打开的文本文件中，而是采用二进制文件的方式。
当磁盘空间不足的时候是可以直接删除这些SQLAUDIT文件。
 
对于不需要归档审计数据的情况，我比较喜欢这种方式，当磁盘容量不够的时候把最老的那个审计文件删除掉
 
当然，你可以把整个sqlaudits文件夹或某个sqlaudit文件进行备份，放到备份磁盘上，然后删除一些较老的sqlaudit文件
 
备份了之后以后就有机会对之前的审计数据进行翻查，都比较灵活
 
我们有两种方法查看审计日志：
 
方法一：对象资源管理器-》安全性-》审计-》选中某个审计对象-》右键-》查看审计日志





审计项目包括有：日期、时间戳记、服务器实例名称、操作ID、类类型、序列号、成功或失败、列权限、数据库主体ID、服务器主体名称、
 
服务器主体SID、被执行的（或尝试）的实际语句等等
 
 
 
方法二：使用新的表值函数sys.[fn\_get\_audit_file]()
 
此函数接受一个或多个审计文件的参数（使用通配符模式匹配）
 
并利用另外两个附加参数可以指定要处理的起始文件，以及开始读取审计的已知偏移位置
 
这两个参数都是可选的，但依然必须使用关键字default指定，此函数随后从文件中读取二进制数据，并将格式化这些审计项目
 
 
 
__服务器级别审计__
 
根据最近时间的那个sqlaudit文件，查询这个文件里面的信息
 
 
 
     SELECT  [event_time] AS '触发审计的日期和时间' ,
 
             sequence_number AS '单个审计记录中的记录顺序' ,
 
             action_id AS '操作的 ID' ,
 
             succeeded AS '触发事件的操作是否成功' ,
 
             permission_bitmask AS '权限掩码' ,
 
             is_column_permission AS '是否为列级别权限' ,
 
             session_id AS '发生该事件的会话的 ID' ,
 
             server_principal_id AS '执行操作的登录上下文 ID' ,
 
             database_principal_id AS '执行操作的数据库用户上下文 ID' ,
 
             target_server_principal_id AS '执行 GRANT/DENY/REVOKE 操作的服务器主体' ,
 
             target_database_principal_id AS '执行 GRANT/DENY/REVOKE 操作的数据库主体' ,
 
             object_id AS '发生审计的实体的 ID(服务器对象，DB,数据库对象，架构对象)' ,
 
             class_type AS '可审计实体的类型' ,
 
             session_server_principal_name AS '会话的服务器主体' ,
 
             server_principal_name AS '当前登录名' ,
 
             server_principal_sid AS '当前登录名 SID' ,
 
             database_principal_name AS '当前用户' ,
 
             target_server_principal_name AS '操作的目标登录名' ,
 
             target_server_principal_sid AS '目标登录名的 SID' ,
 
             target_database_principal_name AS '操作的目标用户' ,
 
             server_instance_name AS '审计的服务器实例的名称' ,
 
             database_name AS '发生此操作的数据库上下文' ,
 
             schema_name AS '此操作的架构上下文' ,
 
             object_name AS '审计的实体的名称' ,
 
             statement AS 'TSQL 语句（如果存在）' ,
 
             additional_information AS '单个事件的唯一信息，以 XML 的形式返回' ,
 
             file_name AS '记录来源的审计日志文件的路径和名称' ,
 
             audit_file_offset AS '包含审计记录的文件中的缓冲区偏移量'
 
     FROM    sys.[fn_get_audit_file]('D:\sqlaudits\MyFileAudit_F0BCDC6F-0A89-459D-B345-9DDEB036CC39_0_130595725124220000.sqlaudit',
 
                                     DEFAULT, DEFAULT)
 
     WHERE   [event_time] BETWEEN '2014-11-04 11:02:00'
 
                         AND     '2014-11-04 11:18:00'
 
 
 
__数据库级别审计__
 
先执行下面脚本查询一些数据
 
 
 
    USE [sss]
 
    GO
 
    SELECT * FROM [dbo].[nums]
 
查询
 
 
 
 
    SELECT  [event_time] AS '触发审计的日期和时间' ,
 
            sequence_number AS '单个审计记录中的记录顺序' ,
 
            action_id AS '操作的 ID' ,
 
            succeeded AS '触发事件的操作是否成功' ,
 
            permission_bitmask AS '权限掩码' ,
 
            is_column_permission AS '是否为列级别权限' ,
 
            session_id AS '发生该事件的会话的 ID' ,
 
            server_principal_id AS '执行操作的登录上下文 ID' ,
 
            database_principal_id AS '执行操作的数据库用户上下文 ID' ,
 
            target_server_principal_id AS '执行 GRANT/DENY/REVOKE 操作的服务器主体' ,
 
            target_database_principal_id AS '执行 GRANT/DENY/REVOKE 操作的数据库主体' ,
 
            object_id AS '发生审计的实体的 ID(服务器对象，DB,数据库对象，架构对象)' ,
 
            class_type AS '可审计实体的类型' ,
 
            session_server_principal_name AS '会话的服务器主体' ,
 
            server_principal_name AS '当前登录名' ,
 
            server_principal_sid AS '当前登录名 SID' ,
 
            database_principal_name AS '当前用户' ,
 
            target_server_principal_name AS '操作的目标登录名' ,
 
            target_server_principal_sid AS '目标登录名的 SID' ,
 
            target_database_principal_name AS '操作的目标用户' ,
 
            server_instance_name AS '审计的服务器实例的名称' ,
 
            database_name AS '发生此操作的数据库上下文' ,
 
            schema_name AS '此操作的架构上下文' ,
 
            object_name AS '审计的实体的名称' ,
 
            statement AS 'TSQL 语句（如果存在）' ,
 
            additional_information AS '单个事件的唯一信息，以 XML 的形式返回' ,
 
            file_name AS '记录来源的审计日志文件的路径和名称' ,
 
            audit_file_offset AS '包含审计记录的文件中的缓冲区偏移量'
 
    FROM    sys.[fn_get_audit_file]('D:\sqldbaudits\MyDBFileAudit_698BA060-CC40-4A3C-B19D-12B370712404_0_130595753193920000.sqlaudit',
 
                                DEFAULT, DEFAULT)
 
 
 
将审计日志保存到文件系统的好处就是可以使用TVP里通过where 和order by对审计数据进行筛选和排序
 
 
 
### 和审计相关的视图
 
    --查询审计相关视图
 
    SELECT * FROM sys.[server_file_audits]
 
    SELECT * FROM sys.[server_audit_specifications]
 
    SELECT * FROM sys.[server_audit_specification_details]
 
    SELECT * FROM sys.[database_audit_specifications]
 
    SELECT * FROM sys.[database_audit_specification_details]
 
    SELECT * FROM sys.[dm_server_audit_status]
 
    SELECT * FROM sys.[dm_audit_actions]
 
    SELECT * FROM sys.[dm_audit_class_type_map]
 
 
 
### 删除相关对象
 
 
    --删除顺序
 
    --删除数据库审计规范
 
    USE [sss]
 
    GO
 
    ALTER DATABASE AUDIT SPECIFICATION [CaptureDBActionToEventLog] WITH (STATE=OFF)
 
    GO
 
    DROP DATABASE AUDIT SPECIFICATION [CaptureDBActionToEventLog]
 
    GO
 
 
 
    --删除服务器审计规范
 
    USE [master]
 
    GO
 
    ALTER SERVER  AUDIT SPECIFICATION [CaptureLoginsToFile] WITH (STATE=OFF)
 
    GO
 
    DROP SERVER AUDIT SPECIFICATION [CaptureLoginsToFile]
 
    GO
 
 
 
    --删除审计对象
 
    ALTER SERVER AUDIT [MyFileAudit] WITH (STATE=OFF)
 
    GO
 
    ALTER SERVER AUDIT [MyAppAudit] WITH (STATE=OFF)
 
    GO
 
    ALTER SERVER AUDIT [MyEventLogAudit] WITH (STATE=OFF)
 
    GO
 
    DROP SERVER AUDIT [MyAppAudit]
 
    GO
 
    DROP SERVER AUDIT [MyFileAudit]
 
    GO
 
    DROP SERVER AUDIT [MyEventLogAudit]
 
    GO  
   
  
  
  
## 总结
 
审计功能最大的好处是：你使用自建审计表来保存审计数据，如果聪明的黑客攻破你的数据库实例，他自然可以把你的那个审计表drop掉，你同样查不出黑客的任何蛛丝马迹，而审计不同，他把审计数据放在SQLSERVER外面，除非你们公司的SA和DBA的安全意识很弱，黑客有机会把磁盘文件删除掉，否则依然有可能查出黑客的蛛丝马迹进行预防！！

















