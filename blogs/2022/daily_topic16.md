---
title: k8s 每日一问 - 添加节点/uncordon 是不是会再次触发 pending pod 的调度
date: '2022-10-19 14:01:51'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---


## **添加节点/uncordon**是不是会再次触发 pending pod 的调度

**会**

**调度流程**

1. 一个 pod 被创建

2. Informer addFunc 调用 addPodToSchedulingQueue 把 pod 添加到 activeQueue, 然后触发 cond.Broadcast() 

3. cond.Wait() -> 弹出这个pod, 调度器处理 pod->scheduleOne

4. 这里假设预选优选调度失败 进入 handleSchedulingFailure 然后把 podInfo 通过 AddUnschedulableIfNotPresent 塞到 unschedulablePods里面

5. 这时候新的节点被**添加**/或者满足 `nodeSchedulingPropertiesChange` 

   - node 从 Unschedulable 变成可调度 `node.Spec.Unschedulable` 原来还有这属性~

   - node Allocatable 属性发生变化

   - node label 发生变化

   - 污点发生变化

   - Status.Conditions发生变化
   
   都会触发 `MoveAllToActiveOrBackoffQueue` 使 `unschedulablePods` 重新触发调度




kube scheduler 源码

```go
informerFactory.Core().V1().Nodes().Informer().AddEventHandler(
  cache.ResourceEventHandlerFuncs{
    AddFunc:    sched.addNodeToCache,
    UpdateFunc: sched.updateNodeInCache,
    DeleteFunc: sched.deleteNodeFromCache,
  },
)
```

```go
func (sched *Scheduler) addNodeToCache(obj interface{}) {
	node, ok := obj.(*v1.Node)
	if !ok {
		klog.ErrorS(nil, "Cannot convert to *v1.Node", "obj", obj)
		return
	}

	nodeInfo := sched.Cache.AddNode(node)
	klog.V(3).InfoS("Add event for node", "node", klog.KObj(node))
	sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.NodeAdd, preCheckForNode(nodeInfo))
}
```

```go
func preCheckForNode(nodeInfo *framework.NodeInfo) queue.PreEnqueueCheck {
// 注意：以下检查不考虑抢占，非常罕见
// 情况（例如，节点调整大小），“pod”可能仍然无法通过检查，但抢占会有所帮助。 我们故意
// 选择忽略这些情况，因为无法调度的 pod 最终将重新排队。
	return func(pod *v1.Pod) bool {
		admissionResults := AdmissionCheck(pod, nodeInfo, false)
		if len(admissionResults) != 0 {
			return false
		}
		_, isUntolerated := corev1helpers.FindMatchingUntoleratedTaint(nodeInfo.Node().Spec.Taints, pod.Spec.Tolerations, func(t *v1.Taint) bool {
			return t.Effect == v1.TaintEffectNoSchedule
		})
		return !isUntolerated
	}
}
```

```go
func (sched *Scheduler) updateNodeInCache(oldObj, newObj interface{}) {
	oldNode, ok := oldObj.(*v1.Node)
	if !ok {
		klog.ErrorS(nil, "Cannot convert oldObj to *v1.Node", "oldObj", oldObj)
		return
	}
	newNode, ok := newObj.(*v1.Node)
	if !ok {
		klog.ErrorS(nil, "Cannot convert newObj to *v1.Node", "newObj", newObj)
		return
	}

	nodeInfo := sched.Cache.UpdateNode(oldNode, newNode)
	// Only requeue unschedulable pods if the node became more schedulable.
	if event := nodeSchedulingPropertiesChange(newNode, oldNode); event != nil {
		sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(*event, preCheckForNode(nodeInfo))
	}
}
```

