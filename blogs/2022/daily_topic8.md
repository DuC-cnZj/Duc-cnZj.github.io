---
title: k8s 每日一问 - 拉取镜像的latest机制?
date: '2022-03-17 18:21:31'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---

## 拉取镜像的**latest**机制

> 文档也有 https://kubernetes.io/docs/concepts/containers/images/#required-image-pull

来看源码:

创建 pod

post  ->  `/api/v1/namespaces/{namespace}/pods`  ->   `createHandler`  ->  `decoder.Decode()` ->  `Default()` -> `SetObjectDefaults_Pod`  ->  `SetDefaults_Container` 

```go
func createHandler(r rest.NamedCreater, scope *RequestScope, admit admission.Interface, includeName bool) http.HandlerFunc {
	return func(w http.ResponseWriter, req *http.Request) {
		// ...
    // hander decode 请求 
		decoder := scope.Serializer.DecoderToVersion(decodeSerializer, scope.HubGroupVersion)
		trace.Step("About to convert to expected version")
		obj, gvk, err := decoder.Decode(body, &defaultGVK, original)
		// ...
	}
}

```

```go
func (c *codec) Decode(data []byte, defaultGVK *schema.GroupVersionKind, into runtime.Object) (runtime.Object, *schema.GroupVersionKind, error) {
	// ...
	if into != nil {
    // 调用 Default 方法
		if c.defaulter != nil {
			c.defaulter.Default(obj)
		}

    // ...
	}

	// ...
	return out, gvk, strictDecodingErr
}
```

最终调用 default 方法，在没有 `ImagePullPolicy` 参数情况下，根据 tag 设置 ImagePullPolicy

```go
func SetDefaults_Container(obj *v1.Container) {
	if obj.ImagePullPolicy == "" {
		// Ignore error and assume it has been validated elsewhere
		_, tag, _, _ := parsers.ParseImageName(obj.Image)

		// Check image tag
		if tag == "latest" {
			obj.ImagePullPolicy = v1.PullAlways
		} else {
			obj.ImagePullPolicy = v1.PullIfNotPresent
		}
	}
	if obj.TerminationMessagePath == "" {
		obj.TerminationMessagePath = v1.TerminationMessagePathDefault
	}
	if obj.TerminationMessagePolicy == "" {
		obj.TerminationMessagePolicy = v1.TerminationMessageReadFile
	}
}
```



