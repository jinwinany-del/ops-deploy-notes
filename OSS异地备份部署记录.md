# OSS 异地备份部署记录

记录日期：2026-11-15
环境：阿里云 ECS（上海），Ubuntu 24.04，2核2G，公网 203.0.113.10，主域名 example.com
对象存储：OSS，Bucket `my-backup-bucket`，华东2（上海）

> 承接 week3 站点部署、week4 监控栈、example.com 域名上线。
> 目标：解决「备份和原数据在同一块云盘上」这个硬伤。

---

## 一、要解决的问题

此前所有备份都落在本机：

```bash
tar czf /root/app-backup-$(date +%F).tar.gz /root/app
tar czf /root/backup/prom-data-$(date +%F).tgz -C /var/lib/docker/volumes/monitoring_prom-data/_data .
```

包在 `/root` 下，而 `/root` 和站点代码、Prometheus 数据卷共用同一块系统盘。

| 事故 | 改造前 | 改造后 |
|---|---|---|
| `rm -rf /root/app` 手滑 | 备份包可能一起没 | 从 OSS 拉回 |
| 系统盘写满/损坏，机器起不来 | 备份一起读不出 | 重装系统后从 OSS 恢复 |
| 实例欠费被释放 | 全丢 | OSS 对象独立存在 |
| 换机器 | 只能重装 | 新机拉备份半小时迁移 |

**结论：同一块盘上复制不叫备份，叫冗余。冗余救不了盘坏。**

---

## 二、最终状态

| 项目 | 现状 |
|---|---|
| Bucket | `my-backup-bucket`，华东2（上海），标准存储，私有，阻止公共访问已开通 |
| 版本控制 | 关闭 |
| 存储冗余 | 同城冗余（ZRS），桶创建时选定，创建后不可改 |
| 访问凭证 | RAM 用户 `backup-robot`，仅编程访问 |
| 权限策略 | 自定义策略 `oss-backup-only`，仅限该桶，**无 DeleteObject** |
| Bucket 策略 | 拒绝非 HTTPS 请求（`acs:SecureTransport=false`），主账号配置 |
| 客户端 | ossutil 2.2.2，`/usr/local/bin/ossutil` |
| 端点 | `https://oss-cn-shanghai-internal.aliyuncs.com`（内网） |
| 备份脚本 | `/root/scripts/backup-to-oss.sh` |
| 定时任务 | `10 3 * * * /root/scripts/backup-to-oss.sh` |
| 本地保留 | 7 天（`find -mtime +7 -delete`） |
| 云端保留 | 永久累积，同日期重跑为覆盖 |
| 月费用 | 约 0.00 元（KB 级数据） |

---

## 三、实施步骤

### 第 1 步：开通 OSS 并创建 Bucket

新账号首次进入 OSS 控制台会提示「尚未开通对象存储服务 OSS」，点**立即开通**并同意协议即可。开通不收费，费用从实际存储/流量产生。

创建 Bucket 的关键参数：

| 参数 | 取值 | 理由 |
|---|---|---|
| 名称 | `my-backup-bucket` | 全局唯一，仅小写字母/数字/短横线 |
| 地域 | 华东2（上海） | **必须与 ECS 同地域**，否则只能走公网，产生流量费且慢 |
| 存储类型 | 标准存储 | 低频访问有最短存储周期和取回费用，演练会被计费规则绊住 |
| 存储冗余 | 同城冗余 / 本地冗余 | ZRS 多可用区冗余，价格约贵 25%~40%；备份场景 LRS 也够 |
| 读写权限 | 私有 | 公共读等于把备份公开给全网 |
| 版本控制 | 关闭 | 开启后每次覆盖都留历史版本，存储量翻倍 |
| 阻止公共访问 | 开通 | 从账号层面禁止匿名访问与公共读 ACL |

### 第 2 步：RAM 子账号 + 最小权限

RAM → 身份管理 → 用户 → 创建用户：

- 登录名称：`backup-robot`
- 访问方式：**只勾「使用永久 AccessKey 访问」**，不勾控制台登录

权限策略 → 创建策略 → 脚本编辑：

