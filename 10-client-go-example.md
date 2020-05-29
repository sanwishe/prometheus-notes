# Example usage of client-go

## example of Counter and Gauge

```go
package main
 
 
import (
    "github.com/prometheus/client_golang/prometheus"
    "math/rand"
    "time"
    "net/http"
    "github.com/prometheus/common/log"
    "strconv"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)
 
var (
    //固定label的定义
    randomCounter = prometheus.NewCounter(
        prometheus.CounterOpts{
            Namespace:   "testPrometheus",
            Subsystem:   "goExporter",
            Name:        "randomCounter",
            Help:        "random add a number each time with constant labels",
            ConstLabels: prometheus.Labels{"a": "1", "b": "2", "c": "3"},
        })
 
    randomGauge = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name:        "randomGauge",
            Help:        "random number each time",
            ConstLabels: prometheus.Labels{"name": "10170261", "version": "0.1"},
        })
 
    //可变label的定义
    variableCounter = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Namespace: "testPrometheus",
            Subsystem: "goExporter",
            Name:      "variableCounter",
            Help:      "random add a number each time with variable labels",
        },
        []string{"version", "stat"})
 
    variableGauge = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Namespace: "testPrometheus",
            Subsystem: "goExporter",
            Name:      "variableGauge",
            Help:      "random generate a number each time with variable labels",
        },
        []string{"version", "name"},
    )
)
 
func init() {
    prometheus.MustRegister(randomCounter, randomCounter, variableCounter, variableGauge)
}
 
func main() {
    // collect metrics by random value
    go func() {
        for {
            detla := float64(rand.Intn(10))
            randomGauge.Set(rand.Float64())
            randomCounter.Add(detla)
            variableCounter.WithLabelValues(strconv.Itoa(rand.Intn(10)), "OK").Add(detla)
            variableGauge.WithLabelValues(strconv.Itoa(rand.Intn(3)), "zte.com.cn").Set(rand.NormFloat64())
 
            time.Sleep(10 * time.Second)
        }
    }()
 
    //prometheus.Handler()提供了额外的四个metrics
    //`http_requests_total(CounterVec)`，`http_request_duration_microseconds(Summary)`，
    //`http_request_size_bytes(Summary)`， `http_response_size_bytes(Summary)`
    http.Handle("/metrics", prometheus.Handler())
 
    // 没有额外的四个metrics
    http.Handle("/metrics", promhttp.Handler())
    err := http.ListenAndServe("0.0.0.0:9109", nil)
    if err != nil {
        log.Fatal(err)
    }
}
```

output

```
# HELP randomGauge random number each time
# TYPE randomGauge gauge
randomGauge{name="10170261",version="0.1"} 0.7623865106846489
# HELP testPrometheus_goExporter_randomCounter random add a number each time with constant labels
# TYPE testPrometheus_goExporter_randomCounter counter
testPrometheus_goExporter_randomCounter{a="1",b="2",c="3"} 16323
# HELP testPrometheus_goExporter_variableCounter random add a number each time with variable labels
# TYPE testPrometheus_goExporter_variableCounter counter
testPrometheus_goExporter_variableCounter{stat="OK",version="0"} 1709
testPrometheus_goExporter_variableCounter{stat="OK",version="1"} 1478
testPrometheus_goExporter_variableCounter{stat="OK",version="2"} 1616
testPrometheus_goExporter_variableCounter{stat="OK",version="3"} 1672
testPrometheus_goExporter_variableCounter{stat="OK",version="4"} 1608
testPrometheus_goExporter_variableCounter{stat="OK",version="5"} 1542
testPrometheus_goExporter_variableCounter{stat="OK",version="6"} 1884
testPrometheus_goExporter_variableCounter{stat="OK",version="7"} 1682
testPrometheus_goExporter_variableCounter{stat="OK",version="8"} 1462
testPrometheus_goExporter_variableCounter{stat="OK",version="9"} 1670
# HELP testPrometheus_goExporter_variableGauge random generate a number each time with variable labels
# TYPE testPrometheus_goExporter_variableGauge gauge
testPrometheus_goExporter_variableGauge{name="zte.com.cn",version="0"} 0.6590270781530927
testPrometheus_goExporter_variableGauge{name="zte.com.cn",version="1"} -0.07884666705672927
testPrometheus_goExporter_variableGauge{name="zte.com.cn",version="2"} 0.2064617454117018
```

