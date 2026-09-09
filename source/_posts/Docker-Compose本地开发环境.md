---
title: Docker Compose 本地开发环境
date: 2026-06-05
tags: [Docker, Docker Compose, 开发环境]
categories: 运维
---

## 一、docker-compose.yml 文件结构

```yaml
version: '3.8'        # Compose 文件版本
services:             # 服务列表
  mysql: ...          # MySQL 服务
  redis: ...          # Redis 服务
volumes:              # 数据卷声明
  mysql_data:
  redis_data:
```

| 部分 | 作用 |
|------|------|
| `version` | Compose 文件格式版本，`3.8` 是常用稳定版本 |
| `services` | 定义需要运行的容器服务，每个服务对应一个容器 |
| `volumes` | 声明持久化数据卷，使数据在容器重启后不丢失 |

---

## 二、MySQL 服务配置详解

```yaml
mysql:
  image:
  environment:
  ports:
  volumes:
    - mysql_data:/var/lib/mysql
  command: --default-authentication-plugin=mysql_native_password
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
    interval: 10s
    timeout: 5s
    retries: 5
```

### 2.1 解析

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `volumes` | `mysql_data:/var/lib/mysql` | 数据持久化到命名卷 |
| `command` | `--default-authentication-plugin=mysql_native_password` | 使用传统密码认证插件 |
| `healthcheck` | 见下文 | 容器健康检查 |

### 2.2 `command` 参数

```yaml
command: --default-authentication-plugin=mysql_native_password
```

- MySQL 8.0 默认使用 `caching_sha2_password` 认证插件
- 部分客户端 / 驱动可能不兼容，改为 `mysql_native_password` 提升兼容性
- **生产环境**建议保留默认插件（更安全）

### 2.3 健康检查

```yaml
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  interval: 10s     # 每 10 秒检查一次
  timeout: 5s        # 单次检查超时时间
  retries: 5         # 失败 5 次判定为 unhealthy
```

> **作用**：确保应用启动时 MySQL 已就绪（配合 `depends_on` + `condition: service_healthy`）

---

## 三、Redis 服务配置详解

```yaml
redis:
  image:
  ports:
  command: redis-server --requirepass redis_pass --appendonly yes
  volumes:
    - redis_data:/data
  healthcheck:
    test: ["CMD", "redis-cli", "-a", "redis_pass", "ping"]
    interval: 10s
    timeout: 5s
    retries: 5
```

### 3.1 解析

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `command` | 带参数的启动命令 | 设置密码 + 开启 AOF 持久化 |
| `volumes` | `redis_data:/data` | 持久化 RDB/AOF 文件 |
| `healthcheck` | `redis-cli -a redis_pass ping` | 通过密码认证后 ping |

### 3.2 启动命令解析

```yaml
command: redis-server --requirepass redis_pass --appendonly yes
```

| 参数 | 含义 |
|------|------|
| `--requirepass redis_pass` | 设置连接密码为 `redis_pass` |
| `--appendonly yes` | 开启 AOF（Append Only File）持久化，每条写命令都记录到磁盘 |


### 3.3 健康检查

```yaml
healthcheck:
  test: ["CMD", "redis-cli", "-a", "redis_pass", "ping"]
```

- 通过密码（`-a redis_pass`）认证后执行 `ping`
- Redis 正常时返回 `PONG`

---

## 四、数据卷 (Volumes) 配置

```yaml
volumes:
  mysql_data:    # MySQL 数据持久化
  redis_data:    # Redis 数据持久化
```

### 4.1 为什么要用命名卷？

| 方式 | 说明 | 缺点 |
|------|------|------|
| **命名卷** (`mysql_data`) | Docker 管理的卷，路径由 Docker 维护 | 无（推荐） |
| **绑定挂载** (`./data:/var/lib/mysql`) | 挂载宿主机目录 | 路径依赖、权限问题 |
| **不挂载** | 容器删除后数据丢失 | 数据无法持久化 |


---

## 五、启动与运维命令

### 5.1 基本操作

```powershell
# 启动并重新构建镜像（Dockerfile 有变更时）
docker-compose up -d --build
```

### 5.2 日志查看

```powershell
# 查看指定服务日志（实时跟踪）
docker-compose logs -f mysql
docker-compose logs -f redis

# 查看最近 100 行日志
docker-compose logs --tail=100 mysql
```

