client_golang: github.com/prometheus/client_golang/prometheus

## package prometheus

```import "github.com/prometheus/client_golang/prometheus"```

prometheus包是exporter的核心包，它为度量代码提供了基本的度量指标，还提供了度量指标的注册。包promhttp提供了通过http的方式供prometheus server拉取数据的能力，包push提供了将指标数据推送到Pushgateway的能力，此外包promauto提供了指标的自动注册功能。

Package prometheus is the core instrumentation package. It provides metrics primitives to instrument code for monitoring. It also offers a registry for metrics. Sub-packages allow to expose the registered metrics via HTTP (package promhttp) or push them to a Pushgateway (package push). There is also a sub-package promauto, which provides metrics constructors with automatic registration.

包prometheus中所有暴露的方法和函数都支持并发，除非特别说明。

All exported functions and methods are safe to be used concurrently unless specified otherwise.

## A Basic Example

As a starting point, a very basic usage example:

```go
package main
 
import (
    "log"
    "net/http"
 
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)
 
var (
    cpuTemp = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "cpu_temperature_celsius",
        Help: "Current temperature of the CPU.",
    })
    hdFailures = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "hd_errors_total",
            Help: "Number of hard-disk errors.",
        },
        []string{"device"},
    )
)
 
func init() {
    // Metrics have to be registered to be exposed:
    prometheus.MustRegister(cpuTemp)
    prometheus.MustRegister(hdFailures)
}
 
func main() {
    cpuTemp.Set(65.3)
    hdFailures.With(prometheus.Labels{"device":"/dev/sda"}).Inc()
 
    // The Handler function provides a default handler to expose metrics
    // via an HTTP server. "/metrics" is the usual endpoint for that.
    http.Handle("/metrics", promhttp.Handler())
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

这是一个能够提供两个metrics（一个`counter`和一个`guage`）的exporter代码，其中hdFailures（`guage`）带有一个label，所以结果体现为一个一维矢量。

This is a complete program that exports two metrics, a Gauge and a Counter, the latter with a label attached to turn it into a (one-dimensional) vector.

## Metrics

The number of exported identifiers in this package might appear a bit overwhelming. However, in addition to the basic plumbing shown in the example above, you only need to understand the different metric types and their vector versions for basic usage. Furthermore, if you are not concerned with fine-grained control of when and how to register metrics with the registry, have a look at the promauto package, which will effectively allow you to ignore registration altogether in simple cases.

除了前面例子介绍的两种metric(Counter和Gauge)，还有两种类型：histogram和summary。

Above, you have already touched the Counter and the Gauge. There are two more advanced metric types: the Summary and Histogram. A more thorough description of those four metric types can be found in the Prometheus docs: https://prometheus.io/docs/concepts/metric_types/

此外还有第五种类型的metric，叫做`Untyped`，行为上它和`Gauge`类似，他告诉server端不要假设其类型。

A fifth "type" of metric is Untyped. It behaves like a Gauge, but signals the Prometheus server not to assume anything about its type.

除了基本的五种metrics类型，prometheus的数据模型中一个很重要的部分就是可以把采集样本通过label区分，产生了多维的vector结果。这使得prometheus提供了五种基本类型的矢量metrics，分别是：`GaugeVec`, `CounterVec`, `SummaryVec`, `HistogramVec`, and `UntypedVec`。

In addition to the fundamental metric types Gauge, Counter, Summary, Histogram, and Untyped, a very important part of the Prometheus data model is the partitioning of samples along dimensions called labels, which results in metric vectors. The fundamental types are GaugeVec, CounterVec, SummaryVec, HistogramVec, and UntypedVec.

只有基本的metrics类型（`Counter`, `Gauge`, `Histogram`和`Summary`）实现了Metric接口，基本`metric`和`*Vec`都实现了`Collector`接口，一个`Collector`管理着一些列的metrics。为了方便，metrics也可以自己单独采集指标。需要注意的是`Counter`, `Gauge`, `Histogram`和`Summary`是接口，但是`CounterVec, GaugeVec, HistogramVec`和`SummaryVec`是具体的`struct`。

While only the fundamental metric types implement the Metric interface, both the metrics and their vector versions implement the Collector interface. A Collector manages the collection of a number of Metrics, but for convenience, a Metric can also “collect itself”. Note that Gauge, Counter, Summary, Histogram, and Untyped are interfaces themselves while GaugeVec, CounterVec, SummaryVec, HistogramVec, and UntypedVec are not.

为了创建metric实例或者`Vec`版本，我们需要一个`...Opts`的结构体，如`GaugeOpts1,CounterOpts, SummaryOpts, HistogramOpts, or UntypedOpts`.

To create instances of Metrics and their vector versions, you need a suitable …Opts struct, i.e. GaugeOpts, CounterOpts, SummaryOpts, HistogramOpts, or UntypedOpts.

Custom Collectors and constant Metrics
自定义metric只需要实现`Collector`接口，看起来`Collector`需要和metric可以很方便的绑定在一起。

While you could create your own implementations of Metric, most likely you will only ever implement the Collector interface on your own. At a first glance, a custom Collector seems handy to bundle Metrics for common registration (with the prime example of the different metric vectors above, which bundle all the metrics of the same name but with different labels).

一个更为复杂的使用场景：在prometheus上下文之外已经有了指标，并不希望实现各种metric接口，只想将已经存在的指标值映射到Prometheus指标中去，则自己实现`Collector`接口非常适合这种场景。可以通过`NewConstMetric, NewConstHistogram, NewConstSummary`等，以及它们的`Must...`版本实时创建metric实例。创建过程可以放到`Collector`接口的`Collect`方法中。

There is a more involved use case, too: If you already have metrics available, created outside of the Prometheus context, you don't need the interface of the various Metric types. You essentially want to mirror the existing numbers into Prometheus Metrics during collection. An own implementation of the Collector interface is perfect for that. You can create Metric instances “on the fly” using NewConstMetric, NewConstHistogram, and NewConstSummary (and their respective Must… versions). That will happen in the Collect method. The Describe method has to return separate Desc instances, representative of the “throw-away” metrics to be created later. NewDesc comes in handy to create those Desc instances.

可以参考`processCollector,goCollector`或者`expvarCollector`来学习包`github.com/prometheus/client_golang/prometheus`的使用

The Collector example illustrates the use case. You can also look at the source code of the processCollector (mirroring process metrics), the goCollector (mirroring Go metrics), or the expvarCollector (mirroring expvar metrics) as examples that are used in this package itself.

如果只希望调用函数获取一个感兴趣的指标，可以考虑`GaugeFunc,CounterFunc,UntypedFunc`。

If you just need to call a function to get a single float value to collect as a metric, GaugeFunc, CounterFunc, or UntypedFunc might be interesting shortcuts.

Advanced Uses of the Registry
`MustRegister`是注册`collector`最常用的方式，它可以一次性注册多个`collector`，如果注册出错会抛出panic，但是有些时候我们需要自行处理注册中的错误。如果使用`Register`方法，可以将错误返回以备处理。

While MustRegister is the by far most common way of registering a Collector, sometimes you might want to handle the errors the registration might cause. As suggested by the name, MustRegister panics if an error occurs. With the Register function, the error is returned and can be handled.

如果注册的`collector`和已有的metric不兼容或者不一致，注册函数将会返回错误信息。理想情况下，不一致或者不兼容需要在注册时发现，而不是数据采集期间。不兼容的情况通常会在exporter启动时发现，不一致的情况可能会稍后才被检测到，但可能不是第一次出错就被检测到，这也是为什么需要在注册期间collector和metric需要提供`Describe`或`Desc`方法的原因。

An error is returned if the registered Collector is incompatible or inconsistent with already registered metrics. The registry aims for consistency of the collected metrics according to the Prometheus data model. Inconsistencies are ideally detected at registration time, not at collect time. The former will usually be detected at start-up time of a program, while the latter will only happen at scrape time, possibly not even on the first scrape if the inconsistency only becomes relevant later. That is the main reason why a Collector and a Metric have to describe themselves to the registry.

到目前为止，前面所有的内容都在默认的注册方法中进行,因为注册方法在全局变量`DefaultRegisterer`中。使用`NewRegister`可以新建一个自定义的`register`。或者自己实现`Register`，`Gather`接口。自定义`register`中的`Register，UnRegister`方法和全局的`DefaultRegisterer`中的`Register，UnRegister`使用方法一样。

So far, everything we did operated on the so-called default registry, as it can be found in the global DefaultRegisterer variable. With NewRegistry, you can create a custom registry, or you can even implement the Registerer or Gatherer interfaces yourself. The methods Register and Unregister work in the same way on a custom registry as the global functions Register and Unregister on the default registry.

自定义`register`有多重用途：

- 可以注册特殊属性的metric，参考`NewPedanticRegisterer`
- 可以避免`DefaultRegister`暴露的全局状态
- 可以使用多个`register`同时提供多个metric，以采集不同的数据
- 可以指定测试单独的`register`

There are a number of uses for custom registries: You can use registries with special properties, see NewPedanticRegistry. You can avoid global state, as it is imposed by the DefaultRegisterer. You can use multiple registries at the same time to expose different metrics in different ways. You can use separate registries for testing purposes.

此外，`DefaultRegister`初始化的时候会注册`goCollector`和`processCollector`,使用自定义的`register`，可以避免这个问题。

Also note that the DefaultRegisterer comes registered with a Collector for Go runtime metrics (via NewGoCollector) and a Collector for process metrics (via NewProcessCollector). With a custom registry, you are in control and decide yourself about the Collectors to register.

## HTTP Exposition

`Register`继承了`Gather`接口,其`caller`方法可以暴露采集到的metric。通常，metric通过http的`/metrics`路径对外暴露。对外暴露metrics的工具包为`promhttp`

The Registry implements the Gatherer interface. The caller of the Gather method can then expose the gathered metrics in some way. Usually, the metrics are served via HTTP on the /metrics endpoint. That's happening in the example above. The tools to expose metrics via HTTP are in the promhttp sub-package. (The top-level functions in the prometheus package are deprecated.)

Pushing to the Pushgateway
Function for pushing to the Pushgateway can be found in the push sub-package.

### Graphite Bridge

Functions and examples to push metrics from a Gatherer to Graphite can be found in the graphite sub-package.

### Other Means of Exposition
More ways of exposing metrics can easily be added by following the approaches of the existing implementations.

Constants

```go
const (
    // DefMaxAge is the default duration for which observations stay
    // relevant.
    DefMaxAge time.Duration = 10 * time.Minute
    // DefAgeBuckets is the default number of buckets used to calculate the
    // age of observations.
    DefAgeBuckets = 5
    // DefBufCap is the standard buffer size for collecting Summary observations.
    DefBufCap = 500
)
```

Default values for SummaryOpts.

## Variables

```go
var (
    DefaultRegisterer Registerer = defaultRegistry
    DefaultGatherer   Gatherer   = defaultRegistry
)
```

`DefaultRegister`和`DefaultGather`是接口`Register`和`Gather`的默认实现，初始化时，这两个变量指向同一个`Register`，这个`register`默认注册了一个只工作在Linux上的`processCollector`和一个`goCollector`。注意的是默认`Gather`和`Register`实例带有全局state，想要避免全局状态的话，可以自己实现`Register`和`Gather`，避免使用默认实例。

DefaultRegisterer and DefaultGatherer are the implementations of the Registerer and Gatherer interface a number of convenience functions in this package act on. Initially, both variables point to the same Registry, which has a process collector (currently on Linux only, see NewProcessCollector) and a Go collector (see NewGoCollector, in particular the note about stop-the-world implication with Go versions older than 1.9) already registered. This approach to keep default instances as global state mirrors the approach of other packages in the Go standard library. Note that there are caveats. Change the variables with caution and only if you understand the consequences. Users who want to avoid global state altogether should not use the convenience functions and act on custom instances instead.

```go
var ( DefBuckets = []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10} )
```

`DefBuckets`是默认的`Histogram`的桶。

DefBuckets are the default Histogram buckets. The default buckets are tailored to broadly measure the response time (in seconds) of a network service. Most likely, however, you will be required to define buckets customized to your use case.

```go
var ( DefObjectives = map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001} )
```

`DefObjectives`是`Summary`的默认位数。已过期。

DefObjectives are the default Summary quantile values.

Deprecated: DefObjectives will not be used as the default objectives in v0.10 of the library. The default Summary will have no quantiles then.

## func BuildFQName

```go
func BuildFQName(namespace, subsystem, name string) string
```

函数`BuildFQName`用`_`连接给定的三个组件,为空则忽略。用来生成指标的全名。可以使用`Opts`结构体来实现创建metric中所用到的各种名称，描述等。

BuildFQName joins the given three name components by "_". Empty name components are ignored. If the name parameter itself is empty, an empty string is returned, no matter what. Metric implementations included in this library use this function internally to generate the fully-qualified metric name from the name component in their Opts. Users of the library will only need this function if they implement their own Metric or instantiate a Desc (with NewDesc) directly.

## func ExponentialBuckets
```go
func ExponentialBuckets(start, factor float64, count int) []float64
```

`ExponentialBuckets`函数创建`count`个桶，最小的桶的上边界为`start`，后面的每个桶的上限是`factor`因子乘以前一个桶的上边界。最后的`+Inf`的桶不记在返回值中，返回值可以用于`HistogramOpts`的桶的定义字段。

ExponentialBuckets creates 'count' buckets, where the lowest bucket has an upper bound of 'start' and each following bucket's upper bound is 'factor' times the previous bucket's upper bound. The final +Inf bucket is not counted and not included in the returned slice. The returned slice is meant to be used for the Buckets field of HistogramOpts.

The function panics if 'count' is 0 or negative, if 'start' is 0 or negative, or if 'factor' is less than or equal 1.

## func Handler

```go
func Handler() http.Handler
```

Handler returns an HTTP handler for the DefaultGatherer. It is already instrumented with InstrumentHandler (using "prometheus" as handler name).

Deprecated: Please note the issues described in the doc comment of InstrumentHandler. You might want to consider using promhttp.InstrumentedHandler instead.

## func InstrumentHandler

```go
func InstrumentHandler(handlerName string, handler http.Handler) http.HandlerFunc
```

提供exporter的HTTP handler。它提供了四个指标：`http_requests_total(CounterVec)，http_request_duration_microseconds(Summary)，http_request_size_bytes(Summary)`和`http_response_size_bytes(Summary)`。每个metric都提供了一个"handle"的label，用来标记`handleName`。

InstrumentHandler wraps the given HTTP handler for instrumentation. It registers four metric collectors (if not already done) and reports HTTP metrics to the (newly or already) registered collectors: http_requests_total (CounterVec), http_request_duration_microseconds (Summary), http_request_size_bytes (Summary), http_response_size_bytes (Summary). Each has a constant label named "handler" with the provided handlerName as value. http_requests_total is a metric vector partitioned by HTTP method (label name "method") and HTTP status code (label name "code").

注意，`InstrumentHandler`有一些问题，现在已经丢弃，可以使用`promhttp`替代之。`InstrumentHandler`的问题在于：1）.使用了`summary`，不适合于多实例的数据聚合。2）.使用`mircSecond`作为单位，应当使用秒替代。3）.request的大小在单独的`goroutine`中计算，在请求处理期间对请求头的读写存在竞争。

Deprecated: InstrumentHandler has several issues. Use the tooling provided in package promhttp instead. The issues are the following:

It uses Summaries rather than Histograms. Summaries are not useful if aggregation across multiple instances is required.

It uses microseconds as unit, which is deprecated and should be replaced by seconds.

The size of the request is calculated in a separate goroutine. Since this calculator requires access to the request header, it creates a race with any writes to the header performed during request handling. httputil.ReverseProxy is a prominent example for a handler performing such writes.

It has additional issues with HTTP/2, cf. https://github.com/prometheus/client_golang/issues/272.

### Example

```go
// Handle the "/doc" endpoint with the standard http.FileServer handler.
// By wrapping the handler with InstrumentHandler, request count,
// request and response sizes, and request latency are automatically
// exported to Prometheus, partitioned by HTTP status code and method
// and by the handler name (here "fileserver").
http.Handle("/doc", prometheus.InstrumentHandler(
    "fileserver", http.FileServer(http.Dir("/usr/share/doc")),
))
// The Prometheus handler still has to be registered to handle the
// "/metrics" endpoint. The handler returned by prometheus.Handler() is
// already instrumented - with "prometheus" as the handler name. In this
// example, we want the handler name to be "metrics", so we instrument
// the uninstrumented Prometheus handler ourselves.
http.Handle("/metrics", prometheus.InstrumentHandler(
    "metrics", prometheus.UninstrumentedHandler(),
))
```

## func InstrumentHandlerFunc

```go
func InstrumentHandlerFunc(handlerName string, handlerFunc func(http.ResponseWriter, *http.Request)) 
```

http.HandlerFunc

InstrumentHandlerFunc wraps the given function for instrumentation. It otherwise works in the same way as InstrumentHandler (and shares the same issues).

Deprecated: InstrumentHandlerFunc is deprecated for the same reasons as InstrumentHandler is. Use the tooling provided in package promhttp instead.

## func InstrumentHandlerFuncWithOpts

```go
func InstrumentHandlerFuncWithOpts(opts SummaryOpts, handlerFunc func(http.ResponseWriter, *http.Request)) http.HandlerFunc
```

InstrumentHandlerFuncWithOpts works like InstrumentHandlerFunc (and shares the same issues) but provides more flexibility (at the cost of a more complex call syntax). See InstrumentHandlerWithOpts for details how the provided SummaryOpts are used.

Deprecated: InstrumentHandlerFuncWithOpts is deprecated for the same reasons as InstrumentHandler is. Use the tooling provided in package promhttp instead.

## func InstrumentHandlerWithOpts

```go
func InstrumentHandlerWithOpts(opts SummaryOpts, handler http.Handler) http.HandlerFunc
```

InstrumentHandlerWithOpts works like InstrumentHandler (and shares the same issues) but provides more flexibility (at the cost of a more complex call syntax). As InstrumentHandler, this function registers four metric collectors, but it uses the provided SummaryOpts to create them. However, the fields "Name" and "Help" in the SummaryOpts are ignored. "Name" is replaced by "requests_total", "request_duration_microseconds", "request_size_bytes", and "response_size_bytes", respectively. "Help" is replaced by an appropriate help string. The names of the variable labels of the http_requests_total CounterVec are "method" (get, post, etc.), and "code" (HTTP status code).

If InstrumentHandlerWithOpts is called as follows, it mimics exactly the behavior of InstrumentHandler:

prometheus.InstrumentHandlerWithOpts( prometheus.SummaryOpts{ Subsystem: "http", ConstLabels: prometheus.Labels{"handler": handlerName}, }, handler, )

Technical detail: "requests_total" is a CounterVec, not a SummaryVec, so it cannot use SummaryOpts. Instead, a CounterOpts struct is created internally, and all its fields are set to the equally named fields in the provided SummaryOpts.

Deprecated: InstrumentHandlerWithOpts is deprecated for the same reasons as InstrumentHandler is. Use the tooling provided in package promhttp instead.

## func LinearBuckets

```go
func LinearBuckets(start, width float64, count int) []float64
```

`LinearBuckets`创建`count`个桶，每个桶的宽度为`width`，最小的桶的上边界为`start`，最后的边界为`+Inf`的桶没有统计在返回值的slice内。返回值用于`HistogramOpts`的`Buckets`中。如果`count`为`0`或者负数，本函数会Panic。

LinearBuckets creates 'count' buckets, each 'width' wide, where the lowest bucket has an upper bound of 'start'. The final +Inf bucket is not counted and not included in the returned slice. The returned slice is meant to be used for the Buckets field of HistogramOpts.

The function panics if 'count' is zero or negative.

## func MustRegister

```go
func MustRegister(cs ...Collector)
```

注册给定的`collectors`，如果任意`collector`注册出错，都会Panic。`MustRegister`调用了`DefaultRegisterer.MustRegister(cs...)`，详见`DefaultRegisterer.MustRegister(cs...)`相关章节。

MustRegister registers the provided Collectors with the DefaultRegisterer and panics if any error occurs.

MustRegister is a shortcut for DefaultRegisterer.MustRegister(cs...). See there for more details.

## func Register

```go
func Register(c Collector) error
```

将给定的`collector`注册到`DefaultRegisterer`。详见`DefaultRegisterer.Register(c)`。

Register registers the provided Collector with the DefaultRegisterer.

Register is a shortcut for DefaultRegisterer.Register(c). See there for more details.

### Example

```go
// Imagine you have a worker pool and want to count the tasks completed.
taskCounter := prometheus.NewCounter(prometheus.CounterOpts{
    Subsystem: "worker_pool",
    Name:      "completed_tasks_total",
    Help:      "Total number of tasks completed.",
})
// This will register fine.
if err := prometheus.Register(taskCounter); err != nil {
    fmt.Println(err)
} else {
    fmt.Println("taskCounter registered.")
}
// Don't forget to tell the HTTP server about the Prometheus handler.
// (In a real program, you still need to start the HTTP server...)
http.Handle("/metrics", prometheus.Handler())
 
