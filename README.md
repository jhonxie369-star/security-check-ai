# Security-Check-AI

**端侧AI驱动**的代码安全检查工具，帮助开发者发现代码中的安全问题。

## 🎯 核心能力

### 1. 代码层面漏洞检测
通过grep搜索 + **端侧AI深度分析**，发现：
- **注入类漏洞**：SQL注入、命令注入、路径穿越、XSS、SSRF、XXE
- **反序列化漏洞**：pickle.loads、ObjectInputStream等危险函数
- **文件上传漏洞**：类型验证、路径过滤、权限配置
- **认证授权问题**：未授权访问、越权访问、JWT/Session安全

### 2. 业务逻辑漏洞检测
**端侧AI理解业务场景**，发现：
- **价格篡改**：前端传入金额未后端验证
- **库存超卖**：并发扣减非原子操作
- **优惠券滥用**：缺少使用次数、互斥规则验证
- **状态机绕过**：订单状态流转未验证
- **限流缺失**：关键接口无频率限制
- **幂等性缺失**：重复提交导致重复扣款

### 3. 敏感信息泄露检测
扫描代码和配置文件：
- **硬编码密钥**：代码中的password、secret、api_key
- **日志泄露**：日志输出密码、Token等敏感信息
- **响应泄露**：API返回不该暴露的字段
- **配置泄露**：.properties/.yml中的明文密钥
- **CI/CD泄露**：GitHub Actions等配置中的密钥

### 4. 微服务安全检测
针对微服务架构：
- **服务间认证**：RestTemplate/HttpClient是否携带认证头
- **证书验证**：是否禁用TLS证书验证（InsecureSkipVerify）
- **内部接口暴露**：内部API是否限制访问来源
- **超时配置**：HTTP客户端是否设置超时防止hang住

### 5. 配置安全检测
检查配置文件和环境：
- **调试模式**：生产环境是否开启debug
- **敏感端点**：/actuator、/debug等端点是否暴露
- **CORS配置**：是否允许任意域名跨域访问
- **容器安全**：Dockerfile是否使用root、privileged模式

## 🎯 特点

- **端侧AI分析**：利用AI深度理解代码逻辑和业务场景
- **无外部依赖**：不需要安装额外工具，只需grep/find
- **低误报率**：通过上下文分析减少误报
- **业务逻辑检测**：不只发现代码漏洞，更发现业务逻辑漏洞

## 📦 目录结构

```
security-check-ai/
├── SKILL.md                    # 主入口，使用指南
├── ARCHITECTURE.md             # 架构设计文档
├── README.md                   # 本文件
├── report-format.md            # 报告格式规范
│
├── knowledge/                  # 安全知识库
│   ├── basic-security.md
│   ├── payment-security.md
│   ├── risk-control.md
│   └── ...
│
├── patterns/                   # 语言特定检查模式
│   ├── ai-analysis-guide.md   # AI分析指导原则
│   ├── java/
│   ├── python/
│   ├── go/
│   └── javascript/
│
├── payloads/                   # 漏洞验证Payload库
│   ├── sqli/
│   ├── cmd-injection/
│   └── ...
│
└── lessons-learned/            # 经验记录
    ├── false-positives.md
    └── ...
```

## 🚀 快速开始

### 1. 调用技能

```bash
# 在项目目录下
/security-check-ai
```

### 2. AI会自动执行

1. 分析项目结构和技术栈
2. 识别业务类型
3. 逐项检查安全问题
4. 生成详细报告

### 3. 查看报告

报告保存在 `security-reports/{timestamp}/` 目录下。

## 📋 检查项

### 1. 基础代码安全（10项）
- SQL注入、命令注入、路径穿越
- XSS、XXE、SSRF
- 反序列化、文件上传、CSRF、点击劫持

### 2. 认证授权（5项）
- 未授权访问、越权访问（IDOR）
- JWT/Token安全、Session安全、密码存储

### 3. 业务逻辑安全（7项）
- 价格验证、库存控制、优惠券验证
- 状态机验证、限流防刷、幂等性、真实IP获取

### 4. 敏感信息泄露（6项）
- 硬编码密钥、日志泄露、响应泄露
- 错误信息泄露、配置文件泄露、CI/CD密钥泄露

### 5. 配置安全（3项）
- 调试模式、敏感端点、CORS配置

### 6. 容器安全（3项，可选）
- Dockerfile安全、容器权限配置、Secret管理

### 7. 微服务安全（4项）
- 服务间认证、TLS证书验证、内部接口暴露、超时配置

**总计：38项安全检查**（其中3项为容器化项目可选）

## 🔍 工作原理

### 三步检测流程

```
第1步：grep定位
├─ 搜索关键函数（executeQuery、system、open等）
├─ 搜索关键配置（debug、cors、timeout等）
└─ 搜索敏感信息（password、secret、token等）

第2步：端侧AI分析
├─ 读取完整上下文（前后50-100行代码）
├─ 分析参数来源（用户输入 vs 配置文件 vs 常量）
├─ 检查验证逻辑（是否有输入验证、权限检查）
├─ 理解业务逻辑（价格计算、库存扣减、状态流转）
└─ 判断真实风险（区分测试代码、示例代码、生产代码）

第3步：生成报告
├─ 漏洞原理说明
├─ 代码位置和片段
├─ 风险等级评估
└─ 修复建议和示例
```

