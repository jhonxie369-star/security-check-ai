# Go语言 - 基础代码安全检查指导

## ⚠️ 核心原则

**本文件提供检查思路和常见模式，不是固定规则。**

**工作流程**：
1. 识别项目特点（框架、输入点、危险操作）
2. 使用grep搜索关键代码位置
3. 读取完整上下文（前后50-100行）
4. AI深度分析，结合源码判断真实风险

---

## 1. SQL注入检查指导

### Go 的 SQL 执行特点
- **database/sql**：Query/Exec 字符串拼接危险，占位符安全
- **GORM**：Where() 参数化安全，Raw() 需检查
- **sqlx**：与database/sql类似

### 搜索关键代码

```bash
# 搜索SQL执行点
grep -rn "db\.Query\|db\.Exec\|db\.QueryRow\|\.Raw\|\.Exec" --include="*.go" -A 10 -B 5

# 搜索字符串拼接
grep -rn "fmt\.Sprintf.*SELECT\|+.*SELECT" --include="*.go" -A 5 -B 5
```

### AI分析要点

**识别参数来源和使用方式**：

```go
// ❌ 危险：字符串拼接
func GetUser(username string) (*User, error) {
    sql := fmt.Sprintf("SELECT * FROM users WHERE username = '%s'", username)
    row := db.QueryRow(sql)
    // ...
}

// ✅ 安全：占位符
func GetUser(username string) (*User, error) {
    sql := "SELECT * FROM users WHERE username = ?"
    row := db.QueryRow(sql, username)
    // ...
}

// ✅ 安全：GORM
func GetUser(username string) (*User, error) {
    var user User
    db.Where("username = ?", username).First(&user)
    return &user, nil
}

// ❌ 危险：动态字段名
func Search(fieldName, value string) ([]User, error) {
    sql := fmt.Sprintf("SELECT * FROM users WHERE %s = ?", fieldName)
    rows, _ := db.Query(sql, value)
    // ...
}

// ✅ 安全：字段名白名单
func Search(fieldName, value string) ([]User, error) {
    allowedFields := map[string]bool{
        "username": true,
        "email": true,
        "status": true,
    }
    if !allowedFields[fieldName] {
        return nil, errors.New("invalid field")
    }
    sql := fmt.Sprintf("SELECT * FROM users WHERE %s = ?", fieldName)
    rows, _ := db.Query(sql, value)
    // ...
}
```

### 检查要点

- [ ] SQL执行是否使用占位符（?）
- [ ] 动态字段名是否使用白名单
- [ ] 是否有输入验证

---

## 2. 命令注入检查指导

### Go 的命令执行特点
- **exec.Command()**：参数数组安全
- **shell执行**：需要特别注意

### 搜索关键代码

```bash
# 搜索命令执行
grep -rn "exec\.Command\|exec\.CommandContext" --include="*.go" -A 10 -B 5
```

### AI分析要点

```go
// ❌ 危险：通过shell执行
func Ping(host string) error {
    cmd := exec.Command("sh", "-c", fmt.Sprintf("ping -c 4 %s", host))
    return cmd.Run()
}

// ✅ 安全：直接执行 + 验证
func Ping(host string) error {
    // 验证输入
    if !regexp.MustCompile(`^[a-zA-Z0-9.-]+$`).MatchString(host) {
        return errors.New("invalid host")
    }
    
    cmd := exec.Command("ping", "-c", "4", host)
    return cmd.Run()
}
```

---

## 3. 路径穿越检查指导

### 搜索关键代码

```bash
# 搜索文件操作
grep -rn "os\.Open\|ioutil\.ReadFile\|os\.ReadFile" --include="*.go" -A 10 -B 5
```

### AI分析要点

```go
// ❌ 危险：直接使用用户输入
func ReadFile(filename string) ([]byte, error) {
    return os.ReadFile(filepath.Join("/uploads", filename))
}

// ✅ 安全：路径验证
func ReadFile(filename string) ([]byte, error) {
    basePath, _ := filepath.Abs("/uploads")
    fullPath, _ := filepath.Abs(filepath.Join(basePath, filename))
    
    if !strings.HasPrefix(fullPath, basePath) {
        return nil, errors.New("path traversal detected")
    }
    
    return os.ReadFile(fullPath)
}
```

---

## 4. SSRF检查指导

### 搜索关键代码

```bash
# 搜索HTTP请求
grep -rn "http\.Get\|http\.Post\|http\.Client" --include="*.go" -A 10 -B 5
```

### AI分析要点

```go
// ❌ 危险：直接请求用户输入的URL
func FetchURL(url string) ([]byte, error) {
    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    return io.ReadAll(resp.Body)
}

// ✅ 安全：URL验证
func FetchURL(urlStr string) ([]byte, error) {
    u, err := url.Parse(urlStr)
    if err != nil {
        return nil, err
    }
    
    // 只允许http/https
    if u.Scheme != "http" && u.Scheme != "https" {
        return nil, errors.New("invalid protocol")
    }
    
    // 解析IP并检查
    ips, _ := net.LookupIP(u.Hostname())
    for _, ip := range ips {
        if ip.IsPrivate() || ip.IsLoopback() || ip.IsLinkLocalUnicast() {
            return nil, errors.New("private IP not allowed")
        }
    }
    
    resp, err := http.Get(urlStr)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    return io.ReadAll(resp.Body)
}
```

---

## 检查清单

- [ ] SQL执行是否使用占位符
- [ ] 命令执行是否直接调用（不通过shell）
- [ ] 文件操作是否验证路径
- [ ] HTTP请求是否验证URL
- [ ] 所有用户输入是否有验证
