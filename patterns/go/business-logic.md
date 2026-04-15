# Go语言 - 业务逻辑安全检查指导

## ⚠️ 核心原则

**本文件提供业务逻辑检查思路，不是固定规则。必须先理解业务，再灵活构造检查。**

**工作流程**：理解业务 → 猜测漏洞 → Grep 搜索 → AI 深度分析

---

## 1. 价格验证检查指导

### 业务风险
前端传入的金额可能被篡改，导致低价购买。

### 搜索关键代码

```bash
# 搜索订单创建
grep -rn "HandleFunc.*order\|GET.*order\|POST.*order" --include="*.go" -A 20 -B 5

# 搜索金额相关
grep -rn "amount\|price\|total\|Price\|Amount" --include="*.go" -A 10 -B 5

# 搜索价格计算
grep -rn "calculatePrice\|computePrice\|getPrice" --include="*.go" -A 15 -B 5
```

### AI分析要点

```go
// ❌ 危险：直接使用前端金额
func createOrder(c *gin.Context) {
    var req struct {
        GoodsID string  `json:"goods_id"`
        Amount  float64 `json:"amount"`  // 前端传入
    }
    c.BindJSON(&req)
    
    order := &Order{
        GoodsID: req.GoodsID,
        Amount:  req.Amount,
    }
    db.Create(order)
}

// ✅ 安全：后端重新计算
func createOrder(c *gin.Context) {
    var req struct {
        GoodsID  string  `json:"goods_id"`
        CouponID string  `json:"coupon_id"`
        Amount   float64 `json:"amount"`
    }
    c.BindJSON(&req)
    
    // 后端计算真实价格
    realPrice := calculatePrice(req.GoodsID, req.CouponID)
    
    // 验证前端金额
    if math.Abs(realPrice-req.Amount) > 0.01 {
        c.JSON(400, gin.H{"error": "价格异常"})
        return
    }
    
    order := &Order{
        GoodsID: req.GoodsID,
        Amount:  realPrice,
    }
    db.Create(order)
}

func calculatePrice(goodsID, couponID string) float64 {
    var goods Goods
    db.First(&goods, goodsID)
    
    price := goods.Price
    
    if couponID != "" {
        var coupon Coupon
        db.First(&coupon, couponID)
        validateCoupon(&coupon, price)
        price = price * coupon.Discount
    }
    
    if price <= 0 {
        panic("价格异常")
    }
    
    return price
}
```

### 检查要点

- [ ] 价格是否由后端计算
- [ ] 是否验证前端金额与后端计算是否一致
- [ ] 是否防止负数金额

---

## 2. 库存控制检查指导

### 业务风险
并发扣减库存可能导致超卖。

### 搜索关键代码

```bash
# 搜索库存操作
grep -rn "stock\|inventory\|Stock\|Inventory" --include="*.go" -A 15 -B 5

# 搜索Redis原子操作
grep -rn "redis.*Decr\|redis.*Incr\|DecrBy\|IncrBy" --include="*.go" -A 10 -B 5

# 搜索数据库更新
grep -rn "UPDATE.*stock\|Exec.*stock" --include="*.go" -A 10 -B 5
```

### AI分析要点

```go
// ❌ 危险：非原子操作
func buyProduct(productID string, quantity int) error {
    var product Product
    db.First(&product, productID)
    
    if product.Stock < quantity {
        return errors.New("库存不足")
    }
    
    product.Stock -= quantity
    db.Save(&product)
    return nil
}

// ✅ 安全：SQL原子操作
func buyProduct(productID string, quantity int) error {
    result := db.Exec(
        "UPDATE products SET stock = stock - ? WHERE id = ? AND stock >= ?",
        quantity, productID, quantity,
    )
    
    if result.RowsAffected == 0 {
        return errors.New("库存不足")
    }
    return nil
}

// ✅ 安全：Redis原子操作
func buyProductRedis(productID string, quantity int) error {
    key := "stock:" + productID
    
    newStock, err := rdb.DecrBy(ctx, key, int64(quantity)).Result()
    if err != nil {
        return err
    }
    
    if newStock < 0 {
        rdb.IncrBy(ctx, key, int64(quantity))  // 回滚
        return errors.New("库存不足")
    }
    return nil
}
```

### 检查要点

- [ ] 是否使用SQL原子操作或Redis原子操作
- [ ] 是否有库存不足时的回滚逻辑
- [ ] 是否在WHERE子句中验证库存充足

---

## 3. 优惠券验证检查指导

### 搜索关键代码

```bash
grep -rn "coupon\|Coupon" --include="*.go" -A 15 -B 5
```

### AI分析要点

```go
func validateCoupon(coupon *CorderAmount float64, userID int64) error {
    // 1. 检查有效期
    now := time.Now()
    if now.Before(coupon.StartTime) || now.After(coupon.EndTime) {
        return errors.New("优惠券已过期")
    }
    
    // 2. 检查使用条件
    if orderAmount < coupon.MinAmount {
        return errors.New("订单金额不满足使用条件")
    }
    
    // 3. 检查使用次数
    var count int64
    db.Model(&CouponUsage{}).Where(
        "user_id = ? AND coupon_id = ?",
        userID, coupon.ID,
    ).Count(&count)
    
    if count >= coupon.MaxUsagePerUser {
        return errors.New("优惠券使用次数已达上限")
    }
    
    // 4. 防止重复使用
    key := fmt.Sprintf("coupon:used:%d:%d", userID, coupon.ID)
    result, _ := rdb.SetNX(ctx, key, 1, 5*time.Minute).Result()
    if !result {
        return errors.New("优惠券正在使用中")
    }
    
    return nil
}
```

