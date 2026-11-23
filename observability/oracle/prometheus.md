# Oracle：Prometheus 与 Node Exporter

本文对应 [Oracle Data Guard 主备部署](../../provision/oracle/README.md)。Node Exporter 采集数据库节点的 Linux 资源；Oracle Database Metrics Exporter 连接数据库查询视图，向 Prometheus 提供数据库指标。主库和备库分别采集，RAC 与多租户还需按实例和容器验证覆盖范围。

## 配置节点采集

按[共享指南](../prometheus-node-exporter.md)安装或检查 Prometheus、Node Exporter、ServiceMonitor 和采集范围。用 `kubectl -n default get pods -o wide` 查看数据库 Pod 所在节点，并在 Prometheus Targets 确认节点 exporter 均为 UP。

可采集 CPU、可用内存、磁盘读写、文件系统容量、网卡流量及错误。例如 `node_filesystem_avail_bytes` 可以观察可见的文件系统剩余空间，但 ASM、网络存储或容器卷的实际容量应结合存储平台及 Oracle 指标核对；节点指标不能直接作为某个数据库实例的资源用量。

## 配置 Oracle 数据库指标

采用 [Oracle Database Metrics Exporter](https://oracle.github.io/oracle-db-appdev-monitoring/docs/getting-started/kubernetes/)。先确认 exporter、数据库及镜像版本兼容，并核对 Oracle 的视图权限和许可。创建只用于监控的数据库账号，按[官方权限清单](https://oracle.github.io/oracle-db-appdev-monitoring/docs/getting-started/basics/)对实际需要的视图授予读取权限；连接到 CDB 或 PDB 的位置影响可见范围。生产环境优先使用加密连接和受保护的 wallet。

下面示例针对一个可访问的数据库服务，主备分别部署并使用各自地址。以下命令使用 Bash，替换镜像版本、命名空间、用户名与服务地址。Secret 仅保存在 Kubernetes 中仍需 RBAC 和静态加密；不要把密码写入 Git。

```bash
read -rsp 'Oracle 监控账号密码: ' ORACLE_MONITOR_PASSWORD; echo
kubectl -n default create secret generic oracle-exporter-auth \
  --from-literal=username=exporteruser \
  --from-literal="password=$ORACLE_MONITOR_PASSWORD"
unset ORACLE_MONITOR_PASSWORD
```

保存为 `oracle-exporter.yaml`。`DB_CONNECT_STRING` 必须是 exporter 能访问、适用于目标角色和数据库容器的实际服务；`<exporter-version>` 使用经验证的镜像标签。此示例使用 TCP 和简单凭据演示，生产环境按官方文档配置 TCPS 与 wallet。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: oracle-exporter-config
  namespace: default
data:
  config.yaml: |
    databases:
      default:
        username: ${DB_USERNAME}
        password: ${DB_PASSWORD}
        url: ${DB_CONNECT_STRING}
    metrics:
      default: /default-metrics.toml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oracle-exporter-primary
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels: {app: oracle-exporter-primary}
  template:
    metadata:
      labels: {app: oracle-exporter-primary}
    spec:
      containers:
        - name: exporter
          image: container-registry.oracle.com/database/observability-exporter:<exporter-version>
          args: ["--config.file=/exporter/config.yaml"]
          env:
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef: {name: oracle-exporter-auth, key: username}
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef: {name: oracle-exporter-auth, key: password}
            - name: DB_CONNECT_STRING
              value: "<主库服务DNS>:1521/<数据库服务名>"
          ports:
            - name: metrics
              containerPort: 9161
          volumeMounts:
            - name: config
              mountPath: /exporter
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: oracle-exporter-config
---
apiVersion: v1
kind: Service
metadata:
  name: oracle-exporter-primary
  namespace: default
  labels: {monitoring: oracle-exporter}
spec:
  selector: {app: oracle-exporter-primary}
  ports:
    - name: metrics
      port: 9161
      targetPort: metrics
```

为备库另建 Deployment 和 Service，修改名称、selector 以及 `DB_CONNECT_STRING`；认证方式和物理备库打开状态需与实际架构兼容。Data Guard 切换后必须核对两个采集目标的实例身份及角色：固定主库入口不适合单独监控两个实例。按官方文档验证备库哪些视图可用。

## 配置 Prometheus 抓取

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: oracle-exporters
  namespace: default
spec:
  selector:
    matchLabels: {monitoring: oracle-exporter}
  endpoints:
    - port: metrics
      interval: 30s
```

应用后检查 `kubectl -n default get deploy,svc,servicemonitor` 和 exporter 日志。Prometheus 必须通过 `serviceMonitorNamespaceSelector` 和 `serviceMonitorSelector` 选中该命名空间及 ServiceMonitor；必要时添加实际要求的标签。Targets 应出现主备两个目标，`up` 只说明 Prometheus 可抓取 exporter，仍需检查数据库连接和采集错误。

## 可以采集的数据

| 类别 | 数据来源示例 | 使用时关注 |
|---|---|---|
| 实例和会话 | `GV$INSTANCE`、`GV$SESSION`、`GV$RESOURCE_LIMIT` | 按实例区分会话、进程及状态 |
| 等待与负载 | `GV$SYSTEM_WAIT_CLASS`、`GV$SYSSTAT` | 累计统计需要比较采样窗口，区分空闲等待 |
| 表空间与数据文件 | `DBA_TABLESPACE_USAGE_METRICS`、`GV$DATAFILE` | 同时检查自动扩展上限和底层容量 |
| 数据库角色 | `GV$DATABASE` 等 | 主备状态应与实际 Data Guard 拓扑一致 |
| 节点资源 | `node_cpu_seconds_total`、`node_memory_MemAvailable_bytes` 和磁盘/网络统计 | 将 Pod、实例与节点关联 |

哪些指标默认可用取决于 exporter 的内置 metrics 配置、数据库版本、连接容器和账号权限；Data Guard 延迟、归档、PDB 或特定业务数据可能需要配置[自定义指标](https://oracle.github.io/oracle-db-appdev-monitoring/docs/configuration/custom-metrics/)。通过 `/metrics`、exporter 日志及相应 SQL 核对，不要将缺失指标视为正常值；额外使用的性能视图或功能须核对许可。
