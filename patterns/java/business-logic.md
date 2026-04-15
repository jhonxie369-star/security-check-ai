# Java语言 - 业务逻辑安全检查指导

## ⚠️ 核心原则

**本文件提供业务逻辑检查思路，不是固定规则。必须先理解业务，再灵活构造检查。**

## ⚠️ 正确的检查流程（重要！）

**四步流程**：理解业务 → 猜测漏洞 → Grep 搜索 → AI 深度分析

**示例**（转账金额验证）：
1. 理解业务：发现转账功能
2. 猜测漏洞：金额可能是负数？
3. Grep 搜索：`grep -rn "amount.*>" --include="*.java"` → 未找到验证
4. AI 分析：读取完整转账函数，确认存在负数转账漏洞

---

## 1. 价格验证检查指导

### 业务风险
前端传入的金额可能被篡改，导致低价购买。

### 搜索关键代码

```bash
# 搜索订单创建接口
grep -rn "@PostMapping.*order\|createOrder\|placeOrder" --include="*.java" -A 20 -B 5

# 搜索金额相关代码
grep -rn "getAmount\|getPrice\|getTotal\|setAmount\|setPrice" --include="*.java" -A 10 -B 5

# 搜索价格计算函数
grep -rn "calculatePrice\|computePrice" --include="*.java" -A 15 -B 5
```

### AI分析要点

**第1步：识别金额来源**

读取订单创建的完整方法，分析：
```java
// ❌ 危险：直接使用前端金额
@PostMapping("/order")
public Order createOrder(@RequestBody OrderRequest request) {
    Order order = new Order();
    order.setAmount(request.getAmount());  // 前端传入
    orderService.save(order);
    return order;
}

// ✅ 安全：后端重新计算
@PostMapping("/order")
public Order createOrder(@RequestBody OrderRequest request) {
    // 后端计算真实价格
    BigDecimal realPrice = calculatePrice(
        request.getGoodsId(), 
        request.getCouponId()
    );
    
    // 验证前端金额
    if (!realPrice.equals(request.getAmount())) {
        throw new SecurityException("价格异常");
    }
    
    Order order = new Order();
    order.setAmount(realPrice);
    orderService.save(order);
    return order;
}
```

**第2步：检查价格计算逻辑**

如果有`calculatePrice()`函数，读取其完整实现：
- 是否考虑了商品原价
- 是否考虑了优惠券折扣
- 是否考虑了会员等级折扣
- 是否考虑了满减活动

**第3步：追踪数据流**

读取从接口到数据库保存的完整调用链：
```
Controller.createOrder() 
  → request.getAmount() (前端输入)
  → order.setAmount() (直接使用)
  → orderService.save() (保存到数据库)
```

检查中间是否有：
- 价格计算函数调用
- 价格验证逻辑
- 价格一致性检查

### 检查要点

- [ ] 价格是否由后端计算
- [ ] 是否验证前端金额与后端计算是否一致
- [ ] 计算函数是否考虑了所有折扣因素
- [ ] 是否防止负数金额

### 修复示例

```java
// ❌ 危险：直接使用前端金额
order.setAmount(request.getAmount());

// ✅ 安全：后端重新计算
BigDecimal realPrice = calculatePrice(request.getGoodsId(), request.getCouponId());
if (!realPrice.equals(request.getAmount())) {
    throw new SecurityException("价格异常");
}
order.setAmount(realPrice);

// ✅ 完整的价格计算
private BigDecimal calculatePrice(Long goodsId, Long couponId) {
    // 1. 获取商品原价
    Goods goods = goodsService.getById(goodsId);
    BigDecimal price = goods.getPrice();
    
    // 2. 应用优惠券
    if (couponId != null) {
        Coupon coupon = couponService.getById(couponId);
        va(coupon, price);  // 验证优惠券
        price = price.multiply(coupon.getDiscount());
    }
    
    // 3. 应用会员折扣
    User user = getCurrentUser();
    if (user.getVipLevel() > 0) {
        price = price.multiply(getVipDiscount(user.getVipLevel()));
    }
    
    // 4. 验证最终价格
    if (price.compareTo(BigDecimal.ZERO) <= 0) {
        throw new BusinessException("价格异常");
    }
    
    return price;
}
```

---

## 2. 库存控制检查指导

### 业务风险
并发扣减库存可能导致超卖。

### ⚠️ 并发控制 vs 业务逻辑漏洞

**重要**：不要混淆并发控制缺失和业务逻辑漏洞

