# DIND-MCP

一个基于 Docker-in-Docker (DinD) 的 MCP (Model Context Protocol) 服务集成平台，支持多个 MCP 服务的统一部署和管理。

## 项目概述

本项目提供了一个完整的容器化解决方案，用于部署和管理多个 MCP 服务。通过 Docker-in-Docker 技术，实现了服务的隔离和统一管理，并通过 Nginx 反向代理提供统一的访问入口。

## 架构设计

### 整体架构

```
宿主机:8080 → DinD容器:8080 → Nginx代理 → MCP服务
                                    ├── firecrawl-mcp:3000
                                    └── mcp-12306:3000
```

### 组件说明

- **DinD 容器**: 提供 Docker 运行环境，支持在容器内运行其他容器
- **Nginx 代理**: 统一入口，负责路由分发和负载均衡
- **MCP 服务**: 具体的业务服务，目前支持 Firecrawl 和 12306 两个服务

## 目录结构

```
dind-mcp/
├── README.md                 # 项目说明文档
├── docker-compose.yml        # 主容器编排文件
└── mcps/                     # MCP 服务目录
    ├── .env                  # 环境变量配置
    ├── docker-compose.yml    # MCP 服务编排文件
    ├── nginx/                # Nginx 代理配置
    │   ├── Dockerfile
    │   └── nginx.conf
    ├── firecrawl/            # Firecrawl MCP 服务
    │   └── Dockerfile
    └── 12306/                # 12306 MCP 服务
        └── Dockerfile
```

## 快速开始

### 前置要求

- Docker
- Docker Compose

### 环境配置

1. 配置环境变量文件 `mcps/.env`：

```bash
# Firecrawl API 密钥
FIRECRAWL_API_KEY=fc-YOUR-FIRECRAWL-API-KEY
# 启用本地 SSE 支持
FIRECRAWL_SSE_LOCAL=true
```

### 启动服务

1. 克隆项目到本地
2. 进入项目目录
3. 启动服务：

```bash
docker compose up -d
```

### 验证部署

服务启动后，可以通过以下方式验证：

```bash
# 检查服务状态
curl http://localhost:8080/_ok

# 访问 Firecrawl 服务
curl http://localhost:8080/firecrawl/sse

# 访问 12306 服务
curl http://localhost:8080/12306/sse
```

## 服务详情

### 当前支持的 MCP 服务

#### 1. Firecrawl MCP
- **功能**: 网页抓取和内容提取服务
- **端口**: 3000
- **路由**: `/firecrawl/sse`
- **环境变量**: 
  - `FIRECRAWL_API_KEY`: Firecrawl API 密钥
  - `FIRECRAWL_SSE_LOCAL`: 本地 SSE 支持

#### 2. 12306 MCP
- **功能**: 12306 相关服务接口
- **端口**: 3000
- **路由**: `/12306/sse`

### 路由配置

Nginx 代理配置了以下路由规则：

- **路径路由**:
  - `/firecrawl/` → `firecrawl-mcp:3000`
  - `/12306/` → `mcp-12306:3000`

- **智能路由**: 基于 Cookie 的服务类型识别
  - Cookie `service-type=firecrawl` → Firecrawl 服务
  - Cookie `service-type=12306` → 12306 服务

- **通用端点**:
  - `/messages` → 根据 Cookie 路由到对应服务
  - `/message` → 根据 Cookie 路由到对应服务

## 添加新的 MCP 服务

本项目设计为可扩展架构，支持轻松添加新的 MCP 服务：

### 步骤 1: 创建服务目录

在 `mcps/` 目录下创建新服务目录，例如 `new-service/`：

```bash
mkdir mcps/new-service
```

### 步骤 2: 创建 Dockerfile

在新服务目录中创建 `Dockerfile`：

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY . /app/

RUN npm install -g your-mcp-package

CMD ["npx", "-y", "your-mcp-package", "--port", "3000"]
```

### 步骤 3: 更新 Docker Compose

在 `mcps/docker-compose.yml` 中添加新服务：

```yaml
services:
  # ... 现有服务 ...
  
  new-service-mcp:
    build: ./new-service
    environment:
      - YOUR_ENV_VAR=${YOUR_ENV_VAR}
    restart: always

  proxy:
    # ... 现有配置 ...
    depends_on:
      - firecrawl-mcp
      - mcp-12306
      - new-service-mcp  # 添加依赖
```

### 步骤 4: 更新 Nginx 配置

在 `mcps/nginx/nginx.conf` 中添加路由规则：

```nginx
# 添加到 map $http_cookie $service_type 块中
"~*service-type=newservice" "newservice";

# 添加到 map $service_type $backend_server 块中
"newservice" "http://new-service-mcp:3000";

# 添加新的 location 块
location /newservice/ {
  proxy_set_header Host $host;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection $connection_upgrade;
  add_header Set-Cookie "service-type=newservice; Path=/" always;
  proxy_pass http://new-service-mcp:3000/;
}
```

### 步骤 5: 更新环境变量

如需要，在 `mcps/.env` 中添加新的环境变量：

```bash
YOUR_ENV_VAR=your-value
```

## 技术特性

### Docker-in-Docker (DinD)
- 提供完全隔离的 Docker 环境
- 支持动态容器管理
- 使用镜像加速器提升构建速度

### 健康检查
- DinD 容器配置了健康检查机制
- 确保 Docker 守护进程正常运行后再启动其他服务

### 自动化部署
- 通过 init 容器自动安装 Docker Compose
- 自动构建和启动所有 MCP 服务

### 反向代理
- 统一访问入口
- 支持 WebSocket 连接升级
- 智能路由分发
- 请求头透传

## 故障排除

### 常见问题

1. **服务无法启动**
   - 检查 Docker 是否正常运行
   - 验证端口 8080 是否被占用
   - 查看容器日志：`docker compose logs`

2. **环境变量未生效**
   - 确认 `.env` 文件格式正确
   - 重新构建容器：`docker compose up -d --build`

3. **代理路由错误**
   - 检查 Nginx 配置语法
   - 验证服务容器是否正常运行
   - 查看 Nginx 日志

### 日志查看

```bash
# 查看所有服务日志
docker compose logs

# 查看特定服务日志
docker compose logs dind
docker compose logs init

# 实时跟踪日志
docker compose logs -f
```

## 贡献指南

欢迎提交 Issue 和 Pull Request 来改进项目。在添加新的 MCP 服务时，请确保：

1. 遵循现有的目录结构和命名规范
2. 更新相关的配置文件
3. 提供清晰的文档说明
4. 测试服务的正常运行
