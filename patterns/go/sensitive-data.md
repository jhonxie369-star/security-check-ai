# Go语言 - 敏感信息泄露检查指导

## ⚠️ 核心原则

**本文件提供敏感信息检查思路，不是固定规则。要根据项目特点灵活查找。**

---

## 1. 硬编码密钥检查指导

### 业务风险
密钥泄露导致加密失效、系统被攻破。

### 搜索关键代码

```bash
# 搜索硬编码密钥
grep -rn "SECRET.*=.*\"\|password.*=.*\"\|key.*=.*\"" --include="*.go" -A 3 -B 3

# 搜索硬编码密码
grep -rn "admin123\|password123\|123456" --include="*.go" -A 3 -B 3

# 搜索API密钥
grep -rn "api_key.*=\|apiKey.*=\|accessKey.*=" --include="*.go" -A 3 -B 3

# 搜索配置文件中的密钥
grep -rn "password\|secret\|key" --include="*.yaml" --include="*.yml" --include="*.json" -A 2 -B 2
```

### AI分析要点

**识别真实的硬编码 vs 测试代码**：

```go
// ❌ 危险：硬编码密钥
const DBPassword = "MyPassword123"
var JWTSecret = "secret_key_123"

// ✅ 安全：环境变量
import "os"

var DBPassword = os.Getenv("DB_PASSWORD")
var JWTSecret = os.Getenv("JWT_SECRET")

// 启动时检查
func init() {
    if DBPassword == "" {
        log.Fatal("DB_PASSWORD not set")
    }
    if JWTSecret == "" {
        log.Fatal("JWT_SECRET not set")
    }
}

// ✅ 更好：使用配置库
import "github.com/spf13/viper"

func loadConfig() {
    viper.SetConfigFile(".env")
    viper.AutomaticEnv()
    viper.ReadInConfig()
    
    dbPassword := viper.GetString("DB_PASSWORD")
    if dbPassword == "" {
        log.Fatal("DB_PASSWORD not set")
    }
}
```

### 检查要点

- [ ] 是否有硬编码的密码、密钥、Token
- [ ] 是否使用环境变量或配置文件
- [ ] 配置文件是否加密
- [ ] 是否区分测试代码和生产代码

### 修复示例

```go
// ❌ 危险：硬编码
const SECRET_KEY = "my-secret-key-123456"

// ✅ 安全：环境变量
var secretKey = os.Getenv("JWT_SECRET")

// ✅ 更好：从密钥管理服务获取
import (
    "github.com/aws/aws-sdk-go/aws/session"
    "github.com/aws/aws-sdk-go/service/secretsmanager"
)

func getSecret(secretName string) (string, error) {
    sess := session.Must(session.NewSession())
    svc := secretsmanager.New(sess)
    
    input := &secretsmanager.GetSecretValueInput{
        SecretId: aws.String(secretName),
    }
    
    result, err := svc.GetSecretValue(input)
    if err != nil {
        return "", err
    }
    
    return *result.SecretString, nil
}
```

---

## 2. 日志泄露检查指导

### 业务风险
日志中记录敏感信息导致泄露。

### 搜索关键代码

```bash
# 搜索明显的日志泄露
grep -rn "log.*password\|log.*token\|log.*secret\|Printf.*password" --include="*.go" -A 5 -B 5

# 搜索打印用户对象
grep -rn "log.*user\|log.*request\|Printf.*user" --include="*.go" -A 5 -B 5

# 搜索打印完整请求体
grep -rn "log.*Body\|Printf.*Body" --include="*.go" -A 5 -B 5
```

### AI分析要点

**第1步：搜索日志调用**

找到所有日志记录的位置。

**第2步：读取完整上下文**

对每个日志调用，读取前后20-30行代码，分析：
- 记录的变量名是否包含敏感词（password、token、secret）
- 是否记录了完整的用户对象（可能包含密码）
- 是否记录了完整的请求体（可能包含敏感信息）

**第3步：追踪变量来源**

如果记录的是对象（如user、request），需要查看对象定义：
```go
// ❌ 危险：记录包含密码的用户对象
log.Printf("User login: %+v", user)  // user对象包含Password字段

// 需要查看User结构体定义
type User struct {
    Username string
 word string  // 敏感字段
    Phone    string
}
```

### 检查要点

