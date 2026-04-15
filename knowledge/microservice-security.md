# 微服务安全

> 微服务架构下的服务间调用安全问题

## 1. 服务间认证

### 缺少认证
```bash
# 搜索HTTP客户端调用
grep -rn "HttpClient\|RestTemplate\|WebClient\|requests\.get\|requests\.post\|http\.Get\|http\.Post\|axios" --include="*.java" --include="*.py" --include="*.go" --include="*.js" -A 10 -B 5

# 检查是否设置认证头
grep -rn "Authorization\|X-API-Key\|X-Service-Token" --include="*.java" --include="*.py" --include="*.go" --include="*.js"
```

**风险**：
- 服务间调用无认证，内部接口可被直接访问
- 使用硬编码的API Key
- Token无过期时间

**防护**：
```java
// ❌ 危险：无认证
RestTemplate restTemplate = new RestTemplate();
String result = restTemplate.getForObject("http://order-service/api/orders", String.class);

// ✅ 安全：JWT认证
RestTemplate restTemplate = new RestTemplate();
HttpHeaders headers = new HttpHeaders();
headers.set("Authorization", "Bearer " + jwtToken);
HttpEntity<String> entity = new HttpEntity<>(headers);
String result = restTemplate.exchange("http://order-service/api/orders", HttpMethod.GET, entity, String.class);

// ✅ 更好：mTLS双向认证
// 配置客户端证书
SSLContext sslContext = SSLContextBuilder.create()
    .loadKeyMaterial(keyStore, keyPassword)
    .loadTrustMaterial(trustStore, null)
    .build();
```

## 2. TLS证书验证

### 禁用证书验证
```bash
# Java: 检查是否禁用证书验证
grep -rn "TrustManager\|trustAllCerts\|setHostnameVerifier" --include="*.java" -A 5 -B 5

# Python: 检查verify=False
grep -rn "verify=False\|verify = False" --include="*.py" -A 3 -B 3

# Go: 检查InsecureSkipVerify
grep -rn "InsecureSkipVerify.*true" --include="*.go" -A 3 -B 3

# Node.js: 检查rejectUnauthorized
grep -rn "rejectUnauthorized.*false" --include="*.js" --include="*.ts" -A 3 -B 3
```

**风险**：
- 禁用证书验证导致中间人攻击
- 开发环境配置泄露到生产环境

**防护**：
```python
# ❌ 危险：禁用证书验证
import requests
response = requests.get('https://api.example.com', verify=False)

# ✅ 安全：启用证书验证
response = requests.get('https://api.example.com', verify=True)

# ✅ 更好：指定CA证书
response = requests.get('https://api.example.com', verify='/path/to/ca-bundle.crt')
```

```go
// ❌ 危险：跳过证书验证
tr := &http.Transport{
    TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
}
client := &http.Client{Transport: tr}

// ✅ 安全：验证证书
client := &http.Client{}
```

## 3. 内部接口暴露

### 未限制访问来源
```bash
# 搜索内部接口
grep -rn "@RequestMapping\|@GetMapping\|@PostMapping\|@RestController" --include="*.java" -A 5 -B 2

# 搜索路由定义
grep -rn "app\.get\|app\.post\|router\.get\|router\.post" --include="*.js" --include="*.ts" -A 5 -B 2

# 搜索Flask路由
grep -rn "@app\.route\|@bp\.route" --include="*.py" -A 5 -B 2

# 搜索Gin路由
grep -rn "router\.GET\|router\.POST\|r\.GET\|r\.POST" --include="*.go" -A 5 -B 2
```

**风险**：
- 内部接口可被外部直接访问
- 管理接口未限制IP
- 调试接口暴露在生产环境