**并发控制缺失（真正的并发问题）**：
- 定义：多个线程同时操作，导致**数据不一致**（丢失更新）
- 特征：实际余额 例如：10 次转账每次扣 100，但余额只减少了 500

**业务逻辑漏洞（不是并发问题）**：
- 定义：业务规则验证不足，允许不合理的操作
- 特征：数据一致，但操作次数超额
- 例如：余额 1000 理论最多转 20 次（每次 50），但实际成功 28 次
- 原因：多个线程同时通过了余额检查

**判断标准**：

| 数据一致性 | 成功次数 | 判断 |
|-----------|---------|------|
| 一致 | 正常 | ❌ 无问题 |
| 一致 | 超额 | ✅ 业务逻辑漏洞 |
| 不一致 | 任意 | ✅ 并发控制缺失 |

### 搜索关键代码

```bash
# 搜索库存操作
grep -rn "stock\|inventory\|setStock\|updateStock" --include="*.java" -A 15 -B 5

# 搜索Redis原子操作
grep -rn "decrement\|increment\|opsForValue" --include="*.java" -A 10 -B 5

# 搜索分布式锁
grep -rn "RedissonClient\|tryLock\|@Transactional" --include="*.java" -A 10 -B 5
```

### AI分析要点

**检查库存扣减方式**：

```java
// ❌ 危险：非原子操作
public void buyProduct(Long productId, Integer quantity) {
    Product product = productService.getById(productId);
    if (product.getStock() < quantity) {
        throw new BusinessException("库存不足");
    }
    product.setStock(product.getStock() - quantity);
    productService.updateById(product);
}

// ✅ 安全：Redis原子操作
public void buyProduct(Long productId, Integer quantity) {
    String key = "stock:" + productId;
    Long newStock = redisTemplate.opsForValue().decrement(key, quantity);
    
    if (newStock < 0) {
        // 回滚
        redisTemplate.opsForValue().increment(key, quantity);
        throw new BusinessException("库存不足");
    }
}

// ✅ 安全：分布式锁
public void buyProduct(Long productId, Integer quantity) {
    RLock lock = redissonClient.getLock("lock:product:" + productId);
    try {
        lock.lock();
        Product product = productService.getById(productId);
        if (product.getStock() < quantity) {
            throw new BusinessException("库存不足");
        }
        product.setStock(product.getStock() - quantity);
        productService.updateById(product);
    } finally {
        lock.unlock();
    }
}
```

### 检查要点

- [ ] 是否使用Redis原子操作（推荐）
- [ ] 是否使用分布式锁
- [ ] 是否有库存不足时的回滚逻辑
- [ ] 是否只依赖数据库事务（性能差）

### 修复示例

```java
// ✅ 推荐：Redis原子操作
@Service
public class StockService {
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    public boolean deductStock(Long productId, Integer quantity) {
        String key = "stock:" + productId;
        Long newStock = redisTemplate.opsForValue().decrement(key, quantity);
        
        if (newStock < 0) {
            // 回滚
            redisTemplate.opsForValue().increment(key, quantity);
            return false;
        }
        
        return true;
    }
}
```

---

## 3. 优惠券验证检查指导

### 业务风险
优惠券未验证可能导致重复使用、超额使用。

### 搜索关键代码

```bash
# 搜索优惠券使用
grep -rn "coupon\|Coupon\|useCoupon\|applyCoupon" --include="*.java" -A 15 -B 5

# 搜索优惠券验证
grep -rn "validateCoupon\|checkCoupon\|verifyCoupon" --include="*.java" -A 15 -B 5
```

### AI分析要点

**检查优惠券验证逻辑**：

```java
// ✅ 完整的优惠券验证
private void validateCoupon(Coupon coupon, BigDecimal orderAmount) {
    // 1. 检查有效期
    LocalDateTime now = LocalDateTime.now();
    if (now.isBefore(coupon.getStartTime()) || now.isAfter(coupon.getEndTime())) {
        throw new BusinessException("优惠券已过期");
    }
    
    // 2. 检查使用条件
    if (orderAmount.compareTo(coupon.getMinAmount()) < 0) {
        throw new BusinessException("订单金额不满足使用条件");
    }
    
    // 3. 检查使用次数
    Long userId = getCurrentUserId();
    Integer usedCount = couponUsageService.countByUserAndCoupon(userId, coupon.getId());
    if (usedCount >= coupon.getMaxUsagePerUser()) {
        throw new BusinessException("优惠券使用次数已达上限");
    }
    
    // 4. 检查互斥规则
    if (coupon.isExclusive() && hasOtherCouponUsed(userId)) {
        throw new BusinessException("该优惠券不可与其他优惠叠加");
    }
    
    // 5. 防止重复使用（幂等性）
    String key = "coupon:used:" + userId + ":" + coupon.getId();
    Boolean success = redisTemplate.opsForValue().setIfAbsent(key, "1", 5, TimeUnit.MINUTES);
    if (!success) {
        throw new BusinessException("优惠券正在使用中");
    }
}
```

