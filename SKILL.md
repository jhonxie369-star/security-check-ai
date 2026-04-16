---
name: security-check-ai
description: AI驱动的纯静态代码安全检查工具，无需外部工具，通过AI理解代码逻辑发现安全问题
---

# AI驱动代码安全检查

## 🎯 核心原则

1. **AI理解优先**：利用AI的代码理解能力，无需外部工具
2. **业务逻辑分析**：深入理解业务场景，识别业务逻辑漏洞
3. **上下文关联**：通过代码上下文分析数据流和调用链
4. **可执行建议**：提供具体的修复代码

## 📋 检查流程

### ⚠️ 执行方式
1. **一次性检查**：自动执行所有检查项，不需要每项都询问用户
2. **即时输出**：每检查完一项，立即输出该项报告
3. **即时保存**：输出报告后，立即保存到 `security-reports/{timestamp}/` 目录
4. **禁止跳跃**：不得跳过任何检查项，必须按顺序完成所有38项检查
5. **最后汇总**：所有检查完成后，生成汇总报告并询问是否上报
6. **完整覆盖**：确保每一项都被检查，即使某项未发现问题也要记录

### 📝 检查项列表（共38项）

**基础安全漏洞（14项）**：
1. SQL注入
2. XSS跨站脚本
3. SSRF服务端请求伪造
4. 命令注入
5. 路径遍历
6. XXE外部实体注入
7. 反序列化漏洞
8. CSRF跨站请求伪造
9. 不安全的文件上传
10. 点击劫持
11. 开放重定向
12. 不安全的随机数
13. 硬编码密钥
14. 弱加密算法

**认证与授权（6项）**：
15. JWT安全问题
16. Session管理问题
17. 密码存储安全
18. RBAC权限控制
19. 未授权访问
20. 权限提升漏洞

**业务逻辑安全（8项）**：
21. 价格篡改
22. 库存控制
23. 优惠券验证
24. 状态机漏洞
25. 限流绕过
26. 幂等性问题
27. 真实IP获取
28. 时间窗口竞态

**敏感数据处理（5项）**：
29. 硬编码敏感信息
30. 日志泄露
31. 响应泄露
32. 错误信息泄露
33. CI/CD密钥泄露

**微服务安全（4项）**：
34. 服务间认证
35. TLS证书验证
36. 内部API暴露
37. 超时配置

**容器安全（1项）**：
38. Dockerfile安全配置

## 📖 检查步骤

### 第零步：扫描模式选择

**检查历史报告**：
```bash
# 检查是否有历史扫描报告
ls -la ~/.security-reports/ 2>/dev/null | tail -5
```

**模式选择逻辑**：
- **有历史报告** → 询问用户：
  - 选项1：增量扫描（仅检查 git 变更的文件，快速）
  - 选项2：全量扫描（检查所有文件，完整）
- **无历史报告** → 自动选择全量扫描（首次扫描）

**排除第三方代码**：
```bash
# 检查是否有排除配置
if [ -f .security-check-exclude ]; then
    cat .security-check-exclude
else
    # 询问用户需要排除哪些目录
    echo "常见第三方目录："
    echo "  - node_modules (Node.js)"
    echo "  - vendor (Go/PHP)"
    echo "  - third_party, lib (通用)"
    echo "  - dist, build, target (构建产物)"
    echo "  - venv, .venv (Python虚拟环境)"
fi
```

**保存排除规则**：
用户首次指定排除目录后，保存到 `.security-check-exclude` 文件：
```
node_modules/
vendor/
third_party/
dist/
build/
target/
venv/
.venv/
```

**获取扫描文件列表**：
- **增量模式**：
  ```bash
  # 获取 git 变更文件（相对于上次提交）
  git diff --name-only HEAD
  # 过滤掉排除目录
  git diff --name-only HEAD | grep -v -f .security-check-exclude
  ```
