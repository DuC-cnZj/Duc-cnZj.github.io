---
title: k8s 每日一问 - k8s 没有基于 cpu 的驱逐？那么预留值有什么用，难道只是为了 request 的计算
date: '2022-02-25 10:57:44'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---

看起来是的，文档本身也没有讲到基于cpu的驱逐，只讲了内存和存储。

> https://kubernetes.io/zh/docs/tasks/administer-cluster/reserve-compute-resources/#system-reserved，cpu只是提了一句"则 Pod 加起来不能使用超过 14.5 CPUs 的资源"而已,节点硬驱逐里面也没有对cpu的驱逐条件，https://kubernetes.io/zh/docs/concepts/scheduling-eviction/node-pressure-eviction/，只有硬驱逐条件
>
> 硬驱逐条件没有宽限期。当达到硬驱逐条件时， kubelet 会立即杀死 pod，而不会正常终止以回收紧缺的资源。



```go
// 节点驱逐的 signal 代码里面也没有 cpu, 只有 memory pid storage, nodefs, imagefs
func buildSignalToRankFunc(withImageFs bool) map[evictionapi.Signal]rankFunc {
	signalToRankFunc := map[evictionapi.Signal]rankFunc{
		evictionapi.SignalMemoryAvailable:            rankMemoryPressure,
		evictionapi.SignalAllocatableMemoryAvailable: rankMemoryPressure,
		evictionapi.SignalPIDAvailable:               rankPIDPressure,
	}
	// usage of an imagefs is optional
	if withImageFs {
		// with an imagefs, nodefs pod rank func for eviction only includes logs and local volumes
		signalToRankFunc[evictionapi.SignalNodeFsAvailable] = rankDiskPressureFunc([]fsStatsType{fsStatsLogs, fsStatsLocalVolumeSource}, v1.ResourceEphemeralStorage)
		signalToRankFunc[evictionapi.SignalNodeFsInodesFree] = rankDiskPressureFunc([]fsStatsType{fsStatsLogs, fsStatsLocalVolumeSource}, resourceInodes)
		// with an imagefs, imagefs pod rank func for eviction only includes rootfs
		signalToRankFunc[evictionapi.SignalImageFsAvailable] = rankDiskPressureFunc([]fsStatsType{fsStatsRoot}, v1.ResourceEphemeralStorage)
		signalToRankFunc[evictionapi.SignalImageFsInodesFree] = rankDiskPressureFunc([]fsStatsType{fsStatsRoot}, resourceInodes)
	} else {
		// without an imagefs, nodefs pod rank func for eviction looks at all fs stats.
		// since imagefs and nodefs share a common device, they share common ranking functions.
		signalToRankFunc[evictionapi.SignalNodeFsAvailable] = rankDiskPressureFunc([]fsStatsType{fsStatsRoot, fsStatsLogs, fsStatsLocalVolumeSource}, v1.ResourceEphemeralStorage)
		signalToRankFunc[evictionapi.SignalNodeFsInodesFree] = rankDiskPressureFunc([]fsStatsType{fsStatsRoot, fsStatsLogs, fsStatsLocalVolumeSource}, resourceInodes)
		signalToRankFunc[evictionapi.SignalImageFsAvailable] = rankDiskPressureFunc([]fsStatsType{fsStatsRoot, fsStatsLogs, fsStatsLocalVolumeSource}, v1.ResourceEphemeralStorage)
		signalToRankFunc[evictionapi.SignalImageFsInodesFree] = rankDiskPressureFunc([]fsStatsType{fsStatsRoot, fsStatsLogs, fsStatsLocalVolumeSource}, resourceInodes)
	}
	return signalToRankFunc
}
```

你可以使用 eviction-hard 标志来配置一组硬驱逐条件， 例如 memory.available<1Gi。

kubelet 具有以下默认硬驱逐条件：(同样没提到 cpu, 大概cpu 的限制完全交给 cgroups 来做了？)

memory.available<100Mi

nodefs.available<10%

imagefs.available<15%

nodefs.inodesFree<5%（Linux 节点）

