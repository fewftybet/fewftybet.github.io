---
title: 数据库安全基线检查与加固指南（安服视角）
date: 2026-07-16 12:20:00 +0800
categories: [经验分享]
tags: [安全加固, 数据库, 基线检查, 等保, MySQL, MSSQL, Oracle]
description: 本文档从安全服务（安服）专业视角出发，依据等级保护2.0三级标准与CIS Benchmark数据库基线，围绕身份鉴别、访问控制、安全审计、资源控制、剩余信息保护、入侵防范、恶意代码防范七大维度，系统性梳理MySQL、Microsoft SQL Server（MSSQL）、Oracle三大主流数据库的安全基线检查与加固方案。
---

# 数据库安全基线检查与加固指南（安服视角）

# 一、概述

本文档从安全服务（安服）专业视角出发，依据等级保护2\.0三级标准与CIS Benchmark数据库基线，围绕身份鉴别、访问控制、安全审计、资源控制、剩余信息保护、入侵防范、恶意代码防范七大维度，系统性梳理MySQL、Microsoft SQL Server（MSSQL）、Oracle三大主流数据库的安全基线检查与加固方案。适用于生产环境数据库合规性检查、等保测评整改与安全加固实施。

**安服实施原则：**数据库加固前必须全量备份配置文件与业务数据，优先在测试环境验证变更影响；高风险操作（权限回收、参数修改）需安排业务低峰期执行，留存回滚脚本；加固完成后执行业务SQL回归测试与连通性验证。

# 二、身份鉴别

## 2\.1 账户清理与命名规范

**默认与冗余账户清理。**删除或禁用测试账户、共享账户、离职人员账户，禁用或重命名默认管理员账户（MySQL的root@%、MSSQL的sa、Oracle的SYS/SYSTEM外的内置账户）。安服检查时需逐一核对用户列表，确认不存在空密码账户与匿名登录账户。

```sql
-- 查看所有用户及可登录主机
SELECT user, host, authentication_string FROM mysql.user;

-- 删除匿名用户
DELETE FROM mysql.user WHERE user='';

-- 删除允许任意主机登录的root
DELETE FROM mysql.user WHERE user='root' AND host='%';

-- 重命名root管理员（建议）
RENAME USER 'root'@'localhost' TO 'dba_admin'@'localhost';

FLUSH PRIVILEGES;

```

```sql
-- 查看所有登录名
SELECT name, type_desc, is_disabled FROM sys.sql_logins;

-- 禁用sa账户（改用Windows认证或自定义管理员）
ALTER LOGIN sa DISABLE;

-- 删除无用登录名
DROP LOGIN test_user;

-- 检查空密码账户（弱口令排查）
SELECT name FROM sys.sql_logins WHERE PWDCOMPARE('', password_hash) = 1;

```

```sql
-- 查看所有用户及状态
SELECT username, account_status, default_tablespace FROM dba_users;

-- 锁定未使用的内置账户
ALTER USER ANONYMOUS ACCOUNT LOCK;
ALTER USER XDB ACCOUNT LOCK;
ALTER USER CTXSYS ACCOUNT LOCK;

-- 删除测试用户
DROP USER test_user CASCADE;

-- 检查无密码用户
SELECT username FROM dba_users WHERE password IS NULL;

```

## 2\.2 密码策略强化

**强口令策略落地。**配置密码复杂度校验，要求密码长度不少于12位，包含大小写字母、数字与特殊字符三类组合；设置密码过期周期（90天），禁止复用最近5次历史密码。安服核查时需确认密码验证插件/Profile已启用，不存在弱口令账户。

