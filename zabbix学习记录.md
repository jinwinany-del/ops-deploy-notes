# Zabbix 学习记录（面试导向 · 7 天计划）

> 承接 Week3/Week4：阿里云轻量（Ubuntu 24.04，2 核 2G，公网 IP `203.0.113.10`）上已有
> 网站（:80）+ Grafana（host 3000）+ Prometheus（127.0.0.1:9090）+ node_exporter（127.0.0.1:9100）+ gitea（3001/2222）。
> 本系列目标：在同一台机器上再部署一套 **Zabbix**，掌握企业监控的另一条技术路线，并与 Prometheus 形成对比。

---

## 计划总览

| 天 | 主题 | 核心产出 |
|----|------|---------|
| Day 1 | 架构概念 + Server 部署 | Zabbix Web 公网可登录 |
| Day 2 | Agent + Item | 监控本机 CPU/内存/磁盘 |
| Day 3 | Trigger + Action | 告警能发到手机 |
| Day 4 | Template | 用官方模板批量监控 |
| Day 5 | 自动发现 + 自动注册 | 免手动接入主机 |
| Day 6 | Proxy + API | 理解分布式与二次开发 |
| Day 7 | 实战 + 面试复盘 | 监控 MySQL/Nginx，梳理高频题 |

---

# Day 1（2026-11-07）：架构概念 + Zabbix Server 部署

> **修订说明（2026-11-10）**：8080 已从 `"8080:8080"`（绑 `0.0.0.0`、对公网发布）改为只绑 `172.17.0.1:8080`（docker0 网关），公网不可达，浏览器访问统一走 SSH 隧道。下文相关段落已同步更正。

## 一、目标

用 Docker Compose 部署 **Zabbix 7.0 LTS**（Server + PostgreSQL + Web 前端），实现：

1. 通过 **SSH 隧道** 在浏览器登录 Zabbix 管理界面（8080 只绑 `172.17.0.1`，不对公网开放）
2. 数据库与 Server 端口**不对公网暴露**（最小暴露面）
3. 给每个容器设内存上限，避免重演 10-20 整机 OOM
4. 理解 Zabbix 五大组件的关系，能对着架构图讲清楚

## 二、版本选择

- **Zabbix 7.0 LTS**（2025-05 发布，支持至 2030）—— 学习与面试都用 LTS，不用中间版本 7.4 或未发布的 8.0
- Docker 镜像用 **`alpine-7.0-latest` 变体**：比 `ubuntu` 变体小很多，2G 小机器优先
- 数据库选 **PostgreSQL 15**：Zabbix 官方推荐，内存占用比 MySQL 更可控

## 三、架构决策：为什么这次用 bridge 网络，而不是 Week4 的 host 网络

```
浏览器 ──SSH隧道──▶ 172.17.0.1:8080 ──▶ [zabbix-web]     ← 只绑 docker0 网关（非公网）
                              │ 内部网络 zbx-net
                              ▼
                         [zabbix-server]                  ← 10051 只绑 127.0.0.1
                              │ 内部网络 zbx-net
                              ▼
                         [zabbix-postgres]                ← 一个端口都不发布
```

**理由**：

1. **数据库和 Server 根本不需要任何对外端口**。用内部网络 + 服务名互访，等于从设计上就不存在暴露面——比 host 网络下靠绑 `127.0.0.1` 收口更彻底。
2. Week4 选 host 网络是为了让 **ufw 规则生效**（端口发布走 DNAT/FORWARD 链会绕过 ufw 的 INPUT 链）。这次把 8080 绑到 **`172.17.0.1`（docker0 网关）**，从根上就不对公网发布，访问走 SSH 隧道——**不再依赖云安全组**。
3. 换个收口方式：放弃"用 ufw 挡住发布端口"的可能性 → 改为绑 `172.17.0.1`，公网到不了这个端口，访问时只需建 SSH 隧道。这是明确的取舍，不是疏忽。

## 四、端口规划

| 端口 | 服务 | 监听范围 | 是否公网 |
|---|---|---|---|
| 8080 | Zabbix Web | `172.17.0.1` | **否**（需 SSH 隧道；安全组无需放行） |
| 10051 | Zabbix Server | `127.0.0.1` | 否（Day 2 本机 Agent 用） |
| 5432 | PostgreSQL | 仅容器网络 | 否（不发布） |
| 10050 | Zabbix Agent | 待 Day 2 安装 | 否 |

## 五、文件清单

```
/root/zabbix/docker-compose.yml        项目名 zabbix
数据卷：zabbix_pg-data → 容器内 /var/lib/postgresql/data
```

```yaml
name: zabbix

services:
  postgres:
    image: postgres:15-alpine
    container_name: zabbix-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=zabbix
      - POSTGRES_PASSWORD=改成你的强密码
      - POSTGRES_DB=zabbix
    volumes:
      - pg-data:/var/lib/postgresql/data
    networks:
      - zbx-net
    mem_limit: 512m            # 关键！256m 会导致首次建表时被 OOM 杀掉（见踩坑）

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:alpine-7.0-latest
    container_name: zabbix-server
    restart: unless-stopped
    environment:
      - DB_SERVER_HOST=postgres
      - POSTGRES_USER=zabbix
      - POSTGRES_PASSWORD=改成你的强密码
      - POSTGRES_DB=zabbix
    ports:
      - "127.0.0.1:10051:10051"
    networks:
      - zbx-net
    depends_on:
      - postgres
    mem_limit: 384m

  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:alpine-7.0-latest
    container_name: zabbix-web
    restart: unless-stopped
    environment:
      - ZBX_SERVER_HOST=zabbix-server
      - DB_SERVER_HOST=postgres
      - POSTGRES_USER=zabbix
      - POSTGRES_PASSWORD=改成你的强密码
      - POSTGRES_DB=zabbix
      - PHP_TZ=Asia/Shanghai        # 不设界面时间差 8 小时
    ports:
      - "172.17.0.1:8080:8080"      # 只绑 docker0 网关，公网不可达，走 SSH 隧道
    networks:
      - zbx-net
    depends_on:
      - zabbix-server
    mem_limit: 256m

volumes:
  pg-data:

networks:
  zbx-net:
```

