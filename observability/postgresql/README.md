# PostgreSQL 数据库监控

本文覆盖 PostgreSQL 数据库内部监控；主机和网络指标见[五层监控总览](../database-monitoring.md)。示例 SQL 按需在目标实例执行，监控账号应遵循最小权限原则；部分统计需要额外权限或扩展，字段也会随版本变化。

> 下列阈值仅供起步；采集累计计数器时应计算采样间隔内的增量，并处理统计重置。延迟和吞吐量需要结合业务目标及历史基线判断。


以下 SQL 可在 `psql` 中执行；采集两次后计算累计字段的差值，并检查 `stats_reset`。连接示例中的地址、数据库和账号按实际环境替换，密码使用安全的凭据管理方式。

## Prometheus 与 Node Exporter

参见 [PostgreSQL Prometheus 采集配置](prometheus.md)，包括 Node Exporter、CloudNativePG 实例指标、PodMonitor 和可采集指标。

## 数据库服务层（Availability / Connections）

| 指标 | 查看方式 | 关注点与告警建议 |
|---|---|---|
| 连接及查询探测 | 从业务网络连接目标实例并执行 `SELECT 1` | 连续失败或超时时及时告警 |
| 连接数量和状态 | `pg_stat_activity`、`max_connections` | 连接使用率 > 80% 警告、> 90% 严重告警；留意预留连接和连接池 |
| 活跃及等待会话 | `pg_stat_activity.state`、`wait_event_type`、`wait_event` | 活跃会话或等待持续高于基线时，结合等待类型定位 |
| 长事务 | `pg_stat_activity.xact_start` | 事务持续超出业务允许时长，尤其是长期 `idle in transaction`，应排查 |

```sql
SELECT state, count(*) AS sessions
FROM pg_stat_activity
GROUP BY state
ORDER BY sessions DESC;

SHOW max_connections;
```

统计视图可能包含后台进程，计算业务连接使用率时应按目标版本和连接类型确定分子、分母，并考虑管理员预留连接。

### 连接、等待与长事务

连接总数接近上限时，先区分活跃连接、空闲连接和 `idle in transaction`，并核对连接池与预留额度。空闲连接多不等于数据库忙；活跃会话有等待事件时，应进一步区分锁、IO 和其他等待。长事务可能阻止旧行版本清理，即使当前会话没有执行查询也要关注。

```sql
SELECT pid, usename, application_name, state,
       now() - xact_start AS transaction_age,
       wait_event_type, wait_event, left(query, 160) AS query
FROM pg_stat_activity
WHERE backend_type = 'client backend' AND xact_start IS NOT NULL
ORDER BY xact_start;
```

先联系事务所属应用并确认业务影响，再处理异常会话；监控账号能看到的查询文本受权限限制。

### 指标与命令对照

| 监控内容 | 命令或查询 | 说明 |
|---|---|---|
| 端到端可用性 | `psql "host=<地址> port=5432 dbname=<数据库> user=<账号> connect_timeout=3" -c "SELECT 1"` | 从业务侧执行并检查退出状态 |
| 活跃连接及等待 | `SELECT state, wait_event_type, wait_event, count(*) FROM pg_stat_activity WHERE backend_type = 'client backend' GROUP BY 1, 2, 3;` | 与 `SHOW max_connections;` 和预留额度一起看；字段以目标版本为准 |
| 长事务 | `SELECT pid, state, now() - xact_start AS age, wait_event_type, wait_event FROM pg_stat_activity WHERE xact_start IS NOT NULL ORDER BY xact_start;` | 重点关注长时间空闲事务 |

## 数据库性能层（Performance）

| 指标 | 查看方式 | 关注点与告警建议 |
|---|---|---|
| 事务吞吐量 | `pg_stat_database.xact_commit`、`xact_rollback` | 对累计值求每秒增量；区分提交和回滚，异常变化时结合业务流量判断 |
| SQL 吞吐与耗时 | `pg_stat_statements` 扩展 | 按调用次数、总耗时和平均耗时定位高负载 SQL；需按版本配置和启用扩展 |
| P95/P99 与最慢请求 | 应用或监控系统的请求耗时分布、慢查询日志 | 持续超过业务 SLO 或请求超时时告警；平均值不能代替分位数 |
| 锁等待 | `pg_stat_activity`、`pg_locks`、`pg_blocking_pids()` | 等待时间超出业务容忍值或阻塞链持续增长时告警 |
| 死锁 | `pg_stat_database.deadlocks` | 对累计计数求增量，出现死锁时记录并分析相关事务 |