### 检查要点

- [ ] 是否验证优惠券有效期
- [ ] 是否验证使用条件（最低金额）
- [ ] 是否检查使用次数限制
- [ ] 是否检查互斥规则（不可叠加）
- [ ] 是否防止重复使用（幂等性）

---

## 4. 状态机检查指导

### 业务风险
订单状态可以任意跳转，导致逻辑绕过。

### 搜索关键代码

```bash
# 搜索状态更新
grep -rn "setStatus\|updateStatus\|changeStatus" --include="*.java" -A 15 -B 5

# 搜索grep -rn "checkStatus\|validateStatus\|getStatus" --include="*.java" -A 10 -B 5
```

### AI分析要点

**检查状态转换规则**：

```java
// ❌ 危险：无状态验证
public void cancelOrder(Long orderId) {
    Order order = orderService.getById(orderId);
    order.setStatus("CANCELLED");
    orderService.updateById(order);
}

// ✅ 安全：状态机验证
public void cancelOrder(Long orderId) {
    Order order = orderService.getById(orderId);
    
    // 定义允许取消的状态
    List<String> allowedStatuses = Arrays.asList("PENDING", "PAID");
    if (!allowedStatuses.contains(order.getStatus())) {
        throw new BusinessException("当前状态);
    }
    
    // 记录状态变更日志
    logStatusChange(orderId, order.getStatus(), "CANCELLED");
    
    order.setStatus("CANCELLED");
    orderService.updateById(order);
}

// ✅ 更好：使用状态机模式
@Component
public class OrderStateMachine {
    private static final Map<String, List<String>> STATE_TRANSITIONS = Map.of(
        "PENDING", Arrays.asList("PAID", "CANCELLED"),
        "PAID", Arrays.asList("SHIPPED", "REFUNDING"),
        "SHIPPED", Arrays.asList("COMPLETED", "REFUNDING"),
        "COMPLETED", Arrays.asList("REFUNDING"n        "REFUNDING", Arrays.asList("REFUNDED"),
        "CANCELLED", Collections.emptyList(),
        "REFUNDED", Collections.emptyList()
    );
    
    public boolean canTransition(String fromStatus, String toStatus) {
        List<String> allowedStatuses = STATE_TRANSITIONS.get(fromStatus);
        return allowedStatuses != null && allowedStatuses.contains(toStatus);
    }
    
    public void transition(Order order, String toStatus) {
        if (!canTransition(order.getStatus(), toStatus)) {
            throw new BusinessException(
                String.format("不允许从 %s 转换到 %s", order.getStatus(), toStatus)
            );
        }
        
        String oldStatus = order.getStatus();
        order.setStatus(toStatus);
        
        // 记录状态变更
        logStatusChange(order.getId(), oldStatus, toStatus);
    }
}
```

### 检查要点

- [ ] 是否定义了状态转换规则
- [ ] 是否检查当前状态是否允许转换
- [ ] 是否有状态转换日志
- [ ] 是否防止状态回退（如已完成→待支付）

---

## 5. 限流检查指导

### 业务风险
接口无限流可能被刷单、刷优惠券。

### 搜索关键代码

```bash
# 搜索限流注解
grep -rn "@RateLimit\|@RateLimiter\|RateLimiter" --include="*.java" -A 5 -B 5

# 搜索Guava RateLimiter
grep -rn "com.google.common.util.concurrent.RateLimiter" --include="*.java" -A 10 -B 5

# 搜索Redis限流
grep -rn "rate_limit\|rateLimit" --include="*.java" -A 10 -B 5
```

### AI分析要点

**检查关键接口是否有限流**：

```java
// ✅ 使用注解限流
@RestController
public class OrderController {
    @PostMapping("/order")
    @RateLimit(limit = 10, period = 60) // 每分钟10次
    public Order createOrder(@RequestBody OrderRequest request) {
        return orderService.createOrder(request);
    }
}

// ✅ 使用Redis限流
@Service
public class RateLimitService {
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    public boolean tryAcquire(String key, int limit, int period) {
        String redisKey = "rate_limit:" + key;
        Long count = redisTemplate.opsForValue().increment(redisKey);
        
        if (count == 1) {
            redisTemplate.expire(redisKey, period, TimeUnit.SECONDS);
        }
        
        return count <= limit;
    }
}
```