### 5.3 针对单个服务的操作

```powershell
# 停止单个服务
docker-compose stop redis

# 启动单个服务
docker-compose start redis
```

---

## 六、Docker vs 裸机安装对比

| 维度 | 裸机安装 | Docker Compose |
|------|---------|---------------|
| **环境一致性** | 每人机器不同，容易出现"在我这能跑啊" | 镜像完全一致，消除环境差异 |
| **版本管理** | 手动安装/卸载，可能残留文件 | `docker-compose down -v` 一键清空 |
| **迁移成本** | 高，需重新配置环境 | 低，拉镜像即用 |
| **资源隔离** | 端口可能冲突 | 独立网络命名空间，互不影响 |
| **启动速度** | 慢（需手动依次启动各服务） | 快（一条命令启动全部） |

---

## 七、常用 Docker 命令速查

### 7.1 容器管理

| 命令 | 作用 |
|------|------|
| `docker ps` | 查看运行中的容器 |
| `docker ps -a` | 查看所有容器（包括已停止的） |
| `docker start <容器名>` | 启动已停止的容器 |
| `docker stop <容器名>` | 停止运行中的容器 |
| `docker restart <容器名>` | 重启容器 |
| `docker rm <容器名>` | 删除已停止的容器 |
| `docker rm -f <容器名>` | 强制删除运行中的容器 |
| `docker exec -it <容器名> bash` | 进入容器的交互式终端 |
| `docker logs <容器名>` | 查看容器日志 |
| `docker logs -f <容器名>` | 实时跟踪容器日志 |
| `docker inspect <容器名>` | 查看容器详细信息（网络、挂载、环境变量等） |
| `docker rename <旧名> <新名>` | 重命名容器 |

### 7.2 镜像管理

| 命令 | 作用 |
|------|------|
| `docker images` | 列出本地所有镜像 |
| `docker pull <镜像名>:<tag>` | 拉取镜像 |
| `docker rmi <镜像名>` | 删除镜像 |
| `docker build -t <名称>:<tag> .` | 根据 Dockerfile 构建镜像 |
| `docker tag <源> <目标>` | 给镜像打标签 |
| `docker push <镜像名>` | 推送镜像到仓库 |
| `docker image prune` | 清理未使用的镜像 |

### 7.3 数据卷管理

| 命令 | 作用 |
|------|------|
| `docker volume ls` | 列出所有数据卷 |
| `docker volume create <卷名>` | 创建数据卷 |
| `docker volume rm <卷名>` | 删除数据卷 |
| `docker volume inspect <卷名>` | 查看数据卷详细信息（挂载路径等） |
| `docker volume prune` | 清理未使用的数据卷 |

### 7.4 网络管理

| 命令 | 作用 |
|------|------|
| `docker network ls` | 列出所有网络 |
| `docker network create <网络名>` | 创建自定义网络 |
| `docker network rm <网络名>` | 删除网络 |
| `docker network inspect <网络名>` | 查看网络详细信息 |
| `docker network prune` | 清理未使用的网络 |

### 7.5 系统清理

| 命令 | 作用 |
|------|------|
| `docker system df` | 查看 Docker 磁盘占用 |
| `docker system prune` | 清理所有未使用的资源（容器、网络、镜像、缓存） |
| `docker system prune -a` | 清理所有未使用的资源（含未使用的镜像） |
| `docker system prune -a --volumes` | 清理所有未使用的资源（含数据卷）⚠️ 谨慎使用 |

### 7.6 Compose 相关

| 命令 | 作用 |
|------|------|
| `docker-compose up -d` | 后台启动所有服务 |
| `docker-compose down` | 停止并删除容器和网络 |
| `docker-compose down -v` | 停止并删除容器、网络和数据卷(⚠️ 重置所有数据!)|
| `docker-compose ps` | 查看 Compose 管理的容器状态 |
| `docker-compose logs -f` | 查看所有服务日志 |
| `docker-compose restart <服务名>` | 重启指定服务 |
| `docker-compose build` | 重新构建镜像 |
| `docker-compose pull` | 拉取最新镜像 |
| `docker-compose exec <服务名> bash` | 进入服务容器内部 |
| `docker-compose config` | 验证并查看合并后的 Compose 配置 |


