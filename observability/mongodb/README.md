# MongoDB 数据库监控

本文只覆盖 MongoDB 服务、性能、存储和集群指标；操作系统和网络指标见[五层监控总览](../database-monitoring.md)。示例适用于副本集，分片集群还需分别监控 `mongos`、Config Server 和各分片。

> 阈值仅供起步，应按业务延迟目标、历史基线及数据恢复要求调整。采集账号应遵循最小权限原则；不同版本和部署方式提供的字段可能不同。


在 `mongosh` 中连接目标节点后执行。`<collection>`、`<filter>` 为占位符；指标字段随版本变化，未提供的字段需查阅目标版本文档。

## Prometheus 与 Node Exporter

参见 [MongoDB Prometheus 采集配置](prometheus.md)，包括 Node Exporter、各副本成员的数据库 exporter、ServiceMonitor 和可采集指标。

## 数据库服务层（Availability / Connections）

| 指标 | 查看方式 | 关注点与告警建议 |
|---|---|---|
| 服务可用性 | 从业务网络连接并运行 `db.runCommand({ ping: 1 })` | 连续探测失败或超时及时告警；同时覆盖业务实际使用的入口 |
| 连接数 | `db.serverStatus().connections` | 查看 `current`、`available`；当前连接数接近实例连接上限时告警 |
| 连接建立与拒绝 | 驱动或连接池指标、服务器日志 | 连接超时、拒绝及建立速率异常上升时排查连接泄漏或连接风暴 |
| 节点角色 | `rs.status()` | 副本集主节点不可用、成员异常或频繁选举时告警 |

`connections.available` 是服务器报告的可用连接数，其含义受版本和部署限制影响；应结合驱动连接池及实际连接上限判断。

### 连接数和连接池

`connections.current` 是当前打开的连接，`connections.available` 是服务器报告的剩余连接容量。持续采集两者，并结合驱动连接池的使用中连接、等待队列、创建和超时次数判断：连接数高但活跃操作少，可能是连接池过大或连接未释放；连接数和新建速率同时快速上升，则应检查应用重启、重试和连接风暴。可用连接数降低时也要检查实例及系统限制，不应只调整连接池上限。

```javascript
const s = db.serverStatus();
printjson(s.connections);
printjson(s.metrics && s.metrics.cursor);
```

### 指标与命令对照

| 监控内容 | 命令或查询 | 说明 |
|---|---|---|
| 可用性 | `db.runCommand({ ping: 1 })` | 对业务实际连接入口执行 |
| 连接数和可用连接 | `db.serverStatus().connections` | 查看 `current`、`available`；连接拒绝及池状态还需查驱动指标或服务器日志 |
| 连接错误日志 | 在服务器上执行 `grep -iE 'connection.*(refused\|failed\|timeout)' <mongod日志路径>` | 日志格式及关键词依版本变化，托管服务需使用其日志平台 |

## 数据库性能层（Performance）

| 指标 | 查看方式 | 关注点与告警建议 |
|---|---|---|
| 操作吞吐量 | `db.serverStatus().opcounters` | 对 `insert`、`query`、`update`、`delete`、`command` 等累计计数求每秒增量；突增突降时结合业务流量排查 |
| 响应时间 | 应用/驱动请求耗时分布 | 监控 P95、P99 和窗口内最慢请求；持续超过业务 SLO 时告警 |
| 慢操作 | 慢查询日志、Profiler（按需启用） | 慢操作数量或占比高于基线时，按命名空间和操作类型定位 |
| 游标 | `db.serverStatus().metrics.cursor` | 关注打开数量及超时数量的变化；持续增长时排查未关闭或长时间闲置的游标 |
| 锁和等待 | `db.serverStatus().locks`、`db.currentOp()` | 等待持续增加或关键操作长时间阻塞时排查；注意字段与权限随版本变化 |

`opcounters` 是服务端操作计数，不一定等于应用请求数；重启后累计值可能重置。Profiler 有性能和敏感数据暴露风险，应根据需要选择采样及保留策略。

### 吞吐量、慢操作和查询计划

`opcounters` 是从统计开始以来的累计操作数。对同一节点两次采集，使用 `(第二次计数 - 第一次计数) / 间隔秒数` 计算速率；重启导致计数下降时丢弃该窗口。操作类型的口径与应用请求不一定一一对应，应同时对照应用流量和延迟。

发现慢操作后，先按集合、操作类型和时间范围聚合日志，再用实际过滤条件、排序和投影分析执行计划：

```javascript
db.orders.find({ customerId: "example" }).explain("executionStats")
```

