# JavaScript/TypeScript 敏感信息检查

## 配置文件硬编码

```bash
# 查找配置文件中的敏感信息
grep -rn "password|secret|token|api_key|apiKey" --include="*.json" --include="*.env" --include="*.config.js" --include="*.config.ts"
```

## 代码中硬编码

```bash
# 查找代码中的硬编码密钥
grep -rn "password.*=.*['\"]|secret.*=.*['\"]|token.*=.*['\"]|apiKey.*=.*['\"]" --include="*.js" --include="*.ts"
```

## 环境变量使用（应该存在）

```bash
# 查找环境变量读取
grep -rn "process\.env\|import\.meta\.env" --include="*.js" --include="*.ts"
```

## 判断逻辑

**❌ 高风险**：
- 找到硬编码 + 没找到环境变量读取

**✅ 安全**：
- 使用 `process.env.XXX` 或 `import.meta.env.XXX`

## 修复示例

### Node.js配置

```javascript
// ❌ 错误：硬编码密钥
const config = {
  database: {
    host: 'localhost',
    user: 'admin',
    password: 'MyPassword123'  // 硬编码
  },
  jwt: {
    secret: 'my-secret-key-12345'  // 硬编码
  },
  aws: {
    accessKeyId: 'AKIAIOSFODNN7EXAMPLE',  // 硬编码
    secretAccessKey: 'wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY'  // 硬编码
  }
};

// ✅ 正确：使用环境变量
const config = {
  database: {
    host: process.env.DB_HOST,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD
  },
  jwt: {
    secret: process.env.JWT_SECRET
  },
  aws: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY
  }
};
```

### .env文件

```bash
# ❌ 错误：.env文件提交到Git
# 应该在.gitignore中添加：
# .env
# .env.local
# .env.*.local

# ✅ 正确：使用.env.example作为模板
# .env.example（提交到Git）
DB_HOST=localhost
DB_USER=
DB_PASSWORD=
JWT_SECRET=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=

# .env（不提交到Git）
DB_HOST=localhost
DB_USER=admin
DB_PASSWORD=actual_password_here
JWT_SECRET=actual_secret_here
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

### 使用dotenv

```javascript
// 1. 安装依赖
// npm install dotenv

// 2. 在应用入口加载
require('dotenv').config();

// 或 ES6
import 'dotenv/config';

// 3. 使用环境变量
const dbPassword = process.env.DB_PASSWORD;
const jwtSecret = process.env.JWT_SECRET;

// 4. 验证必需的环境变量
const requiredEnvVars = [
  'DB_PASSWORD',
  'JWT_SECRET',
  'AWS_ACCESS_KEY_ID'
];

for (const envVar of requiredEnvVars) {
  if (!process.env[envVar]) {
    throw new Error(`Missing required environment variable: ${envVar}`);
  }
}
```

### 前端环境变量（Vite）

```javascript
// ❌ 错误：前端硬编码API密钥
const API_KEY = 'sk-1234567890abcdef';  // 暴露在浏览器中

fetch('https://api.example.com/data', {
  headers: {
    'Authorization': `Bearer ${API_KEY}`
  }
});

// ✅ 正确：通过后端代理
// 前端不应该直接持有敏感密钥
// 应该通过后端API代理请求

// 前端代码
fetch('/api/data');  // 调用自己的后端

// 后端代理
app.get('/api/data', async (req, res) => {
  const API_KEY = process.env.EXTERNAL_API_KEY;
  const response = await fetch('https://api.example.com/data', {
    headers: {
      'Authorization': `Bearer ${API_KEY}`
    }
  });
  res.json(await response.json());
});

// ✅ 如果必须在前端使用配置，使用环境变量
// .env
VITE_API_URL=https://api.example.com

// 前端代码
const apiUrl = import.meta.env.VITE_API_URL;  // 非敏感配置可以暴露
```

### AWS Secrets Manager集成

```javascript
const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secrets-manager');