## example for Histogram

```go
package main
 
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "net/http"
    "log"
)
 
var (
    histogramExporter1 = prometheus.NewHistogram(
        prometheus.HistogramOpts{
            Namespace:   "umc",
            Name:        "histogram_example_with_constant_label",
            Help:        "Example help info for histogram with fixed label",
            ConstLabels: map[string]string{"version": "0.1"},
            Buckets:     []float64{0.3, 1.0, 3, 5, 10},
        },
    )
 
    histogramExporter2 = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Namespace: "umc",
            Name:      "histogram_example_with_variable_label",
            Help:      "Example help info for histogram with variable label",
            Buckets:   []float64{0.3, 1.0, 3, 5, 10},
        },
        []string{"version"},
    )
)
 
func init() {
    prometheus.MustRegister(histogramExporter2, histogramExporter1)
}
 
func main() {
    histogramExporter1.Observe(0.1)
    histogramExporter1.Observe(0.7)
    histogramExporter1.Observe(1.8)
    histogramExporter1.Observe(4.2)
    histogramExporter1.Observe(7.0)
    histogramExporter1.Observe(64)
 
    histogramExporter2.WithLabelValues("v0.2").Observe(0.1)
    histogramExporter2.WithLabelValues("v0.2").Observe(0.7)
    histogramExporter2.WithLabelValues("v0.2").Observe(1.8)
    histogramExporter2.WithLabelValues("v0.2").Observe(4.2)
    histogramExporter2.WithLabelValues("v0.2").Observe(7.0)
    histogramExporter2.WithLabelValues("v0.7").Observe(64)
 
    http.Handle("/metrics", promhttp.Handler())
 
    if err := http.ListenAndServe("0.0.0.0:9111", nil); err != nil {
        log.Fatal(err)
    }
}
```

output

```go
# HELP umc_histogram_example_with_constant_label Example help info for histogram with fixed label
# TYPE umc_histogram_example_with_constant_label histogram
umc_histogram_example_with_constant_label_bucket{version="0.1",le="0.3"} 1
umc_histogram_example_with_constant_label_bucket{version="0.1",le="1"} 2
umc_histogram_example_with_constant_label_bucket{version="0.1",le="3"} 3
umc_histogram_example_with_constant_label_bucket{version="0.1",le="5"} 4
umc_histogram_example_with_constant_label_bucket{version="0.1",le="10"} 5
umc_histogram_example_with_constant_label_bucket{version="0.1",le="+Inf"} 6
umc_histogram_example_with_constant_label_sum{version="0.1"} 77.8
umc_histogram_example_with_constant_label_count{version="0.1"} 6
# HELP umc_histogram_example_with_variable_label Example help info for histogram with variable label
# TYPE umc_histogram_example_with_variable_label histogram
umc_histogram_example_with_variable_label_bucket{version="v0.2",le="0.3"} 1
umc_histogram_example_with_variable_label_bucket{version="v0.2",le="1"} 2
umc_histogram_example_with_variable_label_bucket{version="v0.2",le="3"} 3
umc_histogram_example_with_variable_label_bucket{version="v0.2",le="5"} 4
umc_histogram_example_with_variable_label_bucket{version="v0.2",le="10"} 5
umc_histogram_example_with_variable_label_bucket{version="v0.2",le="+Inf"} 5
umc_histogram_example_with_variable_label_sum{version="v0.2"} 13.8
umc_histogram_example_with_variable_label_count{version="v0.2"} 5
umc_histogram_example_with_variable_label_bucket{version="v0.7",le="0.3"} 0
umc_histogram_example_with_variable_label_bucket{version="v0.7",le="1"} 0
umc_histogram_example_with_variable_label_bucket{version="v0.7",le="3"} 0
umc_histogram_example_with_variable_label_bucket{version="v0.7",le="5"} 0
umc_histogram_example_with_variable_label_bucket{version="v0.7",le="10"} 0
umc_histogram_example_with_variable_label_bucket{version="v0.7",le="+Inf"} 1
umc_histogram_example_with_variable_label_sum{version="v0.7"} 64
umc_histogram_example_with_variable_label_count{version="v0.7"} 1
```