// Now you can start workers and give every one of them a pointer to
// taskCounter and let it increment it whenever it completes a task.
taskCounter.Inc() // This has to happen somewhere in the worker code.
 
// But wait, you want to see how individual workers perform. So you need
// a vector of counters, with one element for each worker.
taskCounterVec := prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Subsystem: "worker_pool",
        Name:      "completed_tasks_total",
        Help:      "Total number of tasks completed.",
    },
    []string{"worker_id"},
)
 
// Registering will fail because we already have a metric of that name.
if err := prometheus.Register(taskCounterVec); err != nil {
    fmt.Println("taskCounterVec not registered:", err)
} else {
    fmt.Println("taskCounterVec registered.")
}
 
// To fix, first unregister the old taskCounter.
if prometheus.Unregister(taskCounter) {
    fmt.Println("taskCounter unregistered.")
}
 
// Try registering taskCounterVec again.
if err := prometheus.Register(taskCounterVec); err != nil {
    fmt.Println("taskCounterVec not registered:", err)
} else {
    fmt.Println("taskCounterVec registered.")
}
// Bummer! Still doesn't work.
 
// Prometheus will not allow you to ever export metrics with
// inconsistent help strings or label names. After unregistering, the
// unregistered metrics will cease to show up in the /metrics HTTP
// response, but the registry still remembers that those metrics had
// been exported before. For this example, we will now choose a
// different name. (In a real program, you would obviously not export
// the obsolete metric in the first place.)
taskCounterVec = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Subsystem: "worker_pool",
        Name:      "completed_tasks_by_id",
        Help:      "Total number of tasks completed.",
    },
    []string{"worker_id"},
)
if err := prometheus.Register(taskCounterVec); err != nil {
    fmt.Println("taskCounterVec not registered:", err)
} else {
    fmt.Println("taskCounterVec registered.")
}
// Finally it worked!
 
// The workers have to tell taskCounterVec their id to increment the
// right element in the metric vector.
taskCounterVec.WithLabelValues("42").Inc() // Code from worker 42.
 
// Each worker could also keep a reference to their own counter element
// around. Pick the counter at initialization time of the worker.
myCounter := taskCounterVec.WithLabelValues("42") // From worker 42 initialization code.
myCounter.Inc()                                   // Somewhere in the code of that worker.
 
// Note that something like WithLabelValues("42", "spurious arg") would
// panic (because you have provided too many label values). If you want
// to get an error instead, use GetMetricWithLabelValues(...) instead.
notMyCounter, err := taskCounterVec.GetMetricWithLabelValues("42", "spurious arg")
if err != nil {
    fmt.Println("Worker initialization failed:", err)
}
if notMyCounter == nil {
    fmt.Println("notMyCounter is nil.")
}
 
// A different (and somewhat tricky) approach is to use
// ConstLabels. ConstLabels are pairs of label names and label values
// that never change. You might ask what those labels are good for (and
// rightfully so - if they never change, they could as well be part of
// the metric name). There are essentially two use-cases: The first is
// if labels are constant throughout the lifetime of a binary execution,
// but they vary over time or between different instances of a running
// binary. The second is what we have here: Each worker creates and
// registers an own Counter instance where the only difference is in the
// value of the ConstLabels. Those Counters can all be registered
// because the different ConstLabel values guarantee that each worker
// will increment a different Counter metric.
counterOpts := prometheus.CounterOpts{
    Subsystem:   "worker_pool",
    Name:        "completed_tasks",
    Help:        "Total number of tasks completed.",
    ConstLabels: prometheus.Labels{"worker_id": "42"},
}
taskCounterForWorker42 := prometheus.NewCounter(counterOpts)
if err := prometheus.Register(taskCounterForWorker42); err != nil {
    fmt.Println("taskCounterVForWorker42 not registered:", err)
} else {
    fmt.Println("taskCounterForWorker42 registered.")
}
// Obviously, in real code, taskCounterForWorker42 would be a member
// variable of a worker struct, and the "42" would be retrieved with a
// GetId() method or something. The Counter would be created and
// registered in the initialization code of the worker.
 
// For the creation of the next Counter, we can recycle
// counterOpts. Just change the ConstLabels.
counterOpts.ConstLabels = prometheus.Labels{"worker_id": "2001"}
taskCounterForWorker2001 := prometheus.NewCounter(counterOpts)
if err := prometheus.Register(taskCounterForWorker2001); err != nil {
    fmt.Println("taskCounterVForWorker2001 not registered:", err)
} else {
    fmt.Println("taskCounterForWorker2001 registered.")
}
 
taskCounterForWorker2001.Inc()
taskCounterForWorker42.Inc()
taskCounterForWorker2001.Inc()
 