async function getSecret(secretName) {
  const client = new SecretsManagerClient({ region: 'us-east-1' });
  
  try {
    const response = await client.send(
      new GetSecretValueCommand({ SecretId: secretName })
    );
    
    return JSON.parse(response.SecretString);
  } catch (error) {
    throw error;
  }
}

// 使用
async function initConfig() {
  const dbCredentials = await getSecret('prod/db/credentials');
  
  return {
    database: {
      host: dbCredentials.host,
      user: dbCredentials.username,
      password: dbCredentials.password
    }
  };
}
```

### 日志中的敏感信息

```javascript
// ❌ 错误：日志记录敏感信息
console.log('User login:', { username, password });
console.log('JWT token:', token);
console.log('Request:', req.headers);  // 可能包含Authorization

// ✅ 正确：过滤敏感字段
function sanitizeLog(obj) {
  const sensitiveFields = ['password', 'token', 'secret', 'authorization'];
  const sanitized = { ...obj };
  
  for (const field of sensitiveFields) {
    if (sanitized[field]) {
      sanitized[field] = '***REDACTED***';
    }
  }
  
  return sanitized;
}

console.log('User login:', sanitizeLog({ username, password }));

// ✅ 使用日志库自动脱敏
const pino = require('pino');
const logger = pino({
  redact: {
    paths: ['password', 'token', 'secret', 'authorization', 'req.headers.authorization'],
    censor: '***REDACTED***'
  }
});

logger.info({ username, password }, 'User login');  // password自动脱敏
```

### Git历史中的敏感信息

```bash
# 检查Git历史中是否有敏感信息
git log -p | grep -i "password\|secret\|token"

# 如果发现敏感信息已提交，需要清理历史
# 使用git-filter-repo（推荐）
pip install git-filter-repo
git filter-repo --path-glob '*.env' --invert-paths

# 或使用BFG Repo-Cleaner
java -jar bfg.jar --delete-files .env
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

### 配置加密（使用crypto）

```javascript
const crypto = require('crypto');

// 加密配置
function encryptConfig(text, key) {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-cbc', Buffer.from(key, 'hex'), iv);
  
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  
  return iv.toString('hex') + ':' + encrypted;
}

// 解密配置
function decryptConfig(encryptedText, key) {
  const parts = encryptedText.split(':');
  const iv = Buffer.from(parts[0], 'hex');
  const encrypted = parts[1];
  
  const decipher = crypto.createDecipheriv('aes-256-cbc', Buffer.from(key, 'hex'), iv);
  
  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  
  return decrypted;
}

// 使用
const ENCRYPTION_KEY = process.env.ENCRYPTION_KEY;  // 32字节hex字符串
const encryptedPassword = 'iv:encrypted_data';
const password = decryptConfig(encryptedPassword, ENCRYPTION_KEY);
```

### TypeScript类型安全

```typescript
// ✅ 使用类型确保环境变量存在
interface Env {
  DB_HOST: string;
  DB_USER: string;
  DB_PASSWORD: string;
  JWT_SECRET: string;
}

function getEnv(): Env {
  const env = process.env as Partial<Env>;
  
  const required: (keyof Env)[] = ['DB_HOST', 'DB_USER', 'DB_PASSWORD', 'JWT_SECRET'];
  
  for (const key of required) {
    if (!env[key]) {
      throw new Error(`Missing required environment variable: ${key}`);
    }
  }
  
  return env as Env;
}

const config = getEnv();
// TypeScript确保config.DB_PASSWORD存在
```

### 检查清单

- [ ] 所有密钥使用环境变量
- [ ] .env文件在.gitignore中
- [ ] 提供.env.example模板
- [ ] 验证必需的环境变量
- [ ] 日志中脱敏敏感信息
- [ ] 前端不持有敏感密钥
- [ ] 检查Git历史无泄露
- [ ] 生产环境使用密钥管理服务