## example for Summary

```go
package main
 
import (
    "github.com/prometheus/client_golang/prometheus"
    "net/http"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "log"
)
 
var (
    summaryExporter1 = prometheus.NewSummaryVec(
        prometheus.SummaryOpts{
            Namespace: "umc",
            Name:      "summary_example_with_constant_label_and_quantile",
            Help:      "Example help info for summary with fixed label and quantile",
 
            //Objectives defines the quantile rank estimates with their respective
            //absolute error. If Objectives[q] = e, then the value reported for q
            //will be the ?-quantile value for some ? between q-e and q+e.  The
            //default value is DefObjectives. It is used if Objectives is left at
            //its zero value (i.e. nil). To create a Summary without Objectives,
            //set it to an empty map (i.e. map[float64]float64{}).
            //
            //Deprecated: Note that the current value of DefObjectives is
            //deprecated. It will be replaced by an empty map in v0.10 of the
            //library. Please explicitly set Objectives to the desired value.
            Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
        },
        []string{"version"},
    )
)
 
func init() {
    prometheus.MustRegister(summaryExporter1)
}
 
func main() {
    for i := 0; i < 908; i++ {
        summaryExporter1.WithLabelValues("0.1").Observe(1)
    }
    for j := 0; j < 92; j++ {
        summaryExporter1.WithLabelValues("0.1").Observe(2)
    }
 
    for i := 0; i < 910; i++ {
        summaryExporter1.WithLabelValues("0.2").Observe(1)
    }
    for j := 0; j < 90; j++ {
        summaryExporter1.WithLabelValues("0.2").Observe(2)
    }
 
    http.Handle("/metrics", promhttp.Handler())
 
    if err := http.ListenAndServe("0.0.0.0:9110", nil); err != nil {
        log.Fatal(err)
    }
}
```

output

```go
# HELP umc_summary_example_with_constant_label_and_quantile Example help info for summary with fixed label and quantile
# TYPE umc_summary_example_with_constant_label_and_quantile summary
umc_summary_example_with_constant_label_and_quantile{version="0.1",quantile="0.5"} 1
umc_summary_example_with_constant_label_and_quantile{version="0.1",quantile="0.9"} 2
umc_summary_example_with_constant_label_and_quantile{version="0.1",quantile="0.99"} 2
umc_summary_example_with_constant_label_and_quantile_sum{version="0.1"} 1092
umc_summary_example_with_constant_label_and_quantile_count{version="0.1"} 1000
umc_summary_example_with_constant_label_and_quantile{version="0.2",quantile="0.5"} 1
umc_summary_example_with_constant_label_and_quantile{version="0.2",quantile="0.9"} 1
umc_summary_example_with_constant_label_and_quantile{version="0.2",quantile="0.99"} 2
umc_summary_example_with_constant_label_and_quantile_sum{version="0.2"} 1090
umc_summary_example_with_constant_label_and_quantile_count{version="0.2"} 1000
```

## 关闭prometheus默认采集的参数

可以通过自定义prometheus.Gatherers 来关闭默认的采集参数：

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

