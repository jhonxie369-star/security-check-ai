# JavaScript/TypeScript 配置安全检查

## 1. 环境变量泄露

```javascript
// ❌ 危险：前端暴露敏感配置
const config = {
  apiKey: 'sk-1234567890',
  dbPassword: 'password123',
  secretKey: 'my-secret'
};

// ✅ 安全：仅暴露公开配置
const config = {
  apiEndpoint: process.env.REACT_APP_API_URL,
  // 敏感配置在后端
};
```

## 2. .env 文件

```bash
# ❌ 危险：提交到 Git
.env

# ✅ 安全：添加到 .gitignore
# .gitignore
.env
.env.local
.env.*.local
```

## 3. Webpack/Vite 配置

```javascript
// ❌ 危险：暴露所有环境变量
new webpack.DefinePlugin({
  'process.env': JSON.stringify(process.env)
});

// ✅ 安全：仅暴露必要的公开变量
new webpack.DefinePlugin({
  'process.env.API_URL': JSON.stringify(process.env.API_URL)
});
```

## 4. CORS 配置

```javascript
// ❌ 危险：允许所有来源
res.setHeader('Access-Control-Allow-Origin', '*');

// ✅ 安全：限制来源
const allowedOrigins = ['https://myapp.com'];
if (allowedOrigins.includes(req.headers.origin)) {
  res.setHeader('Access-Control-Allow-Origin', req.headers.origin);
}
```

## 5. Content Security Policy

```html
<!-- ❌ 危险：无 CSP -->
<head>
  <title>My App</title>
</head>

<!-- ✅ 安全：设置 CSP -->
<head>
  <meta http-equiv="Content-Security-Policy" 
        content="default-src 'self'; script-src 'self' https://trusted-cdn.com">
</head>
```

## 检查清单

- [ ] 是否有敏感信息在前端代码中
- [ ] .env 文件是否在 .gitignore 中
- [ ] 是否只暴露必要的环境变量
- [ ] CORS 是否限制来源
- [ ] 是否设置 CSP
