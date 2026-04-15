# Python语言 - 认证与授权检查

## 认证机制检查

```bash
# 查找JWT使用
grep -rn "jwt\|JWT\|PyJWT" --include="*.py"

# 查找Django认证
grep -rn "@login_required\|LoginRequiredMixin" --include="*.py"

# 查找Flask认证
grep -rn "@login_required\|login_user" --include="*.py"

# 查找密码处理
grep -rn "make_password\|check_password\|bcrypt\|pbkdf2" --include="*.py"
```

## 授权检查（越权防护）

```bash
# 查找权限检查
grep -rn "has_perm\|check_permission\|user\.id.*==" --include="*.py"

# 查找未授权的端点
grep -rn "@app\.route\|@api_view" --include="*.py"
```

## 会话管理检查

```bash
# 查找会话配置
grep -rn "SESSION_COOKIE\|session\[" --include="*.py"
```

## 修复示例

### 水平越权防护（Django）

```python
# ❌ 错误：未检查资源所有权
def get_order(request, order_id):
    order = Order.objects.get(id=order_id)
    return JsonResponse(order.to_dict())

# ✅ 正确：验证资源所有权
from django.shortcuts import get_object_or_404

def get_order(request, order_id):
    order = get_object_or_404(Order, id=order_id, user_id=request.user.id)
    return JsonResponse(order.to_dict())

# ✅ 使用QuerySet过滤
def list_orders(request):
    # 自动过滤当前用户的订单
    orders = Order.objects.filter(user_id=request.user.id)
    return JsonResponse([o.to_dict() for o in orders], safe=False)
```

### 垂直越权防护（Django）

```python
# ❌ 错误：未检查角色权限
def delete_user(request, user_id):
    User.objects.filter(id=user_id).delete()
    return JsonResponse({'success': True})

# ✅ 正确：使用装饰器
from django.contrib.auth.decorators import user_passes_test, login_required

@login_required
@user_passes_test(lambda u: u.is_staff)
def delete_user(request, user_id):
    User.objects.filter(id=user_id).delete()
    return JsonResponse({'success': True})

# ✅ 使用权限系统
from django.contrib.auth.decorators import permission_required

@login_required
@permission_required('auth.delete_user', raise_exception=True)
def delete_user(request, user_id):
    User.objects.filter(id=user_id).delete()
    return JsonResponse({'success': True})
```

### Flask权限检查

```python
from functools import wraps
from flask import abort, g
from flask_login import current_user, login_required

# ❌ 错误：未检查权限
@app.route('/orders/<int:order_id>')
@login_required
def get_order(order_id):
    order = Order.query.get_or_404(order_id)
    return jsonify(order.to_dict())

# ✅ 正确：检查资源所有权
@app.route('/orders/<int:order_id>')
@login_required
def get_order(order_id):
    order = Order.query.filter_by(
        id=order_id, 
        user_id=current_user.id
    ).first_or_404()
    return jsonify(order.to_dict())

# ✅ 使用装饰器
def require_role(role):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            if not current_user.has_role(role):
                abort(403)
            return f(*args, **kwargs)
        return decorated_function
    return decorator

@app.route('/admin/users/<int:user_id>', methods=['DELETE'])
@login_required
@require_role('admin')
def delete_user(user_id):
    User.query.filter_by(id=user_id).delete()
    db.session.commit()
    return jsonify({'success': True})
```

### JWT认证

```python
import jwt
from datetime import datetime, timedelta
from flask import request, jsonify
from functools import wraps

SECRET_KEY = os.getenv('JWT_SECRET_KEY')

# ❌ 错误：弱密钥、无过期时间
def create_token(user_id):
    token = jwt.encode({'user_id': user_id}, 'weak-secret', algorithm='HS256')
    return token

# ✅ 正确：强密钥、设置过期时间
def create_token(user_id):
    payload = {
        'user_id': user_id,
        'exp': datetime.utcnow() + timedelta(hours=1),
        'iat': datetime.utcnow()
    }
    token = jwt.encode(payload, SECRET_KEY, algorithm='HS256')
    return token

def verify_token(token):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
        return payload['user_id']
    except jwt.ExpiredSignatureError:
        return None
    except jwt.InvalidTokenError:
        return None

def token_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        token = request.headers.get('Authorization')
        
        if not token:
            return jsonify({'error': 'Token is missing'}), 401
        
        if token.startswith('Bearer '):
            token = token[7:]
        
        user_id = verify_token(token)
        if not user_id:
            return jsonify({'error': 'Invalid token'}), 401
        
        g.user_id = user_id
        return f(*args, **kwargs)
    
    return decorated

@app.route('/api/protected')
@token_required
def protected():
    return jsonify({'user_id': g.user_id})
```

### 密码安全

