# 在 Kubernetes 部署 MongoDB 副本集

本文使用 MongoDB Controllers for Kubernetes 部署三个数据成员的 Community 副本集：一个 Primary、两个 Secondary。它支持复制、选举和故障切换实验；这不是分片集群，分片需要额外部署 `mongos`、Config Server 和分片副本集。

## 准备

- 安装 `kubectl` 和 Helm，确认 context 指向目标集群，且有安装 CRD、Operator 所需权限。
- 至少准备三个可调度节点和支持动态供给的 StorageClass。每个成员使用独立持久卷；按实际资源调整容量及配额。
- 确认 Operator、MongoDB 版本及 Kubernetes 平台兼容，并能拉取所需镜像。示例使用 MongoDB 8.0.0，正式环境应选择已测试且受支持的补丁版本。
- 以下为 Bash 命令；单节点集群能用于功能实验，但不能抵御节点故障。生产环境需配置故障域分散、备份、TLS 和资源限制。

## 安装 Operator

按 [MongoDB 官方安装文档](https://www.mongodb.com/docs/kubernetes/current/tutorial/install-k8s-operator/)选择匹配的 Operator 和 CRD 版本。以下示例使用文档中的 1.15.0 清单；执行前核对目标平台兼容性及版本：

```bash
kubectl create namespace mongodb
kubectl apply -f https://raw.githubusercontent.com/mongodb/mongodb-kubernetes/1.15.0/public/crds.yaml
helm upgrade --install mongodb-kubernetes-operator \
  oci://quay.io/mongodb/helm-charts/mongodb-kubernetes \
  --version 1.15.0 --namespace mongodb
kubectl -n mongodb get deployments,pods
```

等待 Operator 就绪，并确认它监听 `mongodb` 命名空间；如果集群已有 Operator，请检查其版本和监听范围，不要重复安装。

## 创建账号及副本集

Secret 的 `password` 键由 Operator 读取。使用强密码，不要将 Secret 文件或密码提交到 Git；限制 Secret 访问并启用集群静态加密。

```bash
read -rsp 'MongoDB 管理员密码: ' MONGO_PASSWORD; echo
kubectl -n mongodb create secret generic mongo-admin-password \
  --from-literal="password=$MONGO_PASSWORD"
unset MONGO_PASSWORD
```

保存为 `mongodb-replicaset.yaml` 并执行 `kubectl apply -f mongodb-replicaset.yaml`：

```yaml
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: mongo-rs
  namespace: mongodb
spec:
  members: 3
  type: ReplicaSet
  version: "8.0.0"
  security:
    authentication:
      modes: [SCRAM]
  users:
    - name: admin
      db: admin
      passwordSecretRef:
        name: mongo-admin-password
      scramCredentialsSecretName: mongo-admin-scram
      roles:
        - name: root
          db: admin
  statefulSet:
    spec:
      volumeClaimTemplates:
        - metadata:
            name: data-volume
          spec:
            accessModes: [ReadWriteOnce]
            resources:
              requests:
                storage: 10Gi
```

上例使用默认 StorageClass；无默认值时按 Operator 文档配置 `storageClassName`。PVC 模板名称和可覆盖字段依 Operator 版本确定，申请前可用 `kubectl explain mongodbcommunity.spec.statefulSet` 和 [资源规范](https://www.mongodb.com/docs/kubernetes/current/reference/k8s-operator-community-specification/)核对。该实验配置未启用 TLS；生产中要配置证书、节点反亲和和业务最小权限账号。

## 验证

```bash
kubectl -n mongodb get mongodbcommunity mongo-rs -o yaml
kubectl -n mongodb get pods,pvc,svc -o wide
kubectl -n mongodb describe mongodbcommunity mongo-rs
```

确认三个成员均健康、一个 Primary 和两个 Secondary。查看 Operator 生成的连接 Secret 名称（`kubectl -n mongodb get secret`）及服务发现信息；不要在终端或日志中公开连接串密码。在包含 `mongosh` 的客户端中使用 Operator 生成的连接信息，通过可解析的成员地址连接，执行：

```javascript
rs.status().members.map(({name, stateStr, health}) => ({name, stateStr, health}))
db.hello()
db.runCommand({ ping: 1 })
```

连接到 Primary 后可写入测试文档，并从 Secondary 验证复制；读 Secondary 前需设置相应读偏好。客户端必须能解析并访问副本集通告的所有成员地址，集群外访问需另行配置外部访问，单个端口转发不能代替完整的副本集发现。

## 集群内与内网外部访问

这里的“外部”指 Kubernetes 集群外、企业内网或 VPN 内的客户端，不要求开放公网。集群内应用使用 Operator 生成的副本集连接信息；先执行 `kubectl -n mongodb get svc,endpoints` 和 `kubectl -n mongodb get secret` 确认服务及连接 Secret，再从应用所在 Pod 测试 DNS、认证和 `ping`。

集群外使用副本集时，客户端需要分别访问三个成员，而且成员返回的地址也必须从客户端可解析、可连接。`MongoDBCommunity` 应按 [Community 外部访问指南](https://github.com/mongodb/mongodb-kubernetes/blob/master/docs/mongodbcommunity/external_access.md)配置 TLS 和 split-horizon；不要直接把其他 `MongoDB` 资源的 `spec.externalAccess` 复制到本资源。计划好域名和证书后，在现有 `MongoDBCommunity.spec` 中为三个成员分别配置，例如：

```yaml
# spec 内片段，域名和端口须替换；先配置相应的 TLS 证书和 CA。
replicaSetHorizons:
  - internal: mongo-0.db.internal:31181
  - internal: mongo-1.db.internal:31182
  - internal: mongo-2.db.internal:31183
```

创建每个成员独立的 NodePort Service，例如第一位成员：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo-rs-external-0
  namespace: mongodb
spec:
  type: NodePort
  selector:
    statefulset.kubernetes.io/pod-name: mongo-rs-0
  ports:
    - name: mongodb
      port: 27017
      targetPort: 27017
      nodePort: 31181
```

为另两个成员分别创建 Service，将名称和 Pod 选择器中的 `0` 改为 `1`、`2`，端口分别改为 `31182`、`31183`；先用 `kubectl -n mongodb get pods --show-labels` 核实实际 Pod 名称和标签，并确保 NodePort 在集群允许的范围内。内网 DNS 将各成员域名解析到可达的节点地址或相应四层负载均衡器；防火墙只允许授权客户端访问所需端口。证书 SAN 必须包含客户端使用的成员域名，内部服务及节点连接也要满足证书要求。NodePort 对节点开放的接口由集群配置决定，不能默认它只监听内网。

应用证书、更新副本集配置并创建三个 Service 后，从内网客户端验证域名和端口，再以客户端信任的 CA 连接，例如：

```bash
mongosh 'mongodb://mongo-0.db.internal:31181,mongo-1.db.internal:31182,mongo-2.db.internal:31183/?replicaSet=mongo-rs&authSource=admin&tls=true' \
  --tlsCAFile /path/to/ca.crt --username admin --password
```

副本集名称须以实际 `rs.conf()._id` 为准；按 TLS 配置可能还需客户端证书。输入密码后用 `rs.status()` 和 `db.hello()` 验证发现及主节点，并演练切换。启用 TLS、修改成员地址可能触发滚动更新，应按照已安装 Operator 版本的流程执行。

## 运维

检查 Primary 不可用时的选举、客户端重连及恢复，实验前确保数据可丢失并保有多数投票成员。副本不是备份：应单独配置备份并演练恢复；监控复制延迟和 oplog 窗口，见 [MongoDB 监控](../../observability/mongodb/README.md)。PVC Pending 时检查 StorageClass 与事件；Operator 报错时检查资源状态、日志及版本兼容。删除资源前核实 PVC 保留及 PV 回收策略。
