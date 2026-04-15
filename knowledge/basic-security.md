# 基础代码安全问题

> 纯代码层面的安全漏洞检查

## 1. 内存安全

### 缓冲区溢出
```bash
# C/C++: 不安全的函数
grep -rn "strcpy\|strcat\|sprintf\|gets\|scanf" --include="*.c" --include="*.cpp"
```

**安全替代**：
- `strcpy` → `strncpy` / `strlcpy`
- `strcat` → `strncat` / `strlcat`
- `sprintf` → `snprintf`
- `gets` → `fgets`

### 整数溢出
```bash
grep -rn "malloc.*\*\|new.*\*\|calloc" --include="*.c" --include="*.cpp"
```

**风险场景**：`malloc(size * count)` - 乘法溢出

### Use-After-Free
```bash
grep -rn "free\|delete" --include="*.c" --include="*.cpp"
```

**风险模式**：释放后未置NULL、双重释放、释放后使用

### 空指针解引用
```bash
grep -rn "\*.*=\|->.*=" --include="*.c" --include="*.cpp"
```

## 2. 注入攻击

### 命令注入
```bash
grep -rn "exec\|system\|popen\|Runtime\.exec\|subprocess\|exec\.Command" --include="*.go" --include="*.java" --include="*.py" --include="*.c"
```

**防护**：避免shell执行，使用参数数组

### SQL注入
```bash
grep -rn "SELECT.*+\|INSERT.*+\|UPDATE.*+\|DELETE.*+" --include="*.go" --include="*.java" --include="*.py"
```

**防护**：参数化查询

### LDAP注入
```bash
grep -rn "ldap\.Search\|LdapContext" --include="*.go" --include="*.java"
```

### XXE（XML外部实体）
```bash
grep -rn "xml\.Unmarshal\|DocumentBuilder\|etree\.parse" --include="*.go" --include="*.java" --include="*.py"
```

**防护**：禁用外部实体

## 3. 路径安全

### 目录穿越
```bash
grep -rn "\.\.\|filepath\.Join.*request\|Paths\.get.*request\|os\.path\.join" --include="*.go" --include="*.java" --include="*.py"
```

**防护**：清理路径、检查`..`、验证在允许目录内

### 符号链接攻击
```bash
grep -rn "os\.Open\|open\|fopen" --include="*.go" --include="*.c" --include="*.py"
```

**防护**：使用 `O_NOFOLLOW` 标志

### TOCTOU（检查后使用）
```bash
grep -rn "os\.Stat.*os\.Open\|access.*open" --include="*.go" --include="*.c" --include="*.py"
```

**防护**：原子操作，直接打开并处理错误

## 4. 加密问题

### 弱加密算法
```bash
grep -rn "MD5\|SHA1\|DES\|RC4\|ECB" --include="*.go" --include="*.java" --include="*.py" --include="*.c"
```

**安全替代**：
- 密码：bcrypt, scrypt, Argon2
- 加密：AES-256-GCM
- 哈希：SHA-256, SHA-512

### 不安全的随机数
```bash
grep -rn "rand\(\)\|Random\(\)\|random\.random\|srand" --include="*.go" --include="*.java" --include="*.py" --include="*.c"
```

**安全替代**：
- C: `/dev/urandom`
- Go: `crypto/rand`
- Java: `SecureRandom`
- Python: `secrets`

### 硬编码密钥
```bash
grep -rn "password.*=.*\"\|secret.*=.*\"\|key.*=.*\"" --include="*.go" --include="*.java" --include="*.py"
```

## 5. 反序列化
```bash
grep -rn "pickle\.loads\|yaml\.load\|ObjectInputStream\|unserialize" --include="*.go" --include="*.java" --include="*.py" --include="*.php"
```

**高风险**：
- Python: `pickle.loads`
- Java: `ObjectInputStream`
- YAML: `yaml.load` (非safe_load)

## 6. SSRF（服务端请求伪造）
```bash
grep -rn "http\.Get\|http\.Post\|HttpClient\|requests" --include="*.go" --include="*.java" --include="*.py"
```

**防护**：URL白名单、禁止内网IP、禁止特殊协议

## 7. 资源管理

### 资源未释放
```bash
grep -rn "open\|Open\|fopen\|socket" --include="*.go" --include="*.java" --include="*.py" --include="*.c"
```

**检查**：是否有对应的close

### 无限递归
```bash
grep -rn "func.*{.*\1\|def.*:.*\1" --include="*.go" --include="*.py"
```