// Yet another approach would be to turn the workers themselves into
// Collectors and register them. See the Collector example for details.
```

output

```
taskCounter registered.
taskCounterVec not registered: a previously registered descriptor with the same fully-qualified name as Desc{fqName: "worker_pool_completed_tasks_total", help: "Total number of tasks completed.", constLabels: {}, variableLabels: [worker_id]} has different label names or a different help string
taskCounter unregistered.
taskCounterVec not registered: a previously registered descriptor with the same fully-qualified name as Desc{fqName: "worker_pool_completed_tasks_total", help: "Total number of tasks completed.", constLabels: {}, variableLabels: [worker_id]} has different label names or a different help string
taskCounterVec registered.
Worker initialization failed: inconsistent label cardinality
notMyCounter is nil.
taskCounterForWorker42 registered.
taskCounterForWorker2001 registered.
```

## func UninstrumentedHandler

```go
func UninstrumentedHandler() http.Handler
```

UninstrumentedHandler returns an HTTP handler for the DefaultGatherer.

Deprecated: Use promhttp.Handler instead. See there for further documentation.

## func Unregister

```go
func Unregister(c Collector) bool
```

Unregister removes the registration of the provided Collector from the DefaultRegisterer.

Unregister is a shortcut for DefaultRegisterer.Unregister(c). See there for more details.

## type AlreadyRegisteredError

```go
type AlreadyRegisteredError struct { ExistingCollector, NewCollector Collector }
```

如果注册的`collector`已经注册过，或者不同的`collector`采集了相同的metric，`Register`函数会返回`AlreadyRegisteredError`错误。返回值`AlreadyRegisteredError`错误包括两个字段：已经注册过的`collector`和被拒绝的`collector`。

AlreadyRegisteredError is returned by the Register method if the Collector to be registered has already been registered before, or a different Collector that collects the same metrics has been registered before. Registration fails in that case, but you can detect from the kind of error what has happened. The error contains fields for the existing Collector and the (rejected) new Collector that equals the existing one. This can be used to find out if an equal Collector has been registered before and switch over to using the old one, as demonstrated in the example.

### Example

```go
reqCounter := prometheus.NewCounter(prometheus.CounterOpts{ 
    Name: "requests_total", 
    Help: "The total number of requests served.", 
}) 
if err := prometheus.Register(reqCounter); err != nil { 
    if are, ok := err.(prometheus.AlreadyRegisteredError); ok { 
        // A counter for that metric has been registered before. 
        // Use the old counter from now on. 
        reqCounter = are.ExistingCollector.(prometheus.Counter) 
    } else { 
        // Something else went wrong! 
        panic(err) 
    } 
} 
reqCounter.Inc()
```

## func (AlreadyRegisteredError) Error

```go
func (err AlreadyRegisteredError) Error() string
```

## type Collector

```go
type Collector interface {
    // Describe sends the super-set of all possible descriptors of metrics
    // collected by this Collector to the provided channel and returns once
    // the last descriptor has been sent. The sent descriptors fulfill the
    // consistency and uniqueness requirements described in the Desc
    // documentation. (It is valid if one and the same Collector sends
    // duplicate descriptors. Those duplicates are simply ignored. However,
    // two different Collectors must not send duplicate descriptors.) This
    // method idempotently sends the same descriptors throughout the
    // lifetime of the Collector. If a Collector encounters an error while
    // executing this method, it must send an invalid descriptor (created
    // with NewInvalidDesc) to signal the error to the registry.
    Describe(chan<- *Desc)
    // Collect is called by the Prometheus registry when collecting
    // metrics. The implementation sends each collected metric via the
    // provided channel and returns once the last metric has been sent. The
    // descriptor of each sent metric is one of those returned by
    // Describe. Returned metrics that share the same descriptor must differ
    // in their variable label values. This method may be called
    // concurrently and must therefore be implemented in a concurrency safe
    // way. Blocking occurs at the expense of total performance of rendering
    // all registered metrics. Ideally, Collector implementations support
    // concurrent readers.
    Collect(chan<- Metric)
}
```

`Collector`是prometheus中可以用于采集metric的接口，`collector`必须先在`register`注册。前面介绍的五种metric类型`Gauge, Counter, Summary, Histogram, Untyped`都继承了`collector`接口。`Collector`接口是实现可以同时采集多个metric。`...Vec，ExpvarCollector，goCollector`等都是`collector`的实现。

Collector is the interface implemented by anything that can be used by Prometheus to collect metrics. A Collector has to be registered for collection. See Registerer.Register.

The stock metrics provided by this package (Gauge, Counter, Summary, Histogram, Untyped) are also Collectors (which only ever collect one metric, namely itself). An implementer of Collector may, however, collect multiple metrics in a coordinated fashion and/or create metrics on the fly. Examples for collectors already implemented in this library are the metric vectors (i.e. collection of multiple instances of the same Metric but with different label values) like GaugeVec or SummaryVec, and the ExpvarCollector.

### Example

```go
package main
 
import "github.com/prometheus/client_golang/prometheus"
 
// ClusterManager is an example for a system that might have been built without
// Prometheus in mind. It models a central manager of jobs running in a
// cluster. To turn it into something that collects Prometheus metrics, we
// simply add the two methods required for the Collector interface.
//
// An additional challenge is that multiple instances of the ClusterManager are
// run within the same binary, each in charge of a different zone. We need to
// make use of ConstLabels to be able to register each ClusterManager instance
// with Prometheus.
type ClusterManager struct {
    Zone         string
    OOMCountDesc *prometheus.Desc
    RAMUsageDesc *prometheus.Desc
    // ... many more fields
}
 
// ReallyExpensiveAssessmentOfTheSystemState is a mock for the data gathering a
// real cluster manager would have to do. Since it may actually be really
// expensive, it must only be called once per collection. This implementation,
// obviously, only returns some made-up data.
func (c *ClusterManager) ReallyExpensiveAssessmentOfTheSystemState() (
    oomCountByHost map[string]int, ramUsageByHost map[string]float64,
) {
    // Just example fake data.
    oomCountByHost = map[string]int{
        "foo.example.org": 42,
        "bar.example.org": 2001,
    }
    ramUsageByHost = map[string]float64{
        "foo.example.org": 6.023e23,
        "bar.example.org": 3.14,
    }
    return
}
 
// Describe simply sends the two Descs in the struct to the channel.
func (c *ClusterManager) Describe(ch chan<- *prometheus.Desc) {
    ch <- c.OOMCountDesc
    ch <- c.RAMUsageDesc
}
 
// Collect first triggers the ReallyExpensiveAssessmentOfTheSystemState. Then it
// creates constant metrics for each host on the fly based on the returned data.
//
// Note that Collect could be called concurrently, so we depend on
// ReallyExpensiveAssessmentOfTheSystemState to be concurrency-safe.
func (c *ClusterManager) Collect(ch chan<- prometheus.Metric) {
    oomCountByHost, ramUsageByHost := c.ReallyExpensiveAssessmentOfTheSystemState()
    for host, oomCount := range oomCountByHost {
        ch <- prometheus.MustNewConstMetric(
            c.OOMCountDesc,
            prometheus.CounterValue,
            float64(oomCount),
            host,
        )
    }
    for host, ramUsage := range ramUsageByHost {
        ch <- prometheus.MustNewConstMetric(
            c.RAMUsageDesc,
            prometheus.GaugeValue,
            ramUsage,
            host,
        )
    }
}
 
// NewClusterManager creates the two Descs OOMCountDesc and RAMUsageDesc. Note
// that the zone is set as a ConstLabel. (It's different in each instance of the
// ClusterManager, but constant over the lifetime of an instance.) Then there is
// a variable label "host", since we want to partition the collected metrics by
// host. Since all Descs created in this way are consistent across instances,
// with a guaranteed distinction by the "zone" label, we can register different
// ClusterManager instances with the same registry.
func NewClusterManager(zone string) *ClusterManager {
    return &ClusterManager{
        Zone: zone,
        OOMCountDesc: prometheus.NewDesc(
            "clustermanager_oom_crashes_total",
            "Number of OOM crashes.",
            []string{"host"},
            prometheus.Labels{"zone": zone},
        ),
        RAMUsageDesc: prometheus.NewDesc(
            "clustermanager_ram_usage_bytes",
            "RAM usage as reported to the cluster manager.",
            []string{"host"},
            prometheus.Labels{"zone": zone},
        ),
    }
}
 
func main() {
    workerDB := NewClusterManager("db")
    workerCA := NewClusterManager("ca")
 
    // Since we are dealing with custom Collector implementations, it might
    // be a good idea to try it out with a pedantic registry.
    reg := prometheus.NewPedanticRegistry()
    reg.MustRegister(workerDB)
    reg.MustRegister(workerCA)
}
```


## func NewExpvarCollector

```go
func NewExpvarCollector(exports map[string]*Desc) Collector
```

函数返回一个新的`expvarCollector`，但这个`collector`仍然需要手动注册到register。`expvarCollector`从`expvar`接口获取metrics数据。它可以快速的将`expvar`接口已经存在的的数据通过prometheus的数据格式暴露出来。由于`expvar`的数据模型和collector完全不同，`expvarCollector`会比本地的prometheus metrics慢。生产系统中不建议使用这个collector。

NewExpvarCollector returns a newly allocated expvar Collector that still has to be registered with a Prometheus registry.

An expvar Collector collects metrics from the expvar interface. It provides a quick way to expose numeric values that are already exported via expvar as Prometheus metrics. Note that the data models of expvar and Prometheus are fundamentally different, and that the expvar Collector is inherently slower than native Prometheus metrics. Thus, the expvar Collector is probably great for experiments and prototying, but you should seriously consider a more direct implementation of Prometheus metrics for monitoring production systems.

The exports map has the following meaning:

The keys in the map correspond to expvar keys, i.e. for every expvar key you want to export as Prometheus metric, you need an entry in the exports map. The descriptor mapped to each key describes how to export the expvar value. It defines the name and the help string of the Prometheus metric proxying the expvar value. The type will always be Untyped.

For descriptors without variable labels, the expvar value must be a number or a bool. The number is then directly exported as the Prometheus sample value. (For a bool, 'false' translates to 0 and 'true' to 1). Expvar values that are not numbers or bools are silently ignored.

If the descriptor has one variable label, the expvar value must be an expvar map. The keys in the expvar map become the various values of the one Prometheus label. The values in the expvar map must be numbers or bools again as above.

For descriptors with more than one variable label, the expvar must be a nested expvar map, i.e. where the values of the topmost map are maps again etc. until a depth is reached that corresponds to the number of labels. The leaves of that structure must be numbers or bools as above to serve as the sample values.

Anything that does not fit into the scheme above is silently ignored.

### Example

```go
expvarCollector := prometheus.NewExpvarCollector(map[string]*prometheus.Desc{
    "memstats": prometheus.NewDesc(
        "expvar_memstats",
        "All numeric memstats as one metric family. Not a good role-model, actually... ;-)",
        []string{"type"}, nil,
    ),
    "lone-int": prometheus.NewDesc(
        "expvar_lone_int",
        "Just an expvar int as an example.",
        nil, nil,
    ),
    "http-request-map": prometheus.NewDesc(
        "expvar_http_request_total",
        "How many http requests processed, partitioned by status code and http method.",
        []string{"code", "method"}, nil,
    ),
})
prometheus.MustRegister(expvarCollector)
 
// The Prometheus part is done here. But to show that this example is
// doing anything, we have to manually export something via expvar.  In
// real-life use-cases, some library would already have exported via
// expvar what we want to re-export as Prometheus metrics.
expvar.NewInt("lone-int").Set(42)
expvarMap := expvar.NewMap("http-request-map")
var (
    expvarMap1, expvarMap2                             expvar.Map
    expvarInt11, expvarInt12, expvarInt21, expvarInt22 expvar.Int
)
expvarMap1.Init()
expvarMap2.Init()
expvarInt11.Set(3)
expvarInt12.Set(13)
expvarInt21.Set(11)
expvarInt22.Set(212)
expvarMap1.Set("POST", &expvarInt11)
expvarMap1.Set("GET", &expvarInt12)
expvarMap2.Set("POST", &expvarInt21)
expvarMap2.Set("GET", &expvarInt22)
expvarMap.Set("404", &expvarMap1)
expvarMap.Set("200", &expvarMap2)
// Results in the following expvar map:
// "http-request-count": {"200": {"POST": 11, "GET": 212}, "404": {"POST": 3, "GET": 13}}
 
// Let's see what the scrape would yield, but exclude the memstats metrics.
metricStrings := []string{}
metric := dto.Metric{}
metricChan := make(chan prometheus.Metric)
go func() {
    expvarCollector.Collect(metricChan)
    close(metricChan)
}()
for m := range metricChan {
    if !strings.Contains(m.Desc().String(), "expvar_memstats") {
        metric.Reset()
        m.Write(&metric)
        metricStrings = append(metricStrings, metric.String())
    }
}
sort.Strings(metricStrings)
for _, s := range metricStrings {
    fmt.Println(strings.TrimRight(s, " "))
}
```

output

```
label:<name:"code" value:"200" > label:<name:"method" value:"GET" > untyped:<value:212 >
label:<name:"code" value:"200" > label:<name:"method" value:"POST" > untyped:<value:11 >
label:<name:"code" value:"404" > label:<name:"method" value:"GET" > untyped:<value:13 >
label:<name:"code" value:"404" > label:<name:"method" value:"POST" > untyped:<value:3 >
untyped:<value:42 >
```

## func NewGoCollector

```go
func NewGoCollector() Collector
```

返回一个能够暴露当前go进程本身metrics的collector，包括go内存使用信息等。`goCollector`通过调用`runtime.ReadMemStat`函数采集内存数据的metrics。在Go1.9+的版本中，这个函数调用会导致约~25µs的程序`stop-the-world`暂停。在Go1.9之前的版本中，这个暂停取决于堆的大小（~1.7 ms/GiB）。

NewGoCollector returns a collector which exports metrics about the current Go process. This includes memory stats. To collect those, runtime.ReadMemStats is called. This causes a stop-the-world, which is very short with Go1.9+ (~25µs). However, with older Go versions, the stop-the-world duration depends on the heap size and can be quite significant (~1.7 ms/GiB as per https://go-review.googlesource.com/c/go/+/34937).

## func NewProcessCollector

```go
func NewProcessCollector(pid int, namespace string) Collector
```

返回一个能够暴露当前go进程本身metrics的collector，包括go内存，CPU，文件系统等使用信息等，也能够采集给定namespace和pid的go进程的开始时间。注意，`ProcessCollector`采集文件系统的状态仅限于Linux系统。

NewProcessCollector returns a collector which exports the current state of process metrics including CPU, memory and file descriptor usage as well as the process start time for the given process ID under the given namespace.

Currently, the collector depends on a Linux-style proc filesystem and therefore only exports metrics for Linux.

## func NewProcessCollectorPIDFn

```go
func NewProcessCollectorPIDFn(
    pidFn func() (int, error),
    namespace string,
) Collector
```

NewProcessCollectorPIDFn works like NewProcessCollector but the process ID is determined on each collect anew by calling the given pidFn function.

## type Counter

```go
type Counter interface {
    Metric
    Collector
 
    // Inc increments the counter by 1. Use Add to increment it by arbitrary
    // non-negative values.
    Inc()
    // Add adds the given value to the counter. It panics if the value is <
    // 0.
    Add(float64)
}
```

Counter is a Metric that represents a single numerical value that only ever goes up. That implies that it cannot be used to count items whose number can also go down, e.g. the number of currently running goroutines. Those "counters" are represented by Gauges.

A Counter is typically used to count requests served, tasks completed, errors occurred, etc.

To create Counter instances, use NewCounter.

### Example

```go
pushCounter := prometheus.NewCounter(prometheus.CounterOpts{
    Name: "repository_pushes", // Note: No help string...
})
err := prometheus.Register(pushCounter) // ... so this will return an error.
if err != nil {
    fmt.Println("Push counter couldn't be registered, no counting will happen:", err)
    return
}
 
