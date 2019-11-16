![CLEVER DATA GIT REPO](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/0-clever-data-github.png "李聪明的数据库")

# 使用一个SQL脚本创建SQL作业警报
#### Create SQL Job Alert Notification With One SQL Script
**TIME STAMP**

## Contents

- [中文](#中文)
- [English](#English)
- [SQL Logic](#Logic)
- [Build Info](#Build-Info)
- [Author](#Author)
- [License](#License) 


## 中文
下面是创建一个使用HTML和CSS格式化的SQL作业警报的SQL逻辑（logic）。
这个脚本做什么用的呢？ 它会创建一个名为“SQL Job Alerts”的作业。 如果它已经存在，它将删除已经存在的作业，并重新创建一个。
以下是作业运行后将会采取的行动步骤

1.在MSDB..SysJobHistory表上创建“after insert”触发器。
关于触发器（用作主要的警报程序）
a.	检查插入到MSDB..SysJobHistory表中的任何新添加的“failed”消息
b.	如果检测到“failed”消息,会立即运行Job“SQL Job Alerts”。
2. 使用名为“SQLDatabaseMailProfile”的通用配置文件配置SQL数据库邮件，该配置文件应该适用于大多数SQL环境。
3. 配置后，将测试通知发送到通讯组。 （将你的DBA的放在通讯组中）


首先要做的是做什么？
1.	成为数据库管理员（DBA）。 拿到必要的系统管理员权限来进行这个工作。
2.	创建名为“SQLJobAlerts@MyDomain.com”的通讯组。你的Exchange管理员可以对其进行设置。
注意：确保Exchange管理员给通讯组配置以下设置，否则它将不会发送消息。
a.	使通讯组能够接受到外部发件人发来的邮件。
b.	禁用要求对所有发件人进行身份验证的配置。
3.	把自己和其他数据库管理员(DBA)放进那个组。
4.	将你的SMTP服务器名称添加到SQL脚本。
查找SMTP服务器并用这个名称替换SMTP服务器名称：MySMTPServerName.MyDomain.com
查找分发组并用此替换：SQLJobAlerts@MyDomain.com
这段逻辑（logic）有一个好处是：一旦你插入了SMTP和分发组信息，就可以在任何SQL数据库服务器上运行该脚本，并且它将自动配置你的所有作业通知。我将其用作我的SQL Server构建的一部分。
在你的SQL Server上运行此段脚本。


## English
Here is some SQL logic that will create an SQL Job Alert formatted with HTML and CSS.
What does this script do? It creates a Job called “SQL Job Alerts”. If it already exists it will drop the Job, and recreate. 
These are the actions the Job will take once it’s run.

1. Creates an ‘after insert’ trigger on the MSDB..SysJobHistory table.
About the trigger (serves as the primary alerting process)
a. Checks for any newly added “failed” message that is inserted into the MSDB..SysJobHistory table.
b. If a “failed” message is detected; it immediately runs the Job ‘SQL Job Alerts’.
2. Configures SQL Database Mail with a generic profile called “SQLDatabaseMailProfile” that should work in most SQL environments.
3. Sends a test notification to a distribution group after it’s configured. (Put your DBA’s in the distribution group)

What do I need to do first to make this work?
1. Be the DBA. Have the necessary sysadmin rights on the box to make this work.
2. Create a distribution group called “SQLJobAlerts@MyDomain.com” Your Exchange Admin can set this up.
Note: Make sure the exchange admin configures the following settings for Distribution Groups otherwise it won’t send out messages.
a. Enable the distribution group to receive mail from external senders.
b. Disable the configuration that requires all senders to be authenticated.
3. Put yourself and the other DBA’s into that group.
4. Add your SMTP Server Name to the SQL script.
Find and Replace this for SMTP Server Name: MySMTPServerName.MyDomain.com
Find and Replace this for Distribution Group: SQLJobAlerts@MyDomain.com
The good thing about this logic is once you have SMTP and Distribution Group information plugged in… You can run the script on any SQL Database server, and you’ll have all your Job notifications configured automatically. I use this as part of my SQL Server builds.
Run the script on your SQL Server.

---
## Logic
```SQL
USE [msdb]
GO
 
declare @jobid      binary(16)
select  @jobid      = job_id from msdb.dbo.sysjobs where (name = N'SEND SQL JOB ALERTS')
if  (@jobid is not null)
            begin
              exec msdb.dbo.sp_delete_job @jobid
            end
go
 
BEGIN TRANSACTION
DECLARE @ReturnCode INT
SELECT @ReturnCode = 0
 
IF NOT EXISTS (SELECT name FROM msdb.dbo.syscategories WHERE name=N'[Uncategorized (Local)]' AND category_class=1)
BEGIN
EXEC @ReturnCode = msdb.dbo.sp_add_category @class=N'JOB', @type=N'LOCAL', @name=N'[Uncategorized (Local)]'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
 
END
 
DECLARE @jobId BINARY(16)
EXEC @ReturnCode =  msdb.dbo.sp_add_job @job_name=N'SEND SQL JOB ALERTS', 
        @enabled=1, 
        @notify_level_eventlog=0, 
        @notify_level_email=0, 
        @notify_level_netsend=0, 
        @notify_level_page=0, 
        @delete_level=0, 
        @description=N'This is a new SQL Job notification.  It''s designed to send the a short error report about step failure within a job.  NOTE:  This job is dependent upon a custom trigger on the MSDB..[SysJobHistory] table.', 
        @category_name=N'[Uncategorized (Local)]', 
        @owner_login_name=N'sa', @job_id = @jobId OUTPUT
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
 
EXEC @ReturnCode = msdb.dbo.sp_add_jobstep @job_id=@jobId, @step_name=N'Send email about step failure', 
        @step_id=1, 
        @cmdexec_success_code=0, 
        @on_success_action=1, 
        @on_success_step_id=0, 
        @on_fail_action=2, 
        @on_fail_step_id=0, 
        @retry_attempts=0, 
        @retry_interval=0, 
        @os_run_priority=0, @subsystem=N'TSQL', 
        @command=N'use msdb;
set nocount on
set ansi_nulls on
set quoted_identifier on
 
----------------------------------------------------------------------
--create trigger on msdb..sysjobhistory table
--在msdb..sysjobhistory表格上创建触发器
declare @create_trigger_check_for_job_failure       varchar(max)
set @create_trigger_check_for_job_failure       =
''
create trigger [dbo].[trig_check_for_job_failure] on  [dbo].[sysjobhistory]
after insert
as 
    begin
    set nocount on
    declare @is_fail    int
    set @is_fail    = (select case when [message] like ''''%The step failed%'''' then 1 else 0 end from msdb..sysjobhistory where instance_id in (select max(instance_id) from [msdb]..[sysjobhistory]))
    if  @is_fail    = 1
    begin
        exec msdb.dbo.sp_start_job @job_name = ''''SEND SQL JOB ALERTS''''
    end
end
''
 
if exists(select [name] from sys.objects where type = ''tr'' and [name] = ''trig_check_for_job_failure'')
    drop trigger [trig_check_for_job_failure]
 
exec (@create_trigger_check_for_job_failure)
 
 
----------------------------------------------------------------------
-- 	Configure SQL Database Mail if it''s not already configured.
-- 	配置SQL数据库邮件，如果它还尚未配置的话
if (select top 1 name from msdb..sysmail_profile) is null
    begin
        ----------------------------------------------------------------------
-- 	Enable SQL Database Mail
-- 	启用SQL数据库邮件
        exec master..sp_configure ''show advanced options'',1
        reconfigure;
        exec master..sp_configure ''database mail xps'',1
        reconfigure;
 
        ----------------------------------------------------------------------
-- 	Add a profile
-- 	添加配置文件
        execute msdb.dbo.sysmail_add_profile_sp
            @profile_name       = ''SQLDatabaseMailProfile''
        ,   @description        = ''SQLDatabaseMail'';
 
        ----------------------------------------------------------------------
-- 	Add the account names you want to appear in the email message.
-- 	添加你想在邮件信息中显示的账户名
        execute msdb.dbo.sysmail_add_account_sp
            @account_name       = ''sqldatabasemail@MyDomain.com''
        ,   @email_address      = ''sqldatabasemail@MyDomain.com''
        ,   @mailserver_name    = ''MySMTPServerName.MyDomain.com''  
        --, @port           = ####  --optional
        --, @enable_ssl     = 1     --optional
        --, @username       =''MySQLDatabaseMailProfile'' --optional
        --, @password       =''MyPassword''               --optional
 
-- 	Adding the account to the profile
-- 	添加账户到配置文件
        execute msdb.dbo.sysmail_add_profileaccount_sp
            @profile_name       = ''SQLDatabaseMailProfile''
        ,   @account_name       = ''sqldatabasemail@MyDomain.com''
        ,   @sequence_number    = 1;
 
-- 	Give access to new database mail profile (DatabaseMailUserRole)
-- 	授予对新数据库邮件配置文件(DatabaseMailUserRole)的访问
        execute msdb.dbo.sysmail_add_principalprofile_sp
            @profile_name       = ''SQLDatabaseMailProfile''
        ,   @principal_id       = 0
        ,   @is_default     = 1;
 
        ----------------------------------------------------------------------
-- 	Get Server info for test message
-- 	获取服务器信息进行测试
-- 	 
        declare @get_basic_server_name              varchar(255)
        declare @get_basic_server_name_and_instance_name    varchar(255)
        declare @basic_test_subject_message         varchar(255)
        declare @basic_test_body_message            varchar(max)
        set     @get_basic_server_name          = (select cast(serverproperty(''servername'') as varchar(255)))
        set     @get_basic_server_name_and_instance_name= (select  replace(cast(serverproperty(''servername'') as varchar(255)), ''\'', ''   SQL Instance: ''))
        set     @basic_test_subject_message     = ''Test SMTP email from SQL Server: '' + @get_basic_server_name_and_instance_name
        set     @basic_test_body_message        = ''This is a test SMTP email from SQL Server:  '' + @get_basic_server_name_and_instance_name + char(10) + char(10) + ''If you see this.  It''''s working perfectly :)''
 
        ----------------------------------------------------------------------
-- 	Send quick email to confirm email is properly working.
-- 	发送快速邮件以确认邮件是否正常工作
 
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name       = ''SQLDatabaseMailProfile''
        ,   @recipients     = ''SQLJobAlerts@MyDomain.com''
        ,   @subject        = @basic_test_subject_message
        ,   @body           = @basic_test_body_message;
 
         
         
    end
 
----------------------------------------------------------------------
-- 	get basic server info.
-- 	获取基本的服务器信息
 
declare @server_name_basic      varchar(255)
declare @server_name_instance_name  varchar(255)
declare @server_time_zone       varchar(255)
set @server_name_basic      = (select cast(serverproperty(''servername'') as varchar(255)))
set @server_name_instance_name  = (select  replace(cast(serverproperty(''servername'') as varchar(255)), ''\'', ''   SQL Instance: ''))
exec    master.dbo.xp_regread ''hkey_local_machine'', ''system\currentcontrolset\control\timezoneinformation'',''timezonekeyname'', @server_time_zone out
 
----------------------------------------------------------------------
-- 	set message subject.
-- 	设置短信对象
declare @message_subject        varchar(255)
set @message_subject        = ''SQL Job failure found on Server:  '' + @server_name_instance_name
 
----------------------------------------------------------------------
-- 	find most recent error step error in sysjobhistory and pull name based on instance_id from history table.
-- 	在sysjobhistory中查找最近的步骤错误，并根据历史记录表中的instance_id拉取名称。
 
declare @last_error     varchar(255)
declare @last_error_job_name    varchar(255)
set @last_error     = ( select top 1 instance_id from sysjobhistory where message like ''%The step failed%'' order by run_date desc )
set @last_error_job_name    = ( select sj.name from sysjobs sj join sysjobhistory sjh on sj.job_id = sjh.job_id where instance_id = @last_error )
 
----------------------------------------------------------------------
-- 	create temp table to store error information.
-- 	创建临时表格保存错误信息
 
if object_id(''tempdb..#agent_job_step_error_report'') is not null
    drop table #agent_job_step_error_report
 
create table #agent_job_step_error_report
(
    [id]            int identity (1,1)
,   [server_name]       varchar(255)
,   [time_of_error]     varchar(255)
,   [job_name]      varchar(255)
,   [step_id]       int not null
,   [step_name]     varchar(255)
,   [duration]      varchar(255)
,   [error_message]     varchar(max)
)
 
----------------------------------------------------------------------
-- 	get information from job system tables for job step error report.
-- 	从作业系统表格中获取作业步骤错误报告的信息。

insert into #agent_job_step_error_report ([server_name], [time_of_error], [job_name], [step_id], [step_name], [duration], [error_message])
select
    ''server name''     = @@servername
,   ''time of error''   = datename(dw, msdb.dbo.agent_datetime(run_date, run_time) ) + '':  '' + convert(char, msdb.dbo.agent_datetime(run_date, run_time) , 9)
,   ''job name''        = sj.name
,   ''step id''     = sjh.step_id
,   ''step name''       = sjh.step_name
,   ''duration''        = CAST(sjh.run_duration/10000 as varchar)  + '':'' + CAST(sjh.run_duration/100%100 as varchar) + '':'' + CAST(sjh.run_duration%100 as varchar)
,   ''error message''   = sjh.message
from
    msdb..sysjobs sj join msdb..sysjobhistory sjh on sj.job_id = sjh.job_id
where
    instance_id = @last_error
order by sj.name, step_id asc
 
----------------------------------------------------------------------
-- 	create temp table to store job information
-- 	创建临时表格来保存作业信息
 
if object_id(''tempdb..#agent_job_information'') is not null
    drop table #agent_job_information
 
create table #agent_job_information
(
    [id]                int identity (1,1)
,   [job_name]          varchar(255)
,   [step_id]           int not null
,   [step_name]         varchar(255)
,   [process_type]          varchar(255)
,   [last_ran]          varchar(255)
)
 
----------------------------------------------------------------------
-- 	get all job, and step info for quick reference including the previous run duration and the last known run timestamp before the most previous error.
-- 	获取所有作业和步骤信息以便快速参考，包括之前的运行持续时间和最早一次错误之前的最后一个已知运行时间戳。
 
insert into #agent_job_information ([job_name], [step_id], [step_name], [process_type], [last_ran])
select
    ''job name''            = sj.name
,   ''step id''         = sjs.step_id
,   ''step name''           = sjs.step_name
,   ''process type''        = sjs.subsystem
,   ''last ran''            = datename(dw, dateadd(millisecond, sjs.last_run_time,convert(datetime,cast(nullif(sjs.last_run_date,0) as nvarchar(10))))) + '':  '' + convert(char, dateadd(millisecond, sjs.last_run_time,convert(datetime,cast(nullif(sjs.last_run_date,0) as nvarchar(10)))), 9)
from
    msdb..sysjobs sj join msdb..sysjobsteps sjs on sj.job_id = sjs.job_id
where
    sj.name = @last_error_job_name
order by
    sj.name, sjs.step_id asc
 
----------------------------------------------------------------------
-- 	create conditions for html tables in top and mid sections of email.
-- 	在电子邮件的顶部和中间部分为html表格创建条件。
 
declare @xml_top            NVARCHAR(MAX)
declare @xml_mid            NVARCHAR(MAX)
declare @body_top           NVARCHAR(MAX)
declare @body_mid           NVARCHAR(MAX)
 
----------------------------------------------------------------------
-- set xml top table td''s
-- create html table object for: #agent_job_step_error_report
-- 	设置xml顶部表格td’’s
-- 	为#agent_job_step_error_report创建html表格对象
-- 	
set @xml_top = 
    cast(
        (select
            [server_name]   as ''td''
        ,   ''''
        ,   [time_of_error] as ''td''
        ,   ''''
        ,   [job_name]  as ''td''
        ,   ''''
        ,   [step_id]   as ''td''
        ,   ''''
        ,   [step_name] as ''td''
        ,   ''''
        ,   [duration]  as ''td''
        ,   ''''
        ,   [error_message] as ''td''
        from  #agent_job_step_error_report 
        --order by rank 
        for xml path(''tr'')
    ,   elements)
    as NVARCHAR(MAX)
        )
 
----------------------------------------------------------------------
-- set xml mid table td''s
-- create html table object for: #agent_job_information
-- 	设置xml中部表格 td’’s
-- 	为#agent_job_information创建html表格对象

set @xml_mid = 
    cast(
        (select
            [job_name]      as ''td''
        ,   ''''
        ,   [step_id]       as ''td''
        ,   ''''
        ,   [step_name]     as ''td''
        ,   ''''
        ,   [process_type]      as ''td''
        ,   ''''
        ,   [last_ran]      as ''td''
 
        from  #agent_job_information 
        order by [job_name], [step_id] asc
        for xml path(''tr'')
    ,   elements)
    as NVARCHAR(MAX)
        )
 
----------------------------------------------------------------------
 
set @body_top =
        ''<html>
        <head>
            <style>
                    h1{
                        font-family: sans-serif;
                        font-size: 110%;
                    }
                    h3{
                        font-family: sans-serif;
                        color: red;
                    }
 
                    table, td, tr, th {
                        font-family: sans-serif;
                        border: 1px solid black;
                        border-collapse: collapse;
                    }
                    th {
                        text-align: left;
                        background-color: gray;
                        color: white;
                        padding: 5px;
                    }
 
                    td {
                        padding: 5px;
                    }
            </style>
        </head>
        <body>
        <H3>'' + @message_subject + ''</H3>
        <h1>Note: The Server Time is operating on: '' + @server_time_zone + ''
-- 	注意：服务器时间正在运行：''+ @server_time_zone +''
        <table border = 1>
        <tr>
            <th> Server Name  </th>
            <th> Time of Error    </th>
            <th> Job Name </th>
            <th> Step ID  </th>
            <th> Step Name    </th>
            <th> Duration </th>
            <th> Error Message    </th>
        </tr>''
         
set @body_top = @body_top + @xml_top + ''</table>
 
<h1>Quick Reference: Job Info</h1>
<h1>快速参考：作业信息</h1>
 
<table border = 1>
        <tr>
            <th> Job Name </th>
            <th> Step ID  </th>
            <th> Step Name    </th>
            <th> Process Type </th>
            <th> Last Ran </th>
        </tr>''      
         
+ @xml_mid + ''</table>
        <h1>Go to the server by pasting in the following text under: Start-Run, or (Win + R)</h1>
<h1>通过粘贴以下文本进入服务器：开始 - 运行或（Win + R）</ h1>
        <h1>mstsc -v:'' + @server_name_basic + ''</h1>''
+ ''</body></html>''
 
----------------------------------------------------------------------
-- send email.
 
EXEC msdb.dbo.sp_send_dbmail
    @profile_name       = ''SQLDatabaseMailProfile''
,   @recipients     = ''SQLJobAlerts@MyDomain.com''
,   @subject        = @message_subject
,   @body           = @body_top
,   @body_format        = ''HTML'';
 
drop table #agent_job_step_error_report
drop table #agent_job_information
 
 
', 
        @database_name=N'master', 
        @flags=0
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_update_job @job_id = @jobId, @start_step_id = 1
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_add_jobserver @job_id = @jobId, @server_name = N'(local)'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
COMMIT TRANSACTION
GOTO EndSave
QuitWithRollback:
    IF (@@TRANCOUNT > 0) ROLLBACK TRANSACTION
EndSave:
 
GO


```



[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

- **李聪明的数据库 Lee's Clever Data**
- **Mike的数据库宝典 Mikes Database Collection**
- **李聪明的数据库** "Lee Songming"

[![Gist](https://img.shields.io/badge/Gist-李聪明的数据库-<COLOR>.svg)](https://gist.github.com/congmingshuju)
[![Twitter](https://img.shields.io/badge/Twitter-mike的数据库宝典-<COLOR>.svg)](https://twitter.com/mikesdatawork?lang=en)
[![Wordpress](https://img.shields.io/badge/Wordpress-mike的数据库宝典-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

---
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Lee Songming](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/1-clever-data-github.png "李聪明的数据库")

