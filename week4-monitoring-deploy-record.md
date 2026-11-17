# week4 部署实录：Docker Compose 部署 Prometheus + Grafana + node_exporter 监控栈

> 承接 week3：已在一台阿里云轻量（Ubuntu 24.04，2核2G，公网 IP 203.0.113.10）上用 Docker Compose 跑通了自建镜像 + redis。
> week4 的目标是把 week3 学的"造镜像"升级成"部署一个真实开源项目，并把它运维化"。

---

## 一、目标

用 Docker Compose 部署一套**可观测性监控栈**，实现：

1. 采集这台服务器的 CPU / 内存 / 磁盘 / 网络 / 系统指标
2. 存成时间序列数据（可回溯历史）
3. 用 Grafana 大盘可视化，手机也能看
4. 加一条告警规则，现场演示"服务挂了会报警"
5. 数据卷持久化 + 备份恢复 + 开机自启

对应开源项目：

| 组件 | 仓库 | 作用 |
|---|---|---|
| node_exporter | https://github.com/prometheus/node_exporter | 采集器：把主机指标暴露成 HTTP 文本 |
| Prometheus | https://github.com/prometheus/prometheus | 时序库：定时抓取 + 存储 |
| Grafana | https://github.com/grafana/grafana | 可视化：大盘 + 告警 |

---

## 二、环境与架构

- 云服务器：阿里云轻量，Ubuntu 24.04 LTS，2核2G，公网 IP `203.0.113.10`
- 运行时：Docker + docker compose v2（已有）
- 镜像加速：已配阿里云镜像加速器（`/etc/docker/daemon.json`）
- 已有服务：week3 的自建站占用 `:80`，redis 容器在同一 compose 项目里

### 关键架构决策：三个容器都用 host 网络

```
浏览器/手机 ──公网:3000──> Grafana（host 网络，0.0.0.0:3000）
                              │ 数据源 http://127.0.0.1:9090
                              ▼
                          Prometheus（host 网络，仅 127.0.0.1:9090）
                              │ 抓取 http://127.0.0.1:9100
                              ▼
                          node_exporter（host 网络，仅 127.0.0.1:9100）
                              │ 读 /host/proc、/host/sys
                              ▼
                          Linux 内核（CPU / 内存 / 磁盘 / 网络）
```

**为什么用 host 网络，而不是 compose 内部网络 + 端口发布？**

1. **能用 loopback 收口暴露面**：host 网络下可以直接让 Prometheus、node_exporter 只监听 `127.0.0.1`，从设计上就不对公网暴露——比"靠防火墙挡住"更强。
2. **避开 ufw 与 Docker 的著名冲突**：`ports:` 端口发布走 Docker 的 DNAT/FORWARD 链，**不经过 INPUT 链**，所以 ufw 根本管不住发布的端口；而 host 网络的容器进程直接监听宿主机端口，走 INPUT 链，ufw 规则真正生效。这样 Grafana 的 :3000 就是"安全组 + ufw"双层防护，和 week3 的思路一致。
3. 少一层 NAT，排查链路更短。

**代价**：失去了 compose 内部 DNS（服务名互访）。week3 已经练过内部网络，这次的取舍换来的是更干净的安全模型。

### 端口规划

| 端口 | 服务 | 监听地址 | 是否对公网开放 |
|---|---|---|---|
| 3000 | Grafana | 0.0.0.0 | 是（安全组 + ufw 放行） |
| 9090 | Prometheus | 127.0.0.1 | 否（只走 SSH 隧道访问） |
| 9100 | node_exporter | 127.0.0.1 | 否（只有本机 Prometheus 能抓） |
| 80 | week3 自建站 | 0.0.0.0 | 保持不动 |

---

## 三、文件清单

```
/root/monitoring/
├── docker-compose.yml
└── prometheus/
    └── prometheus.yml
```

数据落在这两个命名卷里（不是匿名卷）：

- `monitoring_prom-data` → 容器内 `/prometheus`
- `monitoring_grafana-data` → 容器内 `/var/lib/grafana`

---

## 四、实施步骤

### 第 0 步：资源与端口体检

