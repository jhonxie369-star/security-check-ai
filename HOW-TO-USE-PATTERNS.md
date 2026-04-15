# 如何使用Patterns文件（AI版）

## 🎯 核心理念

**patterns文件中的内容保持与原版一致，但AI在使用时需要将Joern查询转换为grep搜索 + AI分析。**

---

## 📖 使用方法

### 原版patterns文件的结构

```markdown
## SQL注入检查指导

### Joern 查询思路

**第一步：找用户输入（Source）**
```scala
cpg.call.name(".*getParameter.*").l
```

**第二步：找 SQL 执行（Sink）**
```scala
cpg.call.name(".*executeQuery.*").l
```

**第三步：污点追踪**
```scala
sinks.reachableByFlows(sources).l
```
```

### AI如何转换使用

#### 步骤1：将Joern查询转换为grep搜索

**Joern查询**：
```scala
cpg.call.name(".*getParameter.*").l
```

**转换为grep**：
```bash
grep -rn "getParameter" --include="*.java" -A 10 -B 5
```

**转换规则**：
- `cpg.call.name(".*XXX.*")` → `grep -rn "XXX"`
- `.l` 表示列表，grep自然返回列表
- `-A 10 -B 5` 获取上下文（相当于Joern的调用链）

#### 步骤2：将污点追踪转换为AI分析

**Joern污点追踪**：
```scala
val sources = cpg.call.name(".*getParameter.*").l
val sinks = cpg.call.name(".*executeQuery.*").l
sinks.reachableByFlows(sources).l
```

**AI分析流程**：
1. grep搜索SQL执行点（sinks）
2. 对每个匹配，读取完整函数代码（前后50-100行）
3. AI分析：
   - 参数来源（是否来自用户输入）
   - 是否有验证函数
   - 是否使用安全API
   - 数据流路径

#### 步骤3：使用patterns中的"检查要点"

patterns文件中的检查要点、修复示例、风险判断表格**完全适用**，直接使用。

---

## 🔍 具体示例

### 示例1：SQL注入检查

**patterns文件内容**（保持不变）：
```markdown
### Joern 查询思路

**第一步：找用户输入（Source）**
```scala
cpg.call.name(".*getParameter.*|.*getQueryParam.*").l
```

**第二步：找 SQL 执行（Sink）**
```scala
cpg.call.name(".*executeQuery.*|.*executeUpdate.*").l
```
```

**AI实际执行**：

```bash
# 步骤1：搜索SQL执行点
grep -rn "executeQuery\|executeUpdate" --include="*.java" -A 10 -B 5

# 步骤2：对每个匹配，读取完整上下文
# （假设找到 UserController.java:42）
```

然后AI读取该文件的第1-100行（或包含完整函数的范围），分析：
- 第42行的SQL语句参数来源
- 是否使用PreparedStatement
- 是否有输入验证
- 结合patterns中的"检查要点"判断风险

### 示例2：业务逻辑检查

**patterns文件内容**（保持不变）：
```markdown
### Joern 查询思路

```scala
// 查找金额字段
val amountSources = cpg.call.name(".*POST.*")
  .where(_.argument.code(".*amount.*")).l

// 查找订单创建
val orderSinks = cpg.call.name(".*create.*")
  .where(_.code(".*Order.*")).l

// 追踪数据流
orderSinks.reachableBy(amountSources).l
```
```

**AI实际执行**：

```bash
# 步骤1：搜索订单创建相关代码
grep -rn "createOrder\|create.*Order" --include="*.java" -A 20 -B 10

# 步骤2：在匹配的代码中搜索amount相关
grep -rn "amount\|price\|total" UserController.java
```

然后AI读取完整的订单创建函数，分析：
- amount参数来源（前端 vs 后端计算）
- 是否有价格验证
- 是否可能负数
- 结合patterns中的"业务风险"判断

---

## 📋 转换对照表

### Joern查询 → grep搜索

| Joern查询 | grep等价命令 |
|----------|-------------|
| `cpg.call.name(".*XXX.*").l` | `grep -rn "XXX" --include="*.java"` |
| `cpg.method.name(".*XXX.*").l` | `grep -rn "XXX" --include="*.java"` |
| `cpg.identifier.name("XXX").l` | `grep -rn "\\bXXX\\b" --include="*.java"` |
| `cpg.literal.code(".*XXX.*").l` | `grep -rn "\".*XXX.*\"" --include="*.java"` |
| `.where(_.code(".*YYY.*"))` | `grep "YYY"` (在前一个结果中过滤) |
| `.whereNot(_.file.name(".*test.*"))` | `grep -v "test"` (排除) |

### Joern污点追踪 → AI分析

| Joern操作 | AI等价操作 |
|----------|-----------|
| `sinks.reachableByFlows(sources)` | 读取sink所在函数完整代码，AI分析参数来源 |
| `flow.elements` | 读取函数上下文（前后50-100行） |
| `elem.location.lineNumber` | grep输出的行号 |
| `elem.code` | grep输出的代码片段 |

---

## ✅ 使用清单

AI在使用patterns文件时：

- [ ] 读取patterns文件中的"检查思路"
- [ ] 将Joern查询转换为grep搜索
- [ ] 执行grep搜索，获取匹配位置
- [ ] 读取匹配位置的完整上下文（前后50-100行）
- [ ] 使用patterns中的"检查要点"进行分析
- [ ] 使用patterns中的"风险判断表"评估风险
- [ ] 参考patterns中的"修复示例"提供建议
- [ ] 使用patterns中的"检查清单"确保完整性

---

## 🎯 总结

**patterns文件内容不变**，AI在使用时：
1. **理解意图**：理解Joern查询想要找什么
2. **转换方法**：用grep + 读取上下文替代Joern
3. **保留精华**：检查要点、修复示例、风险判断完全适用
4. **AI优势**：更好的上下文理解和业务逻辑分析

这样既保持了patterns文件的完整性和一致性，又充分发挥了AI的理解能力。
