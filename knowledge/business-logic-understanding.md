# 业务逻辑理解与安全分析指南

## 🎯 核心理念

**不要机械执行规则，而是真正理解业务后，灵活构造安全检查。**

## 📖 理解业务的步骤

### 1. 识别业务领域

**电商类**：
- 关键功能：订单、支付、优惠券、积分、会员、库存
- 风险点：价格篡改、库存超卖、优惠券滥用、刷单

**金融类**：
- 关键功能：转账、充值、提现、理财、借贷
- 风险点：金额篡改、重复转账、越权操作、洗钱

**社交类**：
- 关键功能：发帖、评论、点赞、关注、私信
- 风险点：刷量、垃圾信息、隐私泄露、恶意举报

**内容平台**：
- 关键功能：发布、审核、推荐、打赏、版权
- 风险点：违规内容、刷阅读量、盗版、恶意举报

### 2. 识别关键业务数据

**金额相关**：
- price, amount, total, discount, balance, reward
- 追踪：用户输入 → 计算 → 存储
- 检查：是否后端重新计算、是否验证范围

**数量相关**：
- quantity, count, stock, inventory, limit
- 追踪：用户输入 → 扣减 → 更新
- 检查：是否原子操作、是否防止负数

**状态相关**：
- status, state, stage, phase
- 追踪：状态变更的触发条件和流程
- 检查：是否有状态机、是否可跳过状态

**权限相关**：
- role, permission, level, vip
- 追踪：权限检查的位置和逻辑
- 检查：是否可绕过、是否可提权

### 3. 理解业务流程

**订单流程示例**：
```
1. 选择商品 → 检查库存
2. 应用优惠券 → 验证有效性
3. 计算金额 → 后端重新计算
4. 创建订单 → 锁定库存
5. 支付 → 验证金额一致性
6. 发货 → 检查订单状态
7. 确认收货 → 更新状态
```

**每个环节的风险点**：
- 库存检查：是否原子操作？
- 优惠券验证：是否可重复使用？
- 金额计算：是否信任前端？
- 订单创建：是否可重复下单？
- 支付验证：是否验证回调签名？
- 状态更新：是否可跳过状态？

## 🔍 基于理解的安全检查

### 场景1：发现"积分抵扣"功能

**理解业务**：
- 用户可用积分抵扣订单金额
- 1积分 = 0.01元
- 积分余额存储在用户表

**检查方法**：
```bash
# 1. 搜索积分相关代码
grep -rn "points\|score\|积分" --include="*.go" --include="*.java" --include="*.py" -A 10 -B 5

# 2. 搜索积分计算
grep -rn "calculateAmount\|getFinalPrice\|计算金额" --include="*.go" -A 15 -B 5

# 3. 搜索余额检查
grep -rn "checkBalance\|getBalance\|余额" --include="*.go" -A 10 -B 5
```

**检查点**：
- ✅ 是否验证积分余额充足
- ✅ 是否防止输入负数积分
- ✅ 是否原子扣减积分
- ✅ 是否记录积分使用日志

### 场景2：发现"邀请返利"功能

**理解业务**：
- 用户A邀请用户B注册，A获得10元返利
- 邀请码存储在用户表
- 返利直接加到余额

**检查方法**：
```bash
# 1. 搜索邀请码相关
grep -rn "invite\|referral\|邀请" --include="*.go" -A 10 -B 5

# 2. 搜索返利发放
grep -rn "grantReward\|addBalance\|addMoney\|返利" --include="*.go" -A 15 -B 5

# 3. 搜索自邀检查
grep -rn "checkSelfInvite\|inviter.*!=.*invitee" --include="*.go" -A 10 -B 5
```

**检查点**：
- ✅ 是否防止自己邀请自己
- ✅ 是否限制单个用户返利次数
- ✅ 是否防止同一设备多次注册
- ✅ 是否有返利上限

### 场景3：发现"拼团"功能

**理解业务**：
- 用户发起拼团，邀请好友参团
- 满N人成团，享受团购价
- 未成团自动退款

**检查方法**：
```bash
# 1. 搜索拼团价格
grep -rn "groupPrice\|teamPrice\|团购" --include="*.go" -A 10 -B 5

# 2. 搜索成团判断
grep -rn "checkGroupFull\|isGroupComplete\|成团" --include="*.go" -A 15 -B 5

# 3. 搜索订单创建
grep -rn "createOrder\|saveOrder" --include="*.go" -A 20 -B 5
**检查点**：
- ✅ 是否验证拼团已成功
- ✅ 是否防止未成团使用团购价
- ✅ 是否防止重复参团
- ✅ 是否有超时自动退款

### 场景4：发现"秒杀"功能

**理解业务**：
- 限时限量抢购
- 高并发场景
- 防止超卖

**检查方法**：
```bash
# 1. 搜索秒杀相关
grep -rn "seckill\|flashSale\|秒杀" --include="*.go" -A 10 -B 5