// Try it once more, this time with a help string.
pushCounter = prometheus.NewCounter(prometheus.CounterOpts{
    Name: "repository_pushes",
    Help: "Number of pushes to external repository.",
})
err = prometheus.Register(pushCounter)
if err != nil {
    fmt.Println("Push counter couldn't be registered AGAIN, no counting will happen:", err)
    return
}
 
pushComplete := make(chan struct{})
// TODO: Start a goroutine that performs repository pushes and reports
// each completion via the channel.
for range pushComplete {
    pushCounter.Inc()
}
```

output

```
Push counter couldn't be registered, no counting will happen: descriptor Desc{fqName: "repository_pushes", help: "", constLabels: {}, variableLabels: []} is invalid: empty help string
```

## func NewCounter

```go
func NewCounter(opts CounterOpts) Counter
```

NewCounter creates a new Counter based on the provided CounterOpts.

The returned implementation tracks the counter value in two separate variables, a float64 and a uint64. The latter is used to track calls of the Inc method and calls of the Add method with a value that can be represented as a uint64. This allows atomic increments of the counter with optimal performance. (It is common to have an Inc call in very hot execution paths.) Both internal tracking values are added up in the Write method. This has to be taken into account when it comes to precision and overflow behavior.

## type CounterFunc

```go
type CounterFunc interface { Metric Collector }
```

CounterFunc is a Counter whose value is determined at collect time by calling a provided function.

To create CounterFunc instances, use NewCounterFunc.

## func NewCounterFunc

```go
func NewCounterFunc(opts CounterOpts, function func() float64) CounterFunc
```

NewCounterFunc creates a new CounterFunc based on the provided CounterOpts. The value reported is determined by calling the given function from within the Write method. Take into account that metric collection may happen concurrently. If that results in concurrent calls to Write, like in the case where a CounterFunc is directly registered with Prometheus, the provided function must be concurrency-safe. The function should also honor the contract for a Counter (values only go up, not down), but compliance will not be checked.

## type CounterOpts

```go
type CounterOpts Opts
```

CounterOpts is an alias for Opts. See there for doc comments.

## type CounterVec

```go
type CounterVec struct { // contains filtered or unexported fields }
```

CounterVec is a Collector that bundles a set of Counters that all share the same Desc, but have different values for their variable labels. This is used if you want to count the same thing partitioned by various dimensions (e.g. number of HTTP requests, partitioned by response code and method). Create instances with NewCounterVec.

### Example

```go
httpReqs := prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "How many HTTP requests processed, partitioned by status code and HTTP method.",
    },
    []string{"code", "method"},
)
prometheus.MustRegister(httpReqs)
 
httpReqs.WithLabelValues("404", "POST").Add(42)
 
// If you have to access the same set of labels very frequently, it
// might be good to retrieve the metric only once and keep a handle to
// it. But beware of deletion of that metric, see below!
m := httpReqs.WithLabelValues("200", "GET")
for i := 0; i < 1000000; i++ {
    m.Inc()
}
// Delete a metric from the vector. If you have previously kept a handle
// to that metric (as above), future updates via that handle will go
// unseen (even if you re-create a metric with the same label set
// later).
httpReqs.DeleteLabelValues("200", "GET")
// Same thing with the more verbose Labels syntax.
httpReqs.Delete(prometheus.Labels{"method": "GET", "code": "200"})
```

## func NewCounterVec

```go
func NewCounterVec(opts CounterOpts, labelNames []string) *CounterVec
```

NewCounterVec creates a new CounterVec based on the provided CounterOpts and partitioned by the given label names.

## func (*CounterVec) CurryWith

```go
func (v *CounterVec) CurryWith(labels Labels) (*CounterVec, error)
```

CurryWith returns a vector curried with the provided labels, i.e. the returned vector has those labels pre-set for all labeled operations performed on it. The cardinality of the curried vector is reduced accordingly. The order of the remaining labels stays the same (just with the curried labels taken out of the sequence – which is relevant for the (GetMetric)WithLabelValues methods). It is possible to curry a curried vector, but only with labels not yet used for currying before.

The metrics contained in the CounterVec are shared between the curried and uncurried vectors. They are just accessed differently. Curried and uncurried vectors behave identically in terms of collection. Only one must be registered with a given registry (usually the uncurried version). The Reset method deletes all metrics, even if called on a curried vector.

## func (CounterVec) Delete

```go
func (m CounterVec) Delete(labels Labels) bool
```

Delete deletes the metric where the variable labels are the same as those passed in as labels. It returns true if a metric was deleted.

It is not an error if the number and names of the Labels are inconsistent with those of the VariableLabels in Desc. However, such inconsistent Labels can never match an actual metric, so the method will always return false in that case.

This method is used for the same purpose as DeleteLabelValues(...string). See there for pros and cons of the two methods.

## func (CounterVec) DeleteLabelValues

```go
func (m CounterVec) DeleteLabelValues(lvs ...string) bool
```

DeleteLabelValues removes the metric where the variable labels are the same as those passed in as labels (same order as the VariableLabels in Desc). It returns true if a metric was deleted.

It is not an error if the number of label values is not the same as the number of VariableLabels in Desc. However, such inconsistent label count can never match an actual metric, so the method will always return false in that case.

Note that for more than one label value, this method is prone to mistakes caused by an incorrect order of arguments. Consider Delete(Labels) as an alternative to avoid that type of mistake. For higher label numbers, the latter has a much more readable (albeit more verbose) syntax, but it comes with a performance overhead (for creating and processing the Labels map). See also the CounterVec example.

## func (*CounterVec) GetMetricWith

```go
func (v *CounterVec) GetMetricWith(labels Labels) (Counter, error)
```

GetMetricWith returns the Counter for the given Labels map (the label names must match those of the VariableLabels in Desc). If that label map is accessed for the first time, a new Counter is created. Implications of creating a Counter without using it and keeping the Counter for later use are the same as for GetMetricWithLabelValues.

An error is returned if the number and names of the Labels are inconsistent with those of the VariableLabels in Desc (minus any curried labels).

This method is used for the same purpose as GetMetricWithLabelValues(...string). See there for pros and cons of the two methods.

## func (*CounterVec) GetMetricWithLabelValues

```go
func (v *CounterVec) GetMetricWithLabelValues(lvs ...string) (Counter, error)
```

GetMetricWithLabelValues returns the Counter for the given slice of label values (same order as the VariableLabels in Desc). If that combination of label values is accessed for the first time, a new Counter is created.

It is possible to call this method without using the returned Counter to only create the new Counter but leave it at its starting value 0. See also the SummaryVec example.

Keeping the Counter for later use is possible (and should be considered if performance is critical), but keep in mind that Reset, DeleteLabelValues and Delete can be used to delete the Counter from the CounterVec. In that case, the Counter will still exist, but it will not be exported anymore, even if a Counter with the same label values is created later.

An error is returned if the number of label values is not the same as the number of VariableLabels in Desc (minus any curried labels).

Note that for more than one label value, this method is prone to mistakes caused by an incorrect order of arguments. Consider GetMetricWith(Labels) as an alternative to avoid that type of mistake. For higher label numbers, the latter has a much more readable (albeit more verbose) syntax, but it comes with a performance overhead (for creating and processing the Labels map). See also the GaugeVec example.

## func (*CounterVec) MustCurryWith

```go
func (v *CounterVec) MustCurryWith(labels Labels) *CounterVec
```

MustCurryWith works as CurryWith but panics where CurryWith would have returned an error.

## func (*CounterVec) With

```go
func (v *CounterVec) With(labels Labels) Counter
```

With works as GetMetricWith, but panics where GetMetricWithLabels would have returned an error. Not returning an error allows shortcuts like

```go
myVec.With(prometheus.Labels{"code": "404", "method": "GET"}).Add(42)
func (*CounterVec) WithLabelValues
func (v *CounterVec) WithLabelValues(lvs ...string) Counter
```

WithLabelValues works as GetMetricWithLabelValues, but panics where GetMetricWithLabelValues would have returned an error. Not returning an error allows shortcuts like

```go 
myVec.WithLabelValues("404", "GET").Add(42)
type Desc
type Desc struct {
    // contains filtered or unexported fields
}
```

Desc is the descriptor used by every Prometheus Metric. It is essentially the immutable meta-data of a Metric. The normal Metric implementations included in this package manage their Desc under the hood. Users only have to deal with Desc if they use advanced features like the ExpvarCollector or custom Collectors and Metrics.

Descriptors registered with the same registry have to fulfill certain consistency and uniqueness criteria if they share the same fully-qualified name: They must have the same help string and the same label names (aka label dimensions) in each, constLabels and variableLabels, but they must differ in the values of the constLabels.

Descriptors that share the same fully-qualified names and the same label values of their constLabels are considered equal.

Use NewDesc to create new Desc instances.

## func NewDesc

```go
func NewDesc(fqName, help string, variableLabels []string, constLabels Labels) *Desc
```

NewDesc allocates and initializes a new Desc. Errors are recorded in the Desc and will be reported on registration time. variableLabels and constLabels can be nil if no such labels should be set. fqName and help must not be empty.

variableLabels only contain the label names. Their label values are variable and therefore not part of the Desc. (They are managed within the Metric.)

For constLabels, the label values are constant. Therefore, they are fully specified in the Desc. See the Collector example for a usage pattern.

## func NewInvalidDesc

```go
func NewInvalidDesc(err error) *Desc
```

NewInvalidDesc returns an invalid descriptor, i.e. a descriptor with the provided error set. If a collector returning such a descriptor is registered, registration will fail with the provided error. NewInvalidDesc can be used by a Collector to signal inability to describe itself.

## func (*Desc) String

```go
func (d *Desc) String() string
```

## type Gatherer

```go
type Gatherer interface {
    // Gather calls the Collect method of the registered Collectors and then
    // gathers the collected metrics into a lexicographically sorted slice
    // of uniquely named MetricFamily protobufs. Gather ensures that the
    // returned slice is valid and self-consistent so that it can be used
    // for valid exposition. As an exception to the strict consistency
    // requirements described for metric.Desc, Gather will tolerate
    // different sets of label names for metrics of the same metric family.
    //
    // Even if an error occurs, Gather attempts to gather as many metrics as
    // possible. Hence, if a non-nil error is returned, the returned
    // MetricFamily slice could be nil (in case of a fatal error that
    // prevented any meaningful metric collection) or contain a number of
    // MetricFamily protobufs, some of which might be incomplete, and some
    // might be missing altogether. The returned error (which might be a
    // MultiError) explains the details. Note that this is mostly useful for
    // debugging purposes. If the gathered protobufs are to be used for
    // exposition in actual monitoring, it is almost always better to not
    // expose an incomplete result and instead disregard the returned
    // MetricFamily protobufs in case the returned error is non-nil.
    Gather() ([]*dto.MetricFamily, error)
}
```

Gatherer is the interface for the part of a registry in charge of gathering the collected metrics into a number of MetricFamilies. The Gatherer interface comes with the same general implication as described for the Registerer interface.

## type GathererFunc

```go
type GathererFunc func() ([]*dto.MetricFamily, error)
```

GathererFunc turns a function into a Gatherer.

## func (GathererFunc) Gather

```go
func (gf GathererFunc) Gather() ([]*dto.MetricFamily, error)
```

Gather implements Gatherer.

## type Gatherers

```go
type Gatherers []Gatherer
```

Gatherers is a slice of Gatherer instances that implements the Gatherer interface itself. Its Gather method calls Gather on all Gatherers in the slice in order and returns the merged results. Errors returned from the Gather calles are all returned in a flattened MultiError. Duplicate and inconsistent Metrics are skipped (first occurrence in slice order wins) and reported in the returned error.

Gatherers can be used to merge the Gather results from multiple Registries. It also provides a way to directly inject existing MetricFamily protobufs into the gathering by creating a custom Gatherer with a Gather method that simply returns the existing MetricFamily protobufs. Note that no registration is involved (in contrast to Collector registration), so obviously registration-time checks cannot happen. Any inconsistencies between the gathered MetricFamilies are reported as errors by the Gather method, and inconsistent Metrics are dropped. Invalid parts of the MetricFamilies (e.g. syntactically invalid metric or label names) will go undetected.

### Example

```go
reg := prometheus.NewRegistry()
temp := prometheus.NewGaugeVec(
    prometheus.GaugeOpts{
        Name: "temperature_kelvin",
        Help: "Temperature in Kelvin.",
    },
    []string{"location"},
)
reg.MustRegister(temp)
temp.WithLabelValues("outside").Set(273.14)
temp.WithLabelValues("inside").Set(298.44)
 
