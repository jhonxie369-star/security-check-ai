---
name: security-check-ai
description: AI驱动的纯静态代码安全检查工具，无需外部工具，通过AI理解代码逻辑发现安全问题
---

# AI驱动代码安全检查

## ⚠️ 重要说明

**本工具复用了原版security-check的patterns文件，但使用方式不同：**

- **patterns文件内容**：与原版完全一致（包含Joern查询示例）
- **AI使用方式**：将Joern查询转换为grep搜索 + AI分析
- **详细指导**：参见 `HOW-TO-USE-PATTERNS.md`

**核心转换规则**：
- Joern的`cpg.call.name(".*XXX.*")` → grep的`grep -rn "XXX"`
- Joern的污点追踪 → AI读取完整上下文并分析数据流
- patterns中的检查要点、修复示例、风险判断表 → 完全适用

## 🎯 核心原则

1. **AI理解优先**：利用AI的代码理解能力，无需外部工具
2. **业务逻辑分析**：深入理解业务场景，识别业务逻辑漏洞
3. **上下文关联**：通过代码上下文分析数据流和调用链
4. **可执行建议**：提供具体的修复代码
5. **复用patterns**：patterns文件内容不变，AI转换使用方式

## 📋 检查流程

### ⚠️ 强制约束
1. **逐项检查**：每次只检查一个检查项
2. **即时输出**：每检查完一项，立即输出该项报告
3. **即时保存**：输出报告后，立即保存到 `security-reports/` 目录
4. **等待确认**：保存报告后，询问用户是否继续下一项
5. **禁止跳跃**：不得跳过任何检查项

## 📖 检查步骤

### 第一步：项目分析

**目标**：理解项目结构、技术栈、业务类型

```bash
# 1. 识别项目语言和框架
find . -name "pom.xml" -o -name "go.mod" -o -name "package.json" -o -name "requirements.txt"

# 2. 识别项目结构
tree -L 3 -I 'node_modules|target|dist|build'

# 3. 识别业务类型（通过关键文件名）
find . -type f -name "*Order*" -o -name "*Payment*" -o -name "*Coupon*" | head -20
```

**输出内容**：
- 项目语言（Java/Go/Python/JavaScript）
- 使用框架（Spring/Gin/Flask/Express等）
- 业务类型（支付/订单/优惠券/用户系统等）
- 推荐检查顺序

### 第二步：理解安全知识

**必读文件**：
- `knowledge/basic-security.md` - 基础安全理论（所有项目必读）

**根据业务类型选读**：
- 支付/订单/转账 → `knowledge/payment-security.md`
- 优惠券/积分/邀请 → `knowledge/risk-control.md`
- API 接口 → `knowledge/api-security.md`

### 第三步：执行代码检查

## 🔍 AI检查方法论

### 方法1：关键词搜索 + AI分析

**适用场景**：注入类漏洞、敏感信息泄露

**步骤**：
1. 使用 `grep` 搜索危险函数/关键词
2. 读取匹配文件的完整上下文（前后100行）
3. AI分析：是否有验证、是否使用安全API
4. 判断真实风险

**示例**：SQL注入检查
```bash
# 1. 搜索SQL执行点
grep -rn "executeQuery\|execute\|query\|Query" --include="*.java" | head -50

# 2. 对每个匹配，读取文件上下文
# 3. AI分析：
#    - 参数来源（用户输入 vs 配置文件）
#    - 是否使用PreparedStatement
#    - 是否有输入验证
```

### 方法2：业务流程追踪

**适用场景**：业务逻辑漏洞、认证授权

**步骤**：
1. 识别关键业务流程（订单创建、支付、转账）
2. 读取完整业务代码
3. AI分析数据流：输入 → 验证 → 处理 → 输出
4. 识别缺失的验证环节

**示例**：价格验证检查
```bash
# 1. 找到订单创建接口
grep -rn "@PostMapping.*order\|def.*create.*order" --include="*.java" --include="*.py"

# 2. 读取完整方法实现
# 3. AI分析：
#    - 价格来源（前端 vs 后端计算）
#    - 是否有价格验证
#    - 是否可能负数金额
```

### 方法3：模式匹配 + 上下文验证

**适用场景**：配置安全、密码存储

**步骤**：
1. 搜索特定模式（硬编码密钥、弱加密）
2. 读取上下文确认用途
3. AI判断风险等级

**示例**：硬编码密钥检查
```bash
# 1. 搜索可疑字符串 -rn "password.*=.*['\"].*['\"]" --include="*.java" --include="*.py"

# 2. AI分析：
#    - 是否是测试代码
#    - 是否是示例/注释
#    - 是否真实使用
```

## 📝 检查项执行流程

### 检查项1：基础代码安全（注入类漏洞）

**子项1.1：SQL注入**

