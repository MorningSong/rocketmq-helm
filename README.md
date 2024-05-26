# RocketMQ Helm Chart

https://github.com/itboon/rocketmq-helm

## 版本兼容性

- Kubernetes 1.18+
- Helm 3.3+
- RocketMQ `>= 4.5`

## 添加 helm 仓库

``` shell
## 添加 helm 仓库
helm repo add rocketmq-repo https://helm-charts.itboon.top/rocketmq
helm repo update rocketmq-repo
```

## 部署案例

``` shell
## 部署一个最小化的 rocketmq 集群
## 这里关闭持久化存储，仅演示部署效果
helm upgrade --install rocketmq \
  --namespace rocketmq-demo \
  --create-namespace \
  --set broker.persistence.enabled="false" \
  rocketmq-repo/rocketmq
```

``` shell
## 部署测试集群, 启用 Dashboard (默认已开启持久化存储)
helm upgrade --install rocketmq \
  --namespace rocketmq-demo \
  --create-namespace \
  --set dashboard.enabled="true" \
  rocketmq-repo/rocketmq
```

### 部署高可用集群版

``` shell
## rocketmq-cluster 默认部署 2个 master 节点
## 每个 master 具有1个副节点，共4个 broker 节点
helm upgrade --install rocketmq \
  --namespace rocketmq-demo \
  --create-namespace \
  rocketmq-repo/rocketmq-cluster

```

``` shell
## 部署 3个 master 节点，每个 master 具有1个副节点，共6个 broker 节点
helm upgrade --install rocketmq \
  --namespace rocketmq-demo \
  --create-namespace \
  --set broker.size.master="3" \
  rocketmq-repo/rocketmq-cluster

```

``` shell
## 调整内存配额
helm upgrade --install rocketmq \
  --namespace rocketmq-demo \
  --create-namespace \
  --set broker.master.jvm.maxHeapSize="4G" \
  --set broker.master.resources.requests.memory="6Gi" \
  rocketmq-repo/rocketmq-cluster

```

> 具体资源配额请根据实际环境调整，参考 [examples](https://github.com/itboon/rocketmq-helm/tree/main/examples)

## 部署详情

### 集群外访问

可以将 proxy 暴露到集群外，支持 `LoadBalancer` 和 `NodePort`

> proxy 是 RocketMQ 5.x 版本新增的模块，支持 grpc 和 remoting 协议，SDK接入请参考[官方文档](https://rocketmq.apache.org/zh/docs/sdk/01overview)

``` yaml
proxy:
  service:
    annotations: {}
    type: LoadBalancer  ## LoadBalancer or NodePort
```

### 可选组件

``` yaml
## 关闭 proxy
proxy:
  enabled: false  ## 默认 true

## 关闭 dashboard
dashboard:
  enabled: false  ## 默认 true
```

### Dashboard 登录认证

Dashboard admin 帐号密码:

``` yaml
dashboard:
  enabled: true
  auth:
    enabled: true
    users:
      - name: admin
        password: admin
        isAdmin: true
      - name: user01
        password: userPass
```

### 镜像仓库

``` yaml
image:
  repository: apache/rocketmq
  tag: 5.1.4
```

### 部署特定版本

``` shell
helm upgrade --install rocketmq \
  --namespace rocketmq-demo \
  --create-namespace \
  --set image.tag="5.2.0" \
  rocketmq-repo/rocketmq
```

### 内存管理

集群每个模块提供堆内存管理，例如 `--set broker.master.jvm.maxHeapSize="1024M"` 将堆内存设置为 `1024M`，默认 `Xms` `Xmx` 相等。堆内存配额应该与 Pod `resources` 相匹配。

> 可使用 `jvm.javaOptsOverride` 对 jvm 参数进行修改，设置了此参数则 `maxHeapSize` 失效。

```yaml
broker:
  master:
    jvm:
      maxHeapSize: 1024M
      # javaOptsOverride: "-Xms1024M -Xmx1024M -XX:+UseG1GC"
    resources:
      requests:
        cpu: 100m
        memory: 2Gi

nameserver:
  jvm:
    maxHeapSize: 1024M
    # javaOptsOverride: "-Xms1024M -Xmx1024M -XX:+UseG1GC"
  resources:
    requests:
      cpu: 100m
      memory: 2Gi
```

## Broker 集群架构

### 单 Master 模式

``` yaml
broker:
  size:
    master: 1
    replica: 0
```

### 多 Master 模式

一个集群无Slave，全是Master，例如2个Master或者3个Master，这种模式的优缺点如下：

- 优点：配置简单，单个Master宕机或重启维护对应用无影响，性能最高；
- 缺点：单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到影响。

``` yaml
broker:
  size:
    master: 3
    replica: 0
```

### 多 Master 多 Slave 模式

每个Master配置一个Slave，有多对Master-Slave，HA采用异步复制方式，主备有短暂消息延迟（毫秒级），这种模式的优缺点如下：

- 优点：Master宕机后，消费者仍然可以从Slave消费，而且此过程对应用透明，不需要人工干预，性能同多Master模式几乎一样。

``` yaml
broker:
  size:
    master: 3
    replica: 1
# 3个 master 节点，每个 master 具有1个副节点，共6个 broker 节点
```
