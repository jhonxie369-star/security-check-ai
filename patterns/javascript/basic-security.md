# JavaScript/TypeScript 基础代码安全检查

## ⚠️ 核心原则

**本文件提供检查思路和常见模式，不是固定规则。**

**工作流程**：
1. 识别项目特点（框架、输入点、危险操作）
2. 使用grep搜索关键代码位置
3. 读取完整上下文（前后50-100行）
4. AI深度分析，结合源码判断真实风险

---

## 1. SQL注入检查指导

### JavaScript/TypeScript 的 SQL 执行特点
- **原生SQL**：db.query() 字符串拼接危险，参数化安全
- **Sequelize ORM**：Model.findOne() 安全，sequelize.query() 需检查
- **TypeORM**：query() 需要参数绑定
- **Knex.js**：raw() 需检查

### 搜索关键代码

```bash
# 搜索SQL执行点
grep -rn "\.query\|\.execute\|\.run\|sequelize\.query" --include="*.js" --include="*.ts" -A 10 -B 5

# 搜索字符串拼接
grep -rn "\`.*SELECT\|\+.*SELECT\|concat.*SELECT" --include="*.js" --include="*.ts" -A 5 -B 5

# 搜索模板字符串
grep -rn "\${.*}" --include="*.js" --include="*.ts" | grep -i "select\|insert\|update\|delete" -A 5 -B 5
```

### AI分析要点

**第1步：识别参数来源**
```javascript
// 用户输入（高风险）
const username = req.query.username;
const userId = req.body.id;
const data = req.params.data;

// 配置文件（低风险）
const table = config.tableName;

// 常量（无风险）
const status = 'active';
```

**第2步：检查是否使用安全API**
```javascript
// ✅ 安全：参数化查询
const sql = 'SELECT * FROM users WHERE username = ?';
db.query(sql, [username]);

// ✅ 安全：Sequelize ORM
User.findOne({ where: { username } });

// ✅ 安全：TypeORM参数绑定
connection.query('SELECT * FROM users WHERE id = $1', [userId]);

// ❌ 危险：模板字符串拼接
const sql = `SELECT * FROM users WHERE username = '${username}'`;
db.query(sql);

// ❌ 危险：字符串拼接
const sql = 'SELECT * FROM users WHERE username = ' + username;
db.query(sql);

// ❌ 危险：Sequelize raw query拼接
sequelize.query(`SELECT * FROM users WHERE username = '${username}'`);
```

**第3步：检查输入验证**
```javascript
// ✅ 有验证
if (!/^[a-zA-Z0-9_]+$/.test(username)) {
    throw new Error("Invalid username");
}

// ✅ 使用白名单
const allowedFields = ['username', 'email', 'status'];
if (!allowedFields.includes(fieldName)) {
    throw new Error("Invalid field");
}
```

### 风险判断

| 场景 | 风险等级 |
|------|---------|
| 用户输入 + 字符串拼接 + 无验证 | 严重 |
| 用户输入 + 字符串拼接 + 有验证 | 高危 |
| 用户输入 + 参数化查询 | 安全 |
| 配置文件 + 字符串拼接 | 低危 |

### 修复示例

```javascript
// ❌ 危险：字符串拼接
async function getUser(username) {
    const sql = `SELECT * FROM users WHERE username = '${username}'`;
    return await db.query(sql);
}

// ✅ 安全：参数化查询
async function getUser(username) {
    const sql = 'SELECT * FROM users WHERE username = ?';
    return await db.query(sql, [username]);
}

// ✅ Sequelize ORM
const user = await User.findOne({ where: { username } });

// ❌ 危险：动态字段名
async function search(fieldName, value) {
    const sql = `SELECT * FROM users WHERE ${fieldName} = ?`;
    return await db.query(sql, [value]);
}

// ✅ 安全：字段名白名单
async function search(fieldName, value) {
    const allowedFields = ['username', 'email', 'status'];
    if (!allowedFields.includes(fieldName)) {
        throw new Error("Invalid field");
    }
    const sql = `SELECT * FROM users WHERE ${fieldName} = ?`;
    return await db.query(sql, [value]);
}
```

---

## 2. 命令注入检查指导

