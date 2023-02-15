# cilium

Cilium 是开源软件，用于**透明地**保护使用Linux容器管理平台（如 Docker 和 Kubernetes）部署的应用程序服务之间的网络连接
Cilium 的基础是一种名为 eBPF 的新 Linux 内核技术，它支持在 Linux 本身内动态插入强大的安全可见性和控制逻辑。由于 eBPF 在 Linux 内核中运行，因此可以应用和更新 Cilium 安全策略，而无需对应用程序代码或容器配置进行任何更改

## Hubble

Hubble 是一个完全分布式的网络和安全可观察性平台。它建立在 Cilium 和 eBPF 之上，能够以完全透明的方式深入了解服务的通信和行为以及网络基础设施。通过在 Cilium 之上构建Hubble 可以利用 eBPF 来提高可见性

通过依赖 eBPF，所有可见性都是可编程的，并允许采用动态方法来最大限度地减少开销，同时根据用户的要求提供深入和详细的可见性。Hubble的创建和专门设计是为了充分利用这些新的 eBPF 能力

Hubble主要功能

### Service dependencies & communication map

哪些服务相互通信？多频繁？
服务依赖图是什么样的
正在进行什么 HTTP 调用？
服务从哪些 Kafka 主题消费或生产到哪些主题？

### Network monitoring & alerting

是否有任何网络通信失败？为什么通讯失败？是DNS吗？是应用程序问题还是网络问题？
第 4 层 (TCP) 或第 7 层 (HTTP) 上的通信是否中断？
哪些服务在过去5分钟内遇到了DNS解析问题？
哪些服务最近遇到了TCP连接中断或连接超时？
未响应的 TCP SYN 请求的比率是多少？

### Application monitoring

特定服务或所有集群的 5xx 或 4xx HTTP 响应代码的比率是多少？
我的集群中 HTTP 请求和响应之间的第 95 个和第 99 个百分位延迟是多少(尾延迟)？
哪些服务表现最差？
两个服务之间的延迟是多少？

### Security observability

由于网络策略，哪些服务的连接被阻止？
从集群外部访问了哪些服务？
哪些服务解析了特定的 DNS 名称？


### identity

所有端点都分配了一个身份。身份用于强制端点之间的基本连接。在传统的网络术语中，这相当于第 3 层实施。一个身份由标签标识，并被赋予一个集群范围的唯一标识符。
端点被分配了与端点的安全相关标签匹配的身份，即共享同一组安全相关标签的所有端点将共享相同的身份。此概念允许将策略实施扩展到大量端点，因为随着应用程序的扩展，许多单独的端点通常会共享同一组安全标签

端点的身份是基于与派生到端点的 pod 或容器关联的标签派生的。当一个 pod 或容器启动时，Cilium 会根据容器运行时收到的事件创建一个端点来代表网络上的 pod 或容器。作为下一步，Cilium 将解析所创建端点的身份。每当 Pod 或容器的 Labels 发生变化时，都会重新确认身份并根据需要自动修改。

**安全相关标签**

在派生身份时，并非所有与容器或 pod 关联的标签都是有意义的。标签可用于存储元数据，例如容器启动时的时间戳。 Cilium 需要知道哪些标签是有意义的，并且在推导身份时需要考虑哪些标签。以此目的，
用户需要指定有意义标签的字符串前缀列表。标准行为是包含所有以前缀 id. 开头的标签，例如id.service1、id.service2、id.groupA.service44。启动代理时可以指定有意义的标签前缀列表。

**特殊identity**

Cilium 管理的所有端点都将被分配一个身份。为了允许与不由 Cilium 管理的网络端点进行通信，存在特殊的身份来表示它们。特殊保留标识以字符串 reserved: 为前缀。

destination="
k8s:app=minikube-frpc-app,
k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=minikube-frpc,
k8s:io.cilium.k8s.policy.cluster=default,
k8s:io.cilium.k8s.policy.serviceaccount=default,
k8s:io.kubernetes.pod.namespace=minikube-frpc
"

- [架构分析](https://zhuanlan.zhihu.com/p/474315762)


