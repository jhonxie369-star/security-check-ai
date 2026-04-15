# 快速开始指南

## 🚀 5分钟上手

### 第1步：准备项目

```bash
cd /path/to/your/project
```

### 第2步：调用检查

在Claude Code中输入：
```
/security-check-ai
```

### 第3步：等待分析

AI会自动：
1. 识别项目语言和框架
2. 分析业务类型
3. 执行安全检查
4. 生成详细报告

### 第4步：查看报告

报告保存在 `security-reports/{timestamp}/` 目录。

---

## 📋 检查内容

### 基础安全（10项）
- ✅ SQL注入
- ✅ 命令注入
- ✅ 路径穿越
- ✅ XSS
- ✅ SSRF
- ✅ XXE
- ✅ 反序列化
- ✅ 文件上传
- ✅ CSRF
- ✅ 点击劫持

### 认证授权（5项）
- ✅ 未授权访问
- ✅ 越权访问（IDOR）
- ✅ JWT安全
- ✅ Session安全
- ✅ 密码存储

### 业务逻辑（7项）
- ✅ 价格验证
- ✅ 库存控制
- ✅ 优惠券验证
- ✅ 状态机验证
- ✅ 限流防刷
- ✅ 幂等性
- ✅ 真实IP获取

### 敏感信息（6项）
- ✅ 硬编码密钥
- ✅ 日志泄露
- ✅ 响应泄露
- ✅ 错误信息泄露
- ✅ 配置文件泄露
- ✅ CI/CD密钥泄露

### 配置安全（3项）
- ✅ 调试模式
- ✅ 敏感端点
- ✅ CORS配置

### 容器安全（3项，可选）
- ✅ Dockerfile安全
- ✅ 容器权限配置
- ✅ Secret管理

### 微服务安全（4项）
- ✅ 服务间认证
- ✅ TLS证书验证
- ✅ 内部接口暴露
- ✅ 超时配置

---

## 💡 使用技巧

### 1. 指定检查范围

```
只检查SQL注入和XSS
```

### 2. 指定文件

```
检查 src/main/java/UserController.java 的安全问题
```

### 3. 业务逻辑检查

```
这是一个电商项目，重点检查订单和支付相关的业务逻辑漏洞
```

### 4. 快速扫描

```
快速扫描，只报告严重和高危漏洞
```

---

## 🎯 支持的技术栈

### 后端语言
- ✅ Java (Spring Boot, Spring MVC, MyBatis)
- ✅ Python (Django, Flask, FastAPI)
- ✅ Go (Gin, Echo, Beego)
- ✅ JavaScript/TypeScript (Express, Koa, NestJS)

### 数据库
- ✅ MySQL
- ✅ PostgreSQL
- ✅ MongoDB
- ✅ Redis

### 框架特性
- ✅ ORM (MyBatis, Hibernate, Sequelize, GORM)
- ✅ 模板引擎 (Thymeleaf, Jinja2, Go Template)
- ✅ 认证框架 (Spring Security, JWT, OAuth2)

---

## 📊 报告示例

```markdown
# 安全检查报告

## 概览
- 检查时间：2026-04-15 16:00:00
- 项目语言：Java
- 框架：Spring Boot
- 发现问题：15个（严重3个，高危5个，中危7个）

## 严重漏洞

### 1. SQL注入 - UserController.java:42
**风险等级**：🔴 严重

**代码位置**：
```java
40: String userId = request.getParameter("id");
41: String sql = "SELECT * FROM users WHERE id = " + userId;
42: stmt.executeQuery(sql);
```

**漏洞原理**：
用户输入直接拼接到SQL语句，未使用参数化查询。

**修复建议**：
```java
String sql = "SELECT * FROM users WHERE id = ?";
PreparedStatement stmt = conn.prepareStatement(sql);
stmt.setString(1, userId);
```

**影响范围**：
攻击者可读取、修改、删除数据库中的任意数据。
```

---

## ❓ 常见问题

### Q1: 检查需要多长时间？
- 小型项目（<1000文件）：5-10分钟
- 中型项目（1000-5000文件）：15-30分钟
- 大型项目（>5000文件）：30-60分钟

### Q2: 会修改我的代码吗？
不会。这是纯静态分析工具，只读取代码，不会修改任何文件。

### Q3: 误报率高吗？
AI版通过上下文分析，误报率比传统工具低50%以上。

### Q4: 支持增量扫描吗？
暂不支持。每次都是全量扫描。

### Q5: 可以自定义检查规则吗？
可以。修改 `patterns/` 目录下的对应文件即可。

---

## 🔗 更多文档

- [SKILL.md](SKILL.md) - 完整使用指南
- [ARCHITECTURE.md](ARCHITECTURE.md) - 架构设计
- [HOW-TO-USE-PATTERNS.md](HOW-TO-USE-PATTERNS.md) - 如何使用patterns文件

---

## 🆘 获取帮助

遇到问题？
1. 查看 [SKILL.md](SKILL.md) 详细文档
2. 查看 [lessons-learned/](lessons-learned/) 常见问题
3. 提交 Issue

---

**开始检查吧！** 🚀