### JavaScript/TypeScript 的命令执行特点
- **child_process.exec()**：字符串拼接危险
- **child_process.spawn()**：数组参数安全
- **child_process.execFile()**：相对安全

### 搜索关键代码

```bash
# 搜索命令执行
grep -rn "child_process\|exec\|sp\|execSync\|spawnSync" --include="*.js" --include="*.ts" -A 10 -B 5

# 搜索shell=true
grep -rn "shell.*true" --include="*.js" --include="*.ts" -A 5 -B 5
```

### AI分析要点

```javascript
// ❌ 危险：exec字符串拼接
const { exec } = require('child_process');
const host = req.query.host;
exec(`ping -c 4 ${host}`, (error, stdout) => {
    console.log(stdout);
});

// ❌ 危险：spawn with shell
const { spawn } = require('child_process');
spawn(`ping -c 4 ${host}`, { shell: true });

// ✅ 安全：spawn数组参数
const { spawn } = require('child_process');
const host = req.query.host;
if (!/^[a-zA-Z0-9.-]+$/.test(host)) {
    throw new Error('Invalid host');
}
spawn('ping', ['-c', '4', host]);

// ✅ 安全：execFile
const { execFile } = require('child_process');
execFile('ping', ['-c', '4', host], (error, stdout) => {
    console.log(stdout);
});
```

### 检查要点

- 是否使用数组形式参数
- 是否设置shell=false
- 是否有输入验证（白名单）
- 是否使用execFile代替exec

---

## 3. 路径穿越检查指导

### JavaScript/TypeScript 的文件操作特点
- **fs.readFile()**：直接使用用户输入危险
- **path.join()**：需要验证路径
- **path.normalize()**：可以规范化路径

### 搜索关键代码

```bash
# 搜索文件操作
grep -rn "fs\.readFile\|fs\.writeFile\|fs\.createReadStream" --include="*.js" --include="*.ts" -A 10 -B 5

# 搜索路径操作
grep -rn "path\.join\|path\.resolve" --include="*.js" --include="*.ts" -A 10 -B 5
```

### AI分析要点

```javascript
// ❌ 危险：直接使用用户输入
const filename = req.query.file;
fs.readFile(filename, (err, data) => {
    res.send(data);
});

// ✅ 安全：路径验证
const path = require('path');
const filename = req.query.file;

// 清理路径
const cleanPath = path.normalize(filename).replace(/^(\.\.(\/|\\|$))+/, '');

// 检查..
if (cleanPath.includes('..')) {
    throw new Error("Path traversal detected");
}

// 限制在指定目录
const baseDir = '/var/www/uploads';
const fullPath = path.join(baseDir, cleanPath);

// 验证最终路径在baseDir内
if (!fullPath.startsWith(baseDir)) {
    throw new Error("Invalid path");
}

fs.readFile(fullPath, (err, data) => {
    res.send(data);
});

// ✅ 安全：文件名白名单
if (!/^[a-zA-Z0-9_.-]+$/.test(filename)) {
    throw new Error("Invalid filename");
}
```

### 检查要点

- 是否使用path.normalize()规范化
- 是否验证路径在允许目录内
- 是否过滤../
- 是否验证文件名格式

---

## 4. XSS检查指导

### JavaScript/TypeScript 的输出特点
- **innerHTML**：危险
- **textContent**：安全
- **React**：默认转义，dangerouslySetInnerHTML危险

### 搜索关键代码

```bash
# 搜索innerHTML
grep -rn "innerHTML\|dangerouslySetInnerHTML" --include="*.js" --include="*.ts" --include="*.jsx" --include="*.tsx" -A 5 -B 5

# 搜索document.write
grep -rn "document\.write" --include="*.js" --include="*.ts" -A 5 -B 5
```

### AI分析要点

