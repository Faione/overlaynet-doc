# 使用helm搭建cilium

- [使用helm搭建cilium](#使用helm搭建cilium)
  - [环境准备](#环境准备)
  - [监控配置](#监控配置)

## 环境准备

- 安装脚本
  - `--set hubble.metrics.enabled=""` 来关闭metric
  - `--reuse-values`可以复用已有的value, 搭配`helm upgrade`已避免覆盖先前配置
  - `HUBBLE_METRIC_CONFIG`用来设置需要从hubble中获取的监控指标以及其配置
    - [hubble监控设置](https://docs.cilium.io/en/stable/operations/metrics/)

**更改现有配置**

```shell
RELEASE="cilium"
CHART="cilium/cilium"
VERSION="1.12.1"
NAMESPACE="kube-system"

HUBBLE_METRIC_CONFIG="{flow:destinationContext=pod;sourceContext=pod,http:destinationContext=pod;sourceContext=pod}"

helm upgrade $RELEASE $CHART --version $VERSION \
  --namespace $NAMESPACE \
  --reuse-values \
  --set hubble.relay.enabled=true \
  --set hubble.metrics.enabled=$HUBBLE_METRIC_CONFIG \
  --set hubble.ui.enabled=true
```


**从零开始**

```shell
REPO_NAME="cilium"
REPO_URL="https://helm.cilium.io/"

RELEASE="cilium"
CHART="cilium/cilium"
VERSION="1.12.1"
NAMESPACE="kube-system"

HUBBLE_METRIC_CONFIG="{flow:destinationContext=pod;sourceContext=pod,http:destinationContext=pod;sourceContext=pod}"


helm repo add $REPO_NAME $REPO_URL

helm install $RELEASE $CHART --version $VERSION \
  --namespace $NAMESPACE \
  --set hubble.relay.enabled=true \
  --set hubble.metrics.enabled=$HUBBLE_METRIC_CONFIG \
  --set hubble.ui.enabled=true 
```

hubble本身cilium隔离，当hubble先于cilium启动时，cilium无法获得其信息导致不能管理，因此需要严格两者的启动顺序，或在cilium启动之后，将未被管理的部分纳入其中(重新pod，让cilium感知)

```shell
$ kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOSTNETWORK:.spec.hostNetwork --no-headers=true | grep '<none>' | awk '{print "-n "$1" "$2}' | xargs -L 1 -r kubectl delete pod
```

**prometheus**

部署prometheus

```shell
$ kubectl apply -f <slim-prometheus-yaml>
```

查看prometheus

```shell
$ kubectl -n cilium-monitoring port-forward service/prometheus --address 0.0.0.0 3000:9090
```

## 监控配置

hubble meric配置`flow:destinationContext=pod;sourceContext=pod`，使得每一条flow的metric，都会按`namespace/podname`的格式，增加`destination` 与 `source` label，即这条流产生的源节点与目的节点，通过这些信息，能够不断聚合得到关系图谱
- `protocol`标识了flow的协议, 如HTTP,TCP,UDP等
- `verdict`标识了cilium对flow的处理，DROPPED表示丢弃，FORWARDED表示让行
- `type`标识了flow的层级