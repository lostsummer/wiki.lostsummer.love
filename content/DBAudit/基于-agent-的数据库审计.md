---
title: "基于 Agent 的数据库审计"
date: 2017-04-25 16:13
---

#检测大纲分析


##不能达标点


* ###连接监控：


1. 数据库管理软件的补丁信息
2. 数据库管理软件的安全配置信息
3. 数据库的用户及权限设置
4. 非授权连接数据库
5. 连接失败


* ###直接访问审计：


1. 利用数据库自身日志记录的信息
2. 初始化参数的审计（至少）：
3. 数据库自身审计的启用禁用
4. 日志恢复的启用禁用


* ###类型版本审计：


1. 数据库管理软件名称
2. 数据库管理软件版本
3. 创建时间
4. 启动时间
5. 实例名
6. 数据库主目录
7. 服务器操作系统
       
* ###违规行为审计：
1. 违规连接互联网及其他公共信息网络
2. 未经授权卸载、删除、修改数据库服务器上的代理程序


* ###异常行为审计：


1. 开启不需要的新服务
2. 短时间内使用不同的口令多次尝试连接服务器，数据库，远程登录计算机
3. 短时间内计算机被多次扫描网络端口
4. 服务程序满负荷运行较长时间
5. 服务程序访问不需要的计算机资源
6. 程序开放监听端口，自动发送网络数据包，对网络进行ARP     欺骗
7. 运行中的程序被打开、被写内存或被启动远程线程
8. 程序在操作系统中添加计算机账户或修改用户权限


* ###web服务器访问触发的数据库操作行为进行关联（部分要求靠网络审计已达到）：


1. IP地址
2. 系统用户名
3. URL
4. 数据库IP地址
5. SQL
6. 访问发生时间
  
<br />
#初步调研


##典型部署


* 基于日志的审计技术：该技术通常是通过数据库自身功能实现，Oracle、DB2等主流数据库，均具备自身审计功能，通过配置数据库的自审计功能，即可实现对数据库的审计，其典型部署示意图如图2所示：

