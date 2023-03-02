# 使用Helm安装Cilium

- [使用Helm安装Cilium](#使用helm安装cilium)
  - [一、随集群部署](#一随集群部署)
    - [1. 准备集群](#1-准备集群)
      - [minikube](#minikube)
      - [kubeadm](#kubeadm)
    - [2. 部署cilium](#2-部署cilium)
  - [二、在运行的集群上部署](#二在运行的集群上部署)
    - [1. CNI Chaining](#1-cni-chaining)
    - [2. CNI Migration](#2-cni-migration)
    - [3. CNI Chaining模式下的Cilium功能](#3-cni-chaining模式下的cilium功能)
  - [三、接管Pod网络](#三接管pod网络)
  - [四、Cilium CLI](#四cilium-cli)
  - [五、测试样例](#五测试样例)
  - [~~Cilium Metric设置~~](#cilium-metric设置)

## 一、随集群部署

### 1. 准备集群

#### minikube

使用 `minikube` 创建集群并安装 cilium 网卡，需要注意:
- 集群中至少存在2个节点，否则部分 cilium 功能无法正常使用
- 指定 `--network-plugin=cni` 让k8s使用 cni
- 可选 `--cni cilium` 安装 minikube 提供的 cilium 网卡，但此网卡版本会比较落后，建议稍后安装，而在此处设置 `--cni false`

部署脚本参考

```shell
minikube start -p cilium \
  --driver kvm2 \
  --kubernetes-version=v1.23.0 \
  --cpus=2 \
  --memory=4g \
  --nodes 2 \
  --image-mirror-country='cn' \
  --cni=false \
  --network-plugin=cni
```

#### kubeadm

[install-cilium-while-using-kubeadm](https://docs.cilium.io/en/stable/installation/k8s-install-kubeadm/)

### 2. 部署cilium

随集群部署时，cilium 会自动为集群安装 cilium 网卡来接管集群网络。hubble用以聚合，展示cilium的数据，其由relay和ui两部分组成，relay负责从原始的网络信息聚合出flow数据，而ui则提供丰富的查询选项来将数据展示给用户，因此如需要获取cilium的监控数据，必须部署hubble relay, 而ui可以按照需要选择部署

```shell
REPO_NAME="cilium"
REPO_URL="https://helm.cilium.io/"

RELEASE="cilium"
CHART="cilium/cilium"
VERSION="1.12.7"
NAMESPACE="kube-system"

# 添加 cilium repo
helm repo add $REPO_NAME $REPO_URL

# 安装 cilium，并开启 hubble
helm install $RELEASE $CHART --version $VERSION \
  --namespace $NAMESPACE \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```

## 二、在运行的集群上部署

运行的集群上已经存在如flannel, calico等网卡，此时cilium无法完全接管网络，从而无法发挥全部功能，甚至不能正常运行，而为解决这个问题，目前存在两种方案
- cni chaining: k8s的网络支持链式处理，这意味着能够通过配置将cilium加入到网络链中，从而对集群网络进行介入，支持部分cilium功能
- cni migration: 替换k8s的网卡，然后再接管所有正在运行的pod的网络，这种方式能够完全发挥cilium网卡的功能

### 1. CNI Chaining

> `/etc/cni/net.d` 目录下以 `.conflist` 结尾的为集群网卡配置，k8s支持同时存在多个网卡配置，而以 `.conflist` 结尾是为了与 `.conf` 结尾的其他配置文件进行区分

一般集群初始化时，会选择flannel作为集群网卡，而作为一种基于iptables的网卡，flannel非常简洁，而代价就是功能相对单一。在集群中任意一个node的 `/etc/cni/net.d` 目录下，可以找到集群网络的配置，minikube 中 flannel的网络配置一般为

```json
# /etc/cni/net.d/10-flannel.conflist
{
  "name": "cbr0",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
```

为集群配置 cilium cni chaining，需要增加一个 configMap 来配置链式处理, 使用 `kubectl apply -f chaining.yaml` 来创建 cni-configuration

```yaml
# chaining.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cni-configuration
  namespace: kube-system
data:
  cni-config: |-
    {
        "name": "generic-veth",
        "cniVersion": "0.3.1",
        "plugins": [
            {
                "type": "flannel",
                "delegate": {
                    "hairpinMode": true,
                    "isDefaultGateway": true
                }
            },
            {
                "type": "portmap",
                "capabilities": {
                    "portMappings": true
                }
            },
            {
                "type": "cilium-cni"
            }
        ]
    }
```

安装 cilium, 设置 `chainingMode` 为 `generic-veth`，并应用cni-configuration
- cilium v1.13.0 存在一些问题，因此使用 1.12.7

```shell
helm install cilium cilium/cilium --version 1.12.7 \
  --namespace=kube-system \
  --set cni.chainingMode=generic-veth \
  --set cni.customConf=true \
  --set cni.configMap=cni-configuration \
  --set tunnel=disabled \
  --set enableIPv4Masquerade=false
```

- [参考文档](https://docs.cilium.io/en/stable/installation/cni-chaining/)
- [genric-veth](https://docs.cilium.io/en/stable/installation/cni-chaining-generic-veth/)

### 2. CNI Migration

### 3. CNI Chaining模式下的Cilium功能

|  功能  | 状态  |
| :----: | :---: |
| L3监控 | 正常  |
| L4监控 | 正常  |
| L7监控 | 正常  |
| L7控制 | 异常  |

## 三、接管Pod网络

如果集群中已经存在一些创建好的pod时，需要通过重新部署这些pod来让cilium对他们的网络进行管理

```shell
kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOSTNETWORK:.spec.hostNetwork --no-headers=true | grep '<none>' | awk '{print "-n "$1" "$2}' | xargs -L 1 -r kubectl delete pod
```

## 四、Cilium CLI

**cilium cli**

cilium cli是安装、管理、与调试k8s集群中的cilium集群的命令行工具

```shell
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

- 常用命令

```shell
# 查看状态
$ cilium status

# 测试(部分pod会访问404网站，因而测试总是会失败，略过)
$ cilium connectivity test
```

**hubble cli**

hubble cli是用来观察和检查cilium集群中最近的路由流量的命令行工具

```shell
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
```

- 常用命令

```shell
# 查看 hubble状态, 需要通过`cilium hubble port-forward&`暴露服务给cli
$ hubble status

$ hubble observe
```

## 五、测试样例

样例场景为飞船向星球着陆
- 两类飞机，一个星球
- xwing不允许着陆
- tiefighter不允许着陆在星球的特定位置

创建测试应用

```shell
kubectl apply -f test-demo.yaml
```

如下命令可以进行测试体验

```shell
# tiefighter 请求着陆
$ kubectl -n test-guid-0 exec tiefighter -- curl -s -XPOST deathstar.test-guid-0.svc.cluster.local/v1/request-landing

# xwing请求着陆
$ kubectl -n test-guid-0 exec xwing -- curl -s -XPOST deathstar.test-guid-0.svc.cluster.local/v1/request-landing

# tiefighter 请求在非法地点着陆
$ kubectl -n test-guid-0 exec tiefighter -- curl -s -XPUT deathstar.test-guid-0.svc.cluster.local/v1/exhaust-port
```

```yaml
## test-demo.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test-guid-0
---
## 流量规则
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
  namespace: test-guid-0
spec:
  description: "L7 policy to restrict access to specific HTTP call"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "POST"
          path: "/v1/request-landing"
---
## 测试应用
apiVersion: v1
kind: Service
metadata:
  name: deathstar
  namespace: test-guid-0
  labels:
    app.kubernetes.io/name: deathstar
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    org: empire
    class: deathstar
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deathstar
  namespace: test-guid-0
  labels:
    app.kubernetes.io/name: deathstar
spec:
  replicas: 2
  selector:
    matchLabels:
      org: empire
      class: deathstar
  template:
    metadata:
      labels:
        org: empire
        class: deathstar
        app.kubernetes.io/name: deathstar
    spec:
      containers:
      - name: deathstar
        image: docker.io/cilium/starwars
---
apiVersion: v1
kind: Pod
metadata:
  name: tiefighter
  namespace: test-guid-0
  labels:
    org: empire
    class: tiefighter
    app.kubernetes.io/name: tiefighter
spec:
  containers:
  - name: spaceship
    image: docker.io/tgraf/netperf
---
apiVersion: v1
kind: Pod
metadata:
  name: xwing
  namespace: test-guid-0
  labels:
    app.kubernetes.io/name: xwing
    org: alliance
    class: xwing
spec:
  containers:
  - name: spaceship
    image: docker.io/tgraf/netperf
```

## ~~Cilium Metric设置~~

cilium部署使可通过 `--set hubble.metrics.enabled="HUBBLE_METRIC_CONFIG"` 来启动cilium自带的 metric, `HUBBLE_METRIC_CONFIG` 为metric的生成规则

```shell
RELEASE="cilium"
CHART="cilium/cilium"
VERSION="1.12.7"
NAMESPACE="kube-system"

HUBBLE_METRIC_CONFIG="{flow:destinationContext=pod;sourceContext=pod,http:destinationContext=pod;sourceContext=pod}"

helm upgrade $RELEASE $CHART --version $VERSION \
  --namespace $NAMESPACE \
  --reuse-values \
  --set hubble.relay.enabled=true \
  --set hubble.metrics.enabled=$HUBBLE_METRIC_CONFIG \
  --set hubble.ui.enabled=true
```

配置`flow:destinationContext=pod;sourceContext=pod`，使得每一条flow的metric，都会按`namespace/podname`的格式，增加`destination` 与 `source` label，即这条流产生的源节点与目的节点，通过这些信息，能够聚合得到关系图谱，[其他配置信息](https://docs.cilium.io/en/stable/operations/metrics/)
- `protocol`标识了flow的协议, 如HTTP,TCP,UDP等
- `verdict`标识了cilium对flow的处理，DROPPED表示丢弃，FORWARDED表示让行
- `type`标识了flow的层级