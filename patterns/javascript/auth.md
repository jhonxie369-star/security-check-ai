# JavaScript/TypeScript 认证授权检查

## 1. Token 存储

```javascript
// ❌ 危险：LocalStorage 易受 XSS 攻击
localStorage.setItem('token', jwtToken);
localStorage.setItem('refreshToken', refreshToken);

// ✅ 安全：使用 HttpOnly Cookie（后端设置）
// 前端无需存储 token，浏览器自动携带
// 如必须前端存储，使用 sessionStorage + 短期有效
sessionStorage.setItem('token', shortLivedToken);
```

## 2. Token 传递

```javascript
// ❌ 危险：Token 在 URL 中
fetch(`/api/data?token=${token}`);

// ✅ 安全：使用 Authorization Header
fetch('/api/data', {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});
```

## 3. 前端权限校验

```javascript
// ❌ 危险：仅前端校验权限
if (user.role === 'admin') {
  showAdminPanel();
}

// ✅ 安全：前端隐藏 UI，后端强制校验
if (user.role === 'admin') {
  showAdminPanel();
}
// 后端必须再次验证
```

## 4. JWT 解析

```javascript
// ❌ 危险：信任前端解析的 JWT
const payload = JSON.parse(atob(token.split('.')[1]));
if (payload.role === 'admin') {
  // 执行管理操作
}

// ✅ 安全：前端可解析用于 UI，但后端必须验证
const payload = parseJWT(token);
showUserInfo(payload);
```

## 5. 会话超时

```javascript
// ❌ 危险：无会话超时机制
const token = localStorage.getItem('token');
if (token) {
  // 永久有效
}

// ✅ 安全：检查过期时间
const token = sessionStorage.getItem('token');
const expiry = sessionStorage.getItem('tokenExpiry');
if (token && Date.now() < expiry) {
  // 使用 token
} else {
  logout();
}
```

## 6. 登出处理

```javascript
// ❌ 危险：仅清除前端状态
function logout() {
  localStorage.removeItem('token');
  window.location.href = '/login';
}

// ✅ 安全：通知后端失效 token
async function logout() {
  await fetch('/api/logout', { method: 'POST' });
  sessionStorage.clear();
  window.location.href = '/login';
}
```

## 7. CSRF 防护

```javascript
// ❌ 危险：无 CSRF Token
fetch('/api/transfer', {
  method: 'POST',
  body: JSON.stringify({ to: account, amount })
});

// ✅ 安全：携带 CSRF Token
const csrfToken = document.querySelector('meta[name="csrf-token"]').content;
fetch('/api/transfer', {
  method: 'POST',
  headers: {
    'X-CSRF-Token': csrfToken
  },
  body: JSON.stringify({ to: account, amount })
});
```

## 检查清单

- [ ] Token 是否存储在 LocalStorage
- [ ] Token 是否通过 URL 传递
- [ ] 是否仅依赖前端权限校验
- [ ] JWT 是否仅在前端验证
- [ ] 是否有会话超时机制
- [ ] 登出是否通知后端
- [ ] 是否有 CSRF 防护
