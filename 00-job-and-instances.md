# Job and Instance

prometheus中，一个能够被prometheus服务拉取（pull）metrics的端点叫instance，通常对于一个进程。具有相同目的的instances的集合，例如为了可靠性和伸缩性而复制的某个进程的副本集合，可以组成一个job。

自动生成的label和time series

当prometheus从某个目标订阅数据时，prometheus会自动把下面两个label添加到metrics的标签上:

- job    目标所属的配置任务的名称，如 prometheus
- instance    采样点所在的服务实例，如 localhost:9090

如果它们和metrics中的label冲突，则后续行为取决于honor_labels配置。

honor_labels用来控制prometheus如何处理其拉取的metrics数据中的label和prometheus默认的label冲突问题。
如果honor_labels设置为true，则保留exporter暴露的metrics的labels，丢弃prometheus server端自动添加的labels。

如果honor_labels设置为false，则保留冲突的label，但是拉取数据中的metrics的冲突label会被修改label_name为

> "exported_<origin_label>"（也就是exported_job和exported_instance）。

对于每个instance，prometheus还会在服务端存储一下时序数据：

>up{job="<job_name>", instance="<instance-id>"}:1   如果instance服务正常，值为1，否则值为0
>scrape_duration_seconds{job="<job_name>", instance="<instance-id>"}: pull数据耗时
>scrape_sample_post_metric_relabeling{job_name>", instance="<instance-id>"}:metrics重新标签后，没有变化的metrics数量
>scrape_samples_scraped{job_name>", instance="<instance-id>"}:instance暴露的metrics的数量

up对服务监控的监控很有意义。例如：

```
GET http://localhost:9090/api/v1/query?query=up
 
HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Thu, 28 Jun 2018 08:37:02 GMT
 
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "__name__": "up",
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "value": [
          1530175022.521,
          "1"
        ]
      },
      {
        "metric": {
          "__name__": "up",
          "instance": "localhost:9100",
          "job": "node_apm_server"
        },
        "value": [
          1530175022.521,
          "1"
        ]
      },
      {
        "metric": {
          "__name__": "up",
          "instance": "localhost:9292",
          "job": "postgres"
        },
        "value": [
          1530175022.521,
          "0"
        ]
      }
    ]
  }
}
 
Response code: 200 (OK); Time: 21ms; Content length: 380 bytes
```