**关键字段说明**：

- `POSTGRES_USER/PASSWORD/DB` 三项**必须三处一致**（postgres 段、server 段、web 段），Zabbix 用这组凭据连库；改密码要一起改
- `DB_SERVER_HOST=postgres`：容器 DNS 把**服务名**解析成 IP，所以填服务名而不是 IP
- `"127.0.0.1:10051:10051"`：格式 `宿主机IP:宿主机端口:容器端口`；前缀 `127.0.0.1` 保证公网连不上
- `"172.17.0.1:8080:8080"`：只绑 docker0 网关。**不写 IP 前缀（即 `"8080:8080"`）等于绑 `0.0.0.0`，对全网监听**——两者差别就是"要不要开安全组"
- `PHP_TZ=Asia/Shanghai`：Web 容器时区，影响界面所有时间显示
- `mem_limit`：容器内存硬上限，超出时 cgroup 只杀这个容器，不拖整机

## 六、实施步骤

### 第 0 步：资源与端口体检

```bash
free -h
df -h /
ss -tlnp | grep -E ':(80|3000|8080|10050|10051|9090|9100)'
docker ps
```

逐行解释：

- `free -h`：`-h` 人类可读。**只看 `available`**——它才是真实可用内存，`used` 含可回收缓存不准
- `df -h /`：只查根分区。Docker 数据都在 `/var/lib/docker`，属根分区。**不要不加 `/`**，否则会列出一堆 overlay 挂载点
- `ss -tlnp | grep -E`：`-t` 仅 TCP、`-l` 仅监听、`-n` 不做域名解析、`-p` 显示进程；`-E` 扩展正则，`|` 表示或。**无输出=端口空闲**
- `docker ps`：整机视角列所有运行容器（`docker compose ps` 只列当前目录项目）

### 第 1 步：建目录、写 compose、语法校验

```bash
mkdir -p /root/zabbix
cd /root/zabbix
# 粘贴 compose 内容（heredoc：cat > docker-compose.yml <<'EOF' ... EOF）
cat docker-compose.yml
docker compose config
```

- `cat > 文件 <<'EOF'`：heredoc 整块写入；**`'EOF'` 的引号不能去掉**，否则内容里的 `$变量` 会被 shell 提前展开
- `cat 文件`：写完**必须回读**——YAML 缩进错一个空格要等容器起来才报错
- `docker compose config`：解析并规范化打印，不启动任何东西；**不报错 = 语法合法**，是免费的部署前体检

### 第 2 步：拉镜像（踩了网络坑）

```bash
docker compose pull
```

**现象**：`zabbix-web` 和 `postgres` 各有一个层卡在 1MB/24MB，耗时 600 秒+。

**判读**：不是卡死，是**个别层断流**。`zabbix-server` 78 秒就拉完，说明加速器整体通，只是某些层慢。

**处理**：`Ctrl+C` → 重跑 `docker compose pull`。Docker 按层断点续传，**已完成的层不会重下**，只重试停滞的层。必要时重试 2~3 次，或改为单镜像拉取（`docker pull postgres:15-alpine`）缩小范围。

**验证镜像齐了**：

```bash
docker images | grep -E 'zabbix|postgres'
```

### 第 3 步：启动与验证

```bash
docker compose up -d
docker compose ps
docker compose logs --tail=20 zabbix-server
ss -tlnp | grep -E ':(8080|10051)'
curl -I http://172.17.0.1:8080      # 8080 绑在 docker0，不是 127.0.0.1
```

期望：

- 三容器 `Up`；`zabbix-web` 显示 `(healthy)`
- `ss`：**`172.17.0.1:8080`** + **`127.0.0.1:10051`**（两者都必须带具体 IP 前缀；若出现 `0.0.0.0:8080`，说明 compose 里的 IP 前缀没生效）
- `curl -I`：`200 OK` 或 `302 Found`

### 第 4 步：建 SSH 隧道（不需要放行安全组）

8080 只监听在 `172.17.0.1`，公网本来就连不上，所以**阿里云安全组不需要放行 8080**（放行了也没用，因为没在公网网卡上监听）。

本地执行：

```bash
ssh -L 8080:172.17.0.1:8080 root@203.0.113.10
```

- `-L 本地端口:目标地址:目标端口`：把**本机**的 8080 转发到**服务器上**的 `172.17.0.1:8080`
- 目标地址写 `172.17.0.1`（服务器视角的 docker0 网关），这是 8080 实际监听的地址
- 隧道建好后，本地浏览器访问 `http://127.0.0.1:8080`

**为什么 ufw 也不用动**：8080 没绑 `0.0.0.0`，外部流量连 SYN 都到不了这个 socket，ufw 放行与否已无意义。

### 第 5 步：登录

本地浏览器 `http://127.0.0.1:8080`（走隧道），账号 `Admin` / 密码 `zabbix`（7.0 默认值）。

---

## 七、踩坑（本次最有价值的一条）

### 容器内存上限太小 → 数据库初始化中断 → Server 无限重启

**现象**

`docker compose ps` 显示 `zabbix-server   Restarting (1) 40 seconds ago`，每约 60 秒重启一次，持续十几分钟。

**排查过程**

1. 先看 server 日志末尾 → 只有配置准备阶段（`Updating /etc/zabbix/zabbix_server.conf parameter ...`），看不到实际报错
2. 改抓关键词：`docker compose logs zabbix-server 2>&1 | grep -iE 'error|fail|fatal|cannot|denied' | tail -20`

```bash
docker compose logs zabbix-server 2>&1 | grep -iE 'error|fail|fatal|cannot|denied' | tail -20
```

- 不加 `--tail`：错误可能出现在任意一次重启循环里，要全文搜
- `2>&1`：把 stderr 合并进 stdout，否则 `grep` 接不到错误输出
- `-iE`：忽略大小写 + 扩展正则（多关键词用 `|` 或）

