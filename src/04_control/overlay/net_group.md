# 网络分组

分组功能的目标, 在底层网络相互连通的前提下, 通过标签使得: 
- 同一group能相互访问
- 不同group无法相互访问

**base_group_rule**
- `group-{GROUP}`: `{GROUP}` 为分组名称，需要符合k8s标签规范
- `in/out`: 标签值用于占位，也可以表示当前pod相对group的状态

**across_namespace_rule**
- 当存在跨namespace的group分组需求时，需要在原有`base_group_rule`增加额外的namespace策略，将服务暴漏给请求者
- `{NSD}`: 请求的目标所在的namespace
- `{NSS}`: 请求者所在的namespace

```yaml
# base_group_rule
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: group-{GROUP}-rule
  namespace: {NS}
spec:
  endpointSelector:
    matchLabels:
      group-{GROUP}: in
  ingress:
  - fromEndpoints:
    - matchLabels:
        group-{GROUP}: in

# across_namespace_rule
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: group-{GROUP}-rule
  namespace: {NSD}
spec:
  endpointSelector:
    matchLabels:
      group-{GROUP}: in
  ingress:
  - fromEndpoints:
    - matchLabels:
        k8s:io.kubernetes.pod.namespace: {NSS}
        group-{GROUP}: in
```

### Demo

#### 应用配置

实验中有`test-guid-1` 与 `test-guid-2` 两个namespace，每个namespace中有三个端点，其中 tiefighter、xwing 为client应用，deathstar 为服务应用

```yaml
# raw_apps
apiVersion: v1
kind: Namespace
metadata:
  name: {NS}
---
apiVersion: v1
kind: Service
metadata:
  name: deathstar
  namespace: {NS}
  labels:
    app: deathstar
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: deathstar
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deathstar
  namespace: {NS}
  labels:
    app: deathstar
spec:
  replicas: 2
  selector:
    matchLabels:
      app: deathstar
  template:
    metadata:
      labels:
        app: deathstar
    spec:
      containers:
      - name: deathstar
        image: docker.io/cilium/starwars
---
apiVersion: v1
kind: Pod
metadata:
  name: tiefighter
  namespace: {NS}
  labels:
    app: tiefighter
spec:
  containers:
  - name: spaceship
    image: docker.io/tgraf/netperf
---
apiVersion: v1
kind: Pod
metadata:
  name: xwing
  namespace: {NS}
  labels:
    app: xwing
spec:
  containers:
  - name: spaceship
    image: docker.io/tgraf/netperf
```

#### 应用分组

|    guid     |    pod     | group |
| :---------: | :--------: | :---: |
| test-guid-1 | deathstar  |   A   |
| test-guid-1 | tiefighter | A、B  |
| test-guid-1 |   xwing    |   B   |
| test-guid-2 | deathstar  |   B   |
| test-guid-2 | tiefighter |   A   |
| test-guid-2 |   xwing    |   -   |

构造应用并部署

```shell
$ sed -e "s/{NS}/test-guid-1/g" raw_apps.yaml > app_1/apps.yaml

$ kubectl apply -f ...
```

测试 test-guid-1 与 test-guid-2 中应用的连通性

```shell
#! /bin/bash

namespaces=(test-guid-1 test-guid-2)
clients=(tiefighter xwing)

DEFAULT_TIMEOUT=3

for nsc in ${namespaces[@]}
do
for cl in ${clients[@]}
do
for nss in ${namespaces[@]}
do

echo $nsc"/"$cl" -> "$nss"/deathstar: "
kubectl -n $nsc exec $cl -- curl -s  --connect-timeout $DEFAULT_TIMEOUT -XPOST deathstar.$nss.svc.cluster.local/v1/request-landing

done
done
done

echo "test over"
```

![](./img/2022-09-30-10-18-00.png)


为每个端点的pod增加分组标签

```shell
$ kubectl -n test-guid-1 label pod tiefighter group-a=in

$ kubectl -n test-guid-1 label pod tiefighter group-b=in
```

为每个命名空间构造分组规则并部署

```shell
# 分组A规则
$ sed -e "s/{GROUP}/a/g" -e "s/{NS}/test-guid-1/g" base_group_rule.yaml > app_1/group_a_rule.yaml

# 分组B规则
$ sed -e "s/{GROUP}/b/g" -e "s/{NS}/test-guid-1/g" base_group_rule.yaml > app_1/group_b_rule.yaml

kubectl apply -f ...
```