重点比较 `nReturned`、`totalKeysExamined` 和 `totalDocsExamined`。扫描量远大于返回量时，检查索引选择性、复合索引顺序和查询条件；返回量为零时不要直接计算比值。测试前替换为真实查询并评估执行开销，`executionStats` 会执行获胜计划。日志或 profiler 可能包含敏感查询内容，应控制权限和保留时间。

### 按步骤排查性能问题

先记录业务受影响的时间、接口、数据库入口和错误信息，再将应用 P95/P99 与对应时间段的操作速率、CPU、磁盘延迟对齐。下面的命令在连接到目标成员的 `mongosh` 中执行；如需比较主备，分别连接并标记节点，不能将不同节点的累计计数相减。

**查看操作速率。**间隔十秒执行以下命令两次，并记录两次实际时间：

```javascript
print(new Date().toISOString());
printjson(db.serverStatus().opcounters);
```

例如 `command` 从 100000 增至 105000、实际间隔 10 秒，则该窗口约 500 次/秒。分别计算其他字段；计数下降或进程重启时丢弃窗口。操作速率上升且延迟上升时，检查是否有流量突增；速率下降而延迟上升时，进一步检查等待、慢操作和资源。

**查找正在运行的慢操作。**在 `admin` 数据库执行：

```javascript
db.getSiblingDB("admin").aggregate([
  { $currentOp: { allUsers: true, idleConnections: false } },
  { $match: { active: true, secs_running: { $gte: 5 } } },
  { $sort: { secs_running: -1 } },
  { $limit: 20 },
  { $project: { opid: 1, secs_running: 1, ns: 1, command: 1,
                waitingForLock: 1, locks: 1 } }
]);
```

从命名空间、操作类型和 `secs_running` 找到受影响的操作；`waitingForLock` 为真时结合 `locks`、日志和其他操作检查争用。它是瞬时快照，执行结束后不会保留；完整结果及跨用户查看取决于权限、版本与部署方式。不要仅凭运行时间直接终止操作，先确认业务及事务影响。

**分析慢查询和执行计划。**从慢操作日志或已启用 profiler 的数据库中找出时间段及查询条件；在可控环境使用实际过滤条件、排序和投影执行前面的 `explain("executionStats")`。`COLLSCAN` 可能意味着集合扫描；检查 `totalDocsExamined`、`totalKeysExamined` 与 `nReturned`，同时考虑查询本来就需要返回大量数据的情况。调整索引前先检查写入、存储和现有查询的影响，变更后在同类数据和负载下复测。

**检查游标和等待趋势。**隔一段时间重复执行 `db.serverStatus().metrics.cursor` 和 `db.serverStatus().locks`：观察打开游标是否持续增长、超时累计数是否增加，以及锁等待次数和时间是否加速增长。记录采样时间、节点与统计重置；持续增长时结合应用游标关闭、慢操作和资源等待定位。

### 指标与命令对照

