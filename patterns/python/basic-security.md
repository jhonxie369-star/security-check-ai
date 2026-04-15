# Python语言 - 基础代码安全检查

## ⚠️ 核心原则

**本文件提供检查思路和常见模式，不是固定规则。**

**工作流程**：
1. 识别项目特点（框架、输入点、危险操作）
2. 使用grep搜索关键代码位置
3. 读取完整上下文（前后50-100行）
4. AI深度分析，结合源码判断真实风险

---

## 1. SQL注入检查指导

### Python 的 SQL 执行特点
- **原生SQL**：cursor.execute() 字符串拼接危险，参数化安全
- **Django ORM**：filter() 参数化安全，raw() 和 extra() 需检查
- **SQLAlchemy**：text() 需要参数绑定
- **Flask-SQLAlchemy**：query.filter() 安全，execute() 需检查

### 搜索关键代码

```bash
# 搜索SQL执行点
grep -rn "cursor\.execute\|execute\|executemany\|raw\|extra" --include="*.py" -A 10 -B 5

# 搜索字符串拼接
grep -rn "f\".*SELECT\|%.*SELECT\|+.*SELECT" --include="*.py" -A 5 -B 5

# 搜索Django raw查询
grep -rn "\.raw\|\.extra" --include="*.py" -A 10 -B 5
```

### AI分析要点

**第1步：识别参数来源**
```python
# 用户输入（高风险）
username = request.GET.get('username')
user_id = request.POST.get('id')
data = request.json.get('data')

# 配置文件（低风险）
table = config['table_name']

# 常量（无风险）
status = 'active'
```

**第2步：检查是否使用安全API**
```python
# ✅ 安全：参数化查询
cursor.execute("SELECT * FROM users WHERE username = %s", (username,))

# ✅ 安全：Django ORM
User.objects.filter(username=username)

# ✅ 安全：SQLAlchemy参数绑定
session.execute(text("SELECT * FROM users WHERE id = :id"), {"id": user_id})

# ❌ 危险：f-string拼接
sql = f"SELECT * FROM users WHERE username = '{username}'"
cursor.execute(sql)

# ❌ 危险：%格式化
sql = "SELECT * FROM users WHERE username = '%s'" % username
cursor.execute(sql)

# ❌ 危险：+拼接
sql = "SELECT * FROM users WHERE id = " + user_id
cursor.execute(sql)
```

**第3步：检查输入验证**
```python
# ✅ 有验证
if not user_id.isdigit():
    raise ValueError("Invalid ID")

# ✅ 使用白名单
if field_name not in ['username', 'email', 'status']:
    raise ValueError("Invalid field")
```

### 风险判断

| 场景 | 风险等级 |
|------|---------|
| 用户输入 + 字符串拼接 + 无验证 | 严重 |
| 用户输入 + 字符串拼接 + 有验证 | 高危 |
| 用户输入 + 参数化查询 | 安全 |
| 配置文件 + 字符串拼接 | 低危 |

### 修复示例

```python
# ❌ 危险：字符串拼接
def get_user(username):
    sql = f"SELECT * FROM users WHERE username = '{username}'"
    cursor.execute(sql)

# ✅ 安全：参数化查询
def get_user(username):
    sql = "SELECT * FROM users WHERE username = %s"
    cursor.execute(sql, (username,))

# ✅ Django ORM
User.objects.filter(username=username)

# ❌ 危险：动态字段名
def search(field_name, value):
    sql = f"SELECT * FROM users WHERE {field_name} = %s"
    cursor.execute(sql, (value,))

# ✅ 安全：字段名白名单
def search(field_name, value):
    allowed_fields = {'username', 'email', 'status'}
    if field_name not in allowed_fields:
        raise ValueError("Invalid field")
    sql = f"SELECT * FROM users WHERE {field_name} = %s"
    cursor.execute(sql, (value,))
```

---

## 2. 命令注入检查指导

### Python 的命令执行特点
- **os.system()**：字符串拼接危险
- **subprocess**：shell=True 危险，列表参数安全
- **eval/exec**：极度危险

### 搜索关键代码

```bash
# 搜索命令执行
grep -rn "os\.system\|subprocess\|exec\|eval\|__import__" --include="*.py" -A 10 -B 5

# 搜索shell=True
grep -rn "shell=True" --include="*.py" -A 5 -B 5
```

### AI分析要点

```python
# ❌ 危险：os.system
import os
host = request.args.get('host')
os.system(f'ping -c 4 {host}')

# ❌ 危险：subprocess with shell=True
import subprocess
subprocess.call(f'ping -c 4 }', shell=True)

# ✅ 安全：列表参数 + shell=False
import subprocess
host = request.args.get('host')
if not re.match(r'^[a-zA-Z0-9.-]+$', host):
    raise ValueError("Invalid host")
subprocess.call(['ping', '-c', '4', host], shell=False)

# ❌ 危险：eval/exec
code = request.args.get('code')
eval(code)  # 极度危险
exec(code)  # 极度危险
```

### 检查要点

- 是否使用列表形式参数
- 是否设置shell=False
- 是否有输入验证（白名单）
- 是否使用eval/exec

---

## 3. 路径穿越检查指导

### Python 的文件操作特点
- **open()**：直接使用用户输入危险
- **Path**：resolve() 可以规范化路径
- **os.path**：需要验证路径

### 搜索关键代码