```go
// kubelet_node_status.go
func (kl *Kubelet) defaultNodeStatusFuncs() []func(*v1.Node) error {
// ...
	var setters []func(n *v1.Node) error
	setters = append(setters,
// 这里调用 GetNodeAllocatableReservation 获取预留值
		nodestatus.MachineInfo(string(kl.nodeName), kl.maxPods, kl.podsPerCore, kl.GetCachedMachineInfo, kl.containerManager.GetCapacity,
			kl.containerManager.GetDevicePluginResourceCapacity, kl.containerManager.GetNodeAllocatableReservation, kl.recordEvent),
// ...
	)

	return setters
}

//GetNodeAllocatableReservation 返回调度中必须在此节点上保留的计算或存储资源量。
func (cm *containerManagerImpl) GetNodeAllocatableReservation() v1.ResourceList {
	evictionReservation := hardEvictionReservation(cm.HardEvictionThresholds, cm.capacity)
	result := make(v1.ResourceList)
	for k := range cm.capacity {
		value := resource.NewQuantity(0, resource.DecimalSI)
		if cm.NodeConfig.SystemReserved != nil {
			value.Add(cm.NodeConfig.SystemReserved[k])
		}
		if cm.NodeConfig.KubeReserved != nil {
			value.Add(cm.NodeConfig.KubeReserved[k])
		}
		if evictionReservation != nil {
			value.Add(evictionReservation[k])
		}
		if !value.IsZero() {
			result[k] = *value
		}
	}
	return result
}

func hardEvictionReservation(thresholds []evictionapi.Threshold, capacity v1.ResourceList) v1.ResourceList {
	if len(thresholds) == 0 {
		return nil
	}
	ret := v1.ResourceList{}
	for _, threshold := range thresholds {
		if threshold.Operator != evictionapi.OpLessThan {
			continue
		}
		switch threshold.Signal {
		case evictionapi.SignalMemoryAvailable:
			memoryCapacity := capacity[v1.ResourceMemory]
			value := evictionapi.GetThresholdQuantity(threshold.Value, &memoryCapacity)
			ret[v1.ResourceMemory] = *value
		case evictionapi.SignalNodeFsAvailable:
			storageCapacity := capacity[v1.ResourceEphemeralStorage]
			value := evictionapi.GetThresholdQuantity(threshold.Value, &storageCapacity)
			ret[v1.ResourceEphemeralStorage] = *value
		}
	}
	return ret
}


// 设置节点上的信息 allocatable, capacity
// Capacity = Allocatable + 预留值
func MachineInfo(nodeName string,
	maxPods int,
	podsPerCore int,
	machineInfoFunc func() (*cadvisorapiv1.MachineInfo, error), // typically Kubelet.GetCachedMachineInfo
	capacityFunc func() v1.ResourceList, // typically Kubelet.containerManager.GetCapacity
	devicePluginResourceCapacityFunc func() (v1.ResourceList, v1.ResourceList, []string), // typically Kubelet.containerManager.GetDevicePluginResourceCapacity
	nodeAllocatableReservationFunc func() v1.ResourceList, // typically Kubelet.containerManager.GetNodeAllocatableReservation
	recordEventFunc func(eventType, event, message string), // typically Kubelet.recordEvent
) Setter {
	return func(node *v1.Node) error {
		// Note: avoid blindly overwriting the capacity in case opaque
		//       resources are being advertised.
		if node.Status.Capacity == nil {
			node.Status.Capacity = v1.ResourceList{}
		}

		var devicePluginAllocatable v1.ResourceList
		var devicePluginCapacity v1.ResourceList
		var removedDevicePlugins []string

		// TODO: Post NotReady if we cannot get MachineInfo from cAdvisor. This needs to start
		// cAdvisor locally, e.g. for test-cmd.sh, and in integration test.
		info, err := machineInfoFunc()
		if err != nil {
			// TODO(roberthbailey): This is required for test-cmd.sh to pass.
			// See if the test should be updated instead.
			node.Status.Capacity[v1.ResourceCPU] = *resource.NewMilliQuantity(0, resource.DecimalSI)
			node.Status.Capacity[v1.ResourceMemory] = resource.MustParse("0Gi")
			node.Status.Capacity[v1.ResourcePods] = *resource.NewQuantity(int64(maxPods), resource.DecimalSI)
			klog.ErrorS(err, "Error getting machine info")
		} else {
			node.Status.NodeInfo.MachineID = info.MachineID
			node.Status.NodeInfo.SystemUUID = info.SystemUUID

			for rName, rCap := range cadvisor.CapacityFromMachineInfo(info) {
				node.Status.Capacity[rName] = rCap
			}

			if podsPerCore > 0 {
				node.Status.Capacity[v1.ResourcePods] = *resource.NewQuantity(
					int64(math.Min(float64(info.NumCores*podsPerCore), float64(maxPods))), resource.DecimalSI)
			} else {
				node.Status.Capacity[v1.ResourcePods] = *resource.NewQuantity(
					int64(maxPods), resource.DecimalSI)
			}

			if node.Status.NodeInfo.BootID != "" &&
				node.Status.NodeInfo.BootID != info.BootID {
				// TODO: This requires a transaction, either both node status is updated
				// and event is recorded or neither should happen, see issue #6055.
				recordEventFunc(v1.EventTypeWarning, events.NodeRebooted,
					fmt.Sprintf("Node %s has been rebooted, boot id: %s", nodeName, info.BootID))
			}
			node.Status.NodeInfo.BootID = info.BootID

			if utilfeature.DefaultFeatureGate.Enabled(features.LocalStorageCapacityIsolation) {
				// TODO: all the node resources should use ContainerManager.GetCapacity instead of deriving the
				// capacity for every node status request
				initialCapacity := capacityFunc()
				if initialCapacity != nil {
					if v, exists := initialCapacity[v1.ResourceEphemeralStorage]; exists {
						node.Status.Capacity[v1.ResourceEphemeralStorage] = v
					}
				}
			}

			devicePluginCapacity, devicePluginAllocatable, removedDevicePlugins = devicePluginResourceCapacityFunc()
			for k, v := range devicePluginCapacity {
				if old, ok := node.Status.Capacity[k]; !ok || old.Value() != v.Value() {
					klog.V(2).InfoS("Updated capacity for device plugin", "plugin", k, "capacity", v.Value())
				}
				node.Status.Capacity[k] = v
			}

			for _, removedResource := range removedDevicePlugins {
				klog.V(2).InfoS("Set capacity for removed resource to 0 on device removal", "device", removedResource)
				// Set the capacity of the removed resource to 0 instead of
				// removing the resource from the node status. This is to indicate
				// that the resource is managed by device plugin and had been
				// registered before.
				//
				// This is required to differentiate the device plugin managed
				// resources and the cluster-level resources, which are absent in
				// node status.
				node.Status.Capacity[v1.ResourceName(removedResource)] = *resource.NewQuantity(int64(0), resource.DecimalSI)
			}
		}

		// Set Allocatable.
		if node.Status.Allocatable == nil {
			node.Status.Allocatable = make(v1.ResourceList)
		}
		// Remove extended resources from allocatable that are no longer
		// present in capacity.
		for k := range node.Status.Allocatable {
			_, found := node.Status.Capacity[k]
			if !found && v1helper.IsExtendedResourceName(k) {
				delete(node.Status.Allocatable, k)
			}
		}
		allocatableReservation := nodeAllocatableReservationFunc()
		for k, v := range node.Status.Capacity {
			value := v.DeepCopy()
			if res, exists := allocatableReservation[k]; exists {
				value.Sub(res)
			}
			if value.Sign() < 0 {
				// Negative Allocatable resources don't make sense.
				value.Set(0)
			}
			node.Status.Allocatable[k] = value
		}

		for k, v := range devicePluginAllocatable {
			if old, ok := node.Status.Allocatable[k]; !ok || old.Value() != v.Value() {
				klog.V(2).InfoS("Updated allocatable", "device", k, "allocatable", v.Value())
			}
			node.Status.Allocatable[k] = v
		}
		// for every huge page reservation, we need to remove it from allocatable memory
		for k, v := range node.Status.Capacity {
			if v1helper.IsHugePageResourceName(k) {
				allocatableMemory := node.Status.Allocatable[v1.ResourceMemory]
				value := v.DeepCopy()
				allocatableMemory.Sub(value)
				if allocatableMemory.Sign() < 0 {
					// Negative Allocatable resources don't make sense.
					allocatableMemory.Set(0)
				}
				node.Status.Allocatable[v1.ResourceMemory] = allocatableMemory
			}
		}
		return nil
	}
}
```