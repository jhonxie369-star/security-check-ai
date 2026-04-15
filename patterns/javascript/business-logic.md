# JavaScript/TypeScript 业务逻辑安全检查指导

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
grep -rn "app\.post.*order\|router\.post.*order\|createOrder" --include="*.js" --include="*.ts" -A 20 -B 5

# 搜索金额相关
grep -rn "amount\|price\|total" --include="*.js" --include="*.ts" -A 10 -B 5

# 搜索价格计算
grep -rn "calculatePrice\|computePrice\|getPrice" --include="*.js" --include="*.ts" -A 15 -B 5
```

### AI分析要点

```javascript
// ❌ 危险：直接使用前端金额
app.post('/order', async (req, res) => {
    const { goodsId, amount } = req.body;
    
    const order = await Order.create({
        goodsId,
        amount  // 前端传入
    });
    
    res.json(order);
});

// ✅ 安全：后端重新计算
app.post('/order', async (req, res) => {
    const { goodsId, couponId, amount: requestAmount } = req.body;
    
    // 后端计算真实价格
    const realPrice = await calculatePrice(goodsId, couponId);
    
    // 验证前端金额
    if (Math.abs(realPrice - requestAmount) > 0.01) {
        return res.status(400).json({ error: '价格异常' });
    }
    
    const order = await Order.create({
        goodsId,
        amount: realPrice
    });
    
    res.json(order);
});

