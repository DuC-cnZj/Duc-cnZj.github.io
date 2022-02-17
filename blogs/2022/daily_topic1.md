---
title: k8s 每日一问 - 计算Pod所有容器的内存使用量和Pod请求的内存量的差值， 差值越小，越容易被驱逐?集群资源不够时驱逐流程? 
date: '2022-02-17 16:53:40'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---

### 谁来判断并且执行驱逐操作？

kubelet, evictionManager

```go
// evictionManager -> Start -> synchronize(定期执行) -> signalToRankFunc 中找到对应类型的方法拿出排序后的驱逐的pods列表, 看 buildSignalToRankFunc()
func buildSignalToRankFunc(withImageFs bool) map[evictionapi.Signal]rankFunc {
	signalToRankFunc := map[evictionapi.Signal]rankFunc{
		evictionapi.SignalMemoryAvailable:            rankMemoryPressure,
		evictionapi.SignalAllocatableMemoryAvailable: rankMemoryPressure,
		evictionapi.SignalPIDAvailable:               rankPIDPressure,
	}
//	...
}

// 驱逐种类，cpu不在里面，所以cpu不够时不会驱逐？？？
const (
	SignalMemoryAvailable Signal = "memory.available"
	SignalNodeFsAvailable Signal = "nodefs.available"
	SignalNodeFsInodesFree Signal = "nodefs.inodesFree"
	SignalImageFsAvailable Signal = "imagefs.available"
	SignalImageFsInodesFree Signal = "imagefs.inodesFree"
	SignalAllocatableMemoryAvailable Signal = "allocatableMemory.available"
	SignalPIDAvailable Signal = "pid.available"
)

// 内存驱逐
func rankMemoryPressure(pods []*v1.Pod, stats statsFunc) {
	orderedBy(exceedMemoryRequests(stats), priority, memory(stats)).Sort(pods)
}

// pid 驱逐
func rankPIDPressure(pods []*v1.Pod, stats statsFunc) {
	orderedBy(priority, process(stats)).Sort(pods)
}

// 磁盘驱逐
func rankDiskPressureFunc(fsStatsToMeasure []fsStatsType, diskResource v1.ResourceName) rankFunc {
	return func(pods []*v1.Pod, stats statsFunc) {
		orderedBy(exceedDiskRequests(stats, fsStatsToMeasure, diskResource), priority, disk(stats, fsStatsToMeasure, diskResource)).Sort(pods)
	}
}


```

### 流程

```go
// evictionManager 一个for循环定期检查资源情况
// evictionMonitoringPeriod = time.Second * 10 写死的，10 秒检查一次
go func() {
   for {
      if evictedPods := m.synchronize(diskInfoProvider, podFunc); evictedPods != nil {
         klog.InfoS("Eviction manager: pods evicted, waiting for pod to be cleaned up", "pods", klog.KObjs(evictedPods))
         m.waitForPodsCleanup(podCleanedUpFunc, evictedPods)
      } else {
         time.Sleep(monitoringInterval)
      }
   }
}()
```

### 差值越小，越容易被驱逐?✅

确实是这样，按内存举例子

```go
// kublet 按照这个算法排序去获取驱逐pod列表
// exceedMemoryRequests(stats) 所有pod按照使用内存从大到小排序，大的先死
// priority pod priorityClass 优先级低的先死亡
// memory pod best-effort > burstable > guaranteed，差值越小，先死（见下面：TestOrderedByMemory）
func rankMemoryPressure(pods []*v1.Pod, stats statsFunc) {
	orderedBy(exceedMemoryRequests(stats), priority, memory(stats)).Sort(pods)
}
```



