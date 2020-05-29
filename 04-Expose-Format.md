# metrics导出格式

Prometheus实现了两种不同的数据传输格式，exporter可以使用检测的度量指标给Prometheus服务。

第一种：是基于文本的简单格式；

第二种：更有效和更强大的protocol buffer格式。

Prometheus服务和客户端通过HTTP通信获取要使用的是哪种协议。

注意：prometheus2.0之后的版本已经移除了对protocol-buffer格式的支持。

## Format v0.0.4

v0.0.4是当前metrics的数据传输格式版本

到这个版本，Prometheus有两种备用的数据传输格式：基于协议缓冲的格式和文本格式。客户端必须支持这两种可选格式中的一种。

另外，客户端可选择性地提供不被Prometheus server理解的其他数据文本格式，目的是为了易于理解和方便调试。强烈建议客户端库至少一种可读的格式。v0.0.4的数据传输协议容易被理解，所以它是一个很好的数据传输格式候选者（并且也被Prometheus server理解）。

文本格式和protobuf格式比较

| 比较项目 | 协议缓冲格式 |	文本格式 |
|:----:|:----:|:----:|
| 开始时间 | 2014.4月 | 2014.4月 |
| 传输协议 | HTTP |	HTTP |
| 编码 | 32-bit varint-encoded record length-delimited Protocol Buffer messages of type  io.prometheus.client.MetricFamily | utf-8, \n行结尾 |
| HTTP Content-Type	| application/vnd.google.protobuf; proto=io.prometheus.client.MetricFamily; encoding=delimited | text/plain; version=0.0.4 (A missing version value will lead to a fall-back to the most recent text format version.) |
| Optional HTTP Content-Encoding | gzip |	gzip |
| 优点 | 跨平台;  编解码代价小;  严格的范式;  支持链接和理论流（仅服务端行为需要更改）. | 可读性好;  易于组合，特别适用于简单情况（无需嵌套）;  阅读（处理类型提示和文本字符串外）. |
|  限制 | 不具有可读性 | 信息不全,类型和文档格式不是语法的组成部分，意味着很少到不存在的度量契约验证;  解析代价大. |
| 支持度量指标原子性	| Counter;  Gauge;  Histogram;  Summary; Untyped. | Counter;  Gauge;  Histogram;  Summary; Untyped. |
| 兼容性	| 低于v.0.0.3无效,v0.0.4有效 | 无 |


## Protocol buffer format details

略

## Text format details 文本格式详解

文本格式传输协议是面向行的。换行符(\n)分隔行。最后一行必须以换行字符结束。空行被忽略。

在一行内，token可以被任何数量的空格或tabs分割。头尾部的空白被忽略。

具有"#"作为第一个非空格字符的行是注释，它们被忽略，除非“#”之后的一个token是HELP或TYPE。

如果#后第一个token是HELP，则至少还需要一个token，即metric name。该行所有剩余的字符都被认为是该度量指标名称的描述信息。HELP行可以包含任何UTF-8字符序列。但\和换行字符必须分别转义为\\和\n。相同的度量名称只能有一个HELP行。

如果#后第一个token是TYPE，则至少还需要两个token，第一个是metric name，第二个是metric type，如Counter，Gauge，Histogram，Summary或者Untyped。相同的metric name只能有一个TYPE行。用于metric name的TYPE行必须出现在为该metric报告的第一个样本之前。如果度量名称没有TYPE行，则该类型将设置为无类型。

剩余行用以下语法描述样本：

```
metric_name [ "{" label_name "=" `"` label_value `"` { "," label_name "=" `"` label_value `"` } [ "," ] "}" ] value [ timestamp ]
```

metric_name和label_name具有普通的Prometheus表达式语言限制。 label_value可以是UTF-8字符的任何序列，但\，"和\n符必须分别转义为\，\“和\ n.

value为浮点数，时间戳 一个int64类型的整数（即1970-01-01 00:00:00 UTC起到当前的毫秒数，不包括闰秒）。特别是Nan，+ Inf和 -Inf是有效值。

给定度量的所有行必须作为一个不间断的组提供，可选的HELP和TYPE行首先（不是特定的顺序）。 除此之外，重复展示的可重复排序是优选的，但不是必需的，即不计算计算成本是否可以排序。

每行必须具有度量名称和标签的唯一组合。

histogram和summary类型很难以文本格式表示。 适用以下约定：

- 名为x的Summary或Histogram的样本总和作为名为x_sum的单独样本给出。
- 名为x的Summary或Histogram的样本计数作为名为x_count的单独样本给出。
- 名为x的Summary的每个分位数作为具有相同名称x和label {quantile =“y”}的单独样本行给出。
- 名为x的Histogram的每个桶计数作为单独的样本行（名称为x_bucket）和一个标签{le =“y”}（其中y是存储桶的上限）给出。
- Histogram必须带有label {le =“+ Inf”}的存储桶。 其值必须与x_count的值相同。
- Histogram的桶和总结的分位数必须以其标签值的增加数字顺序（分别为le或分位数标签）出现。 

另见下面的例子。

```
# HELP http_requests_total The total number of HTTP requests.
# TYPE http_requests_total counter
http_requests_total{method="post",code="200"} 1027 1395066363000
http_requests_total{method="post",code="400"}    3 1395066363000

# Escaping in label values:
msdos_file_access_time_seconds{path="C:\\DIR\\FILE.TXT",error="Cannot find file:\n\"FILE.TXT\""} 1.458255915e9

# Minimalistic line:
metric_without_timestamp_and_labels 12.47

# A weird metric from before the epoch:
something_weird{problem="division by zero"} +Inf -3982045

# A histogram, which has a pretty complex representation in the text format:
# HELP http_request_duration_seconds A histogram of the request duration.
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.05"} 24054
http_request_duration_seconds_bucket{le="0.1"} 33444
http_request_duration_seconds_bucket{le="0.2"} 100392
http_request_duration_seconds_bucket{le="0.5"} 129389
http_request_duration_seconds_bucket{le="1"} 133988
http_request_duration_seconds_bucket{le="+Inf"} 144320
http_request_duration_seconds_sum 53423
http_request_duration_seconds_count 144320

# Finally a summary, which has a complex representation, too:
# HELP rpc_duration_seconds A summary of the RPC duration in seconds.
# TYPE rpc_duration_seconds summary
rpc_duration_seconds{quantile="0.01"} 3102
rpc_duration_seconds{quantile="0.05"} 3272
rpc_duration_seconds{quantile="0.5"} 4773
rpc_duration_seconds{quantile="0.9"} 9001
rpc_duration_seconds{quantile="0.99"} 76656
rpc_duration_seconds_sum 1.7560473e+07
rpc_duration_seconds_count 2693
```

tips：暴露的metrics的合法性，可以使用promtool工具检查：

```bash
curl -s http://localhost:9090/metrics | ./promtool check metrics
```
