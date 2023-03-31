# Client

- [Client](#client)
	- [一、Desc \& Metric](#一desc--metric)
	- [二、Basic Metric](#二basic-metric)
		- [Collector](#collector)
		- [Basic Collector](#basic-collector)
		- [Collector Vector](#collector-vector)
	- [三、Registry](#三registry)
		- [数据结构](#数据结构)
		- [Register](#register)
		- [Gather](#gather)
	- [四、PromHandler](#四promhandler)


## 一、Desc & Metric

> Desc是Prometheus指标中使用的描述符，可以看作是Metric的不可变元数据，包含了Metric的名称、帮助文本、标签以及类型等信息
> Desc 在创建 Metric 时被初始化，并在此后一直保持不变。它是 Metric 的唯一标识符，用于区分不同的指标。因此，我们可以通过 Desc 来查询和操作 Metric。
> 需要注意的是，Desc 只是 Metric 的元数据，它并不包含任何数据点。实际上，Metric 的值是由采集器（Collector）生成的，并存储在时间序列（Time Series）中。每个 Time Series 都包含一个具体的值和一组标签，这些标签可以对 Metric 进行分类和筛选。
> 总之，Desc 是 Prometheus Metric 不可或缺的组成部分，它提供了 Metric 的基本信息和结构定义，为采集器和监控系统提供了统一的接口和语义
> fqName: fully-qualified name

用户通过设置 `NewDesc` 创建 Desc, `id`、`dimHash`、`err`则由prometheus生成，其中
- id: 是 `fqName` 与 `ConstLabels Value` 的hash, 作为 `Desc` 的唯一标识符, 这意味着即便 `fqName` 相同，但 `ConstLabel Values` 不同的指标id不相同
- dimHash: 是 `ConstLabels Name` 、`VariableLabels Name` 及 `Help` 的hash, 相同 `fqName` 的 Desc 必须有相同的 dimHash，即只要 `fqName` 相同，Label的组成必然是一样的
- err: `NewDesc` 并不会返回错误，而是保存错误信息，在Register时进行处理

```go
func NewDesc(fqName, help string, variableLabels []string, constLabels Labels) *Desc

type Desc struct {
	// fqName has been built from Namespace, Subsystem, and Name.
	fqName string
	// help provides some helpful information about this metric.
	help string
	// constLabelPairs contains precalculated DTO label pairs based on
	// the constant labels.
	constLabelPairs []*dto.LabelPair
	// variableLabels contains names of labels for which the metric
	// maintains variable values.
	variableLabels []string
	// id is a hash of the values of the ConstLabels and fqName. This
	// must be unique among all registered descriptors and can therefore be
	// used as an identifier of the descriptor.
	id uint64
	// dimHash is a hash of the label names (preset and variable) and the
	// Help string. Each Desc with the same fqName must have the same
	// dimHash.
	dimHash uint64
	// err is an error that occurred during construction. It is reported on
	// registration time.
	err error
}
```

Metric 是一个接口，采集时调用 `Write` 将编码好的 Metric 数据写入到 `*dto.Metric` 中

```go
type Metric interface {
	// Desc returns the descriptor for the Metric. This method idempotently
	// returns the same descriptor throughout the lifetime of the
	// Metric. The returned descriptor is immutable by contract. A Metric
	// unable to describe itself must return an invalid descriptor (created
	// with NewInvalidDesc).
	Desc() *Desc
	// Write encodes the Metric into a "Metric" Protocol Buffer data
	// transmission object.
	//
	// Metric implementations must observe concurrency safety as reads of
	// this metric may occur at any time, and any blocking occurs at the
	// expense of total performance of rendering all registered
	// metrics. Ideally, Metric implementations should support concurrent
	// readers.
	//
	// While populating dto.Metric, it is the responsibility of the
	// implementation to ensure validity of the Metric protobuf (like valid
	// UTF-8 strings or syntactically valid metric and label names). It is
	// recommended to sort labels lexicographically. Callers of Write should
	// still make sure of sorting if they depend on it.
	Write(*dto.Metric) error
	// TODO(beorn7): The original rationale of passing in a pre-allocated
	// dto.Metric protobuf to save allocations has disappeared. The
	// signature of this method should be changed to "Write() (*dto.Metric,
	// error)".
}
```

## 二、Basic Metric

### Collector

`Collector` 接口是 prometheus 客户端库提供的一个接口，用于描述如何采集指标数据，并将其暴露给 Prometheus 服务器。该接口定义了以下方法：

Describe(chan<- *Desc)：将该 collector 添加的指标的 metadata（如名称、帮助信息、标签等）发送到 chan<- *Desc 中
Collect(chan<- Metric)：将该 collector 表示的指标数据发送到 chan<- Metric 中


```go
// prometheus/collector.go #27

// Collector is the interface implemented by anything that can be used by
// Prometheus to collect metrics. A Collector has to be registered for
// collection. See Registerer.Register.
//
// The stock metrics provided by this package (Gauge, Counter, Summary,
// Histogram, Untyped) are also Collectors (which only ever collect one metric,
// namely itself). An implementer of Collector may, however, collect multiple
// metrics in a coordinated fashion and/or create metrics on the fly. Examples
// for collectors already implemented in this library are the metric vectors
// (i.e. collection of multiple instances of the same Metric but with different
// label values) like GaugeVec or SummaryVec, and the ExpvarCollector.
type Collector interface {
	// Describe sends the super-set of all possible descriptors of metrics
	// collected by this Collector to the provided channel and returns once
	// the last descriptor has been sent. The sent descriptors fulfill the
	// consistency and uniqueness requirements described in the Desc
	// documentation.
	//
	// It is valid if one and the same Collector sends duplicate
	// descriptors. Those duplicates are simply ignored. However, two
	// different Collectors must not send duplicate descriptors.
	//
	// Sending no descriptor at all marks the Collector as “unchecked”,
	// i.e. no checks will be performed at registration time, and the
	// Collector may yield any Metric it sees fit in its Collect method.
	//
	// This method idempotently sends the same descriptors throughout the
	// lifetime of the Collector. It may be called concurrently and
	// therefore must be implemented in a concurrency safe way.
	//
	// If a Collector encounters an error while executing this method, it
	// must send an invalid descriptor (created with NewInvalidDesc) to
	// signal the error to the registry.
	Describe(chan<- *Desc)
	// Collect is called by the Prometheus registry when collecting
	// metrics. The implementation sends each collected metric via the
	// provided channel and returns once the last metric has been sent. The
	// descriptor of each sent metric is one of those returned by Describe
	// (unless the Collector is unchecked, see above). Returned metrics that
	// share the same descriptor must differ in their variable label
	// values.
	//
	// This method may be called concurrently and must therefore be
	// implemented in a concurrency safe way. Blocking occurs at the expense
	// of total performance of rendering all registered metrics. Ideally,
	// Collector implementations support concurrent readers.
	Collect(chan<- Metric)
}
```

### Basic Collector

prometheus 提供了 `Counter`, `Guage`, `Histogram`, `Summary` 四个基本指标，他们都实现了 Collector 接口, 主要由 Desc 和 观测值 组成，包含以下特征: 
- 仅对外提供一套接口方法，通过封装私有变量的方式来控制对观测值的操作
- `Histogram` 和 `Summary` 是对观测值的进一步统计，相比于只记录单一观测值的 `Counter`, `Guage` 而言更加复杂, 实质是一种复合指标

如在 gauge 中, desc 是描述符，valBits 保存了观测值比特值，selfCollector 是一个匿名结构体，提供了 Collector 的通用实现，gauge 本身只是实现了 Metric 接口，由于嵌入了 selfCollector 因此也继承了 Collector 的实现

```go
type gauge struct {
	valBits uint64
    desc     *Desc
	selfCollector
	...
}
```
prometheus client 中的 Metric 都是线程安全的，底层通过 atomic 来保证并发访问的正确性
- Go 中并没有提供对 float64 的原子操作函数，因此Client中使用了自旋锁了操作 uint64 变量，并通过 math 库来进行 float64 与 uint64 之间的转换

```go
func (g *gauge) Add(val float64) {
	for {
		oldBits := atomic.LoadUint64(&g.valBits)
		newBits := math.Float64bits(math.Float64frombits(oldBits) + val)
		if atomic.CompareAndSwapUint64(&g.valBits, oldBits, newBits) {
			return
		}
	}
}
```

### Collector Vector

Vector Collector 维护了一个 Metric 集合，这些 Metric 共享相同的 Desc, 通过不同的 VariableLabel Values 进行区分， 可使用 `WithLabelValues`， 来插入一个新的 Metric 或对一个已有的 Metric 进行更新

## 三、Registry


### 数据结构

Registry是Registerer接口的一个实现，负责注册Prometheus collectors, 收集metrics来构造MetricFamilies进行暴露

> 一个 Collector 中可以存在多个不同的 Desc, 将这些 Desc 的 ID 进行异或来得到 CollectorID

Registry中包含三个主要的map：
- collectorsByID: 保存 CollectID 到 Collector 的映射
- descIDs：保存所有 DescID 的集合, 用来判断 Desc 是否重复
- dimHashesByName：保存 fqName 到 dimHash 的映射，判断 Desc 是否合法，即相同 fqName 的 Desc, dimHash 必须相同

```go
type Registry struct {
	collectorsByID        map[uint64]Collector 
	descIDs               map[uint64]struct{}
	dimHashesByName       map[string]uint64
	...
}
```

### Register

`Register` 将 Collector 注册到 metric family 中, 其中会调用 `Collector` 的 `Describe` 方法获取 Desc 来验证 Collector 的合法性以及是否重复

`Collector` 也可以在 `Describe` 中返回空值，这样 `Register` 就会跳过验证过程，这样的 Collecotr 被称为 Unchecked Collector, 它们的注册总是会成功并直接加入到一个Unchecked Collector列表中，这要求开发者保证每个 `Collector` 的唯一性，以避免出现抓取错误

>Register 中的验证是以 Collector 为粒度的，因此会允许 Desc 在同一个 Collector 内重复，而在 Collector 之间这样则会返回错误

Collector 验证过程:
1. 集中处理 Desc 构建时的错误
2. 判断 Desc 是否与之前注册的 Collector 的 Desc 重复
3. 收集 Collector 中非重复的 Desc, 将ID相异或得到 CollectorID
4. 判断相同 fqName 的 Desc, dimHash(label组成)是否相同

验证过程结束后，CollectorID 也构造完成，因此在 Register最后部分，就会对 Collector 进行重复判断，并更新 `collectorsByID`，`descIDs` 与 `dimHashesByName`

```go
// prometheus/promhttp/registry.go #291
// Is the descriptor valid at all?
if desc.err != nil {
	return fmt.Errorf("descriptor %s is invalid: %w", desc, desc.err)
}

// Is the descID unique?
// (In other words: Is the fqName + constLabel combination unique?)
if _, exists := r.descIDs[desc.id]; exists {
	duplicateDescErr = fmt.Errorf("descriptor %s already exists with the same fully-qualified name and const label values", desc)
}
// If it is not a duplicate desc in this collector, XOR it to
// the collectorID.  (We allow duplicate descs within the same
// collector, but their existence must be a no-op.)
if _, exists := newDescIDs[desc.id]; !exists {
	newDescIDs[desc.id] = struct{}{}
	collectorID ^= desc.id
}

// Are all the label names and the help string consistent with
// previous descriptors of the same name?
// First check existing descriptors...
if dimHash, exists := r.dimHashesByName[desc.fqName]; exists {
	if dimHash != desc.dimHash {
		return fmt.Errorf("a previously registered descriptor with the same fully-qualified name as %s has different label names or a different help string", desc)
	}
} else {
	// ...then check the new descriptors already seen.
	if dimHash, exists := newDimHashesByName[desc.fqName]; exists {
		if dimHash != desc.dimHash {
			return fmt.Errorf("descriptors reported by collector have inconsistent label names or help strings for the same fully-qualified name, offender is %s", desc)
		}
	} else {
		newDimHashesByName[desc.fqName] = desc.dimHash
	}
}
```

### Gather

`Gather` 方法会调用所有已注册的 Collector 的 Collect 方法，并将收集到的指标聚合成一个字典序且无重复的MetricFamily protobuf切片

`Gather` 会尽可能地收集指标，即便有些 Collect 返回的是空值

每个 Metric 都需要经过从 Collector 采集，然后再经过 `processMetric` 处理为可编码的格式并保存到 `[]*dto.MetricFamily` 中, 考虑到会同时存在多个Collector, 每个Collector采集 Metric 的速度都不同，简单地，可以为每个 Collector 创建一个协程来处理，但是这样并不一定高效，因为协程的创建与回收都会消耗一定的资源

Gather 内部会进行一定的协程调度，尽可能用最少的协程来满足采集需求，而为实现这一目的，主要经过以下操作:
1. 创建好 Metric 管道，计算当前所有的Collector数量，设置为可开启的协程上限
2. 将 Collector 存放到 channel 中，并创建好 waitGroup 来进行并发控制
3. 启动collectWorker，其中会不断地从管道取出 collector 并执行 Collect 方法
4. 主函数中则不断地从 Metric 管道中读取 Metric 进行处理，而当metric消费速度超过生产速度时，则会根据协程上限与剩余 Collector 的数量，来决定是否增加新的 collectWorker

```go
collectWorker := func() {
	for {
		select {
		case collector := <-checkedCollectors:
			collector.Collect(checkedMetricChan)
		case collector := <-uncheckedCollectors:
			collector.Collect(uncheckedMetricChan)
		default:
			return
		}
		wg.Done()
	}
}
```

## 四、PromHandler

四、PromHandler 主要逻辑: 
1. 调用 Registry 的 `Gather` 方法，收集所有 Collector 的指标
2. 随后构造 Metric 编码器，能够编码 Metric，同时传入 Response 用来写入编码后的 Metric
3. 调用编码器逐个对 Metric 进行处理并写入 Response 中

```go
// prometheus/promhttp/http.go

// #135 调用Gather()方法收集metric
mfs, done, err := reg.Gather()
defer done()
...

// #166 enc负责将metric编码为字符串并写入到返回值中
w := io.Writer(rsp)
...
enc := expfmt.NewEncoder(w, contentType)

...

// #204 逐个编码 metric并向rsp写入
for _, mf := range mfs {
	if handleError(enc.Encode(mf)) {
		return
	}
}
...
```