得到真报错：

```
cannot use database "zabbix": its "users" table is empty (is this the Zabbix proxy database?)
```

3. 排除内存嫌疑：

```bash
dmesg -T | grep -i oom | tail -3
```

- `dmesg -T`：内核日志，`-T` 转可读时间
- 命中关键行：

```
oom-kill:constraint=CONSTRAINT_MEMCG ... task=postgres,pid=423862
Memory cgroup out of memory: Killed process 423862 (postgres) anon-rss:223664kB
```

**根因（完整因果链）**

```
postgres mem_limit 设为 256m，太紧
  ↓ Zabbix 首次启动要导入几百张表的初始化数据（一次性内存高峰）
  ↓ postgres 冲到 223MB 时撞上 256m 上限
  ↓ cgroup OOM Killer 杀掉 postgres（CONSTRAINT_MEMCG = 容器级，非整机）
  ↓ 建表中断 → users 表为空 → 数据库"半身不遂"
  ↓ zabbix-server 每次启动都检测到 users 表为空 → 报错退出（退出码 1）
  ↓ restart: unless-stopped → 无限重启
```

**为什么重启救不回来**：Zabbix 只在"**空数据库**"时自动初始化。库存在但残缺，既不重建、也起不来，锁死。

**修复**

```bash
docker compose down
docker volume rm zabbix_pg-data
sed -i '/postgres:15-alpine/,/mem_limit/ s/256m/512m/' docker-compose.yml
grep mem_limit docker-compose.yml
docker compose up -d
```

逐行解释：

- `docker compose down`：停并删本项目容器，**数据卷默认保留**（所以还需下一步）
- `docker volume rm zabbix_pg-data`：删掉半成品库。**此步安全**——里面只有建了一半的表，无真实数据
- `sed -i '/postgres:15-alpine/,/mem_limit/ s/256m/512m/' ...`：`/起始/,/结束/` 是**行范围**，只改 postgres 段，不会误伤 web 段同样存在的 `256m`
- `grep mem_limit`：**改完必须验证**——预期 `512m`(postgres) / `384m`(server) / `256m`(web)

**修复后验证**（等 3 分钟让初始化跑完）

```bash
docker compose ps
docker compose logs --tail=15 zabbix-server
dmesg -T | grep -i oom | tail -3
```

- server 日志末尾出现 `executing housekeeper` → **健康铁证**（housekeeper 是清理过期历史的内部进程，能干活说明调度循环正常）
- dmesg 只剩修复前的旧记录，**无新增 OOM**

---

## 八、实测数据（2026-11-07）

### 资源

```
free -h: total 1.6Gi / available 483Mi / Swap 0B used
df -h /: 40G 总量, 8.8G 已用, 29G 可用 (24%)
```

### 端口占用（部署后）

| 端口 | 进程 | 说明 |
|---|---|---|
| 8080 | docker-proxy | Zabbix Web，**仅 172.17.0.1**（SSH 隧道访问） |
| 10051 | zabbix server | **仅 127.0.0.1** |
| 80 | docker-proxy | 原有网站 |
| 3000 | grafana | host 网络 |
| 9090 / 9100 | prometheus / node_exporter | 仅 127.0.0.1 |

### 容器内存（docker stats --no-stream）

| 容器 | MEM USAGE / LIMIT | 占比 |
|---|---|---|
| zabbix-web | 51.89MiB / 256MiB | 20% |
| zabbix-server | 56.41MiB / 384MiB | 15% |
| zabbix-postgres | 150.7MiB / 512MiB | 29% |
| grafana | 278.1MiB / 512MiB | 54% |
| gitea | 102.2MiB / 1.575GiB | 6%（**无上限，风险项**） |
| prometheus | 59.86MiB / 512MiB | 12% |
| app-web-1 | 4.57MiB / 128MiB | 4% |
| app-cache-1 | 2.06MiB / 128MiB | 2% |
| node-exporter | 17.55MiB / 64MiB | 27% |

- Zabbix 三容器合计约 **259MiB**，远低于上限
- 全机容器合计约 **724MiB**

### 日志中的"看似错误"项（均为正常）

| 日志 | 含义 |
|---|---|
| `item "zabbix[process,ipmi poller,avg,busy]" became not supported` | 未启用 IPMI/SNMP/Java/connector 等组件，对应进程本就不该起；Zabbix 自监控主机在检查它们 |
| `Zabbix agent item "system.cpu.util[,nice]" failed: another network error` | **Day 2 的内容**：本机还没装 Agent，10050 无监听 → 采集失败。装完 Agent 自动消失 |
| `temporarily disabling Zabbix agent checks: interface unavailable` | 同上，连续失败后临时禁用该主机采集 |

---

## 九、面试可讲点

**概念层**

- **Zabbix vs Prometheus 的分工**：Zabbix 是"全家桶"——采集、存储、告警、可视化、自动发现全内置，Web UI 驱动配置；Prometheus 是"积木式"——采集靠 exporter、告警靠 Alertmanager、可视化靠 Grafana。前者适合传统运维与混合环境（SNMP/Windows/IPMI 全覆盖），后者适合云原生与自定义指标。
- **Zabbix 五大组件**：Server（大脑，调度采集与算触发器）、Database（全部配置与历史数据）、Web Frontend（PHP 管理界面）、Agent（被监控端，分**主动/被动**两种模式）、Proxy（分布式场景的中继，减轻 Server 压力）。
- **Agent 两种模式**：被动=Server 来拉（Agent 监听 10050），主动=Agent 推到 Server 10051。生产跨网段/防火墙密集场景常改用主动模式。
- **为什么用 PostgreSQL 而不是 MySQL**：官方推荐、默认配置下内存更可控；Zabbix 大量使用分区与索引，PG 在大表维护上更省心。

**工程层**