```sql
-- 安装密码验证插件（MySQL 5.7+）
INSTALL PLUGIN validate_password SONAME 'validate_password.so';

-- 密码策略配置（my.cnf / my.ini）
[mysqld]
validate_password_policy = STRONG
validate_password_length = 12
validate_password_mixed_case_count = 1
validate_password_number_count = 1
validate_password_special_char_count = 1

-- 设置密码过期90天（全局）
SET GLOBAL default_password_lifetime = 90;

-- 对指定用户设置
ALTER USER 'app_user'@'192.168.%' PASSWORD EXPIRE INTERVAL 90 DAY;

```

```sql
-- 启用密码复杂度与过期策略（依赖Windows组策略）
-- 创建登录时强制密码策略
CREATE LOGIN app_user WITH PASSWORD = 'ComplexPwd!123',
    CHECK_POLICY = ON,
    CHECK_EXPIRATION = ON;

-- 修改已有登录名启用策略
ALTER LOGIN app_user WITH CHECK_POLICY = ON, CHECK_EXPIRATION = ON;

-- 查看密码策略配置
SELECT name, is_policy_checked, is_expiration_checked FROM sys.sql_logins;

```

```sql
-- 创建安全密码Profile
CREATE PROFILE SECURE_PROFILE LIMIT
    PASSWORD_LIFE_TIME 90
    PASSWORD_GRACE_TIME 7
    PASSWORD_REUSE_TIME 365
    PASSWORD_REUSE_MAX 5
    PASSWORD_LOCK_TIME 0.5
    FAILED_LOGIN_ATTEMPTS 5
    PASSWORD_VERIFY_FUNCTION VERIFY_FUNCTION_11G;

-- 应用Profile到用户
ALTER USER app_user PROFILE SECURE_PROFILE;

-- 启用默认密码复杂度函数
@?/rdbms/admin/utlpwdmg.sql

```

## 2\.3 登录失败锁定

**防暴力破解机制。**配置连续登录失败自动锁定账户策略，MySQL通过connection\-control插件实现，MSSQL依赖Windows账户锁定策略，Oracle通过Profile的FAILED\_LOGIN\_ATTEMPTS参数。安服视角需确认暴力破解防护生效，核心数据库建议配合数据库防火墙做IP级封禁。

```sql
-- 安装连接控制插件
INSTALL PLUGIN CONNECTION_CONTROL SONAME 'connection_control.so';
INSTALL PLUGIN CONNECTION_CONTROL_FAILED_LOGIN_ATTEMPTS SONAME 'connection_control.so';

-- 配置：连续5次失败后延迟响应
SET GLOBAL connection_control_failed_connections_threshold = 5;
SET GLOBAL connection_control_min_connection_delay = 1000;
SET GLOBAL connection_control_max_connection_delay = 2147483647;

-- my.cnf持久化
plugin-load-add = connection_control.so
connection-control = FORCE
connection-control-failed-login-attempts = FORCE
connection_control_failed_connections_threshold = 5

```

## 2\.4 认证方式加固

**优先集成域认证。**MSSQL优先使用Windows身份认证模式，禁用混合模式；Oracle配置OS认证或集成AD；MySQL 8\.0使用caching\_sha2\_password强认证插件。核心数据库管理员账户建议补充双因素认证。

```sql
-- 查看当前认证模式
SELECT SERVERPROPERTY('IsIntegratedSecurityOnly') AS is_windows_only;

-- 设置为仅Windows认证（推荐，安全性最高）
-- 操作路径：SSMS → 服务器属性 → 安全性 → Windows身份验证模式

-- 如必须混合模式，确保sa强密码并禁用
ALTER LOGIN sa WITH PASSWORD = 'StrongAdminPwd!2024';
ALTER LOGIN sa DISABLE;

```

# 三、访问控制

## 3\.1 权限最小化分配

**最小权限原则落地。**禁止应用账号拥有ALL PRIVILEGES或sysadmin/dba角色，按业务需求精细化授权；区分管理员、运维、应用、只读四类角色，实现职责分离。安服检查时需逐一核对高权限用户，确认无越权分配。

