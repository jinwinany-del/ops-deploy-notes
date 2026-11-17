# 阶段一 · SQL 突击（10/29 - 10/31）练习记录

> 目标：三天做到「面试官问 SQL 不慌，能现场写出来并讲清为什么」。
> 规则：**所有题先自己写，写完再对照答案**；表格里的「标准行数」用来判断自己写对了没（不看答案也能自查）。

## 文件清单

| 文件 | 用途 |
|---|---|
| [00_建表与数据.sql](sql_phase1/00_建表与数据.sql) | 建库 `practice` + 用户表 30 条 + 订单表 45 条 |
| [01_单表20题.sql](sql_phase1/01_单表20题.sql) | 10/29 的 20 道单表题（附参考答案 + 一句话要点） |
| [02_多表与聚合.sql](sql_phase1/02_多表与聚合.sql) | 10/30 的 10 道多表题 + 3 条统计 SQL（产出） |
| [03_索引与慢查询.sql](sql_phase1/03_索引与慢查询.sql) | 10/31 的索引、explain 实操 + 3 句问答 |

---

## 0. 环境准备（一次就好）

你本机 MySQL 8.0 服务已经在跑，直接用自己的账号进命令行：

```powershell
mysql -uroot -p
```

逐行解释：
- `mysql`：MySQL 客户端程序（`C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe`）。
- `-u root`：用 root 这个用户登录；`-p` 表示「回车后单独提示输入密码」，密码不会明文回显。
- 没有 `-p` 会以空密码登录，会报 `ERROR 1045 Access denied`（刚才就是这个问题）。

