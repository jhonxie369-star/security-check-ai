# Go语言 - 认证授权检查指导

## ⚠️ 核心原则

**本文件提供认证授权检查思路，不是固定规则。必须理解项目的认证方式，再灵活构造检查。**

**工作流程**：
1. 使用grep发现可疑点
2. 读取完整上下文（前后50-100行）
3. AI深度分析数据流和验证逻辑
4. 结合业务判断真实风险

---

## 1. 未授权访问检查指导

### 业务风险
API 端点缺少权限验证，任何人都可以访问。

### 搜索关键代码

```bash
# 查找所有API端点（net/http）
grep -rn "HandleFunc\|Handle" --include="*.go" -A 10 -B 3

# 查找Gin路由注册
grep -rn "router\.GET\|router\.POST\|router\.PUT\|router\.DELETE\|r\.GET\|r\.POST" --include="*.go" -A 10 -B 3

# 查找Echo路由注册
grep -rn "e\.GET\|e\.POST\|e\.PUT\|e\.DELETE\|echo\.GET" --include="*.go" -A 10 -B 3

# 查找中间件
grep -rn "Use\|middleware\|Middleware" --include="*.go" -A 10 -B 5

# 查找认证中间件
grep -rn "AuthMiddleware\|JWTMiddleware\|auth\|Auth" --include="*.go" -A 15 -B 5

# 查找路由分组
grep -rn "Group\|authorized" --include="*.go" -A 10 -B 5
```

### AI分析要点

**第1步：列出所有API端点**

读取所有路由注册代码，记录：
- 接口路径
- HTTP方法
- 是否有中间件保护

```go
// ✅ 有中间件保护
authorized := r.Group("/admin")
authorized.Use(AuthMiddleware())
{
    authorized.GET("/users", getUsers)
}

// ❌ 无中间件保护（需要检查是否有全局中间件）
r.GET("/admin/users", getUsers)
```

**第2步：检查全局中间件**

如果路由没有显式中间件，查找是否有全局中间件：
```go
// ✅ 有全局中间件
r := gin.Default()
r.Use(AuthMiddleware())

// 白名单
r.GET("/login", login)
r.GET("/register", register)

// 受保护的路由
r.GET("/admin/users", getUsers)
```

**第3步：检查白名单**

确认公开接口（登录、注册等）是否正确配置。

### 检查要点

- [ ] 所有API是否有中间件保护
- [ ] 是否有白名单（登录、注册等公开接口）
- [ ] 中间件是否正确配置路径
- [ ] 管理接口是否有额外的权限验证

### 修复示例

```go
// ❌ 危险：无权限验证
r.GET("/admin/users", getUsers)

// ✅ 安全：使用中间件
authorized := r.Group("/admin")
authorized.Use(AuthMiddleware())
{
    authorized.GET("/users", getUsers)
}

// ✅ 安全：中间件实现
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.JSON(401, gin.H{"error": "Unauthorized"})
            c.Abort()
            return
        }
        
        // 验证 token
        claims, err := parseJWT(token)
        if err != nil {
            c.JSON(401, gin.H{"error": "Invalid token"})
            c.Abort()
            return
        }
        
        c.Set("userID", claims.UserID)
        c.Next()
    }
}

// ✅ 安全：全局中间件 + 白名单
func main() {
    r := gin.Default()
    
    // 公开接口
    r.POST("/login", login)
    r.POST("/register", register)
    
    // 受保护的接口
    authorized := r.Group("/")
    authorized.Use(AuthMiddleware())
    {
        authorized.GET("/profile", getProfile)
        authorized.GET("/orders", getOrders)
    }
    
    r.Run()
}
```

---

## 2. 越权访问检查指导

### 业务风险
用户可以访问/修改其他用户的数据。

### 搜索关键代码

