# 容器与云原生安全

> Docker、Kubernetes等容器环境的安全检查

## 1. Dockerfile安全

### 基础镜像安全
```bash
# 检查是否使用latest标签（不可追溯）
grep -rn "FROM.*:latest\|FROM [^:]*$" Dockerfile

# 检查是否使用官方镜像
grep -rn "FROM" Dockerfile

# 检查基础镜像是否过旧
grep -rn "FROM.*:.*201[0-9]\|FROM.*:.*202[0-2]" Dockerfile
```

**风险**：
- latest标签不固定，可能引入漏洞
- 非官方镜像可能包含后门
- 过旧镜像包含已知漏洞

### 权限配置
```bash
# 检查是否使用root用户
grep -rn "USER root\|^USER 0" Dockerfile

# 检查是否创建非root用户
grep -rn "USER\|useradd\|adduser" Dockerfile

# 检查文件权限
grep -rn "chmod 777\|chmod.*777" Dockerfile
```

**最佳实践**：
```dockerfile
# ❌ 危险：使用root
FROM ubuntu:22.04
RUN apt-get update
CMD ["/app/server"]

# ✅ 安全：创建非root用户
FROM ubuntu:22.04
RUN useradd -m -u 1000 appuser
USER appuser
CMD ["/app/server"]
```

### 敏感信息
```bash
# 检查是否硬编码密钥
grep -rn "ENV.*PASSWORD\|ENV.*SECRET\|ENV.*TOKEN" Dockerfile

# 检查是否复制敏感文件
grep -rn "COPY.*\.env\|COPY.*config\.json\|COPY.*credentials" Dockerfile
```

## 2. Docker Compose安全

### 权限配置
```bash
# 检查是否使用privileged模式
grep -rn "privileged.*true" docker-compose.yml
```

### 卷挂载
```bash
# 检查是否挂载敏感目录
grep -rn "volumes:.*/" docker-compose.yml | grep -E "/etc|/var|/root|/home"
```

## 3. Kubernetes安全

### Pod安全
```bash
# 检查是否使用privileged容器
grep -rn "privileged.*true" k8s/*.yaml

# 检查是否以root运行
grep -rn "runAsUser.*0\|runAsNonRoot.*false" k8s/*.yaml

# 检查是否挂载Docker socket（高危）
grep -rn "/var/run/docker.sock" k8s/*.yaml
```

### Secret管理
```bash
# 检查是否硬编码Secret
grep -rn "kind: Secret" k8s/*.yaml
grep -A 10 "kind: Secret" k8s/*.yaml | grep -E "password|token|key"
```

## 4. 配置管理

### ConfigMap安全
```bash
# 检查ConfigMap是否包含敏感信息
grep -A 20 "kind: ConfigMap" k8s/*.yaml | grep -iE "password|secret|token|key"
```

### 环境变量
```bash
# 检查是否通过环境变量传递密钥
grep -rn "env:" docker-compose.yml k8s/*.yaml | grep -iE "PASSWORD|SECRET|TOKEN"
```

## 防护建议

### Dockerfile最佳实践
```dockerfile
# 1. 使用固定版本的官方镜像
FROM node:18.16.0-alpine

# 2. 创建非root用户
RUN addgroup -g 1000 appgroup && \
    adduser -D -u 1000 -G appgroup appuser

# 3. 切换到非root用户
USER appuser

# 4. 启动命令
CMD ["node", "server.js"]
```

## 检查清单

### 必查项（Critical）
- [ ] 是否使用root用户运行
- [ ] 是否使用privileged模式
- [ ] 是否挂载Docker socket
- [ ] 是否硬编码密钥
- [ ] 是否使用latest标签

### 建议检查（High）
- [ ] 是否使用官方镜像
- [ ] ConfigMap是否包含敏感信息
- [ ] 是否挂载敏感目录