```sql
SELECT pid, now() - query_start AS query_duration,
       wait_event_type, wait_event, pg_blocking_pids(pid) AS blockers
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;
```

事务吞吐量不等于 QPS。`pg_stat_statements` 汇总统计并不直接提供每次请求的精确 P95/P99；需从应用或其他支持耗时分布的监控系统采集。

### 吞吐量、慢 SQL 与阻塞

分别计算 `xact_commit`、`xact_rollback` 在采样窗口内的增量除以秒数；回滚率需要结合业务设计，不能将每次回滚都当作故障。比较 `stats_reset`，统计重置或实例重启后不要直接计算负速率。

`pg_stat_statements` 适合寻找总耗时高或调用频繁的 SQL。比较两次采样的调用和耗时增量，才能定位当前窗口的负载；累计排名可能被历史流量主导。再结合实际参数、执行计划和应用耗时检查索引、行数估计及 IO。获取执行计划时，`EXPLAIN ANALYZE` 会实际执行语句，尤其需要注意写操作的影响。

使用 `pg_blocking_pids(pid)` 找阻塞源，并结合等待持续时间、事务开始时间和 SQL 判断影响。死锁计数增长说明数据库曾检测并中止事务，应结合日志分析涉及的对象和访问顺序。

### 按步骤排查性能问题

先确定业务受影响的时间、数据库和实例，对照应用 P95/P99、吞吐量、错误率以及节点 CPU、磁盘延迟。以下 SQL 在目标实例上执行；统计视图权限和字段以 PostgreSQL 版本为准。

**计算事务速率。**间隔十秒执行两次：

```sql
SELECT clock_timestamp() AS sampled_at, datname,
       xact_commit, xact_rollback, deadlocks, stats_reset
FROM pg_stat_database
WHERE datname = current_database();
```

以每个计数的增量除以实际秒数得到提交、回滚及死锁速率。两次 `stats_reset` 不同或计数下降时，不计算该窗口。提交下降而请求延迟上升时，继续查看等待；回滚增长时结合业务错误日志，不将所有回滚直接判为故障。

**寻找当前等待和阻塞源。**

```sql
SELECT a.pid, a.usename, a.application_name, a.state,
       clock_timestamp() - a.query_start AS query_age,
       a.wait_event_type, a.wait_event,
       pg_blocking_pids(a.pid) AS blocker_pids,
       left(a.query, 200) AS query
FROM pg_stat_activity AS a
WHERE a.backend_type = 'client backend' AND a.state = 'active'
ORDER BY a.query_start;
```

`wait_event_type = 'Lock'` 且 `blocker_pids` 非空时，按 PID 查询 `pg_stat_activity` 的阻塞者，检查其 `xact_start`、状态和 SQL。`query_age` 是查询持续时间，不是精确的锁等待时长；要测量等待持续时间，需要持续采样或查看适用版本的锁信息。处理前先确认阻塞事务的业务归属及回滚影响。

**定位高负载 SQL。**已启用 `pg_stat_statements` 时，两次采集相同 `queryid` 的 `calls` 和 `total_exec_time`，对照增量；该查询按累计耗时排列，仅用于筛选候选：