```json
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "oss:PutObject",
        "oss:GetObject",
        "oss:ListObjects",
        "oss:GetBucketInfo",
        "oss:InitiateMultipartUpload",
        "oss:UploadPart",
        "oss:ListParts",
        "oss:AbortMultipartUpload"
      ],
      "Resource": [
        "acs:oss:*:*:my-backup-bucket",
        "acs:oss:*:*:my-backup-bucket/*"
      ]
    }
  ]
}
```

要点：

- `Resource` 两条都要写。不带 `/*` 的是 Bucket 本身（列文件、查属性），带 `/*` 的是桶内对象。漏一个会报权限错误且提示不明显。
- **没有 `oss:DeleteObject`**。上传与下载不需要它。没有删除权限意味着密钥泄露或被脚本改坏，也删不掉历史备份。
- 分片上传那几个 Action 用于大文件切片，备份包变大后必需。

最后在用户 → 权限管理 → 新增授权 → 选自定义策略 `oss-backup-only`。

### 第 3 步：安装 ossutil 2.2.2

```bash
uname -m                        # 确认架构，x86_64 对应 amd64 包
sudo apt install -y unzip
cd /root
curl -o ossutil-2.2.2-linux-amd64.zip https://gosspublic.alicdn.com/ossutil/v2/2.2.2/ossutil-2.2.2-linux-amd64.zip
unzip ossutil-2.2.2-linux-amd64.zip
cd ossutil-2.2.2-linux-amd64
chmod 755 ossutil
sudo mv ossutil /usr/local/bin/ && sudo ln -s /usr/local/bin/ossutil /usr/bin/ossutil
ossutil
```

说明：

- `unzip` 是必需的，Ubuntu 自带的 `tar` 解不开 zip。
- zip 包**保留了 Unix 权限位**，解出来往往已经是 755，`chmod` 属于兜底。这与「直接下载单个二进制文件」不同，后者权限是 644，必须手动加执行位。
- 装到 `/usr/local/bin` 是因为该目录在 `PATH` 中，且不属于 apt 管理范围（`/usr/bin` 里由包管理器维护）。
- 额外建 `/usr/bin` 软链接，兼容写死该路径的脚本。软链接只存路径字符串，升级时只需替换真实文件，不会出现两个版本。

### 第 4 步：配置凭证

```bash
ossutil config
chmod 600 /root/.ossutilconfig
```

| 提示 | 填写 |
|---|---|
| 配置文件路径 | 回车，默认 `/root/.ossutilconfig` |
| AccessKey ID | RAM 用户的 ID |
| AccessKey Secret | RAM 用户的 Secret（输入不可见） |
| Region | `cn-shanghai` |
| Endpoint | `https://oss-cn-shanghai-internal.aliyuncs.com` |

Endpoint 两种填法的区别：

| 填法 | 网络路径 | 流量费 |
|---|---|---|
| `oss-cn-shanghai-internal.aliyuncs.com` | 阿里云内网 | 免费 |
| 回车走默认（公网） | 公网出口绕回 | 按量计费 |

`-internal` 只在同地域的阿里云机器上可解析，换到本机电脑上使用需改成公网地址。

配置文件存的是**明文 Secret**，权限必须 600。

### 第 5 步：手工上传与验证

```bash
cd /root
tar czf app-$(date +%F).tar.gz app          # 相对路径打包，避免把 /root 写进包
ossutil cp app-2026-11-13.tar.gz oss://my-backup-bucket/backups/
ossutil ls oss://my-backup-bucket/backups/
```

- 目标结尾的 `/` 表示「放到这个前缀下，文件名沿用源文件名」。
- `backups/` 不需要预先创建——对象存储没有目录，`/` 只是对象名里的字符。
- 以 `oss://` 为源、本地为目标的 `cp` 即为下载，语法与 `cp` 同构。

### 第 6 步：恢复演练（关键，不可跳过）

```bash
mkdir -p /root/restore-test
ossutil cp oss://my-backup-bucket/backups/app-2026-11-13.tar.gz /root/restore-test/
tar xzf /root/restore-test/app-2026-11-13.tar.gz -C /root/restore-test/
diff -r /root/app /root/restore-test/app
ls -l /root/restore-test/app/ssl/
```

- 演练**不删除生产目录**，在隔离目录恢复后比对，避免真删源码。
- `diff -r` 无输出 = 内容完全一致，这是恢复成功的直接证据，比「能解压」有力得多。
- `tar` 会保留权限位，所以恢复后必须检查私钥权限。