---

## 4. 状态机检查指导

### 搜索关键代码

```bash
grep -rn "status\|state\|Status\|State" --include="*.go" | grep -i "update\|change\|set" -A 10 -B 5
```

### AI分析要点

```go
// ❌ 危险：无状态验证
func cancelOrder(orderID string) error {
    return db.Model(&Order{}).Where("id = ?", orderID).
        Update("status", "CANCELLED").Error
}

// ✅ 安全：状态机验证
var stateTransitions = map[string][]string{
    "PENDING":   {"PAID", "CANCELLED"},
    "PAID":      {"SHIPPED", "REFUNDING"},
    "SHIPPED":   {"COMPLETED", "REFUNDING"},
    "COMPLETED": {"REFUNDING"},
    "REFUNDING": {"REFUNDED"},
    "CANCELLED": {},
    "REFUNDED":  {},
}

func canTransition(fromStatus, toStatus string) bool {
    allowed := stateTransitions[fromStatus]
    for _, s := range allowed {
        if s == toStatus {
            return true
        }
    }
    return false
}

func cancelOrder(orderID string) error {
    var order Order
    db.First(&order, orderID)
    
    if !canTransition(order.Status, "CANCELLED") {
        return fmt.Errorf("不允许从%s转换到CANCELLED", order.Status)
    }
    
    oldStatus := order.Status
    order.Status = "CANCELLED"
    
    // 记录状态变更
    logStatusChange(orderID, oldStatus, "CANCELLED")
    
    return db.Save(&order).Error
}
```

---

## 5. 限流检查指导

### 搜索关键代码

```bash
grep -rn "limiter\|rate_limit\|RateLimit\|Throttle" --include="*.go" -A 5 -B 5
```

### AI分析要点

```go
// ✅ 使用中间件限流
func RateLimitMiddleware(limit int, window time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        userID := c.GetString("userID")
        key := fmt.Sprintf("rate_limit:%s:%s", userID, c.Request.URL.Path)
        
        count, _ := rdb.Incr(ctx, key).Result()
        if count == 1 {
            rdb.Expire(ctx, key, window)
        }
        
        if count > int64(limit) {
            c.JSON(429, gin.H{"error": "请求太频繁"})
            c.Abort()
            return
        }
        
        c.Next()
    }
}

// 使用
router.POST("/order", RateLimitMiddleware(10, 60*time.Second), createOrder)

// ✅ 使用Redis限流
func rateLimit(key string, limit int, period time.Duration) bool {
    redisKey := "rate_limit:" + key
    count, _ := rdb.Incr(ctx, redisKey).Result()
    
    if count == 1 {
        rdb.Expire(ctx, redisKey, period)
    }
    
    return count <= int64(limit)
}
```

---

## 6. 幂等性检查指导

### 搜索关键代码

```bash
grep -rn "idempotent\|Idempotent-Token" --include="*.go" -A 10 -B 5
```

### AI分析要点

```go
func createOrder(c *gin.Context) {
    token := c.GetHeader("Idempotent-Token")
    if token == "" {
        c.JSON(400, gin.H{"error": "缺少幂等Token"})
        return
    }
    
    key := "idempotent:" + token
    result, _ := rdb.SetNX(ctx, key, "1", 5*time.Minute).Result()
    if !result {
        c.JSON(400, gin.H{"error": "请勿重复提交"})
        return
    }
    
    order, err := createOrderInternal(c)
    if err != nil {
        rdb.Del(ctx, key)  // 失败时删除，允许重试
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(200, order)
}
```

---

## 7. 真实IP获取检查指导

### 搜索关键代码

```bash
grep -rn "RemoteAddr\|X-Forwarded-For\|X-Real-IP" --include="*.go" -A 10 -B 5
```

### AI分析要点

```go
var trustedProxies = map[string]bool{
    "10.0.1":   true,
    "172.16.0.1": true,
}

func getRealIP(c *gin.Context) string {
    remoteAddr := c.ClientIP()
    
    if !trustedProxies[remoteAddr] {
        return remoteAddr
    }
    
    xff := c.GetHeader("X-Forwarded-For")
    if xff == "" {
        return remoteAddr
    }
    
    // 从右往左找第一个非可信代理IP
    ips := strings.Split(xff, ",")
    for i := len(ips) - 1; i >= 0; i-- {
        ip := strings.TrimSpace(ips[i])
        if !trustedProxies[ip] {
            return ip
        }
    }
    
    return remoteAddr
}
```

---

## 检查清单

- [ ] 价格由后端计算
- [ ] 库存使用原子操作
- [ ] 优惠券有完整验证
- [ ] 状态转换有验证
- [ ] 关键[ ] 重要操作有幂等性
- [ ] 正确获取真实IP