```bash
# 查找根据ID查询的方法
grep -rn "GetByID\|FindByID\|SelectByID\|GetById\|FindById" --include="*.go" -A 15 -B 5

# 查找更新/删除方法
grep -rn "UpdateByID\|DeleteByID\|Update.*ById\|Delete.*ById" --include="*.go" -A 15 -B 5

# 查找所有权检查
grep -rn "CheckOwner\|VerifyOwner\|BelongsTo\|getUserId\|GetUserId" --include="*.go" -A 10 -B 5

# 查找SQL中的user_id条件
grep -rn "WHERE.*user_id\|AND.*user_id\|user_id.*=" --include="*.go" -A 3 -B 3

# 查找GORM查询
grep -rn "db\.Where\|db\.First\|db\.Find" --include="*.go" -A 10 -B 5
```

### AI分析要点

**第1步：识别数据查询方法**

找到所有根据ID查询/更新/删除数据的方法。

**第2步：检查所有权验证**

读取完整方法实现，检查：
```go
// ❌ 危险：未验证所有权
func getOrder(c *gin.Context) {
    orderID := c.Param("id")
    order := orderService.GetByID(orderID)  // 任何人都能查
    c.JSON(200, order)
}

// ✅ 安全：验证所有权
func getOrder(c *gin.Context) {
    orderID := c.Param("id")
    userID := c.GetInt64("userID")  // 从中间件获取
    
    order := orderService.GetByID(orderID)
    if order.UserID != userID {
        c.JSON(403, gin.H{"error": "Forbidden"})
        return
    }
    c.JSON(200, order)
}

// ✅ 更好：SQL中加入用户条件
func GetOrder(orderID, userID int64) (*Order, error) {
    var order Order
    err := db.Where("id = ? AND user_id = ?", orderID, userID).First(&order).Error
    return &order, err
}
```

**第3步：追踪userID的使用**

检查userID是否：
- 从认证上下文获取（而非用户输入）
- 加入到SQL查询条件中
- 在业务逻辑中验证

### 检查要点

- [ ] 查询/更新/删除是否验证数据属于当前用户
- [ ] 是否在SQL中加入 `WHERE user_id = ?`
- [ ] 是否有统一的权限检查中间件
- [ ] userID是否从可信来源获取（Context/JWT）

### 修复示例

```go
// ❌ 危险：未验证所有权
func getOrder(c *gin.Context) {
    orderID := c.Param("id")
    order := orderService.GetByID(orderID)
    c.JSON(200, order)
}

// ✅ 安全：验证所有权
func getOrder(c *gin.Context) {
    orderID := c.Param("id")
    userID, _ := c.Get("userID")  // 从中间件获取
    
    order := orderService.GetByID(orderID)
    if order.UserID != userID.(int64) {
        c.JSON(403, gin.H{"error": "无权访问"})
        return
    }
    c.JSON(200, order)
}

// ✅ 更好：SQL中加入用户条件
func GetOrder(orderID, userID int64) (*Order, error) {
    var order Order
    err := db.Where("id = ? AND user_id = ?", orderID, userID).First(&order).Error
    if err != nil {
        return nil, err
    }
    return &order, nil
}

// ✅ 最佳：使用中间件统一处理
func OwnershipMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        resourceID := c.Param("id")
        userID, _ := c.Get("userID")
        
        if !checkOwnership(resourceID, userID.(int64)) {
            c.JSON(403, gin.H{"error": "无权访问"})
            c.Abort()
            return
        }
        
        c.Next()
    }
}
```

---

## 3. JWT Token 检查指导

### 业务风险
Token 验证不当导致伪造、篡改。

### 搜索关键代码

```bash
# 查找JWT使用
grep -rn "jwt\|JWT\|ParseWithClaims\|jwt-go\|golang-jwt" --include="*.go" -A 10 -B 5

# 查找密钥
grep -rn "secret\|SECRET_KEY\|jwtSecret\|JwtSecret" --include="*.go" --include="*.env" --include="*.yaml" -A 3 -B 3

# 查找token解析
grep -rn "ParseWithClaims\|Parse\|Verify" --include="*.go" -A 10 -B 5
```

### AI分析要点

**检查密钥管理**：
```go
// ❌ 危险：硬编码密钥
var jwtSecret = []byte("my-secret-key")

// ✅ 安全：环境变量
var jwtSecret = []byte(os.Getenv("JWT_SECRET"))

// ✅ 更安全：从配置文件获取
type Config struct {
    JWTSecret string `yaml:"jwt_secret"`
}
```