**本次演练发现的真实问题**：

`grafana.example.com.key` 在原目录和恢复目录里都是 `-rw-r--r--`（644），任何人都能读私钥。这不是恢复造成的——`tar` 忠实复制了原本就错的权限。

修复：

```bash
chmod 600 /root/app/ssl/grafana.example.com.key /root/restore-test/app/ssl/grafana.example.com.key
```

教训：**备份不会修正错误，它把错误一起备份。恢复流程的校验项必须包含权限位，而不只是文件和大小。**

### 第 7 步：脚本化与定时任务

`/root/scripts/backup-to-oss.sh`：

```bash
#!/bin/bash

DATE=$(date +%F)
PKG=/root/backup/app-$DATE.tar.gz
LOG=/root/scripts/backup-to-oss.log

mkdir -p /root/backup

tar czf "$PKG" -C /root app
if [ ! -s "$PKG" ]; then
  echo "$(date '+%F %T') FAIL 本地包为空" >> "$LOG"
  exit 1
fi

if ossutil cp "$PKG" oss://my-backup-bucket/backups/ --force --config-file /root/.ossutilconfig >> "$LOG" 2>&1; then
  echo "$(date '+%F %T') OK $(du -h "$PKG" | awk '{print $1}')" >> "$LOG"
else
  echo "$(date '+%F %T') FAIL 上传失败" >> "$LOG"
fi

find /root/backup -name 'app-*.tar.gz' -mtime +7 -delete
```

关键点：

- `--force`：跳过同名对象覆盖确认。**cron 没有终端，任何交互提示都会让任务永久挂起**。
- `>> "$LOG" 2>&1`：cron 抓不到屏幕输出，不写日志等于瞎跑。
- `--config-file`：显式指定配置文件，不依赖 cron 的 HOME 环境。
- `-s` 判断包非空，防止把空包传上去占个名字。
- 本地只留 7 天，OSS 永久累积。

crontab：

```
10 3 * * * /root/scripts/backup-to-oss.sh
```

选 3:10 而非 3:00，是为了和已有的 `log_clean.sh`（3:00）错开，日志时间戳不混。

### 第 8 步：权限验证与传输层加固

**验证最小权限真的生效**（用不存在的对象探路，不会误删真实备份）：

```bash
ossutil rm oss://my-backup-bucket/backups/not-exist-test --force
ossutil ls
ossutil ls oss://my-backup-bucket/backups/
```

| 命令 | 预期 | 说明 |
|---|---|---|
| `rm` 不存在的对象 | 403 `AccessDenied` | 先校验权限再判断对象存在，说明确实无删除权 |
| `ls` 不带桶名 | 403 `forbidden to oss:ListBuckets` | 无法枚举账户下其他桶 |
| `ls` 桶内 | 正常列出 | 收紧未误伤功能 |

**Bucket 策略强制 HTTPS**（主账号在控制台配置，子账号无 `PutBucketPolicy` 权限）：

```json
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": ["*"],
      "Action": ["oss:*"],
      "Resource": [
        "acs:oss:*:*:my-backup-bucket",
        "acs:oss:*:*:my-backup-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "acs:SecureTransport": "false"
        }
      }
    }
  ]
}
```

- `Deny` 优先级高于任何 `Allow`，`Principal: ["*"]` 表示对所有人（含自己）生效。
- `acs:SecureTransport=false` 只在请求走明文 HTTP 时触发，HTTPS 完全不受影响。
- 策略生效有延迟，官方提示为数分钟内刷新，保存后立即测试可能还没生效。

验证：

```bash
ossutil ls oss://my-backup-bucket/backups/                 # 正常
ossutil ls oss://my-backup-bucket/backups/ --endpoint http://oss-cn-shanghai-internal.aliyuncs.com   # 403
```

**如何从报错文案判断是哪一层拒绝**：

| Message | 拒绝来源 |
|---|---|
| `You have no right to access this object because of bucket acl.` | RAM 身份策略（403 通用文案，与 ACL 无关） |
| `You are forbidden to oss:ListBuckets.` | RAM 身份策略，且点明了缺失的 Action |
| `Access denied by bucket policy.` | Bucket 策略（资源层） |

至此四层防线：安全组/ufw（网络）→ Bucket 策略拒绝明文（传输）→ RAM 策略（身份）→ Bucket 策略 + 阻止公共访问（资源）。

