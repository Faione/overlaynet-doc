# 数据结构

## service map数据结构

**hubble ui 数据结构**

hubble ui 源码中的mock数据

```yml
# HubbleLink(edge)
[
  {
    id: 'reserved:world:outgoing',
    sourceId: 'reserved:world:outgoing',
    destinationId: 'a8de92d55119c9a6bb6a6dd66bcf012fabefb32d',
    destinationPort: 443,
    ipProtocol: IPProtocol.TCP,
    verdict: Verdict.Forwarded,
  },
  ...
]

# HubbleService(node)
[
  {
    id: 'reserved:world:outgoing',
    name: 'World',
    namespace: selectedNamespace,
    labels: [{ key: 'reserved:world', value: '' }],
    dnsNames: [],
    egressPolicyEnforced: false,
    ingressPolicyEnforced: false,
    visibilityPolicyStatus: '?unknown?',
    creationTimestamp: dataHelpers.msToPbTimestamp(Date.now()),
  },
  ...
]
```

**旧gluenet ui 数据结构**

基于jaeger trace的数据

```yml
# Links
"dag_links": [
    {
        "source": "nginx-thrift",
        "target": "home-timeline-service",
        "traceIds": [
            "00736683e0e57ee0"
        ],
        "value": 4
    },
    ...
]

# Nodes(not array)
"dag_nodes": {
    "compose-post-service": {
        "agent_guid": "64809fc89e07746706983c09d891cf21",
        "guid": "f18af130113b0db7554d46d9a994b873",
        "id": "docker://526d583390d50a5b9fbc5b4dc8f81bd475d3f65789c0f6b8c0d54d792459f00d",
        "image": "yg397/social-network-microservices:latest",
        "name": "compose-post-service",
        "platform": "kubernetes",
        "pod": "compose-post-service-6d9984c876-6cl6f"
    },
    ...
}
```

**hubble observer**

gRpc请求hubble获得的数据(以json编码输出)
- Summary中为总结性的内容, 包括协议，以及协议的详细信息
- time记录的是实际flow产生的时间

TCP flow

```json
{
    "flow": {
        "time": "2022-09-22T02:45:37.120249940Z",
        "verdict": "FORWARDED",
        "ethernet": {
            "source": "86:eb:f5:0d:13:13",
            "destination": "32:57:95:25:31:5d"
        },
        "IP": {
            "source": "10.0.0.104",
            "destination": "10.0.0.240",
            "ipVersion": "IPv4"
        },
        "l4": {
            "TCP": {
                "source_port": 8181,
                "destination_port": 50810,
                "flags": {
                    "PSH": true,
                    "ACK": true
                }
            }
        },
        "source": {
            "ID": 357,
            "identity": 29648,
            "namespace": "cilium-test",
            "labels": [
                "k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=cilium-test",
                "k8s:io.cilium.k8s.policy.cluster=default",
                "k8s:io.cilium.k8s.policy.serviceaccount=echo-other-node",
                "k8s:io.kubernetes.pod.namespace=cilium-test",
                "k8s:kind=echo",
                "k8s:name=echo-other-node"
            ],
            "pod_name": "echo-other-node-6d8549cc6c-wxdvd",
            "workloads": [
                {
                    "name": "echo-other-node-6d8549cc6c",
                    "kind": "ReplicaSet"
                }
            ]
        },
        "destination": {
            "identity": 1,
            "labels": [
                "reserved:host",
                "reserved:kube-apiserver"
            ]
        },
        "Type": "L3_L4",
        "node_name": "cilium",
        "reply": true,
        "event_type": {
            "type": 4,
            "sub_type": 3
        },
        "traffic_direction": "INGRESS",
        "trace_observation_point": "TO_STACK",
        "is_reply": true,
        "Summary": "TCP Flags: ACK, PSH"
    },
    "node_name": "cilium",
    "time": "2022-09-22T02:45:37.120249940Z"
}
```


Http flow

```json
{
    "flow": {
        "time": "2022-09-22T07:15:13.011142862Z",
        "verdict": "FORWARDED",
        "IP": {
            "source": "10.0.0.10",
            "destination": "10.0.1.61",
            "ipVersion": "IPv4"
        },
        "l4": {
            "TCP": {
                "source_port": 80,
                "destination_port": 51794
            }
        },
        "source": {
            "ID": 235,
            "identity": 4051,
            "namespace": "default",
            "labels": [
                "k8s:app.kubernetes.io/name=deathstar",
                "k8s:class=deathstar",
                "k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default",
                "k8s:io.cilium.k8s.policy.cluster=default",
                "k8s:io.cilium.k8s.policy.serviceaccount=default",
                "k8s:io.kubernetes.pod.namespace=default",
                "k8s:org=empire"
            ],
            "pod_name": "deathstar-6c94dcc57b-hljrj"
        },
        "destination": {
            "identity": 13798,
            "namespace": "default",
            "labels": [
                "k8s:app.kubernetes.io/name=tiefighter",
                "k8s:class=tiefighter",
                "k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default",
                "k8s:io.cilium.k8s.policy.cluster=default",
                "k8s:io.cilium.k8s.policy.serviceaccount=default",
                "k8s:io.kubernetes.pod.namespace=default",
                "k8s:org=empire"
            ],
            "pod_name": "tiefighter"
        },
        "Type": "L7",
        "node_name": "cilium",
        "l7": {
            "type": "RESPONSE",
            "latency_ns": "10246020",
            "http": {
                "code": 200,
                "method": "POST",
                "url": "http: //deathstar.default.svc.cluster.local/v1/request-landing",
                "protocol": "HTTP/1.1",
                "headers": [
                    {
                        "key": "Content-Length",
                        "value": "12"
                    },
                    {
                        "key": "Content-Type",
                        "value": "text/plain"
                    },
                    {
                        "key": "Date",
                        "value": "Thu, 22 Sep 2022 07:15:13 GMT"
                    },
                    {
                        "key": "X-Envoy-Upstream-Service-Time",
                        "value": "6"
                    },
                    {
                        "key": "X-Request-Id",
                        "value": "58d8fc20-a7d3-40f3-bf2d-c853078e5972"
                    }
                ]
            }
        },
        "reply": true,
        "event_type": {
            "type": 129
        },
        "traffic_direction": "INGRESS",
        "is_reply": true,
        "Summary": "HTTP/1.1 200 10ms (POST http://deathstar.default.svc.cluster.local/v1/request-landing)"
    },
    "node_name": "cilium",
    "time": "2022-09-22T07:15:13.011142862Z"
}
```