```sql
-- 应用账号仅授予业务库DML权限，限定IP段
GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_user'@'192.168.1.%';

-- 核查高危权限用户
SELECT user, host, File_priv, Process_priv, Super_priv FROM mysql.user;

-- 回收多余权限
REVOKE DROP, ALTER ON app_db.* FROM 'app_user'@'192.168.1.%';

FLUSH PRIVILEGES;

```

```sql
-- 查看服务器级角色成员
SELECT sp.name AS login_name, srp.name AS server_role
FROM sys.server_role_members srm
JOIN sys.server_principals sp ON srm.member_principal_id = sp.principal_id
JOIN sys.server_principals srp ON srm.role_principal_id = srp.principal_id;

-- 回收sysadmin角色非必要成员
ALTER SERVER ROLE sysadmin DROP MEMBER dev_user;

-- 数据库级只读授权
USE app_db;
CREATE USER app_reader FOR LOGIN app_reader;
ALTER ROLE db_datareader ADD MEMBER app_reader;

```

```sql
-- 查看DBA角色用户
SELECT grantee FROM dba_role_privs WHERE granted_role = 'DBA';

-- 查看系统权限（高危）
SELECT grantee, privilege FROM dba_sys_privs
WHERE privilege IN ('DROP ANY TABLE', 'ALTER ANY TABLE', 'CREATE ANY TABLE');

-- 回收多余权限
REVOKE DBA FROM app_user;
REVOKE UNLIMITED TABLESPACE FROM app_user;

-- 按角色精细化授权
CREATE ROLE app_read_role;
GRANT SELECT ON app_schema.table_name TO app_read_role;

```

## 3\.2 网络访问控制

**监听地址与IP白名单。**数据库仅监听内网业务网卡地址，禁止0\.0\.0\.0全网卡监听；配置防火墙或数据库安全组，仅允许应用服务器与运维网段访问数据库端口。安服核查时确认数据库端口不暴露公网。

```ini
# my.cnf
[mysqld]
bind-address = 10.0.1.100
port = 3306

# 限制local_infile防文件读取
local_infile = OFF

# 账户级IP限制（创建用户时指定host）
# CREATE USER 'app_user'@'192.168.1.%' ...

```

```sql
-- 配置路径：SQL Server配置管理器 → SQL Server网络配置 → 协议
-- 1. 禁用Named Pipes（如非必需）
-- 2. TCP/IP属性 → IP地址 → 仅保留业务网卡IP
-- 3. 修改默认端口1433为非知名端口

-- 查看当前监听
SELECT DISTINCT local_net_address, local_tcp_port
FROM sys.dm_exec_connections WHERE local_net_address IS NOT NULL;

```

```ini
# listener.ora 配置
# 设置监听器密码
PASSWORDS_LISTENER = your_secure_password

# 限制可连接IP
TCP.VALIDNODE_CHECKING = YES
TCP.INVITED_NODES = (10.0.1.10, 10.0.1.20, 10.0.1.0/24)

# sqlnet.ora 连接超时
SQLNET.INBOUND_CONNECT_TIMEOUT = 30
SQLNET.EXPIRE_TIME = 10

```

## 3\.3 三权分立与职责分离

**等保三级强制要求。**分离系统管理员、安全管理员、审计管理员三类角色权限，实现相互制约。安服检查时需确认DBA不拥有审计权限，审计员无法修改业务数据，安全员不直接操作业务库。

```sql
-- 安全管理员（负责权限管理与安全策略）
CREATE USER sec_admin IDENTIFIED BY "SecPwd!2024" PROFILE SECURE_PROFILE;
GRANT CREATE USER, ALTER USER, DROP USER TO sec_admin;
GRANT CREATE ROLE, GRANT ANY ROLE TO sec_admin;

-- 审计管理员（独立审计权限，DBA无法干预）
CREATE USER audit_admin IDENTIFIED BY "AuditPwd!2024" PROFILE SECURE_PROFILE;
GRANT AUDIT SYSTEM, AUDIT ANY TO audit_admin;
GRANT SELECT ON dba_audit_trail TO audit_admin;

```

