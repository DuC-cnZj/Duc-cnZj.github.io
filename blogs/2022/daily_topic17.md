---
title: k8s 每日一问 - k8s 匿名访问
date: '2022-10-19 14:01:51'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---


### 用户伪装(Impersonation)

> https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authentication/#user-impersonation

在使用 `kubectl` 时，可以使用 `--as` 标志来配置 `Impersonate-User` 头部字段值， 使用 `--as-group` 标志配置 `Impersonate-Group` 头部字段值。

```bash
kubectl drain mynode --as=superman --as-group=system:masters
kc auth can-i delete ns --as=default --as-group=system:serviceaccount:default
kc auth can-i  list endpointslices --as=system:kube-proxy
kc auth can-i  watch endpointslices --as=system:kube-proxy
kc auth can-i get endpointslices --as=system:kube-proxy
```

### kubelet port默认端口

```go
package ports

// In this file, we can see all default port of cluster.
// It's also an important documentation for us. So don't remove them easily.
const (
	// ProxyStatusPort is the default port for the proxy metrics server.
	// May be overridden by a flag at startup.
	ProxyStatusPort = 10249
	// KubeletPort is the default port for the kubelet server on each host machine.
	// May be overridden by a flag at startup.
	KubeletPort = 10250
	// KubeletReadOnlyPort exposes basic read-only services from the kubelet.
	// May be overridden by a flag at startup.
	// This is necessary for heapster to collect monitoring stats from the kubelet
	// until heapster can transition to using the SSL endpoint.
	// TODO(roberthbailey): Remove this once we have a better solution for heapster.
	KubeletReadOnlyPort = 10255
	// KubeletHealthzPort exposes a healthz endpoint from the kubelet.
	// May be overridden by a flag at startup.
	KubeletHealthzPort = 10248
	// ProxyHealthzPort is the default port for the proxy healthz server.
	// May be overridden by a flag at startup.
	ProxyHealthzPort = 10256
	// KubeControllerManagerPort is the default port for the controller manager status server.
	// May be overridden by a flag at startup.
	KubeControllerManagerPort = 10257
	// CloudControllerManagerPort is the default port for the cloud controller manager server.
	// This value may be overridden by a flag at startup.
	CloudControllerManagerPort = 10258
)

```

### kubelet 本地 server 提供的接口列表

> 函数如下，深入去代码里看, 挺多的
>
> ```
> /healthz
> /pods
> /metrics
> /metrics/cadvisor
> /metrics/resource
> /metrics/probes
> /stats/
> /logs/
> /debug/pprof/
> /debug/flags/v
> ```

```go
func NewServer(
	host HostInterface,
	resourceAnalyzer stats.ResourceAnalyzer,
	auth AuthInterface,
	tp oteltrace.TracerProvider,
	kubeCfg *kubeletconfiginternal.KubeletConfiguration) Server {

	server := Server{
		host:                 host,
		resourceAnalyzer:     resourceAnalyzer,
		auth:                 auth,
		restfulCont:          &filteringContainer{Container: restful.NewContainer()},
		metricsBuckets:       sets.NewString(),
		metricsMethodBuckets: sets.NewString("OPTIONS", "GET", "HEAD", "POST", "PUT", "DELETE", "TRACE", "CONNECT"),
	}
	if auth != nil {
		server.InstallAuthFilter()
	}
	if utilfeature.DefaultFeatureGate.Enabled(features.KubeletTracing) {
		server.InstallTracingFilter(tp)
	}
	server.InstallDefaultHandlers()
	if kubeCfg != nil && kubeCfg.EnableDebuggingHandlers {
		server.InstallDebuggingHandlers()
		// To maintain backward compatibility serve logs and pprof only when enableDebuggingHandlers is also enabled
		// see https://github.com/kubernetes/kubernetes/pull/87273
		server.InstallSystemLogHandler(kubeCfg.EnableSystemLogHandler)
		server.InstallProfilingHandler(kubeCfg.EnableProfilingHandler, kubeCfg.EnableContentionProfiling)
		server.InstallDebugFlagsHandler(kubeCfg.EnableDebugFlagsHandler)
	} else {
		server.InstallDebuggingDisabledHandlers()
	}
	return server
}
```

### k8s 匿名访问

#### apiserver

 里面的匿名访问,在 1.6 及之后版本中，如果所使用的鉴权模式不是 `AlwaysAllow`，则匿名访问默认是被启用的, **所以现在自己创建的k8s是启用的**！这类请求获得用户名 `system:anonymous` 和对应的用户组 `system:unauthenticated`

> https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authentication/#anonymous-requests

#### kubelet

匿名访问和apiserver一样，主要是鉴权，鉴权有两种方式 AlwaysAllow，Webhook

代码方法名字 `BuildAuthn 认证` `BuildAuthz 鉴权`

#### Webhook

这个方式去 create `SubjectAccessReview` obj 来鉴权

> SubjectAccessReview checks whether or not a user or group can perform an
>  action.

这个种方式可以通过下面的crb来改变权限

```yaml
# 看起来开启匿名好像也没啥问题，只要你不是想下面这样操作。。
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: anonymous-crb
subjects:
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
- kind: User
  name: system:anonymous
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

具体创建描述如下

```

// Authorize makes a REST request to the remote service describing the attempted action as a JSON
// serialized api.authorization.v1beta1.SubjectAccessReview object. An example request body is
// provided below.
//
//	{
//	  "apiVersion": "authorization.k8s.io/v1beta1",
//	  "kind": "SubjectAccessReview",
//	  "spec": {
//	    "resourceAttributes": {
//	      "namespace": "kittensandponies",
//	      "verb": "GET",
//	      "group": "group3",
//	      "resource": "pods"
//	    },
//	    "user": "jane",
//	    "group": [
//	      "group1",
//	      "group2"
//	    ]
//	  }
//	}
//
// The remote service is expected to fill the SubjectAccessReviewStatus field to either allow or
// disallow access. A permissive response would return:
//
//	{
//	  "apiVersion": "authorization.k8s.io/v1beta1",
//	  "kind": "SubjectAccessReview",
//	  "status": {
//	    "allowed": true
//	  }
//	}
//
// To disallow access, the remote service would return:
//
//	{
//	  "apiVersion": "authorization.k8s.io/v1beta1",
//	  "kind": "SubjectAccessReview",
//	  "status": {
//	    "allowed": false,
//	    "reason": "user does not have read access to the namespace"
//	  }
//	}
//
```


