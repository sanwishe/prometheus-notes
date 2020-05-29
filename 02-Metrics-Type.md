# prometheus指标类型

Prometheus客户库提供了四个核心的metrics类型。这四种类型目前仅在客户库和wire协议中区分。Prometheus服务还没有充分利用这些类型。不久的将来就会发生改变。

## Counter

counter是一个累计度量指标，它是一个只能递增的数值。计数器主要用于统计服务的请求数、任务完成数和错误出现的次数等等。counter是一个递增的值。

反例：统计goroutines的数量。计数器的使用方式在下面的各个客户端例子中：

客户端使用计数器的文档：

- [Go](https://godoc.org/github.com/prometheus/client_golang/prometheus#Counter)
- [Java](https://github.com/prometheus/client_java/blob/master/simpleclient/src/main/java/io/prometheus/client/Counter.java)
- [Python](https://github.com/prometheus/client_python#counter)
- [Ruby](https://github.com/prometheus/client_ruby#counter)


## Gauge

gauge是一个度量指标，它表示一个既可以递增, 又可以递减的值。

Gauge主要用于测量类似于温度、当前内存使用量等，也可以统计当前服务运行随时增加或者减少的Goroutines数量

客户端使用计量器的文档：

- [Go](https://godoc.org/github.com/prometheus/client_golang/prometheus#Counter)
- [Java](https://godoc.org/github.com/prometheus/client_golang/prometheus#Counter)
- [Python](https://godoc.org/github.com/prometheus/client_golang/prometheus#Counter)
- [Ruby](https://github.com/prometheus/client_ruby#gauge)


## Histogram

histogram，是柱状图，在Prometheus系统中的查询语言中，有三种作用：

- 对每个采样点进行统计，打到各个分类值中(bucket)
- 对每个采样点值累计和(sum)
- 对采样点的次数累计和(count)

度量指标名称: basename的柱状图, 上面三类的作用度量指标名称:

- [basename]_bucket{le="上边界"}, 这个值为小于等于上边界的所有采样点数量
- [basename]_sum
- [basename]_count

小结：所以如果定义一个度量类型为Histogram，则Prometheus系统会自动生成三个对应的指标

使用histogram_quantile()函数, 计算直方图或者是直方图聚合计算的分位数阈值。 一个直方图计算Apdex值也是合适的, 当在buckets上操作时，记住直方图是累计的。详见直方图和总结

客户库的直方图使用文档：
- [Go](http://godoc.org/github.com/prometheus/client_golang/prometheus#Histogram)
- [Java](https://github.com/prometheus/client_java/blob/master/simpleclient/src/main/java/io/prometheus/client/Histogram.java)
- [Python](https://github.com/prometheus/client_python#histogram)
- [Ruby](https://github.com/prometheus/client_ruby#histogram)

## Summary

类似histogram，summary是采样点分位图统计，(通常的使用场景：请求持续时间和响应大小)。 它也有三种作用：

- 对于每个采样点进行统计，并形成分位图。（如：正态分布一样，统计低于60分不及格的同学比例，统计低于80分的同学比例，统计低于95分的同学比例）
- 统计班上所有同学的总成绩(sum)
- 统计班上同学的考试总人数(count)

带有度量指标的[basename]的summary 在抓取时间序列数据展示。

- 观察时间的φ-quantiles (0 ≤ φ ≤ 1), 显示为`[basename]{分位数="[φ]"}`
- `[basename]_sum`， 是指所有观察值的总和
- `[basename]_count`, 是指已观察到的事件计数值

详见[Histogram和Summaries](03-Histogram_and_Summaries.md)

有关summaries的客户端使用文档：

- [Go](http://godoc.org/github.com/prometheus/client_golang/prometheus#Summary)
- [Java](https://github.com/prometheus/client_java/blob/master/simpleclient/src/main/java/io/prometheus/client/Summary.java)
- [Python](https://github.com/prometheus/client_python#summary)
- [Ruby](https://github.com/prometheus/client_ruby#summary)



