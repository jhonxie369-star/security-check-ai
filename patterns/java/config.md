# Java语言 - 配置安全检查指导

## ⚠️ 核心原则

**本文件提供配置安全检查思路，主要使用 Grep 和文件检查。**

---

## 1. 数据库配置检查指导

### 检查思路（Grep）

```bash
# 查找数据库配置
grep -rn "spring.datasource\|jdbc" --include="*.properties" --include="*.yml"

# 检查是否有弱密码
grep -rn "password.*=.*123\|password.*=.*admin\|password.*=.*root" --include="*.properties"
```

### 检查要点

- 数据库密码是否为弱密码（123456、admin、root）
- 是否使用生产环境配置
- 是否暴露数据库端口到公网

### 修复示例

```properties
# ❌ 危险：弱密码
spring.datasource.password=123456

# ✅ 安全：强密码 + 环境变量
spring.datasource.password=${DB_PASSWORD}
```

---

## 2. Redis 配置检查指导

### 检查思路（Grep）

```bash
# 查找 Redis 配置
grep -rn "spring.redis\|redis.host" --include="*.properties" --include="*.yml"

# 检查是否有密码
grep -rn "spring.redis.password" --include="*.properties"
```

### 检查要点

- Redis 是否设置密码
- 是否绑定到 127.0.0.1（不暴露到公网）
- 是否禁用危险命令（FLUSHALL、KEYS）

### 修复示例

```properties
# ❌ 危险：无密码
spring.redis.host=localhost
spring.redis.port=6379

# ✅ 安全：有密码
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.password=${REDIS_PASSWORD}
```

---

## 3. 日志配置检查指导

### 检查思路（Grep）

```bash
# 查找日志配置
grep -rn "logging.level\|log4j\|logback" --include="*.properties" --include="*.yml" --include="*.xml"

# 检查是否开启 DEBUG
grep -rn "logging.level.*=.*DEBUG" --include="*.properties"
```

### 检查要点

- 生产环境是否开启 DEBUG 日志
- 日志文件是否有大小限制
- 是否记录敏感信息

### 修复示例

```properties
# ❌ 危险：生产环境 DEBUG
logging.level.root=DEBUG

# ✅ 安全：生产环境 INFO
logging.level.root=INFO
logging.level.com.example=INFO
```

---

## 4. CORS 配置检查指导

### 检查思路（Grep）

```bash
# 查找 CORS 配置
grep -rn "addCorsMappings\|@CrossOrigin\|Access-Control" --include="*.java"

# 检查是否允许所有域名
grep -rn "allowedOrigins.*\\*\|setAllowedOrigins.*\\*" --include="*.java"
```

### 检查要点

- 是否允许所有域名（`*`）
- 是否允许携带凭证（`allowCredentials`）
- 是否限制允许的方法

### 修复示例

```java
// ❌ 危险：允许所有域名
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowCredentials(true);
    }
}

// ✅ 安全：白名单
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("https://example.com", "https://www.example.com")
                .allowedMethods("GET", "POST")
                .allowCredentials(true);
    }
}
```

---

## 5. 文件上传配置检查指导

### 检查思路（Grep）

```bash
# 查找文件上传配置
grep -rn "spring.servlet.multipart\|max-file-size" --include="*.properties" --include="*.yml"
```

### 检查要点

- 是否限制文件大小
- 是否限制文件类型
- 上传目录是否可执行

### 修复示例

```properties
# ❌ 危险：无限制
spring.servlet.multipart.enabled=true

# ✅ 安全：限制大小
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=10MB
```

---

## 6. Actuator 配置检查指导

### 检查思路（Grep）

```bash
# 查找 Actuator 配置
grep -rn "management.endpoints\|actuator" --include="*.properties" --include="*.yml"

# 检查是否暴露所有端点
grep -rn "management.endpoints.web.exposure.include.*\\*" --include="*.properties"
```

### 检查要点

- 是否暴露所有端点（`*`）
- 是否有认证保护
- 是否暴露敏感端点（env、heapdump）

### 修复示例

```properties
# ❌ 危险：暴露所有端点
management.endpoints.web.exposure.include=*

# ✅ 安全：只暴露必要端点
management.endpoints.web.exposure.include=health,info
management.endpoint.health.show-details=when-authorized
```

---

## 7. Session 配置检查指导

### 检查思路（Grep）

```bash
# 查找 Session 配置
grep -rn "server.servlet.session\|session.timeout" --include="*.properties" --include="*.yml"
```

### 检查要点

- Session 超时时间是否合理（不要太长）
- 是否设置 HttpOnly、Secure 标志
- 是否使用 Redis 存储 Session

### 修复示例

```properties
# ❌ 危险：超时时间过长
server.servlet.session.timeout=24h

# ✅ 安全：合理超时
server.servlet.session.timeout=30m
server.servlet.session.cookie.http-only=true
server.servlet.session.cookie.secure=true
```

---

## 8. 错误页面配置检查指导

### 检查思路（Grep）

```bash
# 查找错误页面配置
grep -rn "server.error\|error-page" --include="*.properties" --include="*.yml"

# 检查是否暴露堆栈
grep -rn "server.error.include-stacktrace" --include="*.properties"
```

### 检查要点

- 是否暴露堆栈信息
- 是否有自定义错误页面
- 是否暴露异常详情

### 修复示例

```properties
# ❌ 危险：暴露堆栈
server.error.include-stacktrace=always

# ✅ 安全：不暴露
server.error.include-stacktrace=never
server.error.include-message=never
```

---

## 💡 灵活扩展示例

### 发现项目使用 Swagger

```bash
# 检查 Swagger 是否在生产环境开启
grep -rn "springfox\|swagger\|springdoc" --include="*.properties" --include="*.yml"
```

### 发现项目使用消息队列

```bash
# 检查 RabbitMQ/Kafka 配置
grep -rn "spring.rabbitmq\|spring.kafka" --include="*.properties"
```

---

## 检查清单

### 必须检查的配置文件
- [ ] application.properties / application.yml
- [ ] application-prod.properties（生产环境）
- [ ] logback.xml / log4j2.xml
- [ ] .env 文件

### 必须检查的配置项
- [ ] 数据库密码不是弱密码
- [ ] Redis 设置了密码
- [ ] 生产环境日志级别是 INFO
- [ ] CORS 不允许所有域名
- [ ] 文件上传有大小限制
- [ ] Actuator 不暴露敏感端点
- [ ] Session 超时时间合理
- [ ] 错误页面不暴露堆栈