```bash
# 搜索文件操作
grep -rn "open\|read\|write\|Pnclude="*.py" -A 10 -B 5

# 搜索路径操作
grep -rn "os\.path\.join\|Path\(" --include="*.py" -A 10 -B 5
```

### AI分析要点

```python
# ❌ 危险：直接使用用户输入
filename = request.args.get('file')
with open(f'/uploads/{filename}', 'r') as f:
    content = f.read()

# ✅ 安全：路径验证
from pathlib import Path

filename = request.args.get('file')
base_path = Path('/uploads').resolve()
file_path = (base_path / filename).resolve()

if not str(file_path).startswith(str(base_path)):
    raise ValueError("Path traversal detected")

with open(file_path, 'r') as f:
    content = f.read()

# ✅ 安全：文件名白名单
import re
if not re.match(r'^[a-zA-Z0-9_.-]+$', filename):
    raise ValueError("Invalid filename")
```

### 检查要点

- 是否使用Path.resolve()规范化
- 是否验证路径在允许目录内
- 是否过滤../
- 是否验证文件名格式

---

## 4. 模板注入（SSTI）检查指导

### Python 的模板引擎特点
- **Jinja2**：render_template_string() 危险
- **Django**：Template() 需检查
- **Mako**：Template() 危险

### 搜索关键代码

```bash
# 搜索模板渲染
grep -rn "render_template_string\|Template\(" --include="*.py" -A 10 -B 5

# 搜索Jinja2
grep -rn "from jinja2 import\|import jinja2" --include="*.py" -A 10 -B 5
```

### AI分析要点

```python
# ❌ 危险：用户输入作为模板
from flask der_template_string

template = request.args.get('template')
render_template_string(template)

# ✅ 安全：使用预定义模板
from flask import render_template
render_template('user.html', name=user_name)

# ✅ 安全：使用沙箱环境
from jinja2.sandbox import SandboxedEnvironment
env = SandboxedEnvironment()
template = env.from_string(template_string)
```

---

## 5. 反序列化漏洞检查指导

### Python 的序列化特点
- **pickle**：pickle.loads() 可执行任意代码
- **yaml**：yaml.load() 危险，yaml.safe_load() 安全
- **json**：json.loads() 安全

### 搜索关键代码

```bash
# 搜索反序列化
grep -rn "pickle\.loads\|yaml\.load\|marshal\.loads" --include="*.py" -A 10 -B 5
```

### AI分析要点

```python
# ❌ 危险：pickle反序列化用户输入
import pick = request.get_data()
obj = pickle.loads(data)  # 可执行任意代码

# ❌ 危险：yaml.load
import yaml
data = request.get_data()
obj = yaml.load(data)  # 可执行任意代码

# ✅ 安全：使用json
import json
data = request.get_data()
obj = json.loads(data)

# ✅ 安全：yaml.safe_load
import yaml
obj = yaml.safe_load(data)
```

---

## 6. XSS检查指导

### Python Web框架的输出特点
- **Flask**：默认自动转义（Jinja2）
- **Django**：默认自动转义
- **直接输出**：需要手动转义

### 搜索关键代码

```bash
# 搜索直接输出
grep -rn "return.*request\|Response\(" --include="*.py" -A 10 -B 5

# 搜索safe标记
grep -rn "safe\|mark_safe\|Markup" --include="*.py" -A 5 -B 5
```

### AI分析要点

```python
# ❌ 危险：直接输出HTML
from flask import Response

@app.route('/hello')
def hello():
    name = request.args.get('name')
    return Response(f'<h1>Hello {name}</h1>')

# ✅ 安全：使用模板（自动转义）
from flask import render_template

@app.route('/hello')
def hello():
    name = request.args.get('name')
    return render_template('hello.html', name=name)

# ❌ 危险：Django mark_safe
from django.utils.safestring import mark_safe

def view(request):
    user_input = request.GET.get('input')
    return HttpResponse(mark_safe(user_input))

# ✅ 安全：使用模板
def view(request):
    user_input = request.GET.get('input')
    return render(request, 'template.html', {'input': user_input})
```

---

## 7. SSRF检查指导

### Python 的 HTTP 请求特点
- **requests**：requests.get/post
- **urllib**：urllib.request.urlopen
- **httpx**：httpx.get/post

### 搜索关键代码

```bash
# 搜索HTTP请求
grep -rn "requests\.get\|requests\.post\|urllib\|httpx" --include="*.py" -A 10 -B 5
```

### AI分析要点

```python
# ❌ 危险：直接请求用户输入的URL
import requests
url = requests.get('url')
response = requests.get(url)

# ✅ 安全：URL验证
import requests
from urllib.parse import urlparse
import ipaddress

def is_safe_url(url):
    parsed = urlparse(url)
    
    # 只允许http/https
    if parsed.scheme not in ['http', 'https']:
        return False
    
    # 解析IP
    try:
        ip = ipaddress.ip_address(parsed.hostname)
        # 禁止内网IP
        if ip.is_private or ip.is_loopback or ip.is_link_local:
            return False
    except:
        pass
    
    return True

url = request.args.get('url')
if not is_safe_url(url):
    raise ValueError("Invalid URL")
response = requests.get(url)
```

---

## 检查清单

- [ ] SQL执行是否使用参数化查询
- [ ] 命令执行是否使用列表形式 + shell=False
- [ ] 文件操作是否验证路径
-render_template_string
- [ ] 是否使用pickle/yaml.load反序列化用户输入
- [ ] 是否使用eval/exec
- [ ] 所有用户输入是否有验证
- [ ] HTTP请求是否验证URL
