# Java语言 - 认证授权检查指导

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
# 查找所有API端点
grep -rn "@RequestMapping\|@GetMapping\|@PostMapping\|@PutMapping\|@DeleteMapping" --include="*.java" -A 10 -B 3

# 查找权限注解
grep -rn "@PreAuthorize\|@Secured\|@RolesAllowed" --include="*.java" -A 3 -B 3

# 查找拦截器配置
grep -rn "HandlerInterceptor\|WebMvcConfigurer\|addInterceptors" --include="*.java" -A 10 -B 5

# 查找Security配置
grep -rn "SecurityConfig\|WebSecurityConfigurerAdapter\|SecurityFilterChain" --include="*.java" -A 20 -B 5
```

### AI分析要点

**第1步：列出所有API端点**

读取所有带有`@RequestMapping`等注解的方法，记录：
- 接口路径
- HTTP方法
- 是否有权限注解

**第2步：检查权限保护**

对每个API端点，检查：
```java
// ✅ 有方法级权限注解
@GetMapping("/admin/users")
@PreAuthorize("hasRole('ADMIN')")
public List<User> getUsers() {
    return userService.list();
}

// ❌ 无权限注解（需要检查是否有全局拦截器）
@GetMapping("/admin/users")
public List<User> getUsers() {
    return userService.list();
}
```

**第3步：检查全局拦截器**

如果方法没有权限注解，查找是否有全局拦截器：
```java
// ✅ 有全局拦截器
@Configuration
public class SecurityConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/**")
                .excludePathPatterns("/login", "/register");
    }
}
```

**第4步：检查白名单**

确认公开接口（登录、注册等）是否在白名单中。

### 检查要点

- 所有 API 是否有权限注解或全局拦截器
- 是否有白名单（登录、注册等公开接口）
- 拦截器是否正确配置路径
- 管理接口是否有额外的权限验证

### 修复示例

```java
// ❌ 危险：无权限验证
@GetMapping("/admin/users")
public List<User> getUsers() {
    return userService.list();
}

// ✅ 安全：方法级权限
@GetMapping("/admin/users")
@PreAuthorize("hasRole('ADMIN')")
public List<User> getUsers() {
    return userService.list();
}

// ✅ 安全：全局拦截器
@Configuration
public class SecurityConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/**")
                .excludePathPatterns("/login", "/register");
    }
}
```

---

## 2. 越权访问检查指导

### 业务风险
用户可以访问/修改其他用户的数据。

### 搜索关键代码

```bash
# 查找根据ID查询的方法
grep -rn "getById\|findById\|selectById\|get.*ById" --include="*.java" -A 15 -B 5

# 查找更新/删除方法
grep -rn "updateById\|deleteById\|update.*ById\|delete.*ById" --include="*.java" -A 15 -B 5

# 查找所有权检查
grep -rn "checkOwner\|verifyOwner\|belongsTo\|getUserId" --include="*.java" -A 10 -B 5

# 查找SQL中的user_id条件
grep -rn "WHERE.*user_id\|AND.*user_id" --include="*.java" --include="*.xml" -A 3 -B 3
```

### AI分析要点

**第1步：识别数据查询方法**

找到所有根据ID查询/更新/删除数据的方法。

**第2步：检查所有权验证**

读取完整方法实现，检查：
```java
// ❌ 危险：未验证所有权
@GetMapping("/order/npublic Order getOrder(@PathVariable Long id) {
    return orderService.getById(id);  // 任何人都能查
}

// ✅ 安全：验证所有权
@GetMapping("/order/{id}")
public Order getOrder(@PathVariable Long id, @RequestHeader("userId") Long userId) {
    Order order = orderService.getById(id);
    if (!order.getUserId().equals(userId)) {
        throw new ForbiddenException("无权访问");
    }
    return order;
}

// ✅ 更好：SQL中加入用户条件
public Order getOrder(Long id, Long userId) {
    return orderMapper.selectOne(
        new QueryWrapper<Order>()
            .eq("id", id)
            .eq("user_id", use  );
}
```

**第3步：追踪userId的使用**

检查userId是否：
- 从认证上下文获取（而非用户输入）
- 加入到SQL查询条件中
- 在业务逻辑中验证

### 检查要点

- 查询/更新/删除是否验证数据属于当前用户
- 是否在 SQL 中加入 `WHERE user_id = ?`
- 是否有统一的权限检查 AOP
- userId 是否从可信来源获取（Session/JWT）

### 修复示例

```java
// ❌ 危险：未验证所有权
@GetMapping("/order/{id}")
public Order getOrder(@PathVariable Long id) {
    return orderService.getById(id);
}

// ✅ 安全：验证所有权
@GetMapping("/order/{id}")
public Order getOrder(@PathVariable Long id) {
    Long currentUserId = SecurityContextHolder.getContext()
        .getAuthentication().getUserId();
    
    Order order = orderService.getById(id);
    if (!order.getUserId()currentUserId)) {
        throw new ForbiddenException("无权访问");
    }
    return order;
}

// ✅ 更好：SQL中加入用户条件
public Order getOrder(Long id, Long userId) {
    return orderMapper.selectOne(
        new QueryWrapper<Order>()
            .eq("id", id)
            .eq("user_id", userId)
    );
}

// ✅ 最佳：使用AOP统一处理
@Aspect
@Component
public class OwnershipAspect {
    @Around("@annotation(CheckOwnership)")
    public Object checkOwnership(ProceedingJoinPoint pjp) throws Throwable {
        Long resourceId = (Long) pjp.getArgs()[0];
        Long currentUserId = getCurrentUserId();
        
        if (!isOwner(resourceId, currentUserId)) {
            throw new ForbiddenException("无权访问");
        }
        
        return pjp.proceed();
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
grep -rn "JWT\|Jwts\|jjwt\|io.jsonwebtoken" --include="*.java" -A 10 -B 5

# 查找密钥
grep -rn "secret\|SECRET_KEY\|signingKey" --include="*.java" --include="*.properties" --include="*.yml" -A 3 -B 3

# 查找token解析
grep -rn "parseClaimsJws\|parseToken\|verifyToken" --include="*.java" -A 10 -B 5
```

### AI分析要点

**检查密钥管理**：
```java
// ❌ 危险：硬编码密钥
String secret = "my-secret-key";

// ✅ 安全变量
@Value("${jwt.secret}")
private String secret;

// ✅ 更安全：从密钥管理服务获取
String secret = vaultClient.getSecret("jwt-secret");
```

**检查token验证**：
```java
// ✅ 安全：完整验证
public Claims parseToken(String token) {
    return Jwts.parser()
        .setSigningKey(secret)
        .parseClaimsJws(token)
        .getBody();  // 自动验证签名和过期时间
}

// ❌ 危险：不验证签名
public Claims parseToken(String token) {
    String[] parts = token.split("\\.");
    String payload = new String(Base64.getDecoder().decode(parts[1]));
    return new ObjectMapper().readValue(payload, Claims.class);
}
```

### 检查要点

- 密钥是否硬编码（应该用环境变量）
- 是否验证签名
- 是否验证过期时间
- 是否使用强算以上）
- 密钥长度是否足够（至少256位）

### 修复示例

```java
// ❌ 危险：硬编码密钥
public class JwtUtil {
    private static final String SECRET = "my-secret-key";
    
    public String generateToken(String username) {
        return Jwts.builder()
            .setSubject(username)
            .signWith(SignatureAlgorithm.HS256, SECRET)
            .compact();
    }
}

// ✅ 安全：完整的JWT实现
@Component
public class JwtUtil {
    @Value("${jwt.secret}")
    private String secret;
    
    @Value("${jwt.expiration:3600000}") // 1小时
    private Long expiration;
    
    public String generateToken(String username) {
        Date w Date();
        Date expiryDate = new Date(now.getTime() + expiration);
        
        return Jwts.builder()
            .setSubject(username)
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(SignatureAlgorithm.HS256, secret)
            .compact();
    }
    
    public Claims parseToken(String token) {
        return Jwts.parser()
            .setSigningKey(secret)
            .parseClaimsJws(token)
            .getBody();  // 自动验证签名和过期时间
    }
    
    public boolean validateToken(String token) {
        try {
            parseToken(token);
            return true;
        } catch (JwtException e) {
            return false;
        }
    }
}
```

---

## 4. Session 检查指导

### 业务风险
Session 固定、Session 劫持。

### 搜索关键代码

```bash
# 查找Session使用
grep -rn "getSession\|HttpSession" --include="*.java" -A 10 -B 5

# 查找登录后是否重新生成Session ID
grep -rn "invalidate\|changeSessionId" --include="*.java" -A 5 -B 5

# 查找Session配置
grep -rn "session.*timeout\|session.*cookie" --include="*.properties" --include="*.yml" -A 3 -B 3
```

### AI分析要点

**检查Session ID重新生成**：
```java
// ✅ 安全：登录后重新生成 Session ID
@PostMapping("/login")
public void login(HttpServletRequest request, String username, String password) {
    if (authenticate(username, password)) {
        // 重新生成 Session ID，防止固定攻击
        request.changeSessionId();
        
        HttpSession session = request.getSession();
        session.setAttribute("userId", user.getId());
    }
}

// ❌ 危险：未重新生成Session ID
@PostMapping("/login")
public void login(HttpServletRequest request, String username, String password) {
    if (authenticate(username, password)) {
        HttpSession session = request.getSession();
        session.setAttribute("userId", user.getId());
    }
}
```

### 检查要点

- 登录后是否重新生成 Session ID
- 是否设置 HttpOnly、Secure 标志
- Session 超时时间是否合理（建议30分钟）
- 登出时是否清除 Session

### 修复示例

```java
// ✅ 完整的Session安全配置
@PostMapping("/login")
public ResponseEntity<String> login(HttpServletRequest request, 
                                   @RequestBody LoginRequest loginReq) {
    User user = authenticate(loginReq.getUsername(), loginReq.getPassword());
    if (user == null) {
        return ResponseEntity.status(401).body("认证失败");
    }
    
    // 重新生成 Session ID
    request.changeSessionId();
    
    // 设置Session
    HttpSession session = request.getSession();
    session.setAttribute("userId", user.getId());
    session.setMaxInactiveInterval(1800); // 30分钟
    
    return ResponseEntity.ok("登录成功");
}

@PostMapping("/logout")
public ResponseEntity<String> logout(HttpServletRequest request) {
    HttpSession session = request.getSession(false);
    if (session != null) {
        session.invalidate();
    }
    return ResponseEntity.ok("登出成功");
}

// application.yml配置
server:
  servlet:
    session:
      timeout: 30m
      cookie:
        http-only: true
        secure: true
        same-site: strict
```

---

## 5. 密码存储检查指导

### 业务风险
明文或弱加密存储密码。

### 搜索关键代码

```bash
# 查找密码存储
grep -rn "setPassword\|password.*=\|savePassword" --include="*.java" -A 10 -B 5

# 查找加密方式
grep -rn "BCrypt\|Argon2\|PBKDF2\|MD5\|SHA\|DigestUtils" --include="*.java" -A 5 -B 5

# 查找密码验证
grep -rn "checkPassword\|verifyPassword\|matches.*password" --include="*.java" -A 10 -B 5
```

### AI分析要点

**识别加密算法**：
```java
// ❌ 危险：MD5/SHA1
String password = DigestUtils.md5Hex(rawPassword);
String password = DigestUtils.sha1Hex(rawPassword);

// ❌ 危险：明文存储
user.setPasssword);

// ✅ 安全：BCrypt
BCryptPasswordEncoder encoder = new BCryptPasswordEncoder();
String password = encoder.encode(rawPassword);

// ✅ 安全：Argon2
Argon2PasswordEncoder encoder = new Argon2PasswordEncoder();
String password = encoder.encode(rawPassword);
```

### 检查要点

- 是否使用 BCrypt、Argon2、PBKDF2（推荐）
- 是否使用 MD5、SHA1（不安全）
- 是否加盐
- 盐是否随机生成（不能固定）

### 修复示例

```java
// ❌ 危险：MD5
public void registerUser(String username, String rawPassword) {
    String password = DigestUtils.md5Hex(rawPassword);
    user.setPassword(password);
    userRepository.save(user);
}

// ✅ 安全：BCrypt
@Service
public class UserService {
    private final BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
    
    public void registerUser(String username, String rawPassword) {
        User user = new User();
        user.setUsername(username);
        user.setPassword(passwordEncoder.encode(rawPassword));
        userRepository.save(user);
    }
    
    public boolean authenticate(String username, String rawPassword) {
        User user = userRepository.findByUsername(username);
        if (user == null) {
            return false;
        }
        return passwordEncoder.matches(rawPassword, user.getPassword());
    }
}
```

---

## 检查清单

### 认证检查
- [ ] 所有API是否有权限验证（注解或拦截器）
- [ ] 是否有白名单机制（公开接口）
- [ ] JWT密钥是否硬编码
- [ ] JWT是否验证签名和过期时间
- [ ] Session是否在登录后重新生成ID
- [ ] Session是否设置HttpOnly和Secure
- [ ] 密码是否使用BCrypt/Argon2

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