### 第 9 步：凭证硬编码自查

```bash
grep -rl -E 'LTAI[0-9A-Za-z]{12,}' /root --exclude-dir=.git 2>/dev/null
grep -rl -E 'accessKeySecret' /root --exclude-dir=.git 2>/dev/null
grep -rl -E 'oapi\.dingtalk\.com/robot/send\?access_token=[a-f0-9]{20,}' /root --exclude-dir=.git 2>/dev/null
ls -l /root/.ossutilconfig /root/.dingtalk-token 2>/dev/null
find /root -maxdepth 4 -name .git -type d 2>/dev/null
```

写法上的要求：

- 一律用 `-l`（只输出文件名）而不是 `-n`，**绝不把密钥值打到屏幕上**。排查过程本身就是泄露路径。
- `-I` 跳过二进制文件，避免刷乱码；`2>/dev/null` 丢弃权限报错。

本次结果：

| 检查项 | 结果 |
|---|---|
| AccessKey ID 出现位置 | 仅 `/root/.ossutilconfig` |
| Secret 出现位置 | 仅 `/root/.ossutilconfig` |
| 钉钉 token 硬编码 | 无命中（脚本从 `/root/.dingtalk-token` 读取，做法正确） |
| 两个密钥文件权限 | 均为 `-rw-------`（600） |
| Git 仓库 | `/root` 下无 `.git`，无版本历史泄露面 |

**如果排查时把 Secret 明文输出到屏幕**：凭证一旦以明文离开控制边界即视为泄露，正确处理是**轮换密钥**（禁用旧 AccessKey 并重建），而不是清理 shell history ——history 只存命令、不存输出，清理它是无效动作。

判断原则：能用"只输出文件名/计数"解决的检查，就不要把内容打印出来。

---

## 四、费用

| 计费项 | 用量 | 单价 | 月费 |
|---|---|---|---|
| 存储 | 约 300KB/月新增 | 0.12 元/GB/月（标准 LRS） | 0.00 元 |
| PUT 请求 | 30 次/月 | 0.01 元/万次 | 0.00 元 |
| 上传流量 | 内网 endpoint | 免费 | 0 元 |
| 下载流量 | 演练时走内网 | 免费 | 0 元 |

一年累计约 3.6MB，账单基本显示 `0.00`。真正让 OSS 账单变大的不是存储，而是**公共读桶被外网大量下载产生的流量费**。

---

## 五、踩坑清单

| 序号 | 现象 | 根因 | 处理 |
|---|---|---|---|
| 1 | 进入 OSS 提示「尚未开通」 | 新账号未开通服务 | 控制台点立即开通，不收费 |
| 2 | `bash script.sh` 长时间无响应、不返回 | `ossutil cp` 覆盖同名对象时弹交互确认，输出被 `>>` 重定向吞掉 | 加 `--force`。cron 环境无终端会永久挂起 |
| 3 | `chmod 755` 执行后权限没变化 | zip 包已记录 755，解压时已还原 | 无害，属兜底动作 |
| 4 | 恢复后发现私钥 644 | 原始目录权限本就错误，`tar` 原样复制 | `chmod 600`，并把权限校验加入恢复流程 |
| 5 | heredoc 里 `$DATE` 变成空值 | `<<EOF` 不加引号会提前展开变量 | 必须写 `<<'EOF'` |
| 6 | `echo"退出码：$?"` 报 `command not found` | 命令与引号间缺空格，整串被当成命令名 | `echo "退出码: $?"` |
| 7 | 备份与 log_clean 同在 3:00 | 时间撞车，日志顺序混乱 | 改到 3:10 |
| 8 | 下错架构的安装包 | amd64 与 arm64 混用 | 先 `uname -m` 确认，报错是 `Exec format error` |

---

## 六、验收清单

- [x] `ossutil ls oss://my-backup-bucket` 返回 `Object Number is: 0`
- [x] 手工上传后 OSS 列表出现对象，大小与本地一致
- [x] 下载解压后 `diff -r` 无输出
- [x] 两个 `.key` 文件权限为 600
- [x] 脚本手动执行返回 0，日志出现 `OK`
- [x] OSS 上按日期累积多个独立对象，不存在覆盖
- [x] crontab 已挂 `10 3 * * *`，`systemctl is-active cron` 为 active
- [x] 删除不存在的对象返回 403，确认确实没有删除权限
- [x] `ossutil ls` 不带桶名返回 403，无法枚举其他桶
- [x] HTTP 明文请求返回 403 `Access denied by bucket policy`，HTTPS 正常
- [ ] 次日检查日志出现新 `OK`，OSS 多出一个新日期对象

