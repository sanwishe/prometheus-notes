# Histogram和Summary详解

相比较Counter和Gauge，Histogram和Summary是更加复杂的指标类型。单个的Histogram或者Summary指标会创建多个时间序列值，所以正确的使用Histogram和Summary有一定的难度。

## 类库支持

prometheus客户端对Histogram和Summary的支持详见指标类型介绍章节，需要注意的是有些类库只支持两者之一，或者仅仅支持summary的部分功能，使用时需要注意。

## 采样的数量与和

Histogram和Summary都是对被观察对象的采样，典型的实例包括请求耗时或者响应体大小等。Histogram和Summary记录观察样本的数量，并且将观察到的值求和，通过这两个结果，我们可以对观察对象求平均值。 注意：观察次数（Histogram或者Summary的指标中以`_count`为后缀的时间序列）本质上是一个Counter类型的指标，即它们只增不减。观察值的和（Histogram或者Summary的指标中以`_sum`为后缀的时间序列）本质上也是一个Counter。但是对于气温等指标，由于观察到的值可能为负数，这两个序列可能会下降，此时不能再对相应的指标运用`rate()`规则。 为了从Histogram或者Summary的指标`http_request_duration_seconds`中运算过去5分钟内请求耗时的平均值，可以使用如下运算规则：

```
rate(http_request_duration_seconds_sum[5m])/rate(http_request_duration_seconds_count[5m])
```

## Apdex指数

Histogram的典型应用是将观察的值直接放入到指定的桶中。

假设有一个SLA能够在300毫秒内响应95%的请求，那么可以给Histogram指标设一个0.3秒的桶。然后可以根据桶的值和`_count`计算，如果值小于95%，就该发出告警。下面的表达式计算了以job为维度的过去5分钟内响应时间小于300毫秒的比例：

```
sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m])) by (job) / sum(rate(http_request_duration_seconds_count[5m])) by (job)
```

我们可以通过相似的方法估算Apdex Score。 为相应的观察指标配置两个桶，一个桶的上限是目标值，一个可容忍值作为另一个桶的上限（通常是目标值的四倍）。前面的例子中，目标响应时间的值为0.3s，可容忍的响应时间的值为1.2s。下面的表达式可以计算出每个job5分钟内的Apdex分数：

```
(sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m])) by (job) + sum(rate(http_request_duration_seconds_bucket{le="1.2"}[5m])) by (job) ) / 2 / sum(rate(http_request_duration_seconds_count[5m])) by (job)
```

需要指出的是，由于桶的值累加的，即label为le=0.3的桶的值也包括在label为`le=1.2`的桶里面，所以我们把两个桶的和又相加了起来，再除以2猜得到正确的结果。

上面的分析和传统的Apdex Score有一定的出入，因为在计算中包含一些错误和可容忍的偏差。

## Quantiles

Histogram和Summary都可以用来计算所谓的`φ-`分位数，其中`0≦φ≦1`。**φ分位数用来统计N个观察值中，φxN个值的分布。** 例如，0.5分位为中位数，0.95分位为第95百分位。 Histogram和Summary的本质区别在于：Summary在客户端侧计算φ分位数，并且将他们直接暴露给prometheus；Histogram在客户端暴露的是每个桶的统计数，然后通过`histogram_quantile()`函数在prometheus服务端计算分位数。 Histogram和Summary在诸多方面有区别：


| | Histogram |	Summary| 
|:----:|:----:|:----:|
| 所需要的配置 | 选择一组适合于观察值的桶	| 选择希望的分位数`φ`和滑动窗口大小`e` |
| 客户端性能 |	客户端仅需要增加计数器的值，性能消耗很小 | 由于流式的分位计算，客户端性能消耗较大 |
| 服务端性能 | 需要在服务端计算分位数等信息，可以使用记录规则计算 | 服务端性能消耗比较小 |
| 时间序列数量（除了`_sum`和`_count`） |	每个桶一个时间序列（`le="+Inf"`的桶也占用一个时间序列） |	每个分位一个时间序列 |
| 分位误差 | 误差和桶的宽度有关 | 误差与配置的φ的值有关 |
| 分位数`φ`和滑动窗口的规范 | 在服务端通过表达式`histogram_quantile()`配置 | 客户端预定义 |
| 聚合 | 在服务端通过表达式`histogram_quantile()`配置 | 不可聚合 |

注意最后一行！回到前面的SLA在300ms内响应95%的请求的例子，现在，我们不需要技术300ms内响应请求次数的百分比，只需要定义一个95的分位数。为此可以配置一个带有0.95分位数和5分钟的衰减时间的summary，或者配置一个在300ms附近有桶的histogram，如`{le =“0.1”}，{le =“0.2”}，{le =“0.3”}`和`{le =“0.45”}`。如果服务有多个实例，则可以每个实例分别采集然后聚合所有数据到0.95分位。但是如本段开头所述，summary一般不会用于聚合，对某个分位统计序列求平均值是无意义的：

```
avg(http_request_duration_seconds{quantile="0.95"}) // BAD!
```

如果需要聚合，可以使用`histogram_quantile()`聚合histogram：

```
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) // GOOD.
```

此外，如果SLA服务希望统计0.90的分位数，或者要统计最近10分钟而不是最近5分钟的数据，我们仅需要改变上面表达式中的相应值即可。

## 分位统计的误差

无论是在server端还是在client端计算分位数，都是一个估算值。所以需要了解估算的误差。

再回到上面Histogram的例子，假设正常的请求都能够在非常接近220ms的时间时完成，那么就可以在真正的统计直方图上看到值220非常尖锐。但是在上面配置的histogram指标中，几乎所有的的观察数据（95分位数据）都会落到桶`le=0.3`中，即`0.2~0.3`的区间上。如果要返回单个值（而不是间隔），它应用线性插值，在这种情况下可以产生295ms。计算出的分位数给人的印象是，服务接近打破SLA的门限，但实际上，第95百分位数与SLA容限相当远。

下一步：后端路由的更改为所有请求时间添加了固定的100ms。现在，请求持续时间在320ms的统计值急剧上升，几乎所有的观察值多落到300~450ms之间。计算第95百分位数为442.5ms，尽管正确值接近320ms。尽管结果超出300ms一丁点，但是统计结果看起来系统更加糟糕。

。。。

底线是：如果使用summary，则可以通过φ的大小控制误差。如果使用histogram，可以通过根据观察者的范围选择合适的桶来控制误差。φ的微小变化导致观测值的偏差很大。观察值的小间隔覆盖φ的大间隔。

两个经验法则：

- 如果需要聚合，请选择histogram。
- 否则，如果您了解要观察的值的范围和分布，请选择histogram。 如果您需要准确的分位数，无论值的范围和分布如何，请选择summary。