![](http://ooyi4zkat.bkt.clouddn.com/agent1.jpg)
　


图2 日志审计技术部署示意


* 基于代理的审计技术：该技术是通过在数据库系统上安装相应的审计Agent，在Agent上实现审计策略的配置和日志的采集，常见的产品如Oracle公司的Oracle Audit Vault、IBM公司的DB2 Audit Management Expert Tool以及第三方安全公司提供的产品，其典型部署示意图如图3所示：
该技术与日志审计技术比较类似，最大的不同是需要在被审计主机上安装代理程序。代理审计技术从审计粒度上要优于日志审计技术，但是性能上的损耗是要大于日志审计技术，因为数据库系统厂商未公开细节，由数据库厂商提供的代理审计类产品对自有数据库系统的兼容性较好，但是在跨数据库系统的支持上，比如要同时审计Oracle和DB2时，存在一定的兼容性风险。同时由于在引入代理审计后，原数据库系统的稳定性、可靠性、性能或多或少都会有一些影响，实际的应用面较窄。

![](http://ooyi4zkat.bkt.clouddn.com/agent2.jpg)


##数据库技术依赖分析


###[SQL server](http://www.ciotimes.com/infrastructure/sjk/73776.html)


* ####登录审计


可以记录对服务器失败和成功的登录尝试。登录审计写到错误日志中。

记录连接尝试可以用来获知谁在尝试连接数据库，是否发送了恶意攻击或者尝试攻击是否成功。

登录审计可以按如下方式配置：


> 1. 在对象资源管理器中右击SQL Server,选择属性。
> 2. 点击安全。
> 3. 设置登录审计属性。可以选择：无审计，只记录失败登录，只记录成功登录，以及失败和成功登录都记录。
> 4. 点击确定然后重启SQL Server.


* ####C2审计


你可以启用C2审计模式来记录访问语句和对象失败和成功的日志。C2审计模式保存了大量数据，所以日志文件很容易变得非常巨大。当文件达到200 MB时，SQL Server会打开一个新文件。如果记录日志的数据目录空间不足了，SQL Server会自动关闭记录功能。您可以按照如下方式启用C2审计模式：

在对象资源管理器中右击SQL Server,选择属性。


>  1. 点击安全，然后选中“启用C2审计跟踪”.
>  2. 点击确定。
>  3. 当然，您也可以使用“sp_configure”来启动C2审计跟踪。


*请注意，虽然C2安全标准在SQL Server 2012中是可用的，但它将被通用遵从准则替代，而且可能将在未来的SQL Server版本中删除掉。*



* #### 通用遵从准则


通用遵从准则是一套审计标准，它仍然使用SQL Server跟踪捕获数据。通用遵从准则选项启用之后，登录审计也会启用。您可以按如下方式启用“通用遵从准则”:

> 待补充


* ####数据定义语言（DDL）触发器


DDL触发器可以用于把DDL信息和SQL Server有关安全的事件日志记录到表中。DDL事件日志记录提供给你审计和特权提升攻击的潜在警告（用户被赋予太多权限或者用户误用了他们的权限）。



在定义触发器时，你可以指定作用域为服务器或者数据库。服务器作用域的触发器对服务器对象事件（比如登录对象）触发执行。数据库作用域的触发器响应数据库对象事件（比如，数据库模式，表和视图）触发。“EventData”函数提供关于DDL或者安全相关事件（引起DDL触发器被触发的事件）的明细信息。“EventData”函数返回类型为xml的值。该模式的差异与事件类型有关。



下面的代码在服务器上创建了一个DDL触发器，将审计并存储在SQL Server上发生的所有连接和安全有关事件，结果会记录到master数据库的“SecurityLog ”表中。



    USE [master]
    GO
    DROP TABLE [dbo].[SecurityLog]
    GO
    CREATE TABLE [dbo].[SecurityLog]（
    [EventType] [nvarchar]（128） NULL,
    [EventTime] [datetime] NULL,
    [EventLog] [xml] NULL
    ） ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]
    GO


现在，创建一个DDL触发器（ddl_trig_capture_security_events）来捕获并存储所有连接和安全有关事件：


    USE [master]
    GO
    IF EXISTS （SELECT * FROM sys.Server_triggers
    WHERE name = 'ddl_trig_capture_security_events'）
    DROP TRIGGER ddl_trig_capture_security_events
    ON ALL Server;
    GO
    CREATE TRIGGER ddl_trig_capture_security_events
    ON ALL Server;
    FOR LOGON, DDL_Server_SECURITY_EVENTS,
    DDL_DATABASE_SECURITY_EVENTS
    AS
    INSERT INTO [master][SecurityLog] （EventType, EventTime, EventLog）
    SELECT EVENTDATA（）。value（’（/EVENT_INSTANCE/EventType）[1]','nvarchar（128）‘）
    ,EVENTDATA（）。value（’（/EVENT_INSTANCE/PostTime）[1]','datetime‘）
    ,EVENTDATA（）
    GO
    Once the trigger has been created, you can test to see if it is working:
    USE [master]
    GO
    CREATE LOGIN [TestDDL] WITH PASSWORD=N'TestDDL'
    GO
    USE [AdventureWorks2012]
    GO
    CREATE USER [TestDDL] FOR LOGIN [TestDDL]
    GO
    ALTER ROLE [db_datareader] ADD MEMBER [TestDDL]
    GO
    GRANT EXECUTE ON [dbo].[uspGetBillOfMaterials] TO [TestDDL]
    GO


点击xml查看所有事件日志：

![](http://ooyi4zkat.bkt.clouddn.com/agent2-1.jpg)



下图展示了选择查看日志表时展示的结果：

![](http://ooyi4zkat.bkt.clouddn.com/agent2-2.jpg)







* ####事件通知 


事件通知是在SQL Server 2005引入的，提供的功能是通过服务代理和队列收集非常具体的SQL跟踪，DDL和安全相关事件的子集。事件通知支持通过服务代理自动异步处理事件（在交易范围之外）。事件通知可以提供DDL触发器和SQL跟踪的编程替代。


在下面的例子中，我们将使用事件通知审计并存储在SQL Server上出现的所有连接和安全相关事件。  



    USE [msdb]
    GO
    --Creating queue
    CREATE QUEUE [SecurityEventsQueue]
    GO
    --Creating service for the queue
    CREATE SERVICE [//AdventureWorks.com/SecurityEventsService]
    AUTHORIZATION [dbo]
    ON QUEUE [dbo].[SecurityEventsQueue]
    （[http://schemas.microsoft.com/SQL/Notifications/PostEventNotification]）
    GO
    --Creating route for the service
    CREATE ROUTE SecurityEventsRoute
    WITH SERVICE_NAME = '//AdventureWorks.com/SecurityEventsService',
    ADDRESS = 'LOCAL';
    GO
    --Creating Event Notification to capture connection and secrity-related events
    USE [msdb]
    GO
    CREATE EVENT NOTIFICATION NotifySecurityEvents
    ON Server
    FOR AUDIT_LOGIN,
    AUDIT_LOGOUT,
    AUDIT_LOGIN_FAILED,
    DDL_Server_SECURITY_EVENTS,
    DDL_DATABASE_SECURITY_EVENTS
    TO SERVICE '//AdventureWorks.com/SecurityEventsService' ,
    '9D584F73-1796-4494-ADC2-04BDD729FBCE';
    GO
    --Creating the service program that will process the event messages that is
    --generated via Event Notification objects
    IF EXISTS （SELECT * FROM [sys].[objects] WHERE [name] = 'sProcessSecurityEvents'）
    DROP PROCEDURE [dbo].[sProcessSecurityEvents]
    GO
    CREATE PROC [dbo].[sProcessSecurityEvents]
    AS BEGIN
    SET NOCOUNT ON
    SET CONCAT_NULL_YIELDS_NULL ON
    SET ANSI_PADDING ON
    SET ANSI_WARNINGS ON
    BEGIN TRY
    DECLARE @message_body [xml]
    ,@EventTime [datetime]
    ,@EventType [varchar]（128）
    ,@message_type_name [nvarchar]（256）
    ,@dialog [uniqueidentifier]
    -- Endless loop
    WHILE （1 = 1）
    BEGIN
    BEGIN TRANSACTION ;
    -- Receive the next available message
    WAITFOR （RECEIVE TOP（1）
    @message_type_name = [message_type_name],
    @message_body = [message_body],
    @dialog = [conversation_handle]
    FROM [dbo].[SecurityEventsQueue]）， TIMEOUT 2000
    -- Rollback and exit if no messages were found
    IF （@@ROWCOUNT = 0）
    BEGIN
    ROLLBACK TRANSACTION;
    BREAK;
    END;
    -- End conversation of end dialog message
    IF （@message_type_name = 'http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog'）
    BEGIN
    PRINT 'End Dialog received for dialog # ' + CAST（@dialog as [nvarchar]（40））；
    END CONVERSATION @dialog;
    END;
    ELSE
    BEGIN
    SET @EventTime = CAST（CAST（@message_body.query（'/EVENT_INSTANCE/PostTime/text（）'） AS [nvarchar]（max）） AS [datetime]）
    SET @EventType = CAST（@message_body.query（'/EVENT_INSTANCE/EventType/text（）'） AS [nvarchar]（128））
    INSERT INTO [master][SecurityLog] （[EventType], [EventTime], [EventLog]）
    VALUES （@EventType, @EventTime, @message_body）
    END
    COMMIT TRANSACTION
    END --End of loop
    END TRY
    BEGIN CATCH
    SELECT ERROR_NUMBER（）
    ,ERROR_SEVERITY（）
    ,ERROR_STATE（）
    ,ERROR_PROCEDURE（）
    ,ERROR_LINE（）
    ,ERROR_MESSAGE（）
    END CATCH
    END
    GO
    --Once service program is created successfully, execute the following script to
    --activate our service broker queue and reference this Service Program stored procedure:
    ALTER QUEUE [dbo].[SecurityEventsQueue]
    WITH STATUS = ON
    ,ACTIVATION （PROCEDURE_NAME = [sProcessSecurityEvents]
    ,STATUS = ON
    ,MAX_QUEUE_READERS = 1
    ,EXECUTE AS OWNER）
    GO


事件通知是在SQL Server 2005引入的，提供的功能是通过服务代理和队列收集非常具体的SQL跟踪，DDL和安全相关事件的子集。事件通知支持通过服务代理自动异步处理事件（在交易范围之外）。事件通知可以提供DDL触发器和SQL跟踪的编程替代。



* ####SQL Server 审计


我们还可以使用SQL Server审计跟踪安全相关事件。SQL Server审计是SQL Server企业版功能，自SQL Server 2008之后才有的。SQL Server审计使用扩展事件帮助执行审计。
例如：我们将创建针对所有安全相关事件的审计。要实现这个目的，我们首先要创建基于文件的服务器审计规范，我们将把它用作我们的服务器审计规范。



    USE [master]
    GO
    CREATE Server AUDIT [Audit-SecurityEvents]
    TO FILE
    （ FILEPATH = N'D:\Demo_SQLAudit'
    ,MAXSIZE = 200 MB
    ,MAX_ROLLOVER_FILES = 2147483647
    ,RESERVE_DISK_SPACE = OFF ）
    WITH
    （ QUEUE_DELAY = 1000
    ,ON_FAILURE = CONTINUE ）
    GO


我们现在将创建审计规范来审计服务器和数据库安全相关事件，比如登录密码修改，数据库用户或者对象权限修改，失败的登录尝试和服务器策略改变：



    CREATE Server AUDIT SPECIFICATION [ServerAuditSpecification]
    FOR Server AUDIT [Audit-SecurityEvents]
    ADD （LOGIN_CHANGE_PASSWORD_GROUP），
    ADD （DATABASE_PRINCIPAL_CHANGE_GROUP），
    ADD （DATABASE_OBJECT_PERMISSION_CHANGE_GROUP），
    ADD （FAILED_LOGIN_GROUP），
    ADD （Server_PRINCIPAL_CHANGE_GROUP）
    GO




要测试你的审计规范，你可以产生一些事件，然后查看审计文件，以确认这些事件已经被审计到了。请看下图：

![](http://ooyi4zkat.bkt.clouddn.com/agent3.jpg)


