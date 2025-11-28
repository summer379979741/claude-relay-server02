# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供指导。

## 项目概述

这是 **claude-relay-server02** 的部署仓库 - 分布式 Claude 中继服务集群中的第二个节点。该服务器作为3服务器高可用架构的一部分运行，使用 Redis Sentinel 实现自动故障转移。

**架构角色**：本服务器 (96.44.147.237) 扮演以下角色：
- Redis 从节点（从 10.0.0.2 的主节点复制数据）
- Sentinel 节点2（监控集群的3个哨兵之一）
- Claude Relay 服务实例

该服务提供多个 LLM 提供商（Claude/Anthropic、OpenAI、Google Gemini）的 API 网关/中继，具有 JWT 认证、请求加密和完整的审计日志功能。

## 网络架构

**WireGuard VPN 网络 (10.0.0.0/24)**：
- 10.0.0.1 - 堡垒机（Nginx + Sentinel）
- 10.0.0.2 - Server1（Redis + Sentinel + Claude Relay）
- 10.0.0.3 - Server2（本服务器：Redis + Sentinel + Claude Relay）

**Docker 容器网络 (172.20.0.0/24)**：
- 172.20.0.3 - redis-node2（本服务器）
- 172.20.0.13 - sentinel-node2（本服务器）
- 172.20.0.11 - claude-relay（本服务器）

## 容器管理

**启动所有服务**：
```bash
docker compose up -d
```

**查看运行中的容器**：
```bash
docker ps -a
```

**查看服务日志**：
```bash
# Claude relay 服务
docker logs claude-relay -f

# Redis 从节点
docker logs redis-node2 -f

# Sentinel 哨兵
docker logs sentinel-node2 -f
```

**重启服务**：
```bash
docker compose restart claude-relay
# 或者
docker compose restart redis-node2
docker compose restart sentinel-node2
```

**停止所有服务**：
```bash
docker compose down
```

**拉取最新镜像并重启**：
```bash
docker compose pull claude-relay
docker compose up -d claude-relay
```

## Redis Sentinel 配置

**Sentinel 仲裁数**：2（需要3个哨兵中的多数同意才能进行故障转移）
**主节点名称**：`mymaster`
**故障转移超时**：10秒
**宕机判定时间**：5秒无响应

**检查 Sentinel 状态**：
```bash
docker exec sentinel-node2 redis-cli -p 26379 SENTINEL masters
docker exec sentinel-node2 redis-cli -p 26379 SENTINEL slaves mymaster
docker exec sentinel-node2 redis-cli -p 26379 SENTINEL sentinels mymaster
```

**检查 Redis 复制状态**：
```bash
docker exec redis-node2 redis-cli -a "$REDIS_PASSWORD" INFO replication
```

## 环境变量

所有敏感配置存储在 `.env` 文件中，**必须在所有3台服务器上保持一致**：
- `JWT_SECRET` - JWT 令牌签名密钥
- `ENCRYPTION_KEY` - 数据加密密钥（32字节 base64）
- `REDIS_PASSWORD` - Redis 认证密码

Claude Relay 服务使用以下额外的环境变量（在 docker-compose.yml 中设置）：
- `NODE_ENV=production`
- `PORT=3000`
- `REDIS_SENTINEL_ENABLED=true`
- `REDIS_SENTINEL_HOSTS` - 逗号分隔的 sentinel 端点列表
- `REDIS_SENTINEL_MASTER_NAME=mymaster`
- `LOG_LEVEL=info`
- `CLEANUP_INTERVAL=3600000`（1小时，单位毫秒）
- `TIMEZONE_OFFSET=8`（北京时间/CST）

## 数据持久化

**数据卷**：
- `./redis/node2-data` - Redis AOF 和 RDB 持久化数据
- `./logs` - Claude Relay 审计和错误日志（JSON 格式）
- `./data` - 服务配置和初始化数据

**重要数据文件**：
- `data/init.json` - 初始管理员凭证和服务版本
- `data/supported_models.json` - 按提供商分类的可用 LLM 模型
- `data/model_pricing.json` - 价格信息（大文件，约 936KB）

**日志文件**（JSON 格式，带轮转）：
- `.claude-relay-audit.log.json` - 通用审计跟踪
- `.claude-relay-auth-detail-audit.log.json` - 认证事件
- `.claude-relay-security-audit.log.json` - 安全相关事件
- `.claude-relay-error-audit.log.json` - 错误跟踪

## 健康检查

**Claude Relay 服务**：
```bash
curl http://localhost:3000/health
# 或从容器网络访问
curl http://localhost:3000/health
# 或从其他服务器访问
curl http://10.0.0.3:3000/health
```