# 四、安全审计

## 4\.1 审计功能全量启用

**关键操作全审计。**启用数据库审计功能，覆盖登录成功/失败、权限变更、DDL操作、敏感表DML、数据导出等关键事件。安服基线核查需确认审计日志不可被普通用户删除，审计管理员独立管理。

```ini
# MySQL Enterprise Audit或MariaDB Audit Plugin
# my.cnf 通用查询日志（基础审计）
general_log = ON
general_log_file = /var/log/mysql/general.log

# 慢查询日志
slow_query_log = ON
long_query_time = 2

# 企业版审计插件（推荐）
plugin-load-add = audit_log.so
audit_log_format = JSON
audit_log_policy = ALL
audit_log_rotate_on_size = 100M
audit_log_rotations = 30

```

```sql
-- 创建服务器审计
CREATE SERVER AUDIT Security_Audit
TO FILE (FILEPATH = 'D:\SQL_Audit\', MAXSIZE = 100MB, MAX_ROLLOVER_FILES = 30)
WITH (QUEUE_DELAY = 1000, ON_FAILURE = CONTINUE);

ALTER SERVER AUDIT Security_Audit WITH (STATE = ON);

-- 服务器级审计规范
CREATE SERVER AUDIT SPECIFICATION Server_Audit_Spec
FOR SERVER AUDIT Security_Audit
ADD (FAILED_LOGIN_GROUP),
ADD (SUCCESSFUL_LOGIN_GROUP),
ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
ADD (DATABASE_CHANGE_GROUP);

ALTER SERVER AUDIT SPECIFICATION Server_Audit_Spec WITH (STATE = ON);

```

```sql
-- 启用标准审计
ALTER SYSTEM SET audit_trail = DB, EXTENDED SCOPE = SPFILE;
ALTER SYSTEM SET audit_sys_operations = TRUE SCOPE = SPFILE;
-- 重启数据库生效

-- 审计登录事件
AUDIT SESSION BY ACCESS WHENEVER NOT SUCCESSFUL;
AUDIT SESSION BY ACCESS WHENEVER SUCCESSFUL;

-- 审计DDL操作
AUDIT CREATE TABLE, ALTER TABLE, DROP TABLE BY ACCESS;
AUDIT CREATE USER, ALTER USER, DROP USER BY ACCESS;

-- 审计敏感表DML
AUDIT SELECT, INSERT, UPDATE, DELETE ON app_schema.sensitive_table BY ACCESS;

```

## 4\.2 日志留存与保护

**审计日志防篡改。**审计日志文件设置只读权限，仅审计管理员可访问；配置日志轮转与容量上限，核心数据库日志留存不少于180天；建议同步转发至集中日志平台（SIEM），防止本地日志被攻击者清理。

```bash
# MySQL审计日志权限
chmod 600 /var/log/mysql/audit.log
chown mysql:mysql /var/log/mysql/audit.log

# MSSQL审计目录权限
# 仅SQL Server服务账户与审计管理员拥有读写权限

# Oracle审计表空间独立
# CREATE TABLESPACE audit_tbs DATAFILE '/data/oracle/audit01.dbf' SIZE 10G;

```

## 4\.3 关键审计事件清单

|事件类型|MySQL|MSSQL|Oracle|安服关注要点|
|---|---|---|---|---|
|登录失败|general\_log/audit|FAILED\_LOGIN\_GROUP|AUDIT SESSION NOT SUCCESSFUL|暴力破解识别|
|权限变更|GRANT/REVOKE记录|SERVER\_ROLE\_MEMBER\_CHANGE|AUDIT GRANT|越权提权检测|
|DDL操作|general\_log|DATABASE\_CHANGE\_GROUP|AUDIT DDL|未授权结构变更|
|敏感表查询|企业版审计|数据库级审计规范|对象级AUDIT|数据泄露溯源|
|批量删除|binlog\+审计|DELETE审计|DML审计|恶意删库检测|