- **全量模式**：
  ```bash
  # 获取所有代码文件
  find . -type f \( -name "*.java" -o -name "*.go" -o -name "*.py" -o -name "*.js" \)
  # 过滤掉排除目录
  find . -type f \( -name "*.java" -o -name "*.go" -o -name "*.py" -o -name "*.js" \) | grep -v -f .security-check-exclude
  ```

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
- 扫描模式（增量/全量）
- 扫描文件数量
- 排除的目录列表
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

**执行方式**：
- 按照38项检查列表顺序，自动执行所有检查
- **增量模式**：仅检查变更的文件，但保留全局检查项（如认证、CORS配置等）
- **全量模式**：检查所有文件（排除第三方目录）
- 每完成一项，立即输出结果并保存到 `~/.security-reports/{YYYYMMDD_HHMMSS}/` 目录
- 不需要每项都询问用户，一次性完成所有检查
- 检查过程中发现的问题立即记录，最后统一汇总

**报告组织**：
```
~/.security-reports/20260416_112030/
├── scan_info.json              # 扫描信息（模式、文件数、排除目录、时间戳）
├── 01-基础安全漏洞.md          # 包含14项基础漏洞检查结果
├── 02-认证与授权.md            # 包含6项认证授权检查结果
├── 03-业务逻辑安全.md          # 包含8项业务逻辑检查结果
├── 04-敏感数据处理.md          # 包含5项敏感数据检查结果
├── 05-微服务安全.md            # 包含4项微服务检查结果
├── 06-容器安全.md              # 包含1项容器检查结果
└── 00-汇总报告.md              # 所有问题的汇总和统计
```

**scan_info.json 格式**：
```json
{
  "scan_mode": "incremental",
  "scan_time": "2026-04-16T11:20:30",
  "project_path": "/path/to/project",
  "total_files": 15,
  "excluded_dirs": ["node_modules", "vendor", "dist"],
  "language": "java",
  "framework": "spring-boot"
}
```

### 第四步：生成汇总报告

所有检查完成后，生成汇总报告包含：
- 发现的漏洞总数（按严重等级分类）
- 各类别漏洞分布
- 高危问题列表
- 修复优先级建议

### 第五步：询问是否上报

**所有检查完成后，必须执行上报询问流程**：

1. 统计本次扫描发现的漏洞（按类型和严重等级）
2. 显示上报选项（详见下方"漏洞上报"章节）
3. 根据用户选择生成上报数据
4. 显示完整JSON供用户审核
5. 执行上报（如果用户同意）

**重要**：这是检查流程的最后一步，不可跳过。即使用户选择"不上报"，也要完成询问流程。

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

## 📊 漏洞上报（每次扫描后询问）

### 隐私保护承诺

**在上报前，请您了解：**

1. **完全匿名**：使用UUID标识扫描，不记录项目名称、路径、用户信息
2. **不含代码**：绝不上传任何代码片段、密钥、敏感信息
3. **用户选择**：每次扫描完成后都会询问您是否上报及上报方式
4. **失败无影响**：上报失败不影响安全检查正常进行
5. **改进工具**：数据用于统计常见漏洞类型，帮助改进检查规则

### 上报流程（每次扫描后执行）

**步骤1：显示上报选项**

扫描完成后，显示以下交互式提示：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 匿名漏洞统计上报（可选）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

本次扫描发现以下问题：
  • SQL注入: 3个
  • XSS: 1个  
  • 硬编码密钥: 5个
  • 未授权访问: 2个

为了帮助改进Security-Check-AI，建议匿名上报扫描统计。

🔒 隐私保护承诺：
  ✓ 仅上传抽象化的漏洞信息，不会上传代码
  ✓ 不会上传密钥、密码、Token等敏感信息
  ✓ 使用UUID匿名标识，无法追溯到具体项目
  ✓ 上报前会显示完整内容供您审核

请选择上报方式：

  1. 仅上报统计信息（推荐）
     - 只上传漏洞类型和数量
     - 示例：{"SQL注入": 3, "XSS": 1}
     - 隐私性：★★★★★

  2. 上报抽象详情
     - 上传抽象化的漏洞描述（不含代码）
     - 示例：{"类型": "SQL注入", "描述": "发现SQL拼接，未使用参数化查询"}
     - 隐私性：★★★★☆

  3. 不上报
     - 跳过上报，直接结束

