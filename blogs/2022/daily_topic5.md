---
title: k8s 每日一问 - hpa 数量为 0 的时候不会再触发伸缩？
date: '2022-02-17 16:56:54'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---

```go
func (a *HorizontalController) reconcileAutoscaler(ctx context.Context, hpaShared *autoscalingv2.HorizontalPodAutoscaler, key string) error {
  ...
// 如果当前副本数为0，并且自动伸缩最小值不为0，则不会触发伸缩
if scale.Spec.Replicas == 0 && minReplicas != 0 {
		// Autoscaling is disabled for this resource
		desiredReplicas = 0
		rescale = false
		setCondition(hpa, autoscalingv2.ScalingActive, v1.ConditionFalse, "ScalingDisabled", "scaling is disabled since the replica count of the target is zero")

  ...
}
```