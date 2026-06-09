# Docker Networking Modes Explained

> 中文 | [English](#english)

---

# 中文文档

## 📖 目录

- [1. 概述](#1-概述)
- [2. Docker 网络基础](#2-docker-网络基础)
- [3. 网络模式详解](#3-网络模式详解)
  - [3.1 Bridge 模式](#31-bridge-模式)
  - [3.2 Host 模式](#32-host-模式)
  - [3.3 None 模式](#33-none-模式)
  - [3.4 Container 模式](#34-container-模式)
  - [3.5 Overlay 模式](#35-overlay-模式)
  - [3.6 Macvlan 模式](#36-macvlan-模式)
  - [3.7 IPvlan 模式](#37-ipvlan-模式)
- [4. 自定义网络](#4-自定义网络)
- [5. 端口映射](#5-端口映射)
- [6. DNS 配置](#6-dns-配置)
- [7. 网络安全](#7-网络安全)
- [8. Docker Compose 网络](#8-docker-compose-网络)
- [9. 多机网络（Swarm/Overlay）](#9-多机网络-swarmoverlay)
- [10. 网络故障排查](#10-网络故障排查)
- [11. 常用命令](#11-常用命令)
- [12. 常见问题](#12-常见问题)

---

## 1. 概述

Docker 网络是容器化应用的核心组件，它决定了：

| 功能 | 说明 |
|------|------|
| 容器互联 | 容器之间如何通信 |
| 外部访问 | 外部如何访问容器内的服务 |
| 网络隔离 | 不同容器组之间的隔离策略 |
| 服务发现 | 容器如何通过名称互相发现 |
| 负载均衡 | 如何分发到多个容器实例 |

**核心概念：**

- **Network Namespace** — 每个容器拥有独立的网络栈
- **veth pair** — 虚拟以太网对，连接容器和宿主机
- **iptables** — Docker 利用 iptables 实现 NAT 和端口映射
- **DNS** — 内置 DNS 实现容器名称解析

## 2. Docker 网络基础

### 安装 Docker

**Docker Engine：**

```bash
# Ubuntu / Debian
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 验证安装
sudo docker --version
sudo docker info
```

**Docker Desktop：**

从 [docker.com](https://www.docker.com/products/docker-desktop/) 下载安装包，或使用包管理器：

```bash
# macOS (Homebrew)
brew install --cask docker

# Windows (winget)
winget install Docker.DockerDesktop
```

### 查看 Docker 网络

```bash
# 列出所有网络
docker network ls

# 网络列表示例输出：
NETWORK ID     NAME      DRIVER    SCOPE
a1b2c3d4e5f6   bridge    bridge    local
b2c3d4e5f6g7   host      host      local
c3d4e5f6g7h8   none      null      local
```

**默认网络对比：**

| 网络 | 驱动 | 说明 |
|------|------|------|
| bridge | bridge | 默认网络，NAT 模式 |
| host | host | 共享宿主机网络栈 |
| none | null | 无网络 |

## 3. 网络模式详解

### 3.1 Bridge 模式

**默认模式**，Docker 创建一个虚拟网桥 `docker0`，所有容器通过 veth 对连接到这个网桥。

```
宿主机
┌─────────────────────────────────────┐
│  ┌──────────────┐                    │
│  │   docker0    │ 172.17.0.1/16     │
│  │  (网桥)      │                    │
│  └──┬───┬───┬──┘                    │
│     │   │   │  veth 对              │
│  ┌──┘ ┌─┘ ┌─┘                       │
│  │    │    │                         │
│ ┌┴┐  ┌┴┐  ┌┴┐                       │
│ │C1│ │C2│ │C3│  容器 (172.17.0.x)   │
│ └─┘  └─┘  └─┘                       │
│         │                            │
│     eth0 │ 172.17.0.1 -> NAT -> 外部 │
└─────────┼──────────────────────────┘
          │
     Internet
```

**工作原理：**
1. Docker 创建 `docker0` 网桥（默认 172.17.0.0/16）
2. 每个容器创建一对 veth 接口
3. 一端在容器内（eth0），一端在宿主机上
4. 宿主机端连接到 `docker0` 网桥
5. 通过 NAT（iptables MASQUERADE）访问外部网络

**配置示例：**

```bash
# 创建容器并指定网络模式（默认就是 bridge）
docker run -d --name web --network bridge nginx

# 指定 IP 地址
docker run -d --name web --network bridge --ip 172.17.0.10 nginx

# 查看容器网络详情
docker inspect web --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool
```

**默认 bridge 网络 vs 自定义 bridge 网络：**

| 特性 | 默认 bridge | 自定义 bridge |
|------|------------|-------------|
| DNS 解析 | ❌ 需 --link | ✅ 自动 DNS |
| 网络隔离 | ❌ 所有容器互通 | ✅ 可按需隔离 |
| 配置修改 | 需重启 Docker | ✅ 动态修改 |
| 附加/分离 | ❌ 需重建容器 | ✅ 运行时切换 |

### 3.2 Host 模式

容器共享宿主机的网络栈，不进行网络隔离。

```bash
docker run -d --name web --network host nginx
```

**特点：**
- ✅ 性能最高（无 NAT 开销）
- ✅ 端口自动暴露（容器端口就是宿主机端口）
- ❌ 端口冲突风险
- ❌ 网络隔离性差
- ❌ 仅适用于 Linux

**适用场景：**
- 对网络性能要求极高的服务（如 Redis、负载均衡器）
- 需要监控宿主机网络状态的工具
- 单机部署的监控 Agent

```bash
# 示例：在 host 模式下运行 nginx
# 访问 http://宿主机IP:80 即可（无需 -p 映射）
docker run -d --name nginx-host --network host nginx

# 验证：容器内看到的网络就是宿主机的
docker exec nginx-host ip addr
docker exec nginx-host hostname -I
```

### 3.3 None 模式

容器没有网络接口，完全隔离。

```bash
docker run -d --name isolated --network none alpine sleep infinity
```

**特点：**
- ✅ 最高的网络安全性
- ✅ 无 IP 地址，完全隔离
- ❌ 无法访问外部网络
- ❌ 外部也无法访问容器

**适用场景：**
- 离线计算任务
- 本地文件处理
- 安全沙箱

### 3.4 Container 模式

容器共享另一个容器的网络栈。

```
┌──────────────┐
│  容器 A      │
│  eth0: 10.x  │
└──────┬───────┘
       │ 共享网络命名空间
┌──────┴───────┐
│  容器 B      │
│  (无独立eth0) │
│  通过 lo 与 A │
│  通信         │
└──────────────┘
```

```bash
# 先创建一个基础容器
docker run -d --name base --network bridge nginx

# 第二个容器共享 base 的网络
docker run -d --name client --network container:base alpine sleep infinity

# 验证：两个容器 IP 相同
docker exec base hostname -I
docker exec client hostname -I
```

**适用场景：**
- Sidecar 模式（如 Istio Envoy）
- 日志收集器与主容器共享网络
- 网络调试工具

### 3.5 Overlay 模式

跨主机的容器网络，用于 Docker Swarm 集群。

```
┌─ 宿主机 A ─────────────┐    ┌─ 宿主机 B ─────────────┐
│ ┌─ overlay ─┐           │    │ ┌─ overlay ─┐           │
│ │ 10.0.0.2  │───┐       │    │ │ 10.0.0.3  │───┐       │
│ │ 容器 A     │   │       │    │ │ 容器 B     │   │       │
│ └───────────┘   │       │    │ └───────────┘   │       │
│                 │ VXLAN │    │                 │ VXLAN │
│ ┌─ ingress ──┐  │ 隧道  │    │ ┌─ ingress ──┐  │ 隧道  │
│ │  eth0      │──┘       │    │ │  eth0      │──┘       │
│ └───────────┘           │    │ └───────────┘           │
└─────────────────────────┘    └─────────────────────────┘
```

**创建 Overlay 网络：**

```bash
# 初始化 Swarm（在管理节点）
docker swarm init

# 创建 overlay 网络
docker network create -d overlay --attachable my-overlay

# 在不同宿主机上运行容器（需加入 Swarm）
docker service create --name web --network my-overlay --replicas 3 nginx
```

**技术原理：**
- 使用 VXLAN 封装技术在宿主机之间建立隧道
- 每个容器获得独立的虚拟 IP
- 支持服务发现和负载均衡

### 3.6 Macvlan 模式

为容器分配与宿主机同一网段的 MAC 地址和 IP 地址。

```bash
# 创建 macvlan 网络
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  macvlan-net

# 运行容器直接使用物理网络 IP
docker run -d --name web --network macvlan-net --ip 192.168.1.100 nginx
```

**特点：**
- ✅ 容器拥有独立 MAC 地址，看起来像独立设备
- ✅ 无需端口映射，直接通过分配的 IP 访问
- ✅ 性能接近物理机
- ❌ 宿主机无法直接访问 macvlan 容器（需创建子接口）
- ❌ 部分云环境不支持（MAC 地址过滤）

**适用场景：**
- 传统网络架构迁移
- 需要独立 MAC 地址的场景
- 高性能网络需求

### 3.7 IPvlan 模式

类似于 Macvlan，但所有容器共享同一个 MAC 地址。

```bash
# L2 模式
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  -o ipvlan_mode=l2 \
  ipvlan-l2

# L3 模式（无需网关）
docker network create -d ipvlan \
  --subnet=10.0.0.0/24 \
  -o parent=eth0 \
  -o ipvlan_mode=l3 \
  ipvlan-l3
```

**特点：**
- ✅ MAC 地址消耗少
- ✅ 支持大量容器
- ✅ L3 模式减少广播流量
- ❌ 配置相对复杂

**网络模式对比总结：**

| 模式 | 隔离性 | 性能 | 跨主机 | 适用场景 |
|------|--------|------|--------|---------|
| Bridge | 中 | 中 | ❌ | 单机部署，默认选择 |
| Host | 低 | 高 | ❌ | 高性能需求 |
| None | 高 | - | ❌ | 离线/安全场景 |
| Container | - | - | ❌ | Sidecar 模式 |
| Overlay | 中 | 低 | ✅ | Swarm 集群 |
| Macvlan | 低 | 高 | ❌ | 传统网络迁移 |
| IPvlan | 低 | 高 | ❌ | 大规模部署 |

## 4. 自定义网络

**创建自定义网络：**

```bash
# 创建 bridge 网络
docker network create --driver bridge \
  --subnet=10.0.0.0/24 \
  --ip-range=10.0.0.0/24 \
  --gateway=10.0.0.1 \
  --label project=myapp \
  my-network

# 创建网络时指定 MTU
docker network create --driver bridge \
  --opt com.docker.network.driver.mtu=1450 \
  mtu-network
```

**容器加入网络：**

```bash
# 创建时指定网络
docker run -d --name app1 --network my-network nginx

# 运行时加入额外网络
docker network connect my-network app1

# 运行时断开网络
docker network disconnect my-network app1

# 查看容器网络连接
docker inspect app1 --format '{{range $net,$v := .NetworkSettings.Networks}}{{$net}} {{end}}'
```

**网络内部通信（自动 DNS）：**

```bash
# 创建两个容器在同一自定义网络
docker run -d --name web --network my-network nginx
docker run -d --name app --network my-network alpine sleep infinity

# 通过容器名通信（自定义网络支持自动 DNS）
docker exec app ping web

# 默认 bridge 网络需要 --link（不推荐）
docker run -d --name app2 --link web alpine sleep infinity
```

## 5. 端口映射

```bash
# 基本端口映射
docker run -d -p 8080:80 nginx
# 访问: http://宿主机IP:8080 -> 容器80端口

# 指定主机 IP
docker run -d -p 127.0.0.1:8080:80 nginx

# 随机端口（宿主机随机分配）
docker run -d -P nginx

# UDP 端口映射
docker run -d -p 53:53/udp --dns 8.8.8.8 coredns

# 多个端口
docker run -d -p 80:80 -p 443:443 -p 3306:3306 myapp

# 查看端口映射
docker port container_name
```

**端口映射原理（iptables）：**

```bash
# Docker 自动添加的 iptables 规则
# DNAT: 将宿主机 8080 端口流量转发到容器 80 端口
iptables -t nat -L DOCKER

# 输出示例
Chain DOCKER (2 references)
target     prot opt source     destination
DNAT       tcp  --  anywhere   0.0.0.0/0  tcp dpt:8080 to:172.17.0.2:80
```

**端口冲突解决：**

```bash
# 检查端口占用
sudo netstat -tlnp | grep 8080
# 或
sudo lsof -i :8080

# 更改映射端口
docker run -d -p 8081:80 nginx  # 改为 8081

# 删除并重新创建容器
docker rm -f web
docker run -d --name web -p 8080:80 nginx
```

## 6. DNS 配置

**容器 DNS 设置：**

```bash
# 启动时指定 DNS
docker run -d --name web \
  --dns 8.8.8.8 \
  --dns 1.1.1.1 \
  --dns-search example.com \
  --dns-opt ndots:2 \
  nginx

# 查看容器 DNS 配置
docker exec web cat /etc/resolv.conf

# 宿主机 DNS 配置影响
# /etc/docker/daemon.json
{
  "dns": ["8.8.8.8", "1.1.1.1"],
  "dns-opts": ["ndots:2"],
  "dns-search": ["example.com"]
}
```

**自定义网络 DNS：**

```bash
# 自定义 bridge 网络提供自动 DNS 解析
docker network create my-net

docker run -d --name api --network my-net myapi:latest
docker run -d --name web --network my-net nginx

# web 容器可以通过 "api" 主机名访问 api 容器
docker exec web curl http://api:3000/health
```

**Swarm 服务 DNS：**

```bash
# 创建服务时自动注册 DNS 名称
docker service create --name myapp --replicas 3 nginx

# 通过服务名访问（DNS 自动负载均衡）
ping myapp
# 返回多个 VIP 之一
```

## 7. 网络安全

**网络隔离：**

```bash
# 创建隔离的桥接网络
docker network create --internal internal-net
# --internal: 禁止容器访问外部网络

# 容器只能与同网络的容器通信
docker run -d --name db --network internal-net postgres:13
docker run -d --name cache --network internal-net redis:7
docker run -d --name api --network my-net --network-alias backend myapp
```

**网络策略（使用 iptables）：**

```bash
# 阻止容器访问外部网络
docker network create --internal isolated-net

# 限制容器间通信（默认 bridge 所有容器互通）
# 创建多个自定义网络实现隔离
docker network create frontend-net
docker network create backend-net
docker network create database-net

# 前端可访问后端
docker network connect frontend-net web
docker network connect backend-net web  # web 连接两个网络

# 数据库只能后端访问
docker run -d --name postgres --network database-net postgres:13
docker network connect backend-net postgres  # 后端也能访问 database-net
```

**TLS/SSL 加密（容器间通信）：**

```bash
# 使用 Docker 内置加密（Swarm Overlay）
docker network create -d overlay \
  --opt encrypted \
  my-secure-overlay
```

## 8. Docker Compose 网络

```yaml
version: '3.8'

services:
  # Web 前端
  web:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    networks:
      - frontend
      - backend
    depends_on:
      - api

  # API 后端
  api:
    image: myapp:latest
    expose:
      - "3000"
    environment:
      - DB_HOST=database
      - REDIS_HOST=redis
    networks:
      - backend
    depends_on:
      database:
        condition: service_healthy
      redis:
        condition: service_started

  # 数据库
  database:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - internal
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  # 缓存
  redis:
    image: redis:7-alpine
    networks:
      - internal

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: false  # 允许外部访问（通过 api 容器）
  internal:
    driver: bridge
    internal: true   # 完全隔离

volumes:
  pgdata:
```

**Compose 网络配置参数：**

```yaml
networks:
  mynet:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: my-bridge
      com.docker.network.bridge.enable_icc: "true"
      com.docker.network.bridge.enable_ip_masquerade: "true"
    ipam:
      driver: default
      config:
        - subnet: 10.20.0.0/16
          gateway: 10.20.0.1
    labels:
      project: myapp
```

## 9. 多机网络（Swarm/Overlay）

**初始化 Swarm 集群：**

```bash
# 管理节点
docker swarm init --advertise-addr 192.168.1.10

# 工作节点加入
docker swarm join --token SWMTKN-1-xxxxx 192.168.1.10:2377

# 查看节点
docker node ls
```

**创建 Overlay 网络：**

```bash
# 创建可附加的 overlay 网络（允许独立容器加入）
docker network create -d overlay \
  --subnet=10.0.0.0/24 \
  --gateway=10.0.0.1 \
  --attachable \
  my-overlay

# 部署服务
docker service create \
  --name web \
  --replicas 3 \
  --network my-overlay \
  --publish 80:80 \
  nginx
```

**服务发现与负载均衡：**

```bash
# 服务自动注册 DNS
docker service create --name users-api --replicas 2 myapi

# 其他服务通过 "users-api" 访问
# Docker 内置 DNS 轮询负载均衡

# 查看服务端点
docker service ps users-api

# 查看服务详情
docker service inspect users-api
```

**Routing Mesh（路由网格）：**

```bash
# 发布端口到所有节点
docker service create \
  --name web \
  --publish mode=host,target=80,published=8080 \
  --replicas 5 \
  nginx

# 访问任意节点 8080 端口都能到达服务
```

## 10. 网络故障排查

```bash
# 检查容器网络
docker exec container_name ip addr
docker exec container_name ip route
docker exec container_name cat /etc/resolv.conf

# 测试连通性
docker exec container_name ping <target>
docker exec container_name curl -v http://<target>

# 查看网络详情
docker network inspect <network_name>

# 容器网络故障排查
docker run --rm -it --network container:<target> nicolaka/netshoot

# 查看 iptables 规则
sudo iptables -t nat -L -n -v
sudo iptables -L -n -v

# 抓包分析
sudo tcpdump -i docker0 -n
sudo tcpdump -i any port 80 -n

# 查看 bridge 设备
brctl show docker0
ip link show docker0
```

**常见 DNS 问题排查：**

```bash
# 检查容器 DNS 配置
docker exec container_name cat /etc/resolv.conf

# 测试 DNS 解析
docker exec container_name nslookup google.com
docker exec container_name dig google.com

# 检查 Docker daemon DNS 配置
docker info | grep -i dns
```

**网络诊断工具容器：**

```bash
# 使用 netshoot 工具镜像
docker run -it --rm --network host nicolaka/netshoot

# 包含的工具：curl, wget, dig, nslookup, nmap, tcpdump, iperf, iftop 等
```

## 11. 常用命令

```bash
# 网络管理
docker network ls                          # 列出网络
docker network inspect <name>             # 查看网络详情
docker network create <name>              # 创建网络
docker network rm <name>                  # 删除网络
docker network prune                      # 清理未使用的网络
docker network connect <net> <container>  # 容器加入网络
docker network disconnect <net> <container>  # 容器离开网络

# 端口映射
docker port <container>                   # 查看端口映射
docker run -p <host>:<container>          # 端口映射

# DNS
docker exec <container> cat /etc/resolv.conf  # 查看 DNS
docker exec <container> nslookup <name>       # DNS 查询

# 诊断
docker exec <container> ip addr           # 查看 IP
docker exec <container> ip route          # 查看路由
docker exec <container> ping <target>     # Ping 测试
docker exec <container> curl <url>        # HTTP 测试
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container>
```

**Docker daemon 网络配置（/etc/docker/daemon.json）：**

```json
{
  "bip": "10.10.0.1/16",
  "default-address-pools": [
    {"base": "172.16.0.0/12", "size": 24},
    {"base": "172.20.0.0/14", "size": 24}
  ],
  "dns": ["8.8.8.8", "1.1.1.1"],
  "icc": false,
  "iptables": true,
  "ip-forward": true,
  "ip-masq": true,
  "mtu": 1450,
  "userland-proxy": false,
  "experimental": true
}
```

**重启 Docker 服务：**

```bash
sudo systemctl restart docker
# 注意：重启会停止所有运行中的容器
```

## 12. 常见问题

### Q1: 容器无法访问外部网络

**原因：** 可能的原因包括 DNS 配置错误、iptables 规则被修改、IP 转发未启用。

**解决：**

```bash
# 检查 IP 转发
sysctl net.ipv4.ip_forward
# 如果为 0，启用
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

# 检查 iptables
sudo iptables -t nat -L POSTROUTING -n -v

# 检查 DNS
docker exec container cat /etc/resolv.conf

# 重启 Docker
sudo systemctl restart docker
```

### Q2: 容器间通信失败

**原因：** 容器在不同网络、防火墙规则阻止、端口未暴露。

**解决：**

```bash
# 确保在同一网络
docker network connect my-network container-b

# 检查容器 IP
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container-a
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container-b

# 从容器内测试
docker exec container-a ping container-b
docker exec container-a curl http://container-b:port
```

### Q3: 端口映射不生效

**原因：** 端口已被占用、只绑定到了 localhost、防火墙阻止。

**解决：**

```bash
# 检查端口占用
sudo netstat -tlnp | grep <port>

# 检查容器端口映射
docker port container_name

# 检查映射绑定地址（-p 127.0.0.1:80:80 只允许本机访问）
# 改为 -p 0.0.0.0:80:80 或 -p 80:80

# 检查防火墙
sudo iptables -L INPUT -n | grep <port>
sudo ufw status
```

### Q4: Overlay 网络性能差

**原因：** VXLAN 封装开销、MTU 问题、加密开销。

**解决：**

```bash
# 调整 MTU（适配底层网络）
docker network create -d overlay \
  -o "com.docker.network.driver.mtu=1450" \
  overlay-mtu

# 避免不必要的数据加密
# 仅在需要时使用 --opt encrypted

# 使用 macvlan 替代 overlay（如果不需要跨主机）
```

### Q5: Docker 网络占用了公司网络 IP 段

**原因：** Docker 默认网段（172.17.0.0/16 等）与公司网络冲突。

**解决：**

```json
// /etc/docker/daemon.json
{
  "bip": "10.88.0.1/16",
  "default-address-pools": [
    {"base": "10.89.0.0/16", "size": 24},
    {"base": "10.90.0.0/16", "size": 24}
  ]
}
```

```bash
sudo systemctl restart docker
```

### Q6: Macvlan 容器之间或与宿主机无法通信

**原因：** Macvlan 默认隔离宿主机与容器。

**解决：**

```bash
# 方式1：创建 macvlan 子接口
ip link add macvlan0 link eth0 type macvlan mode bridge
ip addr add 192.168.1.10/24 dev macvlan0
ip link set macvlan0 up

# 方式2：使用 IPvlan 替代
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  ipvlan-net
```

### Q7: 容器频繁断连

**原因：** 系统连接跟踪表满了、ARP 表问题。

**解决：**

```bash
# 检查 conntrack 表
sudo sysctl net.netfilter.nf_conntrack_max
sudo sysctl net.netfilter.nf_conntrack_count

# 增大连接跟踪表
echo "net.netfilter.nf_conntrack_max=655360" >> /etc/sysctl.conf
sysctl -p

# 调整 ARP 缓存
echo "net.ipv4.neigh.default.gc_thresh1=2048" >> /etc/sysctl.conf
echo "net.ipv4.neigh.default.gc_thresh2=4096" >> /etc/sysctl.conf
echo "net.ipv4.neigh.default.gc_thresh3=8192" >> /etc/sysctl.conf
sysctl -p
```

---

<a name="english"></a>

# English Documentation

## 📖 Table of Contents

- [1. Overview](#1-overview-1)
- [2. Docker Network Basics](#2-docker-network-basics)
- [3. Network Modes Deep Dive](#3-network-modes-deep-dive)
  - [3.1 Bridge Mode](#31-bridge-mode)
  - [3.2 Host Mode](#32-host-mode)
  - [3.3 None Mode](#33-none-mode)
  - [3.4 Container Mode](#34-container-mode)
  - [3.5 Overlay Mode](#35-overlay-mode)
  - [3.6 Macvlan Mode](#36-macvlan-mode)
  - [3.7 IPvlan Mode](#37-ipvlan-mode)
- [4. Custom Networks](#4-custom-networks)
- [5. Port Mapping](#5-port-mapping)
- [6. DNS Configuration](#6-dns-configuration)
- [7. Network Security](#7-network-security)
- [8. Docker Compose Networking](#8-docker-compose-networking)
- [9. Multi-Host Networking (Swarm/Overlay)](#9-multi-host-networking-swarmoverlay)
- [10. Network Troubleshooting](#10-network-troubleshooting)
- [11. Common Commands](#11-common-commands-1)
- [12. FAQ](#12-faq-1)

---

## 1. Overview

Docker networking is a core component of containerized applications. It determines:

| Feature | Description |
|---------|-------------|
| Container Connectivity | How containers communicate with each other |
| External Access | How external clients access container services |
| Network Isolation | Isolation strategies between container groups |
| Service Discovery | How containers discover each other by name |
| Load Balancing | How requests are distributed across container instances |

**Core Concepts:**
- **Network Namespace** — Each container has an isolated network stack
- **veth pair** — Virtual Ethernet pairs connecting containers to the host
- **iptables** — Docker uses iptables for NAT and port mapping
- **DNS** — Built-in DNS for container name resolution

## 2. Docker Network Basics

### Installing Docker

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo docker --version
```

### Viewing Docker Networks

```bash
docker network ls
```

**Default Networks Comparison:**

| Network | Driver | Description |
|---------|--------|-------------|
| bridge | bridge | Default NAT mode |
| host | host | Shares host network stack |
| none | null | No networking |

## 3. Network Modes Deep Dive

### 3.1 Bridge Mode

**Default mode.** Docker creates a virtual bridge `docker0`, and containers connect to it via veth pairs.

```
Host
┌─────────────────────────────────────┐
│  ┌──────────────┐                    │
│  │   docker0    │ 172.17.0.1/16     │
│  │  (bridge)    │                    │
│  └──┬───┬───┬──┘                    │
│     │   │   │  veth pairs           │
│  ┌──┘ ┌─┘ ┌─┘                       │
│  │    │    │                         │
│ ┌┴┐  ┌┴┐  ┌┴┐                       │
│ │C1│ │C2│ │C3│  172.17.0.x         │
│ └─┘  └─┘  └─┘                       │
│         │                            │
│     eth0 │ NAT -> External          │
└─────────┼──────────────────────────┘
          │
     Internet
```

**How it works:**
1. Docker creates `docker0` bridge (default 172.17.0.0/16)
2. Each container gets a veth pair
3. One end in container (eth0), one on host
4. Host end connects to `docker0`
5. External access via NAT (iptables MASQUERADE)

**Default vs Custom Bridge:**

| Feature | Default Bridge | Custom Bridge |
|---------|---------------|---------------|
| DNS Resolution | ❌ Needs --link | ✅ Automatic DNS |
| Isolation | ❌ All containers | ✅ Configurable |
| Runtime Changes | ❌ Recreate needed | ✅ Hot plug/unplug |

### 3.2 Host Mode

Container shares the host's network stack.

```bash
docker run -d --name web --network host nginx
```

**Pros/Cons:**
- ✅ Best performance (no NAT overhead)
- ✅ Automatic port exposure
- ❌ Port conflicts possible
- ❌ Poor network isolation
- ❌ Linux only

### 3.3 None Mode

No network interface for the container.

```bash
docker run -d --name isolated --network none alpine sleep infinity
```

### 3.4 Container Mode

Shares another container's network stack.

```bash
docker run -d --name base --network bridge nginx
docker run -d --name client --network container:base alpine sleep infinity
```

### 3.5 Overlay Mode

Multi-host networking for Docker Swarm clusters.

```bash
docker swarm init
docker network create -d overlay --attachable my-overlay
docker service create --name web --network my-overlay --replicas 3 nginx
```

### 3.6 Macvlan Mode

Assigns MAC addresses and IPs from the same subnet as the host.

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  macvlan-net
```

### 3.7 IPvlan Mode

Similar to Macvlan but shares a single MAC address.

```bash
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  -o ipvlan_mode=l2 \
  ipvlan-l2
```

**Network Mode Comparison:**

| Mode | Isolation | Performance | Multi-Host | Use Case |
|------|-----------|-------------|------------|----------|
| Bridge | Medium | Medium | ❌ | Default, single host |
| Host | Low | High | ❌ | Performance-critical |
| None | High | - | ❌ | Offline/security |
| Container | - | - | ❌ | Sidecar pattern |
| Overlay | Medium | Low | ✅ | Swarm clusters |
| Macvlan | Low | High | ❌ | Legacy migration |
| IPvlan | Low | High | ❌ | Large scale |

## 4. Custom Networks

```bash
docker network create --driver bridge \
  --subnet=10.0.0.0/24 \
  --gateway=10.0.0.1 \
  my-network

docker run -d --name app1 --network my-network nginx
docker network connect my-network app1
docker exec app1 ping app1  # Automatic DNS resolution
```

## 5. Port Mapping

```bash
docker run -d -p 8080:80 nginx
docker run -d -p 127.0.0.1:8080:80 nginx
docker run -d -P nginx  # Random ports
docker port container_name
```

## 6. DNS Configuration

```bash
docker run -d --name web \
  --dns 8.8.8.8 \
  --dns 1.1.1.1 \
  nginx

# Custom network provides automatic DNS
docker network create my-net
docker run -d --name api --network my-net myapi
docker run -d --name web --network my-net nginx
# web can reach api via hostname "api"
```

## 7. Network Security

```bash
# Internal network (no external access)
docker network create --internal internal-net

# Network isolation via multiple networks
docker network create frontend-net
docker network create backend-net
docker network create database-net
```

## 8. Docker Compose Networking

```yaml
version: '3.8'
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    networks:
      - frontend
      - backend

  api:
    image: myapp:latest
    networks:
      - backend

  database:
    image: postgres:15
    networks:
      - internal

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
  internal:
    driver: bridge
    internal: true
```

## 9. Multi-Host Networking (Swarm/Overlay)

```bash
docker swarm init --advertise-addr 192.168.1.10
docker network create -d overlay --attachable my-overlay
docker service create --name web --replicas 3 --network my-overlay nginx
```

## 10. Network Troubleshooting

```bash
docker exec container ip addr
docker exec container ip route
docker exec container ping <target>
docker network inspect <network>
sudo tcpdump -i docker0 -n

# Use netshoot toolkit
docker run -it --rm --network host nicolaka/netshoot
```

## 11. Common Commands

```bash
docker network ls                    # List networks
docker network inspect <name>        # Inspect network
docker network create <name>         # Create network
docker network rm <name>             # Remove network
docker network prune                 # Clean unused networks
docker network connect <net> <ctr>   # Connect container
docker network disconnect <net> <ctr>  # Disconnect container
docker port <container>              # Show port mappings
```

## 12. FAQ

### Q1: Container cannot access external network
**Fix:** Enable IP forwarding, check iptables, verify DNS.
```bash
sysctl net.ipv4.ip_forward
sudo iptables -t nat -L POSTROUTING -n -v
```

### Q2: Container-to-container communication fails
**Fix:** Ensure same network, test connectivity.
```bash
docker network connect my-network container-b
docker exec container-a ping container-b
```

### Q3: Port mapping not working
**Fix:** Check port bindings, firewall, and address binding.
```bash
sudo netstat -tlnp | grep <port>
docker port container_name
```

### Q4: Overlay network performance issues
**Fix:** Adjust MTU, avoid unnecessary encryption.
```bash
docker network create -d overlay -o "com.docker.network.driver.mtu=1450" overlay-mtu
```

---

## ☕ 支持 / Support

如果这个教程对你有帮助，欢迎请我喝杯咖啡：

**USDT (TRC20)**
```
TVbQerV1SF4MXB1JCcAzQxarewHwEPYTKm
```
