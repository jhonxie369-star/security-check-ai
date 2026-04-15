# 跨服务调用安全检查

适用场景：
- 前端 → 后端
- 服务 A → 服务 B
- Python 服务 → Java 服务
- Node.js 服务 → Go 服务
- 任何跨进程、跨语言的服务调用

## 检查方法

### 1. 识别调用关系
查看项目中的：
- HTTP 客户端调用（axios、requests、HttpClient、http.Client）
- gRPC 调用
- 消息队列（Kafka、RabbitMQ）
- 服务发现配置（Consul、Eureka、Nacos）
- API 网关配置

### 2. 认证机制检查

#### 检查点
```
调用方代码中搜索：
- HTTP Header 设置（Authorization、X-API-Key）
- Token 生成逻辑
- 证书文件引用（.pem、.crt、.key）

被调用方代码中搜索：
- 认证中间件/过滤器
- Token 验证逻辑
- 证书配置
```

#### 常见问题
- [ ] 调用方是否携带认证信息
- [ ] 被调用方是否验证认证信息
- [ ] Token 是否有过期时间
- [ ] 是否使用硬编码的密钥
- [ ] 是否禁用了 TLS 证书验证

### 3. 超时配置检查

#### 检查点
```
搜索关键词：
- timeout
- connectTimeout
- readTimeout
- Timeout
- Duration
```

#### 常见问题
- [ ] HTTP 客户端是否设置超时
- [ ] 超时时间是否合理（建议 5-30 秒）
- [ ] 是否有连接超时和读取超时
- [ ] gRPC 是否设置 deadline

### 4. TLS/SSL 配置检查

#### 检查点
```
搜索关键词：
- InsecureSkipVerify
- verify=False
- rejectUnauthorized: false
- trustAllCerts
- TLSClientConfig
- SSLContext
```

#### 常见问题
- [ ] 是否禁用了证书验证（危险）
- [ ] 是否使用 TLS 1.2+
- [ ] 是否配置了 mTLS（双向认证）
- [ ] 证书文件是否安全存储

### 5. 重试和熔断检查

#### 检查点
```
搜索关键词：
- retry
- circuit breaker
- fallback
- maxRetries
- backoff
```

#### 常见问题
- [ ] 是否有重试机制
- [ ] 重试是否有次数限制
- [ ] 是否有指数退避
- [ ] 是否有熔断器
- [ ] 是否有降级策略

### 6. 请求追踪检查

#### 检查点
```
搜索关键词：
- X-Request-ID
- X-Trace-ID
- X-Service-Name
- requestId
- traceId
```

#### 常见问题
- [ ] 是否传递 Request ID
- [ ] 是否记录调用方信息
- [ ] 是否记录调用日志
- [ ] 日志中是否包含敏感信息

### 7. 服务发现安全检查

#### 检查点
```
搜索关键词：
- consul
- eureka
- nacos
- service discovery
- getService
```

#### 常见问题
- [ ] 服务地址是否验证（防止 SSRF）
- [ ] 是否限制为内网地址
- [ ] 是否有服务白名单

### 8. 内网服务暴露检查

#### 检查点
```
搜索关键词：
- listen
- bind
- 0.0.0.0
- ListenAndServe
- app.run
```

#### 常见问题
- [ ] 内网服务是否监听 0.0.0.0（危险）
- [ ] 是否限制为 127.0.0.1 或内网 IP
- [ ] 是否有网络隔离（VPC、防火墙）

### 9. API 网关配置检查

#### 检查点
查看 API 网关配置文件：
- Nginx 配置
- Kong 配置
- Spring Cloud Gateway
- Traefik 配置

#### 常见问题
- [ ] 是否有限流配置
- [ ] 是否有认证配置
- [ ] 是否有 CORS 配置
- [ ] 是否暴露了内部服务

### 10. 跨语言数据传输检查

#### 检查点
```
搜索关键词：
- JSON.parse
- json.loads
- Jackson
- Gson
- protobuf
```

#### 常见问题
- [ ] 是否验证数据格式
- [ ] 是否有数据大小限制
- [ ] 是否处理了编码问题
- [ ] 是否验证了数据类型

## 检查报告格式

### 调用关系图
```
前端 (React) 
  → API 网关 (Nginx)
    → 用户服务 (Java)
      → 数据库
    → 订单服务 (Go)
      → 支付服务 (Python)
```

### 问题汇总
```
| 调用方 | 被调用方 | 问题 | 严重程度 |
|--------|----------|------|----------|
| 前端 | API网关 | 无 CSRF Token | 🟠 High |
| 订单服务 | 支付服务 | 无认证 | 🔴 Critical |
| 用户服务 | 数据库 | 无超时 | 🟡 Medium |
```

## 通用检查清单

### 认证授权
- [ ] 所有服务间调用是否有认证
- [ ] Token 是否有过期时间
- [ ] 是否验证调用方身份
- [ ] 是否有基于范围的授权

### 网络安全
- [ ] 是否使用 TLS 加密
- [ ] 是否验证证书
- [ ] 内网服务是否限制监听地址
- [ ] 服务发现是否验证地址

### 可靠性
- [ ] 是否设置超时
- [ ] 是否有重试机制
- [ ] 重试是否有次数限制
- [ ] 是否有熔断器

### 可观测性
- [ ] 是否传递 Request ID
- [ ] 是否记录调用日志
- [ ] 是否记录认证失败
- [ ] 日志是否包含敏感信息
