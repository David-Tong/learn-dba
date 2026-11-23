# Oracle 数据库监控

本文覆盖 Oracle 数据库内部监控；主机和网络指标见[五层监控总览](../database-monitoring.md)。以下以常见单实例视图为例；RAC 和多租户环境还需按实例、服务和容器分别观察。不同版本、权限和部署方式的字段可能有差异。

> 动态性能视图及 DBA 视图需要相应权限，监控账号应遵循最小权限原则。AWR、ASH、ADDM 和部分 SQL 监控功能可能涉及额外许可；使用相关功能或视图前应核对具体版本及授权，不要默认已获许可。


以下 SQL 可在 `SQL*Plus` 或兼容客户端中执行。先使用安全的凭据管理方式连接实际业务服务；`<...>` 为占位符。采集累计统计时比较两次采样的增量，并考虑实例重启或游标淘汰。

## Prometheus 与 Node Exporter

参见 [Oracle Prometheus 采集配置](prometheus.md)，包括 Node Exporter、Oracle 数据库 exporter、ServiceMonitor 和可采集指标。

## 数据库服务层（Availability / Connections）

| 指标 | 查看方式 | 关注点与告警建议 |
|---|---|---|
| 服务可用性 | 经业务服务名连接并执行 `SELECT 1 FROM dual` | 连续失败或超时时及时告警；CDB/PDB 环境应探测实际业务 PDB |
| 实例与数据库状态 | `V$INSTANCE`、`V$DATABASE` | 实例状态或数据库打开模式不符合预期时告警 |
| 会话和进程额度 | `V$RESOURCE_LIMIT`、`V$SESSION` | 会话或进程使用率接近配置上限时告警，如 > 80% 警告、> 90% 严重告警 |
| 活跃与等待会话 | `V$SESSION` | 活跃会话和等待会话持续高于基线时，结合等待事件排查 |
| PDB 状态 | `V$PDBS` | 业务 PDB 未按预期打开或服务不可用时告警 |

```sql
SELECT instance_name, status FROM v$instance;
SELECT name, open_mode, database_role FROM v$database;
SELECT resource_name, current_utilization, limit_value
FROM v$resource_limit
WHERE resource_name IN ('processes', 'sessions');
```

`LIMIT_VALUE` 可能不是普通数字，计算使用率前应检查其值；RAC 中需按实例分别检查，避免仅看连接到的一个实例。

### 实例、服务和会话额度

实例为 `OPEN` 不能证明业务服务一定可用。应从业务网络使用实际服务名连接，检查目标 PDB 是否打开并执行轻量查询；RAC 还要分别检查服务与各实例。连接失败时依次核对监听、服务注册、网络、认证和会话额度。

`V$RESOURCE_LIMIT` 的 `CURRENT_UTILIZATION` 和 `LIMIT_VALUE` 可用于检查 `processes`、`sessions`；上限可能不是数字，转换前要判断。结合 `MAX_UTILIZATION` 观察实例启动以来的峰值，再对照当前连接池和实际配置，避免只看某一时刻的连接数。

### 指标与命令对照

| 监控内容 | 命令或查询 | 说明 |
|---|---|---|
| 业务服务可用性 | `SELECT 1 FROM dual;` | 从业务侧连接目标服务后执行；同时检查连接或查询错误 |
| 实例、角色及 PDB | `SELECT instance_name, status FROM v$instance;`；`SELECT name, open_mode, database_role FROM v$database;`；`SELECT name, open_mode FROM v$pdbs;` | 最后一项适用于多租户环境 |
| 会话、进程额度 | `SELECT resource_name, current_utilization, limit_value FROM v$resource_limit WHERE resource_name IN ('processes', 'sessions');` | 在各实例执行，或按需使用 `GV$RESOURCE_LIMIT` 和 `INST_ID` |
| 活跃及等待会话 | `SELECT sid, serial#, event, wait_class, seconds_in_wait FROM v$session WHERE type = 'USER' AND status = 'ACTIVE';` | 排查持续等待，并区分空闲等待 |

## 数据库性能层（Performance）

