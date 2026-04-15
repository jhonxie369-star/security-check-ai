# Python语言 - 敏感信息检查

## 配置文件硬编码

```bash
# 查找配置文件中的敏感信息
grep -rn "password|secret|token|api_key" --include="*.py" --include="*.ini" --include="*.env" --include="*.cfg"
```

## 代码中硬编码

```bash
# 查找硬编码密钥
grep -rn "PASSWORD.*=.*['\"]|SECRET.*=.*['\"]|TOKEN.*=.*['\"]|API_KEY.*=.*['\"]" --include="*.py"
```

## 环境变量使用（应该存在）

```bash
# 查找环境变量读取
grep -rn "os\.getenv\|os\.environ\|config\[" --include="*.py"
```

## 判断逻辑

**❌ 高风险**：
- 找到硬编码 + 没找到环境变量读取

**✅ 安全**：
- 使用 `os.getenv()` 或 `os.environ[]`

## 修复示例

### Django配置

```python
# ❌ 错误：硬编码密钥
# settings.py
SECRET_KEY = 'django-insecure-hardcoded-key-123'
DEBUG = True
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'admin',
        'PASSWORD': 'MyPassword123',  # 硬编码
        'HOST': 'localhost',
    }
}

# AWS配置硬编码
AWS_ACCESS_KEY_ID = 'AKIAIOSFODNN7EXAMPLE'
AWS_SECRET_ACCESS_KEY = 'wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY'

# ✅ 正确：使用环境变量
import os

SECRET_KEY = os.getenv('SECRET_KEY')
DEBUG = os.getenv('DEBUG', 'False') == 'True'

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME'),
        'USER': os.getenv('DB_USER'),
        'PASSWORD': os.getenv('DB_PASSWORD'),
        'HOST': os.getenv('DB_HOST', 'localhost'),
    }
}

AWS_ACCESS_KEY_ID = os.getenv('AWS_ACCESS_KEY_ID')
AWS_SECRET_ACCESS_KEY = os.getenv('AWS_SECRET_ACCESS_KEY')
```

### Flask配置

```python
# ❌ 错误：硬编码
app.config['SECRET_KEY'] = 'hardcoded-secret'
app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://user:password@localhost/db'

# ✅ 正确：使用环境变量
import os
app.config['SECRET_KEY'] = os.getenv('SECRET_KEY')
app.config['SQLALCHEMY_DATABASE_URI'] = os.getenv('DATABASE_URL')

# ✅ 更好：使用配置类
class Config:
    SECRET_KEY = os.getenv('SECRET_KEY')
    SQLALCHEMY_DATABASE_URI = os.getenv('DATABASE_URL')
    REDIS_URL = os.getenv('REDIS_URL')
    
    @staticmethod
    def init_app(app):
        # 验证必需的环境变量
        required = ['SECRET_KEY', 'DATABASE_URL']
        for var in required:
            if not os.getenv(var):
                raise ValueError(f'Missing required environment variable: {var}')

app.config.from_object(Config)
Config.init_app(app)
```

### 使用python-dotenv

```python
# ✅ 使用.env文件
from dotenv import load_dotenv
import os

# 加载.env文件
load_dotenv()

SECRET_KEY = os.getenv('SECRET_KEY')
DB_PASSWORD = os.getenv('DB_PASSWORD')
API_KEY = os.getenv('API_KEY')

# 验证必需的环境变量
def validate_env():
    required_vars = ['SECRET_KEY', 'DB_PASSWORD', 'API_KEY']
    missing = [var for var in required_vars if not os.getenv(var)]
    
    if missing:
        raise EnvironmentError(f'Missing required environment variables: {", ".join(missing)}')

validate_env()
```

### .env文件管理

