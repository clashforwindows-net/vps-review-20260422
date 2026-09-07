# VPS 容器化部署与 Docker 生态完全实战指南

> 从 Docker 入门到 K3s 生产级集群，覆盖镜像优化、Compose 编排、CI/CD 流水线、安全加固、监控告警全链路，附完整可复用配置模板

![Docker](https://img.shields.io/badge/Topic-Container_Deployment-blue) ![Level](https://img.shields.io/badge/Level-Intermediate_to_Advanced-blue) ![Updated](https://img.shields.io/badge/Updated-2026--09--07-brightgreen)

---

## 目录

- [容器化技术全景解析](#容器化技术全景解析)
- [Docker 安装与初始化](#docker-安装与初始化)
- [Dockerfile 最佳实践](#dockerfile-最佳实践)
- [镜像优化与多阶段构建](#镜像优化与多阶段构建)
- [Docker Compose 生产级编排](#docker-compose-生产级编排)
- [容器网络深度配置](#容器网络深度配置)
- [容器存储与数据管理](#容器存储与数据管理)
- [Docker 安全加固](#docker-安全加固)
- [K3s 轻量级 Kubernetes 部署](#k3s-轻量级-kubernetes-部署)
- [CI/CD 流水线构建](#cicd-流水线构建)
- [容器监控与日志](#容器监控与日志)
- [故障排查与性能调优](#故障排查与性能调优)
- [一键部署脚本合集](#一键部署脚本合集)
- [VPS 服务商推荐](#vps-服务商推荐)
- [常见问题](#常见问题)

---

## 容器化技术全景解析

### 为什么在 VPS 上使用容器化

| 传统部署 | 容器化部署 | 对比说明 |
|----------|------------|----------|
| 直接安装服务 | 打包为镜像运行 | 环境一致性强 |
| 依赖冲突频繁 | 隔离的运行环境 | 无依赖冲突 |
| 手动配置 | 声明式配置 | 可重复部署 |
| 升级风险高 | 镜像版本管理 | 回滚秒级完成 |
| 资源利用率低 | 按需分配资源 | 密度高 |
| 迁移需重装 | 镜像即用 | 跨平台迁移快 |
| 故障恢复慢 | 重启容器即可 | 恢复秒级 |

### 容器技术选型矩阵

| 方案 | 资源占用 | 学习成本 | 适用场景 | 推荐度 |
|------|----------|----------|----------|--------|
| Docker | 低 | 中 | 单机多服务 | ★★★★★ |
| Docker Compose | 极低 | 低 | 多容器编排 | ★★★★★ |
| K3s | 中 | 中高 | 轻量集群 | ★★★★☆ |
| Podman | 低 | 中 | 无守护进程 | ★★★☆☆ |
| LXC/LXD | 极低 | 低 | 系统容器 | ★★★☆☆ |

### VPS 容器化架构图

```
┌───────────────────────────────────────────────────┐
│                   VPS 主机                         │
├───────────────────────────────────────────────────┤
│  Docker Engine                                    │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐            │
│  │ Nginx   │ │ MySQL   │ │ Redis   │            │
│  │ :80/443 │ │ :3306   │ │ :6379   │            │
│  └────┬────┘ └────┬────┘ └────┬────┘            │
│       │           │           │                   │
│  ┌────┴────┐ ┌────┴────┐ ┌────┴────┐            │
│  │  App 1  │ │  App 2  │ │  App 3  │            │
│  │ Python  │ │  Node   │ │   Go    │            │
│  └─────────┘ └─────────┘ └─────────┘            │
│                                                   │
│  ┌─────────────────────────────────────────┐     │
│  │       Docker Network (bridge)           │     │
│  └─────────────────────────────────────────┘     │
│                                                   │
│  ┌─────────────────────────────────────────┐     │
│  │       Volume / Bind Mount               │     │
│  │  /opt/data/mysql  /opt/data/redis       │     │
│  └─────────────────────────────────────────┘     │
├───────────────────────────────────────────────────┤
│  监控: cAdvisor + Prometheus + Grafana           │
│  日志: Loki + Promtail                            │
│  CI/CD: Gitea + Drone / GitHub Actions           │
└───────────────────────────────────────────────────┘
```

---

## Docker 安装与初始化

### 一键安装脚本

```bash
#!/bin/bash
# docker-install.sh
# Docker 生产级安装脚本（支持 Ubuntu/Debian/CentOS）

set -e

echo "========== Docker 安装脚本 =========="

# 检测系统
if [ -f /etc/os-release ]; then
    . /etc/os-release
    OS=$ID
    VER=$VERSION_ID
    echo "系统: $OS $VER"
else
    echo "无法检测操作系统"
    exit 1
fi

# 安装 Docker
case "$OS" in
    ubuntu|debian)
        # 卸载旧版本
        apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null || true

        # 安装依赖
        apt update
        apt install -y ca-certificates curl gnupg lsb-release

        # 添加 Docker 官方 GPG key（使用国内镜像）
        mkdir -p /etc/apt/keyrings
        curl -fsSL https://download.docker.com/linux/$OS/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg

        # 添加仓库
        echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/$OS $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list

        # 安装
        apt update
        apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
        ;;

    centos|rhel|rocky|almalinux)
        # 卸载旧版本
        yum remove -y docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine 2>/dev/null || true

        # 安装依赖
        yum install -y yum-utils

        # 添加仓库
        yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

        # 安装
        yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
        ;;

    *)
        echo "不支持的系统: $OS"
        exit 1
        ;;
esac

# 配置 Docker
mkdir -p /etc/docker
cat > /etc/docker/daemon.json << 'EOF'
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "50m",
        "max-file": "3"
    },
    "storage-driver": "overlay2",
    "live-restore": true,
    "userland-proxy": false,
    "max-concurrent-downloads": 10,
    "max-concurrent-uploads": 5,
    "default-ulimits": {
        "nofile": {
            "Name": "nofile",
            "Hard": 65535,
            "Soft": 65535
        }
    },
    "registry-mirrors": [
        "https://docker.mirrors.ustc.edu.cn",
        "https://hub-mirror.c.163.com"
    ]
}
EOF

# 启动
systemctl enable docker
systemctl enable containerd
systemctl start docker

# 验证
docker version
docker info

# 将当前用户加入 docker 组（可选）
if [ "$SUDO_USER" ]; then
    usermod -aG docker "$SUDO_USER"
    echo "已将 $SUDO_USER 加入 docker 组（需重新登录生效）"
fi

echo ""
echo "========== Docker 安装完成 =========="
echo "Docker版本: $(docker --version)"
echo "Compose版本: $(docker compose version)"
echo ""
echo "提示："
echo "  - 配置文件: /etc/docker/daemon.json"
echo "  - 数据目录: /var/lib/docker/"
echo "  - 日志限制: 单容器50MB×3文件"
echo "  - 镜像加速: USTC + 网易"
```

### Docker 初始化后配置

```bash
#!/bin/bash
# docker-post-install.sh
# Docker 安装后优化配置

# 1. 配置日志轮转
cat > /etc/logrotate.d/docker << 'EOF'
/var/lib/docker/containers/*/*.log {
    rotate 7
    daily
    compress
    size=50M
    missingok
    delaycompress
    copytruncate
}
EOF

# 2. 清理 cron
cat > /etc/cron.daily/docker-cleanup << 'CRON'
#!/bin/bash
# 清理悬空镜像和停止的容器
docker container prune -f --filter "until=168h" 2>/dev/null
docker image prune -f --filter "until=168h" 2>/dev/null
docker volume prune -f 2>/dev/null
docker network prune -f 2>/dev/null
CRON
chmod +x /etc/cron.daily/docker-cleanup

# 3. 系统参数优化
cat >> /etc/sysctl.conf << 'EOF'

# Docker 网络优化
net.ipv4.ip_forward = 1
net.ipv4.conf.all.forwarding = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

# 文件描述符
fs.file-max = 1048576
fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches = 1048576
EOF
sysctl -p

# 4. 创建常用网络
docker network create --driver bridge frontend 2>/dev/null || true
docker network create --driver bridge backend 2>/dev/null || true
docker network create --driver bridge database 2>/dev/null || true

echo "Docker 后配置完成"
echo "  - 日志轮转: 7天/50MB"
echo "  - 每日清理: 悬空镜像/停止容器"
echo "  - 网络: frontend/backend/database"
```

---

## Dockerfile 最佳实践

### 通用 Dockerfile 模板

```dockerfile
# Dockerfile.python-app
# Python 应用生产级 Dockerfile

# ========== 阶段1: 构建阶段 ==========
FROM python:3.12-slim AS builder

# 设置工作目录
WORKDIR /build

# 安装构建依赖
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# 复制依赖文件
COPY requirements.txt .

# 安装依赖（使用虚拟环境）
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# ========== 阶段2: 运行阶段 ==========
FROM python:3.12-slim AS runtime

# 安装运行时依赖
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# 复制虚拟环境
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# 创建非root用户
RUN groupadd -r appuser && useradd -r -g appuser -d /app -s /sbin/nologin appuser

# 设置工作目录
WORKDIR /app

# 复制应用代码
COPY --chown=appuser:appuser . .

# 切换用户
USER appuser

# 暴露端口
EXPOSE 8000

# 健康检查
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# 启动命令
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "--timeout", "60", "app:app"]
```

### Node.js 应用 Dockerfile

```dockerfile
# Dockerfile.node-app
# Node.js 应用生产级 Dockerfile

# ========== 构建阶段 ==========
FROM node:20-alpine AS builder

WORKDIR /build

# 复制 package 文件
COPY package*.json ./

# 安装依赖（使用 npm ci 更快更可靠）
RUN npm ci --production=false

# 复制源码
COPY . .

# 构建
RUN npm run build

# ========== 运行阶段 ==========
FROM node:20-alpine AS runtime

# 安装 dumb-init（正确处理信号）
RUN apk add --no-cache dumb-init curl

# 创建非root用户
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 -G nodejs

WORKDIR /app

# 复制构建产物和依赖
COPY --from=builder --chown=nodejs:nodejs /build/package*.json ./
COPY --from=builder --chown=nodejs:nodejs /build/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /build/dist ./dist

USER nodejs

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/main.js"]
```

### Go 应用 Dockerfile（极致优化）

```dockerfile
# Dockerfile.go-app
# Go 应用极致优化 Dockerfile

# ========== 构建阶段 ==========
FROM golang:1.22-alpine AS builder

# 安装构建工具
RUN apk add --no-cache git ca-certificates

WORKDIR /build

# 缓存依赖
COPY go.mod go.sum ./
RUN go mod download

# 复制源码
COPY . .

# 静态编译
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags='-w -s -extldflags "-static" -X main.Version=1.0.0' \
    -a -installsuffix cgo \
    -o app \
    ./cmd/app

# ========== 运行阶段（使用 scratch） ==========
FROM scratch AS runtime

# 复制 CA 证书（支持 HTTPS）
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# 复制二进制
COPY --from=builder /build/app /app

# 复制静态文件（如有）
COPY --from=builder /build/static /static

EXPOSE 8080

ENTRYPOINT ["/app"]
```

### Dockerfile 最佳实践检查清单

| 实践项 | 说明 | 镜像大小影响 |
|--------|------|-------------|
| 多阶段构建 | 分离构建和运行环境 | -40%~70% |
| 使用 slim/alpine | 基础镜像最小化 | -60%~80% |
| 合并 RUN 指令 | 减少镜像层数 | -5%~15% |
| 清理 apt 缓存 | `rm -rf /var/lib/apt/lists/*` | -10~50MB |
| .dockerignore | 排除不必要文件 | 视项目 |
| 非 root 用户 | 安全最佳实践 | 无影响 |
| HEALTHCHECK | 容器健康检查 | 无影响 |
| 固定版本标签 | 避免隐式升级 | 无影响 |
| --no-install-recommends | 不装推荐包 | -20~100MB |

---

## 镜像优化与多阶段构建

### .dockerignore 文件

```gitignore
# .dockerignore
# 排除不需要的文件

# Git
.git
.gitignore
.github

# 依赖
node_modules
vendor
__pycache__
*.pyc
.venv
venv

# 文档
*.md
docs/
LICENSE

# 测试
tests/
test/
*_test.go
__tests__/
coverage/
.nyc_output

# IDE
.idea
.vscode
*.swp
*.swo

# 环境
.env
.env.local
.env.*.local
docker-compose*.yml
Dockerfile*

# 日志
*.log
logs/

# 系统
.DS_Store
Thumbs.db

# 构建产物
dist/
build/
target/
out/
bin/
```

### 镜像分析工具

```bash
#!/bin/bash
# docker-image-analyze.sh
# Docker 镜像分析工具

IMAGE=${1:-"nginx:latest"}

echo "========== 镜像分析: $IMAGE =========="

# 1. 镜像大小
echo ""
echo "--- 镜像大小 ---"
docker images "$IMAGE" --format "{{.Repository}}:{{.Tag}} {{.Size}}"

# 2. 镜像层数
echo ""
echo "--- 镜像层 ---"
docker history "$IMAGE" --no-trunc --format "table {{.CreatedBy}}\t{{.Size}}\t{{.CreatedAt}}" | head -30

# 3. 镜像详细信息
echo ""
echo "--- 镜像详情 ---"
docker inspect "$IMAGE" --format '
架构: {{.Architecture}}
系统: {{.Os}}
大小: {{.Size}} bytes
层数: {{len .RootFS.Layers}}
环境变量:
{{range .Config.Env}}  {{.}}
{{end}}
入口点: {{.Config.Entrypoint}}
命令: {{.Config.Cmd}
暴露端口: {{.Config.ExposedPorts}}
'

# 4. 安装 dive 进行深度分析（如已安装）
if command -v dive &>/dev/null; then
    echo ""
    echo "--- Dive 深度分析 ---"
    dive "$IMAGE"
else
    echo ""
    echo "提示: 安装 dive 可进行更深入分析"
    echo "  wget https://github.com/wagoodman/dive/releases/latest/download/dive_linux_amd64.tar.gz"
    echo "  tar xzf dive_linux_amd64.tar.gz"
    echo "  mv dive /usr/local/bin/"
fi
```

### 镜像安全扫描

```bash
#!/bin/bash
# docker-security-scan.sh
# Docker 镜像安全扫描

IMAGE=${1:?"用法: $0 <镜像名>"}

echo "========== 镜像安全扫描: $IMAGE =========="

# 1. Trivy 扫描
if command -v trivy &>/dev/null; then
    echo ""
    echo "--- Trivy 漏洞扫描 ---"
    trivy image --severity HIGH,CRITICAL "$IMAGE"

    echo ""
    echo "--- Trivy 配置审计 ---"
    trivy config --severity HIGH,CRITICAL .
else
    echo "安装 Trivy 进行漏洞扫描:"
    echo "  apt install -y trivy"
fi

# 2. 检查以 root 运行
echo ""
echo "--- Root 用户检查 ---"
USER=$(docker inspect "$IMAGE" --format '{{.Config.User}}')
if [ -z "$USER" ] || [ "$USER" = "root" ] || [ "$USER" = "0" ]; then
    echo "⚠️  警告: 镜像以 root 用户运行"
else
    echo "✅ 以用户 $USER 运行"
fi

# 3. 检查敏感环境变量
echo ""
echo "--- 环境变量检查 ---"
docker inspect "$IMAGE" --format '{{range .Config.Env}}{{println .}}{{end}}' | grep -iE '(PASS|SECRET|KEY|TOKEN|CRED)' || echo "未发现敏感环境变量"

# 4. 检查暴露端口
echo ""
echo "--- 暴露端口 ---"
docker inspect "$IMAGE" --format '{{json .Config.ExposedPorts}}' | jq . 2>/dev/null || echo "无"

# 5. 检查健康检查
echo ""
echo "--- 健康检查 ---"
docker inspect "$IMAGE" --format '{{json .Config.Healthcheck}}' | jq . 2>/dev/null || echo "⚠️ 未配置健康检查"
```

---

## Docker Compose 生产级编排

### 全栈应用编排

```yaml
# docker-compose.yml
# 生产级全栈应用编排

version: "3.9"

services:
  # ========== Nginx 反向代理 ==========
  nginx:
    image: nginx:1.25-alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - nginx_logs:/var/log/nginx
      - nginx_cache:/var/cache/nginx
    networks:
      - frontend
    depends_on:
      - app
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "3"

  # ========== 应用服务 ==========
  app:
    build:
      context: ./app
      dockerfile: Dockerfile
      args:
        - BUILD_ENV=production
    image: myapp:latest
    container_name: app
    restart: unless-stopped
    environment:
      - DATABASE_URL=postgresql://app:${DB_PASSWORD}@postgres:5432/appdb
      - REDIS_URL=redis://redis:6379/0
      - SECRET_KEY=${SECRET_KEY}
      - DEBUG=false
      - LOG_LEVEL=info
    volumes:
      - app_uploads:/app/uploads
      - app_logs:/app/logs
    networks:
      - frontend
      - backend
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      start_period: 30s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 1G
        reservations:
          cpus: "0.5"
          memory: 256M

  # ========== PostgreSQL 数据库 ==========
  postgres:
    image: postgres:16-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=app
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_INITDB_ARGS=--encoding=UTF-8 --locale=C
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init:/docker-entrypoint-initdb.d:ro
      - ./postgres/conf/postgresql.conf:/etc/postgresql/postgresql.conf:ro
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    networks:
      - database
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 1G

  # ========== Redis 缓存 ==========
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
      --appendonly yes
      --appendfsync everysec
    volumes:
      - redis_data:/data
    networks:
      - database
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M

  # ========== 自动备份 ==========
  backup:
    image: prodrigestivill/postgres-backup-local:16
    container_name: backup
    restart: unless-stopped
    environment:
      - POSTGRES_HOST=postgres
      - POSTGRES_DB=appdb
      - POSTGRES_USER=app
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - SCHEDULE=@daily
      - BACKUP_KEEP_DAYS=7
      - BACKUP_KEEP_WEEKS=4
      - BACKUP_KEEP_MONTHS=6
    volumes:
      - ./backups:/backups
    networks:
      - database
    depends_on:
      - postgres

# ========== 网络 ==========
networks:
  frontend:
    driver: bridge
    name: frontend
  backend:
    driver: bridge
    name: backend
    internal: true  # 内部网络，不对外暴露
  database:
    driver: bridge
    name: database
    internal: true

# ========== 卷 ==========
volumes:
  nginx_logs:
  nginx_cache:
  app_uploads:
  app_logs:
  postgres_data:
  redis_data:
```

### Compose 环境变量管理

```bash
# .env
# Docker Compose 环境变量（不要提交到 Git！）

# 数据库
DB_PASSWORD=your_secure_db_password_here

# Redis
REDIS_PASSWORD=your_secure_redis_password_here

# 应用
SECRET_KEY=your_secret_key_here

# 时区
TZ=Asia/Shanghai
```

```bash
# .env.example
# 环境变量模板（可提交到 Git）

DB_PASSWORD=change_me_to_secure_password
REDIS_PASSWORD=change_me_to_secure_redis_password
SECRET_KEY=change_me_to_secure_secret
TZ=Asia/Shanghai
```

### Compose 管理脚本

```bash
#!/bin/bash
# compose-manager.sh
# Docker Compose 管理脚本

COMPOSE_FILE="docker-compose.yml"
ENV_FILE=".env"

case "$1" in
    up)
        echo "启动所有服务..."
        docker compose --env-file "$ENV_FILE" -f "$COMPOSE_FILE" up -d
        docker compose ps
        ;;
    down)
        echo "停止所有服务..."
        docker compose -f "$COMPOSE_FILE" down
        ;;
    restart)
        echo "重启服务: $2"
        docker compose -f "$COMPOSE_FILE" restart "$2"
        ;;
    logs)
        shift
        docker compose -f "$COMPOSE_FILE" logs -f --tail=100 "$@"
        ;;
    status)
        docker compose -f "$COMPOSE_FILE" ps
        echo ""
        echo "资源使用:"
        docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}" $(docker compose -f "$COMPOSE_FILE" ps -q)
        ;;
    build)
        echo "重新构建镜像..."
        docker compose -f "$COMPOSE_FILE" build --no-cache "$2"
        ;;
    update)
        echo "拉取最新镜像并重启..."
        docker compose -f "$COMPOSE_FILE" pull
        docker compose -f "$COMPOSE_FILE" up -d --remove-orphans
        docker image prune -f
        echo "更新完成"
        ;;
    backup)
        echo "手动备份数据库..."
        docker exec postgres pg_dump -U app appdb | gzip > "backups/manual_$(date +%Y%m%d_%H%M%S).sql.gz"
        echo "备份完成"
        ;;
    shell)
        docker exec -it "$2" sh
        ;;
    *)
        echo "用法: $0 {up|down|restart|logs|status|build|update|backup|shell}"
        echo "  up          - 启动所有服务"
        echo "  down        - 停止所有服务"
        echo "  restart S   - 重启服务S"
        echo "  logs [S]    - 查看日志"
        echo "  status      - 查看状态"
        echo "  build [S]   - 重新构建"
        echo "  update      - 拉取并更新"
        echo "  backup      - 备份数据库"
        echo "  shell S     - 进入容器S"
        ;;
esac
```

---

## 容器网络深度配置

### 网络模式对比

| 模式 | 特点 | 使用场景 | 端口冲突 |
|------|------|----------|----------|
| bridge | 默认模式，虚拟网桥 | 大多数场景 | 不会 |
| host | 使用主机网络 | 性能优先 | 会 |
| none | 无网络 | 离线处理 | 不会 |
| macvlan | 容器有独立MAC | 需要独立IP | 不会 |
| overlay | 跨主机通信 | Docker Swarm | 不会 |

### 自定义网络配置

```bash
#!/bin/bash
# docker-network-setup.sh
# Docker 网络配置脚本

# 创建分层网络
docker network create \
    --driver bridge \
    --subnet 172.20.0.0/16 \
    --gateway 172.20.0.1 \
    --opt com.docker.network.bridge.name=brfrontend \
    frontend

docker network create \
    --driver bridge \
    --subnet 172.21.0.0/16 \
    --gateway 172.21.0.1 \
    --opt com.docker.network.bridge.name=brbackend \
    --internal \
    backend

docker network create \
    --driver bridge \
    --subnet 172.22.0.0/16 \
    --gateway 172.22.0.1 \
    --opt com.docker.network.bridge.name=brdatabase \
    --internal \
    database

echo "网络创建完成:"
docker network ls
echo ""
echo "网络拓扑:"
echo "  frontend (172.20.0.0/16) ← 对外暴露"
echo "  backend  (172.21.0.0/16) ← 内部，app通信"
echo "  database (172.22.0.0/16) ← 内部，仅DB"
```

---

## Docker 安全加固

### 容器安全基线

```bash
#!/bin/bash
# docker-security-hardening.sh
# Docker 安全加固脚本

echo "========== Docker 安全加固 =========="

# 1. 配置守护进程安全
cat > /etc/docker/daemon.json << 'EOF'
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "50m",
        "max-file": "3"
    },
    "storage-driver": "overlay2",
    "live-restore": true,
    "userland-proxy": false,
    "no-new-privileges": true,
    "icc": false,
    "disable-legacy-registry": true,
    "content-trust": true,
    "userns-remap": "default",
    "max-concurrent-downloads": 10,
    "max-concurrent-uploads": 5,
    "default-ulimits": {
        "nofile": {"Name": "nofile", "Hard": 65535, "Soft": 65535},
        "nproc": {"Name": "nproc", "Hard": 4096, "Soft": 2048}
    }
}
EOF

systemctl restart docker

# 2. 配置 AppArmor（如已安装）
if command -v apparmor_parser &>/dev/null; then
    cat > /etc/apparmor.d/docker-default << 'EOF'
#include <tunables/global>

profile docker-default flags=(attach_disconnected,mediate_deleted) {
    #include <abstractions/base>
    network,
    capability,
    file,
    umount,
    deny @{PROC}/* w,
    deny @{PROC}/{[^1-9],[^1-9][^0-9],sys/kernel/shm*} wkx,
    deny @{PROC}/sys/kernel/?* wk,
    deny @{PROC}/sys/net/** w,
    deny /sys/[^f]*/** wklx,
    deny /sys/f[^s]*/** wklx,
    deny /sys/fs/[^c]*/** wklx,
    deny /sys/fs/c[^g]*/** wklx,
    deny /sys/fs/cg[^r]*/** wklx,
}
EOF
    apparmor_parser -r /etc/apparmor.d/docker-default
    echo "AppArmor 配置完成"
fi

# 3. 审计 Docker 守护进程
if command -v auditctl &>/dev/null; then
    auditctl -w /usr/bin/docker -p wa -k docker
    auditctl -w /var/lib/docker -p wa -k docker
    auditctl -w /etc/docker -p wa -k docker
    auditctl -w /usr/bin/dockerd -p wa -k docker
    echo "审计规则已添加"
fi

# 4. 限制容器资源（全局默认）
cat > /etc/docker/daemon.json << 'EOF'
{
    "default-shm-size": "64M",
    "default-ulimits": {
        "nofile": {"Name": "nofile", "Hard": 65535, "Soft": 65535},
        "nproc": {"Name": "nproc", "Hard": 4096, "Soft": 2048}
    }
}
EOF

echo ""
echo "========== 安全检查 =========="

# 检查 Docker 安全配置
echo "1. 守护进程配置:"
docker info --format '{{.SecurityOptions}}'

echo "2. 用户命名空间:"
docker info --format 'Userns: {{.SecurityOptions}}' | grep -o 'userns' && echo "✅ 已启用" || echo "⚠️ 未启用用户命名空间"

echo "3. 内容信任:"
docker trust key list 2>/dev/null || echo "⚠️ 内容信任未配置"

echo "4. 镜像扫描:"
which trivy &>/dev/null && echo "✅ Trivy 已安装" || echo "⚠️ 建议安装 Trivy 进行镜像扫描"
```

### 容器运行安全限制

```yaml
# docker-compose.security.yml
# 安全限制配置示例

version: "3.9"

services:
  secure-app:
    image: myapp:latest
    security_opt:
      - no-new-privileges:true
      - apparmor:docker-default
      - seccomp:default
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=100m
      - /run:noexec,nosuid,size=10m
    user: "1000:1000"
    mem_limit: 512m
    memswap_limit: 512m
    cpus: 1.0
    pids_limit: 100
    ulimits:
      nproc: 65535
      nofile:
        soft: 20000
        hard: 40000
    networks:
      - backend
    ports:
      - "8080:8080"
```

---

## K3s 轻量级 Kubernetes 部署

### K3s 安装与配置

```bash
#!/bin/bash
# k3s-install.sh
# K3s 轻量级 Kubernetes 安装脚本

echo "========== K3s 安装 =========="

# 安装 K3s（使用国内镜像源）
curl -sfL https://get.k3s.io | INSTALL_K3S_MIRROR=cn sh -s - server \
    --write-kubeconfig-mode 644 \
    --disable traefik \
    --disable servicelb \
    --flannel-backend=wireguard-native \
    --data-dir /opt/k3s/data \
    --kube-controller-manager-arg "node-monitor-period=5s" \
    --kube-controller-manager-arg "node-monitor-grace-period=20s"

# 等待就绪
echo "等待 K3s 就绪..."
sleep 10
until kubectl get nodes 2>/dev/null | grep -q Ready; do
    echo "  等待中..."
    sleep 5
done

echo ""
echo "K3s 节点状态:"
kubectl get nodes -o wide

echo ""
echo "K3s Pod 状态:"
kubectl get pods -A

# 配置 kubectl
mkdir -p ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
chmod 600 ~/.kube/config

# 安装 Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

echo ""
echo "========== K3s 安装完成 =========="
echo "kubectl: $(kubectl version --client --short 2>/dev/null)"
echo "helm: $(helm version --short)"
```

### K3s 部署应用示例

```yaml
# k8s-app-deployment.yaml
# K3s 应用部署

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
        ports:
        - containerPort: 8000
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

---

## CI/CD 流水线构建

### GitHub Actions + Docker 自动化

```yaml
# .github/workflows/docker-ci.yml
# Docker CI/CD 流水线

name: Docker CI/CD

on:
  push:
    branches: [main, develop]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  IMAGE_NAME: myapp
  REGISTRY: ghcr.io

jobs:
  # ========== 测试 ==========
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Setup Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.12'

    - name: Install dependencies
      run: |
        pip install -r requirements.txt
        pip install pytest pytest-cov flake8

    - name: Lint
      run: flake8 app/ --max-line-length=120

    - name: Test
      run: pytest tests/ --cov=app --cov-report=xml

    - name: Upload coverage
      uses: codecov/codecov-action@v4

  # ========== 构建并推送镜像 ==========
  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
    - uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Login to Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=sha,prefix=sha-

    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        platforms: linux/amd64,linux/arm64

    - name: Scan image
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
        severity: HIGH,CRITICAL
        exit-code: 1

  # ========== 部署 ==========
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
    - name: Deploy to VPS
      uses: appleboy/ssh-action@v1
      with:
        host: ${{ secrets.VPS_HOST }}
        username: ${{ secrets.VPS_USER }}
        key: ${{ secrets.VPS_SSH_KEY }}
        script: |
          cd /opt/myapp
          docker compose pull
          docker compose up -d --remove-orphans
          docker image prune -f
          docker compose ps
```

---

## 容器监控与日志

### Prometheus + Grafana 监控栈

```yaml
# docker-compose.monitoring.yml
# 容器监控栈

version: "3.9"

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_DOMAIN=grafana.example.com
      - GF_SMTP_ENABLED=true
      - GF_SMTP_HOST=smtp.example.com:587
      - GF_SMTP_USER=${SMTP_USER}
      - GF_SMTP_PASSWORD=${SMTP_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    ports:
      - "3000:3000"
    networks:
      - monitoring
    depends_on:
      - prometheus

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
    networks:
      - monitoring

  loki:
    image: grafana/loki:latest
    container_name: loki
    restart: unless-stopped
    volumes:
      - ./loki/loki-config.yml:/etc/loki/config.yml:ro
      - loki_data:/loki
    command: -config.file=/etc/loki/config.yml
    ports:
      - "3100:3100"
    networks:
      - monitoring

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    restart: unless-stopped
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail/promtail-config.yml:/etc/promtail/config.yml:ro
    command: -config.file=/etc/promtail/config.yml
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data:
  grafana_data:
  loki_data:
```

### 容器资源监控脚本

```bash
#!/bin/bash
# docker-monitor.sh
# Docker 容器监控脚本

echo "========== Docker 容器监控 =========="
echo "时间: $(date)"
echo ""

# 1. 容器状态概览
echo "【1】容器状态"
total=$(docker ps -a -q | wc -l)
running=$(docker ps -q | wc -l)
stopped=$((total - running))
echo "  总计: $total | 运行: $running | 停止: $stopped"
echo ""

# 2. 资源使用
echo "【2】资源使用 Top 10"
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}\t{{.BlockIO}}\t{{.PIDs}}" | head -12
echo ""

# 3. 高CPU容器
echo "【3】CPU 使用 >50% 的容器"
docker stats --no-stream --format "{{.Name}} {{.CPUPerc}}" | awk '{cpu=$2; gsub("%","",cpu); if(cpu+0>50) print "  ⚠️ "$1": "$2}'
echo ""

# 4. 高内存容器
echo "【4】内存使用 >80% 的容器"
docker stats --no-stream --format "{{.Name}} {{.MemPerc}}" | awk '{mem=$2; gsub("%","",mem); if(mem+0>80) print "  ⚠️ "$1": "$2}'
echo ""

# 5. 健康检查异常
echo "【5】健康检查异常"
docker ps --format "{{.Names}}" | while read name; do
    status=$(docker inspect "$name" --format '{{.State.Health.Status}}' 2>/dev/null)
    if [ "$status" = "unhealthy" ]; then
        echo "  ❌ $name: unhealthy"
    elif [ "$status" = "starting" ]; then
        echo "  🟡 $name: starting"
    fi
done
echo ""

# 6. 重启次数
echo "【6】重启次数 >3 的容器"
docker ps --format "{{.Names}}" | while read name; do
    count=$(docker inspect "$name" --format '{{.RestartCount}}' 2>/dev/null)
    if [ "$count" -gt 3 ] 2>/dev/null; then
        echo "  ⚠️ $name: 重启 $count 次"
    fi
done
echo ""

# 7. 磁盘使用
echo "【7】Docker 磁盘使用"
docker system df
echo ""

echo "========== 监控完毕 =========="
```

---

## 故障排查与性能调优

### 容器故障排查手册

```bash
#!/bin/bash
# docker-troubleshoot.sh
# Docker 故障排查工具

CONTAINER=${1:-""}

if [ -z "$CONTAINER" ]; then
    echo "========== 全局诊断 =========="

    # 1. Docker 守护进程状态
    echo "【1】Docker 守护进程"
    systemctl status docker --no-pager | head -10
    echo ""

    # 2. 磁盘空间
    echo "【2】磁盘空间"
    df -h /var/lib/docker
    echo ""

    # 3. Docker 系统使用
    echo "【3】Docker 系统"
    docker system df -v
    echo ""

    # 4. 异常容器
    echo "【4】异常状态容器"
    docker ps -a --filter "status=exited" --filter "status=dead" --filter "status=restarting" --format "table {{.Names}}\t{{.Status}}\t{{.Image}}"
    echo ""

    # 5. 网络
    echo "【5】Docker 网络"
    docker network ls
    echo ""

    # 6. 无用资源
    echo "【6】悬空资源"
    echo "悬空镜像: $(docker images -f 'dangling=true' -q | wc -l) 个"
    echo "停止容器: $(docker ps -a -f 'status=exited' -q | wc -l) 个"
    echo "无用卷: $(docker volume ls -f 'dangling=true' -q | wc -l) 个"
    echo ""
else
    echo "========== 容器诊断: $CONTAINER =========="

    # 1. 基本信息
    echo "【1】基本信息"
    docker inspect "$CONTAINER" --format '
名称: {{.Name}}
镜像: {{.Config.Image}}
状态: {{.State.Status}}
启动: {{.State.StartedAt}}
重启次数: {{.RestartCount}}
IP: {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}
'
    echo ""

    # 2. 日志（最近50行）
    echo "【2】最近日志"
    docker logs --tail 50 "$CONTAINER" 2>&1 | tail -50
    echo ""

    # 3. 资源使用
    echo "【3】资源使用"
    docker stats --no-stream "$CONTAINER"
    echo ""

    # 4. 进程列表
    echo "【4】容器内进程"
    docker top "$CONTAINER"
    echo ""

    # 5. 健康检查
    echo "【5】健康检查"
    docker inspect "$CONTAINER" --format '{{json .State.Health}}' | jq . 2>/dev/null || echo "未配置健康检查"
    echo ""

    # 6. 网络配置
    echo "【6】网络配置"
    docker inspect "$CONTAINER" --format '{{json .NetworkSettings}}' | jq . 2>/dev/null | head -30
fi
```

### 常见故障速查表

| 故障现象 | 可能原因 | 解决方案 |
|----------|----------|----------|
| 容器立即退出 | 启动命令错误 | 检查 CMD/ENTRYPOINT |
| 容器 OOM 被杀 | 内存不足 | 增加 mem_limit |
| 端口绑定失败 | 端口被占用 | `ss -tulnp | grep PORT` |
| 无法连接网络 | 网络配置错误 | 检查 network 模式 |
| 镜像拉取失败 | DNS/代理问题 | 配置镜像加速 |
| 磁盘空间满 | 镜像/日志堆积 | `docker system prune` |
| 容器间无法通信 | 防火墙/网络隔离 | 检查网络/internal |
| Build 极慢 | 无缓存/网络差 | 使用 BuildKit 缓存 |

---

## 一键部署脚本合集

### 常用服务一键部署

```bash
#!/bin/bash
# docker-quick-deploy.sh
# 常用服务一键部署

deploy_nginx() {
    echo "部署 Nginx..."
    docker run -d \
        --name nginx \
        --restart unless-stopped \
        -p 80:80 -p 443:443 \
        -v /opt/nginx/conf:/etc/nginx/conf.d:ro \
        -v /opt/nginx/ssl:/etc/nginx/ssl:ro \
        -v /opt/nginx/logs:/var/log/nginx \
        -v /opt/nginx/html:/usr/share/nginx/html:ro \
        --network frontend \
        --health-cmd "wget --spider -q http://localhost/" \
        --health-interval 30s \
        --health-retries 3 \
        nginx:1.25-alpine
    echo "Nginx 已部署 → http://$(hostname -I | awk '{print $1}')"
}

deploy_mysql() {
    local password=${1:-"ChangeMe123!"}
    echo "部署 MySQL..."
    docker run -d \
        --name mysql \
        --restart unless-stopped \
        -e MYSQL_ROOT_PASSWORD="$password" \
        -v /opt/mysql/data:/var/lib/mysql \
        -v /opt/mysql/conf:/etc/mysql/conf.d:ro \
        --network database \
        --health-cmd "mysqladmin ping -h localhost -p$password" \
        --health-interval 10s \
        --health-retries 5 \
        mysql:8.0 \
        --character-set-server=utf8mb4 \
        --collation-server=utf8mb4_unicode_ci \
        --max-connections=200
    echo "MySQL 已部署 → root密码: $password"
}

deploy_redis() {
    local password=${1:-""}
    echo "部署 Redis..."
    local cmd="redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru --appendonly yes"
    [ -n "$password" ] && cmd="$cmd --requirepass $password"
    docker run -d \
        --name redis \
        --restart unless-stopped \
        -v /opt/redis/data:/data \
        --network database \
        redis:7-alpine $cmd
    echo "Redis 已部署"
}

deploy_portainer() {
    echo "部署 Portainer..."
    docker run -d \
        --name portainer \
        --restart unless-stopped \
        -p 9000:9000 \
        -v /var/run/docker.sock:/var/run/docker.sock \
        -v /opt/portainer/data:/data \
        portainer/portainer-ce:latest
    echo "Portainer 已部署 → http://$(hostname -I | awk '{print $1}'):9000"
}

deploy_watchtower() {
    echo "部署 Watchtower（自动更新镜像）..."
    docker run -d \
        --name watchtower \
        --restart unless-stopped \
        -v /var/run/docker.sock:/var/run/docker.sock \
        -e WATCHTOWER_CLEANUP=true \
        -e WATCHTOWER_POLL_INTERVAL=86400 \
        containrrr/watchtower
    echo "Watchtower 已部署 → 每日检查镜像更新"
}

case "$1" in
    nginx) deploy_nginx ;;
    mysql) deploy_mysql "$2" ;;
    redis) deploy_redis "$2" ;;
    portainer) deploy_portainer ;;
    watchtower) deploy_watchtower ;;
    all)
        deploy_nginx
        deploy_portainer
        deploy_watchtower
        echo "基础服务已全部部署"
        ;;
    *)
        echo "用法: $0 {nginx|mysql|redis|portainer|watchtower|all}"
        echo "  nginx            - 部署 Nginx"
        echo "  mysql [password] - 部署 MySQL 8.0"
        echo "  redis [password] - 部署 Redis 7"
        echo "  portainer        - 部署容器管理面板"
        echo "  watchtower       - 部署自动更新工具"
        echo "  all              - 部署 nginx + portainer + watchtower"
        ;;
esac
```

---

## VPS 服务商推荐

### 容器化最佳 VPS 推荐

容器化部署对 VPS 有特殊要求：足够的内存、SSD 存储、较高的 CPU 性能、支持嵌套虚拟化（部分场景）。

#### ⭐ VPSVIP（强烈推荐）

**官网**：[https://vpsvip.net](https://vpsvip.net)

| 项目 | 详情 |
|------|------|
| 机房 | 香港 / 日本 / 美国 / 新加坡 / 韩国 |
| 线路 | CN2 GIA / 优化线路 / BGP 多线 |
| 虚拟化 | KVM（完整虚拟化，支持 Docker） |
| 存储 | NVMe SSD（IOPS 高，容器启动快） |
| 配置 | 1核1G 入门 → 8核16G 企业级 |
| 支付 | 支付宝 / 微信 / 加密货币 |
| 面板 | 快照备份 / 一键重装 / VNC 控制台 |

**容器化推荐理由**：
1. **KVM 虚拟化**：完整支持 Docker/K3s，无嵌套限制
2. **NVMe SSD**：容器镜像读写快，启动时间从分钟级降到秒级
3. **快照备份**：容器环境整盘快照，出问题快速回滚
4. **CN2 线路**：镜像拉取速度有保障，管理响应快
5. **弹性升级**：随容器增长可快速升级配置

#### 其他选择

| 服务商 | 特点 | Docker支持 | 适合场景 |
|--------|------|------------|----------|
| 腾讯云 | 国内大厂 | 优秀 | 国内业务 |
| 阿里云 | 容器服务ACS | 优秀 | 企业级 |
| Vultr | 全球节点 | 良好 | 海外部署 |
| DigitalOcean | 开发者友好 | 良好 | 小型项目 |

---

## 常见问题

### Q: VPS 上应该用 Docker 还是直接安装服务？

A: 如果只有单个简单服务，直接安装即可。如果有2个以上服务、需要频繁更新、或需要在多台 VPS 间迁移，强烈推荐 Docker。

### Q: Docker 占用太多磁盘怎么办？

A: 1) 配置日志轮转（daemon.json 设置 max-size）；2) 定期 `docker system prune`；3) 使用多阶段构建减小镜像；4) 清理无用卷 `docker volume prune`。

### Q: K3s 和 Docker Compose 怎么选？

A: 单机场景用 Docker Compose（简单、学习成本低）；多机或需要高可用（自动故障转移、滚动更新）时用 K3s。1-2台 VPS 选 Compose，3台以上选 K3s。

### Q: 容器被入侵了怎么办？

A: 1) 立即停止容器 `docker stop`；2) 保存容器日志 `docker logs`；3) 导出容器快照 `docker export`；4) 分析入侵路径（通常是通过暴露的端口或弱密码）；5) 重建容器并加固（非root、只读文件系统、网络隔离）；6) 更新所有密码和密钥。

### Q: 如何减小 Docker 镜像大小？

A: 1) 使用多阶段构建；2) 选择 alpine/slim 基础镜像；3) 合并 RUN 指令并清理缓存；4) 使用 .dockerignore 排除无用文件；5) Go/静态语言可用 scratch 镜像；6) 使用 `docker-slim` 工具自动优化。

### Q: 容器间如何安全通信？

A: 1) 使用自定义 Docker 网络（不用默认 bridge）；2) 数据库/缓存用 `internal: true` 内部网络（不对外暴露）；3) 服务间用容器名通信（DNS解析）；4) 敏感信息通过环境变量或 Docker Secrets 传递。

---

## 相关资源

- [VPSVIP](https://vpsvip.net) - 优质 VPS 推荐，KVM/NVMe，容器化首选
- [ClashVIP](https://clashvip.net) - 精选机场推荐
- [导航站](https://nav.clashvip.net) - 工具与资源导航
- [ClashHub](https://clashhub.net) - Clash 配置与教程
- [ClashHub 论坛](https://bbs.clashhub.net) - 技术交流社区
- [Clash for Windows](https://clash-for-windows.net) - 客户端下载
- [Docker 官方文档](https://docs.docker.com) - 官方文档
- [K3s 文档](https://docs.k3s.io) - K3s 官方
- [Compose 规范](https://compose-spec.io) - Compose 标准

---

## 免责声明

1. 本仓库仅提供技术参考
2. 生产环境部署前请在测试环境验证
3. 定期更新基础镜像和依赖
4. 遵守容器镜像的许可证条款
5. 注意敏感信息不要写入 Dockerfile 或 Compose 文件

---

## 许可证

MIT License

---

> 更新时间：2026-09-07 | 专题：VPS 容器化部署与 Docker 生态完全实战 | 字节：~16000+