---
title: k8s 每日一问 - deployment 不允许 spec.selector 发生变化的逻辑？
date: '2022-02-17 16:56:00'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---


在 BeforeUpdate 行为里面做 validate

```go

// ValidateUpdate is the default update validation for an end user.
func (deploymentStrategy) ValidateUpdate(ctx context.Context, obj, old runtime.Object) field.ErrorList {
	newDeployment := obj.(*apps.Deployment)
	oldDeployment := old.(*apps.Deployment)

	opts := pod.GetValidationOptionsFromPodTemplate(&newDeployment.Spec.Template, &oldDeployment.Spec.Template)
	allErrs := validation.ValidateDeploymentUpdate(newDeployment, oldDeployment, opts)

	// Update is not allowed to set Spec.Selector for all groups/versions except extensions/v1beta1.
	// If RequestInfo is nil, it is better to revert to old behavior (i.e. allow update to set Spec.Selector)
	// to prevent unintentionally breaking users who may rely on the old behavior.
	// TODO(#50791): after apps/v1beta1 and extensions/v1beta1 are removed,
	// move selector immutability check inside ValidateDeploymentUpdate().
	if requestInfo, found := genericapirequest.RequestInfoFrom(ctx); found {
		groupVersion := schema.GroupVersion{Group: requestInfo.APIGroup, Version: requestInfo.APIVersion}
		switch groupVersion {
		case appsv1beta1.SchemeGroupVersion, extensionsv1beta1.SchemeGroupVersion:
			// no-op for compatibility
		default:
			// 禁止选择器发生变化
			allErrs = append(allErrs, apivalidation.ValidateImmutableField(newDeployment.Spec.Selector, oldDeployment.Spec.Selector, field.NewPath("spec").Child("selector"))...)
		}
	}

	return allErrs
}
```