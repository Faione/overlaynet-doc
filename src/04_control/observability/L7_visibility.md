# L7网络可见性

cilium默认只提供L3/L4可见性，只有当为服务提供者设置 L7 policy 时，才会提供L7可见性，这是因为L7监控本身的实现是依赖envy代理完成，不过相较于linkerd/istio的sidecar代理，cilium代理中流量截获的实现是依托bpf完成，对用户透明

除此之外，cilium提供基于annotation方式实现L7可见性功能, 注释格式为:
- `<{Traffic Direction}/{L4 Port}/{L4 Protocol}/{L7 Protocol}>`
- 常见端口如http默认80，https默认443
- 只有为pod注解时才会生效
- 如果配置了L7 policy，则不会产生相关标识

**为client类应用加上Egress注解**

```shell
$ kubectl -n test-guid-0 annotate pod tiefighter io.cilium.proxy-visibility="<Egress/53/UDP/DNS>,<Egress/80/TCP/HTTP>"

# 取消注解
$ kubectl -n test-guid-0 annotate pod tiefighter io.cilium.proxy-visibility-
```

**为server类应用加上Ingress注解**

```shell
$ kubectl -n test-guid-0 annotate pod deathstar-6c94dcc57b-jgszb io.cilium.proxy-visibility="<Ingress/80/TCP/HTTP>"
```