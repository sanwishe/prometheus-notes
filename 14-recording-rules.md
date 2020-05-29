# recording rules

prometheus支持通过制定规则来定是执行相关动作，包括记录规则（recording rules）和告警规则（alerting rules）。为了使规则在prometheus中生效，需要将规则的声明记录到yaml文件中，在prometheus.yml的rule_files域中配置规则文件。

## 语法检查

在没有启动Prometheus服务之前，可以通过安装和运行Prometheus的promtool命令行工具检验:

```bash
./promtool check rules /path/to/examples.rules
```

当记录规则文件是有效的，则这个检查会打印出解析到规则的文本表示，并以返回值0退出程序。
如果有任何语法错误的话，则这个命令行会打印出一个错误信息到标准输出，并以返回值1退出程序。

## 记录规则

记录规则允许预先定时计算经常需要查询的或者计算复杂度高的表达式，并将结果保存为一组新的时间序列数据，所以查询这些预先计算好的数据要比在需要查询是计算更快。在一些情况下，需要反复查询同样的表达式，比如dashboard的刷新，这种方式就特别有意义。

记录规则必须在一个规则组rule group下，同一个rule group下的规则有着相同的执行间隔。规则文件中，规则组定义如下：

```
groups:
    [ - <rule_group> ]
一个简单的规则文件样例：

groups:
  - name: example
    rules:
    - record: job:http_inprogress_requests:sum
      expr: sum(http_inprogress_requests) by (job)
```

###  rule group

```
# The name of the group. Must be unique within a file.
name: <string>
 
# How often rules in the group are evaluated.
# 规则执行的间隔，默认为global.evaluation_interval
[ interval: <duration> | default = global.evaluation_interval ]
 
rules:
  [ - <rule> ... ]
```

### rules

规则定义的语法如下：

```
# 该规则定时计算输出的新的时间序列数据的名称，必须是合法的metric名称
record: <string>
 
# The PromQL expression to evaluate. Every evaluation cycle this is
# evaluated at the current time, and the result recorded as a new set of
# time series with the metric name as given by 'record'.
# prometheus查询表达式(PromQL)，每一次的计算结果都保留在record字段定义的新的时间序列数据中。
expr: <string>
 
# Labels to add or overwrite before storing the result.
labels:
  [ <labelname>: <labelvalue> ]
```

### example

每五分钟计算一次Counter指标test_counter_increase_total在过去5分钟内的增量，存储为新的metric名称：count_every_five_minutes_total

```
groups:
  - name: count_every_5m
    interval: 5m
    rules:
    - record: count_every_five_minutes_total
      expr: increase(test_counter_increase_total[5m])
prometheus.yml规则配置：

rule_files:
  # 配置文件路径为相对prometheus.yml路径
  - "rule.yml"
```

查询结果：

```
GET http://10.62.127.88:9090/api/v1/query?query=count_every_five_minutes_total%5B1h%5D
 
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
Date: Mon, 02 Jul 2018 00:14:34 GMT
 
{
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [
      {
        "metric": {
          "__name__": "count_every_five_minutes_total",
          "instance": "localhost:9299",
          "job": "test"
        },
        "values": [
          [
            1530486965.255,
            "149.4736842105263"
          ],
          [
            1530487265.255,
            "149.4736842105263"
          ],
          [
            1530487565.255,
            "149.4736842105263"
          ],
          [
            1530487865.255,
            "149.4736842105263"
          ],
          [
            1530488165.255,
            "149.47420868143396"
          ],
          [
            1530488465.255,
            "149.4736842105263"
          ],
          [
            1530488765.255,
            "149.4736842105263"
          ],
          [
            1530489065.255,
            "149.4736842105263"
          ],
          [
            1530489365.255,
            "149.4736842105263"
          ],
          [
            1530489665.255,
            "149.4736842105263"
          ],
          [
            1530489965.255,
            "149.4736842105263"
          ],
          [
            1530490265.255,
            "149.4736842105263"
          ]
        ]
      }
    ]
  }
}
 
Response code: 200 (OK); Time: 18ms; Content length: 616 bytes
```
