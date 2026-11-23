# 使用 Prometheus 和 Node Exporter 监控 Kubernetes 节点

Node Exporter 采集 Linux 节点的 CPU、内存、文件系统、磁盘和网络指标。一个节点上即使运行多个数据库 Pod，也只需一份 Node Exporter；Prometheus 再按节点抓取。它不提供数据库连接、SQL、复制或数据库缓存等内部指标，后者需要数据库专用 exporter 或 Operator 的指标端点。

以下以 Kubernetes 上的 `kube-prometheus-stack` 为例，包含 Prometheus Operator、Prometheus 和 Node Exporter。命令使用 Bash，适合实验；安装前确认 Kubernetes 兼容性、节点资源、RBAC、存储和网络策略，并固定经过验证的 chart 版本。Node Exporter 在 Linux 节点运行，需要访问主机信息；受限集群需先核对安全策略。

## 安装或检查已有组件

如果集群已有 Prometheus，先检查是否已经部署 Node Exporter，避免重复抓取；直接进入下文的检查和抓取配置。不要对现有 Helm release 直接套用下面的安装命令，否则可能更改现有配置。

```bash
kubectl get daemonsets -A | grep -i node-exporter
kubectl get servicemonitors -A | grep -i node-exporter
kubectl get prometheus -A
```

新建实验环境时，可按官方 chart 安装。将 `<chart-version>` 替换为已验证版本；如不存在默认 StorageClass 或需要持久化 Prometheus 数据，请根据 chart values 事先配置。

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update prometheus-community
helm search repo prometheus-community/kube-prometheus-stack --versions
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace --version <chart-version>
kubectl -n monitoring get pods,daemonsets,svc,servicemonitors
```

确认所选 chart 的 values 中启用了 Node Exporter 和它的 ServiceMonitor；不同版本的默认值可能变化。节点有 taint、nodeSelector 限制或安全策略时，检查 DaemonSet 是否能调度到所有承载数据库的节点。仅有一个 Prometheus Pod Running 并不代表已成功采集。

## 为已有 Prometheus 添加 Node Exporter

如果已有 Prometheus Operator 和 ServiceMonitor CRD，但没有 Node Exporter，可以单独安装 chart。以下启用 ServiceMonitor；将 `<chart-version>` 替换为实际版本：

```bash
helm upgrade --install node-exporter prometheus-community/prometheus-node-exporter \
  --namespace monitoring --create-namespace --version <chart-version> \
  --set prometheus.monitor.enabled=true
kubectl -n monitoring get daemonsets,pods,svc,servicemonitors
```

Prometheus 必须通过 `serviceMonitorSelector` 选中这个 ServiceMonitor 的标签，并通过 `serviceMonitorNamespaceSelector` 选中它所在的命名空间；根据实际 Prometheus 配置，在 chart 的 `prometheus.monitor.additionalLabels` 中添加需要的标签，例如某些 Helm 安装使用的 `release` 标签。还要保证 Prometheus 的 RBAC 和网络策略允许发现并抓取目标。不要假定所有安装都使用同一套选择器。

```bash
kubectl -n monitoring get prometheus -o yaml
kubectl -n monitoring get servicemonitors --show-labels
kubectl -n monitoring get endpointslices
```

普通 Prometheus 没有 ServiceMonitor 机制，需要在 Prometheus 中配置 Kubernetes 服务发现和相应 RBAC，逐个发现 Node Exporter 端点。不要只把 ClusterIP Service 当作一个静态目标：那样通常只会轮询到一个后端，无法可靠采集所有节点。

## 验证抓取和查询

检查 Node Exporter DaemonSet 的期望数与就绪数，并检查数据库 Pod 分布：

```bash
kubectl -n monitoring get daemonsets,pods -o wide
kubectl get pods -A -o wide
```

通过受控方式访问 Prometheus UI，例如将下面的 `<prometheus-service>` 替换为 `kubectl -n monitoring get svc` 查到的 Prometheus 服务名称：

```bash
kubectl -n monitoring port-forward svc/<prometheus-service> 9090:9090
```

打开 `http://localhost:9090/targets`，确认每个预期节点的 Node Exporter target 为 UP；若缺失或 DOWN，检查 ServiceMonitor 的标签与命名空间选择器、Service EndpointSlice、RBAC、端口和网络策略。端口转发默认用于本机访问；远程访问应按组织方式配置认证和网络访问。

在 Prometheus 的查询界面验证以下指标：

```promql
node_uname_info
100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
100 * (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"})
rate(node_network_receive_errs_total[5m])
```

确认节点指标能与数据库 Pod 所在节点对应；`instance` 常是抓取地址，不一定是 Kubernetes 节点名，按实际标签或服务发现做映射。文件系统告警应只选择数据库数据、日志和备份实际使用的挂载点；网络错误计数不等于端到端丢包率。PromQL 的标签和可用指标以实际 exporter 及采集配置为准。

参考：[Node Exporter 文档](https://prometheus.io/docs/guides/node-exporter/)、[Node Exporter chart 配置](https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus-node-exporter/values.yaml)、[Prometheus Operator 的 ServiceMonitor](https://prometheus-operator.dev/docs/developer/getting-started/)。
