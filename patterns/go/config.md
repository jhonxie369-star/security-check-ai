# Go语言 - 配置安全检查

## 调试模式检查

```bash
# 查找调试模式配置
grep -rn "debug.*true\|DEBUG.*true" --include="*.yaml" --include="*.go"

# 查找详细错误输出
grep -rn "gin\.SetMode\|gin\.DebugMode" --include="*.go"
```

## 敏感端点检查

```bash
# 查找管理端点
grep -rn "/admin\|/debug\|/actuator\|/metrics" --include="*.go"

# 查找pprof（性能分析端点）
grep -rn "pprof\|_ \"net/http/pprof\"" --include="*.go"
```

## 中间件配置检查

```bash
# 查找Redis配置
grep -rn "redis\.NewClient\|redis\.Options" --include="*.go"

# 查找数据库配置
grep -rn "gorm\.Open\|sql\.Open" --include="*.go"
```

## CORS配置检查

```bash
# 查找CORS配置
grep -rn "AllowOrigins\|AllowAllOrigins" --include="*.go"
```

## 修复示例

### 生产环境禁用调试模式

```go
// ❌ 错误：生产环境开启调试
func main() {
    gin.SetMode(gin.DebugMode)  // 泄露详细错误信息
    router := gin.Default()
}

// ✅ 正确：根据环境变量设置
func main() {
    if os.Getenv("ENV") == "production" {
        gin.SetMode(gin.ReleaseMode)
    }
    router := gin.Default()
}
```

### 保护敏感端点

```go
// ❌ 错误：pprof暴露在公网
import _ "net/http/pprof"

func main() {
    router := gin.Default()
    router.Run(":8080")  // pprof可被公网访问
}

// ✅ 正确：pprof只监听内网
import (
    "net/http"
    _ "net/http/pprof"
)

func main() {
    // 业务接口
    router := gin.Default()
    go router.Run(":8080")
    
    // pprof只监听内网
    go http.ListenAndServe("127.0.0.1:6060", nil)
}

// ✅ 更好：pprof需要认证
func main() {
    router := gin.Default()
    
    // pprof需要管理员权限
    admin := router.Group("/debug/pprof")
    admin.Use(RequireRole("admin"))
    {
        admin.GET("/", gin.WrapF(pprof.Index))
        admin.GET("/profile", gin.WrapF(pprof.Profile))
    }
}
```

### 安全的CORS配置

```go
// ❌ 错误：允许所有来源
import "github.com/gin-contrib/cors"

func main() {
    router := gin.Default()
    router.Use(cors.New(cors.Config{
        AllowAllOrigins: true,  // 危险！
    }))
}

// ✅ 正确：指定允许的来源
func main() {
    router := gin.Default()
    router.Use(cors.New(cors.Config{
        AllowOrigins:     []string{"https://example.com"},
        AllowMethods:     []string{"GET", "POST"},
        AllowHeaders:     []string{"Authorization", "Content-Type"},
        AllowCredentials: true,
    }))
}
```

### Redis安全配置

```go
// ❌ 错误：无密码连接
rdb := redis.NewClient(&redis.Options{
    Addr: "localhost:6379",
})

// ✅ 正确：使用密码
rdb := redis.NewClient(&redis.Options{
    Addr:     "localhost:6379",
    Password: os.Getenv("REDIS_PASSWORD"),
    DB:       0,
})
```

### 错误处理不泄露信息

```go
// ❌ 错误：返回详细错误信息
func handler(c *gin.Context) {
    err := doSomething()
    if err != nil {
        c.JSON(500, gin.H{"error": err.Error()})  // 可能泄露敏感信息
    }
}

// ✅ 正确：返回通用错误，详细信息记录日志
func handler(c *gin.Context) {
    err := doSomething()
    if err != nil {
        log.Error("Operation failed", "error", err)  // 记录到日志
        c.JSON(500, gin.H{"error": "Internal server error"})  // 通用错误
    }
}
```

### 安全响应头

```go
// ✅ 添加安全响应头
func SecurityHeaders() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("X-Frame-Options", "DENY")
        c.Header("X-Content-Type-Options", "nosniff")
        c.Header("X-XSS-Protection", "1; mode=block")
        c.Header("Content-Security-Policy", "default-src 'self'")
        c.Header("Strict-Transport-Security", "max-age=31536000")
        c.Next()
    }
}

router.Use(SecurityHeaders())
```

### 限制请求体大小

```go
// ✅ 防止大文件攻击
func main() {
    router := gin.Default()
    router.MaxMultipartMemory = 8 << 20  // 8 MB
    
    // 或使用中间件
    router.Use(func(c *gin.Context) {
        c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, 10<<20)  // 10MB
        c.Next()
    })
}
```

## 检查清单

- [ ] 生产环境禁用调试模式
- [ ] 敏感端点有认证保护
- [ ] CORS不允许所有来源
- [ ] Redis/数据库使用密码
- [ ] 错误信息不泄露敏感信息
- [ ] 配置了安全响应头
- [ ] 限制了请求体大小