- [ ] 是否记录密码、Token、密钥
- [ ] 是否记录完整的用户对象（可能包含密码）
- [ ] 是否记录完整的请求体（可能包含敏感信息）
- [ ] 是否对敏感信息进行脱敏

### 修复示例

```go
// ❌ 危险：记录密码
log.Printf("User login: username=%s, password=%s", username, password)

// ❌ 危险：记录完整用户对象
log.Printf("User login: %+v", user)  // user包含Password

// ✅ 安全：只记录必要信息
log.Printf("User login: userID=%d, username=%s", user.ID, user.Username)

// ✅ 安全：脱敏处理
func maskPhone(phone string) string {
    if len(phone) < 11 {
        return phone
    }
    return phone[:3] + "****" + phone[7:]
}

log.Printf("User login: username=%s, phone=%s", 
    user.Username, 
    maskPhone(user.Phone))

// ✅ 更好：重写String方法
type User struct {
    ID       int64
    Username string
    Password string
    Phone    string
}

func (u User) String() string {
    return fmt.Sprintf("User{ID:%d, Username:%s, Phone:%s}", 
        u.ID, u.Username, maskPhone(u.Phone))
    // 不包含Password
}
```

---

## 3. 响应泄露检查指导

### 业务风险
API响应包含敏感信息。

### 搜索关键代码

```bash
# 搜索User结构体的密码字段
grep -rn "Password\|Token\|Secret" --include="*.go" -A 2 -B 2

# 检查是否有json:"-"标签
grep -rn "json:\"-\"" --include="*.go" -A 2 -B 2

# 搜索返回User对象的接口
grep -rn "c.JSON.*user\|JSON.*User" --include="*.go" -A 5 -B 10
```

### AI分析要点

**第1步：找到实体结构体**

搜索所有结构体（User、Order等），检查是否有敏感字段。

**第2步：检查序列化配置**

对每个敏感字段，检查是否有`json:"-"`标签：
```go
// ❌ 危险：密码字段未标记
type User struct {
    ID       int64  `json:"id"`
    Username string `json:"username"`
    Password string `json:"password"`  // 没有json:"-"
}

// ✅ 安全：使用json:"-"
type User struct {
    ID       int64  `json:"id"`
    Username string `json:"username"`
    Password string `json:"-"`
}
```

**第3步：检查API返回**

搜索返回User对象的接口，确认是否直接返回：
```go
// ❌ 危险：直接返回User对象
func getUser(c *gin.Context) {
    var user User
    db.First(&usec.Param("id"))
    c.JSON(200, user)  // 可能包含Password
}

// ✅ 安全：使用DTO
type UserDTO struct {
    ID       int64  `json:"id"`
    Username string `json:"username"`
}

func getUser(c *gin.Context) {
    var user User
    db.First(&user, c.Param("id"))
    
    dto := UserDTO{
        ID:       user.ID,
        Username: user.Username,
    }
    c.JSON(200, dto)
}
```

### 检查要点

- [ ] 敏感字段是否有json:"-"标签
- [ ] 是否使用DTO而非直接返回实体
- [ ] DTO是否包含敏感字段
- [ ] 是否有全局序列化配置

### 修复示例

```go
// ❌ 危险：实体类直接暴露
type User struct {
    ID       int64  `json:"id"`
    Username string `json:"username"`
    Password string `json:"password"`
    Email    string `json:"email"`
}

func getUser(c *gin.Context) {
    var user User
    db.First(&user, c.Param("id"))
    c.JSON(200, user)
}

// ✅ 安全：使用json:"-"
type User struct {
    ID       int64  `json:"id"`
    Username string `json:"username"`
    Password string `json:"-"`
    Email    string `json:"email"`
}

// ✅ 更好：使用DTO
type UserDTO struct {
    ID       int64  `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
    // 不包含Password
}

func toDTO(user *User) *UserDTO {
    return &UserDTO{
        ID:       user.ID,
        Username: user.Username,
        Email:    user.Email,
    }
}

func getUser(c *gin.Context) {
    var user User
    db.First(&user, c.Param("id"))
    c.JSON(200, toDTO(&user))
}
```

---

## 4. 错误信息泄露检查指导

### 业务风险
错误信息暴露系统内部细节。

### 搜索关键代码

```bash
# 搜索异常处理
grep -rn "c.JSON.*error\|c.String.*err\|JSON.*err" --include="*.go" -A 10 -B 5