async function calculatePrice(goodsId, couponId) {
    const goods = await Goods.findByPk(goodsId);
    if (!goods) throw new Error('商品不存在');
    
    let price = goods.price;
    
    if (couponId) {
        const coupon = await Coupon.findByPk(couponId);
        if (coupon && coupon.isValid()) {
            price = price * (1 - coupon.discount);
        }
    }
    
    if (price <= 0) {
        throw new Error('价格异常');
    }
    
    return price;
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
grep -rn "stock\|inventory" --include="*.js" --include="*.ts" -A 15 -B 5

# 搜索Redis原子操作
grep -rn "redis.*decr\|redis.*incr\|decrby\|incrby" --include="*.js" --include="*.ts" -A 10 -B 5

# 搜索数据库事务
grep -rn "transaction\|sequelize\.transaction" --include="*.js" --include="*.ts" -A 10 -B 5
```

### AI分析要点

```javascript
// ❌ 危险：非原子操作
async function buyProduct(productId, quantity) {
    const product = await Product.findByPk(productId);
    
    if (product.stock < quantity) {
        throw new Error('库存不足');
    }
    
    productuantity;
    await product.save();
}

// ✅ 安全：数据库事务 + 原子更新
async function buyProduct(productId, quantity) {
    const result = await sequelize.transaction(async (t) => {
        const [affectedRows] = await Product.update(
            { stock: sequelize.literal(`stock - ${quantity}`) },
            {
                where: {
                    id: productId,
                    stock: { [Op.gte]: quantity }
                },
                transaction: t
            }
        );
        
        if (affectedRows === 0) {
            throw new Error('库存不足');
        }
    });
    
    return result;
}

// ✅ 安全：Redis原子操作
const Redis = require('ioredis');
const redis = new Redis();

async function buyProduct(productId, quantity) {
    const key = `stock:${productId}`;
    
    const newStock = await redis.decrby(key, quantity);
    
    if (newStock < 0) {
        await redis.incrby(key, quantity);  // 回滚
        throw new Error('库存不足');
    }
    
    return newStock;
}
```

### 检查要点

- [ ] 是否使用数据库事务或Redis原子操作
- [ ] 是否有库存不足时的回滚逻辑
- [ ] 是否在WHERE子句中验证库存充足

---

## 3. 优惠券验证检查指导

### 搜索关键代码

```bash
grep -rn "coupon\|Coupon" --include="*.js" --include="*.ts" -A 15 
### AI分析要点

```javascript
async function validateCoupon(coupon, orderAmount, userId) {
    // 1. 检查有效期
    const now = new Date();
    if (now < coupon.startTime || now > coupon.endTime) {
        throw new Error('优惠券已过期');
    }
    
    // 2. 检查使用条件
    if (orderAmount < coupon.minAmount) {
        throw new Error('订单金额不满足使用条件');
    }
    
    // 3. 检查使用次数
    const count = await CouponUsage.count({
        where: { userId, couponId: coupon.id }
    });
    
    if (count >= coupon.maxUsagePerUser) {
        throw new Error('优惠券使用次数已达上限');
    }
    
    // 4. 防止重复使用
    const key = `coupon:used:${userId}:${coupon.id}`;
    const acquired = await redis.set(key, '1', 'EX', 300, 'NX');
    if (!acquired) {
        throw new Error('优惠券正在使用中');
    }
    
    return true;
}
```

---

## 4. 状态机检查指导

### 搜索关键代码

```bash
grep -rn "status\|state" --include="*.js" --include="*.ts" | grep -i "update\|change\|set" -A 10 -B 5
```

### AI分析要点

```javascript
// ❌ 危险：无状态验证
async function cancelOrder(orderId) {
    await Order.update(
        { status: 'CANCELLED' },
        { where: { id: orderId } }
    );
}

// ✅ 安全：状态机验证
const STATE_TRANSITIONS = {
    'PENDING': ['PAID', 'CANCELLED'],
    'PAID'['SHIPPED', 'REFUNDING'],
    'SHIPPED': ['COMPLETED', 'REFUNDING'],
    'COMPLETED': ['REFUNDING'],
    'REFUNDING': ['REFUNDED'],
    'CANCELLED': [],
    'REFUNDED': []
};

function canTransition(fromStatus, toStatus) {
    const allowed = STATE_TRANSITIONS[fromStatus] || [];
    return allowed.includes(toStatus);
}

async function cancelOrder(orderId) {
    const order = await Order.findByPk(orderId);
    
    if (!canTransition(order.status, 'CANCELLED')) {
        throw new Error(`不允许从${order.status}转换到CANCELLED`);
    }
    
    const oldStatus = order.status;
    order.status = 'CANCELLED';
    
    // 记录状态变更
    await logStatusChange(orderId, oldStatus, 'CANCELLED');
    
    await order.save();
}
```

---

## 5. 限流检查指导

### 搜索关键代码

```bash
grep -rn "rateLimit\|express-rate-limit\|throttle" --include="*.js" --include="*.ts" -A 5 -B 5
```

### AI分析要点

```javascript
// ✅ 使用express-rate-limit
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');

const limiter = rateLimit({
    store: new RedisStore({
        client: redis,
        prefix: 'rate_limit:'
    }),
    windowMs: 15 * 60 * 1000,
    max: 100,
    message: '请求过于频繁'
});

app.use(limiter);

// 特定接口限流
const orderLimiter = rateLwindowMs: 60 * 1000,
    max: 5,
    keyGenerator: (req) => req.user.id
});

app.post('/order', orderLimiter, async (req, res) => {
    // 处理订单
});

// ✅ 使用Redis自定义限流
async function rateLimit(userId, action, limit, windowMs) {
    const key = `rate_limit:${userId}:${action}`;
    const count = await redis.incr(key);
    
    if (count === 1) {
        await redis.expire(key, Math.ceil(windowMs / 1000));
    }
    
    if (count > limit) {
        throw new Error('操作过于频繁');
    }
}
```

---

## 6. 幂等性检查指导

### 搜索关键代码

```bash
grep -rn "idempotent\|x-request-id\|requestId" --include="*.js" --include="*.ts" -A 10 -B 5
```

### AI分析要点

```javascript
async function idempotentMiddleware(req, res, next) {
    const requestId = req.headers['x-request-id'];
    
    if (!requestId) {
        return res.status(400).json({ error: '缺少请求ID' });
    }
    
    const key = `idempotent:${requestId}`;
    
    // 检查是否已处理
    const cached = await redis.get(key);
    if (cached) {
        return res.json(JSON.parse(cached));
    }
    
    // 标记正在处理
    const acquired = await redis.set(key, 'processing', 'EX', 86400, 'NX');
    if (!acquired) {
        return res.status(4son({ error: '请求正在处理中' });
    }
    
    const originalSend = res.send;
    
    res.send = function(data) {
        redis.setex(key, 86400, JSON.stringify(data));
        originalSend.call(this, data);
    };
    
    next();
}

app.post('/payment', idempotentMiddleware, async (req, res) => {
    // 处理支付
});
```

---

## 7. 真实IP获取检查指导

### 搜索关键代码

```bash
grep -rn "req\.ip\|x-forwarded-for\|x-real-ip\|trust proxy" --include="*.js" --include="*.ts" -A 10 -B 5
```

### AI分析要点

```javascript
// ❌ 危险：直接信任X-Forwarded-For
function getClientIP(req) {
    return req.headers['x-forwarded-for']?.split(',')[0] || req.ip;
}

// ✅ 安全：配置可信代理
const express = require('express');
const app = express();

// 设置可信代理
app.set('trust proxy', ['127.0.0.1', '::1', '10.0.0.0/8']);

function getClientIP(req) {
    return req.ip;
}

// ✅ 更安全：手动验证
function getClientIP(req) {
    const forwarded = req.headers['x-forwarded-for'];
    
    if (forwarded) {
        const ips = forwarded.split(',').map(ip => ip.trim());
        // 取最后一个可信IP之前的IP
        return ips[ips.length - 2] || ips[ips.length - 1];
    }
    
    return req.socket.remoteAddress;
}
```

---

## 检查清单

- [ ] 价格由后端计算
- [ ] 库存使用原子操作
- [ ] 优惠券有完整验证
- [ ] 状态转换有验证
- [ ] 关键接口有限流
- [ ] 重要操作有幂等性
- [ ] 正确获取真实IP
