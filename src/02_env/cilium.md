# cilium

## 环境部署


cilium集群对cni有限制，启动集群时通过`--cni false`让minikube不安装cni插件，或者使用`--cni cilium`安装cilium官方网路插件(版本可能落后)
- 注意指定`--network-plugin=cni`让集群使用cni

### 安装cilium

- 安装脚本
  - `--set hubble.relay.enabled=true` 来启动hubble基础服务
  - ` --set hubble.ui.enabled=true` 来启动hubble ui

```shell
REPO_NAME="cilium"
REPO_URL="https://helm.cilium.io/"

RELEASE="cilium"
CHART="cilium/cilium"
VERSION="1.12.1"
NAMESPACE="kube-system"

helm repo add $REPO_NAME $REPO_URL

helm install $RELEASE $CHART --version $VERSION \
  --namespace $NAMESPACE \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```

### 安装CLI

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

### 测试样例

样例场景为飞船向星球着陆
- 两类飞机，一个星球
- xwing不允许着陆
- tiefighter不允许着陆在星球的特定位置

```shell
# 安装测试样例
$ kubectl create -f https://raw.githubusercontent.com/cilium/cilium/1.12.1/examples/minikube/http-sw-app.yaml

# 清理样例
$ kubectl delete -f https://raw.githubusercontent.com/cilium/cilium/1.12.1/examples/minikube/http-sw-app.yaml

# 删除规则
$ kubectl delete cnp rule1
```

```shell
# tiefighter 请求着陆
$ kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

# xwing请求着陆
$ kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

# tiefighter 请求在非法地点着陆
$ kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
```

应用

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: deathstar
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
  labels:
    app.kubernetes.io/name: xwing
    org: alliance
    class: xwing
spec:
  containers:
  - name: spaceship
    image: docker.io/tgraf/netperf
```

流量规则

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
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
```


```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test-guid-0
---
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