**Redis 健康检查**：
```bash
docker exec redis-node2 redis-cli -a "$REDIS_PASSWORD" ping
# 应返回：PONG
```

**Sentinel 健康检查**：
```bash
docker exec sentinel-node2 redis-cli -p 26379 ping
# 应返回：PONG
```

## 故障排查

**如果 Redis 从节点失去与主节点的连接**：
1. 检查服务器之间的网络连接
2. 验证 WireGuard VPN 正在运行：`wg show`
3. 检查主节点是否可达：`ping 10.0.0.2`
4. 查看 Redis 日志：`docker logs redis-node2`

**如果 Sentinel 发起非预期的故障转移**：
1. 检查 `down-after-milliseconds` 设置是否合适（当前为 5000ms）
2. 验证服务器之间的网络稳定性
3. 查看 sentinel 日志：`docker logs sentinel-node2`

**如果 Claude Relay 无法连接到 Redis**：
1. 确保所有3个哨兵都在运行且健康
2. 验证 `REDIS_SENTINEL_HOSTS` 包含所有哨兵端点
3. 检查 `REDIS_PASSWORD` 在所有组件中是否匹配
4. 查看中继服务日志：`docker logs claude-relay`

## 服务配置

Claude Relay 服务镜像（`summer379979741/claude-relay-service:latest`）单独维护，从 Docker Hub 拉取。本仓库仅包含部署配置。

**默认管理员凭证**（来自 `data/init.json`）：
- 用户名：`relayservice`
- 密码：`summer379979741`
（生产环境中应更改这些凭证）

## Git 工作流程

本仓库仅跟踪部署配置（docker-compose.yml、.env 模板等）。

**重要提示**：`.env` 文件包含生产环境机密信息，已在 `.gitignore` 中。切勿提交敏感凭证。

**做出更改前**：
- 配置更改可能影响集群中的所有3台服务器
- 先在非生产环境测试更改
- 修改 sentinel 配置前与其他服务器管理员协调

## 多服务器协调

本服务器是3节点集群的一部分。进行更改时：
1. **Sentinel 配置**必须在所有服务器上协调一致
2. **环境变量**（JWT_SECRET、ENCRYPTION_KEY、REDIS_PASSWORD）必须完全匹配
3. **服务更新**可以顺序推出而不会造成停机
4. **Redis 故障转移**是自动的 - 如果当前主节点失败，哨兵会选举新的主节点


## Docker 网络架构

本项目采用**混合网络模式**：堡垒机使用 host 模式，Server1/Server2 使用 Docker bridge 网络。

**堡垒机 (Server3 - 10.0.0.1)**:
- **网络模式**: `network_mode: host`
- **作用**: 直接使用主机网络栈，简化配置
- **服务暴露**:
  - Sentinel 监听: `10.0.0.1:26379` (通过 WireGuard VPN)
  - Nginx: `80/443` (公网访问)
  - V免签: `8080` (内网访问)

**Server1 (10.0.0.2)**:
- **网络模式**: Docker **bridge** 网络
- **网络名称**: `claude-relay_wireguard`
- **子网段**: `172.20.0.0/24` (避免与 WireGuard VPN 的 10.0.0.0/24 冲突)
- **容器内部IP**:
  - `redis-node1`: 172.20.0.2
  - `sentinel-node1`: 172.20.0.12
  - `claude-relay`: 172.20.0.10
- **端口映射**:
  - `10.0.0.2:6379` → 容器 `172.20.0.2:6379` (Redis Master)
  - `10.0.0.2:3000` → 容器 `172.20.0.10:3000` (Claude Relay API)

**Server2 (10.0.0.3)**:
- **网络模式**: Docker **bridge** 网络
- **网络名称**: `claude-relay_wireguard`
- **子网段**: `172.20.0.0/24`
- **容器内部IP**:
  - `redis-node2`: 172.20.0.3
  - `sentinel-node2`: 172.20.0.13
  - `claude-relay`: 172.20.0.11
- **端口映射**:
  - `10.0.0.3:3000` → 容器 `172.20.0.11:3000` (Claude Relay API)

**网络通信流程**:
1. **外部访问**: 公网 → 堡垒机 Nginx (443) → WireGuard VPN (10.0.0.2:3000 或 10.0.0.3:3000) → Docker bridge → 容器
2. **Redis 主从复制**: Server2 容器通过 WireGuard VPN 访问 `10.0.0.2:6379` → 端口映射 → Server1 容器 Redis
3. **Sentinel 集群**: 三个 Sentinel 通过 WireGuard VPN 相互通信

