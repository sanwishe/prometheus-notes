prometheus允许PromQL表达式定义告警规则并将告警发送到外部系统。

告警规则的定义与记录规则定义类似，下面是一个告警规则示例：

```
groups:
- name: goroutines_monitoring
  rules:
  - alert: TooMuchGoroutines
    expr: go_goroutines{job="prometheus"} > 35
    for: 5m
    labels:
      cat: "thread"
    annotations:
      summary: "too much goroutines of job prometheus."
      description: "test desc"
 
- name: go_threads_monitoring
  rules:
  - alert: TooMuchGoThreads
    expr: go_threads > 20
    labels:
      cat: "thread"
    annotations:
      summary: "too much go thread of job prometheus"
      description: "testing"
```