```go
func (sched *Scheduler) handleSchedulingFailure(ctx context.Context, fwk framework.Framework, podInfo *framework.QueuedPodInfo, err error, reason string, nominatingInfo *framework.NominatingInfo) {
	pod := podInfo.Pod
	var errMsg string
	if err != nil {
		errMsg = err.Error()
	}
	if err == ErrNoNodesAvailable {
		klog.V(2).InfoS("Unable to schedule pod; no nodes are registered to the cluster; waiting", "pod", klog.KObj(pod))
	} else if fitError, ok := err.(*framework.FitError); ok {
		// Inject UnschedulablePlugins to PodInfo, which will be used later for moving Pods between queues efficiently.
		podInfo.UnschedulablePlugins = fitError.Diagnosis.UnschedulablePlugins
		klog.V(2).InfoS("Unable to schedule pod; no fit; waiting", "pod", klog.KObj(pod), "err", errMsg)
	} else if apierrors.IsNotFound(err) {
		klog.V(2).InfoS("Unable to schedule pod, possibly due to node not found; waiting", "pod", klog.KObj(pod), "err", errMsg)
		if errStatus, ok := err.(apierrors.APIStatus); ok && errStatus.Status().Details.Kind == "node" {
			nodeName := errStatus.Status().Details.Name
			// when node is not found, We do not remove the node right away. Trying again to get
			// the node and if the node is still not found, then remove it from the scheduler cache.
			_, err := fwk.ClientSet().CoreV1().Nodes().Get(context.TODO(), nodeName, metav1.GetOptions{})
			if err != nil && apierrors.IsNotFound(err) {
				node := v1.Node{ObjectMeta: metav1.ObjectMeta{Name: nodeName}}
				if err := sched.Cache.RemoveNode(&node); err != nil {
					klog.V(4).InfoS("Node is not found; failed to remove it from the cache", "node", node.Name)
				}
			}
		}
	} else {
		klog.ErrorS(err, "Error scheduling pod; retrying", "pod", klog.KObj(pod))
	}

	// Check if the Pod exists in informer cache.
	podLister := fwk.SharedInformerFactory().Core().V1().Pods().Lister()
	cachedPod, e := podLister.Pods(pod.Namespace).Get(pod.Name)
	if e != nil {
		klog.InfoS("Pod doesn't exist in informer cache", "pod", klog.KObj(pod), "err", e)
	} else {
		// In the case of extender, the pod may have been bound successfully, but timed out returning its response to the scheduler.
		// It could result in the live version to carry .spec.nodeName, and that's inconsistent with the internal-queued version.
		if len(cachedPod.Spec.NodeName) != 0 {
			klog.InfoS("Pod has been assigned to node. Abort adding it back to queue.", "pod", klog.KObj(pod), "node", cachedPod.Spec.NodeName)
		} else {
			// As <cachedPod> is from SharedInformer, we need to do a DeepCopy() here.
			// ignore this err since apiserver doesn't properly validate affinity terms
			// and we can't fix the validation for backwards compatibility.
			podInfo.PodInfo, _ = framework.NewPodInfo(cachedPod.DeepCopy())
			if err := sched.SchedulingQueue.AddUnschedulableIfNotPresent(podInfo, sched.SchedulingQueue.SchedulingCycle()); err != nil {
				klog.ErrorS(err, "Error occurred")
			}
		}
	}

	// Update the scheduling queue with the nominated pod information. Without
	// this, there would be a race condition between the next scheduling cycle
	// and the time the scheduler receives a Pod Update for the nominated pod.
	// Here we check for nil only for tests.
	if sched.SchedulingQueue != nil {
		sched.SchedulingQueue.AddNominatedPod(podInfo.PodInfo, nominatingInfo)
	}

	if err == nil {
		// Only tests can reach here.
		return
	}

	msg := truncateMessage(errMsg)
	fwk.EventRecorder().Eventf(pod, nil, v1.EventTypeWarning, "FailedScheduling", "Scheduling", msg)
	if err := updatePod(ctx, sched.client, pod, &v1.PodCondition{
		Type:    v1.PodScheduled,
		Status:  v1.ConditionFalse,
		Reason:  reason,
		Message: errMsg,
	}, nominatingInfo); err != nil {
		klog.ErrorS(err, "Error updating pod", "pod", klog.KObj(pod))
	}
}
```

```go
func nodeSchedulingPropertiesChange(newNode *v1.Node, oldNode *v1.Node) *framework.ClusterEvent {
	if nodeSpecUnschedulableChanged(newNode, oldNode) {
		return &queue.NodeSpecUnschedulableChange
	}
	if nodeAllocatableChanged(newNode, oldNode) {
		return &queue.NodeAllocatableChange
	}
	if nodeLabelsChanged(newNode, oldNode) {
		return &queue.NodeLabelChange
	}
	if nodeTaintsChanged(newNode, oldNode) {
		return &queue.NodeTaintChange
	}
	if nodeConditionsChanged(newNode, oldNode) {
		return &queue.NodeConditionChange
	}

	return nil
}
```

```go
func (p *PriorityQueue) MoveAllToActiveOrBackoffQueue(event framework.ClusterEvent, preCheck PreEnqueueCheck) {
	p.lock.Lock()
	defer p.lock.Unlock()
	unschedulablePods := make([]*framework.QueuedPodInfo, 0, len(p.unschedulablePods.podInfoMap))
	for _, pInfo := range p.unschedulablePods.podInfoMap {
		if preCheck == nil || preCheck(pInfo.Pod) {
			unschedulablePods = append(unschedulablePods, pInfo)
		}
	}
	p.movePodsToActiveOrBackoffQueue(unschedulablePods, event)
}

// NOTE: this function assumes lock has been acquired in caller
func (p *PriorityQueue) movePodsToActiveOrBackoffQueue(podInfoList []*framework.QueuedPodInfo, event framework.ClusterEvent) {
	activated := false
	for _, pInfo := range podInfoList {
		// If the event doesn't help making the Pod schedulable, continue.
		// Note: we don't run the check if pInfo.UnschedulablePlugins is nil, which denotes
		// either there is some abnormal error, or scheduling the pod failed by plugins other than PreFilter, Filter and Permit.
		// In that case, it's desired to move it anyways.
		if len(pInfo.UnschedulablePlugins) != 0 && !p.podMatchesEvent(pInfo, event) {
			continue
		}
		pod := pInfo.Pod
		if p.isPodBackingoff(pInfo) {
			if err := p.podBackoffQ.Add(pInfo); err != nil {
				klog.ErrorS(err, "Error adding pod to the backoff queue", "pod", klog.KObj(pod))
			} else {
				klog.V(5).InfoS("Pod moved to an internal scheduling queue", "pod", klog.KObj(pInfo.Pod), "event", event, "queue", backoffQName)
				metrics.SchedulerQueueIncomingPods.WithLabelValues("backoff", event.Label).Inc()
				p.unschedulablePods.delete(pod)
			}
		} else {
			if err := p.activeQ.Add(pInfo); err != nil {
				klog.ErrorS(err, "Error adding pod to the scheduling queue", "pod", klog.KObj(pod))
			} else {
				klog.V(5).InfoS("Pod moved to an internal scheduling queue", "pod", klog.KObj(pInfo.Pod), "event", event, "queue", activeQName)
				activated = true
				metrics.SchedulerQueueIncomingPods.WithLabelValues("active", event.Label).Inc()
				p.unschedulablePods.delete(pod)
			}
		}
	}
	p.moveRequestCycle = p.schedulingCycle
	if activated {
		p.cond.Broadcast()
	}
}
```