```bash
# Java
grep -rn "executeQuery\|executeUpdate\|createQuery" --include="*.java" -A 5 -B 5

# Python
grep -rn "execute\|cursor\|session.query" --include="*.py" -A 5 -B 5

# Go
grep -rn "db.Query\|db.Exec\|db.Raw" --include="*.go" -A 5 -B 5

# JavaScript
grep -rn "query\|execute" --include="*.js" -A 5 -B 5
```

**AI分析要点**：
- 参数来源：`request.getParameter` / `request.args` / `c.Query` / `req.query`
- 是否使用参数化查询：`PreparedStatement` / `?` 占位符
- 是否有输入验证：`validate` / `sanitize` 函数

**子项1.2：命令注入**

```bash
# Java
grep -rn "Runtime.exec\|ProcessBuilder" --include="*.java" -A 5 -B 5

# Python
grep -rn "os.system\|subprocess\|exec\|eval" --include="*.py" -A 5 -B 5

# Go
grep -rn "exec.Command\|os.Exec" --include="*.go" -A 5 -B 5

# JavaScript
grep -rn "exec\|spawn\|eval" --include="*.js" -A 5 -B 5
```

**子项1.3：路径穿越**

```bash
grep -rn "File\|Path\|readFile\|write" --include="*.java" --include="*.py" --include="*.go" --include="*.js" -A 5 -B 5
```

**子项1.4：XSS**

```bash
grep -rn "innerHTML\|document.write\|eval\|dangerouslySetInnerHTML" --include="*.js" --include="*.jsx" --include="*.html" -A 5 -B 5
```

**保存报告**：`security-reports/{timestamp}/01-基础代码安全.md`

### 检查项2：敏感信息泄露

**子项2.1：硬编码密钥**

```bash
# 搜索密钥模式
grep -rn "password\s*=\s*['\"].*['\"]" --include="*.java" --include="*.py" --include="*.go" --include="*.js"
grep -rn "secret\s*=\s*['\"].*['\"]" --include="*.java" --include="*.py" --include="*.go" --include="*.js"
grep -rn "api_key\s*=\s*['\"].*['\"]" --include="*.java" --include="*.py" --include="*.go" --include="*.js"
```

**子项2.2：配置文件泄露**

```bash
# 查找配置文件
find . -name "*.properties" -o -name "*.yml" -o -name "*.yaml" -o -name ".env" -o -name "config.json"

# 检查敏感配置
grep -rn "password\|secret\|key\|token" --include="*.properties" --include="*.yml" --include="*.env"
```

**子项2.3：日志泄露**

```bash
grep -rn "log.*password\|log.*token\|log.*secret" --include="*.java" --include="*.py" --include="*.go" --include="*.js" -A 2 -B 2
```

**保存报告**：`security-reports/{timestamp}/02-敏感信息泄露.md`

### 检查项3：认证授权

**子项3.1：未授权访问**

```bash
# Java Spring
grep -rn "@RequestMapping\|@GetMapping\|@PostMapping" --include="*.java" -A 10

# Python Flask
grep -rn "@app.route\|@blueprint.route" --include="*.py" -A 10

# Go Gin
grep -rn "router.GET\|router.POST\|r.GET\|r.POST" --include="*.go" -A 10

# JavaScript Express
grep -rn "app.get\|app.post\|router.get\|router.post" --include="*.js" -A 10
```

**AI分析要点**：
- 是否有权限注解：`@PreAuthorize` / `@login_required` / `authMiddleware`
- 是否有全局拦截器
- 是否有白名单机制

**子项3.2：越权访问（IDOR）**

```bash
# 查找ID参数查询
grep -rn "getById\|findById\|selectById\|get.*id" --include="*.java" --include="*.py" --include="*.go" --include="*.js" -A 10 -B 5
```

**AI分析要点**：
- 查询前是否验证所有权
- SQL是否包含 `WHERE user_id = ?`
- 是否有统一的权限检查

**子项3.3：JWT/Token安全**

```bash
grep -rn "JWT\|jwt\|token\|Token" --include="*.java" --include="*.py" --include="*.go" --include="*.js" -A 10 -B 5
```

**AI分析要点**：
- 密钥是否硬编码
- 是否验证签名
- 是否验证过期时间

**保存报告**：`security-reports/{timestamp}/03-认证授权.md`

### 检查项4：业务逻辑安全

**⚠️ 核心流程：理解业务 → 识别风险点 → 分析代码 → 判断漏洞**

**子项4.1：价格验证**

```bash
# 查找订单/支付相关代码
grep -rn "amount\|price\|tal\|payment" --include="*.java" --include="*.py" --include="*.go" --include="*.js" | grep -i "create\|update\|order\|pay"
```

**AI分析要点**：
- 价格来源（前端传入 vs 后端计算）
- 是否验证负数
- 是否验证价格一致性

**子项4.2：库存控制**

```bash
grep -rn "stock\|inventory\|quantity" --include="*.java" --include="*.py" --include="*.go" --include="*.js" -A 10 -B 5
```

