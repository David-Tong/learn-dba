# MongoDB：Prometheus 与 Node Exporter

本指南对应 [MongoDB 副本集部署](../../provision/mongodb/README.md)。Prometheus 同时采集节点指标和 MongoDB 指标：Node Exporter 在每个 Linux 节点采集宿主机资源；MongoDB exporter 连接各副本集成员采集数据库内部状态。示例使用 Prometheus Operator、ServiceMonitor 及 Percona `mongodb_exporter`；实施前确认版本兼容、服务发现和 TLS。

## 采集节点指标

先按[共享安装指南](../prometheus-node-exporter.md)安装或检查 Prometheus、Node Exporter 和 ServiceMonitor。Node Exporter 无需为每种数据库重复安装。执行 `kubectl -n mongodb get pods -o wide`，将 MongoDB 成员所在节点与 Prometheus 的 Node Exporter targets 对照。

| 类别 | 可采集的数据 | 排查用途 |
|---|---|---|
| CPU | `node_cpu_seconds_total` | 判断节点 CPU 忙碌和 IO 等待趋势 |
| 内存 | `node_memory_MemAvailable_bytes` | 对照 WiredTiger 缓存和操作系统可用内存 |
| 磁盘 | `node_disk_read_bytes_total`、`node_disk_write_bytes_total` | 对照复制、查询及检查点的磁盘负载 |
| 文件系统 | `node_filesystem_avail_bytes`、`node_filesystem_size_bytes` | 监控实际数据卷所在文件系统的空间 |
| 网络 | `node_network_receive_errs_total` | 检查网卡错误并结合节点间网络排查 |

这些是节点级指标，不能直接视为某个 MongoDB Pod 的消耗；磁盘或文件系统与 PVC 的对应关系需要结合实际存储驱动确认。

## 配置 MongoDB exporter

使用 [Percona MongoDB exporter](https://github.com/percona/mongodb_exporter) 时，应分别抓取每个副本集成员，才能发现单个 Secondary 落后或节点不可用。先通过 MongoDB Operator 创建只用于监控的账号，按 exporter 官方权限说明授予 `admin` 库的 `clusterMonitor` 和 `local` 库的 `read`；启用其他 collector 可能需要额外权限。密码使用 Secret 管理，凭据变化后按部署方式更新 exporter。

以下是一个成员的示例模式。使用各成员可解析的集群内 DNS，复制为三个 Deployment，分别修改名称、标签和连接目标；`<exporter-version>` 替换为经过验证的镜像版本。创建 `mongo-exporter-uri-0` Secret 的 `uri` 键，内容为该成员的连接 URI，例如 `mongodb://monitor:<URL编码后的密码>@<成员0的DNS>:27017/admin?authSource=admin&directConnection=true`。不要把真实 URI 写入 Git 或终端历史。TLS 环境还需挂载 CA 并按 exporter 和驱动文档配置验证。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-exporter-0
  namespace: mongodb
spec:
  replicas: 1
  selector:
    matchLabels: {app: mongo-exporter-0}
  template:
    metadata:
      labels: {app: mongo-exporter-0, monitoring: mongo-exporter}
    spec:
      containers:
        - name: exporter
          image: percona/mongodb_exporter:<exporter-version>
          args: ["--collector.replicasetstatus"]
          env:
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef: {name: mongo-exporter-uri-0, key: uri}
          ports:
            - name: metrics
              containerPort: 9216
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-exporter-0
  namespace: mongodb
  labels: {monitoring: mongo-exporter}
spec:
  selector: {app: mongo-exporter-0}
  ports:
    - name: metrics
      port: 9216
      targetPort: metrics
```

每个 Service 只选择一个 exporter；每个 exporter 只连接一个成员。复制时还要分别创建 `mongo-exporter-uri-1`、`mongo-exporter-uri-2` Secret，并检查 Pod DNS 与 Service endpoints。若 exporter 版本对参数或默认 collector 有差异，以其 `--help` 和[参数说明](https://github.com/percona/mongodb_exporter/blob/main/REFERENCE.md)为准。

## 配置 Prometheus 抓取

以下 ServiceMonitor 选择 `mongodb` 命名空间中上述三个 Service。Prometheus 必须允许发现该命名空间，并通过自己的 `serviceMonitorSelector` 选中此资源；如需 `release` 等标签，添加与实际 Prometheus 一致的标签。

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mongo-exporters
  namespace: mongodb
spec:
  selector:
    matchLabels: {monitoring: mongo-exporter}
  endpoints:
    - port: metrics
      interval: 30s
```

应用清单后执行 `kubectl -n mongodb get deploy,svc,servicemonitor`，检查 exporter 日志与 endpoints；在 Prometheus Targets 中确认三个独立目标为 UP。查询 `mongodb_up` 并检查各成员标签：Prometheus `up=1` 只说明能访问 exporter，还需数据库连接指标判断 exporter 是否能访问 MongoDB。

## 数据库指标与验证

| 数据来源 | 可以观察什么 | 注意事项 |
|---|---|---|
| `serverStatus` | 连接、操作计数、WiredTiger 缓存、锁与等待 | 操作累计计数需按时间窗口计算速率 |
| `replSetGetStatus` | 成员状态和复制进度 | 示例显式启用 replica set status collector；分别检查每个成员 |
| `$collStats`、`$indexStats` | 集合大小、索引访问等 | 需按需启用 collector、配置集合范围及权限；注意采集开销 |
| `getDiagnosticData` | 支持版本的诊断统计 | 需按需启用，字段和权限依版本而异 |

指标名称及标签随 exporter 版本和模式变化；通过 exporter `/metrics` 的 `HELP`、`TYPE` 和标签确定实际名称，不要把旧版本仪表盘直接当作标准。复制延迟还应结合 `rs.status()`、oplog 窗口及无写入情况判断；业务 P95/P99 通常来自应用或驱动直方图。Node Exporter 和数据库 exporter 应限制在监控网络访问，生产中启用 TLS、资源限制和告警。
