# Java语言 - 敏感信息泄露检查指导

## ⚠️ 核心原则

**本文件提供敏感信息检查思路，不是固定规则。要根据项目特点灵活查找。**

---

## 1. 硬编码密钥检查指导

### 业务风险
密钥泄露导致加密失效、系统被攻破。

### 搜索关键代码

```bash
# 搜索硬编码密钥
grep -rn "SECRET.*=.*\"\|password.*=.*\"\|key.*=.*\"" --include="*.java" -A 3 -B 3

# 搜索硬编码密码
grep -rn "admin123\|password123\|123456" --include="*.java" -A 3 -B 3

# 搜索API密钥
grep -rn "api_key.*=\|apiKey.*=\|accessKey.*=" --include="*.java" -A 3 -B 3
```

### AI分析要点

**识别真实的硬编码 vs 测试代码**：

```java
// ❌ 危险：硬编码密钥
public class JwtUtil {
    private static final String SECRET_KEY = "1234567890123456";
    private static final String DB_PASSWORD = "admin123";
}

// ✅ 安全：环境变量
@Component
public class JwtUtil {
    @Value("${jwt.secret}")
    private String secretKey;
    
    @Value("${spring.datasource.password}")
    private String dbPassword;
}

// ✅ 更安全：配置加密（Jasypt）
@Component
public class JwtUtil {
    @Value("${encrypted.jwt.secret}")
    private String secretKey;
}
```

**检查配置文件**：
```bash
# 搜索配置文件中的密钥
grep -rn "password\|secret\|key" --include="*.properties" --include="*.yml" -A 2 -B 2
```

### 检查要点

- [ ] 是否有硬编码的密码、密钥、Token
- [ ] 是否使用环境变量或配置文件
- [ ] 配置文件是否加密
- [ ] 是否区分测试代码和生产代码

### 修复示例

```java
// ❌ 危险：硬编码
private static final String SECRET_KEY = "my-secret-key-123456";

// ✅ 安全：环境变量
@Value("${jwt.secret}")
private String secretKey;

// ✅ 更好：从密钥管理服务获取
@Autowired
private VaultClient vaultClient;

public String getSecretKey() {
    return vaultClient.getSecret("jwt-secret");
}
```

---

## 2. 日志泄露检查指导

### 业务风险
日志中记录敏感信息导致泄露。

### 搜索关键代码

```bash
# 搜索明显的日志泄露
grep -rn "log.*password\|log.*token\|log.*secret\|logger.*password" --include="*.java" -A 5 -B 5

# 搜索打印用户对象
grep -rn "log.*user\|log.*request\|logger.*user" --include="*.java" -A 5 -B 5

# 搜索打印完整请求体
grep -rn "log.*getBody\|log.*RequestBody" --include="*.java" -A 5 -B 5
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
```java
// ❌ 危险：记录包含密码的用户对象
logger.info("User login: {}", user);  // user对象包含password字段

// 需要查看User类定义
public class User {
    private String username;
    private String password;  // 敏感字段
    private String phone;
    // ...
}
```

### 检查要点

- [ ] 是否记录密码、Token、密钥
- [ ] 是否记录完整的用户对象（可能包含密码）
- [ ] 是否记录完整的请求体（可能包含敏感信息）
- [ ] 是否对敏感信息进行脱敏

### 修复示例

```java
// ❌ 危险：记录密码
logger.info("User login: username={}, password={}", username, password);

// ❌ 危险：记录完整用户对象
logger.info("User login: {}", user);  // user包含password

// ✅ 安全：只记录必要信息
logger.info("User login: userId={}, username={}", user.getId(), user.getUsername());

// ✅ 安全：脱敏处理
logger.info("User login: username={}, phone={}", 
    user.getUsername(), 
    maskPhone(user.getPhone()));

private String maskPhone(String phone) {
    if (phone == null || phone.length() < 11) {
        return phone;
    }
    return phone.replaceAll("(\\d{3})\\d{4}(\\d{4})", "$1****$2");
}

// ✅ 更好：重写toString方法
public class User {
    private String username;
    private String password;
    private String phone;
    
    @Override
    public String toString() {
        return "User{" +
            "username='" + username + '\'' +
            ", phone='" + maskPhone(phone) + '\'' +
            // 不包含password
            '}';
    }
}
```

---

## 3. 响应泄露检查指导

### 业务风险
API响应包含敏感信息。

### 搜索关键代码

```bash
# 搜索User类的密码字段
grep -rn "private.*password\|private.*token" --include="*.java" -A 2 -B 2

# 检查是否有@JsonIgnore注解
grep -rn "@JsonIgnore" --include="*.java" -A 2 -B 2

# 搜索返回User对象的接口
grep -rn "return.*user\|return.*User" --include="*.java" -A 5 -B 10
```

### AI分析要点

**第1步：找到实体类**

搜索所有实体类（User、Order等），检查是否有敏感字段。

**第2步：检查序列化配置**

对每个敏感字段，检查是否有`@JsonIgnore`注解：
```java
// ❌ 危险：密码字段未标记
public class User {
    private Long id;
    private String username;
    private String password;  // 没有@JsonIgnore
}

