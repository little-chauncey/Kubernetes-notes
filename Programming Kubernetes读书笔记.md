## Edge-Driven VS Level-Driven

在异步I/O机制里，有两种IO中断触发机制：edge-triggered和level-triggered。edge-triggered表面意思是“边缘触发”，实际是状态变化时触发中断，以后若状态一直未发生改变，将不再通知应用；level-triggered是指“状态”触发，表达的是在某种状态下触发，如果一直在这种状态那么就一直触发。

在k8s里面，对于状态变化有两个驱动原则：

- Edge-Driven：在某一时刻有资源实例的状态发生变化，比如Pod从Pending到Running，那么就触发一个事件，比如说Pod controller的informer收到事件
- Level-Driven：通过周期性的状态检查，以触发事件来完成当前状态到目标状态的调和

Level-Driven是一种轮训机制，在大容量的对象实例存在时，它的性能是存在问题的；因为它取决于`周期`和`apiserver的响应时长`。

Edge-Driven在大容量场景下，是一种更有效的方式。它的时延主要取决于worker的数量。所以K8s大量采取了这种模式。

但是在分布式系统中，同时运行着a number of actors，到处都可能产生event(Edge-Driven模式)，这些event的顺序也无法得到保证。如果我们任何一个controller有bug，或者任何一个状态机发生了问题，或者外部服务发生问题（比如FST对接FS发放虚机or网络资源，但FS服务不在位），那么很有可能会丢失一个event。所以这个时候，仍然需要Level-Driven这种模式。

## 乐观锁Optimistic Concurrency

K8s apiserver有一个很关键的设计原则就是乐观锁。什么是乐观锁？简而言之就是建立在数据一般情况下不会冲突的假设上，对于对象实例的更新，同时可以有多个client去更新，但是真正进行提交更新的时候，才会正式检测冲突。如果冲突，则会给用户返回异常。

K8s对于乐观锁的实现，其实就是每个资源实例都会有自己的ResourceVersion。所以K8s一般在更新实例代码，会这样写：

```go
	foo, err := client.Get("foo", metav1.GetOptions{})
	if err != nil {
		return
	}
	
	<update-the-world-and-foo>

	_, err := client.Update(foo)
	if err != nil && errors.IsConflict(err) {
		continue
	}
```