```bash
free -h
df -h /
ss -tlnp | grep -E ':(3000|9090|9100)'
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

逐行解释：

- `free -h`：-h 是人类可读（Gi/Mi）。重点看两行：`Mem` 的 available（真实可用内存）和 `Swap`（现在是 0 就对了，下一步要加）。
- `df -h /`：看根分区剩余。这套栈镜像约 1GB、指标数据按 15 天保留约几百 MB，建议至少留 3GB。
- `ss -tlnp | grep -E ...`：`ss` 列套接字，`-t` 只看 TCP，`-l` 只看 listening，`-n` 不做域名解析（快），`-p` 显示进程名。`grep -E` 是扩展正则，`|` 表示或。**没有输出=端口空闲**。
- `docker ps`：`--format` 自定义列。确认 week3 的容器状态，别把 80 端口占了又重复启动。

**记录**：把三条命令的输出贴进本文档末尾的"实测输出"区。

---

### 第 1 步：加 2G swap（防 OOM 把 SSH 一起拖死）

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
free -h
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

逐行解释：

- `fallocate -l 2G /swapfile`：`-l` 指定长度，**瞬间分配 2GB 的连续块**（不像 `dd` 要真写一遍数据，秒级完成）。
- `chmod 600`：只有 root 可读写。swap 里可能有明文内存内容，权限必须收紧。
- `mkswap`：把普通文件格式化成 swap 区，写进 swap 头部信息。
- `swapon`：立即启用该 swap 文件，本次开机内生效。
- `free -h`：**校验点**——Swap 行应从 0 变成 2.0Gi。
- `echo '...' | sudo tee -a /etc/fstab`：写入 `/etc/fstab` 实现开机自动挂载。这里必须用 `| sudo tee -a` 而不是 `sudo echo ... >>`，因为 shell 重定向是由**当前用户**打开的，sudo 只提权了 echo，重定向会因权限不足失败（这是个经典的 shell 权限坑，面试也能讲）。

可选：降低换出倾向（默认 60 太激进，容易把热数据换出去）：

```bash
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

- `sysctl -p`：重新加载 `/etc/sysctl.conf` 使参数立即生效。

---

### 第 2 步：建目录、写 prometheus.yml

```bash
mkdir -p /root/monitoring/prometheus
cd /root/monitoring
cat > prometheus/prometheus.yml <<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: node
    static_configs:
      - targets: ['127.0.0.1:9100']
EOF
cat prometheus/prometheus.yml
```

逐行解释：

- `mkdir -p`：`-p` 递归建目录，父目录不存在也不报错。
- `cat > 文件 <<'EOF' ... EOF`：heredoc，把 EOF 之间的内容整块写进文件，**避免逐行 echo 引号转义地狱**。
- `<<'EOF'` 给 EOF 加了单引号：**关闭变量替换**。如果写 `<<EOF`（不加引号），内容里的 `$变量` 会被 shell 提前展开成空值——写配置文件时这是最容易踩的坑，YAML 里只要有 `$` 就可能被吃掉。
- `scrape_interval: 15s`：Prometheus 每 15 秒抓一次。演示用 15s 足够；生产高频指标可调到 5~15s，但抓得越勤数据量越大。
- `job_name: node`：这个 job 名会变成指标的 `job="node"` 标签，Grafana 大盘模板按这个标签筛选数据。
- `targets: ['127.0.0.1:9100']`：抓取目标。因为三个容器都用 host 网络，Prometheus 直接走宿主机 loopback 就能摸到 node_exporter。
- `cat 文件`：**写完必须回读校验**——YAML 缩进错一个空格就解析失败，而报错往往要等容器起来才暴露（呼应 week3 那句"命令不报错 ≠ 做对了"）。

---

### 第 3 步：写 docker-compose.yml

```bash
cd /root/monitoring
cat > docker-compose.yml <<'EOF'
name: monitoring

services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    network_mode: host
    pid: host
    volumes:
      - /:/host:ro,rslave
    command:
      - --path.rootfs=/host
      - --web.listen-address=127.0.0.1:9100

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prom-data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=15d
      - --web.listen-address=127.0.0.1:9090

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    network_mode: host
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=换成你自己的强密码
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_DATE_FORMATS_DEFAULT_TIMEZONE=Asia/Shanghai
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus

volumes:
  prom-data:
  grafana-data:
EOF
cat docker-compose.yml
```