您的选择 [1/2/3]：
```

**步骤2：根据用户选择生成上报内容**

**选项1 - 仅统计模式**（推荐）：
```json
{
  "scan_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "report_type": "statistics_only",
  "timestamp": "2026-04-16T10:30:00",
  "language": "java",
  "vulnerability_statistics": {
    "SQL注入": 3,
    "XSS": 1,
    "硬编码密钥": 5,
    "未授权访问": 2
  }
}
```

**选项2 - 抽象详情模式**：
```json
{
  "scan_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "report_type": "abstract_details",
  "timestamp": "2026-04-16T10:30:00",
  "language": "java",
  "findings": [
    {
      "vulnerability_type": "SQL注入",
      "severity": "严重",
      "description": "发现SQL语句拼接用户输入，未使用参数化查询",
      "pattern": "字符串拼接到executeQuery",
      "category": "注入类漏洞"
    },
    {
      "vulnerability_type": "硬编码密钥",
      "severity": "高危",
      "description": "配置文件中发现硬编码的数据库密码",
      "pattern": "properties文件包含password字段",
      "category": "敏感信息泄露"
    }
  ]
}
```

**选项3 - 不上报**：
直接跳过，结束检查流程。

**步骤3：显示上报内容供审核**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 上报内容预览
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

将要上报的内容（JSON格式）：

{
  "scan_uuid": "550e8400-e29b-41d4-a716-446655440000",
  "report_type": "statistics_only",
  "timestamp": "2026-04-16T10:30:00",
  "language": "java",
  "vulnerability_statistics": {
    "SQL注入": 3,
    "XSS": 1,
    "硬编码密钥": 5,
    "未授权访问": 2
  }
}

✓ 已确认：不包含代码、密钥、文件路径等敏感信息

确认上报？[Y/n]：
```

**步骤4：执行上报**

用户确认后，使用curl异步上报：

```bash
# 生成UUID（如果还没有）
SCAN_UUID=$(uuidgen)

# 生成上报文件
cat > /tmp/security-report-${SCAN_UUID}.json <<'EOF'
{上报内容JSON}
EOF

# 异步上报（失败不影响主流程）
(
  curl -X POST \
    -H "Content-Type: application/json" \
    -d @/tmp/security-report-${SCAN_UUID}.json \
    http://172.30.11.213:8081/report \
    --max-time 5 \
    --silent \
    --show-error \
    && echo "✓ 上报成功，感谢您的支持！" \
    || echo "✗ 上报失败（网络问题），不影响检查结果"
  
  # 清理临时文件
  rm -f /tmp/security-report-${SCAN_UUID}.json
) &
```

**步骤5：保存扫描记录**

将本次扫描信息保存到用户目录，用于下次增量扫描判断：

```bash
# 创建扫描记录目录
mkdir -p ~/.security-reports/

# 保存本次扫描的基本信息
cat > ~/.security-reports/last_scan.json <<EOF
{
  "scan_time": "$(date -Iseconds)",
  "scan_mode": "full",
  "project_path": "$(pwd)",
  "report_dir": "~/.security-reports/$(date +%Y%m%d_%H%M%S)",
  "scan_uuid": "${SCAN_UUID}"
}
EOF
```

**步骤6：继续后续流程**

无论上报成功或失败，都继续正常流程，不阻塞用户。

### 重要提醒

⚠️ **上报内容绝不包含**：
- 代码片段
- 文件路径
- 项目名称
- 密钥、密码、Token
- 任何可识别身份的信息

✅ **仅包含**：
- 漏洞类型统计
- 抽象化的漏洞描述（如选择抽象详情模式）
- 匿名UUID
- 编程语言
- 时间戳

## 🎯 总结

**核心理念**：
- AI理解 > 工具扫描
- 业务逻辑 > 代码模式
- 上下文分析 > 关键词匹配
- 深度分析 > 表面检查

**优势**：
- 端侧AI分析，深度理解代码逻辑
- 灵活适应各种项目
- 深度理解业务逻辑
- 可选的匿名上报帮助改进工具
