## Collector

`Collector`是一个接口，用来标识任何能够采集prometheus metrics的类型。Collector必须先在`register`注册，然后才可以使用。[client_golang](https://godoc.org/github.com/prometheus/client_golang/prometheus)项目提供的五种metric类型`Gauge`, `Counter`, `Summary`, `Histogram`, `Untyped`都继承了`Collector`接口。`Collector`接口是实现可以同时采集多个metric，这些metrics可以是已经创建好的metrics，也可以是在采集时新建的metrics。

[client_golang](https://godoc.org/github.com/prometheus/client_golang/prometheus)项目已经为我们创建了几个开箱即用的`Collector`——`[NewGoCollector](https://github.com/prometheus/client_golang/blob/master/prometheus/go_collector.go#L64)`，`[NewProcessCollector](https://github.com/prometheus/client_golang/blob/master/prometheus/process_collector.go#L63)`等。

- `[NewGoCollector](https://github.com/prometheus/client_golang/blob/master/prometheus/go_collector.go#L64)`返回能够对外暴露当前进程的golang相关指标的`Collector`，包括内存状态等。内存状态需要调用`runtime.ReadMemStats`方法，这个方法会STW(Stop The World)。此外，如果GC已经STW了，此刻正好调用`NewGoCollector.Collect()`方法，则需要等到GC的STW完成后，才可以继续采集到go进程指标。如果当前进程对性能有较高的要求，应该谨慎使用`NewGoCollector`。
- `[NewProcessCollector](https://github.com/prometheus/client_golang/blob/master/prometheus/process_collector.go#L63)`返回能够采集当前进程的cpu，内存，文件系统等指标的`Collector`，但是该`Collector`只会工作在Windows和类Linux系统上，其他系统则不会采集任何指标。

需要注意的是，**`client_golang`默认会采集这两个`Collector`指标**，如果不需要该指标可以使用自定义`Gatherer`将其排除。

`Collector`对外提供两个方法，分别是`Describe`和`Collect`。

- `Describe`通过管道对外暴露当前`Collector`所采集指标的描述信息。提交到管道的`Desc`必须满足唯一性和一致性的要求。如果当前指标`Desc`(指标的元数据，不可变，用来描述一个metrics)和其它指标`Desc`冲突了，则会panic。另外，可以不必把所有的metrics的`Desc`都发送到`Describe`管道。
- `Collect`用来采集当前`Collector`的所有metrics。`Collect`方法一般在采集指标时由prometheus `registry`调用，`Collect`方法只需将所需要的指标提交到管道即可。

最后，`Collector`接口应该是线程安全的，而且可能会被多协程调用，所有自定义的Collector在实现这两个方法是也应该做到线程安全。

## Gatherer

`Gatherer`是一个接口，它用来标识一个`registry`的收集metrics到`metricsFamily`的行为，它和`Registerer`接口描述的功能类似。`Gatherer`对外提供`Gather()`方法，并在`Gather`方法中调用注册到当前`registry`的`Collector`的`Collect`方法，以采集指标。`Gather()`方法会尝试尽最大努力的采集更多指标，如果某个指标采集出错，则返回值中的`MetricFamily`为nil。

`Gatherer`接口在`client_golang`中有几个继承实现，包括`Registry`和`Gatherers`。

`Registry`将在下一小节具体讨论，而`Gatherers`则是`Gathterer`切片的封装，它一般用来合并多个`Gatherer`实例的结果。

## Registerer

`Registerer`是一个接口，它和`Gatherer`接口一起定义了`Register`的行为。`Registerer`接口描述了`Registry`关于注册和去注册的行为。

结构体`Registry`是prometheus里一个非常重要的概念，它负责注册`Collector`，采集所有`Collector`的指标，然后收集这些指标到`MetricFamily`，为对外暴露指标做准备。需要注意的是`Registry`的零值无效，必须用过`NewRegistry`或者`NewPedanticRegistry`函数创建新的`Registry`实例。
