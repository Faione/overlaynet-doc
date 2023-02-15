# 网络分组

## 一、分组创建

```go
type NetGroup struct{
    Name string
    State State
    Rules []Rule
    Entrypoints []Entrypoint
}
```

新增分组 `foo`

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "group-foo"
specs:
  - description: "For endpoints with env=prod, only allow if source also has label env=prod"
    endpointSelector:
      matchLabels:
        group-foo: in
    ingress:
    - fromRequires:
      - matchLabels:
          group-foo: in
```

### 基础规则

允许同命名空间中的同组进行访问

```yml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: group-foo-base
spec:
  endpointSelector:
    endpointSelector:
      matchLabels:
        group-foo: in
  ingress:
  - fromEndpoints:
    - matchLabels:
        {}
```

### 分组创建流程

1. 校验分组信息，同一个命名空间中不允许重复
2. 创建分组规则，根据需求创建基础规则(默认为仅允许同组之间相互访问)
3. 如给出相应的节点信息，则需要为目标增加label
4. 操作完成，修改group状态的为ready

## 二、分组修改


将节点加入分组中

```shell
$ kubectl label pod <entrypoint> group-foo=in
```

## 三、分组删除

将Entrypoint移出分组

```
$ kubectl label pod <entrypoint> group-foo-
```

### 分组删除流程

1. 将Entrypoint移除分组
2. 删除分组相关的所有rule

## 四、分组查询


查询当前命名空间下的所有分组

查询分组的详细信息