```sql
SELECT queryid, calls, round(total_exec_time::numeric, 1) AS total_ms,
       round(mean_exec_time::numeric, 1) AS mean_ms, left(query, 200) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

对窗口内调用频繁或单次耗时上升的查询，用真实参数在可控环境执行 `EXPLAIN (ANALYZE, BUFFERS) SELECT ...`。比较估算行数与实际行数、扫描方式、循环次数和缓冲区读取；`ANALYZE` 会实际执行语句，不要对生产写操作随意运行。索引、统计信息或 SQL 调整后，用相同时间窗口复测延迟、吞吐量和资源。

### 指标与命令对照

| 监控内容 | 命令或查询 | 说明 |
|---|---|---|
| TPS、回滚和死锁 | `SELECT datname, xact_commit, xact_rollback, deadlocks, stats_reset FROM pg_stat_database;` | 分别对提交、回滚和死锁计数计算速率 |
| SQL 调用和耗时 | `SELECT queryid, calls, total_exec_time, mean_exec_time, query FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;` | 需安装并配置扩展；字段名称随版本变化，比较采样窗口增量 |
| P95/P99、最慢请求 | 按[响应时间](../database-monitoring.md#响应时间)查询应用直方图和日志 | `pg_stat_statements` 均值不能还原精确分位数 |
| 慢查询日志 | `SHOW log_min_duration_statement;`；在服务器执行 `grep 'duration:' <PostgreSQL日志路径>` | 需已配置慢查询日志；日志输出位置和格式以实际配置为准 |
| 锁及阻塞 | `SELECT pid, wait_event_type, wait_event, pg_blocking_pids(pid) FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;` | 结合上方长事务查询判断阻塞时长和影响 |

## 数据与存储层（Data / Storage）

| 指标 | 查看方式 | 关注点与告警建议 |
|---|---|---|
| 数据库、表及索引大小 | `pg_database_size()`、`pg_total_relation_size()` | 监控增长率和剩余空间，预计耗尽时提前告警 |
| 缓存读取 | `pg_stat_database.blks_hit`、`blks_read` | 对采样窗口内增量计算命中率，持续偏离基线且 IO 延迟上升时排查 |
| 表和索引访问 | `pg_stat_user_tables`、`pg_stat_user_indexes` | 结合查询计划、扫描量和业务周期判断索引是否有效；不要仅凭低扫描量删除索引 |
| 死元组及清理 | `pg_stat_user_tables.n_dead_tup`、VACUUM 相关统计 | 关注估算值和变化趋势；清理积压持续增长并影响空间或性能时排查 |
| 事务 ID 年龄 | `age(datfrozenxid)`、autovacuum 状态 | 接近反卷绕保护阈值前按版本和配置提前告警；及时排查 VACUUM 障碍 |
| Checkpoint | `pg_stat_bgwriter`；较新版本另见 `pg_stat_checkpointer` | 检查 checkpoint 频率、耗时和写入压力；统计字段随版本变化 |

```sql
SELECT datname, pg_size_pretty(pg_database_size(datname)) AS size,
       age(datfrozenxid) AS xid_age
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

SELECT relname, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 10;
```

表统计是估算值；`blks_hit` 不包含所有 OS 文件缓存命中，不能仅凭固定的 99% 阈值判断内存是否充足。XID 年龄告警阈值应留足维护时间，不能等到数据库接近强制保护才处理。

### 缓存、表膨胀与 VACUUM

数据库缓存命中率可按窗口增量计算：`Δblks_hit / (Δblks_hit + Δblks_read)`；分母为零时显示无数据。`blks_read` 是 PostgreSQL 向操作系统请求读取的块数，可能由操作系统缓存满足，并不必然代表物理磁盘读取。命中率下降时还应检查工作集、查询计划和磁盘延迟。

`n_dead_tup` 是估算值，不等于实际膨胀大小。持续增长时，检查 autovacuum 是否运行、是否被长事务或复制槽阻碍，以及表的更新量和参数；结合表大小与清理进度判断。`age(datfrozenxid)` 接近配置阈值时要预留足够处理时间，尤其关注长事务和未完成的反卷绕 VACUUM。

```sql
SELECT pid, datname, relid::regclass AS relation, phase,
       heap_blks_total, heap_blks_scanned, heap_blks_vacuumed