- **`mem_limit` 的真实语义**：上限不是预留。超限时 **cgroup 只杀该容器**（`CONSTRAINT_MEMCG`），整机其他服务不受影响——这是 OOM 事故后正确的加固姿势，而不是"给机器加 swap 硬扛"。
- **初始化为什么要留足内存**：Zabbix 首次启动导入几百张表，是一次性内存高峰，比稳态运行时高得多。**上限要按峰值定，不是按稳态定**。
- **数据库初始化的原子性问题**：初始化被中断后库处于"半身不遂"，程序既不会重初始化也起不来。处置方式只有"删库重来"——所以初始化阶段监控 `dmesg` 比看应用日志更早发现根因。
- **最强的暴露面控制是「不在公网网卡上监听」**：把发布端口再绑到 `172.17.0.1`（docker0 网关），比绑 `0.0.0.0` 后再靠安全组/防火墙挡更可靠——前者是「连不上」，后者是「挡得住」。本例的 8080 就是这样收口的。
- **端口发布 vs host 网络的暴露面取舍**：发布端口多一层 `docker-proxy` 且绕过 ufw；host 网络进程直接监听、ufw 生效。选择依据是"这个端口该不该对公网开"。

**方法论层**

- **日志分级定位**：`docker compose ps` 看状态 → `logs --tail` 看尾部 → 关键词 `grep` 搜全文 → 内核 `dmesg` 查 cgroup 事件。**应用日志说"表是空的"，内核日志才说"为什么空"**。
- **容器视角 vs 整机视角**：`docker compose ps` 只列当前目录项目，`docker ps` 列全机；排查"我其他服务还在吗"必须用后者。
- **报错行 ≠ 真错误**：postgres 启动时的 `FATAL: not yet accepting connections` 是正常过程，**要看最后一行状态**。

---

## 十、Day 2 预告

- 在宿主机上 `apt install zabbix-agent`（不用容器跑 Agent：容器里的 `/proc` 是它自己的，采不到宿主机真实指标）
- 配置 `Server=127.0.0.1` 对接本机 Server 的 10051
- 在 Web UI 里为 `Zabbix server` 主机配 Item：CPU / 内存 / 磁盘
- 目标：Day 1 日志里那些 `agent item failed` 红字自动消失，Latest data 出曲线

---

# Day 2（2026-11-10）：Zabbix Agent 接入与全链路打通

## 一、目标

1. 在宿主机安装 **zabbix-agent 7.0.32**（官方源），采到本机真实 CPU / 内存 / 磁盘
2. 搞清 **被动（10050）** 与 **主动（10051）** 两种采集模式的差别
3. Agent 端口**不对公网暴露**
4. 打通 `Zabbix Server 容器 → 宿主机 Agent` 这条跨命名空间链路

## 二、为什么 Agent 装宿主机，不装容器

Zabbix Agent 靠读 `/proc`、`/sys` 拿系统指标。**容器的 `/proc` 是容器自己的隔离视图**，只能看到容器内那几个进程——采出来的数据毫无意义。装在宿主机才读得到真机数据。

## 三、两种采集模式

| 模式 | 谁发起连接 | 端口 | 适用场景 |
|---|---|---|---|
| 被动 Passive | Server 主动连 Agent | Server → Agent **10050** | 默认；Server 能直达 Agent |
| 主动 Active | Agent 主动连 Server | Agent → Server **10051** | Agent 在 NAT 后 / 防火墙拦入站 |

两种都要配：官方模板里的 Item **两种都有**，少配一种会出现"一半 Item 有数据、一半没有"。

## 四、实施步骤

### 第 1 步：官方源安装

```bash
cd /tmp
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
apt update
apt install -y zabbix-agent zabbix-get
```

- `dpkg -i` 装的**不是 Agent 本体**，只是往 `/etc/apt/sources.list.d/` 写一个官方软件源
- `apt update` 不能省：不刷新索引，apt 不认识 `zabbix-agent` 这个包
- `zabbix-get` 是**测试工具**，用来模拟"Server 来拉数据"这个动作

版本确认：`zabbix_agentd --version` → **7.0.32（Zabbix SIA 官方源）**

### 第 2 步：四项核心配置

```bash
cp /etc/zabbix/zabbix_agentd.conf /etc/zabbix/zabbix_agentd.conf.bak
```

| 配置项 | 值 | 作用 |
|---|---|---|
| `Server=` | `127.0.0.1,172.20.0.0/16` | **被动模式白名单**（源 IP 匹配），不是"目标地址" |
| `ServerActive=` | `127.0.0.1:10051` | 主动模式推送目标 |
| `Hostname=` | `Zabbix server` | Agent 自报名，必须和 UI 主机名**完全一致** |
| `ListenIP=` | `127.0.0.1,172.20.0.1` | 监听地址；默认 `0.0.0.0` 等于对全网开放 |

### 第 3 步：启动与验证

```bash
systemctl enable --now zabbix-agent
systemctl status zabbix-agent --no-pager -l
ss -tlnp | grep 10050
zabbix_get -s 127.0.0.1 -k 'system.cpu.load[all,avg1]'
```

- `enable --now`：开机自启 + 立即启动
- `--no-pager`：不进 `less` 分页（否则脚本里卡住）
- `zabbix_get -s 目标 -p 端口 -k '键名'`：**完整模拟 Server 的被动采集**，返回数字即链路通

---

## 五、踩坑（本次共 6 个，按发现顺序）

### 坑 1：sed 去掉注释后，把"空值行"也激活了

**现象**：`grep -nE '^(Server|ServerActive|Hostname|ListenIP)='` 出现成对的行

```
115:Server=            ← 空值（原文件的 Default 占位）
117:Server=127.0.0.1   ← 真实值
171:ServerActive=
173:ServerActive=127.0.0.1:10051
```

**成因**：原配置文件里每项都有 `# Default:` 下的空占位行和 `# 示例值` 行，手动编辑时把**两行的 `#` 都去掉了**。

**为什么当时功能还正常**：Zabbix 的解析规则是「**同参数出现多次，最后一次生效**」。但这属于依赖隐式规则，`ServerActive=` 若生效为空会**静默禁用主动模式**，极难排查。