| 指标 | 查看方式 | 关注点与告警建议 |
|---|---|---|
| 事务吞吐量 | `V$SYSSTAT` 的提交和回滚统计 | 对累计计数求每秒增量，提交下降或回滚异常增加时排查 |
| SQL 负载 | `V$SQL` 中执行次数、累计耗时等 | 按采样窗口增量定位耗时或执行频繁的 SQL；注意共享池及统计生命周期 |
| P95/P99 与最慢请求 | 应用或监控系统的请求耗时分布 | 持续超过业务 SLO 或出现超时时告警；不要用平均耗时代替分位数 |
| 等待事件 | `V$SESSION`、`V$SYSTEM_EVENT` | 对等待时间和次数求增量，找出持续增加的非空闲等待 |
| 锁与阻塞 | `V$SESSION`、`V$LOCK` | 阻塞时间超过业务容忍值、阻塞链持续增长或发生死锁时排查 |

```sql
SELECT name, value
FROM v$sysstat
WHERE name IN ('user commits', 'user rollbacks');

SELECT sid, serial#, status, event, wait_class, blocking_session
FROM v$session
WHERE type = 'USER' AND status = 'ACTIVE';
```

QPS 的定义和采集口径应与应用侧一致；单独的提交计数不是 QPS。等待事件要排除空闲等待，并结合采样窗口及业务影响分析。

### SQL、等待和阻塞

`V$SQL` 的执行次数和耗时是游标生命周期内的累计值；`ELAPSED_TIME` 以微秒计，包含等待时间。比较相同游标两次采样的增量，并留意游标被淘汰或重新加载。总耗时高可能来自高频调用，也可能来自单次执行缓慢，应结合执行次数、计划及业务时段分析。

```sql
SELECT sid, serial#, username, sql_id, event, wait_class,
       state, seconds_in_wait, blocking_session
FROM v$session
WHERE type = 'USER' AND status = 'ACTIVE';
```

判断等待时查看 `STATE` 和 `WAIT_CLASS`，排除空闲等待，再结合等待事件、阻塞会话和业务影响分析。系统事件统计应计算采样区间内等待次数及时间的增量，而不是直接按启动以来的累计值决定当前瓶颈。RAC 跨实例阻塞需结合 `GV$` 视图的实例信息。

### 按步骤排查性能问题

先记录业务慢的时间、服务名、实例和错误，再对照请求延迟、事务量、节点 CPU 和磁盘延迟。以下 SQL 以单实例为例；RAC 环境应分别检查实例，或使用 `GV$` 并保留 `INST_ID`。动态视图需要适当权限，使用 AWR、ASH 等功能前应核对许可。

**计算提交和回滚速率。**间隔十秒执行两次：

```sql
SELECT SYSTIMESTAMP AS sampled_at, name, value
FROM v$sysstat
WHERE name IN ('user commits', 'user rollbacks');
```

例如提交数增加 5000、间隔 10 秒，则约 500 次提交/秒；回滚单独计算。实例重启或计数下降时丢弃该窗口。提交速率不等于 SQL QPS，应与应用请求量一起分析。

**检查当前等待及阻塞。**

```sql
SELECT sid, serial#, username, sql_id, status, state,
       event, wait_class, seconds_in_wait,
       blocking_instance, blocking_session
FROM v$session
WHERE type = 'USER' AND status = 'ACTIVE'
ORDER BY seconds_in_wait DESC;
```

优先看 `STATE = 'WAITING'` 且 `WAIT_CLASS` 不是 `Idle` 的会话。`BLOCKING_SESSION` 有值时，结合 `BLOCKING_INSTANCE` 在对应实例查找阻塞者的 SQL、事务和业务归属；没有阻塞者不代表不存在其他资源等待。`SECONDS_IN_WAIT` 的解释与会话状态相关，不应将所有行当作当前等待时长。结束会话前评估事务回滚和业务影响。

**找出高负载 SQL。**以下查询是累计耗时排名，作为候选而非当前窗口排名：

```sql
SELECT sql_id, child_number, executions, elapsed_time,
       cpu_time, buffer_gets, disk_reads, SUBSTR(sql_text, 1, 120) AS sql_text
FROM v$sql
ORDER BY elapsed_time DESC
FETCH FIRST 20 ROWS ONLY;
```

对同一实例、`SQL_ID` 和子游标间隔采样，比较 `executions`、`elapsed_time`、`buffer_gets` 的增量；耗时单位为微秒，窗口平均耗时可用耗时增量除以执行次数增量，分母为零时不计算。游标淘汰、重新加载或实例重启后需重新建立基线。对候选 SQL 检查实际执行计划、绑定变量和访问路径；变更前确认采集方式与许可，优化后在相近负载下复测。