var parser expfmt.TextParser
 
text := `
# TYPE humidity_percent gauge
# HELP humidity_percent Humidity in %.
humidity_percent{location="outside"} 45.4
humidity_percent{location="inside"} 33.2
# TYPE temperature_kelvin gauge
# HELP temperature_kelvin Temperature in Kelvin.
temperature_kelvin{location="somewhere else"} 4.5
`
 
parseText := func() ([]*dto.MetricFamily, error) {
    parsed, err := parser.TextToMetricFamilies(strings.NewReader(text))
    if err != nil {
        return nil, err
    }
    var result []*dto.MetricFamily
    for _, mf := range parsed {
        result = append(result, mf)
    }
    return result, nil
}
 
gatherers := prometheus.Gatherers{
    reg,
    prometheus.GathererFunc(parseText),
}
 
gathering, err := gatherers.Gather()
if err != nil {
    fmt.Println(err)
}
 
out := &bytes.Buffer{}
for _, mf := range gathering {
    if _, err := expfmt.MetricFamilyToText(out, mf); err != nil {
        panic(err)
    }
}
fmt.Print(out.String())
fmt.Println("----------")
 
// Note how the temperature_kelvin metric family has been merged from
// different sources. Now try
text = `
# TYPE humidity_percent gauge
# HELP humidity_percent Humidity in %.
humidity_percent{location="outside"} 45.4
humidity_percent{location="inside"} 33.2
# TYPE temperature_kelvin gauge
# HELP temperature_kelvin Temperature in Kelvin.
# Duplicate metric:
temperature_kelvin{location="outside"} 265.3
 # Missing location label (note that this is undesirable but valid):
temperature_kelvin 4.5
`
 
gathering, err = gatherers.Gather()
if err != nil {
    fmt.Println(err)
}
// Note that still as many metrics as possible are returned:
out.Reset()
for _, mf := range gathering {
    if _, err := expfmt.MetricFamilyToText(out, mf); err != nil {
        panic(err)
    }
}
fmt.Print(out.String())
```

output

```
# HELP humidity_percent Humidity in %.
# TYPE humidity_percent gauge
humidity_percent{location="inside"} 33.2
humidity_percent{location="outside"} 45.4
# HELP temperature_kelvin Temperature in Kelvin.
# TYPE temperature_kelvin gauge
temperature_kelvin{location="inside"} 298.44
temperature_kelvin{location="outside"} 273.14
temperature_kelvin{location="somewhere else"} 4.5
----------
collected metric temperature_kelvin label:<name:"location" value:"outside" > gauge:<value:265.3 >  was collected before with the same name and label values
# HELP humidity_percent Humidity in %.
# TYPE humidity_percent gauge
humidity_percent{location="inside"} 33.2
humidity_percent{location="outside"} 45.4
# HELP temperature_kelvin Temperature in Kelvin.
# TYPE temperature_kelvin gauge
temperature_kelvin 4.5
temperature_kelvin{location="inside"} 298.44
temperature_kelvin{location="outside"} 273.14
```

## func (Gatherers) Gather

```go
func (gs Gatherers) Gather() ([]*dto.MetricFamily, error)
```

Gather implements Gatherer.

## type Gauge

```go
type Gauge interface {
    Metric
    Collector
 
    // Set sets the Gauge to an arbitrary value.
    Set(float64)
    // Inc increments the Gauge by 1. Use Add to increment it by arbitrary
    // values.
    Inc()
    // Dec decrements the Gauge by 1. Use Sub to decrement it by arbitrary
    // values.
    Dec()
    // Add adds the given value to the Gauge. (The value can be negative,
    // resulting in a decrease of the Gauge.)
    Add(float64)
    // Sub subtracts the given value from the Gauge. (The value can be
    // negative, resulting in an increase of the Gauge.)
    Sub(float64)
 
    // SetToCurrentTime sets the Gauge to the current Unix time in seconds.
    SetToCurrentTime()
}
```

Gauge is a Metric that represents a single numerical value that can arbitrarily go up and down.

A Gauge is typically used for measured values like temperatures or current memory usage, but also "counts" that can go up and down, like the number of running goroutines.

To create Gauge instances, use NewGauge.

### Example

```go
opsQueued := prometheus.NewGauge(prometheus.GaugeOpts{
    Namespace: "our_company",
    Subsystem: "blob_storage",
    Name:      "ops_queued",
    Help:      "Number of blob storage operations waiting to be processed.",
})
prometheus.MustRegister(opsQueued)
 
// 10 operations queued by the goroutine managing incoming requests.
opsQueued.Add(10)
// A worker goroutine has picked up a waiting operation.
opsQueued.Dec()
// And once more...
opsQueued.Dec()
```

## func NewGauge

```go
func NewGauge(opts GaugeOpts) Gauge
```

NewGauge creates a new Gauge based on the provided GaugeOpts.

The returned implementation is optimized for a fast Set method. If you have a choice for managing the value of a Gauge via Set vs. Inc/Dec/Add/Sub, pick the former. For example, the Inc method of the returned Gauge is slower than the Inc method of a Counter returned by NewCounter. This matches the typical scenarios for Gauges and Counters, where the former tends to be Set-heavy and the latter Inc-heavy.

## type GaugeFunc

```go
type GaugeFunc interface {
    Metric
    Collector
}
```

GaugeFunc is a Gauge whose value is determined at collect time by calling a provided function.

To create GaugeFunc instances, use NewGaugeFunc.

### Example

```go
if err := prometheus.Register(prometheus.NewGaugeFunc(
    prometheus.GaugeOpts{
        Subsystem: "runtime",
        Name:      "goroutines_count",
        Help:      "Number of goroutines that currently exist.",
    },
    func() float64 { return float64(runtime.NumGoroutine()) },
)); err == nil {
    fmt.Println("GaugeFunc 'goroutines_count' registered.")
}
// Note that the count of goroutines is a gauge (and not a counter) as
// it can go up and down.
```

output

```
GaugeFunc 'goroutines_count' registered.
```

## func NewGaugeFunc

```go
func NewGaugeFunc(opts GaugeOpts, function func() float64) GaugeFunc
```

NewGaugeFunc creates a new GaugeFunc based on the provided GaugeOpts. The value reported is determined by calling the given function from within the Write method. Take into account that metric collection may happen concurrently. If that results in concurrent calls to Write, like in the case where a GaugeFunc is directly registered with Prometheus, the provided function must be concurrency-safe.

## type GaugeOpts

```go
type GaugeOpts Opts
```

GaugeOpts is an alias for Opts. See there for doc comments.

## type GaugeVec

```go
type GaugeVec struct {
    // contains filtered or unexported fields
}
```

GaugeVec is a Collector that bundles a set of Gauges that all share the same Desc, but have different values for their variable labels. This is used if you want to count the same thing partitioned by various dimensions (e.g. number of operations queued, partitioned by user and operation type). Create instances with NewGaugeVec.

### Example

```go
opsQueued := prometheus.NewGaugeVec(
    prometheus.GaugeOpts{
        Namespace: "our_company",
        Subsystem: "blob_storage",
        Name:      "ops_queued",
        Help:      "Number of blob storage operations waiting to be processed, partitioned by user and type.",
    },
    []string{
        // Which user has requested the operation?
        "user",
        // Of what type is the operation?
        "type",
    },
)
prometheus.MustRegister(opsQueued)
 
// Increase a value using compact (but order-sensitive!) WithLabelValues().
opsQueued.WithLabelValues("bob", "put").Add(4)
// Increase a value with a map using WithLabels. More verbose, but order
// doesn't matter anymore.
opsQueued.With(prometheus.Labels{"type": "delete", "user": "alice"}).Inc()
```

## func NewGaugeVec

```go
func NewGaugeVec(opts GaugeOpts, labelNames []string) *GaugeVec
```
NewGaugeVec creates a new GaugeVec based on the provided GaugeOpts and partitioned by the given label names.

## func (*GaugeVec) CurryWith

```go
func (v *GaugeVec) CurryWith(labels Labels) (*GaugeVec, error)
```

CurryWith returns a vector curried with the provided labels, i.e. the returned vector has those labels pre-set for all labeled operations performed on it. The cardinality of the curried vector is reduced accordingly. The order of the remaining labels stays the same (just with the curried labels taken out of the sequence – which is relevant for the (GetMetric)WithLabelValues methods). It is possible to curry a curried vector, but only with labels not yet used for currying before.

The metrics contained in the GaugeVec are shared between the curried and uncurried vectors. They are just accessed differently. Curried and uncurried vectors behave identically in terms of collection. Only one must be registered with a given registry (usually the uncurried version). The Reset method deletes all metrics, even if called on a curried vector.

## func (GaugeVec) Delete

```go
func (m GaugeVec) Delete(labels Labels) bool
```

Delete deletes the metric where the variable labels are the same as those passed in as labels. It returns true if a metric was deleted.

It is not an error if the number and names of the Labels are inconsistent with those of the VariableLabels in Desc. However, such inconsistent Labels can never match an actual metric, so the method will always return false in that case.

This method is used for the same purpose as DeleteLabelValues(...string). See there for pros and cons of the two methods.

## func (GaugeVec) DeleteLabelValues

```go
func (m GaugeVec) DeleteLabelValues(lvs ...string) bool
```

DeleteLabelValues removes the metric where the variable labels are the same as those passed in as labels (same order as the VariableLabels in Desc). It returns true if a metric was deleted.

It is not an error if the number of label values is not the same as the number of VariableLabels in Desc. However, such inconsistent label count can never match an actual metric, so the method will always return false in that case.

Note that for more than one label value, this method is prone to mistakes caused by an incorrect order of arguments. Consider Delete(Labels) as an alternative to avoid that type of mistake. For higher label numbers, the latter has a much more readable (albeit more verbose) syntax, but it comes with a performance overhead (for creating and processing the Labels map). See also the CounterVec example.

## func (*GaugeVec) GetMetricWith

```go
func (v *GaugeVec) GetMetricWith(labels Labels) (Gauge, error)
```

GetMetricWith returns the Gauge for the given Labels map (the label names must match those of the VariableLabels in Desc). If that label map is accessed for the first time, a new Gauge is created. Implications of creating a Gauge without using it and keeping the Gauge for later use are the same as for GetMetricWithLabelValues.

An error is returned if the number and names of the Labels are inconsistent with those of the VariableLabels in Desc (minus any curried labels).

This method is used for the same purpose as GetMetricWithLabelValues(...string). See there for pros and cons of the two methods.

## func (*GaugeVec) GetMetricWithLabelValues

```go
func (v *GaugeVec) GetMetricWithLabelValues(lvs ...string) (Gauge, error)
```

GetMetricWithLabelValues returns the Gauge for the given slice of label values (same order as the VariableLabels in Desc). If that combination of label values is accessed for the first time, a new Gauge is created.

It is possible to call this method without using the returned Gauge to only create the new Gauge but leave it at its starting value 0. See also the SummaryVec example.

Keeping the Gauge for later use is possible (and should be considered if performance is critical), but keep in mind that Reset, DeleteLabelValues and Delete can be used to delete the Gauge from the GaugeVec. In that case, the Gauge will still exist, but it will not be exported anymore, even if a Gauge with the same label values is created later. See also the CounterVec example.

An error is returned if the number of label values is not the same as the number of VariableLabels in Desc (minus any curried labels).

Note that for more than one label value, this method is prone to mistakes caused by an incorrect order of arguments. Consider GetMetricWith(Labels) as an alternative to avoid that type of mistake. For higher label numbers, the latter has a much more readable (albeit more verbose) syntax, but it comes with a performance overhead (for creating and processing the Labels map).

## func (*GaugeVec) MustCurryWith

```go
func (v *GaugeVec) MustCurryWith(labels Labels) *GaugeVec
```

MustCurryWith works as CurryWith but panics where CurryWith would have returned an error.

## func (*GaugeVec) With

```go
func (v *GaugeVec) With(labels Labels) Gauge
```

With works as GetMetricWith, but panics where GetMetricWithLabels would have returned an error. Not returning an error allows shortcuts like

```go
myVec.With(prometheus.Labels{"code": "404", "method": "GET"}).Add(42)
```

## func (*GaugeVec) WithLabelValues

```go
func (v *GaugeVec) WithLabelValues(lvs ...string) Gauge
```

WithLabelValues works as GetMetricWithLabelValues, but panics where GetMetricWithLabelValues would have returned an error. Not returning an error allows shortcuts like

```go
myVec.WithLabelValues("404", "GET").Add(42)
```

## type Histogram

```go
type Histogram interface {
    Metric
    Collector
 
    // Observe adds a single observation to the histogram.
    Observe(float64)
}
```

A Histogram counts individual observations from an event or sample stream in configurable buckets. Similar to a summary, it also provides a sum of observations and an observation count.

On the Prometheus server, quantiles can be calculated from a Histogram using the histogram_quantile function in the query language.

Note that Histograms, in contrast to Summaries, can be aggregated with the Prometheus query language (see the documentation for detailed procedures). However, Histograms require the user to pre-define suitable buckets, and they are in general less accurate. The Observe method of a Histogram has a very low performance overhead in comparison with the Observe method of a Summary.

To create Histogram instances, use NewHistogram.

### Example

```go
temps := prometheus.NewHistogram(prometheus.HistogramOpts{
    Name:    "pond_temperature_celsius",
    Help:    "The temperature of the frog pond.", // Sorry, we can't measure how badly it smells.
    Buckets: prometheus.LinearBuckets(20, 5, 5),  // 5 buckets, each 5 centigrade wide.
})
 
