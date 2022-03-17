---
title: k8s 每日一问 - 健康检查来杀死 pod 的流程
date: '2022-03-17 18:23:44'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---


## **k8s** 健康检查来杀死**pod**的流程

健康检查的端口 `port` 可以是 string 或者 int，string 可以填 名称

>    port	<string> -required-
>      Name or number of the port to access on the container. Number must be in
>      the range 1 to 65535. Name must be an IANA_SVC_NAME.

```go
// 源码找端口的时候会判断，如果 port 是字符串，会去找 conatiner ports 部分，名称相同的 containerPort, 如果是整数字符串，那么会直接调用 strconv.Atoi(param.StrVal)
func extractPort(param intstr.IntOrString, container v1.Container) (int, error) {
	port := -1
	var err error
	switch param.Type {
	case intstr.Int:
		port = param.IntValue()
	case intstr.String:
		if port, err = findPortByName(container, param.StrVal); err != nil {
			// Last ditch effort - maybe it was an int stored as string?
			if port, err = strconv.Atoi(param.StrVal); err != nil {
				return port, err
			}
		}
	default:
		return port, fmt.Errorf("intOrString had no kind: %+v", param)
	}
	if port > 0 && port < 65536 {
		return port, nil
	}
	return port, fmt.Errorf("invalid port number: %v", port)
}
```

`net.JoinHostPort(host, strconv.Itoa(port))` ipv4/ipv6 通用



首先新增pod触发 `HandlePodAdditions`

```go
// HandlePodAdditions is the callback in SyncHandler for pods being added from
// a config source.
func (kl *Kubelet) HandlePodAdditions(pods []*v1.Pod) {
   start := kl.clock.Now()
   sort.Sort(sliceutils.PodsByCreationTime(pods))
   for _, pod := range pods {
     // ...
     // 这里会为新pod创建 startup readiness liveness 三个健康检查器
      kl.probeManager.AddPod(pod)
   }
}
```



每个检查器都会定时执行 `doProbe`

```go
func (w *worker) doProbe() (keepGoing bool) {
	defer func() { recover() }() // Actually eat panics (HandleCrash takes care of logging)
	defer runtime.HandleCrash(func(_ interface{}) { keepGoing = true })

	//...
  // 当检查失败时会设置结果
	w.resultsManager.Set(w.containerID, result, w.pod)

	//...

	return true
}

// 设置结果触发 syncLoopIteration 中的 case, 从而触发 syncPod
//	case update := <-kl.livenessManager.Updates():
//		if update.Result == proberesults.Failure {
//			handleProbeSync(kl, update, handler, "liveness", "unhealthy")
//		}
func (m *manager) Set(id kubecontainer.ContainerID, result Result, pod *v1.Pod) {
	if m.setInternal(id, result) {
		m.updates <- Update{id, result, pod.UID}
	}
}
```



`syncPod` 计算 `computePodActions` 根据健康检查计算出 `ContainersToKill`

```go
func (m *kubeGenericRuntimeManager) computePodActions(pod *v1.Pod, podStatus *kubecontainer.PodStatus) podActions {
	klog.V(5).InfoS("Syncing Pod", "pod", klog.KObj(pod))

  // 省略...

	keepCount := 0
	for idx, container := range pod.Spec.Containers {
		containerStatus := podStatus.FindContainerStatusByName(container.Name)
		// ...

		var message string
		var reason containerKillReason
		restart := shouldRestartOnFailure(pod)
		if _, _, changed := containerChanged(&container, containerStatus); changed {
			message = fmt.Sprintf("Container %s definition changed", container.Name)
			restart = true
		} else if liveness, found := m.livenessManager.Get(containerStatus.ID); found && liveness == proberesults.Failure {
			message = fmt.Sprintf("Container %s failed liveness probe", container.Name)
			reason = reasonLivenessProbe
		} else if startup, found := m.startupManager.Get(containerStatus.ID); found && startup == proberesults.Failure {
			message = fmt.Sprintf("Container %s failed startup probe", container.Name)
			reason = reasonStartupProbe
		} else {
			keepCount++
			continue
		}

		if restart {
			message = fmt.Sprintf("%s, will be restarted", message)
			changes.ContainersToStart = append(changes.ContainersToStart, idx)
		}

		changes.ContainersToKill[containerStatus.ID] = containerToKillInfo{
			name:      containerStatus.Name,
			container: &pod.Spec.Containers[idx],
			message:   message,
			reason:    reason,
		}
		klog.V(2).InfoS("Message for Container of pod", "containerName", container.Name, "containerStatusID", containerStatus.ID, "pod", klog.KObj(pod), "containerMessage", message)
	}

	if keepCount == 0 && len(changes.ContainersToStart) == 0 {
		changes.KillPod = true
	}

	return changes
}
```



最后杀死容器

```go
func (m *kubeGenericRuntimeManager) SyncPod(pod *v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, backOff *flowcontrol.Backoff) (result kubecontainer.PodSyncResult) {
	// ...
	if podContainerChanges.KillPod {
		// ...
	} else {
		// 杀死不健康的容器
		for containerID, containerInfo := range podContainerChanges.ContainersToKill {
			klog.V(3).InfoS("Killing unwanted container for pod", "containerName", containerInfo.name, "containerID", containerID, "pod", klog.KObj(pod))
			killContainerResult := kubecontainer.NewSyncResult(kubecontainer.KillContainer, containerInfo.name)
			result.AddSyncResult(killContainerResult)
			if err := m.killContainer(pod, containerID, containerInfo.name, containerInfo.message, containerInfo.reason, nil); err != nil {
				killContainerResult.Fail(kubecontainer.ErrKillContainer, err.Error())
				klog.ErrorS(err, "killContainer for pod failed", "containerName", containerInfo.name, "containerID", containerID, "pod", klog.KObj(pod))
				return
			}
		}
	}
	// ...
	return
}
```