# 2. 搜索库存扣减
grep -rn "Decr\|decrStock\|扣减库存" --include="*.go" -A 15 -B 5

# 3. 搜索分布式锁
grep -rn "Lock\|TryLock\|Mutex\|redis.*lock" --include="*.go" -A 10 -B 5
```

**检查点**：
- ✅ 是否使用Redis原子操作
- ✅ 是否有分布式锁
- ✅ 是否限制单用户购买数量
- ✅ 是否验证秒杀时间

## 💡 发现新业务功能的方法

### 1. 关键字搜索

```bash
# 搜索业务关键词
grep -r "coupon\|discount\|reward\|invite\|group\|seckill\|flash" --include="*.go"

# 搜索金额相关
grep -r "amount\|price\|balance\|money" --include="*.go"

# 搜索状态相关
grep -r "status\|state\|stage" --include="*.go"
```

### 2. 数据库表结构

```bash
# 查看数据库迁移文件
ls db/migrations/
cat db/migrations/*.sql

# 关注表名：orders, coupons, rewards, invites, groups
```

### 3. API路由

```bash
# 查看路由定ngrep -r "POST\|GET\|PUT\|DELETE" --include="*route*.go"

# 关注路径：/order, /pay, /coupon, /invite, /group
```

### 4. 配置文件

```bash
# 查看业务配置
cat config/*.yml

# 关注：优惠券规则、返利金额、限流配置
```

## 🎯 报告格式

```markdown
## 业务逻辑安全检查

### 业务理解
- **业务类型**：电商平台
- **关键功能**：订单、支付、优惠券、积分、邀请返利
- **特殊功能**：拼团、秒杀

### 固定规则检查
1. ✅ 价格验证：后端重新计算
2. ❌ 优惠券验证：未检查互斥规则

### 基于业务理解的发现

#### 1. 积分抵扣未验证余额
- **严重程度**：🔴 Critical
- **业务逻辑**：用户可用积分抵扣订单金额，1积分=0.01元
- **代码位置**：order.go:50-55
- **风险分析**：
  - 用户可输入负数积分，导致订单金额增加
  - 用户可输入超过余额的积分，免费获得商品
  - 积分未原子扣减，存在并发问题
- **修复建议**：
  1. 验证积分余额
  userBalance := getUserBalance(userId)
  if points > userBalance {
      return errors.New("积分余额不足")
  }
  
  // 2. 防止负数
  if points < 0 {
      return errors.New("积分不能为负数")
  }
  
  // 3. 原子扣减
  newBalance := redis.DecrBy("balance:"+userId, points)
  if newBalance < 0 {
      redis.IncrBy("balance:"+userId, points) // 回滚
      return errors.New("积分余额不足")
  }
  ```

#### 2. 邀请返利可自己邀请自己
- **严重程度**：🔴 Critical
- **业务逻辑**：用户A邀请用户B注册，A获得10元返利
- **代码位置**：register.go:20-25
- **风险分析**：
  - 用户可用自己的邀请码注册小号，刷返利
  - 未限制返利次数，可无限刷
  - 未检查设备指纹，同设备可多次注册
- **修复建议**：
  ```go
  查邀请人≠被邀请人
  if inviter.ID == currentUser.ID {
      return errors.New("不能邀请自己")
  }
  
  // 2. 限制返利次数
  count := redis.Get("invite_count:" + inviter.ID)
  if count >= 10 {
      return errors.New("返利次数已达上限")
  }
  
  // 3. 检查设备指纹
  fingerprint := c.GetHeader("Device-Fingerprint")
  deviceCount := redis.SCard("device:" + fingerprint)
  if deviceCount >= 3 {
      return errors.New("同设备注册次数过多")
  }
  ```
```

## 📚 总结

**关键原则**：
1. **规则是指导，不是限制**：patterns 文件中的规则仅作为参考，不是固定模板
2. **理解业务优先**：先理解业务，再构造检查
3. **灵活扩充检测**：发现业务特有功能时，构造针对性的检查
4. **标注检测来源**：在报告中明确标注哪些是参考规则，哪些是基于业务理解扩充
则：使用 patterns 文件中的规则发现的问题
   - 📝 基于业务理解扩充：根据业务特点自行构造检查发现的问题
5. **提供可执行方案**：不仅指出问题，还要提供具体的修复代码
