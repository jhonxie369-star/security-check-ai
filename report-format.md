# 报告存储规范

## 目录结构

### 单语言项目（全量扫描）
```
{项目根目录}/
└── security-reports/
    ├── .scan-state.json              # 扫描状态文件（记录历史）
    ├── 20260212-100000/              # 时间戳目录（YYYYMMdd-HHmmss）
    │   ├── scan-info.json            # 本次扫描详细信息
    │   ├── java/                     # 单语言项目直接放这里
    │   │   ├── 01-基础代码安全.md
    │   │   ├── 02-敏感信息泄露.md
    │   │   ├── 03-认证授权.md
    │   │   ├── 04-业务逻辑安全.md
    │   │   └── 05-配置安全.md
    │   └── FINAL-REPORT.md           # 汇总报告
    └── 20260212-150000/              # 下一次扫描
        └── ...
```

### 增量扫描
```
{项目根目录}/
└── security-reports/
    ├── .scan-state.json
    └── 20260212-150000/              # 增量扫描
        ├── scan-info.json
        ├── JOERN-DATAFLOW.md         # Joern数据流分析报告（增量扫描特有）
        ├── java/
        │   ├── 01-基础代码安全.md
        │   ├── 02-敏感信息泄露.md
        │   ├── 03-认证授权.md
        │   ├── 04-业务逻辑安全.md
        │   └── 05-配置安全.md
        └── FINAL-REPORT.md
```

### 多语言/多服务项目
```
{项目根目录}/
└── security-reports/
    ├── .scan-state.json
    └── 20260212-100000/
        ├── scan-info.json
        ├── java/
        │   ├── 01-基础代码安全.md
        │   └── ...
        ├── go/
        │   ├── 01-基础代码安全.md
        │   └── ...
        ├── cross-service/
        │   ├── 06-跨服务业务逻辑.md
        │   └── 07-跨服务配置安全.md
        └── FINAL-REPORT.md
```

## 状态文件格式

### .scan-state.json
位于 `security-reports/` 目录下：

```json
{
  "last_scan": {
    "time": "2026-02-12T15:00:00Z",
    "commit": "def456abc789",
    "branch": "main",
    "mode": "incremental",
    "report_dir": "20260212-150000",
    "cpg_path": "workspace/project.bin",
    "files_scanned": 23,
    "issues_found": 3
  },
  "history": [
    {
      "time": "2026-02-12T10:00:00Z",
      "commit": "abc123def456",
      "mode": "full",
      "report_dir": "20260212-100000",
      "cpg_path": "workspace/project.bin",
      "files_scanned": 1523,
      "issues_found": 15
    }
  ]
}
```

### scan-info.json
位于每个时间戳目录下：

```json
{
  "scan_time": "2026-02-12T15:00:00Z",
  "scan_mode": "incremental",
  "joern_info": {
    "cpg_path": "workspace/project.bin",
    "cpg_build_time": "2026-02-12T14:55:00Z",
    "language": "JAVA"
  },
  "git_info": {
    "commit": "def456abc789",
    "branch": "main",
    "base_commit": "abc123def456"
  },
  "statistics": {
    "total_files": 1523,
    "scanned_files": 23,
    "changed_files": 23,
    "issues_found": 3,
    "critical": 1,
    "high": 2,
    "medium": 0
  },
  "changed_files": [
    "src/main/java/User.java",
    "src/main/java/Order.java"
  ],
  "joern_dataflow_paths": 15
}
```

## 时间戳规则

- 格式：`YYYYMMdd-HHmmss`（示例：`20260212-140500`）
- **重要**：整个检查过程使用同一个时间戳，第一次保存时生成，后续复用

## 文件命名

- `01-基础代码安全.md`
- `02-敏感信息泄露.md`
- `03-认证授权.md`
- `04-业务逻辑安全.md`
- `05-配置安全.md`
- `06-跨服务业务逻辑.md` (多服务项目)
- `07-跨服务配置安全.md` (多服务项目)
- `JOERN-DATAFLOW.md` (增量扫描时的数据流分析)
- `FINAL-REPORT.md` (汇总报告)

## 报告内容模板

```markdown
# {检查项名称}检查报告

**检查时间**: 2026-02-12 14:05:00
**项目路径**: {实际项目路径}
**检查语言**: Java / Go / 跨服务
**扫描方式**: Joern CPG污点分析

## Joern扫描配置
- CPG路径: workspace/project.bin
- 扫描规则: sql-injection, command-injection, xss, path-traversal
- 数据流深度: 5层

## 检查结果
### 1. SQL注入
- 🔴 **Critical**: 发现用户输入直接拼接SQL
  - **数据流路径**:
    ```
    Source: request.getParameter("userId") @ UserController.java:35
      ↓ 赋值
      String userId = request.getParameter("userId")
      ↓ 参数传递
      buildQuery(userId) @ UserController.java:40
      ↓ 字符串拼接
      "SELECT * FROM users WHERE id = " + userId @ QueryBuilder.java:15
      ↓ 参数传递
    Sink: stmt.executeQuery(sql) @ UserController.java:42
    ```
  - **风险**: SQL注入，攻击者可执行任意SQL
  - **建议**: 使用PreparedStatement参数化查询

### 2. 命令注入
- ✅ 未发现问题

## 统计
- 🔴 Critical: 1
- 🟡 Medium: 0
- 🟢 Low: 0
```

## 保存路径规则

**单语言项目**：
```
security-reports/{timestamp}/01-基础代码安全.md
```

**多语言项目**：
```
security-reports/{timestamp}/java/01-基础代码安全.md
security-reports/{timestamp}/go/01-基础代码安全.md
security-reports/{timestamp}/cross-service/06-跨服务业务逻辑.md
```

## 保存失败处理

提示"⚠️ 报告保存失败，但检查结果已输出"，继续检查流程
