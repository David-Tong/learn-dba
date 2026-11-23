# PostgreSQL：Prometheus 与 Node Exporter

本文对应 [CloudNativePG 三实例部署](../../provision/postgresql/README.md)：每个数据库 Pod 的内置监控端点提供 PostgreSQL 指标，Node Exporter 提供数据库所在 Linux 节点的资源指标。Prometheus 应抓取每个主备实例，而不是只抓取主库 Service。

## 配置节点采集

按[共享指南](../prometheus-node-exporter.md)安装或检查 Prometheus、Node Exporter 与 ServiceMonitor。通过 `kubectl -n postgresql get pods -o wide` 将主备 Pod 映射到节点，并确认每个节点的 exporter target 为 UP。

Node Exporter 可采集 CPU 时间、可用内存、磁盘读写字节和 IO 时间、文件系统容量、网络流量和错误。重点关注实际数据目录与 WAL 文件所在卷；存储可能不直接对应节点挂载点。节点指标是整机数据，不等于 PostgreSQL 进程或 Pod 的用量。

## 配置 PostgreSQL 指标抓取

CloudNativePG 实例提供 Prometheus 指标端点。为避免依赖已弃用的自动生成选项，创建独立 PodMonitor；下面的标签与部署示例中的 `pg-cluster` 一致。已安装的 Prometheus Operator 必须具有 PodMonitor CRD；Prometheus 的 `podMonitorSelector` 和 `podMonitorNamespaceSelector` 必须选中该资源及其命名空间，必要时在 metadata 中加实际需要的 `release` 等标签。

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: pg-cluster
  namespace: postgresql
spec:
  selector:
    matchLabels:
      cnpg.io/cluster: pg-cluster
  podMetricsEndpoints:
    - port: metrics
      interval: 30s
```

```bash
kubectl apply -f postgresql-podmonitor.yaml
kubectl -n postgresql get pods --show-labels
kubectl -n postgresql get podmonitor pg-cluster -o yaml
kubectl -n monitoring get prometheus -o yaml
```

先确认目标 Operator 版本实际为实例 Pod 提供名为 `metrics` 的端口（通常为 9187），并确保网络策略允许 Prometheus 访问。Prometheus Targets 应出现三个独立实例；若为空，检查 Pod 标签、命名空间与 Prometheus 选择器、RBAC 和指标端口。具体配置见 [CloudNativePG 监控文档](https://cloudnative-pg.io/docs/devel/monitoring/)。

## 可以采集的数据

| 类别 | 内容及数据来源 | 使用时关注 |
|---|---|---|
| 可用性 | exporter 抓取状态及数据库实例状态 | `up` 表示抓取端点成功，并不单独证明业务 SQL 成功 |
| 连接与会话 | 数据库连接数、活跃会话及等待 | 对照连接上限、连接池及等待类型 |
| 事务 | 提交、回滚等累计统计 | 对累计值按窗口计算速率，注意统计重置 |
| 缓存及表 | 块命中/读取、表和索引活动、死元组估计 | 按窗口比较并结合执行计划及 VACUUM |
| 复制与 WAL | 主备连接、复制进度、WAL 生成等 | 分别看各实例；不可用或缺失值不等于零延迟 |
| 节点资源 | `node_cpu_seconds_total`、`node_memory_MemAvailable_bytes`、磁盘及网络指标 | 按主备所在节点关联，区分节点和 Pod |

默认查询与名称随 CloudNativePG、PostgreSQL 版本和自定义查询配置变化，使用 `/metrics` 的 `HELP`、`TYPE` 查看当前指标。某些 SQL、表或扩展指标需配置额外的查询及权限；不要假设默认端点包含应用 P95/P99。

## 验证

在 Prometheus Targets 页面确认三个 PostgreSQL 目标及对应节点目标均为 UP；分别检查主库和两个备库的标签。对照 SQL 的 `pg_stat_activity`、`pg_stat_replication`、`pg_is_in_recovery()` 核对数值及角色；同时检查采集延迟和抓取错误。若使用非 CloudNativePG 部署，可使用 [postgres_exporter](https://github.com/prometheus-community/postgres_exporter) 分别连接各实例，按其版本和 collector 配置监控账号与 TLS；不要仅指向读写 Service 后认为已覆盖所有备库。
