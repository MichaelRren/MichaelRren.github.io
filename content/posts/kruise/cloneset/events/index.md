---
title: "CloneSet 调谐事件源分析"
date: 2024-09-14T23:59:43+08:00
tags: ["CloneSet", "Events"]
categories: ["Kruise"]
---
本文旨通过源码分析，找到触发 CloneSet 调谐的事件源。

CloneSet 的事件源来自三个部分：
1. CloneSet 资源自身的 CRUD 动作；
2. Pod 的变更；
3. PVC 的变更；

下面开始深入源码分析，本文分析的代码基于 Kruise 的 1.7.0 版本。

# CloneSet 资源的 CRUD 事件
调谐函数关注 CloneSet 的 CRUD 事件。其中对 Update 事件增加扩缩容相关的日志输出。
```go
// pkg/controller/cloneset/cloneset_event_handler.go

// add adds a new Controller to mgr with r as the reconcile.Reconciler
func add(mgr manager.Manager, r reconcile.Reconciler) error {
	// Create a new controller
	...

	// Watch for changes to CloneSet
	err = c.Watch(source.Kind(mgr.GetCache(), &appsv1alpha1.CloneSet{}), &handler.EnqueueRequestForObject{}, predicate.Funcs{
		UpdateFunc: func(e event.UpdateEvent) bool {
			oldCS := e.ObjectOld.(*appsv1alpha1.CloneSet)
			newCS := e.ObjectNew.(*appsv1alpha1.CloneSet)
			if *oldCS.Spec.Replicas != *newCS.Spec.Replicas {
				klog.V(4).InfoS("Observed updated replicas for CloneSet",
					"cloneSet", klog.KObj(newCS), "oldReplicas", *oldCS.Spec.Replicas, "newReplicas", *newCS.Spec.Replicas)
			}
			return true
		},
	})
	...
}
```