逐行解释（重点行）：

- `name: monitoring`：项目名，决定卷名前缀（`monitoring_prom-data`）和容器分组名。要 Compose v2.3.3+，Ubuntu 24.04 自带版本满足。
- `restart: unless-stopped`：**这行直接解决 week3 第 8 条"重启后容器不自动启动"**。区别：`always` 会在你手动 `docker stop` 后也自动拉起；`unless-stopped` 会记住"你手动停过"，不再自作主张——手动维护场景用后者更可控。
- `network_mode: host`：容器共享宿主机网络栈。**注意：用了它就不要再写 `ports:`，两者冲突且 ports 会被忽略**。
- `pid: host`：共享宿主机 PID 命名空间，node_exporter 才能按主机视角看进程。
- `volumes: - /:/host:ro,rslave`：把宿主机根目录**只读**挂到容器内 `/host`，`rslave` 表示递归从属挂载，让 `/proc`、`/sys` 这些子挂载点跟着一起可见。没有这一挂载，node_exporter 只能看到容器自己那点指标。
- `--path.rootfs=/host`：告诉 node_exporter"主机根文件系统在 /host"，它就会去读 `/host/proc`、`/host/sys`，**这是拿到真实主机指标的关键参数**。
- `--web.listen-address=127.0.0.1:9100`：**只监听回环**。默认是 `:9100`（即 0.0.0.0，全网可达）。node_exporter 没有任何鉴权，暴露到公网等于把主机指标送人。
- `--storage.tsdb.path=/prometheus`：时序数据存放路径，必须和下面的卷对上。
- `--storage.tsdb.retention.time=15d`：只保留 15 天，超期自动清理。不设会一直涨直到把盘吃满——这是监控系统最常见的自伤事故。
- `--web.listen-address=127.0.0.1:9090`：Prometheus 是**管理面**（能改配置、能看全部数据），同样只绑回环，外部要访问走 SSH 隧道。
- `GF_SECURITY_ADMIN_PASSWORD`：首次启动时设置 admin 密码。不设就是默认 `admin/admin`，而 :3000 是对公网开的，等于开了个后门。
- `GF_USERS_ALLOW_SIGN_UP=false`：关闭自助注册，避免陌生人注册账号。
- `GF_DATE_FORMATS_DEFAULT_TIMEZONE=Asia/Shanghai`：不改的话大盘时间轴按 UTC 显示，比你本地时间**差 8 小时**，演示时会很尴尬。
- `prom-data` / `grafana-data` 写在顶层 `volumes:` 之下：声明为命名卷。**不要用匿名卷**——匿名卷没有名字，备份和恢复时你找不到它。

---

### 第 4 步：起 node_exporter，验证数据源

```bash
cd /root/monitoring
docker compose up -d node-exporter
docker compose ps
docker compose logs --tail=20 node-exporter
```

```bash
curl -s localhost:9100/metrics | head -20
curl -s localhost:9100/metrics | grep -c '^node_'
curl -s localhost:9100/metrics | grep '^node_memory_MemTotal_bytes'
curl -s localhost:9100/metrics | grep '^node_boot_time_seconds'
free -h
```

逐行解释：

- `docker compose up -d node-exporter`：只启动指定服务。`-d` 后台运行（detached）。先起采集源，**验证数据源头可信了再往上搭**，这是分层排查的思路。
- `docker compose logs --tail=20`：`--tail=20` 只看最后 20 行。容器"起来了"不等于"工作正常"，日志才是证据。
- `curl -s ... | head -20`：`-s` 静默模式（不显示进度条），`head` 看前 20 行了解输出格式——你会发现它是**纯文本**，一行一个指标：`指标名{标签}  数值`。
- `grep -c '^node_'`：`-c` 统计行数，`^node_` 表示行首。数量应在几百到上千（视采集器开启数量），**如果是 0 或个位数，说明没读到主机**。
- `grep '^node_memory_MemTotal_bytes'`：**关键校验点**——这个值应该和 `free -h` 里 Mem 的 total **基本一致**（可能有几 MB 差异）。如果它显示的是几百 MB，说明没加 `--path.rootfs` 或没挂 `/:/host`，你采到的是容器自己的指标。
- `grep '^node_boot_time_seconds'`：主机开机时间戳。拿它和 `uptime -s` 对比，能再次确认"看的是宿主机"。

