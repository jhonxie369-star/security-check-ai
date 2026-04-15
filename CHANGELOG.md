# 更新日志

## [1.0.0] - 2026-04-15

### ✨ 首次发布

#### 核心特性
- 🎯 AI驱动的纯静态代码安全检查
- 🚀 无需外部工具依赖
- 🔒 完全本地运行，保护代码隐私
- 🧠 深度理解代码逻辑和业务场景
- 📊 低误报率，高准确性

#### 支持的语言
- ✅ Java (Spring Boot, Spring MVC, MyBatis)
- ✅ Python (Django, Flask, FastAPI)
- ✅ Go (Gin, Echo, Beego)
- ✅ JavaScript/TypeScript (Express, Koa, NestJS)

#### 检查项（30+）
- **基础安全**：SQL注入、命令注入、路径穿越、XSS、SSRF、XXE、反序列化等
- **认证授权**：未授权访问、越权、JWT、Session、密码存储
- **业务逻辑**：价格验证、库存控制、优惠券、状态机、限流、幂等性
- **敏感信息**：硬编码密钥、日志泄露、响应泄露、错误信息泄露
- **配置安全**：调试模式、敏感端点、CORS配置

#### 文档
- 📖 SKILL.md - 完整使用指南
- 🏗️ ARCHITECTURE.md - 架构设计文档
- 🚀 QUICK-START.md - 5分钟快速上手
- 📋 HOW-TO-USE-PATTERNS.md - Patterns使用指南
- 📝 report-format.md - 报告格式规范

#### Patterns文件（24个）
- Java: 5个（basic-security, auth, business-logic, sensitive-data, config）
- Python: 5个
- Go: 5个
- JavaScript: 5个
- Cross-service: 2个
- Cross-language: 2个

#### 知识库
- 7个安全知识文档
- 完整的Payload模板库
- 经验教训记录

### 🔧 技术实现
- 使用grep进行代码搜索定位
- AI分析完整代码上下文
- 智能判断真实风险
- 生成详细修复建议

### 📦 目录结构
```
security-check-ai/
├── SKILL.md                 # 主入口
├── QUICK-START.md           # 快速开始
├── patterns/                # 检查模式（24个文件）
├── knowledge/               # 知识库（7个文件）
├── payloads/                # Payload库
└── lessons-learned/         # 经验记录
```

### 🎯 与Joern版对比
- ✅ 无需安装复杂工具
- ✅ 无需学习Scala语法
- ✅ 更好的业务逻辑理解
- ✅ 更低的误报率
- ✅ 完全本地运行

---

## 未来计划

### v1.1.0（计划中）
- [ ] 增量扫描支持
- [ ] 自定义规则配置
- [ ] HTML格式报告
- [ ] 漏洞优先级排序

### v1.2.0（计划中）
- [ ] 支持更多语言（PHP, Ruby, Rust）
- [ ] 集成CI/CD
- [ ] API接口
- [ ] Web界面

### v2.0.0（远期）
- [ ] 实时代码检查
- [ ] IDE插件
- [ ] 团队协作功能
- [ ] 漏洞趋势分析