# Pod 事件监听
Pod 的 CRUD 是 CloneSet 调谐的事件来源之一。由 `podEventHandler`这个实现了`EventHandler`的结构体过滤 Pod 事件。
## Create
1. 获取当前 Pod；
2. 如果 Pod 处于删除状态，调用 [Delete](#delete) 事件处理；
3. 获取当前 Pod 对应的 CloneSet，如果能获取到：
    1. 通过 [resolveControllerRef](#resolvecontrollerref) 构造入队请求；
    2. 构造出的入队请求为空，说明 Group 或者 Kind 与 CloneSet 不匹配，返回；
    3. 检查是否有重复事件，并入队；
4. 否则是个孤儿 Pod：
    1. [获取当前 Pod Namespace 下，匹配 Pod Label 的 CloneSet 列表](#getpodclonesets)；
    2. 遍历列表生成请求（预期有且只有一个匹配的 CloneSet）并入队；

```go
// pkg/controller/cloneset/cloneset_event_handler.go

func (e *podEventHandler) Create(ctx context.Context, evt event.CreateEvent, q workqueue.RateLimitingInterface) {
	pod := evt.Object.(*v1.Pod)
	if pod.DeletionTimestamp != nil {
		...
		e.Delete(ctx, event.DeleteEvent{Object: evt.Object}, q)
		return
	}

	// If it has a ControllerRef, that's all that matters.
	if controllerRef := metav1.GetControllerOf(pod); controllerRef != nil {
		req := resolveControllerRef(pod.Namespace, controllerRef)
		if req == nil {
			return
		}
		...

		isSatisfied, _, _ := clonesetutils.ScaleExpectations.SatisfiedExpectations(req.String())
		clonesetutils.ScaleExpectations.ObserveScale(req.String(), expectations.Create, pod.Name)
		if isSatisfied {
			...
			q.AddAfter(*req, initialingRateLimiter.When(req))
		} else {
			...
			initialingRateLimiter.Forget(req)
			q.Add(*req)
		}
		return
	}

	...
	csList := e.getPodCloneSets(pod)
	if len(csList) == 0 {
		return
	}
	...
	for _, cs := range csList {
		q.Add(reconcile.Request{NamespacedName: types.NamespacedName{
			Name:      cs.GetName(),
			Namespace: cs.GetNamespace(),
		}})
	}
}
```

### Update
1. 获取旧的 Pod；
2. 获取新的 Pod；
3. 比较两个 Pod ResourceVersion，一致则不处理；
4. 比较两个 Pod 的 Label 是否有变化；
5. 比较当前 Pod 是否处于删除阶段，如果处于删除阶段：
    1. [构造删除**新 Pod** 的事件](#delete)；
    2. [如果 label 变化，构造删除**旧 Pod** 的事件](#delete)；
	3. 不再进行后续处理返回；
6. 获取新 Pod 的 CloneSet；
7. 获取旧 Pod 的 CloneSet；
8. 比较两个 Pod 的 CloneSet 是否有变化；
9. 如果变化了并且 old pod 存在对应的 CloneSet：
    1. 通过 [resolveControllerRef](#resolvecontrollerref)，使用 **old pod 的 namespace 和 CloneSet** 构造入队请求；
    2. 构造出的入队请求不为空，则将请求入队；
10. 检查 new pod 的 CloneSet 是否为空：
    1. 通过 [resolveControllerRef](#resolvecontrollerref)，使用 **new pod 的 namespace 和 CloneSet** 构造入队请求；
    2. 请求为空则返回；
    3. 不为空则检查是否开启了`CloneSetEventHandlerOptimization`这个 featureGate，如果开启了：
        1. 新旧 Pod 的 CloneSet 没有变化且新旧 Pod 的 Label 没有变化且是[可以忽略的属性变化](#shouldignoreupdate)，则跳过这次 Update；
    4. 将第 10.1 步构造的请求入队并返回；
11. 上述条件都不满足，则这是一个孤儿 Pod，检查这个孤儿 Pod 的 label 是否变化或者 CloneSet 是否变化：
    1. [获取到匹配的 CloneSet 列表](#getpodclonesets)（预期只有一个）；
    2. 遍历该列表：
        1. 根据遍历到的 CloneSet 构造入队请求；
12. 方法执行结束；

```go
// pkg/controller/cloneset/cloneset_event_handler.go

func (e *podEventHandler) Update(ctx context.Context, evt event.UpdateEvent, q workqueue.RateLimitingInterface) {
	oldPod := evt.ObjectOld.(*v1.Pod)
	curPod := evt.ObjectNew.(*v1.Pod)
	if curPod.ResourceVersion == oldPod.ResourceVersion {
		// Periodic resync will send update events for all known pods.
		// Two different versions of the same pod will always have different RVs.
		return
	}

	labelChanged := !reflect.DeepEqual(curPod.Labels, oldPod.Labels)
	if curPod.DeletionTimestamp != nil {
		// when a pod is deleted gracefully it's deletion timestamp is first modified to reflect a grace period,
		// and after such time has passed, the kubelet actually deletes it from the store. We receive an update
		// for modification of the deletion timestamp and expect an rs to create more replicas asap, not wait
		// until the kubelet actually deletes the pod. This is different from the Phase of a pod changing, because
		// an rs never initiates a phase change, and so is never asleep waiting for the same.
		e.Delete(ctx, event.DeleteEvent{Object: evt.ObjectNew}, q)
		if labelChanged {
			// we don't need to check the oldPod.DeletionTimestamp because DeletionTimestamp cannot be unset.
			e.Delete(ctx, event.DeleteEvent{Object: evt.ObjectOld}, q)
		}
		return
	}

	curControllerRef := metav1.GetControllerOf(curPod)
	oldControllerRef := metav1.GetControllerOf(oldPod)
	controllerRefChanged := !reflect.DeepEqual(curControllerRef, oldControllerRef)
	if controllerRefChanged && oldControllerRef != nil {
		// The ControllerRef was changed. Sync the old controller, if any.
		if req := resolveControllerRef(oldPod.Namespace, oldControllerRef); req != nil {
			q.Add(*req)
		}
	}

	// If it has a ControllerRef, that's all that matters.
	if curControllerRef != nil {
		req := resolveControllerRef(curPod.Namespace, curControllerRef)
		if req == nil {
			return
		}

		if utilfeature.DefaultFeatureGate.Enabled(features.CloneSetEventHandlerOptimization) {
			if !controllerRefChanged && !labelChanged && e.shouldIgnoreUpdate(req, oldPod, curPod) {
				return
			}
		}

		klog.V(4).InfoS("Pod updated", "pod", klog.KObj(curPod), "owner", req)
		q.Add(*req)
		return
	}

	// Otherwise, it's an orphan. If anything changed, sync matching controllers
	// to see if anyone wants to adopt it now.
	if labelChanged || controllerRefChanged {
		csList := e.getPodCloneSets(curPod)
		if len(csList) == 0 {
			return
		}
		klog.V(4).InfoS("Orphan Pod updated", "pod", klog.KObj(curPod), "owner", e.joinCloneSetNames(csList))
		for _, cs := range csList {
			q.Add(reconcile.Request{NamespacedName: types.NamespacedName{
				Name:      cs.GetName(),
				Namespace: cs.GetNamespace(),
			}})
		}
	}
}
```

### shouldIgnoreUpdate

用于在 feature 打开的情况下，检查本次 UpdatePod 事件是否可以不进行调谐处理。

1. 根据 NamespaceName 获取到对应的 CloneSet；
2. 通过调用 [IgnorePodUpdateEvent](#ignorepodupdateevent) 检查是否应该跳过本次更新事件；

```go
// pkg/controller/cloneset/cloneset_event_handler.go

func (e *podEventHandler) shouldIgnoreUpdate(req *reconcile.Request, oldPod, curPod *v1.Pod) bool {
	cs := &appsv1alpha1.CloneSet{}
	if err := e.Get(context.TODO(), req.NamespacedName, cs); err != nil {
		return false
	}

	return clonesetcore.New(cs).IgnorePodUpdateEvent(oldPod, curPod)
}
```

## Delete
1. 获取当前 Pod；
2. 获取当前 Pod 对应的 CloneSet；
3. 通过 [resolveControllerRef](#resolvecontrollerref) 构造入队请求；

```go
// pkg/controller/cloneset/cloneset_event_handler.go

func (e *podEventHandler) Delete(ctx context.Context, evt event.DeleteEvent, q workqueue.RateLimitingInterface) {
	pod, ok := evt.Object.(*v1.Pod)
	if !ok {
		klog.ErrorS(nil, "Skipped pod deletion event", "deleteStateUnknown", evt.DeleteStateUnknown, "obj", evt.Object)
		return
	}
	clonesetutils.ResourceVersionExpectations.Delete(pod)

	controllerRef := metav1.GetControllerOf(pod)
	if controllerRef == nil {
		// No controller should care about orphans being deleted.
		return
	}
	req := resolveControllerRef(pod.Namespace, controllerRef)
	if req == nil {
		return
	}

	klog.V(4).InfoS("Pod deleted", "pod", klog.KObj(pod), "owner", req)
	clonesetutils.ScaleExpectations.ObserveScale(req.String(), expectations.Delete, pod.Name)
	q.Add(*req)
}
```

# PVC 事件监听
PVC 的 CRUD 是 CloneSet 调谐的事件来源之一。由 `pvcEventHandler`这个实现了`EventHandler`的结构体过滤 Pod 事件。
## Create
1. 获取当前 pvc；
2. 如果 pvc 处于删除阶段，直接调用 [Delete](#delete-1) 事件；
3. 获取当前 pvc 的 CloneSet：
    1. 检查请求是否重复，重复则通过 AddAfter 入队；
    2. 否则先重置队列中同名请求，再将当前请求直接入队；

```go
// pkg/controller/cloneset/cloneset_event_handler.go

func (e *pvcEventHandler) Create(ctx context.Context, evt event.CreateEvent, q workqueue.RateLimitingInterface) {
	pvc := evt.Object.(*v1.PersistentVolumeClaim)
	if pvc.DeletionTimestamp != nil {
		e.Delete(ctx, event.DeleteEvent{Object: evt.Object}, q)
		return
	}

	if controllerRef := metav1.GetControllerOf(pvc); controllerRef != nil {
		if req := resolveControllerRef(pvc.Namespace, controllerRef); req != nil {
			isSatisfied, _, _ := clonesetutils.ScaleExpectations.SatisfiedExpectations(req.String())
			clonesetutils.ScaleExpectations.ObserveScale(req.String(), expectations.Create, pvc.Name)
			if isSatisfied {
				// If the scale expectation is satisfied, it should be an existing Pod and the Informer
				// cache should have just synced.
				q.AddAfter(*req, initialingRateLimiter.When(req))
			} else {
				// Otherwise, add it immediately and reset the rate limiter
				initialingRateLimiter.Forget(req)
				q.Add(*req)
			}
		}
	}
}
```

## Update
1. 获取待更新的 pvc；
2. 检查 pvc 是否处于删除阶段，如果是，则直接执行 [Delete](#delete-1) 方法处理；
3. 非 Delete 类型的 Update 请求不会触发调谐；
```go
// pkg/controller/cloneset/cloneset_event_handler.go

func (e *pvcEventHandler) Update(ctx context.Context, evt event.UpdateEvent, q workqueue.RateLimitingInterface) {
	pvc := evt.ObjectNew.(*v1.PersistentVolumeClaim)
	if pvc.DeletionTimestamp != nil {
		e.Delete(ctx, event.DeleteEvent{Object: evt.ObjectNew}, q)
	}
}
```
## Delete
1. 获取将要删除的 pvc；
2. 获取当前 pvc 的 controller 相关 CloneSet；
3. [根据 pvc 的 Namespace 和 controllerRef 生成 CloneSet 相关的入队请求](#resolvecontrollerref)；
4. 将调谐请求入队；
```go
// pkg/controller/cloneset/cloneset_event_handler.go

func (e *pvcEventHandler) Delete(ctx context.Context, evt event.DeleteEvent, q workqueue.RateLimitingInterface) {
	pvc, ok := evt.Object.(*v1.PersistentVolumeClaim)
	if !ok {
		...
		return
	}

	if controllerRef := metav1.GetControllerOf(pvc); controllerRef != nil {
		if req := resolveControllerRef(pvc.Namespace, controllerRef); req != nil {
			clonesetutils.ScaleExpectations.ObserveScale(req.String(), expectations.Delete, pvc.Name)
			q.Add(*req)
		}
	}
}
```
# 通用方法
## resolveControllerRef
1. 解析对应的 GroupVersion；
2. 比较当前 controllerRef 的 Kind 和 CloneSet 的 Kind，以及 controllerRef 的 Group 和 CloneSet 的 Kind 是否一致；
3. 一致则使用 namespace 和 controllerRef 的 Name 生成入队请求；
```go
// pkg/controller/cloneset/cloneset_event_handler.go

func resolveControllerRef(namespace string, controllerRef *metav1.OwnerReference) *reconcile.Request {
	// Parse the Group out of the OwnerReference to compare it to what was parsed out of the requested OwnerType
	refGV, err := schema.ParseGroupVersion(controllerRef.APIVersion)
	if err != nil {
		klog.ErrorS(err, "Could not parse APIVersion in OwnerReference", "ownerRef", controllerRef)
		return nil
	}

	// Compare the OwnerReference Group and Kind against the OwnerType Group and Kind specified by the user.
	// If the two match, create a Request for the objected referred to by
	// the OwnerReference.  Use the Name from the OwnerReference and the Namespace from the
	// object in the event.
	if controllerRef.Kind == clonesetutils.ControllerKind.Kind && refGV.Group == clonesetutils.ControllerKind.Group {
		// Match found - add a Request for the object referred to in the OwnerReference
		req := reconcile.Request{NamespacedName: types.NamespacedName{
			Namespace: namespace,
			Name:      controllerRef.Name,
		}}
		return &req
	}
	return nil
}
```
## getPodCloneSets

1. 根据 Pod Namespace 获取该 Ns 下所有的 CloneSets；
2. 遍历 CloneSets：
    1. 根据 CloneSet 的 Spec 中的 Selector 生成对应的 selector；
    2. 生成 selector 发生错误，或者生成的 selector 为空，或者 selector 和 pod 的 labels 不匹配都会跳过这个 CloneSet；
    3. 否则将这个 CloneSet 加入到 csMatched 列表中；
3. 返回列表；

```go
// pkg/controller/cloneset/cloneset_event_handler.go

func (e *podEventHandler) getPodCloneSets(pod *v1.Pod) []appsv1alpha1.CloneSet {
	csList := appsv1alpha1.CloneSetList{}
	if err := e.List(context.TODO(), &csList, client.InNamespace(pod.Namespace)); err != nil {
		return nil
	}

	var csMatched []appsv1alpha1.CloneSet
	for _, cs := range csList.Items {
		selector, err := metav1.LabelSelectorAsSelector(cs.Spec.Selector)
		if err != nil || selector.Empty() || !selector.Matches(labels.Set(pod.Labels)) {
			continue
		}

		csMatched = append(csMatched, cs)
	}

	if len(csMatched) > 1 {
		// ControllerRef will ensure we don't do anything crazy, but more than one
		// item in this list nevertheless constitutes user error.
		klog.InfoS("Error! More than one CloneSet is selecting pod", "pod", klog.KObj(pod), "cloneSets", e.joinCloneSetNames(csMatched))
	}
	return csMatched
}

```
## core.Control

### IgnorePodUpdateEvent

检查一些关键的字段，比如 Generation，PodReady 状态变化等。

主要是检查是否有 InPlaceUpdateReady 类型的 Readiness Gate，如果存在，则需要进一步检查是否完成了原地升级。完成了原地升级可以跳过此次事件，没有完成原地升级的则无法跳过。

1. 对比新旧 Pod 的 Generation，不一致则返回 false（shouldn’t ignore）；
2. 对比新旧 Pod 的生命周期 Finalizer 是否变化，不一致则返回 false（shouldn’t ignore）；
3. 比较是否 Paused（IsPodUpdatePaused 方法只会返回 false，所以必然一致）；
4. 比较 Pod Ready 是否变化；
5. 检查当前 Pod 是否包含了类型为 InPlaceUpdateReady ReadinessGate：
    1. [获取当前 CloneSet 的基础 Update Options](#getupdateoptions)（表示更新的字段）；
    2. 基于第 5.a 步的结构，设置一些通用的 UpdateOptions：
    3. 通过  [CheckContainersUpdateCompleted](#defaultcheckcontainerinplaceupdatecompleted) 检查当前 Pod 是否处于更新完成原地升级的状态，**报错则代表当前 Pod 没有完成原地升级，那么则跳过本次更新事件**：
        1. 检查当前 Pod 的 Condition，如果为空或者不等于 True，则返回 false（shouldn’t ignore）；
6. 否则返回 true（should ignore）；

```go
// pkg/controller/cloneset/cloneset_event_handler.go

func (c *commonControl) IgnorePodUpdateEvent(oldPod, curPod *v1.Pod) bool {
	if oldPod.Generation != curPod.Generation {
		return false
	}

	if lifecycleFinalizerChanged(c.CloneSet, oldPod, curPod) {
		return false
	}

	if c.IsPodUpdatePaused(oldPod) != c.IsPodUpdatePaused(curPod) {
		return false
	}

	if podutil.IsPodReady(oldPod) != podutil.IsPodReady(curPod) {
		return false
	}

	containsReadinessGate := func(pod *v1.Pod) bool {
		for _, r := range pod.Spec.ReadinessGates {
			if r.ConditionType == appspub.InPlaceUpdateReady {
				return true
			}
		}
		return false
	}

	if containsReadinessGate(curPod) {
		opts := c.GetUpdateOptions()
		opts = inplaceupdate.SetOptionsDefaults(opts)
		if err := containersUpdateCompleted(curPod, opts.CheckContainersUpdateCompleted); err == nil {
			if cond := inplaceupdate.GetCondition(curPod); cond == nil || cond.Status != v1.ConditionTrue {
				return false
			}
		}
	}

	return true
}

func lifecycleFinalizerChanged(cs *appsv1alpha1.CloneSet, oldPod, curPod *v1.Pod) bool {
	if cs.Spec.Lifecycle == nil {
		return false
	}

	if cs.Spec.Lifecycle.PreDelete != nil {
		for _, f := range cs.Spec.Lifecycle.PreDelete.FinalizersHandler {
			if controllerutil.ContainsFinalizer(oldPod, f) != controllerutil.ContainsFinalizer(curPod, f) {
				return true
			}
		}
	}

	if cs.Spec.Lifecycle.InPlaceUpdate != nil {
		for _, f := range cs.Spec.Lifecycle.InPlaceUpdate.FinalizersHandler {
			if controllerutil.ContainsFinalizer(oldPod, f) != controllerutil.ContainsFinalizer(curPod, f) {
				return true
			}
		}
	}

	return false
}

func SetOptionsDefaults(opts *UpdateOptions) *UpdateOptions {
	if opts == nil {
		opts = &UpdateOptions{}
	}

	if opts.CalculateSpec == nil {
		opts.CalculateSpec = defaultCalculateInPlaceUpdateSpec
	}

	if opts.PatchSpecToPod == nil {
		opts.PatchSpecToPod = defaultPatchUpdateSpecToPod
	}

	if opts.CheckPodUpdateCompleted == nil {
		opts.CheckPodUpdateCompleted = DefaultCheckInPlaceUpdateCompleted
	}

	if opts.CheckContainersUpdateCompleted == nil {
		opts.CheckContainersUpdateCompleted = defaultCheckContainersInPlaceUpdateCompleted
	}

	return opts
}

func containersUpdateCompleted(pod *v1.Pod, checkFunc func(pod *v1.Pod, state *appspub.InPlaceUpdateState) error) error {
	if stateStr, ok := appspub.GetInPlaceUpdateState(pod); ok {
		state := appspub.InPlaceUpdateState{}
		if err := json.Unmarshal([]byte(stateStr), &state); err != nil {
			return err
		}
		return checkFunc(pod, &state)
	}
	return fmt.Errorf("pod %v has no in-place update state annotation", klog.KObj(pod))
}

```

### GetUpdateOptions

1. 检查当前 CloneSet 的 UpdateStrategy 中的 InPlaceUpdateStrategy 是否为空，不为空则：
    1. 维护 opts 中的 GracePeriodSeconds；
2. 检查当前 CloneSet 的 UpdateStrategy 中的 Type，如果为 InPlaceOnlyCloneSetUpdateStrategy：
    1. 维护 opts 中的 IgnoreVolumeClaimTempaltesHashDiff；
3. 返回 opts；

```go
// pkg/controller/cloneset/cloneset_event_handler.go

func (c *commonControl) GetUpdateOptions() *inplaceupdate.UpdateOptions {
	opts := &inplaceupdate.UpdateOptions{}
	if c.Spec.UpdateStrategy.InPlaceUpdateStrategy != nil {
		opts.GracePeriodSeconds = c.Spec.UpdateStrategy.InPlaceUpdateStrategy.GracePeriodSeconds
	}
	// For the InPlaceOnly strategy, ignore the hash comparison of VolumeClaimTemplates.
	// Consider making changes through a feature gate.
	if c.Spec.UpdateStrategy.Type == appsv1alpha1.InPlaceOnlyCloneSetUpdateStrategyType {
		opts.IgnoreVolumeClaimTemplatesHashDiff = true
	}
	return opts
}
```

### defaultCheckContainerInPlaceUpdateCompleted

检查 Pod 中容器的原地升级是否完成。错误则表示没有完成原地升级。

1. 通过 Pod 上的 Annotation`apps.kruise.io/runtime-containers-meta`（这个 Annotation 由 kruise-daemon 上报，表示当前 runtime containers 的状态）；
2. 检查入参 inPlaceUpdateState 的 UpdateEnvFromMetadata，如果为 true，则先检查环境变量是否变更：
    1. 如果入参的 runtimeContainerMetaSet 为空，则直接返回错误；
    2. [检查 container 和 kruise-daemon 上报的 container 之间的 hash 值是否一致](#checkallcontainershashconsistent)（比较 env），不一致则返回错误；
3. 如果入参的 runtimeContainerMetaSet 不为空，[检查 container 和 kruise-daemon 上报的 container 之间的 hash 值是否一致](#checkallcontainershashconsistent)（比较 containerSpec，即 container 的所有元信息），一致则返回 nil；
4. 定义一个 containerImages dictionary，用于存储 container 和 image 的映射；
5. 遍历当前 pod 中的所有 container：
    1. 将 Image 写入 containerImages dictionary 中；
6. 遍历 pod status 中的 containerStatuses：
    1. 检查入参 inPlaceUpdateState 中，更新的
        1. 如果两者 imageID 一致，但是容器名又不相同，说明已经感知到更新了但是还没有完成更新，此时返回错误；
        2. 否则如果两者 imageID 不一致，或者 imageID 一致且 image 也一致，那么则删除 inPlaceUpdateState 中这个 container 对应的旧状态；
7. 检查剩余容器的状态，如果还有 inPlaceUpdate 中还有其他容器状态，返回错误；
8. 返回 nil；

```go
// pkg/controller/cloneset/cloneset_event_handler.go

func defaultCheckContainersInPlaceUpdateCompleted(pod *v1.Pod, inPlaceUpdateState *appspub.InPlaceUpdateState) error {
	runtimeContainerMetaSet, err := appspub.GetRuntimeContainerMetaSet(pod)
	if err != nil {
		return err
	}

	if inPlaceUpdateState.UpdateEnvFromMetadata {
		if runtimeContainerMetaSet == nil {
			return fmt.Errorf("waiting for all containers hash consistent, but runtime-container-meta not found")
		}
		if !checkAllContainersHashConsistent(pod, runtimeContainerMetaSet, extractedEnvFromMetadataHash) {
			return fmt.Errorf("waiting for all containers hash consistent")
		}
	}

	if runtimeContainerMetaSet != nil {
		if checkAllContainersHashConsistent(pod, runtimeContainerMetaSet, plainHash) {
			klog.V(5).InfoS("Check Pod in-place update completed for all container hash consistent", "namespace", pod.Namespace, "name", pod.Name)
			return nil
		}
		// If it needs not to update envs from metadata, we don't have to return error here,
		// in case kruise-daemon has broken for some reason and runtime-container-meta is still in an old version.
	}

	containerImages := make(map[string]string, len(pod.Spec.Containers))
	for i := range pod.Spec.Containers {
		c := &pod.Spec.Containers[i]
		containerImages[c.Name] = c.Image
		if len(strings.Split(c.Image, ":")) <= 1 {
			containerImages[c.Name] = fmt.Sprintf("%s:latest", c.Image)
		}
	}

	for _, cs := range pod.Status.ContainerStatuses {
		if oldStatus, ok := inPlaceUpdateState.LastContainerStatuses[cs.Name]; ok {
			// TODO: we assume that users should not update workload template with new image which actually has the same imageID as the old image
			if oldStatus.ImageID == cs.ImageID {
				if containerImages[cs.Name] != cs.Image {
					return fmt.Errorf("container %s imageID not changed", cs.Name)
				}
			}
			delete(inPlaceUpdateState.LastContainerStatuses, cs.Name)
		}
	}

	if len(inPlaceUpdateState.LastContainerStatuses) > 0 {
		return fmt.Errorf("not found statuses of containers %v", inPlaceUpdateState.LastContainerStatuses)
	}

	return nil
}

```

### checkAllContainersHashConsistent

这里声明了对 hash 一致的定义：

- spec.containers 中的容器同样应该存在于 status.containerStatuses 和 runtime-container-meta 中；
- 容器在 containerStatuses 中的 containerID 应该与在 runtime-container-meta 中的一致；
- spec.containers 和 runtime-container-meta 的容器应该有相同的 hash 值；
1. 遍历 Pod 中的 Containers：
2. 遍历 pod 的 ContainerStatus：
    1. 从中找到和当前 containerName 一致的 containerStatus；
    2. 找不到则直接返回 false；
    3. 遍历入参 runtimeContainerMetaSet 的容器列表，找到和当前 containerName 一致的，由 kruise-daemon 上报的状态 containerMeta；
    4. 找不到则直接返回 false；
    5. 找到则先比较 containerStatus 与 containerMeta 中的 ContainerID 是否一致，不一致返回 false；
    6. 检查比较的类型：
        1. plainHash：比较整个 container 的元信息；
        2. extractedEnvFromMetadataHash：比较 containerMeta 的 hash 和当前 container 的 env hash（这里只比较 downward api 中的 env）不一致返回 false；
3. 否则返回 true；

```go
// The requirements for hash consistent:
// 1. all containers in spec.containers should also be in status.containerStatuses and runtime-container-meta
// 2. all containers in status.containerStatuses and runtime-container-meta should have the same containerID
// 3. all containers in spec.containers and runtime-container-meta should have the same hashes
func checkAllContainersHashConsistent(pod *v1.Pod, runtimeContainerMetaSet *appspub.RuntimeContainerMetaSet, hashType hashType) bool {
	for i := range pod.Spec.Containers {
		containerSpec := &pod.Spec.Containers[i]

		var containerStatus *v1.ContainerStatus
		for j := range pod.Status.ContainerStatuses {
			if pod.Status.ContainerStatuses[j].Name == containerSpec.Name {
				containerStatus = &pod.Status.ContainerStatuses[j]
				break
			}
		}
		if containerStatus == nil {
			klog.InfoS("Find no container in status for Pod", "containerName", containerSpec.Name, "namespace", pod.Namespace, "podName", pod.Name)
			return false
		}

		var containerMeta *appspub.RuntimeContainerMeta
		for i := range runtimeContainerMetaSet.Containers {
			if runtimeContainerMetaSet.Containers[i].Name == containerSpec.Name {
				containerMeta = &runtimeContainerMetaSet.Containers[i]
				continue
			}
		}
		if containerMeta == nil {
			klog.InfoS("Find no container in runtime-container-meta for Pod", "containerName", containerSpec.Name, "namespace", pod.Namespace, "podName", pod.Name)
			return false
		}

		if containerMeta.ContainerID != containerStatus.ContainerID {
			klog.InfoS("Find container in runtime-container-meta for Pod has different containerID with status",
				"containerName", containerSpec.Name, "namespace", pod.Namespace, "podName", pod.Name,
				"metaID", containerMeta.ContainerID, "statusID", containerStatus.ContainerID)
			return false
		}

		switch hashType {
		case plainHash:
			if expectedHash := kubeletcontainer.HashContainer(containerSpec); containerMeta.Hashes.PlainHash != expectedHash {
				klog.InfoS("Find container in runtime-container-meta for Pod has different plain hash with spec",
					"containerName", containerSpec.Name, "namespace", pod.Namespace, "podName", pod.Name,
					"metaHash", containerMeta.Hashes.PlainHash, "expectedHash", expectedHash)
				return false
			}
		case extractedEnvFromMetadataHash:
			hasher := utilcontainermeta.NewEnvFromMetadataHasher()
			if expectedHash := hasher.GetExpectHash(containerSpec, pod); containerMeta.Hashes.ExtractedEnvFromMetadataHash != expectedHash {
				klog.InfoS("Find container in runtime-container-meta for Pod has different extractedEnvFromMetadataHash with spec",
					"containerName", containerSpec.Name, "namespace", pod.Namespace, "podName", pod.Name,
					"metaHash", containerMeta.Hashes.ExtractedEnvFromMetadataHash, "expectedHash", expectedHash)
				return false
			}
		}
	}

	return true
}

// HashContainer returns the hash of the container. It is used to compare
// the running container with its desired spec.
// Note: remember to update hashValues in container_hash_test.go as well.
func HashContainer(container *v1.Container) uint64 {
	hash := fnv.New32a()
	// Omit nil or empty field when calculating hash value
	// Please see https://github.com/kubernetes/kubernetes/issues/53644
	containerJSON, _ := json.Marshal(container)
	hashutil.DeepHashObject(hash, containerJSON)
	return uint64(hash.Sum32())
}

func (h *envFromMetadataHasher) GetExpectHash(c *v1.Container, objMeta metav1.Object) uint64 {
	var envs []v1.EnvVar
	for i := range c.Env {
		if c.Env[i].Value != "" || c.Env[i].ValueFrom == nil || c.Env[i].ValueFrom.FieldRef == nil {
			continue
		} else if excludeEnvs.Has(c.Env[i].Name) {
			continue
		}

		// Currently only supports `metadata.labels['<KEY>']`, `metadata.annotations['<KEY>']`
		path, subscript, ok := fieldpath.SplitMaybeSubscriptedPath(c.Env[i].ValueFrom.FieldRef.FieldPath)
		if !ok {
			continue
		}

		env := v1.EnvVar{Name: c.Env[i].Name}
		switch path {
		case "metadata.annotations":
			env.Value = objMeta.GetAnnotations()[subscript]
		case "metadata.labels":
			env.Value = objMeta.GetLabels()[subscript]
		default:
			continue
		}

		envs = append(envs, env)
	}

	sort.SliceStable(envs, func(i, j int) bool {
		return envs[i].Name < envs[j].Name
	})
	return hashEnvs(envs)
}
```





# 引用
[1] [CloneSet 性能优化](https://openkruise.io/zh/docs/user-manuals/cloneset/#性能优化)

[2] [Optimize CloneSet event handler codes](https://github.com/openkruise/kruise/pull/1219/files)