### 与传统工具对比

| 维度 | 传统SAST工具 | Security-Check-AI |
|-----|-------------|-------------------|
| **安装** | 需要安装工具 | 无需安装，直接使用 |
| **依赖** | 需要规则库 | 无外部依赖 |
| **检测方式** | 正则匹配 | grep + **端侧AI深度分析** |
| **误报率** | 高（30-50%） | 低（AI理解上下文） |
| **业务逻辑** | 无法检测 | 可检测（价格、库存等） |
| **学习成本** | 需要学习工具 | 零学习成本 |

### 核心优势

1. **端侧AI理解代码逻辑**
   - 不只是关键词匹配，而是理解代码的真实意图
   - 能区分：`password = "test123"` (测试) vs `password = "prod123"` (生产)
   - 能判断：参数是否经过验证、是否使用安全API

2. **业务逻辑理解**
   - 理解电商场景：价格应该后端计算，不能信任前端
   - 理解支付场景：订单状态流转必须验证
   - 理解风控场景：限流、幂等性、真实IP获取

3. **低误报率**
   - 传统工具：看到 `executeQuery` + 字符串拼接就报SQL注入
   - 端侧AI版：分析参数来源，如果是配置文件的表名则不报
   - 结果：误报率降低50%以上

## 💡 适用场景

- **代码安全自查**：开发者检查自己代码的安全问题
- **代码审计**：人工审计前的自动化预筛查
- **安全培训**：帮助开发者理解常见安全问题
- **业务逻辑审计**：电商、支付、金融等业务系统

## 📖 文档

- [SKILL.md](SKILL.md) - 详细使用指南
- [ARCHITECTURE.md](ARCHITECTURE.md) - 架构设计文档
- [patterns/ai-analysis-guide.md](patterns/ai-analysis-guide.md) - AI分析指导原则

## 🆚 与其他工具对比

### vs 传统SAST工具（SonarQube、Checkmarx等）
| 对比项 | 传统SAST | Security-Check-AI |
|-------|---------|-------------------|
| 安装部署 | 需要安装配置 | 无需安装 |
| 学习成本 | 需要学习规则配置 | 零学习成本 |
| 误报率 | 30-50% | 显著降低（AI理解上下文） |
| 业务逻辑 | 无法检测 | 可检测 |

### vs 专业工具（Joern、CodeQL等）
| 对比项 | 专业工具 | Security-Check-AI |
|-------|---------|-------------------|
| 精确度 | 高（污点追踪） | 中（AI分析） |
| 安装难度 | 复杂 | 极简 |
| 适用场景 | 大型项目深度分析 | 快速安全评估 |
| 业务逻辑 | 弱 | 强 |

### 选择建议
- **代码自查**：Security-Check-AI（快速、易用）
- **深度审计**：专业工具（Joern、CodeQL）
- **合规要求**：传统SAST（有认证报告）

## 📊 匿名上报（可选）

Security-Check-AI 提供可选的匿名漏洞统计上报功能，帮助改进工具。

### 隐私保护承诺

- ✅ **完全匿名**：使用UUID标识，不记录项目、用户信息
- ✅ **不含代码**：绝不上传代码片段、密钥、敏感信息
- ✅ **用户审核**：上报前显示内容供您确认
- ✅ **失败无影响**：上报失败不影响检查流程

### 两种上报模式

**仅统计模式**（推荐）：只上传漏洞类型和数量
```json
{
  "vulnerability_statistics": {
    "SQL注入": 3,
    "XSS": 1,
    "硬编码密钥": 5
  }
}
```

**抽象详情模式**：上传抽象化描述（不含代码）
```json
{
  "findings": [{
    "vulnerability_type": "SQL注入",
    "description": "发现SQL拼接，未使用参数化查询"
  }]
}
```

配置文件 `report-config.yaml`：
```yaml
report_mode: "statistics_only"  # 或 disabled / abstract_details
show_preview_before_report: true
```

详见 [REPORTING.md](REPORTING.md)

## 🔧 技术栈支持

- Java (Spring Boot, Spring MVC)
- Python (Django, Flask, FastAPI)
- Go (Gin, Echo, Beego)
- JavaScript/TypeScript (Express, Koa, NestJS)

## 📝 报告示例

```markdown
## SQL注入漏洞

**风险等级**：严重

**文件位置**：src/main/java/com/example/UserController.java:42

**代码片段**：
```java
40: String userId = request.getParameter("id");
41: String sql = "SELECT * FROM users WHERE id = " + userId;
42: stmt.executeQuery(sql);
```

**漏洞原理**：
用户输入直接拼接到SQL语句中，未使用参数化查询，存在SQL注入风险。

**修复建议**：
```java
String userId = request.getParameter("id");
String sql = "SELECT * FROM users WHERE id = ?";
PreparedStatement stmt = conn.prepareStatement(sql);
stmt.setString(1, userId);
ResultSet rs = stmt.executeQuery();
```
```

## 🤝 贡献

欢迎提交Issue和PR！

## 📄 许可

MIT License
