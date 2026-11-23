# 在 Kubernetes 部署 PostgreSQL 集群

本文使用 CloudNativePG 部署三个 PostgreSQL 实例：一个主库和两个物理流复制备库。Operator 负责实例管理和故障切换；示例默认异步复制，主库故障时可能丢失尚未复制的事务。同步复制需按业务 RPO 和可用性要求另行配置。

## 准备和安装 Operator

准备 `kubectl`、Helm、可动态供给持久卷的 StorageClass，以及足够运行三个数据库实例的资源。生产环境应将实例和存储分布在不同故障域；单节点仅适合实验。Operator 安装需要相应集群权限。

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update cnpg
helm upgrade --install cnpg cnpg/cloudnative-pg \
  --namespace cnpg-system --create-namespace
kubectl -n cnpg-system rollout status deployment/cnpg-cloudnative-pg --timeout=5m
kubectl create namespace postgresql
```

实际部署应参考 [CloudNativePG 安装文档](https://cloudnative-pg.io/docs/)固定并核对 Operator 和 chart 版本、Kubernetes 兼容性及监视范围。若已有 Operator，先确认其覆盖 `postgresql` 命名空间。

## 创建数据库和账号

以下 Bash 命令创建应用用户的初始化密码。Secret 是 `kubernetes.io/basic-auth` 类型，`username` 必须与数据库所有者一致；不要提交密码，限制 Secret 访问并启用静态加密。

```bash
read -rsp 'PostgreSQL 应用密码: ' PG_PASSWORD; echo
kubectl -n postgresql create secret generic app-user \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=app --from-literal="password=$PG_PASSWORD"
unset PG_PASSWORD
```

保存为 `postgresql-cluster.yaml`，按需设置 `storageClass` 和资源，再执行 `kubectl apply -f postgresql-cluster.yaml`：

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-cluster
  namespace: postgresql
spec:
  instances: 3
  bootstrap:
    initdb:
      database: appdb
      owner: app
      secret:
        name: app-user
  storage:
    size: 10Gi
  resources:
    requests:
      cpu: "500m"
      memory: 1Gi
    limits:
      memory: 2Gi
```

Operator 为每个实例创建独立 PVC。示例使用 Operator 默认镜像；生产环境应设置 `spec.imageName` 为经过验证的明确版本或摘要，并按文档配置节点亲和和故障域。数据库初始化后修改 Secret 不等于完成既有用户密码轮换，操作前查阅 Operator 对角色和凭据管理的说明。

## 验证和访问

```bash
kubectl -n postgresql get cluster pg-cluster -o yaml
kubectl -n postgresql get pods,pvc,svc -l cnpg.io/cluster=pg-cluster -o wide
kubectl -n postgresql describe cluster pg-cluster
```

确认三个实例就绪，Operator 状态显示一个主库及两个备库。应用写入使用 `pg-cluster-rw.postgresql.svc`；`pg-cluster-ro` 是只读副本入口，具体域名取决于集群配置。使用已装 `psql` 的客户端 Pod，从应用所在网络通过 `pg-cluster-rw`、`appdb` 和 `app` 连接，密码交互输入，执行：

```sql
SELECT current_database(), pg_is_in_recovery();
SELECT application_name, state, sync_state FROM pg_stat_replication;
```

主库 `pg_is_in_recovery()` 应为 `false`；复制视图需要适当权限。备库可通过只读入口检查 `pg_is_in_recovery()` 为 `true`。再写入测试数据并确认备库回放；异步复制下应允许短暂延迟。

## 集群内与内网外部访问

集群内应用使用 `pg-cluster-rw.postgresql.svc:5432` 连接当前主库；只读流量可使用 `pg-cluster-ro`。先检查 `kubectl -n postgresql get svc,endpoints`，再从应用 Pod 使用业务账号验证连接及查询。服务名在故障切换后仍是固定入口，但客户端需要重新建立断开的连接。

如集群外的企业内网或 VPN 客户端需要连接，可创建仅供内网访问的 TCP LoadBalancer。先确认云平台或负载均衡实现支持**内部**负载均衡器，并设置该平台要求的注解、子网和访问控制；下方占位注解不是可直接使用的厂商配置。没有负载均衡器的环境可使用内部支持的 NodePort 或 TCP 代理，并限制节点端口访问。

先检查 Operator 生成的读写服务所用选择器，避免手工选择到备库：

```bash
kubectl -n postgresql get service pg-cluster-rw -o yaml
```

以下示例使用常见的 CloudNativePG 角色标签；如果安装版本的 `pg-cluster-rw` 选择器不同，应以实际服务为准进行调整：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: pg-cluster-rw-internal-lb
  namespace: postgresql
  annotations:
    # 替换为云平台创建内部负载均衡器所需的注解。
spec:
  type: LoadBalancer
  selector:
    cnpg.io/cluster: pg-cluster
    cnpg.io/instanceRole: primary
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

保存并应用后，使用 `kubectl -n postgresql get svc pg-cluster-rw-internal-lb` 和 EndpointSlice 确认地址及当前主库；`EXTERNAL-IP` 是 Kubernetes 字段名，不必然表示公网地址。配置内部 DNS、路由和防火墙，从内网客户端连接：

```bash
psql 'host=pg.example.internal port=5432 dbname=appdb user=app sslmode=verify-full sslrootcert=/path/to/ca.crt' -W
```

`verify-full` 要求服务端证书包含该内网 DNS 名称，需按 CloudNativePG 的证书配置方式提供匹配证书和受信任 CA。连接后执行 `SELECT pg_is_in_recovery();`，写入口应返回 `false`。切换后检查新 EndpointSlice、重新连接并验证写入；网络策略、负载均衡和防火墙均须允许实际客户端来源。TLS 与业务账号权限应在开放前配置。

## 运维

在隔离实验环境演练主库故障、服务切换和客户端重连，确认恢复时间及可能的数据丢失；不要在生产环境直接删除 Pod 测试。配置基础备份、WAL 归档及恢复演练，副本和 PVC 均不能替代备份。监控复制槽、WAL 空间和延迟，见 [PostgreSQL 监控](../../observability/postgresql/README.md)。清理前检查 PVC 保留和 PV 回收策略。

参考：[CloudNativePG 快速入门](https://cloudnative-pg.io/docs/devel/quickstart/)。