**防护**：
```java
// ❌ 危险：内部接口无限制
@RestController
@RequestMapping("/internal")
public class InternalController {
    @GetMapping("/users")
    public List<User> getAllUsers() {
        return userService.findAll();
    }
}

// ✅ 安全：限制访问来源
@RestController
@RequestMapping("/internal")
public class InternalController {
    
    @GetMapping("/users")
    public List<User> getAllUsers(HttpServletRequest request) {
        String clientIp = getClientIp(request);
        if (!isInternalIp(clientIp)) {
            throw new ForbiddenException("Access denied");
        }
        return userService.findAll();
    }
    
    private boolean isInternalIp(String ip) {
        // 检查是否为内网IP
        return ip.startsWith("10.") || 
               ip.startsWith("172.16.") || 
               ip.startsWith("192.168.");
    }
}

// ✅ 更好：使用API网关限制
// 在API网关层配置内部接口不对外暴露
```

## 4. 超时配置

### 缺少超时设置
```bash
# 搜索HTTP客户端配置
grep -rn "RestTemplate\|HttpClient\|OkHttpClient\|requests\|http\.Client\|axios" --include="*.java" --include="*.py" --include="*.go" --include="*.js" -A 10 -B 5

# 搜索超时配置
grep -rn "timeout\|Timeout\|connectTimeout\|readTimeout" --include="*.java" --include="*.py" --include="*.go" --include="*.js" --include="*.properties" --include="*.yml"
```

**风险**：
- 无超时导致请求hang住
- 连接池耗尽
- 级联故障

**防护**：
```java
// ❌ 危险：无超时配置
RestTemplate restTemplate = new RestTemplate();

// ✅ 安全：设置超时
HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
factory.setConnectTimeout(5000);  // 连接超时5秒
factory.setReadTimeout(10000);    // 读取超时10秒
RestTemplate restTemplate = new RestTemplate(factory);
```

```python
# ❌ 危险：无超时
response = requests.get('https://api.example.com')

# ✅ 安全：设置超时
response = requests.get('https://api.example.com', timeout=10)

# ✅ 更好：分别设置连接和读取超时
response = requests.get('https://api.example.com', timeout=(5, 10))
```

```go
// ❌ 危险：无超时
client := &http.Client{}

// ✅ 安全：设置超时
client := &http.Client{
    Timeout: 10 * time.Second,
}
```

## 5. 权限传# 用户身份丢失
```bash
# 搜索Token传递
grep -rn "Authorization\|X-User-Id\|X-User-Token" --include="*.java" --include="*.py" --include="*.go" --include="*.js" -A 5 -B 5

# 搜索请求头设置
grep -rn "setHeader\|addHeader\|headers\[" --include="*.java" --include="*.py" --include="*.go" --include="*.js" -A 3 -B 3
```

**风险**：
- 服务A验证了用户身份，但调用服务B时未传递
- 服务B无法知道最终用户是谁
- 导致越权访问

**防护**：
```java
// ❌ 危险：未传递用户身份
@Service
public class OrderService {
    public void createOrder(OrderRequest request) {
        // 调用库存服务，但未传递用户信息
        restTemplate.postForObject("http://inventory-service/api/deduct", request, Void.class);
    }
}

// ✅ 安全：传递用户Token
@Service
public class OrderService {
    public void createOrder(OrderRequest request, String userToken) {
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + userToken);
        HttpEntity<OrderRequest> entity = new HttpEntity<>(request, headers);
        restTemplate.postForObject("http://inventory-service/api/deduct", entity, Void.class);
    }
}
```

## 6. 服务发现安全

### 未认证的服务注册
```bash
# 搜索服务注册配置
grep -rn "eureka\|consul\|nacos\|etcd" --include="*.properties" --include="*.yml" --include="*.yaml" -A 5 -B 5

# 搜索服务发现客户端
grep -rn "@EnableDiscoveryClient\|@EnableEurekaClient" --include="*.java"
```

**风险**：
- 恶意服务可以注册到服务中心
- 服务被劫持

**防护**：
- 服务注册需要认证
- 使用服务白名单
- 启用ACL控制

## 检查清单

### 必查项（Critical）
- [ ] 服务间调用是否有认证
- [ ] 是否禁用了TLS证书验证
- [ ] 内部接口是否限制访问来源
- [ ] HTTP客户端是否设置超时

### 建议检查（High）
- [ ] 是否传递用户身份
- [ ] 是否使用硬编码的API Key
- [ ] 服务注册是否需要认证
- [ ] 是否有熔断降级机制
