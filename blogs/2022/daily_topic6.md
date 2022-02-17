---
title: k8s 每日一问 - sa token 删除后自动创建逻辑, 为什么删了之后会自动创建?
date: '2022-02-17 16:57:36'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---

从 controller-manager 切入

```go

saTokenControllerInitFunc := serviceAccountTokenControllerStarter{rootClientBuilder: rootClientBuilder}.startServiceAccountTokenController


---

// Run runs controller blocks until stopCh is closed
func (e *TokensController) Run(workers int, stopCh <-chan struct{}) {
	// Shut down queues
	defer utilruntime.HandleCrash()
	defer e.syncServiceAccountQueue.ShutDown()
	defer e.syncSecretQueue.ShutDown()

	if !cache.WaitForNamedCacheSync("tokens", stopCh, e.serviceAccountSynced, e.secretSynced) {
		return
	}

	klog.V(5).Infof("Starting workers")
	for i := 0; i < workers; i++ {
		go wait.Until(e.syncServiceAccount, 0, stopCh)
		go wait.Until(e.syncSecret, 0, stopCh)
	}
	<-stopCh
	klog.V(1).Infof("Shutting down")
}
```
当删除 sa token secrets 的时候
1. 先进入  `syncSecret`, 删除 sa secrets 关联的 secrets
```yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2022-01-18T14:05:47Z"
  name: default
  namespace: default
  resourceVersion: "328623"
  uid: 04029b83-c0bb-45db-beb0-c78ee6534715
secrets:
# 先删除这个 变成 []
- name: default-token-vxwl5
```

```go

	secret, err := e.getSecret(secretInfo.namespace, secretInfo.name, secretInfo.uid, false)
	switch {
	case err != nil:
		klog.Error(err)
		retry = true
  // 因为 secret 被删了，进这里面，把 sa 的 secrets 字段更新掉, 移除被删除的那个 secret name
	case secret == nil:
		// If the service account exists
		if sa, saErr := e.getServiceAccount(secretInfo.namespace, secretInfo.saName, secretInfo.saUID, false); saErr == nil && sa != nil {
			// secret no longer exists, so delete references to this secret from the service account
			if err := clientretry.RetryOnConflict(RemoveTokenBackoff, func() error {
				return e.removeSecretReference(secretInfo.namespace, secretInfo.saName, secretInfo.saUID, secretInfo.name)
			}); err != nil {
				klog.Error(err)
			}
		}
	default:

```

然后 第一个 loop 发挥作用 `syncServiceAccount`,

```go

func (e *TokensController) syncServiceAccount() {
	key, quit := e.syncServiceAccountQueue.Get()
	if quit {
		return
	}
	defer e.syncServiceAccountQueue.Done(key)

	retry := false
	defer func() {
		e.retryOrForget(e.syncServiceAccountQueue, key, retry)
	}()

	saInfo, err := parseServiceAccountKey(key)
	if err != nil {
		klog.Error(err)
		return
	}

	sa, err := e.getServiceAccount(saInfo.namespace, saInfo.name, saInfo.uid, false)
	switch {
	case err != nil:
		klog.Error(err)
		retry = true
	case sa == nil:
		// service account no longer exists, so delete related tokens
		klog.V(4).Infof("syncServiceAccount(%s/%s), service account deleted, removing tokens", saInfo.namespace, saInfo.name)
		sa = &v1.ServiceAccount{ObjectMeta: metav1.ObjectMeta{Namespace: saInfo.namespace, Name: saInfo.name, UID: saInfo.uid}}
		retry, err = e.deleteTokens(sa)
		if err != nil {
			klog.Errorf("error deleting serviceaccount tokens for %s/%s: %v", saInfo.namespace, saInfo.name, err)
		}
	default:
		// ensure a token exists and is referenced by this service account
    // 确保 sa 有 secrets
		retry, err = e.ensureReferencedToken(sa)
		if err != nil {
			klog.Errorf("error synchronizing serviceaccount %s/%s: %v", saInfo.namespace, saInfo.name, err)
		}
	}
}
```

1. 因为唯一的 token 被删掉了，所以会重新生成一个
2. 更新sa的secrets列表把新的加进去

