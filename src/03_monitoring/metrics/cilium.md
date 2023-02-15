# cilium

- [cilium](#cilium)
  - [service map相关指标](#service-map相关指标)
  - [统计相关指标](#统计相关指标)
  - [Cilium exporter 指标](#cilium-exporter-指标)

## service map相关指标

**hubble_flows_processed_total**

hubble_flows_processed_total对边有基础的描述

```yml
hubble_flows_processed_total
{
    # service map info
    source="default/tiefighter",
    destination="default/deathstar-6c94dcc57b-2chb9",
    
    # flow info
    type="L7",
    protocol="HTTP",
    subtype="HTTP",
    verdict="DROPPED",  

    # prometheus job info
    app_kubernetes_io_managed_by="Helm",
    instance="192.168.50.139:9965",
    job="kubernetes-endpoints",
    k8s_app="hubble",
    namespace="kube-system",
    service="hubble-metrics"
} 
    2
```

**hubble_http_requests_total**

hubble_http_requests_total描述http请求

```yml
hubble_http_requests_total
{
    # service map info
    source="default/tiefighter",
    destination="default/deathstar-6c94dcc57b-2chb9",
    
     # l7 info
    protocol="HTTP/1.1",
    method="POST",  

    # prometheus job info
    app_kubernetes_io_managed_by="Helm",
    instance="192.168.50.139:9965",
    job="kubernetes-endpoints",
    k8s_app="hubble",
    namespace="kube-system",
    service="hubble-metrics"
} 
    1
```
**hubble_http_responses_total**

hubble_http_requests_total描述http响应
- 状态码可用于统计

```yml
hubble_http_responses_total
{   
    # service map info
    source="default/deathstar-6c94dcc57b-2chb9",
    destination="default/tiefighter",
    
    # l7 info
    method="POST",
    status="200",   

    # prometheus job info
    app_kubernetes_io_managed_by="Helm",
    instance="192.168.50.139:9965",
    job="kubernetes-endpoints",
    k8s_app="hubble",
    namespace="kube-system",
    service="hubble-metrics"
} 
    4
```


## 统计相关指标

**hubble_http_request_duration_seconds_bucket**

http请求时间的频数分布直方图数据
- `le`: less than or equal, `le=0.005`即频数区间`[0, 0.005)`, 即延迟在 0.005 秒以内的请求数量
- 使用 histogram_quantile(0.5, prometheus_http_request_duration_seconds_bucket) 计算50分位尾延迟

```yml
hubble_http_request_duration_seconds_bucket
{
    # service map info
    source="test-guid-0/deathstar-6c94dcc57b-jgszb",
    destination="test-guid-0/tiefighter",
    
    # l7 info
    method="POST",

    # stat info
    le="0.005",

    # prometheus job info
    app_kubernetes_io_managed_by="Helm",
    k8s_app="hubble",
    instance="192.168.50.192:9965",
    job="kubernetes-endpoints",
    namespace="kube-system",
    service="hubble-metrics"
} 
    14
```

**hubble_http_request_duration_seconds_sum**

http请求的总时间
- 结合 http请求的总次数 可计算平均请求时间

```yml
hubble_http_request_duration_seconds_sum
{
    # service map info
    source="test-guid-0/deathstar-6c94dcc57b-jgszb",
    destination="test-guid-0/tiefighter",
    
    # l7 info
    method="POST",

    # prometheus job info
    instance="192.168.50.192:9965",
    job="kubernetes-endpoints",
    k8s_app="hubble",
    app_kubernetes_io_managed_by="Helm",
    namespace="kube-system",
    service="hubble-metrics"
}
    2979605 
```

**hubble_http_request_duration_seconds_count**

http请求的总次数

```yml
hubble_http_request_duration_seconds_count
{
    # service map info
    source="test-guid-0/deathstar-6c94dcc57b-jgszb",    
    destination="test-guid-0/tiefighter",
    method="POST",

    # prometheus job info   
    namespace="kube-system",
    service="hubble-metrics",
    instance="192.168.50.192:9965",
    job="kubernetes-endpoints",
    k8s_app="hubble",
    app_kubernetes_io_managed_by="Helm"
} 
    14
```

## Cilium exporter 指标

支持自定义label

**cilium_http_response_latence**

延迟

```yml
cilium_http_response_latence
{
    # service map info
    source="test-guid-0/deathstar-6c94dcc57b-jh55s", 
    destination="test-guid-0/tiefighter",

    # l4 info
    port="80", 
    
    # l7 info
    protocol="HTTP/1.1", 
    code="200", 
    method="POST", 
    url="http://deathstar.test-guid-0.svc.cluster.local/v1/request-landing", 
    
    # stat info
    quantile="0.5",

    # prometheus job info
    instance="172.16.31.38:19091", 
    job="cilium-exporter"
}
    2979605 # value: latency (ns)
```

**cilium_http_response_latence_count**

响应计数

```yml
cilium_http_response_latence_count
{
    # service map info
    source="test-guid-0/deathstar-6c94dcc57b-jh55s", 
    destination="test-guid-0/tiefighter",

    # l4 info
    port="80", 
    
    # l7 info
    protocol="HTTP/1.1", 
    code="200", 
    method="POST", 
    url="http://deathstar.test-guid-0.svc.cluster.local/v1/request-landing", 

    # prometheus job info
    instance="172.16.31.38:19091", 
    job="cilium-exporter"
}   
    32 # value: count
```

**cilium_http_response_latence_sum**

延迟总和
- 结合响应计数计算平均延迟

```yml
cilium_http_response_latence_count
{
    # service map info
    source="test-guid-0/deathstar-6c94dcc57b-jh55s", 
    destination="test-guid-0/tiefighter",

    # l4 info
    port="80", 
    
    # l7 info
    protocol="HTTP/1.1", 
    code="200", 
    method="POST", 
    url="http://deathstar.test-guid-0.svc.cluster.local/v1/request-landing", 

    # prometheus job info
    instance="172.16.31.38:19091", 
    job="cilium-exporter"
}  
    53803229 # value: total latency (ns)
```