// Simulate some observations.
for i := 0; i < 1000; i++ {
    temps.Observe(30 + math.Floor(120*math.Sin(float64(i)*0.1))/10)
}
 
// Just for demonstration, let's check the state of the histogram by
// (ab)using its Write method (which is usually only used by Prometheus
// internally).
metric := &dto.Metric{}
temps.Write(metric)
fmt.Println(proto.MarshalTextString(metric))
```

output

```
histogram: <
  sample_count: 1000
  sample_sum: 29969.50000000001
  bucket: <
    cumulative_count: 192
    upper_bound: 20
  >
  bucket: <
    cumulative_count: 366
    upper_bound: 25
  >
  bucket: <
    cumulative_count: 501
    upper_bound: 30
  >
  bucket: <
    cumulative_count: 638
    upper_bound: 35
  >
  bucket: <
    cumulative_count: 816
    upper_bound: 40
  >
>
```

## func NewHistogram

```go
func NewHistogram(opts HistogramOpts) Histogram
```

NewHistogram creates a new Histogram based on the provided HistogramOpts. It panics if the buckets in HistogramOpts are not in strictly increasing order.

## type HistogramOpts

```go
type HistogramOpts struct {
    // Namespace, Subsystem, and Name are components of the fully-qualified
    // name of the Histogram (created by joining these components with
    // "_"). Only Name is mandatory, the others merely help structuring the
    // name. Note that the fully-qualified name of the Histogram must be a
    // valid Prometheus metric name.
    Namespace string
    Subsystem string
    Name      string
 
    // Help provides information about this Histogram. Mandatory!
    //
    // Metrics with the same fully-qualified name must have the same Help
    // string.
    Help string
 
    // ConstLabels are used to attach fixed labels to this metric. Metrics
    // with the same fully-qualified name must have the same label names in
    // their ConstLabels.
    //
    // ConstLabels are only used rarely. In particular, do not use them to
    // attach the same labels to all your metrics. Those use cases are
    // better covered by target labels set by the scraping Prometheus
    // server, or by one specific metric (e.g. a build_info or a
    // machine_role metric). See also
    // https://prometheus.io/docs/instrumenting/writing_exporters/#target-labels,-not-static-scraped-labels
    ConstLabels Labels
 
    // Buckets defines the buckets into which observations are counted. Each
    // element in the slice is the upper inclusive bound of a bucket. The
    // values must be sorted in strictly increasing order. There is no need
    // to add a highest bucket with +Inf bound, it will be added
    // implicitly. The default value is DefBuckets.
    Buckets []float64
}
```

HistogramOpts bundles the options for creating a Histogram metric. It is mandatory to set Name and Help to a non-empty string. All other fields are optional and can safely be left at their zero value.

## type HistogramVec

```go
type HistogramVec struct {
    // contains filtered or unexported fields
}
```

HistogramVec is a Collector that bundles a set of Histograms that all share the same Desc, but have different values for their variable labels. This is used if you want to count the same thing partitioned by various dimensions (e.g. HTTP request latencies, partitioned by status code and method). Create instances with NewHistogramVec.

## func NewHistogramVec

```go
func NewHistogramVec(opts HistogramOpts, labelNames []string) *HistogramVec
```

NewHistogramVec creates a new HistogramVec based on the provided HistogramOpts and partitioned by the given label names.

## func (*HistogramVec) CurryWith

```go
func (v *HistogramVec) CurryWith(labels Labels) (ObserverVec, error)
```

CurryWith returns a vector curried with the provided labels, i.e. the returned vector has those labels pre-set for all labeled operations performed on it. The cardinality of the curried vector is reduced accordingly. The order of the remaining labels stays the same (just with the curried labels taken out of the sequence – which is relevant for the (GetMetric)WithLabelValues methods). It is possible to curry a curried vector, but only with labels not yet used for currying before.

The metrics contained in the HistogramVec are shared between the curried and uncurried vectors. They are just accessed differently. Curried and uncurried vectors behave identically in terms of collection. Only one must be registered with a given registry (usually the uncurried version). The Reset method deletes all metrics, even if called on a curried vector.

## func (HistogramVec) Delete

```go
func (m HistogramVec) Delete(labels Labels) bool
```

Delete deletes the metric where the variable labels are the same as those passed in as labels. It returns true if a metric was deleted.

It is not an error if the number and names of the Labels are inconsistent with those of the VariableLabels in Desc. However, such inconsistent Labels can never match an actual metric, so the method will always return false in that case.

This method is used for the same purpose as DeleteLabelValues(...string). See there for pros and cons of the two methods.

## func (HistogramVec) DeleteLabelValues

```go
func (m HistogramVec) DeleteLabelValues(lvs ...string) bool
```

DeleteLabelValues removes the metric where the variable labels are the same as those passed in as labels (same order as the VariableLabels in Desc). It returns true if a metric was deleted.

It is not an error if the number of label values is not the same as the number of VariableLabels in Desc. However, such inconsistent label count can never match an actual metric, so the method will always return false in that case.

Note that for more than one label value, this method is prone to mistakes caused by an incorrect order of arguments. Consider Delete(Labels) as an alternative to avoid that type of mistake. For higher label numbers, the latter has a much more readable (albeit more verbose) syntax, but it comes with a performance overhead (for creating and processing the Labels map). See also the CounterVec example.

## func (*HistogramVec) GetMetricWith

```go
func (v *HistogramVec) GetMetricWith(labels Labels) (Observer, error)
```

GetMetricWith returns the Histogram for the given Labels map (the label names must match those of the VariableLabels in Desc). If that label map is accessed for the first time, a new Histogram is created. Implications of creating a Histogram without using it and keeping the Histogram for later use are the same as for GetMetricWithLabelValues.

An error is returned if the number and names of the Labels are inconsistent with those of the VariableLabels in Desc (minus any curried labels).

This method is used for the same purpose as GetMetricWithLabelValues(...string). See there for pros and cons of the two methods.

## func (*HistogramVec) GetMetricWithLabelValues

```go
func (v *HistogramVec) GetMetricWithLabelValues(lvs ...string) (Observer, error)
```

GetMetricWithLabelValues returns the Histogram for the given slice of label values (same order as the VariableLabels in Desc). If that combination of label values is accessed for the first time, a new Histogram is created.

It is possible to call this method without using the returned Histogram to only create the new Histogram but leave it at its starting value, a Histogram without any observations.

Keeping the Histogram for later use is possible (and should be considered if performance is critical), but keep in mind that Reset, DeleteLabelValues and Delete can be used to delete the Histogram from the HistogramVec. In that case, the Histogram will still exist, but it will not be exported anymore, even if a Histogram with the same label values is created later. See also the CounterVec example.

An error is returned if the number of label values is not the same as the number of VariableLabels in Desc (minus any curried labels).

Note that for more than one label value, this method is prone to mistakes caused by an incorrect order of arguments. Consider GetMetricWith(Labels) as an alternative to avoid that type of mistake. For higher label numbers, the latter has a much more readable (albeit more verbose) syntax, but it comes with a performance overhead (for creating and processing the Labels map). See also the GaugeVec example.

## func (*HistogramVec) MustCurryWith

```go
func (v *HistogramVec) MustCurryWith(labels Labels) ObserverVec
```

MustCurryWith works as CurryWith but panics where CurryWith would have returned an error.

## func (*HistogramVec) With

```go
func (v *HistogramVec) With(labels Labels) Observer
```

With works as GetMetricWith but panics where GetMetricWithLabels would have returned an error. Not returning an error allows shortcuts like

```go
myVec.With(prometheus.Labels{"code": "404", "method": "GET"}).Observe(42.21)
```

## func (*HistogramVec) WithLabelValues

```go
func (v *HistogramVec) WithLabelValues(lvs ...string) Observer
```

WithLabelValues works as GetMetricWithLabelValues, but panics where GetMetricWithLabelValues would have returned an error. Not returning an error allows shortcuts like

```go
myVec.WithLabelValues("404", "GET").Observe(42.21)
```

## type LabelPairSorter

```go
type LabelPairSorter []*dto.LabelPair
```

LabelPairSorter implements sort.Interface. It is used to sort a slice of dto.LabelPair pointers. This is useful for implementing the Write method of custom metrics.

### Example

```go
lablePairs := []*dtoLabelPair{
    {Name: proto.String("status"), Value: proto.String("404")},
    {Name: proto.String("method"), Value: proto.String("get")},
}
 
sort.Sort(prometheus.LabelPairSorter(labelPair))
 
fmt.Println(labelPair)
```

output

```
[name:"method" value:"get" name:"status" value:"404"]
```


## func (LabelPairSorter) Len

```go
func (s LabelPairSorter) Len() int
```

## func (LabelPairSorter) Less

```go
func (s LabelPairSorter) Less(i, j int) bool
```

## func (LabelPairSorter) Swap

```go
func (s LabelPairSorter) Swap(i, j int)
```

## type Labels

```go
type Labels map[string]string
```

Labels represents a collection of label name -> value mappings. This type is commonly used with the With(Labels) and GetMetricWith(Labels) methods of metric vector Collectors, e.g.:

```go
myVec.With(Labels{"code": "404", "method": "GET"}).Add(42)
```

The other use-case is the specification of constant label pairs in Opts or to create a Desc.

## type Metric

```go
type Metric interface {
// Desc returns the descriptor for the Metric. This method idempotently
    // returns the same descriptor throughout the lifetime of the
    // Metric. The returned descriptor is immutable by contract. A Metric
    // unable to describe itself must return an invalid descriptor (created
    // with NewInvalidDesc).
    Desc() *Desc
    // Write encodes the Metric into a "Metric" Protocol Buffer data
    // transmission object.
    //
    // Metric implementations must observe concurrency safety as reads of
    // this metric may occur at any time, and any blocking occurs at the
    // expense of total performance of rendering all registered
    // metrics. Ideally, Metric implementations should support concurrent
    // readers.
    //
    // While populating dto.Metric, it is the responsibility of the
    // implementation to ensure validity of the Metric protobuf (like valid
    // UTF-8 strings or syntactically valid metric and label names). It is
    // recommended to sort labels lexicographically. (Implementers may find
    // LabelPairSorter useful for that.) Callers of Write should still make
    // sure of sorting if they depend on it.
    Write(*dto.Metric) error
}
```

A Metric models a single sample value with its meta data being exported to Prometheus. Implementations of Metric in this package are Gauge, Counter, Histogram, Summary, and Untyped.

## func MustNewConstHistogram

```go
func MustNewConstHistogram(
    desc *Desc,
    count uint64,
    sum float64,
    buckets map[float64]uint64,
    labelValues ...string,
) Metric
```

MustNewConstHistogram is a version of NewConstHistogram that panics where NewConstMetric would have returned an error.

## func MustNewConstMetric

```go
func MustNewConstMetric(desc *Desc, valueType ValueType, value float64, labelValues ...string) Metric
```

MustNewConstMetric is a version of NewConstMetric that panics where NewConstMetric would have returned an error.

## func MustNewConstSummary

```go
func MustNewConstSummary(
    desc *Desc,
    count uint64,
    sum float64,
    quantiles map[float64]float64,
    labelValues ...string,
) Metric
```

MustNewConstSummary is a version of NewConstSummary that panics where NewConstMetric would have returned an error.

## func NewConstHistogram

```go
func NewConstHistogram(
    desc *Desc,
    count uint64,
    sum float64,
    buckets map[float64]uint64,
    labelValues ...string,
) (Metric, error)
```

NewConstHistogram returns a metric representing a Prometheus histogram with fixed values for the count, sum, and bucket counts. As those parameters cannot be changed, the returned value does not implement the Histogram interface (but only the Metric interface). Users of this package will not have much use for it in regular operations. However, when implementing custom Collectors, it is useful as a throw-away metric that is generated on the fly to send it to Prometheus in the Collect method.

buckets is a map of upper bounds to cumulative counts, excluding the +Inf bucket.

NewConstHistogram returns an error if the length of labelValues is not consistent with the variable labels in Desc.

### Example

```go
desc := prometheus.NewDesc(
    "http_request_duration_seconds",
    "A histogram of HTTP request durations.",
    []string{"code", "method"},
    prometheus.Lables{"owner": example},
)
 
// Create a constant histogram of values we get form a 3rd party telemetry system
h: = prometheus.MustNewConstHistogram(
    desc,
    4711, 403.34,
    map[float64]unint64{25:121, 50:2403, 100:3221, 200:4233},
    "200", "get",
)
 
// Just for demonstration, lets check the  state of the histogram by
// (ab)using its Write method (which is usually only used by Prometheus internally).
metric := &dto.Metric{}
h.Write(metric)
fmt.Println(proto.MarshTextString(metric))
```

output

```
label: <
  name: "code"
  value: "200"
>
label: <
  name: "method"
  value: "get"
>
label: <
  name: "ower"
  value: "example"
>
histogram: <
  smaple_count:4711
  smaple_sum:403.34
  bucket: <
    cumulative_count: 121
    upper_bound: 25
  >
  bucket: <
    cumulative_count: 2403
    upper_bound: 50
  >
  bucket: <
    cumulative_count: 3221
    upper_bound: 100
  >
  bucket: <
    cumulative_count: 4233
    upper_bound: 200
  >
