# Java语言 - 基础代码安全检查指导

## ⚠️ 核心原则

**本文件提供检查思路和常见模式，不是固定规则。**

**工作流程**：
1. 识别项目特点（框架、输入点、危险操作）
2. 使用grep搜索关键代码位置
3. 读取完整上下文（前后50-100行）
4. AI深度分析，结合源码判断真实风险

---

## 1. SQL注入检查指导

### Java 的 SQL 执行特点
- **JDBC**：Statement 拼接危险，PreparedStatement 参数化安全
- **MyBatis**：`#{}` 参数化安全，`${}` 拼接危险
- **JPA/Hibernate**：createQuery、createNativeQuery 可能有注入
- **Spring JdbcTemplate**：query() 方法需要检查参数

### 搜索关键代码

```bash
# 搜索SQL执行点
grep -rn "executeQuery\|executeUpdate\|execute" --include="*.java" -A 10 -B 5

# 搜索JPA查询
grep -rn "createQuery\|createNativeQuery" --include="*.java" -A 10 -B 5

# 搜索JdbcTemplate
grep -rn "JdbcTemplate.*query\|JdbcTemplate.*update" --include="*.java" -A 10 -B 5

# 搜索MyBatis的${}（危险）
grep -rn '\${' --include="*.xml" -A 3 -B 3
```

### AI分析要点

**第1步：识别参数来源**
```java
// 用户输入（高风险）
String id = request.getParameter("id");
String name = request.getHeader("X-User-Name");
String data = request.getBody();

// 配置文件（低风险）
String table = config.getProperty("table.name");

// 常量（无风险）
String status = "ACTIVE";
```

**第2步：检查是否使用安全API**
```java
// ✅ 安全：PreparedStatement
String sql = "SELECT * FROM users WHERE id = ?";
PreparedStatement stmt = conn.prepareStatement(sql);
stmt.setInt(1, userId);

// ✅ 安全：JPA参数化
User user = entityManager.createQuery("FROM User WHERE id = :id", User.class)
    .setParameter("id", userId)
    .getSingleResult();

// ✅ 安全：MyBatis #{} 
<select id="getUser">
    SELECT * FROM users WHERE id = #{id}
</select>

// ❌ 危险：字符串拼接
String sql = "SELECT * FROM users WHERE id = " + userId;
stmt.executeQuery(sql);

// ❌ 危险：MyBatis ${}
<select id="getUser">
    SELECT * FROM users WHERE id = ${id}
</select>
```

**第3步：检查输入验证**
```java
// ✅ 有验证
if (!userId.matches("\\d+")) {
    throw new IllegalArgumentException();
}

// ✅ 使用白名单
if (!Arrays.asList("name", "age", "email").contains(sortField)) {
    throw new IllegalArgumentException();
}
```

**第4步：追踪数据流**

读取包含SQL执行的完整函数（前后50-100行），分析：
- SQL语句中的参数从哪里来
- 参数是否经过验证函数
- 是否使用了PreparedStatement或参数化查询
- 是否有ORM框架的保护

### 风险判断

| 场景 | 风险等级 |
|------|---------|
| 用户输入 + 字符串拼接 + 无验证 | 严重 |
| 用户输入 + 字符串拼接 + 有验证 | 高危 |
| 用户输入 + PreparedStatement | 安全 |
| 配置文件 + 字符串拼接 | 低危 |
| 常量拼接 | 安全 |

### 常见误报

- PreparedStatement 虽然调用 execute，但参数化了，**不是漏洞**
- 动态表名/字段名用白名单验证，**不是漏洞**
- SQL 常量拼接（如 `"SELECT * FROM " + TABLE_CONSTANT`），**不是漏洞**

### 修复示例

```java
// ❌ 危险：字符串拼接
String sql = "SELECT * FROM users WHERE username = '" + username + "'";
stmt.executeQuery(sql);

// ✅ 安全：PreparedStatement
String sql = "SELECT * FROM users WHERE username = ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setString(1, username);
pstmt.executeQuery();

// ❌ 危险：动态字段名
String sql = "SELECT * FROM users WHERE " + fieldName + " = ?";

// ✅ 安全：字段名白名单
Set<String> allowedFields = Set.of("username", "email", "status");
if (!allowedFields.contains(fieldName)) {
    throw new IllegalArgumentException("Invalid field");
}
String sql = "SELECT * FROM users WHERE " + fieldName + " = ?";
```

---

## 2. 命令注入检查指导

### Java 的命令执行特点
- **Runtime.exec()**：字符串拼接危险，数组参数安全
- **ProcessBuilder**：参数数组安全
- **脚本引擎**：ScriptEngine.eval() 危险

### 搜索关键代码

```bash
# 搜索命令执行
grep -rn "Runtime\.exec\|ProcessBuilder" --include="*.java" -A 10 -B 5

# 搜索脚本引擎
grep -rn "ScriptEngine.*eval" --include="*.java" -A 10 -B 5
```

### AI分析要点