进去后执行建表脚本（路径用正斜杠，MySQL 命令行里 `\` 会被当成转义符）：

```sql
SOURCE sql_phase1/00_建表与数据.sql;
USE practice;
```

逐行解释：
- `SOURCE`：读取并执行一个本地 SQL 文件，是 mysql 命令行专用命令，末尾必须带分号。
- `USE practice`：切换到刚建的库，之后所有题都在这个库里执行（漏了这句会报「No database selected」）。

最后应看到 `users 行数 = 30`、`orders 行数 = 45`。这两个数不对，说明数据没插全，重新 SOURCE 一次即可（脚本里是 `DROP TABLE IF EXISTS`，可重复执行）。

---

## 1. Day1（10/29）单表查询

对应清单：`select / where / order by / limit` 熟练；建表插 30 条；完成 20 道单表题；产出练习记录。

### 记录表（把自己的 SQL 贴进「我的答案」列）

| 题号 | 考点 | 标准行数 | 我的答案 | 通过 |
|---|---|---|---|---|
| 1 | select + order by | 30 | | |
| 2 | where 等值 + 中文值引号 | 4 | | |
| 3 | and 多条件 | 18 | | |
| 4 | in | 19 | | |
| 5 | between 日期区间 | 8 | | |
| 6 | like 前缀匹配 | 1 | | |
| 7 | is null（不能 = null） | 2 | | |
| 8 | where + order by desc | 25 | | |
| 9 | between + and + order by | 13 | | |
| 10 | order by + limit | 5 | | |
| 11 | limit + offset 分页 | 10 | | |
| 12 | 计算列 + round + 别名 | 45 | | |
| 13 | count(*) | 30 | | |
| 14 | count(distinct) | 3 | | |
| 15 | sum + where | 1845.90 | | |
| 16 | 排序取 1 条 vs 子查询（并列坑） | 1 | | |
| 17 | order by 日期 desc + limit | 3 | | |
| 18 | 多列排序 | 3 | | |
| 19 | and / or 优先级（括号） | 10 | | |
| 20 | 组合条件过滤 | 3 | | |

### 今天必须能脱口而出的 5 个点

1. `where` 过滤行、`order by` 排序、`limit` 截取，执行顺序是 **from → where → order by → limit**。
2. `NULL` 只能用 `IS NULL / IS NOT NULL` 判断，`= NULL` 永远不成立（面试高频陷阱）。
3. `limit 10 offset 20` = 跳过 20 条取 10 条 = 第 3 页（每页 10 条）。
4. `count(*)` 数行数，`count(列)` **跳过该列的 NULL**，`count(distinct 列)` 去重后计数。
5. `and` 优先级高于 `or`，复杂条件一律加括号。

---

## 2. Day2（10/30）多表与聚合

对应清单：`group by / having` 会写；`inner join / left join` 会写且能说清区别；完成 10 道多表题；产出 3 条统计 SQL。

### 记录表

| 题号 | 考点 | 标准行数 | 我的答案 | 通过 |
|---|---|---|---|---|
| 1 | inner join | 42 | | 过 |
| 2 | left join（保留全部订单） | 45 | | 过 |
| 3 | group by 城市计数 | 9 | | 过 |
| 4 | join + where + sum + group by | 9 | | 过 |
| 5 | left join ... is null 找「没下过单的人」 | 3 | | 过 |
| 6 | group by 用户 + sum + top3 | 3 | | 过 |
| 7 | having 过滤分组 | 2 | | 过 |
| 8 | date() 按天分组 | 29 | | 过 |
| 9 | avg + round | 5 | | 过 |
| 10 | 多聚合函数并列 | 3 | | 过 |

### inner join 和 left join 的区别（面试原话，背下来）

> `inner join` 只保留两张表都能匹配上的行，是交集；`left join` 保留左表所有行，右表匹配不上就补 NULL。
>
> 拿这份数据说：45 条订单里，有 1 条 `user_id` 为 NULL、2 条 `user_id` 在用户表里不存在，所以 `inner join` 只剩 **42 行**，`left join` 仍是 **45 行**、这 3 行用户名为 NULL。反过来用 `left join` 从用户表出发，还能查出 28/29/30 这 **3 个一条订单都没有的用户**，这正是「查流失/未转化用户」的标准写法。

### 产出：3 条统计 SQL（已写在 [02_多表与聚合.sql](sql_phase1/02_多表与聚合.sql) 末尾，标准行数 26 / 9 / 5）

1. 按天统计已支付订单量 + GMV（口径：只算 `paid`，退款不计入）。
2. 每个城市的客单价（`sum/ count`，并说明「人均消费」时分母换成 `count(distinct user_id)`）。
3. 消费 TOP5 用户名单（`paid` → 按用户聚合 → 排序取前 5）。

> 面试讲这 3 条时，一定要先说**口径**（算不算退款、算不算未支付），再说 SQL。只会写 SQL 不说口径，是初级和会做业务的分界线。

### 今天必须能脱口而出的 3 个点

1. `where` 在聚合前过滤行，`having` 在聚合后过滤组；能放 `where` 的别放 `having`。
2. `select` 里出现的非聚合列，必须出现在 `group by` 里（否则 MySQL 报错或语义不确定）。
3. `not in (子查询)` 遇到 NULL 会整体返回空，所以子查询里要先 `where user_id is not null` 过滤。

### 课堂实录：今天踩的坑（10/30 实敲）

10 道题全部自己敲通，行数全部对上标准值。问题集中在下面几类，复习只看这张表：

| 类型 | 具体踩点 |
|---|---|
| 关键字拼写 | `join` 写成 `jion`；`order by` 写成 `orders by`；`order pr_da` 漏了 by |
| 别名 | 把字母 `o` 敲成数字 `0`，报错位置显示 `0.user_id` |
| 连表用错词 | 用 `and` 当连表关键字（`and` 是条件连接词，连表只能用 join 家族） |
| 子句顺序 | `having` 写到 `group by` 前面；`desc` 跟在 `group by` 后面 |
| 漏子句/漏符号 | 第二行漏 `from`；`select` 两列之间漏逗号 |
| 分组规则 | 分组后 select 里写未聚合的 `amount`；`group by` 的列没写进 `select` |
| 口径意识 | 题6 自己加了 `where status = 'paid'`，口径变了但当时没意识到 |

三条当铁律记：

1. 子句顺序固定：`select → from(join on) → where → group by → having → order by → limit`
2. 分组后 select 的每一列，要么在 `group by` 里，要么套聚合函数
3. 写统计 SQL 先讲口径，再写语句

### 当前进度（暂停点）

- 题1~10：已全部跑通并核对行数
- 产出 1（按天 GMV）：**未开始**
- 产出 2（城市客单价）：未开始
- 产出 3（消费 TOP5 用户）：未开始

> 恢复学习时，从 [02_多表与聚合.sql](sql_phase1/02_%E5%A4%9A%E8%A1%A8%E8%81%9A%E5%90%88.sql) 末尾「产出要求」那三条开始，一条一条自己写，标准行数 26 / 9 / 5。

---

## 3. Day3（10/31）索引与慢查询

对应清单：解释索引是什么、为什么快；会用 `explain` 看 `type`、`key`；说出 3 个慢查询原因；在自己库上 explain 3 条 SQL；产出 3 句问答。

### 3 个 explain 实操（都在 [03_索引与慢查询.sql](sql_phase1/03_索引与慢查询.sql) 里，按顺序执行就能看到对比）

| 实验 | 优化前 | 优化后 |
|---|---|---|
| 1 城市查询 | `type=ALL`、`key=NULL`（全表扫） | 建 `idx_city` 后 `type=ref`、`key=idx_city`；只查 city 列时 Extra 出现 `Using index`（覆盖索引） |
| 2 索引失效 | `amount * 2 > 400` → `type=ALL` | 同列改写成 `amount > 200` → `type=range`、`key=idx_amount` |
| 3 排序 | `order by created_at` → `Using filesort` | 建 `idx_created_at` 后 filesort 消失 |

另外两个「索引失效」的标准例子：`like '%伟%'`（% 在前）和 `left(city, 2)`（函数包裹索引列）。

### 产出：3 句问答（照抄进笔记，务必能背）

**Q1：索引是什么，为什么能加速查询？**
A：索引是建在列上的 B+ 树有序结构，相当于给这列数据建了一本目录；查询时逐层缩小范围，把全表 O(n) 扫描降到近似 O(log n)，不用一行行比对整张表。

**Q2：你怎么判断一条 SQL 慢在哪？**
A：先用慢查询日志（`slow_query_log` + `long_query_time`）找出它，再用 `explain` 看 `type`、`key`、`rows`：`type=ALL` 或 `key=NULL` 说明没走索引；`rows` 很大说明扫描量大；Extra 里出现 `Using filesort` / `Using temporary` 说明排序或分组有额外开销。

**Q3：建了索引还是慢，可能是什么原因？**
A：大概率是索引失效——对索引列做运算或套函数、用了 `like '%xx%'`、发生隐式类型转换、组合索引没走最左前缀；也可能是数据量太大、返回行太多，那要改查询（限列、分页）而不是继续加索引。

> 加分句：索引不是越多越好，它会占空间并拖慢增删改，因为写数据时要同步维护索引。

---

## 4. 三天防忘 checklist（离开电脑前过一遍）

- [ ] 我能不看答案写出：`where` + `order by` + `limit` 的查询
- [ ] 我能说清 `is null` 为什么不能写成 `= null`
- [ ] 我能说清 `inner join` 和 `left join` 的区别，并用这份数据的 42 / 45 行举例
- [ ] 我能说清 `where` 和 `having` 的区别
- [ ] 我能用 `explain` 说出 `type`、`key`、`rows`、`Extra` 各代表什么
- [ ] 我能背出 3 个慢查询原因 + 3 句问答
- [ ] 20 + 10 道题已全部写进上面的记录表（这就是「SQL 练习记录」产出）
