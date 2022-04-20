---
title: grpc istio 负载均衡
date: '2022-04-20 17:47:42'
sidebar: false
categories:
 - 技术
tags:
 - grpc
 - istio
 - 负载均衡
publish: true
---

# grpc istio 负载均衡

因为 grpc 是 http2 导致 k8s 上是无法靠原生服务发现去做负载均衡的，网上有三种解决方式

1. 客户端负载均衡，缺点需要改造客户端，不能适应 hpa 的情况
2. 使用 headless service, 缺点不能适应 hpa 的情况
3. service mesh，缺点需要引入外部组件 istio/linkerd

## 使用 istio 来为 grpc 实现负载均衡的能力

> Istio 1.13

需要添加以下几个文件

### destinationRule.yaml

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: destination-rule-grpc
spec:
  host: grpc-service
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

### virtualService.yaml

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grpc-virtual-service
spec:
  hosts:
    - service-grpc
    - grpc.istio.local
  gateways:
  - mesh # 网格内部访问要走 vs 需要加这个使得所有网格生效，不然只会对 gateway 流量生效，内部会走原生 service，达不到控制流量的效果，https://istio.io/latest/zh/docs/reference/config/networking/virtual-service/
  - grpc-gateway
  http:
  - route:
    - destination:
        host: service-grpc
        subset: v1
        port:
          number: 50000
      weight: 50
    - destination:
        host: service-grpc
        subset: v2
        port:
          number: 50000
      weight: 50
```

### Gateway.yaml

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: grpc-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: grpc
      # protocol 要写 grpc
      protocol: grpc
    hosts:
    - grpc.istio.local

```

### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-grpc
  labels:
    app: books
spec:
  type: ClusterIP
  ports:
    - port: 50000
      targetPort: grpc
      protocol: TCP
      name: grpc
  selector:
    app: books
```


