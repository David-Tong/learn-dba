# 在 Kubernetes 部署 Oracle Data Guard 主备集群

本文采用 Oracle Database Operator 的 Data Guard 流程：一个 Primary 数据库、一个 Physical Standby 和 Data Guard Broker。主备拥有独立实例和存储，通过 redo 传输与回放保持同步，可用于学习复制及切换；它不是多个实例共享数据库的 Oracle RAC。是否可用、受支持及许可要求取决于 Oracle Database、镜像、Operator 和平台的具体版本；下文以官方 Enterprise 样例为起点。

## 准备

- 可访问 Kubernetes 集群，安装 `kubectl`、`git`、`jq`、Ruby，并具备安装 Operator、CRD 和所需组件的权限。
- 提供主备独立的持久卷及可相互访问的服务地址；生产环境应分散到不同节点和故障域，并规划容量与备份。
- 获取经过授权且与 Operator 匹配的数据库镜像；如 Oracle Container Registry 要求登录、接受条款或拉取凭据，先完成对应配置。不要将密码、镜像凭据和 wallet 提交到 Git。
- 对照 [Operator 前置条件](https://github.com/oracle/oracle-database-operator/blob/main/docs/sidb/PREREQUISITES.md)安装并验证相应版本的 Operator、CRD、证书组件和存储。Oracle 官方 [SIDB 与 Data Guard 文档](https://github.com/oracle/oracle-database-operator/blob/main/docs/sidb/README.md)是具体版本的操作依据。

## 取得对应版本的样例

克隆官方仓库并检出**与已安装 Operator 相同的发布标签**；以下 Bash 中的 `<operator-version>` 必须先替换。确认该版本提供 `database.oracle.com/v4` API 和本文引用的 Data Guard 样例，否则使用该版本官方文档。

```bash
git clone https://github.com/oracle/oracle-database-operator.git
cd oracle-database-operator
git checkout <operator-version>
kubectl get crd | grep -E 'singleinstancedatabases|dataguardbrokers'
```

Operator 的安装命令与前置组件因版本、平台而异，请按该标签下的安装文档执行并确认控制器运行后再继续。以下使用 Operator 监听的 `default` 命名空间和官方样例名；若采用其他命名空间，所有资源、Secret 和命令都要一致修改。

## 创建主库及备库

使用所检出版本的以下样例，并先审阅、复制到自己的工作目录修改：

```text
config/samples/sidb/singleinstancedatabase_create.yaml
config/samples/sidb/singleinstancedatabase_standby.yaml
```

参考样例和前置文档创建管理员密码 Secret、必要的镜像拉取 Secret，并配置镜像地址、命名空间、存储类及容量。确认主备引用的凭据和镜像兼容，数据库网络互通。若启用 TDE，备库初始化还需按官方流程导出主库 wallet 并创建备库 wallet Secret。

主库样例需设置 `spec.createAs: primary`；备库样例设置 `spec.createAs: standby`，并通过 `spec.primarySource.databaseRef` 指向同命名空间的主库。两个资源都按官方 Data Guard 样例启用：

```yaml
# 以下是主库或备库的 spec 内片段，不是完整资源。
dataguard:
  prereqs:
    enabled: true
```

假设修改后的文件名为 `primary.yaml` 和 `standby.yaml`，主库资源名为 `sidb-sample`，备库名为 `standbydatabase-sample`：

```bash
kubectl apply -f primary.yaml
kubectl -n default get singleinstancedatabase sidb-sample -w
# 主库 Healthy 后，另开终端执行：
kubectl apply -f standby.yaml
kubectl -n default get singleinstancedatabase,pods,pvc -o wide
```

等待两个资源均为 `Healthy`，角色分别为 `PRIMARY` 和 `PHYSICAL_STANDBY`。资源健康本身不代表复制已配置：必须继续创建 Broker。初始化耗时较长；失败时用 `kubectl describe singleinstancedatabase <名称> -n default`、Pod 日志和事件检查。

## 配置 Data Guard Broker

在仓库根目录使用官方脚本，根据备库状态生成 Broker 配置。以下 Secret 名称及键必须和主库样例实际配置一致；`jq`、Ruby 和 `kubectl` 是脚本依赖。

```bash
PRIMARY_ADMIN_SECRET_NAME=<主库管理员Secret名> \
PRIMARY_ADMIN_SECRET_KEY=<密码键名> \
  bash config/samples/sidb/render-dg-broker-from-status.sh \
  sidb standbydatabase-sample default standbydatabase-sample-dg > dataguardbroker.yaml
```

确认命令成功，检查生成的文件中主备成员、地址、命名空间、Secret、镜像及卷配置，再执行：

```bash
kubectl apply -f dataguardbroker.yaml
kubectl -n default get dataguardbroker,singleinstancedatabase,pods -o wide
kubectl -n default describe dataguardbroker standbydatabase-sample-dg
```

确认 Broker 状态健康并检查其条件和事件。切换操作及具体状态字段依 Operator 版本而异；官方当前流程使用 DGMGRL 执行 switchover/failover。主备健康但 Broker 未配置时，不能认为数据已同步。

## 集群内与内网外部访问

先获取 Operator 为主库生成的服务及连接串，确认业务 PDB 的服务已注册：

```bash
kubectl -n default get svc,endpoints
kubectl -n default get singleinstancedatabase sidb-sample \
  -o jsonpath='{.status.connectString}{"\n"}'
```

集群内客户端使用实际主库 Service 的 DNS、监听端口和业务 PDB 服务名连接，并执行 `SELECT 1 FROM dual`。上述主库资源的连接串针对该资源，并不保证切换后自动指向新的 Primary。

集群外的企业内网或 VPN 客户端可以通过内网 TCP/TCPS LoadBalancer、NodePort 或受控 TCP 代理连接。按所安装版本的 [Oracle Operator SIDB 服务配置](https://github.com/oracle/oracle-database-operator/blob/main/docs/sidb/README.md)设置 `spec.services.endpoints`，选择支持的 `LoadBalancer` 或 `NodePort` 类型；使用云厂商指定的**内部**负载均衡配置及子网，限制客户端来源，并配置内网 DNS。`LoadBalancer` 默认是否内网取决于平台，不能仅凭 Kubernetes 类型判断。可先从单个当前主库端点验证监听和 PDB 服务，再配置切换后的角色感知入口。

Data Guard 切换后，固定指向原主库 Pod 的 Service 不会自动成为新主库入口。应按照所用 Operator 版本的官方 Data Guard 自定义服务示例，配置只在 Primary 上运行的 PDB 服务及能够跟随角色变化的网络入口，或使用具备角色感知能力的代理；切换后验证新主库的监听注册、Service 后端、客户端重连和写入。在未完成角色感知之前，不要把固定主库地址描述成自动切换入口。

从授权内网客户端通过 `sqlplus` 或其他 Oracle 客户端连接，使用实际 DNS、端口和 PDB 业务服务名，交互输入密码：

```bash
sqlplus 'appuser@//oracle-db.example.internal:1521/<业务PDB服务名>'
```

使用 TCPS 时需在 Operator、监听及客户端 wallet 中配置证书和受信任 CA，连接描述符也应使用匹配的 TCPS 端口与服务名。除 DNS、路由外，检查防火墙、网络策略和数据库账号授权。

## 验证复制和运维

从各实例使用有权限的 SQL 客户端检查角色和打开模式，并在备库检查延迟及数据时间：

```sql
SELECT name, database_role, open_mode FROM v$database;
SELECT name, value, unit, time_computed, datum_time
FROM v$dataguard_stats
WHERE name IN ('transport lag', 'apply lag');
```

使用 Broker 的 DGMGRL 执行 `SHOW CONFIGURATION;`，确认配置和成员状态。按业务方案测试主库写入、备库回放及受控切换；物理备库能否打开查询取决于版本、配置及许可，不能仅凭查询失败认定复制失败。规划业务服务切换、客户端重连、备份和恢复演练，监控延迟、归档及恢复区，见 [Oracle 监控](../../observability/oracle/README.md)。删除资源前确认依赖、PVC 保留及底层回收策略。