# 搜索错误响应
grep -rn "err.Error()\|fmt.Errorf" --include="*.go" -A 5 -B 5
```

### AI分析要点

**检查异常处理方式**：

```go
// ❌ 危险：暴露数据库错误
func getUser(c *gin.Context) {
    var user User
    err := db.First(&user, c.Param("id")).or
    if err != nil {
        c.JSON(500, gin.H{"error": err.Error()})  // 暴露SQL错误
        return
    }
    c.JSON(200, user)
}

// ✅ 安全：通用错误信息
func getUser(c *gin.Context) {
    var user User
    err := db.First(&user, c.Param("id")).Error
    if err != nil {
        log.Printf("Database error: %v", err)  // 记录到日志
        c.JSON(500, gin.H{"error": "Internal server error"})  // 通用错误
        return
    }
    c.JSON(200, user)
}

// ✅ 更好：区分错误类型
func getUser(c *gin.Context) {
    var user User
    err := db.First(&user, c.Param("id")).Error
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            c.JSON(404, gin.H{"error": "User not found"})
        } else {
            log.Printf("Database error: %v", err)
            c.JSON(500, gin.H{"error": "Internal server error"})
        }
        return
    }
    c.JSON(200, user)
}
```

### 检查要点

- [ ] 是否有统一的异常处理
- [ ] 是否暴露堆栈信息
- [ ] 是否暴露SQL错误信息
- [ ] 是否区分业务异常和系统异常

---

## 5. 配置文件泄露检查指导

### 业务风险
配置文件包含敏感信息被泄露。

### 搜索关键代码

```bash
# 搜索配置文件
find . -name "*.yaml" -o -name "*.yml" -o -name "*.json" -o -name ".env"

# 检查配置文件内容
grep -rn "password\|secret\|key\|token" --include="*.yaml" --include="*.yml" -A 2检查是否在git中
git ls-files | grep -E "\.yaml$|\.yml$|\.env$"
```

### 检查要点

- [ ] 配置文件是否包含明文密码
- [ ] 敏感配置文件是否在.gitignore中
- [ ] 是否使用配置加密
- [ ] 是否使用环境变量

### 修复示例

```yaml
# ❌ 危险：明文密码
database:
  password: admin123
jwt:
  secret: my-secret-key

# ✅ 安全：环境变量
database:
  password: ${DB_PASSWORD}
jwt:
  secret: ${JWT_SECRET}
```

**.gitignore配置**：
```
# 敏感配置文件
config.yaml
config-prod.yaml
.env
*.key
*.pem
```

---

## 6. 注释泄露检查指导

### 业务风险
代码注释包含敏感信息。

### 搜索关键代码

```bash
# 搜索注释中的密码
grep -rn "//.*password\|//.*密码\|//.*secret" --include="*.go" -A 2 -B 2

# 搜索TODO中的敏感信息
grep -rn "TODO.*password\|FIXME.*secret" --include="*.go" -A 2 -B 2
```

### 检查要点

- [ ] 注释中是否包含密码、密钥
- [ ] TODO/FIXME中是否有敏感信息
- [ ] 是否有调试用的临时密码

---

## 7. 备份文件泄露检查指导

### 业务风险
备份文件、临时文件包含敏感信息。

### 搜索关键代码

```bash
# 搜索备份文件
find . -name "*.bak" -o -name "*.backup" -o -name "*.old" -o -name "*~"

# 搜索临时文件
find . -name "*.tmp" -o -name "*.temp" -o -name "*.swp"

# 检查是否在git中
git ls-files | grep -E "\.bak$|\.backup$|\.old$|\.tmp$"
```

### 检查要点

- [ ] 是否有备份文件
- [ ] 备份文件是否在.gitignore中
- [ ] 是否有临时文件包含敏感信息

---

## 检查清单

### 代码检查
- [ ] 无硬编码密钥
- [ ] 日志不记录敏感信息
- [ ] API响应不包含敏感字段
- [ ] 异常处理不暴露堆栈信息

### 配置检查
- [ ] 配置文件使用环境变量
- [ ] 敏感配置文件在.gitignore中
- [ ] 生产环境配置已加密

### 文件检查
- [ ] 注释中无敏感信息
- [ ] 无备份文件泄露
- [ ] 无临时文件包含敏感信息

### 序列化检查
- [ ] 敏感字段有json:"-"标签
- [ ] 使用DTO而非直接返回实体
- [ ] 有全局序列化配置