**比较等待事件变化。**对 `V$SYSTEM_EVENT` 的非空闲等待间隔采样，比较 `TOTAL_WAITS` 和 `TIME_WAITED_MICRO` 的增量；等待次数和时间同时增长时，结合具体事件、会话和资源进一步定位，而不是只按累计排名操作。

### 指标与命令对照

| 监控内容 | 命令或查询 | 说明 |
|---|---|---|
| TPS、回滚 | `SELECT name, value FROM v$sysstat WHERE name IN ('user commits', 'user rollbacks');` | 两次采样分别计算提交和回滚速率 |
| SQL 调用和耗时 | `SELECT sql_id, executions, elapsed_time, sql_text FROM v$sql ORDER BY elapsed_time DESC FETCH FIRST 10 ROWS ONLY;` | `elapsed_time` 为累计微秒；比较增量，SQL 语法需目标版本支持 |
| P95/P99、最慢请求 | 按[响应时间](../database-monitoring.md#响应时间)查询应用直方图和日志 | `V$SQL` 累计值不能还原精确分位数或单次最大值 |
| 等待事件 | `SELECT event, total_waits, time_waited_micro FROM v$system_event WHERE wait_class <> 'Idle' ORDER BY time_waited_micro DESC;` | 对两次采样求差值；比较等待次数和时间 |
| 锁及阻塞 | `SELECT sid, serial#, blocking_session, event, seconds_in_wait FROM v$session WHERE blocking_session IS NOT NULL;` | 查看阻塞会话，结合业务确认阻塞链 |
| 死锁及错误 | 在服务器执行 `grep -iE 'ORA-00060\|deadlock' <alert日志路径>` | 日志路径与格式按实际部署调整，并结合 trace 文件排查 |

## 数据与存储层（Data / Storage）

| 指标 | 查看方式 | 关注点与告警建议 |
|---|---|---|
| 表空间容量 | `DBA_TABLESPACE_USAGE_METRICS`、数据文件配置 | 使用率 > 85% 警告、> 95% 严重告警；结合自动扩展上限及底层可用空间 |
| 表与索引空间 | `DBA_SEGMENTS`、`DBA_INDEXES` | 关注大对象、增长趋势和异常状态；清理或重建前评估业务影响 |
| UNDO | `V$UNDOSTAT`、UNDO 表空间 | 关注空间压力、长事务和快照过旧错误；按保留要求和业务峰值告警 |
| 临时表空间 | `V$TEMP_SPACE_HEADER` 等 | 持续接近可用容量或排序/临时段分配失败时告警 |
| Redo 与归档 | `V$LOG`、`V$ARCHIVE_DEST_STATUS`、`V$RECOVERY_FILE_DEST` | 关注日志切换、归档失败和 FRA 空间；归档失败或空间预计耗尽时及时告警 |

```sql
SELECT tablespace_name, used_percent
FROM dba_tablespace_usage_metrics
ORDER BY used_percent DESC;

SELECT name, space_limit, space_used, space_reclaimable
FROM v$recovery_file_dest;
```

表空间指标与底层磁盘容量必须一起看，自动扩展不等于空间无限；FRA 未配置时相关视图可能没有行。Redo 日志通常循环使用，归档积压和目标空间不足需要单独监控。

### 表空间、UNDO 和临时空间

表空间使用率要同时核对数据文件自动扩展上限、ASM 或文件系统剩余容量，以及增长速度。即使数据文件允许扩展，底层空间耗尽仍会导致分配失败。分区表和索引需要分别检查段及分区状态；索引重建或空间回收应先评估业务影响。

`V$UNDOSTAT` 以时间区间统计 UNDO 使用和错误。`SSOLDERRCNT` 增长表示该区间发生快照过旧错误，`NOSPACEERRCNT` 增长表示发生空间不足错误；同时检查长事务、UNDO 容量及保留策略。临时空间接近上限时，应找到消耗空间的会话和排序操作，并结合临时文件实际可扩展空间判断。

### Redo、归档和恢复区

Redo 日志通常循环使用；持续切换不必然异常，但切换速率偏离基线时应检查写入量和归档速度。检查 `V$ARCHIVE_DEST_STATUS` 的状态和错误，并结合 FRA 已用、可回收空间和底层剩余空间。归档失败或 FRA 空间耗尽会影响后续操作，应及时告警；删除归档前必须确认备份和备库恢复需求。

### 指标与命令对照

| 监控内容 | 命令或查询 | 说明 |
|---|---|---|
| 表空间容量 | `SELECT tablespace_name, used_percent FROM dba_tablespace_usage_metrics ORDER BY used_percent DESC;` | 同时核对自动扩展上限及底层剩余空间 |
| 表与索引大小 | `SELECT owner, segment_name, segment_type, bytes FROM dba_segments WHERE segment_type IN ('TABLE', 'INDEX') ORDER BY bytes DESC FETCH FIRST 10 ROWS ONLY;` | 分区对象需按实际 segment 类型另行查看 |
| 索引状态 | `SELECT owner, index_name, table_name, status FROM dba_indexes WHERE status <> 'VALID';` | 分区索引还需检查分区状态 |
| UNDO 压力 | `SELECT begin_time, undoblks, txncount, ssolderrcnt, nospaceerrcnt FROM v$undostat ORDER BY begin_time DESC FETCH FIRST 12 ROWS ONLY;` | 检查快照过旧及空间错误，结合 UNDO 容量 |
| 临时空间 | `SELECT tablespace_name, SUM(bytes_used) AS used_bytes, SUM(bytes_free) AS free_bytes FROM v$temp_space_header GROUP BY tablespace_name;` | 结合临时文件大小及可扩展空间判断 |
| Redo、归档及 FRA | `SELECT group#, sequence#, bytes, status, archived FROM v$log;`；`SELECT dest_id, status, error FROM v$archive_dest_status;`；`SELECT name, space_limit, space_used, space_reclaimable FROM v$recovery_file_dest;` | 分别检查日志组、归档错误和恢复区容量 |

## 集群层（HA / Replication）

| 指标 | 查看方式 | 关注点与告警建议 |
|---|---|---|
| 数据库角色与打开模式 | `V$DATABASE` | 主备角色或打开模式不符合预期时告警 |
| Data Guard 传输与应用 | `V$DATAGUARD_STATS`、`V$ARCHIVE_DEST_STATUS` | 关注传输延迟、应用延迟、目标状态和错误；超过业务允许值或复制中断时告警 |
| 延迟数据新鲜度 | `V$DATAGUARD_STATS` 的时间信息 | 数据过期或无法获得时单独告警，不能将缺失延迟视为零 |
| RAC 实例与服务 | `GV$INSTANCE` 及服务状态 | 逐实例检查状态和服务；节点或业务服务不可用时告警 |
| 切换事件 | 集群管理及数据库告警日志 | 切换失败、反复切换或恢复时间超过 RTO 时告警 |

```sql
SELECT name, value, unit, time_computed, datum_time
FROM v$dataguard_stats
WHERE name IN ('transport lag', 'apply lag');
```

Data Guard 指标以适用实例及其角色为准；比较延迟的同时确认统计数据持续更新。RAC 和 Data Guard 是否适用取决于实际架构。

### Data Guard 与 RAC

在适用的备库检查传输延迟和应用延迟，并同时检查 `DATUM_TIME` 是否持续更新；旧值或空值不能当作零延迟。传输落后时检查主备网络和归档目标，应用落后时检查备库恢复进程、负载和存储。RAC 则应按实例观察连接、等待、资源额度和业务服务，避免集群平均值掩盖单实例问题。

### 指标与命令对照

| 监控内容 | 命令或查询 | 说明 |
|---|---|---|
| Data Guard 延迟 | `SELECT name, value, unit, time_computed, datum_time FROM v$dataguard_stats WHERE name IN ('transport lag', 'apply lag');` | 在适用实例查看，同时检查数据是否过期 |
| RAC 实例 | `SELECT inst_id, instance_name, status FROM gv$instance ORDER BY inst_id;` | 数据库服务状态还需用实际集群管理工具检查 |
| 切换及归档错误 | 在服务器执行 `grep -iE 'switchover\|failover\|ORA-\|archive' <alert日志路径>` | 结合集群管理和 Data Guard 事件日志 |

Oracle 版本、许可与工具会影响可用视图和命令；遇到缺失字段或权限错误时应明确检查版本与授权，不应把缺失数据当作正常。