**修复**：

```bash
sed -i -E '/^(Server|ServerActive|Hostname)=$/d' /etc/zabbix/zabbix_agentd.conf
```

- `/^(...)=$/`：整行只有「键=」且等号后立即结束
- `d`：删除匹配行，**只删空值行，有值的不动**

### 坑 2：ListenIP 默认 0.0.0.0，Agent 对公网敞开

**现象**：`ss` 显示 `0.0.0.0:10050` + `[::]:10050`

**风险**：Zabbix Agent **没有任何认证机制**，谁能连上 10050 谁就能 `zabbix_get` 读你的进程列表、内存、磁盘、网络连接。此时只靠云安全组挡，是**单层防护**。

**修复**：`ListenIP=127.0.0.1,172.20.0.1` → 改完 `ss` 只剩两个具体地址，`0.0.0.0` 和 `[::]` 消失。

**顺带一个细节**：`ss` 里同一端口列了 **13 个 PID**——Agent 是「1 主进程 + 多个子进程」模式，**子进程共享同一个监听 socket**（同一个 `fd`），这是正常的，不是起了 13 个实例。

### 坑 3（核心）：容器里的 `127.0.0.1` 不是宿主机

**现象**：UI 仪表盘 `主机可用性 0 可用 / 1 不可用`，Current problems 显示 `Linux: Zabbix agent is not available (for 3m)`，而宿主机上 `zabbix_get -s 127.0.0.1` 明明是通的。

**根因**：

```
[zabbix-server 容器] ──去连 127.0.0.1:10050──▶ [它自己容器内部]  ← 这里没有 Agent
[宿主机]             127.0.0.1:10050          ← Agent 在这里，但容器看不见
```

- 每个容器有**独立网络命名空间**，容器内的 `127.0.0.1` 指向容器自己
- **端口发布是单向的**：`"127.0.0.1:10051:10051"` 只做「容器端口 → 宿主机端口」，**不会反向**让容器访问宿主机端口

**修复三步**：

```bash
# 1) 查 compose 网络的网关与网段
docker network inspect zabbix_zbx-net | grep -E '"Subnet"|"Gateway"'
#     → Subnet 172.20.0.0/16, Gateway 172.20.0.1

# 2) Agent 多监听一个网桥地址，并把容器网段加进白名单
sed -i 's|^ListenIP=.*|ListenIP=127.0.0.1,172.20.0.1|' /etc/zabbix/zabbix_agentd.conf
sed -i 's|^Server=.*|Server=127.0.0.1,172.20.0.0/16|' /etc/zabbix/zabbix_agentd.conf
systemctl restart zabbix-agent

# 3) UI：数据采集 → 主机 → Zabbix server → 接口 → Agent 行 IP 改成 172.20.0.1
```

- `s|旧|新|` 用 `|` 作分隔符，避免内容里的 `/16` 需要转义
- **`Server` 必须加容器网段**：容器发起连接时源 IP 是 `172.20.0.3`，不加会被拒绝（最容易漏的一步）
- 改完 `ss` 应看到**两个**监听：`127.0.0.1:10050` + `172.20.0.1:10050`

### 坑 4：ufw 拦截「容器 → 宿主机」的流量

**现象**：

```bash
docker exec zabbix-server zabbix_get -s 172.20.0.1 -k 'system.cpu.load[all,avg1]'
→ cannot connect to [[172.20.0.1]:10050]: connection timed out
```

**根因**：`ufw status verbose` 显示

```
Default: deny (incoming), allow (outgoing), deny (routed)
已放行：22/tcp, 80/tcp, 443/tcp   ← 没有 10050
```

容器发往 `172.20.0.1:10050` 的包落在网桥 `br-xxxxxxxxxxxx` 上，进 **INPUT 链** → 命中 `deny (incoming)` → 丢包。

**关键认知（两条链的差别）**：

| 流量类型 | 走的链 | ufw 是否生效 |
|---|---|---|
| 端口发布（如 8080 → docker-proxy） | **FORWARD / DNAT** | ❌ 不生效 |
| 容器访问宿主机服务（如 → 172.20.0.1:10050） | **INPUT** | ✅ 生效 |

同一个 ufw、两种流量走两条链——这解释了 Day 1 "8080 不受 ufw 影响"的现象，也解释了这里为什么被拦。（注：Day 1 的 8080 最初绑 `0.0.0.0`，后来改为绑 `172.17.0.1` 收口，正是因为「发布端口绕过 ufw」这个机制靠 ufw 挡不住。）

**修复**：

```bash
ufw allow from 172.20.0.0/16 to any port 10050 proto tcp
```

- **不要写 `ufw allow 10050`**（那会对全网开放）
- 立即生效，不需要 `ufw reload`
- 规则持久化在 `/etc/ufw/user.rules`，重启不丢（比裸 `iptables` 命令强）

### 坑 5：白名单里"看到的源 IP"和预期不一致

**现象**：Agent 日志

```
failed to accept an incoming connection: connection from "10.0.53.107" rejected,
allowed hosts: "127.0.0.1,172.20.0.0/16"
```

**排查**：`ip -4 addr show | grep -E '^[0-9]+:|inet '`

```
2: eth0            inet 10.0.53.107/18    ← 阿里云内网网卡
6: br-xxxxxxxxxxxx inet 172.20.0.1/16       ← zbx-net 网桥（网关）
5: docker0         inet 172.17.0.1/16  DOWN ← 老的默认网桥，未使用
```

**结论**：`10.0.53.107` 是**阿里云内网网卡地址**（公网 IP 是 NAT 映射的）。**宿主机自测连接 `172.20.0.1` 时，源地址被选成了 eth0 的内网 IP**，所以被白名单拒绝——这是"测试方式"的问题，不代表真实链路有问题。

**方法论**：**宿主机自测不能作为验收依据，必须用容器视角**

```bash
docker exec zabbix-server zabbix_get -s 172.20.0.1 -k 'system.cpu.load[all,avg1]'
→ 0.000000   ← 链路打通
```