>
```

## func NewConstMetric

```go
func NewConstMetric(
    desc *Desc,
    valueType ValueType,
    value float64,
    labelValues ...string,
) (Metric, error)
```

NewConstMetric returns a metric with one fixed value that cannot be changed. Users of this package will not have much use for it in regular operations. However, when implementing custom Collectors, it is useful as a throw-away metric that is generated on the fly to send it to Prometheus in the Collect method. NewConstMetric returns an error if the length of labelValues is not consistent with the variable labels in Desc.

## func NewConstSummary

```go
func NewConstSummary(
    desc *Desc,
    count uint64,
    sum float64,
    quantiles map[float64]float64,
    labelValues ...string,
) (Metric, error)
```

NewConstSummary returns a metric representing a Prometheus summary with fixed values for the count, sum, and quantiles. As those parameters cannot be changed, the returned value does not implement the Summary interface (but only the Metric interface). Users of this package will not have much use for it in regular operations. However, when implementing custom Collectors, it is useful as a throw-away metric that is generated on the fly to send it to Prometheus in the Collect method.

quantiles maps ranks to quantile values. For example, a median latency of 0.23s and a 99th percentile latency of 0.56s would be expressed as:

```go
map[float64]float64{0.5: 0.23, 0.99: 0.56}
```

NewConstSummary returns an error if the length of labelValues is not consistent with the variable labels in Desc.

### Example:

```go
desc := prometheus.NewDesc(
    "http_request_duration_seconds",
    "A histogram of HTTP request durations.",
    []string{"code", "method"},
    prometheus.Lables{"owner": example},
)
 
// Create a constant summary of values we get form a 3rd party telemetry system
h: = prometheus.MustNewConstSummary(
    desc,
    4711, 403.34,
    map[float64]float{0.5:42.3, 0.9:323.3},
    "200", "get",
)
 
// Just for demonstration, lets check the  state of the summary by
// (ab)using its Write method (which is usually only used by Prometheus internally).
metric := &dto.Metric{}
h.Write(metric)
fmt.Println(proto.MarshTextString(metric))
```

output

```
label: <
  name: "code"
  value: "200"
>
label: <
  name: "method"
  value: "get"
>
label: <
  name: "ower"
  value: "example"
>
summary: <
  sample_count: 4711
  sample_sum:403.34
  quantile: <
    quantile: 0.5
    value: 42.3
  >
  quantile: <
    quantile: 0.9
    value: 323.3
  >
>
```

## func NewInvalidMetric

```go
func NewInvalidMetric(desc *Desc, err error) Metric
```

NewInvalidMetric returns a metric whose Write method always returns the provided error. It is useful if a Collector finds itself unable to collect a metric and wishes to report an error to the registry.

## type MultiError

```go
type MultiError []error
```

MultiError is a slice of errors implementing the error interface. It is used by a Gatherer to report multiple errors during MetricFamily gathering.

## func (*MultiError) Append

```go
func (errs *MultiError) Append(err error)
```

Append appends the provided error if it is not nil.

## func (MultiError) Error

```go
func (errs MultiError) Error() string
```

## func (MultiError) MaybeUnwrap

```go
func (errs MultiError) MaybeUnwrap() error
```

MaybeUnwrap returns nil if len(errs) is 0. It returns the first and only contained error as error if len(errs is 1). In all other cases, it returns the MultiError directly. This is helpful for returning a MultiError in a way that only uses the MultiError if needed.

## type Observer

```go
type Observer interface { Observe(float64) }
```

Observer is the interface that wraps the Observe method, which is used by Histogram and Summary to add observations.

## type ObserverFunc

```go
type ObserverFunc func(float64)
```

The ObserverFunc type is an adapter to allow the use of ordinary functions as Observers. If f is a function with the appropriate signature, ObserverFunc(f) is an Observer that calls f.

This adapter is usually used in connection with the Timer type, and there are two general use cases:

The most common one is to use a Gauge as the Observer for a Timer. See the "Gauge" Timer example.

The more advanced use case is to create a function that dynamically decides which Observer to use for observing the duration. See the "Complex" Timer example.

## func (ObserverFunc) Observe

```go
func (f ObserverFunc) Observe(value float64)
```

Observe calls f(value). It implements Observer.

## type ObserverVec

```go
type ObserverVec interface {
    GetMetricWith(Labels) (Observer, error)
    GetMetricWithLabelValues(lvs ...string) (Observer, error)
    With(Labels) Observer
    WithLabelValues(...string) Observer
    CurryWith(Labels) (ObserverVec, error)
    MustCurryWith(Labels) ObserverVec
 
    Collector
}
```

ObserverVec is an interface implemented by HistogramVec and SummaryVec.

## type Opts

```go
type Opts struct {
    // Namespace, Subsystem, and Name are components of the fully-qualified
    // name of the Metric (created by joining these components with
    // "_"). Only Name is mandatory, the others merely help structuring the
    // name. Note that the fully-qualified name of the metric must be a
    // valid Prometheus metric name.
    Namespace string
    Subsystem string
    Name      string
 
    // Help provides information about this metric. Mandatory!
    //
    // Metrics with the same fully-qualified name must have the same Help
    // string.
    Help string
 
    // ConstLabels are used to attach fixed labels to this metric. Metrics
    // with the same fully-qualified name must have the same label names in
    // their ConstLabels.
    //
    // ConstLabels are only used rarely. In particular, do not use them to
    // attach the same labels to all your metrics. Those use cases are
    // better covered by target labels set by the scraping Prometheus
    // server, or by one specific metric (e.g. a build_info or a
    // machine_role metric). See also
    // https://prometheus.io/docs/instrumenting/writing_exporters/#target-labels,-not-static-scraped-labels
    ConstLabels Labels
}
```

Opts bundles the options for creating most Metric types. Each metric implementation XXX has its own XXXOpts type, but in most cases, it is just be an alias of this type (which might change when the requirement arises.)

It is mandatory to set Name and Help to a non-empty string. All other fields are optional and can safely be left at their zero value.

## type Registerer

```go
type Registerer interface {
    // Register registers a new Collector to be included in metrics
    // collection. It returns an error if the descriptors provided by the
    // Collector are invalid or if they — in combination with descriptors of
    // already registered Collectors — do not fulfill the consistency and
    // uniqueness criteria described in the documentation of metric.Desc.
    //
    // If the provided Collector is equal to a Collector already registered
    // (which includes the case of re-registering the same Collector), the
    // returned error is an instance of AlreadyRegisteredError, which
    // contains the previously registered Collector.
    //
    // It is in general not safe to register the same Collector multiple
    // times concurrently.
    Register(Collector) error
    // MustRegister works like Register but registers any number of
    // Collectors and panics upon the first registration that causes an
    // error.
    MustRegister(...Collector)
    // Unregister unregisters the Collector that equals the Collector passed
    // in as an argument.  (Two Collectors are considered equal if their
    // Describe method yields the same set of descriptors.) The function
    // returns whether a Collector was unregistered.
    //
    // Note that even after unregistering, it will not be possible to
    // register a new Collector that is inconsistent with the unregistered
    // Collector, e.g. a Collector collecting metrics with the same name but
    // a different help string. The rationale here is that the same registry
    // instance must only collect consistent metrics throughout its
    // lifetime.
    Unregister(Collector) bool
}
```

Registerer is the interface for the part of a registry in charge of registering and unregistering. Users of custom registries should use Registerer as type for registration purposes (rather than the Registry type directly). In that way, they are free to use custom Registerer implementation (e.g. for testing purposes).

## type Registry

```go
type Registry struct { // contains filtered or unexported fields }
```

Registry registers Prometheus collectors, collects their metrics, and gathers them into MetricFamilies for exposition. It implements both Registerer and Gatherer. The zero value is not usable. Create instances with NewRegistry or NewPedanticRegistry.

## func NewPedanticRegistry

```go
func NewPedanticRegistry() *Registry
```

NewPedanticRegistry returns a registry that checks during collection if each collected Metric is consistent with its reported Desc, and if the Desc has actually been registered with the registry.

Usually, a Registry will be happy as long as the union of all collected Metrics is consistent and valid even if some metrics are not consistent with their own Desc or a Desc provided by their registered Collector. Well-behaved Collectors and Metrics will only provide consistent Descs. This Registry is useful to test the implementation of Collectors and Metrics.

## func NewRegistry

```go
func NewRegistry() *Registry
```

NewRegistry creates a new vanilla Registry without any Collectors pre-registered.

## func (*Registry) Gather

```go
func (r *Registry) Gather() ([]*dto.MetricFamily, error)
```

Gather implements Gatherer.

## func (*Registry) MustRegister

```go
func (r *Registry) MustRegister(cs ...Collector)
```

MustRegister implements Registerer.

## func (*Registry) Register

```go
func (r *Registry) Register(c Collector) error
```

Register implements Registerer.

func (*Registry) Unregister
func (r *Registry) Unregister(c Collector) bool

Unregister implements Registerer.

## type Summary

```go
type Summary interface {
    Metric
    Collector
 
    // Observe adds a single observation to the summary.
    Observe(float64)
}
```

A Summary captures individual observations from an event or sample stream and summarizes them in a manner similar to traditional summary statistics: 1. sum of observations, 2. observation count, 3. rank estimations.

A typical use-case is the observation of request latencies. By default, a Summary provides the median, the 90th and the 99th percentile of the latency as rank estimations. However, the default behavior will change in the upcoming v0.10 of the library. There will be no rank estiamtions at all by default. For a sane transition, it is recommended to set the desired rank estimations explicitly.

Note that the rank estimations cannot be aggregated in a meaningful way with the Prometheus query language (i.e. you cannot average or add them). If you need aggregatable quantiles (e.g. you want the 99th percentile latency of all queries served across all instances of a service), consider the Histogram metric type. See the Prometheus documentation for more details.

To create Summary instances, use NewSummary.

### Example

```go
temps :=prometheus.NewSummary(prometheus.SummaryOpts{
    Name: "pond_temperature_celsius",
    Help: "The temperature of the frog pond.",
    Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
})
 
// Simulate some objectivations
for i:= 0; i < 1000; i++ {
    temps.Observe(30 + math.Floor(120*math.Sin(float64(i)*0.1))/100)
}
 
// Just for domestration, let's check the state of the summary by (ab)using
// its Write method (which is usually only used by Prometheus internally)
metric:=&dto.Metric{}
temps.Write(metric)
fmt.Println(proto.MarshalTextString(metric))
```

output

```
''' summary: < sample_count: 1000 sample_sum: 29969.5000000000001 quantile: < quantile: 0.5 value 31.1

quantile: < quantile: 0.9 value 41.3

quantile: < quantile: 0.99 value 41.9

'''
```

## func NewSummary

```go
func NewSummary(opts SummaryOpts) Summary
```

NewSummary creates a new Summary based on the provided SummaryOpts.

## type SummaryOpts

```go
type SummaryOpts struct {
    // Namespace, Subsystem, and Name are components of the fully-qualified
    // name of the Summary (created by joining these components with
    // "_"). Only Name is mandatory, the others merely help structuring the
    // name. Note that the fully-qualified name of the Summary must be a
    // valid Prometheus metric name.
    Namespace string
    Subsystem string
    Name      string
 
    // Help provides information about this Summary. Mandatory!
    //
    // Metrics with the same fully-qualified name must have the same Help
    // string.
    Help string
 
    // ConstLabels are used to attach fixed labels to this metric. Metrics
    // with the same fully-qualified name must have the same label names in
    // their ConstLabels.
    //
    // ConstLabels are only used rarely. In particular, do not use them to
    // attach the same labels to all your metrics. Those use cases are
    // better covered by target labels set by the scraping Prometheus
    // server, or by one specific metric (e.g. a build_info or a
    // machine_role metric). See also
    // https://prometheus.io/docs/instrumenting/writing_exporters/#target-labels,-not-static-scraped-labels
    ConstLabels Labels
 
    // Objectives defines the quantile rank estimates with their respective
    // absolute error. If Objectives[q] = e, then the value reported for q
    // will be the φ-quantile value for some φ between q-e and q+e.  The
    // default value is DefObjectives. It is used if Objectives is left at
    // its zero value (i.e. nil). To create a Summary without Objectives,
    // set it to an empty map (i.e. map[float64]float64{}).
    //
    // Deprecated: Note that the current value of DefObjectives is
    // deprecated. It will be replaced by an empty map in v0.10 of the
    // library. Please explicitly set Objectives to the desired value.
    Objectives map[float64]float64
 
    // MaxAge defines the duration for which an observation stays relevant
    // for the summary. Must be positive. The default value is DefMaxAge.
    MaxAge time.Duration
 
    // AgeBuckets is the number of buckets used to exclude observations that
    // are older than MaxAge from the summary. A higher number has a
    // resource penalty, so only increase it if the higher resolution is
    // really required. For very high observation rates, you might want to
    // reduce the number of age buckets. With only one age bucket, you will
    // effectively see a complete reset of the summary each time MaxAge has
    // passed. The default value is DefAgeBuckets.
    AgeBuckets uint32
 
    // BufCap defines the default sample stream buffer size.  The default
    // value of DefBufCap should suffice for most uses. If there is a need
    // to increase the value, a multiple of 500 is recommended (because that
    // is the internal buffer size of the underlying package
    // "github.com/bmizerany/perks/quantile").
    BufCap uint32
}
```

SummaryOpts bundles the options for creating a Summary metric. It is mandatory to set Name and Help to a non-empty string. While all other fields are optional and can safely be left at their zero value, it is recommended to explicitly set the Objectives field to the desired value as the default value will change in the upcoming v0.10 of the library.

## type SummaryVec

```go
type SummaryVec struct {
    // contains filtered or unexported fields
}
```

SummaryVec is a Collector that bundles a set of Summaries that all share the same Desc, but have different values for their variable labels. This is used if you want to count the same thing partitioned by various dimensions (e.g. HTTP request latencies, partitioned by status code and method). Create instances with NewSummaryVec.

### Example

```go
temps :=prometheus.NewSummaryVec(
    prometheus.SummaryOpts{
        Name: "pond_temperature_celsius",
        Help: "The temperature of the frog pond.",
        Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
    },
    []string{"species"},
)
 
// Simulate some objectivations
for i:= 0; i < 1000; i++ {
    temps.WithLabelValues("litoria-caerulea").Observe(30 + math.Floor(120*math.Sin(float64(i)*0.1))/10)
    temps.WithLabelValues("lithobates-catesbeianus").Observe(32 + math.Floor(100*math.Sin(float64(i)*0.11))/10)
}
 
// Create a summary without any observations.
temps.WithLabelValues("leiopelma-hochstetteri")
 
// Just for domestration, let's check the state of the summary vector by registering
// it with s custom registry and then let it collect the metrics
reg := prometheus.NewRegistry()
reg.MustRegisty(temps)
 
metricFamilies, err := reg.Getheer()
if err!= nil || len(metricFamilier) != 1 {
    panic("unexpected behavior of custom test registory")
}
 
fmt.Println(proto.MarshalTextString(metricFamilier[0]))
```

output

```
name: "pond_temperature_celsius"
help: "The temperature of the frog pond."
type: "summary"
metric: <
  label: <
    name: "species"
    value: "leiopelma-hochstetteri"
  >
  summary: <
    sample_count: 0
    sample_sum: 0
    quantile: <
      quantile: 0.5
      value: nan
    >
    quantile: <
      quantile: 0.9
      value: nan
    >
    quantile: <
      quantile: 0.99
      value: nan
    >
  >
>
metric: <
  label: <
    name: "species"
    value: "lithobates-catesbeianus"
  >
  summary: <
    sample_count: 1000
    sample_sum: 31956.1000000000007
    quantile: <
      quantile: 0.5
      value: 32.4
    >
    quantile: <
      quantile: 0.9
      value: 41.4
    >
    quantile: <
      quantile: 0.99
      value: 41.9
    >
  >
>
metric: <
  label: <
    name: "species"
    value: "litoria-caerulea"
  >
  summary: <
    sample_count: 1000
    sample_sum: 29969.5000000000001
    quantile: <
      quantile: 0.5
      value: 31.1
    >
    quantile: <
      quantile: 0.9
      value: 41.3
    >
    quantile: <
      quantile: 0.99
      value: 41.9
    >
  >
>
```

## func NewSummaryVec

```go
func NewSummaryVec(opts SummaryOpts, labelNames []string) *SummaryVec
```

NewSummaryVec creates a new SummaryVec based on the provided SummaryOpts and partitioned by the given label names.

## func (*SummaryVec) CurryWith

```go
func (v *SummaryVec) CurryWith(labels Labels) (ObserverVec, error)
```

CurryWith returns a vector curried with the provided labels, i.e. the returned vector has those labels pre-set for all labeled operations performed on it. The cardinality of the curried vector is reduced accordingly. The order of the remaining labels stays the same (just with the curried labels taken out of the sequence – which is relevant for the (GetMetric)WithLabelValues methods). It is possible to curry a curried vector, but only with labels not yet used for currying before.

The metrics contained in the SummaryVec are shared between the curried and uncurried vectors. They are just accessed differently. Curried and uncurried vectors behave identically in terms of collection. Only one must be registered with a given registry (usually the uncurried version). The Reset method deletes all metrics, even if called on a curried vector.

## func (SummaryVec) Delete

```go
func (m SummaryVec) Delete(labels Labels) bool
```

Delete deletes the metric where the variable labels are the same as those passed in as labels. It returns true if a metric was deleted.

It is not an error if the number and names of the Labels are inconsistent with those of the VariableLabels in Desc. However, such inconsistent Labels can never match an actual metric, so the method will always return false in that case.

This method is used for the same purpose as DeleteLabelValues(...string). See there for pros and cons of the two methods.

## func (SummaryVec) DeleteLabelValues

```go
func (m SummaryVec) DeleteLabelValues(lvs ...string) bool
```

DeleteLabelValues removes the metric where the variable labels are the same as those passed in as labels (same order as the VariableLabels in Desc). It returns true if a metric was deleted.

It is not an error if the number of label values is not the same as the number of VariableLabels in Desc. However, such inconsistent label count can never match an actual metric, so the method will always return false in that case.

Note that for more than one label value, this method is prone to mistakes caused by an incorrect order of arguments. Consider Delete(Labels) as an alternative to avoid that type of mistake. For higher label numbers, the latter has a much more readable (albeit more verbose) syntax, but it comes with a performance overhead (for creating and processing the Labels map). See also the CounterVec example.

## func (*SummaryVec) GetMetricWith

```go
func (v *SummaryVec) GetMetricWith(labels Labels) (Observer, error)
```

GetMetricWith returns the Summary for the given Labels map (the label names must match those of the VariableLabels in Desc). If that label map is accessed for the first time, a new Summary is created. Implications of creating a Summary without using it and keeping the Summary for later use are the same as for GetMetricWithLabelValues.

An error is returned if the number and names of the Labels are inconsistent with those of the VariableLabels in Desc (minus any curried labels).

This method is used for the same purpose as GetMetricWithLabelValues(...string). See there for pros and cons of the two methods.

## func (*SummaryVec) GetMetricWithLabelValues

```go
func (v *SummaryVec) GetMetricWithLabelValues(lvs ...string) (Observer, error)
```

GetMetricWithLabelValues returns the Summary for the given slice of label values (same order as the VariableLabels in Desc). If that combination of label values is accessed for the first time, a new Summary is created.

It is possible to call this method without using the returned Summary to only create the new Summary but leave it at its starting value, a Summary without any observations.

Keeping the Summary for later use is possible (and should be considered if performance is critical), but keep in mind that Reset, DeleteLabelValues and Delete can be used to delete the Summary from the SummaryVec. In that case, the Summary will still exist, but it will not be exported anymore, even if a Summary with the same label values is created later. See also the CounterVec example.

An error is returned if the number of label values is not the same as the number of VariableLabels in Desc (minus any curried labels).

Note that for more than one label value, this method is prone to mistakes caused by an incorrect order of arguments. Consider GetMetricWith(Labels) as an alternative to avoid that type of mistake. For higher label numbers, the latter has a much more readable (albeit more verbose) syntax, but it comes with a performance overhead (for creating and processing the Labels map). See also the GaugeVec example.

## func (*SummaryVec) MustCurryWith

```go
func (v *SummaryVec) MustCurryWith(labels Labels) ObserverVec
```

MustCurryWith works as CurryWith but panics where CurryWith would have returned an error.

## func (*SummaryVec) With

```go
func (v *SummaryVec) With(labels Labels) Observer
```

With works as GetMetricWith, but panics where GetMetricWithLabels would have returned an error. Not returning an error allows shortcuts like

```go
myVec.With(prometheus.Labels{"code": "404", "method": "GET"}).Observe(42.21)
```

## func (*SummaryVec) WithLabelValues

```go
func (v *SummaryVec) WithLabelValues(lvs ...string) Observer
```

WithLabelValues works as GetMetricWithLabelValues, but panics where GetMetricWithLabelValues would have returned an error. Not returning an error allows shortcuts like

```go
myVec.WithLabelValues("404", "GET").Observe(42.21)
```

## type Timer

```go
type Timer struct { // contains filtered or unexported fields }
```

Timer is a helper type to time functions. Use NewTimer to create new instances.

### Example

```go
package main
 
import (
    "math/rand"
    "time"
   
    "github.com/prometheus/client_golang/prometheus"
)
 
var (
    requestDuration = prometheus.NewHistogram(prometheus.HistogramOpts{
        Name:      "example_request_duration_seconds",
    Help:      "Histogram for the runtime of a simple example function."
    Buckets    prometheus.LinearBuckets(0.01, 0.01, 10),
    })
)
 
func main() {
    // timer  times the example function. It uses a Histogram but a summary would also works.
    // as both impliment both impliment Observer. Check out: https://prometheus.io/docs/practices/histogram
    timer:=prometheus.NetTimer(requestDuration)
    defer timer.ObserverDuration()
     
    //Do something here that task time
    time.Sleep(time.Duration(rand.NormFloat64()*10000+50000)*time.MicroSecond)
}
```

### Example (Complex)

```go
package main
 
import (
    "net/http"
 
    "github.com/prometheus/client_golang/prometheus"
)
 
var (
    // apiRequestDuration tracks the duration separate for each HTTP status
    // class (1xx, 2xx, ...). This creates a fair amount of time series on
    // the Prometheus server. Usually, you would track the duration of
    // serving HTTP request without partitioning by outcome. Do something
    // like this only if needed. Also note how only status classes are
    // tracked, not every single status code. The latter would create an
    // even larger amount of time series. Request counters partitioned by
    // status code are usually OK as each counter only creates one time
    // series. Histograms are way more expensive, so partition with care and
    // only where you really need separate latency tracking. Partitioning by
    // status class is only an example. In concrete cases, other partitions
    // might make more sense.
    apiRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "api_request_duration_seconds",
            Help:    "Histogram for the request duration of the public API, partitioned by status class.",
            Buckets: prometheus.ExponentialBuckets(0.1, 1.5, 5),
        },
        []string{"status_class"},
    )
)
 