---

### 第 5 步：起 Prometheus，验证抓取

```bash
cd /root/monitoring
docker compose up -d prometheus
docker compose logs --tail=30 prometheus
```

```bash
curl -s localhost:9090/-/healthy
curl -s localhost:9090/api/v1/targets | grep -o '"health":"[a-z]*"'
curl -s localhost:9090/api/v1/query --data-urlencode 'query=up' | head -c 500
```

逐行解释：

- `curl -s localhost:9090/-/healthy`：Prometheus 的健康检查端点，正常返回 `Prometheus Server is Healthy.`。
- `/api/v1/targets`：抓取目标列表接口，用 `grep -o` 抽出 health 字段。**要看到 `"health":"up"`**；如果是 `down`，往下看 `lastError` 字段会直接告诉你原因（最常见是 `connection refused`——node_exporter 没起或端口写错）。
- `/api/v1/query --data-urlencode 'query=up'`：PromQL 查询接口。`--data-urlencode` 会把 query 参数做 URL 编码（`{}`、`=` 这些字符不编码会破坏请求）。`up` 是 Prometheus 自己生成的指标：**1=目标存活，0=目标不可达**。这是 pull 模型的天然优势——采集端能自己判断被采集对象是否健康，不需要被采集方额外上报心跳。
- `head -c 500`：只看前 500 字节，避免 JSON 刷屏。

想看 Prometheus 的 Web UI（因为它只绑回环，必须走隧道）：

```bash
ssh -L 9090:127.0.0.1:9090 root@203.0.113.10
```

- `-L 9090:127.0.0.1:9090`：本地端口转发。把**本地** 9090 的流量通过 SSH 隧道送到**服务端**的 127.0.0.1:9090。这样隧道是加密的，服务端也不需要对公网开放端口。执行后保持这个 SSH 窗口别关，本地浏览器打开 `http://localhost:9090` 即可看到 Targets 页面全是 UP。

---

### 第 6 步：起 Grafana，接数据源，导入大盘

```bash
cd /root/monitoring
docker compose up -d grafana
docker compose ps
```

浏览器打开 `http://203.0.113.10:3000`（**前提：安全组和 ufw 已放行 3000**，见第 8 步），用 `admin` + 你设的密码登录。

**接数据源**（UI 操作）：

1. 左侧 `Connections` → `Data sources` → `Add data source` → 选 `Prometheus`
2. `Connection` → `Prometheus server URL` 填 `http://127.0.0.1:9090`
   - 因为 Grafana 也是 host 网络，直接走宿主机回环即可，不需要容器名
3. 拉到底点 `Save & test`，必须出现绿色的 `Successfully queried the Prometheus API`

**导入现成大盘**：

1. 左侧 `Dashboards` → `New` → `Import`
2. `Import via grafana.com` 输入 `1860`（Node Exporter Full）→ `Load`
3. 底部 `Prometheus` 数据源选刚建的那个 → `Import`
4. 打开后应看到 CPU、内存、磁盘、网络的曲线开始出点

> 如果 Grafana 容器连不上 grafana.com 拉不到模板：在能上网的机器上打开 `https://grafana.com/grafana/dashboards/1860`，把 JSON 复制出来存成 `1860.json`，再走 `Import` → `Upload dashboard JSON file` 上传。

**校验要点**：

- 大盘右上角时间范围选 `Last 1 hour`
- 面板 `CPU Busy` / `Memory` / `Disk Space Used` 有没有数据。**如果全空**，八成是数据源 URL 填错，或 Prometheus 里 `job` 标签和大盘变量不匹配
- 顺手把 Grafana 右上角 `Save dashboard` 存一份，避免改动丢失

---

### 第 7 步：锁定镜像版本（可选但推荐）