### 坑 6：UI 里把 `172.20.0.1` 打成 `127.20.0.1`

**现象**：主机列表显示 `127.20.0.1:10050`，ZBX 图标仍为红色（不可用）。

**判读**：

| 地址 | 是什么 | 结果 |
|---|---|---|
| `172.20.0.1` | zbx-net 网桥网关，Agent 监听在此 | ✅ 通 |
| `127.20.0.1` | 落在 `127.0.0.0/8` 回环段，容器里指向容器自己 | ❌ 拒绝 |

**修复**：接口 IP 改回 `172.20.0.1`，端口 `10050` 不变。

---

## 六、验收结果

| 检查项 | 结果 |
|---|---|
| Agent 版本 | 7.0.32（官方源） |
| 监听地址 | `127.0.0.1:10050` + `172.20.0.1:10050`（无 `0.0.0.0`） |
| 容器视角采集 | `zabbix_get -s 172.20.0.1` → `0.000000` ✅ |
| 主机可用性 | `1 可用 / 0 不可用` |
| 触发器 | `Linux: Zabbix agent is not available` **自动解决**（2026-11-10 09:43:26，持续 2d 15h 35m） |
| 数据入库 | `Values processed per second` 从 1.08 涨到 ~10 vps |

**"自动解决"是比日志更硬的证据**：Zabbix 触发器是自愈的——只有数据真正回来，问题才会被标记为 resolved。

**可选的入库验证（绕开 UI）**：

```bash
docker exec zabbix-postgres psql -U zabbix -d zabbix -c "select count(*) from history_uint where clock > extract(epoch from now()) - 300;"
```

- `history_uint`：存无符号整数型历史值（CPU 使用率、字节数等都在这里）
- `clock > extract(epoch from now()) - 300`：只统计最近 5 分钟
- **返回值 > 0 且上百 = Agent 数据正在入库**

---

## 七、面试可讲点

**网络与隔离**

- **容器网络命名空间**：容器内的 `127.0.0.1` 指向容器自己；容器访问宿主机服务要用**网桥网关地址**（如 `172.20.0.1`），或用 `host.docker.internal` / host 网络
- **端口发布是单向的**：`-p 127.0.0.1:10051:10051` 只做容器→宿主机方向的映射，**不会反向**让容器访问宿主机端口
- **ufw 的两条链**：发布端口走 DNAT/FORWARD（ufw 不生效），容器→宿主机服务走 INPUT（ufw 生效）。这是排查"为什么端口明明放行了还是不通"的标准切入点

**安全**

- **Zabbix Agent 无认证**：`ListenIP` 必须收口到具体地址；确需远程监控时用 `Server=` 白名单放行**指定 Server IP**，而不是放行全网或开放安全组 10050
- **`Server` 是源 IP 白名单**，不是"目标地址"——这是与 `ServerActive`（目标地址）最容易混淆的一点
- **ufw 规则要限定源网段**：`ufw allow from 172.20.0.0/16 to any port 10050`，不要 `ufw allow 10050`

**故障排查方法论**

- **报错要看主体**：`zabbix_get` 在宿主机成功、UI 不可用，说明"测试主体和执行主体不在同一网络命名空间"
- **日志抓关键行**：`connection from "X" rejected, allowed hosts: "Y"` 一行就同时给出了「实际源 IP」和「白名单内容」，直接对比即可定位
- **不要用宿主机自测做验收**：本机访问自己的网桥地址时，源地址可能被选成别的网卡地址（本例是 eth0 内网 IP），结果不具备代表性。**验收要用真实调用方（容器）**
- **触发器自愈是恢复的硬证据**：问题自动 resolved 说明数据真的回来了

---

## 八、待办

- [ ] 给 **gitea** 补 `mem_limit`（当前无上限，是整机 OOM 的潜在风险）
- [ ] 默认密码 `Admin/zabbix` 必须首登即改；面板即使只经 SSH 隧道访问、不对公网开放，同机任意用户仍可直接登录
- [ ] 网桥网关 `172.20.0.1` 是 docker 动态分配的，若重建网络可能变化 → 可考虑在 compose 里用 `ipam` 固定子网
- [ ] Day 3 ~ Day 7 按计划推进，每天在本文件追加一节

---

# Day 3（2026-11-10）：Trigger + Action 告警链路全打通

## 一、目标

1. 建一个 **Trigger**：CPU 空闲率 < 20% 即告警
2. 建 **Media Type**（脚本类型）+ **Action**，把告警投递到本地脚本
3. 验证完整闭环：**造压 → PROBLEM 告警 → 恢复 → OK 告警**
4. 搞清 Zabbix 告警链路每一环的职责（Trigger → Event → Escalation → Alerter → Media Type → 脚本）

## 二、告警链路（必须能画出来）

```
Item 采到值
   ↓
Trigger 表达式求值 → 状态翻转(OK↔PROBLEM)
   ↓
event 表写入事件（value=1 表示 PROBLEM，value=0 表示恢复）
   ↓
escalator 进程：匹配 Action 条件 → 建 escalation → 执行 operation
   ↓
add_message_alert()：逐个用户 × media 创建 alert 记录（alerts 表）
   ↓
alert manager / alerter：取 status=NEW 的 alert → 调用 Media Type
   ↓
脚本类型 Media Type：执行 /usr/lib/zabbix/alertscripts/xxx.sh $1 $2 $3
```

**关键进程分工**（`ps aux` 能直接看到）：

| 进程 | 职责 |
|---|---|
| `escalator` | 匹配 Action、推进升级步骤、**创建 alert 记录** |
| `alert manager` | 调度 alerter、维护告警队列 |
| `alerter` | 实际执行 Media Type（发邮件/跑脚本） |
| `alert syncer` | 把发送结果回写数据库 |

**排查时看哪个进程，就决定了你在链路哪一环找问题。**

## 三、本次配置清单