func handler(w http.ResponseWriter, r *http.Request) {
    status := http.StatusOK
    // The ObserverFunc gets called by the deferred ObserveDuration and
    // decides which Histogram's Observe method is called.
    timer := prometheus.NewTimer(prometheus.ObserverFunc(func(v float64) {
        switch {
        case status >= 500: // Server error.
            apiRequestDuration.WithLabelValues("5xx").Observe(v)
        case status >= 400: // Client error.
            apiRequestDuration.WithLabelValues("4xx").Observe(v)
        case status >= 300: // Redirection.
            apiRequestDuration.WithLabelValues("3xx").Observe(v)
        case status >= 200: // Success.
            apiRequestDuration.WithLabelValues("2xx").Observe(v)
        default: // Informational.
            apiRequestDuration.WithLabelValues("1xx").Observe(v)
        }
    }))
    defer timer.ObserveDuration()
 
    // Handle the request. Set status accordingly.
    // ...
}
 
func main() {
    http.HandleFunc("/api", handler)
}
```

### Example (Gauge)

```go
package main
 
import (
    "os"
 
    "github.com/prometheus/client_golang/prometheus"
)
 
var (
    // If a function is called rarely (i.e. not more often than scrapes
    // happen) or ideally only once (like in a batch job), it can make sense
    // to use a Gauge for timing the function call. For timing a batch job
    // and pushing the result to a Pushgateway, see also the comprehensive
    // example in the push package.
    funcDuration = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "example_function_duration_seconds",
        Help: "Duration of the last call of an example function.",
    })
)
 
func run() error {
    // The Set method of the Gauge is used to observe the duration.
    timer := prometheus.NewTimer(prometheus.ObserverFunc(funcDuration.Set))
    defer timer.ObserveDuration()
 
    // Do something. Return errors as encountered. The use of 'defer' above
    // makes sure the function is still timed properly.
    return nil
}
 
func main() {
    if err := run(); err != nil {
        os.Exit(1)
    }
}
```

## func NewTimer

```go
func NewTimer(o Observer) *Timer
```
NewTimer creates a new Timer. The provided Observer is used to observe a duration in seconds. Timer is usually used to time a function call in the following way:

```go
func TimeMe() {
    timer := NewTimer(myHistogram)
    defer timer.ObserveDuration()
    // Do actual work.
}
```

## func (*Timer) ObserveDuration

```go
func (t *Timer) ObserveDuration()
```

ObserveDuration records the duration passed since the Timer was created with NewTimer. It calls the Observe method of the Observer provided during construction with the duration in seconds as an argument. ObserveDuration is usually called with a defer statement.

Note that this method is only guaranteed to never observe negative durations if used with Go1.9+.

## type UntypedFunc

```go
type UntypedFunc interface { Metric Collector }
```

UntypedFunc works like GaugeFunc but the collected metric is of type "Untyped". UntypedFunc is useful to mirror an external metric of unknown type.

To create UntypedFunc instances, use NewUntypedFunc.

## func NewUntypedFunc

```go
func NewUntypedFunc(opts UntypedOpts, function func() float64) UntypedFunc
```

NewUntypedFunc creates a new UntypedFunc based on the provided UntypedOpts. The value reported is determined by calling the given function from within the Write method. Take into account that metric collection may happen concurrently. If that results in concurrent calls to Write, like in the case where an UntypedFunc is directly registered with Prometheus, the provided function must be concurrency-safe.

## type UntypedOpts

```go
type UntypedOpts Opts
```

UntypedOpts is an alias for Opts. See there for doc comments.

## type ValueType

```go
type ValueType int
```

ValueType is an enumeration of metric types that represent a simple value.

```go
const ( CounterValue ValueType GaugeValue UntypedValue )
```

Possible values for the ValueType enum.

## Directories
Path
Synopsis
graphite	Package graphite provides a bridge to push Prometheus metrics to a Graphite server.
promauto	Package promauto provides constructors for the usual Prometheus metrics that return them already registered with the global registry (prometheus.DefaultRegisterer).
promhttp	Package promhttp provides tooling around HTTP servers and clients.
push	Package push provides functions to push metrics to a Pushgateway.
Package prometheus imports 27 packages (graph) and is imported by 5702 packages. Updated 6 days ago. Refresh now. Tools for package owners.