**防护**：递归深度限制

### DoS（资源耗尽）
```bash
grep -rn "make\(.*request\|new.*request\|malloc.*input" --include="*.go" --include="*.java" --include="*.c"
```

**风险**：无限制的内存分配、正则表达式DoS

## 8. 并发安全

### 竞态条件
```bash
grep -rn "go func\|thread\|pthread_create" --include="*.go" --include="*.java" --include="*.c"
```

**风险**：共享变量无锁保护、非原子操作

### 死锁
```bash
grep -rn "Lock\|Mutex\|pthread_mutex" --include="*.go" --include="*.java" --include="*.c"
```

**防护**：固定加锁顺序、超时机制

## 9. 日志安全

### 敏感信息泄露
```bash
grep -rn "log\|Log\|print\|printf" --include="*.go" --include="*.java" --include="*.py"
```

**不应记录**：密码、Token、完整卡号

### 日志注入
```bash
grep -rn "log.*request\|Log.*input" --include="*.go" --include="*.java" --include="*.py"
```

**防护**：过滤换行符 `\n` `\r`

## 10. 格式化字符串漏洞
```bash
grep -rn "printf.*%\|sprintf.*%" --include="*.c" --include="*.cpp"
```

**风险**：用户输入作为格式化字符串

## 11. 文件上传安全

### 文件类型验证
```bash
# 搜索文件上传处理
grep -rn "upload\|multipart\|MultipartFile\|FileUpload" --include="*.java" --include="*.py" --include="*.go" --include="*.js" -A 10 -B 5
```

**风险点**：
- 未验证文件类型（通过扩展名伪造）
- 未验证文件内容（MIME类型可伪造）
- 未限制文件大小
- 文件名未过滤（路径穿越）
- 上传目录可执行脚本

### 文件存储位置
```bash
# 检查上传目录配置
grep -rn "upload.*path\|upload.*dir\|file.*path" --include="*.properties" --include="*.yml" --include="*.json"
```

**安全要求**：
- 上传目录不在Web根目录下
- 上传目录禁止执行权限
- 文件名随机化，不使用原始文件名

## 12. CSRF防护

### Token验证
```bash
# 搜索CSRF配置
grep -rn "csrf\|CsrfToken\|_csrf" --include="*.java" --include="*.py" --include="*.js" -A 5 -B 5

# Spring Security CSRF配置
grep -rn "csrf().disable()" --include="*.java"
```

**风险**：
- 关闭了CSRF防护
- 状态变更接口未验证CSRF Token
- GET请求执行了状态变更操作

## 13. 点击劫持防护

### X-Frame-Options检查
```bash
# 检查响应头配置
grep -rn "X-Frame-Options\|frame-options\|frameOptions" --include="*.java" --include="*.py" --include="*.js" --include="*.xml" --include="*.yml"

# Spring Security配置
grep -rn "frameOptions\|headers().frameOptions()" --include="*.java"
```

**安全配置**：
- X-Frame-Options: DENY（完全禁止）
- X-Frame-Options: SAMEORIGIN（同源允许）
- Content-Security-Policy: frame-ancestors 'none'

## 14. CI/CD配置泄露

### 敏感信息泄露
```bash
# 检查CI配置文件中的密钥
grep -rn "password\|secret\|token\|key" .github/workflows/*.yml .gitlab-ci.yml Jenkinsfile

# 检查是否硬编码凭证
grep -rn "AWS_SECRET\|GITHUB_TOKEN\|DOCKER_PASSWORD" .github/workflows/*.yml .gitlab-ci.yml
```

**风险**：CI配置文件中硬编码密钥会被提交到代码仓库

### 不安全的脚本
```bash
# 检查是否下载并执行未知脚本
grep -rn "curl.*|.*sh\|wget.*|.*bash\|curl.*|.*python" .github/workflows/*.yml .gitlab-ci.yml
```

**风险**：下载并执行未知脚本可能引入恶意代码

## 检查优先级

### Critical
1. 缓冲区溢出
2. 命令注入
3. SQL注入
4. 目录穿越
5. 硬编码密钥
6. Use-After-Free

### High
7. 整数溢出
8. 弱加密算法
9. 不安全的随机数
10. 反序列化
11. SSRF
12. 竞态条件

### Medium
13. 空指针解引用
14. 资源未释放
15. 日志注入
16. 符号链接攻击
17. 格式化字符串漏洞