| 对象 | 名称 | 关键配置 |
|---|---|---|
| Trigger | `CPU idle < 20% (Day3 测试)` | `last(/Zabbix server/system.cpu.util[,idle])<20`，严重性 Warning |
| Media Type | `demo script` | 类型=脚本，脚本名 `demo_alert.sh`，参数 3 个：`{ALERT.SENDTO}` / `{ALERT.SUBJECT}` / `{ALERT.MESSAGE}` |
| Action | `Day3 Demo Action` | 事件来源=触发器，条件=触发器严重性 >= 警告，操作=发给 Zabbix administrators + demo script |
| 用户 Media | Admin 的 demo script | **启用状态**，severity 全选，period `1-7,00:00-24:00` |

告警脚本 `/usr/lib/zabbix/alertscripts/demo_alert.sh`：接收 `$1/$2/$3`，追加写入同目录 `alert.log`。

## 四、踩坑（本次 5 个，第 1 个是核心）

### 坑 1（核心）：`media.active` / `media_type.status` 语义是反的 —— 0=启用，1=禁用

**现象**：链路全通、Trigger 正常翻转、escalation 正常创建，但 `alerts` 表始终为空、脚本一次都没执行、`alert.log` 空。

**排查**：开 `DebugLevel=4` 后抓到决定性日志：

```
In add_message_alert()
severity:2, media severity:63, period:'1-7,00:00-24:00', userid:1
will not send message (user media disabled)      ← 媒体被判定为"禁用"
```

把 `media.active` 改成 1 后，报错变成：

```
Media type disabled.                            ← 媒体类型被判定为"禁用"
```

**根因**：Zabbix 这两个字段的取值含义与直觉相反：

| 字段 | 0 | 1 |
|---|---|---|
| `media.active` | **启用** | **禁用** |
| `media_type.status` | **启用** | **禁用** |

我按"1=开"的直觉把两处都设成了 1，等于**把媒体和媒体类型全禁用了**。Zabbix 对此**静默跳过**：不建 alert 记录、不报失败、日志默认级别也看不到——所以从表面看像"Action 没生效"。

**修复**（用 API，别直接改 SQL，见坑 3）：

```bash
# 取 token
TOKEN=$(curl -s -X POST -H 'Content-Type: application/json-rpc' \
  -d '{"jsonrpc":"2.0","method":"user.login","params":{"username":"Admin","password":"zabbix"},"id":1}' \
  http://172.17.0.1:8080/api_jsonrpc.php | sed -n 's/.*"result":"\([^"]*\)".*/\1/p')

# 启用媒体类型（status=0）
curl -s -X POST -H 'Content-Type: application/json-rpc' \
  -d "{\"jsonrpc\":\"2.0\",\"method\":\"mediatype.update\",\"params\":{\"mediatypeid\":\"72\",\"status\":0},\"auth\":\"$TOKEN\",\"id\":2}" \
  http://172.17.0.1:8080/api_jsonrpc.php

# 启用用户媒体（active=0）
curl -s -X POST -H 'Content-Type: application/json-rpc' \
  -d "{\"jsonrpc\":\"2.0\",\"method\":\"user.update\",\"params\":{\"userid\":\"1\",\"medias\":[{\"mediatypeid\":\"72\",\"sendto\":\"demo-alert\",\"active\":0,\"severity\":63,\"period\":\"1-7,00:00-24:00\"}]},\"auth\":\"$TOKEN\",\"id\":3}" \
  http://172.17.0.1:8080/api_jsonrpc.php
```

**划重点**：UI 里"已启用"的复选框对应 `0`，不是 `1`。以后凡是看到 `status`/`active` 这类字段，**先查 Zabbix 文档确认取值**，别按直觉写。

### 坑 2：Docker 版 Zabbix 不会自动挂载 alertscripts 目录

自定义脚本要能被 Server 容器执行，必须显式挂载：

```yaml
volumes:
  - /usr/lib/zabbix/alertscripts:/usr/lib/zabbix/alertscripts
```

否则脚本在宿主机存在，容器里却找不到。

**权限细节**：容器内 Zabbix 以 `zabbix` 用户（uid 894）运行脚本。日志文件要让容器能追加写：

```bash
chmod 777 /usr/lib/zabbix/alertscripts          # 目录
chmod 666 /usr/lib/zabbix/alertscripts/alert.log # 日志文件
```

脚本本身 `755` 即可（有执行位）。

### 坑 3：直接改数据库不会刷新 configuration cache

**现象**：直接 `UPDATE media_type SET status=1`、`UPDATE media SET active=1`，DB 里改成功了，重启 Server 也没用，Server 仍按旧值判断。

**根因**：Zabbix Server 把配置（actions / operations / media / media_type / users 等）加载进内存缓存，靠 `changelog` 表做增量同步。**直接用 SQL 改表不会写 changelog**，追溯不到变更。

**正确姿势**：
- 走 **Web UI** 或 **API**（两者都会写 changelog），缓存自动刷新
- 临时排查可以用 SQL，但改完必须重启 Server 强制全量重载
- 想长期可靠 → 一律用 API/UI

### 坑 4：`DebugLevel` 改配置文件不生效

**现象**：`sed` 改了 `/etc/zabbix/zabbix_server.conf` 里的 `DebugLevel=4`，重启后日志还是 INFO 级别。

**根因**：官方镜像的 `docker-entrypoint.sh` **每次启动都重写配置文件**，手工改动被覆盖。日志第一屏全是：

```
** Updating /etc/zabbix/zabbix_server.conf parameter '...' ... removed/exists
```

**正确姿势**：用环境变量，entrypoint 会把它写进配置：

```yaml
environment:
  - ZBX_DEBUGLEVEL=4
```

排查完记得删掉，否则 debug 日志量极大（几分钟几十 MB）。

### 坑 5：`system.cpu.util[,idle]` 采集间隔 1 分钟，造压后别急着看

**现象**：`yes > /dev/null` 造压 50 秒，Trigger 还是 OK，以为告警坏了。