```javascript
// ❌ 危险：innerHTML
function displayMessage(message) {
    document.getElementById('msg').innerHTML = message;
}

// ✅ 安全：textContent
function displayMessage(message) {
    document.getElementById('msg').textContent = message;
}

// React
// ❌ 危险：dangerouslySetInnerHTML
function Comment({ text }) {
    return <div dangerouslySetInnerHTML={{ __html: text }} />;
}

// ✅ 安全：直接渲染
function Comment({ text }) {
    return <div>{text}</div>;
}

// ✅ 如必须渲染HTML，使用DOMPurify
import DOMPurify from 'dompurify';
function Comment({ html }) {
    const clean = DOMPurify.sanitize(html);
    return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

---

## 5. SSRF检查指导

### JavaScript/TypeScript 的 HTTP 请求特点
- **axios**：axios.get/post
- **fetch**：fetch()
- **http/https**：http.request()

### 搜索关键代码

```bash
# 搜索HTTP请求
grep -rn "axios\|fetch\|http\.request\|https\.request" --include="*.js" --include="*.ts" -A 10 -B 5
```

### AI分析要点

```javascript
// ❌ 危险：直接请求用户输入的URL
const axios = require('axios');
const url = req.query.url;
const response = await axios.get(url);

// ✅ 安全：URL验证
const axios = require('axios');
const { URL } = require('url');
const dns = require('dns').promises;

async function isPrivateIP(hostname) {
    try {
        const addresses = await dns.resolve4(hostname);
        for (const ip of addresses) {
            const parts = ip.split('.').map(Number);
            if (parts[0] === 10 ||
                (parts[0] === 172 && parts[1] >= 16 && parts[1] <= 31) ||
                (parts[0] === 192 && parts[1] === 168) ||
                parts[0] === 127) {
                return true;
            }
        }
        return false;
    } catch {
        return true;
    }
}

const urlStr = req.query.url;
const url = new URL(urlStr);

// 只允许http/https
if (!['http:', 'https:'].includes(url.protocol)) {
    throw new Error('Invalid protocol');
}

// 禁止内网IP
if (await isPrivateIP(url.hostname)) {
    throw new Error('Private IP not allowed');
}

const response = await axios.get(urlStr, {
    timeout: 5000,
    maxRedirects: 0
});
```

---

## 6. 原型污染检查指导

### 搜索关键代码

```bash
grep -rn "Object\.assign\|\.merge\|\.extend\|lodash" --include="*.js" --include="*.ts" -A 10 -B 5
```

### AI要点

```javascript
// ❌ 危险：直接合并用户输入
const _ = require('lodash');
const user = {};
_.merge(user, req.body);  // 攻击：{"__proto__":{"isAdmin":true}}

// ✅ 安全：使用白名单
const allowedFields = ['name', 'email', 'age'];
const user = {};

for (const field of allowedFields) {
    if (req.body[field] !== undefined) {
        user[field] = req.body[field];
    }
}

// ✅ 或使用Object.create(null)
const user = Object.create(null);
Object.assign(user, req.body);
```

---

## 7. eval和动态代码执行检查指导

### 搜索关键代码

```bash
grep -rn "eval\|Function\|setTimeout.*\"\|setInterval.*\"" --include="*.js" --include="*.ts" -A 10 -B 5
```

### AI分析要点

``pt
// ❌ 危险：使用eval
function calculate(expression) {
    return eval(expression);
}

// ✅ 安全：使用安全的表达式解析器
const math = require('mathjs');
function calculate(expression) {
    return math.evaluate(expression);
}

// ❌ 危险：setTimeout字符串
setTimeout("console.log('hello')", 1000);

// ✅ 安全：setTimeout函数
setTimeout(() => console.log('hello'), 1000);
```

---

## 8. 不安全的随机数检查指导

### 搜索关键代码

```bash
grep -rn "Math\.random" --include="*.js" --include="*.ts" -A 5 -B 5
```

### AI分析要点

```javascript
// ❌ 危险：Math.random生成token
function generateToken() {
    ret Math.random().toString(36).substring(2);
}

// ✅ 安全：使用crypto
const crypto = require('crypto');
function generateToken() {
    return crypto.randomBytes(32).toString('hex');
}
```

---

## 检查清单

- [ ] SQL执行是否使用参数化查询
- [ ] 命令执行是否使用数组形式参数
- [ ] 文件操作是否验证路径
- [ ] 是否使用innerHTML/dangerouslySetInnerHTML
- [ ] 是否使用eval/Function
- [ ] 所有用户输入是否有验证
- [ ] HTTP请求是否验证URL
- [ ] 是否直接合并用户输入到对象
- [ ] 是否使用Math.random生成敏感数据
