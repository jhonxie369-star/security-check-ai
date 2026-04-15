# Payload 模板库

安全测试 Payload 模板集合，用于漏洞验证和安全评估。

## 目录结构

```
payloads/
├── payload-assembly-guide.md    # Payload 组装指南
├── README.md                    # 本文件
│
├── sqli/                        # SQL 注入
│   ├── templates/
│   │   ├── mysql.md
│   │   ├── postgresql.md
│   │   ├── mssql.md
│   │   └── oracle.md
│   └── bypass/
│       └── waf-bypass.md
│
├── cmd-injection/               # 命令注入
│   ├── templates/
│   │   └── common.md
│   └── bypass/
│       └── waf-bypass.md
│
├── xss/                         # 跨站脚本
│   ├── templates/
│   │   └── common.md
│   └── bypass/
│       └── waf-bypass.md
│
├── ssrf/                        # 服务端请求伪造
│   ├── templates/
│   │   └── common.md
│   └── bypass/
│       └── waf-bypass.md
│
├── path-traversal/              # 目录穿越
│   ├── templates/
│   │   └── common.md
│   └── bypass/
│       └── waf-bypass.md
│
├── deserialization/             # 反序列化
│   ├── templates/
│   │   └── common.md
│   └── bypass/
│
└── business-logic/              # 业务逻辑漏洞
    ├── templates/
    │   └── common.md
    └── bypass/
```

## 使用方法

### 1. 环境识别

首先识别目标环境：

```bash

# 或使用外部扫描
nmap -sV target.com
whatweb http://target.com
```

### 2. 漏洞类型选择

根据漏洞类型选择对应目录：

| 漏洞类型 | 目录 | CWE |
|---------|------|-----|
| SQL 注入 | sqli/ | CWE-89 |
| 命令注入 | cmd-injection/ | CWE-78 |
| XSS | xss/ | CWE-79 |
| SSRF | ssrf/ | CWE-918 |
| 目录穿越 | path-traversal/ | CWE-22 |
| 反序列化 | deserialization/ | CWE-502 |
| 业务逻辑 | business-logic/ | - |

### 3. Payload 选择

根据目标环境选择合适的 Payload：

```
1. 查看 templates/ 目录中的基础 Payload
2. 如果基础 Payload 被拦截，查看 bypass/ 目录中的绕过技术
3. 参考 payload-assembly-guide.md 进行环境适配
```

### 4. 验证流程

```
1. 测试基础 Payload
2. 观察响应，判断是否成功
3. 如失败，尝试绕过技术
4. 确认漏洞后进行深入利用
```

## Payload 分类

### 按检测方法

| 类型 | 说明 | 示例 |
|-----|------|-----|
| 回显 | 直接在响应中看到结果 | UNION SELECT |
| 报错 | 通过错误信息获取数据 | EXTRACTVALUE() |
| 盲注 | 通过条件判断获取数据 | SLEEP() |
| 带外 | 通过外部通道获取数据 | DNS/HTTP 外带 |

### 按危害程度

| 级别 | 说明 | 示例漏洞 |
|-----|------|---------|
| Critical | 直接获取系统控制权 | RCE, SQLi (堆叠查询) |
| High | 获取敏感数据或权限提升 | SQLi, 命令注入 |
| Medium | 需要特定条件才能利用 | XSS, SSRF (有限制) |
| Low | 信息泄露或需要用户交互 | 目录穿越 (只读) |


## 安全声明

**本 Payload 库仅用于授权的安全测试。**

使用本库进行未授权的测试可能违反法律。请确保：

1. 获得明确的书面授权
2. 在授权范围内进行测试
3. 遵守测试协议和边界
4. 保护测试过程中获取的数据
5. 测试完成后清理测试数据

## 参考资料

- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)
- [HackTricks](https://book.hacktricks.xyz/)

## 更新日志

- 2024-01: 初始版本，包含 SQL 注入、命令注入、XSS、SSRF、目录穿越、反序列化、业务逻辑漏洞模板