| 监控内容 | 命令或查询 | 说明 |
|---|---|---|
| 操作数及吞吐量 | `db.serverStatus().opcounters` | 两次采集各字段，增量除以采样秒数 |
| 响应时间、P95/P99、最慢请求 | 按[响应时间](../database-monitoring.md#响应时间)查询应用直方图和日志 | 服务端操作计数不能计算请求分位数 |
| 慢操作日志 | `grep -i 'slow query' <mongod日志路径>` | 不同版本日志格式不同；也可查询已启用的 profiler，见下方 |
| 游标 | `db.serverStatus().metrics.cursor` | 查看打开和超时游标；累计超时计数需计算增量 |
| 锁与当前操作 | `db.serverStatus().locks`；`db.currentOp({ active: true })` | 对锁等待等累计值计算增量，并检查长时间运行的操作 |

已启用数据库 profiler 时，可查看最近记录；不要仅为执行此查询而直接开启全量 profiler：

```javascript
db.getCollection("system.profile").find().sort({ ts: -1 }).limit(10)
```

## 数据与存储层（Data / Storage）

| 指标 | 查看方式 | 关注点与告警建议 |
|---|---|---|
| 集合及索引大小 | `db.collection.stats()` | 关注数据、存储和索引大小及增长率；预计耗尽可用空间时告警 |
| 索引使用 | `$indexStats`、`explain("executionStats")` | 对重要查询检查扫描量与返回量、执行计划和索引使用；不要仅凭低使用量删除索引 |
| WiredTiger 缓存 | `db.serverStatus().wiredTiger.cache` | 关注缓存使用量、dirty bytes、驱逐和等待；持续接近容量且延迟上升时排查 |
| Oplog 空间与覆盖窗口 | `rs.printReplicationInfo()`、`local.oplog.rs` 相关统计 | 关注 oplog 增长及可覆盖时间；副本落后接近覆盖窗口时及时处理，避免需要重新全量同步 |

示例：

```javascript
db.runCommand({ ping: 1 })
db.serverStatus().connections
db.serverStatus().opcounters
db.serverStatus().wiredTiger.cache
db.orders.stats()
db.orders.aggregate([{ $indexStats: {} }])
```

部分字段仅在使用 WiredTiger 时存在。`$indexStats` 等统计有节点和重启生命周期限制；对查询计划分析要结合真实查询和参数。

### WiredTiger 缓存和存储

比较缓存当前字节数与配置上限，并观察脏页、驱逐活动、应用线程等待和请求延迟的变化。缓存使用率高本身可能是正常缓存利用；若脏页或驱逐等待持续增加，同时写延迟和磁盘延迟升高，应检查工作集、写入速率及存储性能。字段名称和可用性随版本变化，不要将某个固定字段用于所有版本。

集合的 `size` 表示逻辑数据大小，`storageSize` 是为集合分配的存储空间，`totalIndexSize` 是索引空间。比较趋势时应区分逻辑数据增长和物理空间增长，并同时检查文件系统剩余空间；删除数据后文件系统空间不一定立即下降。

### 指标与命令对照

| 监控内容 | 命令或查询 | 说明 |
|---|---|---|
| 集合、索引大小 | `db.getCollection("<collection>").stats()` | 查看 `size`、`storageSize`、`totalIndexSize` |
| 索引使用及查询计划 | `db.getCollection("<collection>").aggregate([{ $indexStats: {} }])`；`db.getCollection("<collection>").find(<filter>).explain("executionStats")` | 查看索引访问和扫描量；应使用真实查询条件 |
| WiredTiger 缓存及驱逐 | `db.serverStatus().wiredTiger.cache` | 比较使用量、dirty bytes、驱逐和等待相关字段 |

## 集群层（HA / Replication）

| 指标 | 查看方式 | 关注点与告警建议 |
|---|---|---|
| 成员状态和选举 | `rs.status()`、日志 | 主节点不可用、成员长期不健康或频繁选举时告警 |
| 复制进度与延迟 | `rs.status()`；版本支持时使用 `rs.printSecondaryReplicationInfo()` | 副本持续落后、复制中断或延迟超过业务允许值时告警 |
| Oplog 覆盖窗口 | `rs.printReplicationInfo()` | 比较最落后副本与可用 oplog 窗口；接近窗口上限时告警 |
| 分片状态 | `sh.status()`、各分片与 Config Server 状态 | 分片或配置副本集不可用、数据倾斜或迁移长期失败时排查 |

复制延迟为缺失或无法计算时不能视为零；无写入期间应结合复制状态和最新操作时间判断。分片环境中应分别观察各副本集，不能只看 `mongos` 是否在线。

### 副本延迟与 Oplog 窗口

在各成员检查 `rs.status()` 中的健康状态、角色和复制进度；将最落后副本的进度与主节点及 oplog 最早记录比较。副本落后的时间接近 oplog 覆盖窗口时，短暂故障可能导致其无法继续增量追赶。排查时区分网络传输不足、磁盘写入慢和副本负载过高；无新写入或节点不可达时，缺失时间差不等于零延迟。

### 分片集群

从业务入口探测 `mongos`，并分别监控 Config Server 和各分片副本集。按分片比较数据量、请求负载和容量增长；不均衡可能表现为某个分片延迟高或空间先耗尽。迁移失败或长期积压时检查 balancer 状态、集群事件与日志，不要只依据 `mongos` 进程是否存活判断集群健康。

### 指标与命令对照

| 监控内容 | 命令或查询 | 说明 |
|---|---|---|
| 副本集成员及复制 | `rs.status()`；`rs.printSecondaryReplicationInfo()` | 查看成员状态及复制进度；后者需目标版本支持 |
| Oplog 窗口 | `rs.printReplicationInfo()` | 查看覆盖时间，并与最落后副本对照 |
| 分片状态 | `sh.status()` | 在 `mongos` 执行，并分别检查各分片及 Config Server 副本集 |
| 选举、切换和迁移错误 | `grep -iE 'election\|stepdown\|migration.*(fail\|error)' <mongod日志路径>` | 关键词依版本变化；结合集群监控事件检查 |
