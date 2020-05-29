# 关闭prometheus client-go的默认go metrics的方法

prometheus默认采集go metrics，可以通过自定义prometheus.Gatherers 来关闭默认的采集参数：

```go
var gathers prometheus.Gatherers

if *disableDefaultMetrics {
		gathers = prometheus.Gatherers{reg}
} else {
		gathers = prometheus.Gatherers{reg, prometheus.DefaultGatherer}
}

h := promhttp.HandlerFor(gathers, promhttp.HandlerOpts{
			ErrorLog:      log.NewErrorLogger(),
			ErrorHandling: promhttp.ContinueOnError,
  })

http.HandleFunc(*metricPath, func(w http.ResponseWriter, r *http.Request) {
		h.ServeHTTP(w, r)
})
```

代码中prometheus.Gatherers用来定义一个采集数据的收集器集合，可以merge多个不同的采集数据到一个结果集合，reg是我们之前生成的一个注册对象，用来自定义采集数据，DefaultGatherer是go运行时默认的一些指标信息。

(1)原来默认使用 http.Handle(*metricPath, prometheus.Handler())会产生默认的采集数据：
prometheus.Handler()包含了DefaultGather，所以会产生默认的采集数据，想要关闭默认的采集数据，可以自定义一个prometheus.Gatherers，通过传入disableDefaultMetrics参数来判断是否包含默认的采集数据。

(2)promhttp.HandlerFor()函数传递之前自定义的Gatherers对象，并返回一个httpHandler对象，这个httpHandler对象可以调用其自身的ServHTTP函数来接收http请求，并返回响应。其中promhttp.HandlerOpts定义了采集过程中如果发生错误时，继续采集其他的数据。