先用 `latest` 跑通，再从正在运行的服务里读出真实版本号：

```bash
curl -s localhost:9100/metrics | grep '^node_exporter_build_info'
curl -s localhost:9090/api/v1/status/buildinfo
curl -s localhost:3000/api/health
```

逐行解释：

- `node_exporter_build_info`：node_exporter 自带的构建信息指标，标签 `version="x.y.z"` 就是版本号。
- `/api/v1/status/buildinfo`：Prometheus 的版本查询接口。
- `/api/health`：Grafana 健康接口，返回 JSON 里带 `version`。

拿到版本后，把 compose 里的 `:latest` 改成具体版本号（如 `prom/prometheus:v3.1.0`），再 `docker compose up -d` 重建。

**为什么面试会问"为什么不用 latest"**：`latest` 是浮动标签，别人（或三个月后的你）再拉一次可能拿到不同版本，行为变了、配置语法改了，问题却无法复现。锁定版本 = 可复现的部署。

---

### 第 8 步：安全收口（双层：安全组 + ufw）

**阿里云控制台安全组**（入方向）：

| 端口 | 来源 | 说明 |
|---|---|---|
| 22 | 你的 IP（或 0.0.0.0/0） | SSH，建议限自己 IP |
| 80 | 0.0.0.0/0 | week3 自建站 |
| 443 | 0.0.0.0/0 | 预留 |
| 3000 | 0.0.0.0/0 | Grafana（演示用；考完可收窄到你的 IP） |
| 9090 / 9100 | **不放行** | 管理面与采集面，永不对公网开 |

**服务器 ufw**：

```bash
sudo ufw status numbered
sudo ufw allow 3000/tcp
sudo ufw status numbered
```

逐行解释：

- `ufw status numbered`：`numbered` 给每条规则加编号，方便后续按编号删除（`ufw delete 序号`）。**改规则前先看现状**，别盲加。
- `sudo ufw allow 3000/tcp`：显式放行 3000 的 TCP。其余入站按 ufw 默认策略（deny incoming）拒绝。
- 再次 `status numbered`：**校验规则真的加上了**。

**这里有个经典认知点，值得专门记住**：

> `ufw` 的规则挂在 iptables 的 **INPUT 链**上。而用 `ports:` 发布容器端口时，流量走的是 nat/PREROUTING 的 DNAT + FORWARD 链的 DOCKER 链，**根本不经过 INPUT**——所以 ufw 管不住 docker 发布的端口。
>
> 本次三个服务都用 `network_mode: host`，容器进程直接监听宿主机端口，走 INPUT 链，**ufw 规则真正生效**。
>
> 结论：安全组是"机房门口的闸机"，ufw 是"服务器本机门锁"。一旦用端口发布模式，本机门锁对那个端口就失效了——**云安全组才是最后一道防线**。

---

### 第 9 步：备份与恢复演练

先找到卷在宿主机上的真实路径：

```bash
docker volume ls | grep monitoring
docker volume inspect monitoring_prom-data --format '{{.Mountpoint}}'
docker volume inspect monitoring_grafana-data --format '{{.Mountpoint}}'
```

逐行解释：

- `docker volume ls | grep monitoring`：确认卷名。加了 `name: monitoring` 项目名后，卷会带上 `monitoring_` 前缀。
- `docker volume inspect ... --format '{{.Mountpoint}}'`：`--format` 用 Go 模板语法直接取字段。`{{.Mountpoint}}` 就是卷在宿主机上的落盘目录（通常在 `/var/lib/docker/volumes/<卷名>/_data`）。**"卷"是逻辑名，数据永远在宿主机磁盘上**——这句话面试常被追问。

备份：

```bash
mkdir -p /root/backup
docker compose stop prometheus grafana
sudo tar czf /root/backup/prom-data-$(date +%F).tgz -C /var/lib/docker/volumes/monitoring_prom-data/_data .
ls -lh /root/backup/
```

逐行解释：