**检查token验证**：
```go
// ✅ 安全：完整验证
func parseJWT(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        // 验证算法
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return jwtSecret, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }
    
    return nil, errors.New("invalid token")
}

// ❌ 危险：不验证签名
func parseJWT(tokenString string) (*Claims, error) {
    parts := strings.Split(tokenString, ".")
    payload, _ := base64.RawURLEncoding.DecodeString(parts[1])
    
    var claims Claims
    json.Unmarshal(payload, &claims)
    return &claims, nil
}
```

### 检查要点

- [ ] 密钥是否硬编码（应该用环境变量）
- [ ] 是否验证签名
- [ ] 是否验证过期时间
- [ ] 是否使用强算法（HS256以上）
- [ ] 密钥长度是否足够（至少256位）

### 修复示例

```go
// ❌ 危险：硬编码密钥
var jwtSecret = []byte("my-secret-key")

func generateToken(userID int64) (string, error) {
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
        "user_id": userID,
        "exp":     time.Now().Add(time.Hour * 24).Unix(),
    })
    return token.SignedString(jwtSecret)
}

// ✅ 安全：完整的JWT实现
var jwtSecret = []byte(os.Getenv("JWT_SECRET"))

type Claims struct {
    UserID int64 `json:"user_id"`
    jwt.StandardClaims
}

func generateToken(userID int64) (string, error) {
    claims := Claims{
        UserID: userID,
        StandardClaims: jwt.StandardClaims{
            ExpiresAt: time.Now().Add(time.Hour * 24).Unix(),
            IssuedAt:  time.Now().Unix(),
            Issuer:    "my-app",
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

func parseJWT(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        // 验证算法
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method")
        }
        return jwtSecret, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }
    
    return nil, errors.New("invalid token")
}
```

---

## 4. Session 检查指导

### 业务风险
Session 固定、Session 劫持。

### 搜索关键代码

```bash
# 查找Session使用
grep -rn "Session\|session\|sessions" --include="*.go" -A 10 -B 5

# 查找Session库
grep -rn "gorilla/sessions\|gin-contrib/sessions" --include="*.go" -A 10 -B 5

# 查找Session配置
grep -rn "NewStore\|Options\|MaxAge" --include="*.go" -A 10 -B 5
```

### AI分析要点

**检查Session配置**：
```go
// ✅ 安全：Session配置
import (
    "github.com/gin-contrib/sessions"
    "github.com/gin-contrib/sessions/cookie"
)

store := cookie.NewStore([]byte("secret"))
store.Options(sessions.Options{
    Path:     "/",
    MaxAge:   3600,           // 1小时
    HttpOnly: true,           // 防止XSS
    Secure:   true,           // 只在HTTPS传输
    SameSite: http.SameSiteStrictMode,
})

// ❌ 危险：不安全的Session配置
store := cookie.NewStore([]byte("secret"))
store.Options(sessions.Options{
    Path:     "/",
    MaxAge:   86400,          // 24小时太长
    HttpOnly: false,          // 可被JS访问
    Secure:   false,          // HTTP也可传输
})
```

**检查Session重新生成**：
```go
// ✅ 安全：登录后重新生成Session
func login(c *gin.Context) {
    session := sessions.Default(c)
    
    // 清除旧Session
    session.Clear()
    
    // 设置新Session
    session.Set("userID", user.ID)
    session.Save()
}

// ❌ 危险：未重新生成Session
func login(c *gin.Context) {
    session := sessions.Default(c)
    session.Set("userID", user.ID)
    session.Save()
}
```

### 检查要点

- [ ] 登录后是否重新生成Session ID
- [ ] 是否设置HttpOnly、Secure标志
- [ ] Session超时时间是否合理（建议30分钟）
- [ ] 登出时是否清除Session

### 修复示例