**识别危险模式**：
```java
// ❌ 危险：字符串拼接
String cmd = "ping -c 4 " + host;
Runtime.getRuntime().exe);

// ✅ 安全：参数数组 + 验证
if (!host.matches("^[a-zA-Z0-9.-]+$")) {
    throw new IllegalArgumentException("Invalid host");
}
ProcessBuilder pb = new ProcessBuilder("ping", "-c", "4", host);
pb.start();
```

### 检查要点

- 是否使用参数数组（安全）
- 是否有输入验证（白名单）
- 理解命令用途（是否必须执行）

### 修复示例

```java
// ❌ 危险：字符串拼接
Runtime.getRuntime().exec("ping -c 4 " + host);

// ✅ 安全：参数数组 + 验证
if (!host.matches("^[a-zA-Z0-9.-]+$")) {
    throw new IllegalArgumentException("Invalid host");
}
ProcessBuilder pb = new ProcessBuilder("ping", "-c", "4", host);
pb.start();
```

---

## 3. 路径穿越检查指导

### Java 的文件操作特点
- **File**：new File(path) 直接使用用户输入危险
- **Path/Paths**：resolve() 可能穿越
- **FileInputStream/FileOutputStream**：需要验证路径

### 搜索关键代码

```bash
# 搜索文件操作
grep -rn "new File\|Files\.read\|FileInputStream\|FileOutputStream" --include="*.java" -A 10 -B 5

# 搜索Path操作
grep -rn "Paths\.get\|Path\.resolve" --include="*.java" -A 10 -B 5
```

### AI分析要点

**检查路径验证**：
```java
// ❌ 危险：直接使用用户输入
File file = new File(filename);

// ✅ 安全：路径验证
Path baseDir = Paths.get("/var/www/uploads").toAbsolutePath().normalize();
Path filePath = basolve(filename).normalize();

if (!filePath.startsWith(baseDir)) {
    throw new SecurityException("Invalid path");
}
```

### 检查要点

- 检查是否有路径规范化（normalize）
- 检查是否验证在允许目录内（startsWith）
- 检查是否过滤 `..`

### 修复示例

```java
// ❌ 危险：直接使用用户输入
File file = new File(filename);

// ✅ 安全：路径验证
Path baseDir = Paths.get("/var/www/uploads").toAbsolutePath().normalize();
Path filePath = baseDir.resolve(filename).normalize();

if (!filePath.startsWith(baseDir)) {
    throw new SecurityException("Invalid path");
}
```

---

## 4. 文件上传检查指导

### Java 的文件上传特点
- **MultipartFile**：Spring 的文件上传
- **Part**：Servlet 的文件上传
- **Commons FileUpload**：Apache 的文件上传

### 搜索关键代码

```bash
# 搜索文件上传
grep -rn "MultipartFile\|@RequestParam.*file" --include="*.java" -A 20 -B 5

# 搜索文件保存
grep -rn "transferTo\|write.*File" --include="*.java" -A 10 -B 5
```

### 检查要点

- 是否限制文件大小
- 是否验证 Content-Type（白名单）
- 是否验证扩展名（白名单）
- 是否重命名文件（防止覆盖）
- 是否限制上传目录

### 修复示例

```java
// ✅ 完整的文件上传验证
@PostMapping("/upload")
public ResponseEntity<String> upload(@RequestParam("file") MultipartFile file) {
    // 1. 大小验证
    if (file.getSize() > 10 * 1024 * 1024) {
        return ResponseEntity.badRequest().body("文件过大");
    }
    
    // 2. 类型验证
    List<String> allowedTypes = Arrays.asList("image/jpeg", "image/png");
    if (!allowedTypes.contains(file.getContentType())) {
        return ResponseEntity.badRequest().body("不支持的类型");
    }
    
    // 3. 扩展名验证
    String ext = file.getOriginalFilename().substring(
        file.getOriginalFilename().lastIndexOf(".")
    );
    if (!Arrays.asList(".jpg", ".png").contains(ext.toLowerCase())) {
        return ResponseEntity.badRequest().body("不支持的扩展名");
    }
    
    // 4. 重命名
    String newFilename = UUID.randomUUID().toString() + ext;
    Path uploadPath = Paths.get("/var/www/uploads", newFilename);
    
    file.transferTo(uploadPath.toFile());
    return ResponseEntity.ok(newFilename);
}
```

---

## 5. XSS 检查指导

### Java 的输出特点
- **JSP**：直接输出到页面
- **Servlet**：response.getWriter().write()
- **模板引擎**：Thymeleaf、FreeMarker、Velocity
- **JSON 响应**：自动转义

### 搜索关键代码

```bash
# 搜索输出点
grep -rn "response\.getWriter\|out\.print\|out\.write" --include="*.java" -A 10 -B 5

# 搜索JSP输出
grep -rn "<%=.*%>" --include="*.jsp" -A 3 -B 3

# 搜索Thymeleaf不安全输出
grep -rn "th:utext" --include="*.html" -A 3 -B 3
```