**AI分析要点**：
- 是否使用原子操作（Redis DECR / 数据库锁）
- 是否有并发控制
- 是否有超卖风险

**子项4.3：状态机验证**

```bash
grep -rn "status\|state\|orderStatus" --include="*.java" --include="*.py" --include="*.go" --include="*.js" | grep -i "update\|change\|set"
```

**AI分n- 状态转换是否有验证
- 是否可以跳过状态
- 是否可以回退状态

**子项4.4：优惠券/折扣滥用**

```bash
grep -rn "coupon\|discount\|voucher\|promo" --include="*.java" --include="*.py" --include="*.go" --include="*.js" -A 10 -B 5
```

**AI分析要点**：
- 是否验证使用次数
- 是否验证有效期
- 是否可以重复使用

**子项4.5：限流/防刷**

```bash
grep -rn "ratelimit\|throttle\|limiter\|RateLimit" --include="*.java" --include="*.py" --include="*.go" --include="*.js"
```

**AI分析要点**：
- 关键接口是否有限流
- 限流粒度（IP/用户/全局）
- 是否使用分布式限流

**保存报告**：`security-reports/{timestamp}/04-业务逻辑安全.md`

### 检查项5全

```bash
# 查找配置文件
find . -name "*.yml" -o -name "*.yaml" -o -name "*.properties" -o -name ".env" -o -name "config.*"

# 检查危险配置
grep -rn "debug.*true\|DEBUG.*True\|test.*true" --include="*.yml" --include="*.properties" --include="*.env"
grep -rn "password\|secret\|key" --include="*.yml" --include="*.properties" --include="*.env"
```

**AI分析要点**：
- 是否开启调试模式
- 是否暴露敏感信息
- 是否使用默认密码

**保存报告**：`security-reports/{timestamp}/05-配置安全.md`

### 检查项6：SSRF（服务端请求伪造）

```bash
# Java
grep -rn "HttpClient\|HttpURLConnection\|RestTemplate" --include="*.java" -A 10 -B 5

# Python
grep -rn "requests.get\|requests.post\|urllib" --include="*.py" -A 10 -B 5

# Go
grep -rn "http.Get\|http.Post\|http.Client" --include="*.go" -A 10 -B 5

# JavaScript
grep -rn "axios\|fetch\|request\|http.get" --include="*.js" -A 10 -B 5
```

**AI分析要点**：
- URL是否来自用户输入
- 是否有URL白名单
- 是否禁止内网IP
- 是否禁止危险协议（file://）

**保存报告**：`security-reports/{timestamp}/06-SSRF检查.md`

## 🔓 漏洞验证

**⚠️ 代码扫描完成后的漏洞验证阶段**

### 第一步：获取测试地址

询问用户提供测试环境地址

### 第二步：Payload测试

参考 `payloads/` 目录中的Payload模板

## 📊 报告格式

参考 `report-format.md`

每个检查项报告包含：
- 漏洞类型
- 风险等级（严重/高危/中危/低危/信息）
- 影响范围
- 漏洞位置（文件:行号）
- 代码片段
- 漏洞原理
- 修复建议（含代码示例）

## 💡 AI使用要点

### DO ✅

1. **深度理解代码**：不只看关键词，理解完整逻辑
2. **上下文分析**：读取足够的上下文（前后50-100行）
3. **业务理解**：结合业务场景判断风险
4. **减少误报**：区分测试代码、示例代码、真实代码
5. **提供修复**：给出可执行的修复代码

### DON'T ❌

1. **机械匹配**：不要只看关键词就报告漏洞
2. **忽略上下文**：不要忽略验证函数、安全API
3. **忽略业务**：不要忽略业务逻辑漏洞
4. **过度报告**：不要把测试代码当成漏洞

## 📁 文件索引

### 知识库
- `knowledge/basic-security.md` - 基础安全理论
- `knowledge/payment-security.md` - 支付系统安全
- `knowledge/risk-control.md` - 风控与防薅羊毛
- `knowledge/api-security.md` - API安全
- `knowledge/business-landing.md` - 业务逻辑理解方法

### 语言特定模式
- `patterns/java/` - Java检查模式
- `patterns/go/` - Go检查模式
- `patterns/python/` - Python检查模式
- `patterns/javascript/` - JavaScript检查模式

### Payload库
- `payloads/sqli/` - SQL注入Payload
- `payloads/cmd-injection/` - 命令注入Payload
- `payloads/xss/` - XSS Payload
- `payloads/ssrf/` - SSRF Payload

## 🎯 总结

**核心理念**：
- AI理解 > 工具扫描
- 业务逻辑 > 代码模式
- 上下文分析 > 关键词匹配
- 深度分析 > 表面检查

**优势**：
- 无需外部工具（Joern等）
- 端侧AI分析，深度理解代码逻辑
- 灵活适应各种项目
- 深度理解业务逻辑
