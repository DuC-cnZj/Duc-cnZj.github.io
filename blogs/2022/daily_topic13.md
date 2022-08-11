---
title: k8s 每日一问 - ns 为什么 /finalize 的 url 才能改 finalize
date: '2022-08-11 14:11:52'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---

## ns 为什么 /finalize 的 url 才能改 finalize



```go
func NewREST(optsGetter generic.RESTOptionsGetter) (*REST, *StatusREST, *FinalizeREST, error) {
	store := &genericregistry.Store{
		NewFunc:                  func() runtime.Object { return &api.Namespace{} },
		NewListFunc:              func() runtime.Object { return &api.NamespaceList{} },
		PredicateFunc:            namespace.MatchNamespace,
		DefaultQualifiedResource: api.Resource("namespaces"),

		CreateStrategy:      namespace.Strategy,
		UpdateStrategy:      namespace.Strategy,
		DeleteStrategy:      namespace.Strategy,
		ResetFieldsStrategy: namespace.Strategy,
		ReturnDeletedObject: true,

		ShouldDeleteDuringUpdate: ShouldDeleteNamespaceDuringUpdate,

		TableConvertor: printerstorage.TableConvertor{TableGenerator: printers.NewTableGenerator().With(printersinternal.AddHandlers)},
	}
	options := &generic.StoreOptions{RESTOptions: optsGetter, AttrFunc: namespace.GetAttrs}
	if err := store.CompleteWithOptions(options); err != nil {
		return nil, nil, nil, err
	}

	statusStore := *store
	statusStore.UpdateStrategy = namespace.StatusStrategy
	statusStore.ResetFieldsStrategy = namespace.StatusStrategy

	finalizeStore := *store
	finalizeStore.UpdateStrategy = namespace.FinalizeStrategy
	finalizeStore.ResetFieldsStrategy = namespace.FinalizeStrategy

	return &REST{store: store, status: &statusStore}, &StatusREST{store: &statusStore}, &FinalizeREST{store: &finalizeStore}, nil
}
```

// store 把 status spec.Finalizers 还原成old值

```go
func (namespaceStrategy) PrepareForUpdate(ctx context.Context, obj, old runtime.Object) {
	newNamespace := obj.(*api.Namespace)
	oldNamespace := old.(*api.Namespace)
	newNamespace.Spec.Finalizers = oldNamespace.Spec.Finalizers
	newNamespace.Status = oldNamespace.Status
}
```

// finalizeStore 只还原 Status

```
// PrepareForUpdate clears fields that are not allowed to be set by end users on update.
func (namespaceFinalizeStrategy) PrepareForUpdate(ctx context.Context, obj, old runtime.Object) {
	newNamespace := obj.(*api.Namespace)
	oldNamespace := old.(*api.Namespace)
	newNamespace.Status = oldNamespace.Status
}
```