**根因**：`system.cpu.util[,idle]` 的 `delay=1m`，CPU 数据 1 分钟才采一次。造压后要等 **1~2 分钟** Trigger 才会翻转。

**验证采集是否正常**：

```bash
docker exec zabbix-postgres psql -U zabbix -d zabbix -c \
"select itemid,key_,delay from items where key_='system.cpu.util[,idle]';"
```

## 五、诊断方法论（本次最有价值的沉淀）

按链路从后往前查，比从 Trigger 往下猜快得多：

**第 1 步：看 `alerts` 表的 error 字段（信息量最大）**

```bash
docker exec zabbix-postgres psql -U zabbix -d zabbix -c \
"select alertid,actionid,mediatypeid,userid,eventid,status,retries,error from alerts order by alertid desc limit 10;"
```

- `status`：0=待发 1=已发 2=失败 3=新建
- `error` 直接告诉你卡在哪：`No media defined for user.` / `Media type disabled.` / `...`

**第 2 步：看 `escalations` 表（Action 是否匹配 + 升级是否在推进）**

```bash
docker exec zabbix-postgres psql -U zabbix -d zabbix -c \
"select escalationid,actionid,triggerid,eventid,esc_step,status,nextcheck from escalations;"
```

- 有记录 = Action 匹配成功
- `nextcheck` 在未来 = 还没到处理时间

**第 3 步：开 `DebugLevel=4` 抓 `add_message_alert`**

```bash
docker logs zabbix-server 2>&1 | grep -A5 'In add_message_alert()' | head -30
```

这一步能直接看到 `will not send message (xxx)` 的**具体原因**，是定位"静默丢弃"的关键。

**第 4 步：确认脚本本身能跑**

```bash
docker exec zabbix-server sh /usr/lib/zabbix/alertscripts/demo_alert.sh demo-alert "测试主题" "测试正文"
docker exec zabbix-server cat /usr/lib/zabbix/alertscripts/alert.log
```

脚本能手动跑通，就把问题范围收缩到"Zabbix 没调用它"。

## 六、验收结果（2026-11-10 09:12）

**`alert.log` 实际内容**：

```
2026-11-10 09:09:24 | 恢复告警 | 主机: Zabbix server 恢复正常 | 触发器: CPU idle < 20% (Day3 测试) | 严重性: Warning | 时间: 09:08:24
2026-11-10 09:10:24 | (PROBLEM)  | 主机: Zabbix server | 触发器: CPU idle < 20% (Day3 测试) | 严重性: Warning | 时间: 09:10:24
2026-11-10 09:12:24 | 恢复告警 | 主机: Zabbix server 恢复正常 | 触发器: CPU idle < 20% (Day3 测试) | 严重性: Warning | 时间: 09:10:24
```

**`alerts` 表**：3 条记录全部 `status=1`（已发送），`error` 为空。

| 检查项 | 结果 |
|---|---|
| Trigger 翻转 | CPU idle 掉到 0% → PROBLEM，恢复后 → OK ✅ |
| Action 匹配 | escalations 表有记录，actionid=7 ✅ |
| Media 判定 | 修复 active/status 反语义后通过 ✅ |
| 脚本执行 | `alert.log` 写入 3 条 ✅ |
| 告警投递 | alerts 表 `status=1` ✅ |
| PROBLEM + 恢复 闭环 | 双向都成功 ✅ |

## 七、面试可讲点

**概念层**

- **Trigger 是"有状态"的**：不是每次表达式为真都告警，而是 `OK → PROBLEM` 翻转那一刻产生一次事件，恢复时再产生一次 `PROBLEM → OK`。所以**告警只发状态变化，不是持续轰炸**。
- **Action 只负责"匹配 + 编排"，不负责"发送"**：Action 决定"什么条件下、发给谁、用什么方式"，真正的发送是 alerter 执行 Media Type。
- **Media Type 是通道，User Media 是"某人的某个通道"**：光有 Media Type 不够，还得给用户绑一条 Media，两者都必须是**启用**状态。
- **升级（Escalation）机制**：`esc_period` 控制"多久没响应就升级到下一级"，`esc_step_to` 控制"最多升几级"。生产里常用于"先发值班群，10 分钟没人 ack 就打电话"。

**工程层**

- **Zabbix 配置字段的 0/1 语义不能想当然**：`active`、`status` 这类字段大量存在"0=启用"的反直觉设计。**读文档 > 凭直觉**，这次就是踩在这个坑上。
- **配置缓存的坑**：Zabbix Server 把配置放内存缓存，靠 `changelog` 同步。直接改库不生效——这是"为什么我改了数据库却不生效"的经典答案。
- **容器化 Zabbix 的调试开关**：官方镜像会重写配置文件，`DebugLevel` 必须走 `ZBX_DEBUGLEVEL` 环境变量。
- **告警脚本的权限模型**：脚本以容器内 `zabbix` 用户运行，宿主机挂载目录要放行写权限，否则"脚本能跑但写不进日志"。

**方法论层**

- **从结果端往源头查**：`alerts` 表 error → `escalations` 表 → debug 日志 → 脚本手动测试。比"从 Trigger 往下瞎猜"快一个数量级。
- **"静默丢弃"是最难查的故障**：Zabbix 对"媒体禁用"这种情况既不报错也不建记录，默认日志级别完全看不到。**遇到"什么都没发生"，第一反应应该是开 debug 日志找 `will not send`**。
- **区分"配置没生效"和"逻辑没走到"**：前者查缓存/重启，后者查条件/权限。这次先误判为前者，走了不少弯路。

## 八、待优化

- [ ] 告警模板里的 `\r` 是**字面量**没被转义成换行（UI 里输入 `\r` 不会自动转换），可改成真正的回车或在脚本里处理
- [ ] Demo Action 的 `esc_period` 仍是默认 1h，可改成 5m 便于演示升级
- [ ] Admin 默认口令不能长期沿用，首次登录必须改掉
- [ ] 下一步 Day 4：用官方模板（Linux by Zabbix agent）批量接管 Item/Trigger，替换手动建的几条