```go
// 按内存排序 Memory：内存实际占用对比 request 排序
func TestOrderedByMemory(t *testing.T) {
	pod1 := newPod("best-effort-high", defaultPriority, []v1.Container{
		newContainer("best-effort-high", newResourceList("", "", ""), newResourceList("", "", "")),
	}, nil)
	pod2 := newPod("best-effort-low", defaultPriority, []v1.Container{
		newContainer("best-effort-low", newResourceList("", "", ""), newResourceList("", "", "")),
	}, nil)
	pod3 := newPod("burstable-high", defaultPriority, []v1.Container{
		newContainer("burstable-high", newResourceList("", "100Mi", ""), newResourceList("", "1Gi", "")),
	}, nil)
	pod4 := newPod("burstable-low", defaultPriority, []v1.Container{
		newContainer("burstable-low", newResourceList("", "100Mi", ""), newResourceList("", "1Gi", "")),
	}, nil)
	pod5 := newPod("guaranteed-high", defaultPriority, []v1.Container{
		newContainer("guaranteed-high", newResourceList("", "1Gi", ""), newResourceList("", "1Gi", "")),
	}, nil)
	pod6 := newPod("guaranteed-low", defaultPriority, []v1.Container{
		newContainer("guaranteed-low", newResourceList("", "1Gi", ""), newResourceList("", "1Gi", "")),
	}, nil)
	stats := map[*v1.Pod]statsapi.PodStats{
		pod1: newPodMemoryStats(pod1, resource.MustParse("500Mi")), // 500 relative to request
		pod2: newPodMemoryStats(pod2, resource.MustParse("300Mi")), // 300 relative to request
		pod3: newPodMemoryStats(pod3, resource.MustParse("800Mi")), // 700 relative to request
		pod4: newPodMemoryStats(pod4, resource.MustParse("300Mi")), // 200 relative to request
		pod5: newPodMemoryStats(pod5, resource.MustParse("800Mi")), // -200 relative to request
		pod6: newPodMemoryStats(pod6, resource.MustParse("200Mi")), // -800 relative to request
	}
	statsFn := func(pod *v1.Pod) (statsapi.PodStats, bool) {
		result, found := stats[pod]
		return result, found
	}
	pods := []*v1.Pod{pod1, pod2, pod3, pod4, pod5, pod6}
	orderedBy(memory(statsFn)).Sort(pods)
	expected := []*v1.Pod{pod3, pod1, pod2, pod4, pod5, pod6}
	for i := range expected {
		if pods[i] != expected[i] {
			t.Errorf("Expected pod[%d]: %s, but got: %s", i, expected[i].Name, pods[i].Name)
		}
	}
}

// ExceedsRequestMemory
func TestOrderedByExceedsRequestMemory(t *testing.T) {
	below := newPod("below-requests", -1, []v1.Container{
		newContainer("below-requests", newResourceList("", "200Mi", ""), newResourceList("", "", "")),
	}, nil)
	exceeds := newPod("exceeds-requests", 1, []v1.Container{
		newContainer("exceeds-requests", newResourceList("", "100Mi", ""), newResourceList("", "", "")),
	}, nil)
	stats := map[*v1.Pod]statsapi.PodStats{
		below:   newPodMemoryStats(below, resource.MustParse("199Mi")),   // -1 relative to request
		exceeds: newPodMemoryStats(exceeds, resource.MustParse("101Mi")), // 1 relative to request
	}
	statsFn := func(pod *v1.Pod) (statsapi.PodStats, bool) {
		result, found := stats[pod]
		return result, found
	}
	pods := []*v1.Pod{below, exceeds}
	orderedBy(exceedMemoryRequests(statsFn)).Sort(pods)

	expected := []*v1.Pod{exceeds, below}
	for i := range expected {
		if pods[i] != expected[i] {
			t.Errorf("Expected pod: %s, but got: %s", expected[i].Name, pods[i].Name)
		}
	}
}

// Priority
func TestOrderedByPriority(t *testing.T) {
	low := newPod("low-priority", -134, []v1.Container{
		newContainer("low-priority", newResourceList("", "", ""), newResourceList("", "", "")),
	}, nil)
	medium := newPod("medium-priority", 1, []v1.Container{
		newContainer("medium-priority", newResourceList("100m", "100Mi", ""), newResourceList("200m", "200Mi", "")),
	}, nil)
	high := newPod("high-priority", 12534, []v1.Container{
		newContainer("high-priority", newResourceList("200m", "200Mi", ""), newResourceList("200m", "200Mi", "")),
	}, nil)

	pods := []*v1.Pod{high, medium, low}
	orderedBy(priority).Sort(pods)

	expected := []*v1.Pod{low, medium, high}
	for i := range expected {
		if pods[i] != expected[i] {
			t.Errorf("Expected pod: %s, but got: %s", expected[i].Name, pods[i].Name)
		}
	}
}
```

### 每次循环只驱逐一个pod

```go
	// we kill at most a single pod during each eviction interval
	for i := range activePods {
		pod := activePods[i]
		gracePeriodOverride := int64(0)
		if !isHardEvictionThreshold(thresholdToReclaim) {
			gracePeriodOverride = m.config.MaxPodGracePeriodSeconds
		}
		message, annotations := evictionMessage(resourceToReclaim, pod, statsFunc)
		if m.evictPod(pod, gracePeriodOverride, message, annotations) {
			metrics.Evictions.WithLabelValues(string(thresholdToReclaim.Signal)).Inc()
			return []*v1.Pod{pod}
		}
	}
```