```bash
# ❌ 错误：.env文件提交到Git
# 应该在.gitignore中添加：
.env
.env.local
.env.*.local
*.env

# ✅ 正确：使用.env.example作为模板
# .env.example（提交到Git）
SECRET_KEY=
DB_PASSWORD=
API_KEY=
REDIS_URL=redis://localhost:6379

# .env（不提交到Git）
SECRET_KEY=actual_secret_key_here
DB_PASSWORD=actual_password_here
API_KEY=actual_api_key_here
REDIS_URL=redis://localhost:6379
```

### AWS Secrets Manager集成

```python
import boto3
import json
from botocore.exceptions import ClientError

def get_secret(secret_name, region_name='us-east-1'):
    session = boto3.session.Session()
    client = session.client(
        service_name='secretsmanager',
        region_name=region_name
    )
    
    try:
        response = client.get_secret_value(SecretId=secret_name)
        return json.loads(response['SecretString'])
    except ClientError as e:
        raise e

# 使用
secrets = get_secret('prod/db/credentials')
DB_PASSWORD = secrets['password']
DB_USER = secrets['username']
```

### 日志中的敏感信息

```python
import logging

# ❌ 错误：日志记录敏感信息
logging.info(f'User login: username={username}, password={password}')
logging.debug(f'JWT token: {token}')
logging.info(f'Request headers: {request.headers}')

# ✅ 正确：过滤敏感字段
def sanitize_log(data):
    sensitive_fields = ['password', 'token', 'secret', 'authorization', 'api_key']
    sanitized = data.copy() if isinstance(data, dict) else data
    
    if isinstance(sanitized, dict):
        for field in sensitive_fields:
            if field in sanitized:
                sanitized[field] = '***REDACTED***'
    
    return sanitized

logging.info(f'User login: {sanitize_log({"username": username, "password": password})}')

# ✅ 使用自定义日志过滤器
class SensitiveDataFilter(logging.Filter):
    def filter(self, record):
        if hasattr(record, 'msg'):
            msg = str(record.msg)
            # 替换敏感信息
            for pattern in ['password=\\S+', 'token=\\S+', 'secret=\\S+']:
                msg = re.sub(pattern, '***REDACTED***', msg, flags=re.IGNORECASE)
            record.msg = msg
        return True

logger = logging.getLogger(__name__)
logger.addFilter(SensitiveDataFilter())
```

### 配置加密（使用cryptography）

```python
from cryptography.fernet import Fernet
import os

# 生成密钥（只需一次）
# key = Fernet.generate_key()
# print(key.decode())

# 加密配置
def encrypt_config(text, key):
    f = Fernet(key.encode())
    return f.encrypt(text.encode()).decode()

# 解密配置
def decrypt_config(encrypted_text, key):
    f = Fernet(key.encode())
    return f.decrypt(encrypted_text.encode()).decode()

# 使用
ENCRYPTION_KEY = os.getenv('ENCRYPTION_KEY')
encrypted_password = 'gAAAAABh...'
password = decrypt_config(encrypted_password, ENCRYPTION_KEY)
```

### Pydantic配置验证

```python
from pydantic import BaseSettings, Field, validator

class Settings(BaseSettings):
    secret_key: str = Field(..., env='SECRET_KEY')
    db_password: str = Field(..., env='DB_PASSWORD')
    api_key: str = Field(..., env='API_KEY')
    debug: bool = Field(False, env='DEBUG')
    
    @validator('secret_key', 'db_password', 'api_key')
    def validate_not_empty(cls, v):
        if not v or v.strip() == '':
            raise ValueError('Cannot be empty')
        return v
    
    class Config:
        env_file = '.env'
        case_sensitive = False

# 使用
settings = Settings()  # 自动验证和加载环境变量
```

### 检查清单

- [ ] 所有密钥使用环境变量
- [ ] .env文件在.gitignore中
- [ ] 提供.env.example模板
- [ ] 验证必需的环境变量
- [ ] 日志中脱敏敏感信息
- [ ] 检查Git历史无泄露
- [ ] 生产环境使用密钥管理服务
- [ ] 配置文件权限正确（600）