- `docker compose stop prometheus grafana`：**先停服务再备份**。Prometheus 在写时序块，热备份可能拿到半写状态的文件，恢复后可能出现数据块损坏。
- `tar czf 目标 -C 目录 .`：`c` 创建，`z` gzip 压缩，`f` 指定输出文件；`-C` 先切换到哪个目录再打包（避免把长绝对路径打进包里），最后的 `.` 表示打包当前目录全部内容。
- `$(date +%F)`：命令替换，展开成 `2026-10-19` 这种日期，用于文件名带日期。
- `ls -lh`：`-l` 长格式，`-h` 可读大小。确认备份文件非 0 字节——**0 字节是最危险的"看起来成功了"**。

恢复演练（真正的验证，别跳过）：

```bash
sudo tar xzf /root/backup/prom-data-2026-10-19.tgz -C /var/lib/docker/volumes/monitoring_prom-data/_data
docker compose start prometheus grafana
curl -s localhost:9090/-/healthy
curl -s localhost:9090/api/v1/query --data-urlencode 'query=up' | head -c 300
```

逐行解释：

- `tar xzf`：`x` 解包，其余同上。注意文件名改成你实际的备份文件名。
- `docker compose start ...`：启动之前 stop 的容器，`start` 不会重建容器。
- 两条 curl：确认服务恢复且能查历史数据。**只备份不演练，等于没有备份**——面试问"你怎么保证数据不丢"，答"我有备份"没什么分量，答"我做过恢复演练并验证了数据完整性"才有分量。

---

### 第 10 步：告警现场演示

**在 Grafana 里建规则**：

1. 左侧 `Alerting` → `Alert rules` → `New alert rule`
2. `Name` 填 `Node Exporter Down`
3. `Query A`：数据源选 Prometheus，查询填 `up{job="node"}`，类型选 `Instant`
4. `Condition B`：`IS BELOW 1`
5. `Evaluation`：`Every 15s`，`Pending period` 设为 `15s`（即异常持续 15 秒才真正进入 Alerting）
6. 保存（`Contact point` 可以先用默认；想真正收到通知就加一个 webhook，比如钉钉/飞书机器人的地址）

**然后演示故障**：

```bash
cd /root/monitoring
docker compose stop node-exporter
```

等 30~60 秒，在 Grafana 的 `Alerting` → `Alert rules` 页面看状态变为 **Alerting / Firing**；大盘上的 `up` 曲线也会掉到 0。

恢复：

```bash
docker compose start node-exporter
```

再过一会儿状态自动回到 Normal。**"采集源消失 → 采集端自己发现 → 触发告警 → 恢复自动解除"，这一条链路现场演一遍，比只展示大盘说服力强得多。**

---

## 五、验收清单

- [ ] `free -h` 里 Swap 显示 2.0Gi
- [ ] `docker compose ps` 三个容器都是 `Up`，且 `restart` 策略为 `unless-stopped`
- [ ] `curl localhost:9100/metrics` 能取到指标，`node_memory_MemTotal_bytes` ≈ `free -h` 的 total
- [ ] `curl localhost:9090/api/v1/targets` 中 `health` 为 `up`
- [ ] 重启整机后（`sudo reboot`）三个容器自动回来
- [ ] 手机访问 `http://203.0.113.10:3000` 能看到大盘曲线
- [ ] 外网无法访问 `:9090` 和 `:9100`（在别的机器上 curl 应超时）
- [ ] 备份文件非 0 字节，且恢复演练后数据仍可查
- [ ] 停掉 node_exporter 后告警能触发，重启后自动恢复

---

## 六、踩坑预案

