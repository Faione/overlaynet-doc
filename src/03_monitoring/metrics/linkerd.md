# linkerd

**tcp_open_total**

```yml
tcp_open_total
{
    app="emoji-svc",
    client_id="web.emojivoto.serviceaccount.identity.linkerd.cluster.local",
    control_plane_ns="linkerd",
    deployment="emoji",
    direction="inbound",
    instance="172.17.0.10:4191",
    job="linkerd-proxy",
    namespace="emojivoto",
    peer="src",
    pod="emoji-55c59cf485-rkszq",
    pod_template_hash="55c59cf485",
    srv_kind="default",
    srv_name="all-unauthenticated",
    target_addr="172.17.0.10:8080",
    target_ip="172.17.0.10",
    target_port="8080",
    tls="true",
    version="v11",
    workload_ns="emojivoto"
} 
    1
```

**http_client_requests_total**

```yml
http_client_requests_total
{
    client="k8s",
    code="200",
    component="destination",
    instance="172.17.0.4:9996",
    job="linkerd-controller",
    method="get"
} 
    1
```
