# Python语言 - 配置安全检查

## 调试模式检查

```bash
# 查找调试模式
grep -rn "DEBUG.*=.*True\|debug.*=.*True" --include="*.py"
```

## 敏感端点检查

```bash
# 查找Django admin
grep -rn "admin\.site\|/admin" --include="*.py"

# 查找调试工具
grep -rn "django_debug_toolbar\|werkzeug\.debug" --include="*.py"
```

## CORS配置检查

```bash
# 查找CORS配置
grep -rn "CORS\|Access-Control-Allow-Origin" --include="*.py"
```

## 错误信息检查

```bash
# 查找详细错误信息
grep -rn "traceback\|exc_info=True" --include="*.py"
```

## 修复示例

### Django生产环境配置

```python
# ❌ 错误：生产环境开启DEBUG
# settings.py
DEBUG = True
ALLOWED_HOSTS = ['*']
SECRET_KEY = 'hardcoded-key'

# ✅ 正确：生产环境配置
import os

DEBUG = os.getenv('DEBUG', 'False') == 'True'
ALLOWED_HOSTS = os.getenv('ALLOWED_HOSTS', '').split(',')
SECRET_KEY = os.getenv('SECRET_KEY')

# 安全响应头
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# 会话安全
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = 'Strict'
CSRF_COOKIE_HTTPONLY = True
CSRF_COOKIE_SAMESITE = 'Strict'

# 禁用调试工具
if not DEBUG:
    INSTALLED_APPS = [app for app in INSTALLED_APPS if 'debug_toolbar' not in app]
```

### Flask生产环境配置

```python
# ❌ 错误：开启调试
app.run(debug=True)

# ✅ 正确：生产环境配置
import os

app.config['DEBUG'] = False
app.config['SECRET_KEY'] = os.getenv('SECRET_KEY')
app.config['SESSION_COOKIE_SECURE'] = True
app.config['SESSION_COOKIE_HTTPONLY'] = True
app.config['SESSION_COOKIE_SAMESITE'] = 'Strict'

# 安全响应头
from flask_talisman import Talisman
Talisman(app, 
    force_https=True,
    strict_transport_security=True,
    content_security_policy={
        'default-src': "'self'",
        'script-src': "'self'",
        'style-src': "'self'"
    }
)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)
```

### CORS配置

```python
# ❌ 错误：允许所有来源
from flask_cors import CORS
CORS(app, origins='*')

# ✅ 正确：指定允许的来源
CORS(app, 
    origins=['https://example.com', 'https://www.example.com'],
    methods=['GET', 'POST'],
    allow_headers=['Content-Type', 'Authorization'],
    supports_credentials=True
)

# Django CORS
# settings.py
CORS_ALLOWED_ORIGINS = [
    'https://example.com',
    'https://www.example.com',
]
CORS_ALLOW_CREDENTIALS = True
CORS_ALLOWED_METHODS = ['GET', 'POST', 'PUT', 'DELETE']
```

### 错误处理

```python
# ❌ 错误：返回详细错误信息
@app.errorhandler(Exception)
def handle_error(e):
    return jsonify({'error': str(e), 'traceback': traceback.format_exc()}), 500

# ✅ 正确：生产环境隐藏详细信息
import logging

@app.errorhandler(Exception)
def handle_error(e):
    # 记录详细错误到日志
    logging.exception('Unhandled exception')
    
    # 返回通用错误信息
    if app.config['DEBUG']:
        return jsonify({'error': str(e)}), 500
    else:
        return jsonify({'error': 'Internal server error'}), 500

# Django错误处理
# settings.py
if not DEBUG:
    ADMINS = [('Admin', 'admin@example.com')]
    LOGGING = {
        'version': 1,
        'handlers': {
            'file': {
                'level': 'ERROR',
                'class': 'logging.FileHandler',
                'filename': '/var/log/django/error.log',
            },
        },
        'loggers': {
            'django': {
                'handlers': ['file'],
                'level': 'ERROR',
            },
        },
    }
```

### 日志配置

```python
# ❌ 错误：日志级别过低
import logging
logging.basicConfig(level=logging.DEBUG)

# ✅ 正确：根据环境设置日志级别
import os
import logging

log_level = os.getenv('LOG_LEVEL', 'INFO')
logging.basicConfig(
    level=getattr(logging, log_level),
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('/var/log/app/app.log'),
        logging.StreamHandler()
    ]
)

# 生产环境不输出DEBUG日志
if not DEBUG:
    logging.getLogger().setLevel(logging.INFO)
```

### 文件上传配置

```python
# Flask文件上传
app.config['MAX_CONTENT_LENGTH'] = 16 * 1024 * 1024  # 16MB
app.config['UPLOAD_FOLDER'] = '/var/www/uploads'
app.config['ALLOWED_EXTENSIONS'] = {'png', 'jpg', 'jpeg', 'gif'}

def allowed_file(filename):
    return '.' in filename and \
           filename.rsplit('.', 1)[1].lower() in app.config['ALLOWED_EXTENSIONS']

# Django文件上传
# settings.py
FILE_UPLOAD_MAX_MEMORY_SIZE = 16 * 1024 * 1024  # 16MB
MEDIA_ROOT = '/var/www/media'
MEDIA_URL = '/media/'
```

### 数据库连接池

```python
# ❌ 错误：无连接池限制
from sqlalchemy import create_engine
engine = create_engine('postgresql://user:pass@localhost/db')

# ✅ 正确：配置连接池
engine = create_engine(
    'postgresql://user:pass@localhost/db',
    pool_size=10,
    max_overflow=20,
    pool_timeout=30,
    pool_recycle=3600
)

# Django数据库配置
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'CONN_MAX_AGE': 600,  # 连接持久化
        'OPTIONS': {
            'connect_timeout': 10,
        }
    }
}
```

### 中间件安全

```python
# Flask安全中间件
from flask import Flask, request, abort

@app.before_request
def security_headers():
    # 限制请求大小
    if request.content_length and request.content_length > 16 * 1024 * 1024:
        abort(413)

@app.after_request
def add_security_headers(response):
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-Frame-Options'] = 'DENY'
    response.headers['X-XSS-Protection'] = '1; mode=block'
    response.headers['Strict-Transport-Security'] = 'max-age=31536000; includeSubDomains'
    return response

# Django中间件
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    # ...
]
```

### 环境分离

```python
# ✅ 使用不同的配置文件
# config/base.py
class BaseConfig:
    SECRET_KEY = os.getenv('SECRET_KEY')
    SQLALCHEMY_TRACK_MODIFICATIONS = False

# config/development.py
class DevelopmentConfig(BaseConfig):
    DEBUG = True
    SQLALCHEMY_ECHO = True

# config/production.py
class ProductionConfig(BaseConfig):
    DEBUG = False
    SQLALCHEMY_ECHO = False
    SESSION_COOKIE_SECURE = True

# app.py
env = os.getenv('FLASK_ENV', 'production')
if env == 'development':
    app.config.from_object('config.development.DevelopmentConfig')
else:
    app.config.from_object('config.production.ProductionConfig')
```

### 检查清单

- [ ] 生产环境关闭DEBUG
- [ ] ALLOWED_HOSTS正确配置
- [ ] 安全响应头已设置
- [ ] CORS不允许所有来源
- [ ] 错误信息不泄露详细信息
- [ ] 日志级别适当（INFO/WARNING）
- [ ] 会话Cookie设置Secure和HttpOnly
- [ ] 文件上传有大小和类型限制
- [ ] 数据库连接池配置合理
- [ ] 禁用调试工具（debug_toolbar等）
