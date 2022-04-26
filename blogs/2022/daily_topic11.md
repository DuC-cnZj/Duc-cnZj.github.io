---
title: k8s 每日一问 - k8s default sa 的权限根据什么决定的
date: '2022-04-26 11:18:34'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---

## **k8s default sa** 的权限根据什么决定的

首先需要知道 default sa是怎么生成的，不难猜测，肯定是在 controller-manager 中生成，我们找到了 `startServiceAccountController`, 这个控制器专门来监听 sa 的变化，比如 namespace 增加/删除, sa 的删除，它确保了 `serviceAccountsToEnsure` 必须存在， 而 这个值来源于 `DefaultServiceAccountsControllerOptions`, 可以看到 default 就在里面

```go
func DefaultServiceAccountsControllerOptions() ServiceAccountsControllerOptions {
	return ServiceAccountsControllerOptions{
		ServiceAccounts: []v1.ServiceAccount{
			{ObjectMeta: metav1.ObjectMeta{Name: "default"}},
		},
	}
}
```

来看 `syncNamespace`，一句话概括就是它确保了 default sa 的存在

```go
func (c *ServiceAccountsController) syncNamespace(ctx context.Context, key string) error {
	startTime := time.Now()
	defer func() {
		klog.V(4).Infof("Finished syncing namespace %q (%v)", key, time.Since(startTime))
	}()

	ns, err := c.nsLister.Get(key)
	if apierrors.IsNotFound(err) {
		return nil
	}
	if err != nil {
		return err
	}
	if ns.Status.Phase != v1.NamespaceActive {
		// If namespace is not active, we shouldn't try to create anything
		return nil
	}

	createFailures := []error{}
	for _, sa := range c.serviceAccountsToEnsure {
		switch _, err := c.saLister.ServiceAccounts(ns.Name).Get(sa.Name); {
		case err == nil:
			continue
		case apierrors.IsNotFound(err):
		case err != nil:
			return err
		}
		// this is only safe because we never read it and we always write it
		// TODO eliminate this once the fake client can handle creation without NS
		sa.Namespace = ns.Name

		if _, err := c.client.CoreV1().ServiceAccounts(ns.Name).Create(ctx, &sa, metav1.CreateOptions{}); err != nil && !apierrors.IsAlreadyExists(err) {
			// we can safely ignore terminating namespace errors
			if !apierrors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
				createFailures = append(createFailures, err)
			}
		}
	}

	return utilerrors.Flatten(utilerrors.NewAggregate(createFailures))
}
```

然后再看看这个 sa 的默认权限是怎么拿到的

起初我猜想应该也是默认 controller 监听到 sa 的创建之后生成一个 clusterrolebinding 或者 rolebinding。

找了一圈没找到这样的控制器，猜测是不是在 api-sever 认证授权的时候做的。

于是开始找这块逻辑

因为 sa token 是 jwt token，所以我找到了

`func (j *jwtTokenAuthenticator) AuthenticateToken(ctx context.Context, tokenData string) (*authenticator.Response, bool, error) {}` 这个方法，这里面做了认证用户，拿到用户信息的操作，在获取用户信息里面看到这样一段逻辑, 可以看到所有的sa都有这两个组 `system:serviceaccounts`,  `system:serviceaccounts:${namespace}`

```go

type ServiceAccountInfo struct {
	Name, Namespace, UID string
	PodName, PodUID      string
}

func (sa *ServiceAccountInfo) UserInfo() user.Info {
	info := &user.DefaultInfo{
		Name:   MakeUsername(sa.Namespace, sa.Name),
		UID:    sa.UID,
		Groups: MakeGroupNames(sa.Namespace),
	}
	if sa.PodName != "" && sa.PodUID != "" {
		info.Extra = map[string][]string{
			PodNameKey: {sa.PodName},
			PodUIDKey:  {sa.PodUID},
		}
	}
	return info
}

// MakeGroupNames generates service account group names for the given namespace
func MakeGroupNames(namespace string) []string {
	return []string{
		AllServiceAccountsGroup,
		MakeNamespaceGroupName(namespace),
	}
}
func MakeNamespaceGroupName(namespace string) string {
	return ServiceAccountGroupPrefix + namespace
}
```

知道了用户信息，接下来获取用户权限，之前讲过了，权限是遍历所有的 clusterrolebinding, 和 rolebinding 来判断你的操作到底有没有权限，所以接下来我们重点看 权限 crb/rb subject 是 Group 的 role/clusterrole



简单看了下相关的 crb (docker-desktop)

```yaml
# system:service-account-issuer-discovery

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2022-04-16T06:31:04Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:service-account-issuer-discovery
  resourceVersion: "155"
  uid: cfd836b4-ffe8-4a70-ba32-107559b2efe8
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:service-account-issuer-discovery
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2022-04-16T06:31:03Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:service-account-issuer-discovery
  resourceVersion: "109"
  uid: de6c04e6-e186-4547-871a-bb9bbfb9201e
rules:
- nonResourceURLs:
  - /.well-known/openid-configuration
  - /openid/v1/jwks
  verbs:
  - get
```

可以看到默认所有的sa都只能 `get` 访问 `/.well-known/openid-configuration` `/openid/v1/jwks`, 这俩地址是 sso OpenID Connect 协议的固定地址

再来看看 `system:serviceaccounts:${namespace}`, 使用下面这个 crb 就可以让 default namespace 中的 所有 sa 拥有 admin 的权限, 也解释了[文档中](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/#referring-to-subjects)这个的写法 

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: default-namespace-crb
subjects:
- kind: Group
  name: system:serviceaccounts:default 
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin 
  apiGroup: rbac.authorization.k8s.io

```