# 五、资源控制

## 5\.1 连接数与会话限制

**资源滥用防护。**配置最大连接数上限，防止连接耗尽DoS攻击；设置单用户最大会话数、空闲会话超时自动断开。安服核查时确认资源限制已配置，防止单应用异常占满连接池。

```ini
# my.cnf
max_connections = 500
wait_timeout = 600
interactive_timeout = 600
max_connect_errors = 100

# 用户级资源限制
# CREATE USER 'app_user'@'192.168.%'
# WITH MAX_USER_CONNECTIONS 100
# MAX_QUERIES_PER_HOUR 10000

```

```sql
-- Profile中配置会话资源限制
ALTER PROFILE SECURE_PROFILE LIMIT
    SESSIONS_PER_USER 10
    CPU_PER_SESSION 10000
    CONNECT_TIME 480
    IDLE_TIME 30
    LOGICAL_READS_PER_SESSION 100000;

```

## 5\.2 存储与文件权限

**数据文件权限收敛。**数据库数据目录、日志目录权限严格控制，仅数据库运行账户可读写；禁止数据库进程以root/system管理员权限运行；备份文件加密存储且权限收敛。

```bash
# 数据目录权限加固
chmod 700 /var/lib/mysql
chown -R mysql:mysql /var/lib/mysql

# 配置文件权限
chmod 600 /etc/my.cnf
chown root:root /etc/my.cnf

# 检查FILE权限用户
SELECT user, host FROM mysql.user WHERE File_priv = 'Y';

```

## 5\.3 传输加密

**链路数据加密。**启用SSL/TLS加密数据库连接，防止中间人攻击窃听敏感数据；生产环境强制应用侧使用加密连接。安服检查时确认非加密连接已禁用或仅限内网可信网段。

```ini
# my.cnf 启用SSL
[mysqld]
ssl-ca = /etc/mysql/ssl/ca.pem
ssl-cert = /etc/mysql/ssl/server-cert.pem
ssl-key = /etc/mysql/ssl/server-key.pem
require_secure_transport = ON

# 用户级强制SSL
# GRANT SELECT ON app_db.* TO 'app_user'@'192.168.%' REQUIRE SSL;

```

```sql
-- SQL Server配置管理器 → 协议属性 → 标志 → 强制加密：是

-- 查看连接加密状态
SELECT encrypt_option, auth_scheme, COUNT(*)
FROM sys.dm_exec_connections GROUP BY encrypt_option, auth_scheme;

```

```ini
# sqlnet.ora 配置网络加密
SQLNET.ENCRYPTION_SERVER = REQUIRED
SQLNET.ENCRYPTION_TYPES_SERVER = (AES256, AES192, AES18)
SQLNET.CRYPTO_CHECKSUM_SERVER = REQUIRED
SQLNET.CRYPTO_CHECKSUM_TYPES_SERVER = (SHA256, SHA384, SHA512)

```

# 六、剩余信息保护

## 6\.1 敏感数据脱敏与加密

**存储加密落地。**敏感字段（身份证、手机号、密码）入库加密存储，禁止明文；透明数据加密（TDE）启用整库加密，防止数据文件被盗后离线挂载读取。安服核查时确认密码字段使用哈希加盐，敏感数据不可逆存储。

```sql
-- 透明页加密（InnoDB企业版）
-- 建表时启用加密
CREATE TABLE sensitive_table (
    id INT PRIMARY KEY,
    id_card VARCHAR(255)
) ENCRYPTION='Y';

-- 密码字段哈希存储（应用层加盐）
-- 推荐：bcrypt / Argon2，禁止MD5/SHA1

```

