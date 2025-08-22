# DinD-MCP

基于 Docker-in-Docker 的 MCP (Model Context Protocol) 服务器

## 项目简介

本项目提供了一个完整的 Docker-in-Docker 环境，用于运行 MCP 服务器。通过容器化的方式，可以安全地在隔离环境中运行各种 MCP 服务，目前主要包含 Firecrawl MCP 服务用于网页爬取和内容提取。

## 架构说明

```
┌─────────────────────────────────────────┐
│              宿主机                      │
│  ┌─────────────────────────────────────┐ │
│  │           DinD 容器                 │ │
│  │  ┌─────────────┐  ┌─────────────┐   │ │
│  │  │ Firecrawl   │  │   Nginx     │   │ │
│  │  │    MCP      │  │   Proxy     │   │ │
│  │  │             │  │             │   │ │
│  │  └─────────────┘  └─────────────┘   │ │
│  │         :3000           :8080       │ │
│  └─────────────────────────────────────┘ │
│                    :8080                 │
└─────────────────────────────────────────┘
```

### 服务组件

- **DinD (Docker-in-Docker)**: 提供隔离的 Docker 运行环境
- **Firecrawl MCP**: 网页爬取和内容提取服务
- **Nginx 反向代理**: 提供统一的访问入口和负载均衡
- **Init 服务**: 负责初始化和启动内部服务

## 快速开始

### 前置要求

- Docker 20.10+
- Docker Compose 2.0+

### 配置环境变量

1. 编辑 `mcps/.env` 文件：

```bash
# 替换为你的 Firecrawl API 密钥
FIRECRAWL_API_KEY=fc-your-actual-api-key-here
FIRECRAWL_SSE_LOCAL=true
```

### 启动服务

```bash
# 启动所有服务
docker compose up -d

# 查看服务状态
docker compose ps

# 查看日志
docker compose logs -f
```

### 访问服务

- **健康检查**: http://localhost:8080/_ok
- **Firecrawl MCP**: http://localhost:8080/firecrawl/

## 服务配置

### Firecrawl MCP 配置

Firecrawl MCP 服务用于网页爬取和内容提取，支持以下功能：

- 网页内容爬取
- 结构化数据提取
- 批量处理
- SSE (Server-Sent Events) 支持

### Nginx 代理配置

Nginx 作为反向代理，提供：

- 统一的访问入口 (端口 8080)
- 路径路由 (`/firecrawl/` -> `firecrawl-mcp:3000`)
- WebSocket 支持
- 健康检查端点

## 开发指南

### 目录结构

```
dind-mcp/
├── README.md                 # 项目文档
├── docker-compose.yml        # 主要的 Docker Compose 配置
└── mcps/                     # MCP 服务目录
    ├── .env                  # 环境变量配置
    ├── docker-compose.yml    # 内部服务配置
    ├── firecrawl/           # Firecrawl MCP 服务
    │   └── Dockerfile
    └── nginx/               # Nginx 反向代理
        ├── Dockerfile
        └── nginx.conf
```

### 添加新的 MCP 服务

1. 在 `mcps/` 目录下创建新的服务目录
2. 编写 Dockerfile 和相关配置
3. 在 `mcps/docker-compose.yml` 中添加服务定义
4. 在 `nginx/nginx.conf` 中添加路由规则

### 调试和故障排除

```bash
# 查看所有容器状态
docker compose ps -a

# 查看特定服务日志
docker compose logs firecrawl-mcp
docker compose logs nginx

# 进入 DinD 容器调试
docker compose exec dind sh

# 重启服务
docker compose restart [service-name]
```

## 安全注意事项

⚠️ **重要安全提醒**：

1. **特权模式**: DinD 服务运行在特权模式下，请确保在受信任的环境中使用
2. **TLS 加密**: 当前配置禁用了 Docker TLS 加密，生产环境建议启用
3. **API 密钥**: 请妥善保管 Firecrawl API 密钥，不要提交到版本控制系统
4. **网络访问**: 默认暴露 8080 端口，请根据需要调整防火墙规则

## 常见问题

### Q: 服务启动失败怎么办？

A: 检查以下几点：
- Docker 服务是否正常运行
- 端口 8080 是否被占用
- `.env` 文件中的 API 密钥是否正确
- 查看容器日志获取详细错误信息

### Q: 如何更新服务？

A: 执行以下命令：
```bash
docker compose down
docker compose pull
docker compose up -d --build
```

### Q: 如何备份数据？

A: Docker 数据存储在 `dind-data` 卷中：
```bash
docker run --rm -v dind-data:/data -v $(pwd):/backup alpine tar czf /backup/dind-backup.tar.gz -C /data .
```

## 许可证

本项目采用 MIT 许可证，详见 LICENSE 文件。

## 贡献

欢迎提交 Issue 和 Pull Request 来改进本项目。

## 更新日志

- **v1.0.0**: 初始版本，支持 Firecrawl MCP 服务
- 添加 Nginx 反向代理支持
- 完善文档和配置说明