---

## 七、面试可讲

**方案层**

- 备份必须存在于**独立故障域**。同机复制是冗余，不是备份。
- 选择同地域内网端点：免流量费、延迟低；代价是无法覆盖「整个人地域故障」，跨地域容灾需开 CRR，成本另计。
- 本地保留 7 天 + 云端永久累积：本地省磁盘，云端省钱，两端保留策略不同。

**安全层**

- RAM 子账号 + 自定义策略，最小权限，只到单个 Bucket。
- 策略里不授予 `DeleteObject`：备份系统最大的威胁是被删除，权限层面直接断掉这条路径。
- 配置文件 600，Secret 不进脚本、不进仓库。

**工程层**

- 一切进 cron 的命令要按「无人值守」设计，交互提示必须用 `--force` 关掉。
- cron 环境的 PATH 与 HOME 与登录 shell 不同，路径写绝对、配置文件显式指定。
- 备份必须演练恢复，且校验项要包含**权限位**。只比对文件存在与否会漏掉本次的 644 私钥问题。
- 脚本要有非空校验，避免把 0 字节包当成成功。

**30 秒版**

> 我之前的备份和站点代码放在同一块云盘上，等于只有冗余没有备份。后来改成每日定时打包上传 OSS，走内网端点不产生流量费；用 RAM 子账号做最小权限，只允许对该桶上传下载、没有任何删除权限；本地留 7 天、云端永久累积。做完之后在隔离目录做过完整恢复演练，用 `diff -r` 校验一致，过程中还发现 Grafana 私钥一直是 644 可读，收口成 600，并把权限校验写进了恢复检查项。

---

## 八、遗留项

- [ ] 定期轮换该子账号的 AccessKey，轮换后更新 `/root/.ossutilconfig` 并复查权限为 600；备份范围若扩大到数据库或用户数据，需先完成凭证隔离再启用
- [ ] 监控数据卷（`monitoring_prom-data`、`monitoring_grafana-data`）尚未纳入备份
- [ ] 未配置 OSS 生命周期规则；数据量变大后需为 `backups/` 前缀设转低频/过期删除
- [ ] 未做跨地域复制（CRR），当前只覆盖单地域内故障
- [ ] 外部探活（告警盲区）仍未补，见《故障复盘-2026-10-20-整机内存耗尽》第十三节
- [ ] 证书 90 天到期无自动续期，仍依赖日历提醒

---

## 附录：为什么对象存储也要权限体系

1. **入口是公网 API，凭证就是唯一门锁**
   OSS 没有 SSH、没有系统用户。谁拿到 AccessKey，谁就能以该身份读写对象。没有策略，就等于门锁只有一把钥匙，钥匙丢了全套失守。

2. **主账号密钥的爆炸半径是整个账户**
   主账号 AccessKey 能删 ECS、改账单、释放实例、访问所有云产品。一旦它被写进脚本、提交进 Git、被打包到镜像里，后果远超「备份被读」。

3. **最小权限限制爆炸半径**
   `backup-robot` 只能往一个桶里放和取。即使泄露，攻击者也只能写垃圾进这一个桶，删不掉历史备份，也碰不到 ECS 和其他云资源。

4. **不授予删除权，是备份系统的设计前提**
   勒索软件与误操作的共同点是「先删备份再加密/再破坏」。备份账号没有 `DeleteObject`，这条攻击链在权限层就断了。

5. **可审计、可轮换**
   不同用途用不同子账号，访问日志能区分是谁在操作；密钥泄露时只轮换一个身份的凭证，不必动整个账户。

6. **真实事故形态**
   最常见的事故不是「被人攻破控制台」，而是密钥硬编码在脚本或代码里，被提交到公开仓库后被自动化工具扫到，主账号密钥被拿去开矿机，账单在几小时内涨到数千元。子账号 + 最小权限 + 不落盘明文，是防这条路径最省事的手段。