```sql
-- 创建主密钥与证书
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'MasterKeyPwd!2024';
CREATE CERTIFICATE TDE_Cert WITH SUBJECT = 'TDE Certificate';

-- 启用TDE
USE app_db;
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE TDE_Cert;

ALTER DATABASE app_db SET ENCRYPTION ON;

-- 查看加密状态
SELECT name, is_encrypted FROM sys.databases;

```

```sql
-- 表空间级加密
CREATE TABLESPACE secure_tbs DATAFILE '/data/oracle/secure01.dbf' SIZE 10G
ENCRYPTION USING 'AES256' DEFAULT STORAGE (ENCRYPT);

-- 列级加密
ALTER TABLE sensitive_table MODIFY (id_card ENCRYPT USING 'AES256' SALT);

```

## 6\.2 备份数据保护

**备份安全管控。**数据库备份文件加密存储，备份目录权限严格控制；备份介质异地存放，定期执行备份恢复演练。安服检查时确认备份文件未存放在Web可访问目录，备份账号权限最小化。

```bash
# MySQL备份目录权限
mkdir -p /backup/mysql
chmod 700 /backup/mysql
chown mysql:mysql /backup/mysql

# 备份时加密
# mysqldump --all-databases | gzip | openssl enc -aes-256-cbc -salt -out backup.enc

```

## 6\.3 会话残留清理

**内存敏感数据清理。**配置会话超时自动断开，临时表与会话变量自动清理；连接池配置连接重置，防止复用连接时泄漏上一会话上下文。

```ini
# my.cnf
wait_timeout = 600
interactive_timeout = 600

# 临时表会话断开自动删除
# 连接池配置：testOnBorrow + connectionReset

```

# 七、入侵防范

## 7\.1 补丁与版本管理

**漏洞修复闭环。**定期升级数据库补丁包，高危漏洞（如CVE级注入、提权）72小时内完成修复；建立版本基线，禁止使用已停止维护的过期版本。安服视角需定期执行数据库漏洞扫描，确认无高危CVE遗留。

```sql
-- MySQL版本
SELECT VERSION();

-- MSSQL版本
SELECT @@VERSION;

-- Oracle版本
SELECT * FROM v$version;
SELECT patch_id, patch_type, action, status FROM dba_registry_sqlpatch;

```

## 7\.2 SQL注入防护

**数据库侧注入缓解。**关闭错误回显详细信息，限制存储过程执行权限，禁用高危内置函数；部署数据库审计或WAF检测SQL注入特征。

```sql
-- MySQL禁用local_infile防文件读取
SET GLOBAL local_infile = OFF;

-- MSSQL禁用xp_cmdshell（关键）
sp_configure 'show advanced options', 1;
RECONFIGURE;
sp_configure 'xp_cmdshell', 0;
RECONFIGURE;

-- 禁用OLE自动化存储过程
sp_configure 'Ole Automation Procedures', 0;
RECONFIGURE;

-- Oracle禁用Java权限（非必需时）
REVOKE JAVAUSERPRIV FROM public;

```

## 7\.3 提权漏洞防范

**数据库运行权限收敛。**数据库服务以低权限专用账户运行，禁止以root/LOCAL SYSTEM启动；回收数据库进程对系统目录的写入权限；定期排查数据库提权漏洞CVE并打补丁。

```bash
# MySQL：确认以mysql用户运行
ps aux | grep mysqld | grep -v grep

# MSSQL：服务账户应为虚拟服务账户或专用域账户，非Local System

```

# 八、恶意代码防范

## 8\.1 存储过程与触发器审计

**数据库后门排查。**定期审计新增存储过程、触发器、函数、Job作业，识别未授权的持久化后门；核查启动时自动执行的存储过程与代理作业。安服应急响应场景需重点排查数据库层面的持久化入口。

```sql
-- 查看所有存储过程与函数
SELECT db, name, type, definer, created FROM mysql.proc;

-- 查看定时事件
SELECT event_name, status, execute_at FROM information_schema.EVENTS;

-- 查看触发器
SELECT trigger_schema, trigger_name, event_manipulation
FROM information_schema.TRIGGERS;

```

