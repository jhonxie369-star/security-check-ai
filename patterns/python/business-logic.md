# Python语言 - 业务逻辑安全检查指导

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
grep -rn "@app.route.*order\|def.*create.*order\|def.*place.*order" --include="*.py" -A 20 -B 5

# 搜索金额相关
grep -rn "amount\|price\|total" --include="*.py" -A 10 -B 5

# 搜索价格计算
grep -rn "calculate.*price\|compute.*price" --include="*.py" -A 15 -B 5
```

### AI分析要点

```python
# ❌ 危险：直接使用前端金额
@app.route('/order', methods=['POST'])
def create_order():
    data = request.json
    order = Order(
        goods_id=data['goods_id'],
        amount=data['amount']  # 前端传入
    )
    db.session.add(order)
    db.session.commit()

# ✅ 安全：后端重新计算
@app.route('/order', methods=['POST'])
def create_order():
    data = request.json
    
    # 后端计算真实价格
    real_price = calculate_price(
        data['goods_id'],
        data.get('coupon_id')
    )
    
    # 验证前端金额
    if abs(real_price - Decimal(data['amount'])) > Decimal('0.01'):
        return jsonify({'error': '价格异常'}), 400
    
    order = Order(
        goods_id=data['goods_id'],
        amount=real_price
    )
    db.session.add(order)
    db.session.commit()

def calculate_price(goods_id, coupon_id=None):
    goods = Goods.query.get(goods_id)
    price = goods.price
    
    if coupon_id:
        coupon = Coupon.query.get(coupon_id)
        validate_coupon(coupon, price)
        price = price * coupon.discount
    
    if price <= 0:
        raise ValueError("价格异常")
    
    return price
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
grep -rn "stock\|inventory" --include="*.py" -A 15 -B 5

# 搜索Redis原子操作
grep -rn "redis.*decr\|redis.*incr" --include="*.py" -A 10 -B 5
```

### AI分析要点

```python
# ❌ 危险：非原子操作
def buy_product(product_id, quantity):
    product = Product.query.get(product_id)
    if product.stock < quantity:
        raise Exception('库存不足')
    product.stock -= quantity
    db.session.commit()

# ✅ 安全：Redis原子操作
impoedis

r = redis.Redis()

def buy_product(product_id, quantity):
    key = f'stock:{product_id}'
    new_stock = r.decrby(key, quantity)
    
    if new_stock < 0:
        r.incrby(key, quantity)  # 回滚
        raise Exception('库存不足')
```

### 检查要点

- [ ] 是否使用Redis原子操作
- [ ] 是否有库存不足时的回滚逻辑

---

## 3. 优惠券验证检查指导

### 搜索关键代码

```bash
grep -rn "coupon\|Coupon" --include="*.py" -A 15 -B 5
```

### AI分析要点

```python
def validate_coupon(coupon, order_amount):
    from datetime import datetime
    
    # 1. 检查有效期
    now = datetime.now()
    if not (coupon.start_time <= now <= coupon.end_time):
        raise ValueError('优惠券已过期')
    
    # 2. 检查使用条件
    if order_amount < coupon.min_amount:
        raise ValueError('订单金额不满足使用条件')
    
    # 3. 检查使用次数
    user_id = get_current_user_id()
    used_count = CouponUsage.query.filter_by(
        user_id=user_id,
        coupon_id=coupon.id
    ).count()
    
    if used_count >= coupon.max_usage_per_user:
        raise ValueError('优惠券使用次数已达上限')
    
    # 4. 防止重复使用
    key = f'coupon:used:{user_id}:{coupon.id}'
    if not r.setnx(key, 1):
        raise ValueError('优惠券正在使用中')
    r.expire(key, 300)
```

---

## 4. 状态机检查指导

### 搜索关键代码

```bash
grep -rn "status\|state" --include="*.py" | grep -i "update\|change\|set" -A 10 -B 5
```

### AI分析要点

```python
# ❌ 危险：无状态验证
def cancel_order(order_id):
    order = Order.query.get(order_id)
    order.status = 'CANCELLED'
    db.session.commit()

# ✅ 安全：状态机验证
STATE_TRANSITIONS = {
    'PENDING': ['PAID', 'CANCELLED'],
    'PAID': ['SHIPPED', 'REFUNDING'],
    'SHIPPED': ['COMPLETED', 'REFUNDING'],
    'COMPLETED': ['REFUNDING'],
    'REFUNDING': ['REFUNDED'],
    'CANCELLED': [],
    'REFUNDED': []
}

def can_transition(from_status, to_status):
    allowed = STATE_TRANSITIONS.get(from_status, [])
    return to_status in allowed

def cancel_order(order_id):
    order = Order.query.get(order_id)
    
    if not can_transition(order.status, 'CANCELLED'):
        raise ValueError(f'不允许从{order.status}转换到CANCELLED')
    
    old_status = order.status
    order.status = 'CANCELLED'
    
    # 记录状态变更
    log_status_change(order_id, old_status, 'CANCELLED')
    
    db.session.commit()
```

---

## 5. 限流检查指导

### 搜索关键代码

```bash
grep -rn "limiter\|rate_limit\|RateLimit" --include="*.py" -A 5 -B 5
```

### AI分析要点

```python
# ✅ 使用Flask-Limiter
from flask_limiter import Limiter
from flask_limiort get_remote_address

limiter = Limiter(
    app,
    key_func=get_remote_address,
    default_limits=["200 per day", "50 per hour"]
)

@app.route('/order', methods=['POST'])
@limiter.limit("10 per minute")
def create_order():
    pass

# ✅ 使用Redis限流
def rate_limit(key, limit, period):
    redis_key = f'rate_limit:{key}'
    count = r.incr(redis_key)
    
    if count == 1:
        r.expire(redis_key, period)
    
    return count <= limit
```

---

## 6. 幂等性检查指导

### 搜索关键代码

```bash
grep -rn "idempotent\|Idempotent-Token" --include="*.py" -A 10 -B 5
```

### AI分析要点
hon
@app.route('/order', methods=['POST'])
def create_order():
    token = request.headers.get('Idempotent-Token')
    if not token:
        return jsonify({'error': '缺少幂等Token'}), 400
    
    key = f'idempotent:{token}'
    if not r.setnx(key, '1'):
        return jsonify({'error': '请勿重复提交'}), 400
    
    r.expire(key, 300)
    
    try:
        order = create_order_internal(request.json)
        return jsonify(order.to_dict())
    except Exception as e:
        r.delete(key)  # 失败时删除，允许重试
        raise
```

---

## 7. 真实IP获取检查指导

### 搜索关键代码

```bash
grep -rn "remote_addr\|X-Forwarded-For\|X-Real-IP" --include="*.py" -A 10 -B 5
```

### AI分析要点

```python
TRUSTED_PROXIES = {'10.0.0.1', '172.16.0.1'}

def get_real_ip(request):
    remote_addr = request.remote_addr
    
    if remote_addr not in TRUSTED_PROXIES:
        return remote_addr
    
    xff = request.headers.get('X-Forwarded-For')
    if not xff:
        return remote_addr
    
    # 从右往左找第一个非可信代理IP
    ips = [ip.strip() for ip in xff.split(',')]
    for ip in reversed(ips):
        if ip not in TRUSTED_PROXIES:
            return ip
    
    return remote_addr
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