### AI分析转义**：
```java
// ❌ 危险：直接输出用户输入
String name = req.getParameter("name");
resp.getWriter().write("<h1>Hello " + name + "</h1>");

// ✅ 安全：使用 HTML 转义
import org.apache.commons.text.StringEscapeUtils;

String name = req.getParameter("name");
String escaped = StringEscapeUtils.escapeHtml4(name);
resp.getWriter().write("<h1>Hello " + escaped + "</h1>");

// ✅ 更好：使用模板引擎
// JSP with JSTL
<c:out value="${param.name}" />

// Thymeleaf
<div th:text="${name}">Default Name</div>  // 自动转义

// ❌ 危险：Thymeleaf th:utext 不转义
<div th:utext="${userInput}">...</div>  // 危险！
```

### 检查要点

- 检查是否使用模板自动转义
- 检查是否有手动转义（StringEscapeUtils.escapeHtml4）
- 检查 JSON 输出（检查 Content-Type 设置

---

## 6. SSRF检查指导

### Java 的 HTTP 请求特点
- **URL.openConnection()**：直接请求
- **HttpClient**：Apache/Java 11+ 的 HTTP 客户端
- **RestTemplate**：Spring 的 HTTP 客户端
- **OkHttp**：第三方 HTTP 库

### 搜索关键代码

```bash
# 搜索HTTP请求
grep -rn "URL.*openConnection\|HttpClient.*execute\|RestTemplate" --include="*.java" -A 10 -B 5

# 搜索URL构造
grep -rn "new URL" --include="*.java" -A 5 -B 5
```

### AI分析要点

**检查URL来源和验证**：
```java
// ❌ 危险：直接请求用户输入的 URL
URL url = new URL(urlStr);
url.openStream();

// ✅ 安全：URL 验证
public void safeHTTPReqng urlStr) throws Exception {
    URL url = new URL(urlStr);

    // 只允许 http/https
    if (!url.getProtocol().matches("^https?$")) {
        throw new SecurityException("Invalid protocol");
    }

    // 解析 IP 并检查是否为内网
    InetAddress[] ips = InetAddress.getAllByName(url.getHost());
    for (InetAddress addr : ips) {
        if (addr.isLoopbackAddress() || addr.isSiteLocalAddress() || addr.isLinkLocalAddress()) {
            throw new SecurityException("Private IP not allowed");
        }
    }

    url.openStream();
}
```

### 检查要点

- 检查是否限制协议（只允许 http/https）
- 检查是否禁止内网 IP（10.x, 172.1168.x, 127.x, 169.254.x）
- 检查是否有 URL 白名单
- 检查是否有 DNS rebinding 防护

---

## 7. XXE检查指导

### Java 的 XML 解析特点
- **DocumentBuilder**：DOM 解析
- **SAXParser**：SAX 解析
- **XMLReader**：底层解析
- **Unmarshaller**：JAXB 反序列化

### 搜索关键代码

```bash
# 搜索XML解析
grep -rn "DocumentBuilder\|SAXParser\|XMLReader" --include="*.java" -A 10 -B 5

# 检查是否禁用外部实体
grep -rn "setFeature.*disallow-doctype-decl\|external-general-entities" --include="*.java" -A 3 -B 3
```

### 修复示例

```java
// ✅ 安全：禁用外部实体
DocumentBuilderFactory factory = DocutBuilderFactory.newInstance();
factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
DocumentBuilder builder = factory.newDocumentBuilder();
```

---

## 8. 不安全反序列化检查指导

### Java 的反序列化特点
- **ObjectInputStream.readObject()**：Java 原生反序列化
- **XStream**：XML 反序列化
- **Jackson/Gson**：JSON 反序列化（部分场景）

### 搜索关键代码

```bash
# 搜索反序列化
grep -rn "ObjectInputStream.*readObject\|XStream.*fromXML" --include="*.java" -A 10 -B 5
```

### 修复示例

```java
// ✅ 安全：类白名单
class SafeObjectInputStream extends ObjectInputStream {
    private static final Set<String> ALLOWED_CLASSES = Set.of(
        "com.example.SafeClass1",
        "com.example.SafeClass2"
    );
    
    @Override
    protected Class<?> resolveClass(ObjectStreamClass desc) 
            throws IOException, ClassNotFoundException {
        if (!ALLOWED_CLASSES.contains(desc.getName())) {
            throw new InvalidClassException("Unauthorized deserialization");
        }
        return super.resolveClass(desc);
    }
}
```
\n
## 检查优先级

### Critical（立即修复）
1. SQL注入
2. 命令注入
3. 路径穿越
4. 不安全反序列化
5. XXE

### High（一周内修复）
6. XSS
7. SSRF
8. 文件上传（无验证）

### Medium（两周内修复）
9. 日志注入
10. 资源泄露

---

## 检查清单

- [ ] SQL执行是否使用PreparedStatement/参数化查询
- [ ] 命令执行是否使用数组形式
- [ ] 文件操作是否验证路径
- [ ] 文件上传是否有完整验证
- [ ] 输出是否有HTML转义
- [ ] HTTP请求是否验证URL
- [ ] XML解析是否禁用外部实体
- [ ] 反序列化是否有类白名单