```sql
-- 查看SQL Agent作业
SELECT name, enabled, date_created FROM msdb.dbo.sysjobs ORDER BY date_created DESC;

-- 查看服务器级触发器
SELECT name, type_desc, is_disabled, create_date FROM sys.server_triggers;

-- 查看启动存储过程
SELECT name FROM sys.procedures WHERE OBJECTPROPERTY(object_id, 'ExecIsStartup') = 1;

```

```sql
-- 查看定时JOB
SELECT job_name, owner, enabled, job_type FROM dba_scheduler_jobs;

-- 查看数据库级触发器
SELECT owner, trigger_name, trigger_type, status
FROM dba_triggers ORDER BY created DESC;

```

## 8\.2 Webshell与提权后门检测

**数据库层面恶意代码排查。**检查是否存在UDF提权函数（MySQL）、CLR程序集（MSSQL）、Java存储过程（Oracle）等数据库级后门载体；核查异常的数据库链接（DBLINK）配置。

```sql
-- 检查自定义函数（UDF提权）
SELECT * FROM mysql.func;

-- 检查是否存在sys_exec、sys_eval等高危UDF
SELECT name, dl FROM mysql.func WHERE name IN ('sys_exec', 'sys_eval', 'sys_get');

```

```sql
-- 查看已注册程序集
SELECT name, permission_set_desc, create_date FROM sys.assemblies;

-- 禁用CLR集成（非必需）
sp_configure 'clr enabled', 0;
RECONFIGURE;

```

# 九、附录：安服基线核查速查表

|加固维度|核心检查项|MySQL合规标准|MSSQL合规标准|Oracle合规标准|
|---|---|---|---|---|
|身份鉴别|默认账户|无匿名用户、root@%删除|sa禁用或强密码|闲置内置账户锁定|
||密码策略|validate\_password启用，≥12位|CHECK\_POLICY=ON|Profile复杂度\+90天过期|
||登录锁定|connection\-control插件|Windows账户锁定策略|FAILED\_LOGIN\_ATTEMPTS=5|
|访问控制|权限最小化|无ALL PRIVILEGES应用账号|非必要成员退出sysadmin|普通用户无DBA角色|
||网络监听|bind\-address绑定内网IP|仅业务网卡监听|TCP\.VALIDNODE\_CHECKING|
||三权分立|管理员/审计员分离|服务器角色分离|SYSDBA/审计员独立|
|安全审计|审计启用|审计插件/通用日志开启|Server Audit启用|audit\_trail=DB|
||日志留存|≥180天，权限600|≥180天，目录受限|审计表独立表空间|
|资源控制|连接超时|wait\_timeout=600|连接池超时配置|IDLE\_TIME=30分钟|
||传输加密|require\_secure\_transport=ON|强制加密启用|SQLNET\.ENCRYPTION=REQUIRED|
|剩余信息保护|存储加密|TDE/表加密|TDE启用|TDE表空间加密|
||备份加密|备份文件加密|备份加密选项|RMAN加密备份|
|入侵防范|高危组件|local\_infile=OFF|xp\_cmdshell禁用|Java权限回收|
||补丁版本|最新稳定版|最新CU补丁|最新PSU/RU补丁|
|恶意代码防范|存储过程审计|定期核查mysql\.func|CLR程序集审计|JOB与触发器审计|
||后门检测|UDF函数排查|启动存储过程排查|DBLINK异常排查|

**安服实施注意事项：**数据库加固前必须全量备份数据与配置文件，优先在测试环境验证；权限回收操作需与业务方确认影响范围，防止应用报错；审计开启前评估性能损耗，高并发场景调整审计范围与队列参数；xp\_cmdshell、UDF等高危组件禁用前确认无业务依赖。