```go
func (e *TokensController) ensureReferencedToken(serviceAccount *v1.ServiceAccount) ( /* retry */ bool, error) {
  // 因为唯一的 token 被删掉了，所以会重新生成一个
	if hasToken, err := e.hasReferencedToken(serviceAccount); err != nil {
		// Don't retry cache lookup errors
		return false, err
	} else if hasToken {
		// A service account token already exists, and is referenced, short-circuit
		return false, nil
	}

	// We don't want to update the cache's copy of the service account
	// so add the secret to a freshly retrieved copy of the service account
	serviceAccounts := e.client.CoreV1().ServiceAccounts(serviceAccount.Namespace)
	liveServiceAccount, err := serviceAccounts.Get(context.TODO(), serviceAccount.Name, metav1.GetOptions{})
	if err != nil {
		// Retry if we cannot fetch the live service account (for a NotFound error, either the live lookup or our cache are stale)
		return true, err
	}
	if liveServiceAccount.ResourceVersion != serviceAccount.ResourceVersion {
		// Retry if our liveServiceAccount doesn't match our cache's resourceVersion (either the live lookup or our cache are stale)
		klog.V(4).Infof("liveServiceAccount.ResourceVersion (%s) does not match cache (%s), retrying", liveServiceAccount.ResourceVersion, serviceAccount.ResourceVersion)
		return true, nil
	}

	// Build the secret
	secret := &v1.Secret{
		ObjectMeta: metav1.ObjectMeta{
			Name:      secret.Strategy.GenerateName(fmt.Sprintf("%s-token-", serviceAccount.Name)),
			Namespace: serviceAccount.Namespace,
			Annotations: map[string]string{
				v1.ServiceAccountNameKey: serviceAccount.Name,
				v1.ServiceAccountUIDKey:  string(serviceAccount.UID),
			},
		},
		Type: v1.SecretTypeServiceAccountToken,
		Data: map[string][]byte{},
	}

	// Generate the token
	token, err := e.token.GenerateToken(serviceaccount.LegacyClaims(*serviceAccount, *secret))
	if err != nil {
		// retriable error
		return true, err
	}
	secret.Data[v1.ServiceAccountTokenKey] = []byte(token)
	secret.Data[v1.ServiceAccountNamespaceKey] = []byte(serviceAccount.Namespace)
	if e.rootCA != nil && len(e.rootCA) > 0 {
		secret.Data[v1.ServiceAccountRootCAKey] = e.rootCA
	}

	// Save the secret
	createdToken, err := e.client.CoreV1().Secrets(serviceAccount.Namespace).Create(context.TODO(), secret, metav1.CreateOptions{})
	if err != nil {
		// if the namespace is being terminated, create will fail no matter what
		if apierrors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
			return false, err
		}
		// retriable error
		return true, err
	}
	// Manually add the new token to the cache store.
	// This prevents the service account update (below) triggering another token creation, if the referenced token couldn't be found in the store
	e.updatedSecrets.Mutation(createdToken)

	// Try to add a reference to the newly created token to the service account
	addedReference := false
	err = clientretry.RetryOnConflict(clientretry.DefaultRetry, func() error {
		// refresh liveServiceAccount on every retry
		defer func() { liveServiceAccount = nil }()

		// fetch the live service account if needed, and verify the UID matches and that we still need a token
		if liveServiceAccount == nil {
			liveServiceAccount, err = serviceAccounts.Get(context.TODO(), serviceAccount.Name, metav1.GetOptions{})
			if err != nil {
				return err
			}

			if liveServiceAccount.UID != serviceAccount.UID {
				// If we don't have the same service account, stop trying to add a reference to the token made for the old service account.
				return nil
			}

			if hasToken, err := e.hasReferencedToken(liveServiceAccount); err != nil {
				// Don't retry cache lookup errors
				return nil
			} else if hasToken {
				// A service account token already exists, and is referenced, short-circuit
				return nil
			}
		}

		// Try to add a reference to the token
		liveServiceAccount.Secrets = append(liveServiceAccount.Secrets, v1.ObjectReference{Name: secret.Name})
		if _, err := serviceAccounts.Update(context.TODO(), liveServiceAccount, metav1.UpdateOptions{}); err != nil {
			return err
		}

		addedReference = true
		return nil
	})

	if !addedReference {
		// we weren't able to use the token, try to clean it up.
		klog.V(2).Infof("deleting secret %s/%s because reference couldn't be added (%v)", secret.Namespace, secret.Name, err)
		deleteOpts := metav1.DeleteOptions{Preconditions: &metav1.Preconditions{UID: &createdToken.UID}}
		if err := e.client.CoreV1().Secrets(createdToken.Namespace).Delete(context.TODO(), createdToken.Name, deleteOpts); err != nil {
			klog.Error(err) // if we fail, just log it
		}
	}

	if err != nil {
		if apierrors.IsConflict(err) || apierrors.IsNotFound(err) {
			// if we got a Conflict error, the service account was updated by someone else, and we'll get an update notification later
			// if we got a NotFound error, the service account no longer exists, and we don't need to create a token for it
			return false, nil
		}
		// retry in all other cases
		return true, err
	}

	// success!
	return false, nil
}
```