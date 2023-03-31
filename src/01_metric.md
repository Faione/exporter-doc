# Metric

- [Metric](#metric)
  - [一、格式](#一格式)
  - [二、类型](#二类型)
    - [基本类型](#基本类型)
    - [Vec类型](#vec类型)
    - [Untyped类型](#untyped类型)

## 一、格式

metric格式[^1]一般为 `<name> {<label>} <value>`，主要由三部分组成
- name: 名称，通常用来唯一标识某一指标，并能一定程度上表达指标的含义，如`http_requests_total`表明此指标为http请求的总量
- labels: 标签，是一组 key="value" 对，为同一个指标提供多维度的信息，可以选择其中的一个或多个维度来对指标进行筛选
- value：值，通常为整数或浮点数，exporter在抓取数据的时候对value进行更新，而name和lalels通常是静态的文本描述信息

```
my_metric{label="a"} 1
```

## 二、类型

### 基本类型

Metric有四种类型[^2], Counter、Gauge、Histogram和Summary，用于不同数值特征的数据[^3]

**Counter**

基础类型，表示单个单调递增的单个数值类型，对应的value只能增加或是通过重启程序来清0，如请求的次数，完成任务的数量，错误数量等

![](2023-03-16-17-38-15.png)

**Guage**

基础类型，表示单个数值类型，但能够任意的增加或减少，如当前的温度，内存使用，并发请求的数量等

![](2023-03-16-17-38-25.png)

**Histogram**: 

复合类型，Histogram对数据进行采样，并在可配置的buckets中进行计数，同时还提供所有采样值的总和。Histogram并不保存单个数值类型指标的所有值，而对单个数值类型指标的值进一步统计，因此也是一个复合指标，基于设置的base_name, 衍生出了三个子指标[^4]:
- `base_name_bucket{le="<upper inclusive bound>"}`: Counter类型, lower or equal, 保存小于或等于bucket上界的采样值的数量
- `base_name_totoal_count`: Counter类型, 保存采样数量(次数)，也即`base_name_bucket{le="inf"}` 的值
- `base_name_totoal_sum`: Counter类型, 保存采样值之和

![](2023-03-16-17-38-41.png)

**Summary**

复合类型, Summay与Histogram类似，同样是一个复合指标，衍生了三个子指标
- `base_name{quantile="0.5"}`: 保存Summary采样周期内，50分位采样值
- `base_name_totoal_count`: Counter类型，保存采样次数
- `base_name_totoal_sum`: Counter类型，保存采样值之和

默认情况下， Summary会在内存中创建10个quantile objects[^5]，所有的observation(采样值)会送往每个quantile objects, 而每个quantile object会在上一个object开始1min之后才开始追踪observation，这意味者最早的quantile object会保存最近10min的采样数据，随后是9，8...2,1min，而在此之后，最久的object会被移除，然后一个新的quantile object会开始追踪。显然，如果在采样周期内(10min)，没有任何数据被采样到，则`base_name{quantile="x"}`的结果将会是`NaN`

**Histogram Or Summary**

由于Summary会在采集时进行计算，应此会往往占用更多采集端的资源，且由于quantile百分比已经计算好，因此难以与其他节点数据进行计算;与之相反，通过histogram计算quantile数据依赖函数`histogram_quantile()`, 因此在query处会消耗更多资源，造成响应不及时，而好处在于采集端资源消耗低，易于与其他节点的数据聚合[^6]

### Vec类型

每种基本类型都有对应的Vec类型, Vec类型保存同一基本类型的指标向量，这些指标类型相同，基础信息如`metric_name`, `label_key`, `description`等都相同，只是在个别 `label_value` 上存在差异。Vec类型提供了非常便捷的方式去定义一组相似基本指标，当设置不同的`label_value`时能够动态地创建新指标

### Untyped类型

当抓取的数据数值特征不明显时，可使用 Untyped 

[^1]: [metric_format](https://prometheus.io/docs/instrumenting/writing_exporters/)

[^2]: [metric_types](https://prometheus.io/docs/concepts/metric_types/)

[^3]: [understanding_metric_types](https://prometheus.io/docs/tutorials/understanding_metric_types/)

[^4]: [how_histogram_work](https://www.robustperception.io/how-does-a-prometheus-histogram-work/)

[^5]: [how_summary_work](https://www.robustperception.io/how-does-a-prometheus-summary-work/)

[^6]: [histogram_or_summary](https://bryce.fisher-fleig.org/prometheus-histograms/)