**为什么使用混合模式？**
- **堡垒机 host 模式**: 简化 Nginx 和 VPN 配置，直接监听公网端口
- **Server1/2 bridge 模式**: 提供网络隔离，避免端口冲突，便于管理多个服务

**⚠️ 重要注意事项**:
- Docker bridge 网络的子网**必须使用 172.20.0.0/24**，不能使用 10.0.0.0/24
- 使用 10.0.0.0/24 会与 WireGuard VPN 冲突，导致路由表混乱和网络中断
- Sentinel 配置必须包含 `sentinel auth-pass mymaster ${REDIS_PASSWORD}`，否则无法连接到 Redis

**验证网络配置**:
```bash
# 检查容器网络模式
docker inspect redis-node2 --format='{{.HostConfig.NetworkMode}}'  # 应显示 "claude-relay_wireguard"

# 检查 bridge 网络子网
docker network inspect claude-relay_wireguard --format="{{range .IPAM.Config}}{{.Subnet}}{{end}}"  # 应显示 "172.20.0.0/24"

# 检查容器 IP 地址
docker inspect redis-node2 --format="{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}"  # Server2 应显示 "172.20.0.3"
```

## 容器命名规范

**当前容器名称** (2025-11-28 更新):
- **Server1**: `redis-node1`, `sentinel-node1`, `claude-relay`
- **Server2**: `redis-node2`, `sentinel-node2`, `claude-relay`
- **堡垒机**: `sentinel-node3`, `nginx-lb`, `vmq-payment`, `vmq-mysql`

**为什么使用 node1/node2 而不是 master/slave？**
- ✅ **容器名称是固定的**，不会随着故障转移而改变
- ✅ **Redis 角色是动态的**，master 和 slave 会在故障转移时互换
- ✅ 避免混淆：`redis-master` 容器可能实际运行的是 slave 角色
- ✅ 更清晰：node1/node2 表示物理节点，角色通过 Sentinel 查询确定

**验证实际角色**:
```bash
# 检查本机 Redis 节点的实际角色（可能是 master 或 slave）
source .env && docker exec redis-node2 redis-cli -a "$REDIS_PASSWORD" info replication 2>/dev/null | grep "role:"

# 通过 Sentinel 查询当前 master 位置（任意节点都可以）
docker exec sentinel-node2 redis-cli -p 26379 sentinel get-master-addr-by-name mymaster
```

## Docker 网络冲突故障排查

**症状**：
- Docker Compose 执行卡在 "Network creating"
- VPN 连接中断，无法 ping 通其他服务器
- `docker network ls` 命令无响应

**原因**：
- Docker bridge 网络使用了与 WireGuard VPN 相同的 IP 段（10.0.0.0/24）
- 路由表冲突导致网络混乱

**解决步骤** (在本服务器上执行):
```bash
# 1. 停止所有容器
cd /opt/claude-relay
docker compose down --remove-orphans

# 2. 清理冲突的网络
docker network prune -f
docker network rm claude-relay_wireguard 2>/dev/null || true

# 3. 重启 WireGuard VPN
sudo systemctl restart wg-quick@wg0

# 4. 验证 VPN 恢复
sleep 5
ping -c 2 10.0.0.1  # ping 堡垒机

# 5. 检查 docker-compose.yml 中的网络配置
grep -A 5 "^networks:" docker-compose.yml
# 确保使用 172.20.0.0/24，而不是 10.0.0.0/24

# 6. 重新启动容器
docker compose up -d
```

**预防措施**：
- ⚠️ 永远不要在 docker-compose.yml 中使用 `subnet: 10.0.0.0/24`
- ✅ 必须使用 `subnet: 172.20.0.0/24` 或其他非冲突网段
- ✅ 修改配置前先备份：`cp docker-compose.yml docker-compose.yml.backup`

## 跨服务器访问

**从本服务器访问其他节点**:
```bash
# 访问堡垒机
ssh relayservice@10.0.0.1

# 访问 Server1 (如果当前在 Server2)
ssh claude@10.0.0.2

# 访问 Server2 (如果当前在 Server1)
ssh relayservice@10.0.0.3

# 测试 VPN 连通性
ping -c 2 10.0.0.1  # 堡垒机
ping -c 2 10.0.0.2  # Server1
ping -c 2 10.0.0.3  # Server2
```

**环境变量同步要求**：
以下环境变量在所有三台服务器上**必须完全一致**：
```bash
JWT_SECRET       # JWT token 验证
ENCRYPTION_KEY   # 数据加密/解密
REDIS_PASSWORD   # Redis 集群认证
```

原因：
- JWT token 需要在不同服务器之间相互验证
- 加密数据存储在共享的 Redis 集群中
- 所有服务器连接同一个 Redis Sentinel 集群