FROM pg_stat_progress_vacuum;
```

进度视图中的字段依版本而异；没有行可能表示当前没有符合条件的 VACUUM，而非历史上没有运行。

### 指标与命令对照

| 监控内容 | 命令或查询 | 说明 |
|---|---|---|
| 表、索引大小 | `SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) FROM pg_stat_user_tables ORDER BY pg_total_relation_size(relid) DESC LIMIT 10;` | 总大小包含表、索引及相关 TOAST 数据 |
| 索引使用 | `SELECT relname, indexrelname, idx_scan FROM pg_stat_user_indexes ORDER BY idx_scan;` | 检查统计重置时间和业务周期后再评估 |
| 缓存读取 | `SELECT datname, blks_hit, blks_read, stats_reset FROM pg_stat_database;` | 命中率使用窗口内 `hit / (hit + read)`；无读取时不计算 |
| 死元组与 VACUUM | `SELECT relname, n_dead_tup, last_autovacuum, last_autoanalyze FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;` | 统计为估算值，结合增长趋势判断 |
| XID 年龄与阈值 | `SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY age(datfrozenxid) DESC;`；`SHOW autovacuum_freeze_max_age;` | 提前告警并检查各表的实际配置和清理进度 |
| Checkpoint | `SELECT * FROM pg_stat_bgwriter;`；支持时 `SELECT * FROM pg_stat_checkpointer;` | `pg_stat_checkpointer` 在 PostgreSQL 17 起提供；按版本选用字段 |

## 集群层（HA / Replication）

| 指标 | 查看方式 | 关注点与告警建议 |
|---|---|---|
| WAL 生成与空间 | `pg_stat_wal`（版本支持时）、WAL 所在文件系统 | WAL 生成速率或占用异常增长、预计耗尽空间时告警 |
| 归档状态 | `pg_stat_archiver`、归档目标状态 | 归档失败数增加或归档积压持续增长时告警 |
| 主库复制状态 | `pg_stat_replication` | 关注副本连接、发送和回放进度，以及写入、持久化、回放延迟 |
| 复制槽保留 | `pg_replication_slots` | 失效或长期未推进的槽可能保留 WAL；积压持续增长时排查 |
| 备库恢复状态 | `pg_is_in_recovery()`、`pg_last_wal_replay_lsn()` | 副本停止回放或持续落后超过业务允许值时告警 |

```sql
SELECT application_name, state, sync_state, write_lag, flush_lag, replay_lag
FROM pg_stat_replication;

SELECT slot_name, slot_type, active, restart_lsn
FROM pg_replication_slots;
```

`pg_stat_replication` 反映直接连接到当前主库的副本；延迟字段可能为 `NULL`，不能当作零，也不能在无写入时仅凭该值判断副本健康。应同时检查连接、LSN 进度、归档状态和 WAL 保留量。

### WAL、归档和复制槽

WAL 容量排查同时看生成速率、实际文件系统剩余空间、归档失败和复制槽保留。以下在主库估算每个有 `restart_lsn` 的槽距当前 WAL 写入位置的字节差；多个槽可能保留相同 WAL，不能将结果相加作为实际磁盘占用。

```sql
SELECT slot_name, active, restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_distance
FROM pg_replication_slots
WHERE restart_lsn IS NOT NULL;
```

检查槽所属消费者是否仍然需要，以及是否有长时间未推进的复制或归档。删除槽可能影响消费者恢复，应先确认依赖。主库 `pg_stat_replication` 的延迟字段可能为空；应同时观察发送和回放位置、连接状态以及是否有新写入。

### 指标与命令对照

| 监控内容 | 命令或查询 | 说明 |
|---|---|---|
| WAL 生成 | `SELECT * FROM pg_stat_wal;` | PostgreSQL 14 起提供；对生成字节数计算增量 |
| WAL 空间 | `SHOW data_directory;`；在服务器检查 `df -h <实际WAL目录>`、`du -sh <实际WAL目录>` | WAL 目录可能通过符号链接位于其他文件系统 |
| 归档 | `SELECT * FROM pg_stat_archiver;` | 关注失败计数增量和最近成功归档时间 |
| 副本、复制槽 | `SELECT * FROM pg_stat_replication;`；`SELECT * FROM pg_replication_slots;` | 分别检查进度、状态和槽的 WAL 保留 |
| 备库回放 | `SELECT pg_is_in_recovery(), pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn(), pg_last_xact_replay_timestamp();` | 结合主库写入和连接状态判断；空闲期间时间戳旧不一定是故障 |
| 切换事件 | 在服务器执行 `grep -iE 'promot\|failover\|standby' <PostgreSQL日志路径>` | 还需检查实际 HA 管理器的事件日志 |