### 检查要点

- [ ] 关键接口（下单、支付、注册）是否有限流
- [ ] 限流是否基于用户/IP
- [ ] 是否使用分布式限流（Redis）
- [ ] 限流阈值是否合理

---

## 6. 幂等性检查指导

### 业务风险
重复提交导致重复扣款、重复下单。

### 搜索关键代码

```bash
# 搜索幂等Token
grep -rn "Idempotent-Token\|idempotent\|IdempotentToken" --include="*.java" -A 10 -B 5

# 搜索Redis SetNX
grep -rn "setIfAbsent\|setnx" --include="*.java" -A 10 -B 5
```

### AI分析要点

**检查重要操作是否有幂等保护**：

```java
// ✅ 使用幂等Token
@PostMapping("/order")
public Order createOrder(@RequestHeader("Idempotent-Token") String token,
                        @RequestBody OrderRequest request) {
    // 验证Token
    String key = "idempotent:" + token;
    Boolean success = redisTemplate.opsForValue()
        .setIfAbsent(key, "1", 5, TimeUnit.MINUTES);
    
    if (!success) {
        throw new BusinessException("请勿重复提交");
    }
    
    try {
        return orderService.createOrder(request);
    } catch (Exception e) {
        // 失败时删除Token，允许重试
        redisTemplate.delete(key);
        throw e;
    }
}
```

### 检查要点

- [ ] 重要操作是否要求幂等Token
- [ ] 是否使用Redis SetNX防重复
- [ ] Token是否有过期时间
- [ ] 失败时是否允许重试

---

## 7. 真实IP获取检查指导

### 业务风险
IP伪造导致限流、风控失效。

### 搜索关键代码

```bash
# 搜索IP获取
grep -rn "getRemoteAddr\|X-Forwarded-For\|X-Real-IP" --include="*.java" -A 10 -B 5
```

### AI分析要点

**检查IP获取方式**：

```java
// ❌ 危险：直接使用X-Forwarded-For
String ip = request.getHeader("X-Forwarded-For");

// ✅ 安全：正确解析X-Forwarded-For
public String getRealIP(HttpServletRequest request) {
    String remoteAddr = request.getRemoteAddr();
    
    // 可信代理 Set<String> TRUSTED_PROXIES = Set.of("10.0.0.1", "172.16.0.1");
    
    // 不是可信代理，直接返回
    if (!TRUSTED_PROXIES.contains(remoteAddr)) {
        return remoteAddr;
    }
    
    // 从右往左找第一个非可信代理IP
    String xff = request.getHeader("X-Forwarded-For");
    if (xff == null || xff.isEmpty()) {
        return remoteAddr;
    }
    
    String[] ips = xff.split(",");
    for (int i = ips.length - 1; i >= 0; i--) {
        String ip = ips[i].trim();
        if (!TRUSTED_PROXIES.contains(ip)) {
            return ip;
        }
    }
    
    return remoteAddr;
}
```

### 检查要点

- [ ] 是否使用可信代理n- [ ] 是否正确解析X-Forwarded-For（从右往左）
- [ ] 是否验证IP格式
- [ ] 是否防止IP伪造

---

## 8. 异常行为检测检查指导

### 业务风险
批量注册、批量下单等异常行为。

### 搜索关键代码

```bash
# 搜索频率限制
grep -rn "register_count\|login_count\|order_count" --include="*.java" -A 10 -B 5

# 搜索设备指纹
grep -rn "Device-Fingerprint\|device_id" --include="*.java" -A 10 -B 5
```

### 检查要点

- [ ] 是否限制同IP/设备注册/登录次数
- [ ] 是否有设备指纹检测
- [ ] 是否有异常行为告警
- [ ] 是否有风控规则引擎

---

## 检查清单

### 价格相关
- [ ] 价格由后端计算
- [ ] 验证前端金额与后端计算是否一致
- [ ] 防止负数金额

### 库存相关
- [ ] 使用Redis原子操作或分布式锁
- [ ] 有库存不足时的回滚逻辑

### 优惠券相关
- [ ] 验证有效期
- [ ] 验证使用条件
- [ ] 检查使用次数限制
- [ ] 防止重复使用

### 状态机相关
- [ ] 定义状态转换规则
- [ ] 检查当前状态是否允许转换
- [ ] 有状态转换日志

### 防刷相关
- [ ] 关键接口有限流
- [ ] 重要操作有幂等性
- [ ] 正确获取真实IP
- [ ] 有异常行为检测
