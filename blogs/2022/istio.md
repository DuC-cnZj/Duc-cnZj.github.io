---
title: istio 学习笔记
date: '2022-04-25 12:57:19'
sidebar: false
categories:
 - 技术
tags:
 - istio
publish: true
---

# istio 笔记

> 基本功能都知道了，但是底层实现原理，架构还没摸清楚

集群中的 pod 通过 svc 访问istio服务时，默认轮训，不会使用 VirtualService，如果要用，必须指定 gateway `mesh`,  **并且如果不是通过 istio 管理的pod内部也无法控制流量**

> https://istio.io/latest/zh/docs/ops/common-problems/network-issues/#route-rules-have-no-effect-on-ingress-gateway-requests

```yaml
  gateways:
  - mesh
```



## todo



- 熔断管理的几个参数不是很懂，回头再看看
- 地域负载均衡先不看，没那么多集群测试
- Sds tls 没怎么看懂，和之前直接用secret证书方式没差别，看文章的意思是sds能够热加载 创建更新的证书，如果不开启sds，secrets更新了gateway感知不到？
- 过一下整片文档
- 一个 gateway 多个证书？
- 多集群

## 注意

1. Istio 支持 k8s 本身的 ingress，只需要把 `ingressClassName` 改成 `istio`

2. 使用 k8s 原生 ingress 的 TLS ， `Ingress` 支持[TLS](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/#tls)设置。 Istio 支持此功能，但是引用的 `Secret` 必须存在于`istio-ingressgateway` 部署的名称空间（通常是 `istio-system` ）中。 [cert-manager](https://istio.io/latest/zh/docs/ops/integrations/certmanager/)可用于生成这些证书。
   - [ ] 待测试
   - [ ] https://istio.io/latest/zh/docs/tasks/traffic-management/ingress/kubernetes-ingress/
   
3. VirtualService 如果需要在其它 pod 内也遵守规则需要在 vs 的 hosts 中添加内部  dns 记录

   1. ```yaml
      apiVersion: networking.istio.io/v1alpha3
      kind: VirtualService
      metadata:
        name: gw
        namespace: test
      spec:
        hosts:
        # 如果其它内部pod要通过 vs 访问demo，必须在这里加 demo，"demo.test/demo.test.svc.cluster.local" 可以不加，进过测试只需要加一个就行
      # https://istio.io/latest/zh/docs/tasks/traffic-management/ingress/ingress-control/  
        - "demo"
        - "demo.istio.local"
      # ...
      ```

4. 只有被 istio 注入 sidecar 的 pod 去 curl 才具备vs规则，普通pod是通过curl 访问 vs 的时候还是 rr。

5. VirtualService HTTP 请求的默认超时时间是 15 秒

6. HTTP 请求的默认重试行为是在返回错误之前重试两次

7. [使用 istio 应用必须满足的条件](https://istio.io/latest/zh/docs/ops/deployment/requirements/)

8. 接上，[service 协议选择协议](https://istio.io/latest/zh/docs/ops/configuration/traffic-management/protocol-selection/)

9. [同一张 https 证书不能配置多个 gateway](https://istio.io/latest/zh/docs/ops/common-problems/network-issues/#not-found-errors-occur-when-multiple-gateways-configured-with-same-TLS-certificate) **已测试确实是这样√**

10. [istio 帮你检查端口名称规范](https://istio.io/latest/zh/docs/ops/diagnostic-tools/istioctl-analyze/)

11. [istio 注解 annotations](https://istio.io/latest/zh/docs/reference/config/annotations/)

11. 修改 istio 自动注入策略貌似只能通过 `kubectl edit cm -n istio-system istio-sidecar-injector`

12. ```yaml
    # 强制跳转到 https
    # https://istio.io/latest/zh/docs/reference/config/networking/gateway/
    tls:
      httpsRedirect: true # sends 301 redirect for http requests
    ```
    
13. ```yaml
    # 控制网格到外部的流量，需要 ServiceEntry + VirtualService 一起工作
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: vs-google
      namespace: test
    spec:
      hosts:
      - "google.com"
      http:
      - timeout: 2s
        route:
        - destination:
            host: google.com
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: ServiceEntry
    metadata:
      name: external-svc-google
      namespace: test
    spec:
      hosts:
      - google.com
      location: MESH_EXTERNAL
      ports:
      - number: 80
        name: example-http
        protocol: HTTP
      resolution: DNS
    ```

14. VirtualService http redirect **在重定向时，使用此值覆盖 URL 的路径部分。请注意，无论请求 URI 是否匹配为确切路径或前缀，都将替换整个路径。**

15. HTTPRewrite rewrite **用这个值重写 URI 的路径（或前缀）部分。如果原始 URI 是根据前缀匹配的，则此字段中提供的值将替换相应匹配的前缀。**

17. [gateway 中的 selector 默认会在所有 namespace 中匹配](https://istio.io/latest/docs/reference/config/networking/gateway/#Gateway)

18. 小坑 mars 的 etag 不能用了，如果用了 istio 多版本，可能需要考虑单独打包前端代码，可能不适应于这种场景

16. 因此，用一句话来说明 `analyze` 就是：尽管运行！它非常安全，不用考虑，可能会有帮助，最坏的情况就是浪费您一分钟！