```go
// ✅ 完整的Session安全配置
func setupSession(r *gin.Engine) {
    store := cookie.NewStore([]byte(os.Getenv("SESSION_SECRET")))
    store.Options(sessions.Options{
        Path:     "/",
        MaxAge:   1800,  // 30分钟
        HttpOnly: true,
        Secure:   true,
        SameSite: http.SameSiteStrictMode,
    })
    r.Use(sessions.Sessions("mysession", store))
}

func login(c *gin.Context) {
    var req LoginRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": "Invalid request"})
        return
    }
    
    user := authenticate(req.Username, req.Password)
    if user == nil {
        c.JSON(401, gin.H{"error": "认证失败"})
        return
    }
    
    session := sessions.Default(c)
    
    // 清除旧Session
    session.Clear()
    
    // 设置新Session
    session.Set("userID", user.ID)
    session.Save()
    
    c.JSON(200, gin.H{"message": "登录成功"})
}

func logout(c *gin.Context) {
    session := sessions.Default(c)
    session.Clear()
    session.Save()
    c.JSON(200, gin.H{"message": "登出成功"})
}
```

---

## 5. 密码存储检查指导

### 业务风险
明文或弱加密存储密码。

### 搜索关键代码

```bash
# 查找密码存储
grep -rn "Password.*=\|password.*=\|SetPassword" --include="*.go" -A 10 -B 5

# 查找加密方式
grep -rn "bcrypt\|argon2\|md5\|sha\|crypto" --include="*.go" -A 5 -B 5

# 查找密码验证
grep -rn "CompareHashAndPassword\|VerifyPassword\|CheckPassword" --include="*.go" -A 10 -B 5
```

### AI分析要点

**识别加密算法**：
```go
// ❌ 危险：MD5/SHA1
import "crypto/md5"
hash := md5.Sum([]byte(password))

import "crypto/sha1"
hash := sha1.Sum([]byte(password))

// ❌ 危险：明文存储
user.Password = password

// ✅ 安全：bcrypt
import "golang.org/x/crypto/bcrypt"
hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)

// ✅ 安全：argon2
import "golang.org/x/crypto/argon2"
hash := argon2.IDKey([]byte(password), salt, 1, 64*1024, 4, 32)
```

### 检查要点

- [ ] 是否使用bcrypt、argon2（推荐）
- [ ] 是否使用md5、sha1（不安全）
- [ ] 是否加盐
- [ ] 盐是否随机生成（不能固定）

### 修复示例

```go
// ❌ 危险：MD5
import "crypto/md5"

func registerUser(username, password string) error {
    hash := md5.Sum([]byte(password))
    user := &User{
        Username: username,
        Password: hex.EncodeToString(hash[:]),
    }
    return db.Create(user).Error
}

// ✅ 安全：bcrypt
import "golang.org/x/crypto/bcrypt"

func registerUser(username, password string) error {
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    if err != nil {
        return err
    }
    
    user := &User{
        Username: username,
        Password: string(hashedPassword),
    }
    return db.Create(user).Error
}

func authenticate(username, password string) (*User, error) {
    var user User
    if err := db.Where("username = ?", username).First(&user).Error; err != nil {
        return nil, err
    }
    
    if err := bcrypt.CompareHashAndPassword([]byte(user.Password), []byte(password)); err != nil {
        return nil, errors.New("密码错误")
    }
    
    return &user, nil
}
```

---

## 检查清单

### 认证检查
- [ ] 所有API是否有权限验证（中间件）
- [ ] 是否有白名单机制（公开接口）
- [ ] JWT密钥是否硬编码
- [ ] JWT是否验证签名和过期时间
- [ ] Session是否在登录后重新生成ID
- [ ] Session是否设置HttpOnly和Secure
- [ ] 密码是否使用bcrypt/argon2

### 授权检查
- [ ] 数据查询是否验证所有权
- [ ] SQL是否包含user_id条件
- [ ] 是否有统一的权限检查机制
- [ ] 管理接口是否有额外验证

### 配置检查
- [ ] Session超时时间是否合理
- [ ] Cookie是否设置安全标志
- [ ] 是否使用HTTPS
- [ ] 密钥是否从环境变量获取