| 现象 | 大概率原因 | 排查动作 |
|---|---|---|
| 目标 health 是 `down` | node_exporter 没起 / 端口写错 / 监听地址被改 | `docker compose logs node-exporter`；`ss -tlnp \| grep 9100` 看监听在不在 127.0.0.1 |
| 指标数量极少、内存总量对不上 | 漏了 `--path.rootfs=/host` 或漏挂 `/:/host` | 检查 compose 里 command 和 volumes 两处 |
| Prometheus 启动即退出 | YAML 缩进错 / `prometheus.yml` 只读挂载的路径不存在 | `docker compose logs prometheus`；`docker compose config` 校验语法 |
| Grafana 打不开（3000 连不上） | 安全组或 ufw 没放行 | 先本机 `curl -I localhost:3000`；本机通说明是防火墙/安全组问题 |
| Grafana 登录不了 | `GF_SECURITY_ADMIN_PASSWORD` 拼错，或用了旧卷里的旧密码 | 该变量**只在首次创建卷时生效**，改密码要 `docker compose down -v` 或进 UI 改 |
| 大盘打开全是 No data | 数据源 URL 错 / 时间范围选了未来 / job 标签不匹配 | 在 Grafana `Explore` 里直接查 `up`，先确认数据源活着 |
| 大盘时间轴差 8 小时 | 时区没设 | 确认 `GF_DATE_FORMATS_DEFAULT_TIMEZONE=Asia/Shanghai` |
| 磁盘被吃满 | 没设 retention | `docker system df`；确认 `--storage.tsdb.retention.time=15d` |
| 机器突然卡死/SSH 掉线 | 内存打满触发 OOM | `dmesg -T \| grep -i oom`；确认 swap 已生效 |
| 改完 compose 不生效 | 只 `restart` 不会重建容器 | 配置/镜像改动要用 `docker compose up -d`（会重建差异容器） |

---

## 七、面试可讲的点

**概念层**

- **pull 模型 vs push 模型**：Prometheus 主动来拉，而不是被采集方推。好处是采集端掌握节奏、能通过 `up` 指标自检目标存活；代价是需要服务发现和可达性配置。
- **Exporter 模式**：被监控对象不需要改造，只需旁边跑一个 exporter 把状态翻译成 HTTP 文本指标。解耦、易扩展。
- **时序数据模型**：`指标名 + 标签集 + 时间戳 + 数值`。为什么不用关系库：写入量极大、按时间范围扫描、标签基数问题，关系库索引扛不住。
- **指标类型**：Counter（只增累计，如请求总数）、Gauge（可增可减瞬时值，如内存占用）、Histogram / Summary（分位数，如 P99 延迟）。查速率要用 `rate()` 对 Counter 做处理。

**工程层**

- **命名卷 vs 匿名卷 vs bind mount**：卷是逻辑名、数据在宿主机落盘、可备份可迁移；匿名卷无名字，运维上等于黑洞。
- **`restart: unless-stopped` vs `always`**：前者尊重"人工停止"的意图，后者会强行拉起。可控性不同。
- **最小暴露面**：管理面（Prometheus）和采集面（node_exporter）只监听 `127.0.0.1`，外部访问走 SSH 隧道——**从设计上就不暴露，而不是靠"应该没人知道端口"**。
- **ufw 与 Docker 的冲突**：`ports:` 发布走 DNAT/FORWARD，绕过 ufw 的 INPUT 链。这是很多生产事故的根因，也是"为什么云安全组必须配"的答案。
- **存储规划**：`retention.time` 是必须显式设置的自保参数，否则监控系统会把磁盘吃满把自己搞死。
- **可复现性**：锁镜像版本，不用 `latest`；配置文件纳入版本管理。

**方法论层**

- **分层定位**：数据源（9100）→ 采集端（9090 targets）→ 展示端（3000）→ 客户端（浏览器/防火墙/安全组）。每层单独 curl 验证，一层通了再上一层，比"一起起然后猜哪里坏了"快得多。
- **命令不报错 ≠ 做对了**（week3 的教训在本周的新形态）：`docker compose up -d` 成功只代表容器被创建了，`docker compose ps` 显示 Up 也不代表业务健康。真正的证据是**接口返回的内容**。
- **备份必须演练恢复**，否则备份只是心理安慰。

---

## 八、实测输出（自己填写）

### 第 0 步

```
free -h:

df -h /:

ss -tlnp:

docker ps:
```

### 第 4 步

```
node_ 指标条数:

node_memory_MemTotal_bytes:

free -h 的 Mem total:

是否一致:
```

### 第 5 步

```
targets health:

```

### 第 10 步

```
告警状态变化:

```

---

## 九、本周新增踩坑记录（自己填写）

| 序号 | 现象 | 排查过程 | 根因 | 结论/教训 |
|---|---|---|---|---|
| 1 |  |  |  |  |