```python
# ❌ 错误：明文存储密码
def create_user(username, password):
    user = User(username=username, password=password)
    db.session.add(user)

# ❌ 错误：使用MD5/SHA1
import hashlib
password_hash = hashlib.md5(password.encode()).hexdigest()

# ✅ 正确：使用bcrypt（Django）
from django.contrib.auth.hashers import make_password, check_password

def create_user(username, password):
    user = User(username=username, password=make_password(password))
    user.save()

def verify_password(user, password):
    return check_password(password, user.password)

# ✅ 正确：使用bcrypt（Flask）
from werkzeug.security import generate_password_hash, check_password_hash

def create_user(username, password):
    user = User(
        username=username,
        password=generate_password_hash(password, method='pbkdf2:sha256')
    )
    db.session.add(user)

def verify_password(user, password):
    return check_password_hash(user.password, password)
```

### 会话管理

```python
# Django会话配置
# settings.py
SESSION_COOKIE_SECURE = True  # 仅HTTPS传输
SESSION_COOKIE_HTTPONLY = True  # 防止XSS
SESSION_COOKIE_SAMESITE = 'Strict'  # 防止CSRF
SESSION_COOKIE_AGE = 3600  # 1小时过期
SESSION_SAVE_EVERY_REQUEST = True  # 每次请求刷新

# Flask会话配置
app.config['SESSION_COOKIE_SECURE'] = True
app.config['SESSION_COOKIE_HTTPONLY'] = True
app.config['SESSION_COOKIE_SAMESITE'] = 'Strict'
app.config['PERMANENT_SESSION_LIFETIME'] = timedelta(hours=1)

@app.before_request
def make_session_permanent():
    session.permanent = True
```

### API密钥认证

```python
# ❌ 错误：API密钥在URL中
@app.route('/api/data')
def get_data():
    api_key = request.args.get('api_key')
    if api_key != 'secret-key':
        abort(401)

# ✅ 正确：API密钥在Header中
def require_api_key(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        api_key = request.headers.get('X-API-Key')
        
        if not api_key:
            return jsonify({'error': 'API key is missing'}), 401
        
        # 从数据库验证API密钥
        key = APIKey.query.filter_by(key=api_key, is_active=True).first()
        if not key:
            return jsonify({'error': 'Invalid API key'}), 401
        
        g.api_key = key
        return f(*args, **kwargs)
    
    return decorated

@app.route('/api/data')
@require_api_key
def get_data():
    return jsonify({'data': 'protected data'})
```

### OAuth2集成

```python
from authlib.integrations.flask_client import OAuth

oauth = OAuth(app)

# 配置OAuth提供商
oauth.register(
    name='google',
    client_id=os.getenv('GOOGLE_CLIENT_ID'),
    client_secret=os.getenv('GOOGLE_CLIENT_SECRET'),
    server_metadata_url='https://accounts.google.com/.well-known/openid-configuration',
    client_kwargs={'scope': 'openid email profile'}
)

@app.route('/login/google')
def login_google():
    redirect_uri = url_for('authorize_google', _external=True)
    return oauth.google.authorize_redirect(redirect_uri)

@app.route('/authorize/google')
def authorize_google():
    token = oauth.google.authorize_access_token()
    user_info = oauth.google.parse_id_token(token)
    
    # 创建或更新用户
    user = User.query.filter_by(email=user_info['email']).first()
    if not user:
        user = User(email=user_info['email'], name=user_info['name'])
        db.session.add(user)
        db.session.commit()
    
    login_user(user)
    return redirect('/')
```

### 多因素认证（MFA）

```python
import pyotp
import qrcode
from io import BytesIO

def setup_mfa(user):
    # 生成密钥
    secret = pyotp.random_base32()
    user.mfa_secret = secret
    db.session.commit()
    
    # 生成二维码
    totp = pyotp.TOTP(secret)
    uri = totp.provisioning_uri(user.email, issuer_name='MyApp')
    
    qr = qrcode.make(uri)
    buffer = BytesIO()
    qr.save(buffer, format='PNG')
    buffer.seek(0)
    
    return buffer

def verify_mfa(user, code):
    totp = pyotp.TOTP(user.mfa_secret)
    return totp.verify(code, valid_window=1)

@app.route('/login', methods=['POST'])
def login():
    username = request.json.get('username')
    password = request.json.get('password')
    mfa_code = request.json.get('mfa_code')
    
    user = User.query.filter_by(username=username).first()
    
    if not user or not verify_password(user, password):
        return jsonify({'error': 'Invalid credentials'}), 401
    
    # 如果启用了MFA
    if user.mfa_enabled:
        if not mfa_code or not verify_mfa(user, mfa_code):
            return jsonify({'error': 'Invalid MFA code'}), 401
    
    login_user(user)
    return jsonify({'success': True})
```

### 检查清单

- [ ] 所有API端点有认证保护
- [ ] 资源访问检查所有权
- [ ] 管理功能检查角色权限
- [ ] 密码使用bcrypt/pbkdf2加密
- [ ] JWT设置过期时间
- [ ] 会话Cookie设置Secure和HttpOnly
- [ ] API密钥在Header中传递
- [ ] 敏感操作考虑MFA
- [ ] 登录失败有限流保护