// ✅ 安全：使用@JsonIgnore
public class User {
    private Long id;
    private String username;
    
    @JsonIgnore
    private String password;
}
```

**第3步：检查API返回**

搜索返回User对象的接口，确认是否直接返回：
```java
// ❌ 危险：直接返回User对象
@GetMapping("/user/{id}")
public User getUser(@PathVariable Long id) {
    return userService.getById(id);  // 可能包含password
}

// ✅ 安全：使用DTO
@GetMapping("/user/{id}")
public UserDTO getUser(@PathVariable Long id) {
    User user = userService.getById(id);
    return UserDTO.fromUser(user);  // DTO不包含password
}
```

### 检查要点

- [ ] 敏感字段是否有@JsonIgnore注解
- [ ] 是否使用DTO而非直接返回实体
- [ ] DTO是否包含敏感字段
- [ ] 是否有全局序列化配置

### 修复示例

```java
// ❌ 危险：实体类直接暴露
@Entity
public class User {
    private Long id;
    private String username;
    private String password;
    private String email;
}

@GetMapping("/user/{id}")
public User getUser(@PathVariable Long id) {
    return userService.getById(id);
}

// ✅ 安全：使用@JsonIgnore
@Entity
public class User {
    private Long id;
    private String username;
    
    @JsonIgnore
    private String password;
    
    private String email;
}

// ✅ 更好：使用DTO
@Data
public class UserDTO {
    private Long id;
    private String username;
    private String email;
    // 不包含password
    
    public static UserDTO fromUser(User user) {
        UserDTO dto = new UserDTO();
        dto.setId(user.getId());
        dto.setUsername(user.getUsername());
        dto.setEmail(user.getEmail());
        return dto;
    }
}

@GetMapping("/user/{id}")
public UserDTO getUser(@PathVariable Long id) {
    User user = userService.getById(id);
    return UserDTO.fromUser(user);
}
```

---

## 4. 错误信息泄露检查指导

### 业务风险
错误信息暴露系统内部细节。

### 搜索关键代码

```bash
# 搜索异常处理
grep -rtionHandler\|catch.*Exception" --include="*.java" -A 10 -B 5

# 搜索错误响应
grep -rn "printStackTrace\|e.getMessage\|e.toString" --include="*.java" -A 5 -B 5
```

### AI分析要点

**检查异常处理方式**：

```java
// ❌ 危险：暴露堆栈信息
@ExceptionHandler(Exception.class)
public ResponseEntity<String> handleException(Exception e) {
    return ResponseEntity.status(500).body(e.getMessage());
}

// ❌ 危险：打印堆栈
catch (Exception e) {
    e.printStackTrace();
    throw e;
}

// ✅ 安全：统一错误处理
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception e) {
        // 记录详细错误到日志
        logger.error("System error", e);
        
        // 返回通用错误信息
        ErrorResponse response = new ErrorResponse();
        response.setCode("SYSTEM_ERROR");
        response.setMessage("系统错误，请稍后重试");
        return ResponseEntity.status(500).body(response);
    }
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException e) {
        ErrorResponse response = new ErrorResponse();
        response.setCode(e.getCode());
        response.setMessage(e.getMessage());
        return ResponseEntity.status(400).body(response);
    }
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
find . -name "*.properties" -o -name "*.yml" -o -name "*.yaml" -o -name ".env"

# 检查配置文件内容
grep -rn "password\|secret\|key\|token" --include="*.properties" --include="*.yml" -A 2 -B 2

# 检查是否在git中
git ls-files | grep -E "\.properties$|\.yml$|\.env$"
```

### 检查要点

- [ ] 配置文件是否包含明文密码
- [ ] 敏感配置文件是否在.gitignore中
- [ ] 是否使用配置加密（Jasypt）
- [ ] 是否使用环境变量

### 修复示例

```properties
# ❌ 危险：明文密码
spring.datasource.password=admin123
jwt.secret=my-secret-key

# ✅ 安全：环境变量
spring.datasource.password=${DB_PASSWORD}
jwt.secret=${JWT_SECRET}

# ✅ 更好：配置加密（Jasypt）
spring.datasource.password=ENC(encrypted_password_here)
jwt.secret=ENC(encrypted_secret_here)
```

**.gitignore配置**：
```
# 敏感配置文件
application-prod.properties
application-local.properties
.env
```

---

## 6. 注释泄露检查指导

### 业务风险
代码注释包含敏感信息。

### 搜索关键代码

```bash
# 搜索注释中的密码
grep -rn "//.*password\|//.*密码\|//.*ret" --include="*.java" -A 2 -B 2

# 搜索TODO中的敏感信息
grep -rn "TODO.*password\|FIXME.*secret" --include="*.java" -A 2 -B 2
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
- [ ] 备份文件是否在.gitig] 是否有临时文件包含敏感信息

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
- [ ] 敏感字段有@JsonIgnore
- [ ] 使用DTO而非直接返回实体
- [ ] 有全局序列化配置
