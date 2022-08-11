---
title: k8s 每日一问 - 什么是 拓扑感知提示 
date: '2022-08-11 14:17:19'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---

## [拓扑感知提示 TopologyAwareHints](https://kubernetes.io/zh-cn/docs/concepts/services-networking/topology-aware-hints/)

> TopologyAwareHints 实验 https://blog.csdn.net/shida_csdn/article/details/124285905
> 
> 参考了别人的实验，自己也做了，确实有用，但要注意

前提

node 带有以下label会被ignore

1. `node-role.kubernetes.io/control-plane`
2. `node-role.kubernetes.io/master`

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/topology-aware-hints: auto
  name: nginx
  namespace: default
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
  - name: nginx
    protocol: TCP
    port: 80
    targetPort:  http
    nodePort: 31200
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  nginx
  namespace: default
  labels:
    app:  nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app:  nginx
    spec:
      # affinity:
      #   podAntiAffinity:
      #     requiredDuringSchedulingIgnoredDuringExecution:
      #       - topologyKey: topology.kubernetes.io/zone
      containers:
      - name:  nginx
        image:  nginx
        ports:
        - name:  http
          containerPort: 80
          protocol: TCP
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 100m
            memory: 100Mi
        livenessProbe:
          httpGet:
            port: http
            path: /
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
          periodSeconds: 10
        readinessProbe:
          httpGet:
            port: http
            path: /
          timeoutSeconds: 2
          successThreshold: 1
          failureThreshold: 3
          periodSeconds: 10
```


