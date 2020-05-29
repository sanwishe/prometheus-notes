# 为什么使用pushGateway

在某些情况下，没有可以从中抓取指标的目标。造成这种情况的原因有很多：

- 安全性或连接性问题，使你无法访问目标资源。这是一种非常常见的情况，比如服务或应用程序仅允许特定端口或路径访问。
- 目标资源的生命周期太短，例如容器的启动、执行和停止。在这种情况下，Prometheus作业将会发现目标已完成执行并且不再可以被抓取。
- 目标资源没有可以抓取的端点，例如批处理作业。批处理作业不太可能具有可被抓取的HTTP服务，即使假设作业运行的时间足够长。

在这些情况下，我们需要某种方法来将时间序列传递或推送到Prometheus服务器。

Pushgateway只是指标的临时停靠站。推送到Pushgateway，指标已经有了job和instance标签，用于指示指标的来源。如果我们希望在Prometheus中保留这些信息，而不是在抓取网关时由服务器进行重写。如果honor_labels设置为true，那么Prometheus将使用Pushgateway上的job和instance标签。如果设置为false，那么它将重命名这些值，在它们前面加上exported_前缀，并在服务器上为这些标签附加新值。

所以，prometheus官方给出的pushgateway的场景就是批处理任务监控：

> Usually, the only valid use case for the Pushgateway is for capturing the outcome of a service-level batch job.

