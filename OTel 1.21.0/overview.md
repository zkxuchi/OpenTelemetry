# 概述

<details>
<summary>目录</summary>

<!-- toc -->

- [概述](#概述)
  - [OTel客户端架构](#otel客户端架构)
    - [API](#api)
    - [SDK](#sdk)
    - [语义规范](#语义规范)
    - [Contrib程序包](#contrib程序包)
    - [版本控制与稳定性](#版本控制与稳定性)
  - [调用链信号](#调用链信号)
    - [调用链](#调用链)
    - [Spans](#spans)
    - [SpanContext](#spancontext)
    - [span间的链接](#span间的链接)
  - [指标信号](#指标信号)
    - [Recording raw measurements](#recording-raw-measurements)
      - [Measure](#measure)
      - [Measurement](#measurement)
    - [Recording metrics with predefined aggregation](#recording-metrics-with-predefined-aggregation)
    - [Metrics data model and SDK](#metrics-data-model-and-sdk)
  - [Log Signal](#log-signal)
    - [Data model](#data-model)
  - [Baggage Signal](#baggage-signal)
  - [Resources](#resources)
  - [Context Propagation](#context-propagation)
  - [Propagators](#propagators)
  - [Collector](#collector)
  - [Instrumentation Libraries](#instrumentation-libraries)

<!-- tocstop -->

</details>

本文档提供了OpenTelemetry（OTel）项目的概述，并定义了重要的基本术语。

其他术语定义可以在[术语表](glossary.md)中找到。

## OTel客户端架构

![Cross cutting concerns](img/architecture.svg)

架构最上层, OTel客户端由[**信号（signals）**](glossary.md#信号)组成。
**信号**是可观测性的一种特定数据结构，如：tracing、metrics、baggage是三类信号。
**信号**共用一个子系统**context propagation**，但每种信号独立发挥作用。

每种**信号**都是软件描述自身的一种方式。
各类代码库（如：web框架、数据库客户端）都需要通过各种**信号**来描述自身。
OTel的**插桩（instrumentation）**代码可插入各代码库的源码中（该过程称之为**插码**），从而使OTel成为**横切关注点（[cross-cutting concern](https://en.wikipedia.org/wiki/Cross-cutting_concern)）**。
由于**横切关注点**本质上违反了SOC（分离关注点separation of concerns）设计原则，因此使用**横切**（cross-cutting） APIs进行**插码**时需要额外谨慎，以避免源码库产生问题。

OTel客户端在设计上，将每种信号中必须作为**横切关注点**插码的部分与可独立管理的部分拆分，同时作为一个可扩展框架。
因此，每种信号由4类包组成：API、SDK、语义规范（Semantic Conventions）、Contrib。

### API

API包由用于插码的**横切公共接口**（cross-cutting public interfaces）组成。OTel客户端中，任何导入第三方库或应用代码的部分，都被认为是API的一部分。

### SDK

SDK是OTel项目提供的API实现。在一个应用程序中，SDK由**应用所有者**（[application owner](glossary.md#应用所有者)）安装与管理。
需要注意，SDK包含了额外的公共接口（public interfaces），这些公共接口并不是API的一部分，因为它们不是**横切关注点**（cross-cutting concerns）。这些公共接口被定义为**构造函数**（[constructors](glossary.md#构造函数)）和**插件接口**（[plugin interfaces](glossary.md#插件接口)）。
**应用所有者**使用SDK构造函数；**插件作者**（[plugin authors](glossary.md#插件作者)）使用SDK插件接口。
插桩作者（[Instrumentation authors](glossary.md#插桩作者)）**不能**直接引用任何类型的SDK包，只能引用API。

### 语义规范

**语义规范**（Semantic Conventions）定义了键和值（key-value），以描述应用程序广泛使用的概念、协议和操作。
语义规范位于独立的仓库：https://github.com/zkxuchi/OpenTelemetry/tree/main/Semantic%20Conventions

OTel的collector和客户端lib库都应该将语义规范中的**键**与其**枚举值**自动生成为常量。
在语义规范版本稳定前，规范键值对**不能**写入程序中，而必须使用**YAML**文件作为配置来源。
每种语言的OTel实现（SDK）都应该提供该语言相应的[**代码生成器**](https://github.com/zkxuchi/OpenTelemetry/tree/main/Semantic%20Conventions#代码生成器)。

此外，语义规范中的[**保留属性**](semantic-conventions.md#保留属性)不能被使用。

### Contrib程序包

OTel项目也会维护与一些常见OSS项目的集成，这些OSS项目对web服务的可观测性有重要作用。API集成示例包含：web框架的插码，数据库客户端，以及消息队列等。SDK集成示例包含将OTel信号输出至常见分析工具或OTel存储系统的各类插件。

OTel规范要求提供OTLP exporters、TraceContext Propagators等插件，并作为SDK的一部分。
插件以及插桩程序包可选，且与SDK分离，作为**Contrib**程序包。
**API Contrib**是指仅依赖API的程序包；**SDK Contrib**是指同时依赖SDK的程序包。

术语**Contrib**特指OTel项目维护的插件与插桩的合集，不涉及第三方插件。

### 版本控制与稳定性

OTel项目重视稳定性及向后兼容性，详情可参阅[版本控制与稳定性](versioning-and-stability.md)。

## 调用链信号

调用链追踪信号（Tracing Signal）主体是**分布式调用链**（distributed trace）。
每个分布式调用链由一组事件（events）组成，每个事件由单次逻辑操作生成，并跨应用的各个组件合并而成。其所包含的事件横跨进程（process）、网络（network）及各安全域边界（security boundaries）。
一个分布式调用链可能由一次页面按钮点击开始，涵盖为处理此点击产生的请求链，下游服务之间的调用。

### 调用链

OTel中的调用链（Traces）通过其**spans**隐式定义。一个调用链可以认为是一组span的有向无环图（DAG: directed acyclic graph），且span之间的边（edges）被定义为父/子（parent/child）关系。

例如，下图是6个span组成的调用链：

```
单个调用链中span之间的因果关系

        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C is a `child` of Span A)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F]
```

调用链的可视化通常使用带时间轴的瀑布图表示：

```
单个调用链中span之间的时序关系

––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time

 [Span A···················································]
   [Span B··········································]
      [Span D······································]
    [Span C····················································]
         [Span E·······]        [Span F··]
```

### Spans

Span代表对一个事务的一次操作，封装了以下状态：

- 操作名称（operation name）
- 起止时间
- [属性](./common/README.md#属性)（Attributes）：一组键值对。
- 事件（Events）：元组（tuple），包含时间戳、名称与属性，名称必须是字符串。
- 父span标识符
- [链接](#span间的链接)（Links）：具有因果关系的其他span。
- SpanContext：索引span所需信息

### SpanContext

SpanContext是在调用链中，标识一个span所需的信息，包含调用链识符，以及从父span传播（propagate）至子span的标示符。SpanContext必须可横跨进程边界传播到子span。

- **TraceId** 调用链标示符，16字节，随机生成，全局唯一。用以跨进程组合调用链的所有span。
- **SpanId** span标示符，8字节，随机生成，全局唯一。 并作为父span id传递至子span。
- **TraceFlags** 调用链选项，1字节（bitmap）。
  - Sampling bit -  该bit位标识该调用链是否被采样 (mask `0x1`)。
- **Tracestate** 键值对，用于携带特定上下文信息。**Tracestate** 允许各厂商传播附加信息，并可与其legacy Id格式相互操作。详情参阅[w3c规范](https://w3c.github.io/trace-context/#tracestate-field)。

### span间的链接

每个span可链接多个具有因果关系的其他span。**链接**（Links）可指向相同或不同调用链中的span。**链接**可表示批处理操作，当一个span由多个初始化中的span发起，则每个被链接的span表示该批处理中的单个传入项目。

**链接**还可用于声明初始调用链与后续调用链直接的关系，如：当一个调用链进入服务的信任边界（trusted boundaries），该服务的策略要求生成新的调用链，而非信任原调用链上下文（context）。
此外，链接的调用链也可用于标识高并发下，一个请求发起的长时间运行的异步数据操作。

当使用分散scatter/聚集gather（分支fork/合并join）模式时，根操作会启动多个下游处理操作，每个下游操作生成一个span的同时，又并会将所有操作结果聚合回最后一个Span中，这个span被链接到这些被聚合操作的span上，且所有spans属于同一个调用链。这时**链接**和span的父字段（parent field）作用类似，但是该场景下父字段不适用，因为父字段用于标识只有一个父span的场景，在分散/集合与批处理场景下，最后一个span会有多个父span。

## 指标信号

OTel支持记录原始的测量值或者指标，
Metric Signal
OpenTelemetry allows to record raw measurements or metrics with predefined
aggregation and a [set of attributes](./common/README.md#attribute).

Recording raw measurements using OpenTelemetry API allows to defer to end-user
the decision on what aggregation algorithm should be applied for this metric as
well as defining attributes (dimensions). It will be used in client libraries like
gRPC to record raw measurements "server_latency" or "received_bytes". So end
user will decide what type of aggregated values should be collected out of these
raw measurements. It may be simple average or elaborate histogram calculation.

Recording of metrics with the pre-defined aggregation using OpenTelemetry API is
not less important. It allows to collect values like cpu and memory usage, or
simple metrics like "queue length".

### Recording raw measurements

The main classes used to record raw measurements are `Measure` and
`Measurement`. List of `Measurement`s alongside the additional context can be
recorded using OpenTelemetry API. So user may define to aggregate those
`Measurement`s and use the context passed alongside to define additional
attributes of the resulting metric.

#### Measure

`Measure` describes the type of the individual values recorded by a library. It
defines a contract between the library exposing the measurements and an
application that will aggregate those individual measurements into a `Metric`.
`Measure` is identified by name, description and a unit of values.

#### Measurement

`Measurement` describes a single value to be collected for a `Measure`.
`Measurement` is an empty interface in API surface. This interface is defined in
SDK.

### Recording metrics with predefined aggregation

The base class for all types of pre-aggregated metrics is called `Metric`. It
defines basic metric properties like a name and attributes. Classes inheriting from
the `Metric` define their aggregation type as well as a structure of individual
measurements or Points. API defines the following types of pre-aggregated
metrics:

- Counter metric to report instantaneous measurement. Counter values can go
  up or stay the same, but can never go down. Counter values cannot be
  negative. There are two types of counter metric values - `double` and `long`.
- Gauge metric to report instantaneous measurement of a numeric value. Gauges can
  go both up and down. The gauges values can be negative. There are two types of
  gauge metric values - `double` and `long`.

API allows to construct the `Metric` of a chosen type. SDK defines the way to
query the current value of a `Metric` to be exported.

Every type of a `Metric` has it's API to record values to be aggregated. API
supports both - push and pull model of setting the `Metric` value.

### Metrics data model and SDK

Metrics data model is [specified here](metrics/data-model.md) and is based on
[metrics.proto](https://github.com/open-telemetry/opentelemetry-proto/blob/master/opentelemetry/proto/metrics/v1/metrics.proto).
This data model defines three semantics: An Event model used by the API, an
in-flight data model used by the SDK and OTLP, and a TimeSeries model which
denotes how exporters should interpret the in-flight model.

Different exporters have different capabilities (e.g. which data types are
supported) and different constraints (e.g. which characters are allowed in attribute
keys). Metrics is intended to be a superset of what's possible, not a lowest
common denominator that's supported everywhere. All exporters consume data from
Metrics Data Model via a Metric Producer interface defined in OpenTelemetry SDK.

Because of this, Metrics puts minimal constraints on the data (e.g. which
characters are allowed in keys), and code dealing with Metrics should avoid
validation and sanitization of the Metrics data. Instead, pass the data to the
backend, rely on the backend to perform validation, and pass back any errors
from the backend.

See [Metrics Data Model Specification](metrics/data-model.md) for more
information.

## Log Signal

### Data model

[Log Data Model](logs/data-model.md) defines how logs and events are understood by
OpenTelemetry.

## Baggage Signal

In addition to trace propagation, OpenTelemetry provides a simple mechanism for propagating
name/value pairs, called `Baggage`. `Baggage` is intended for
indexing observability events in one service with attributes provided by a prior service in
the same transaction. This helps to establish a causal relationship between these events.

While `Baggage` can be used to prototype other cross-cutting concerns, this mechanism is primarily intended
to convey values for the OpenTelemetry observability systems.

These values can be consumed from `Baggage` and used as additional attributes for metrics,
or additional context for logs and traces. Some examples:

- a web service can benefit from including context around what service has sent the request
- a SaaS provider can include context about the API user or token that is responsible for that request
- determining that a particular browser version is associated with a failure in an image processing service

For backward compatibility with OpenTracing, Baggage is propagated as `Baggage` when
using the OpenTracing bridge. New concerns with different criteria should consider creating a new
cross-cutting concern to cover their use-case; they may benefit from the W3C encoding format but
use a new HTTP header to convey data throughout a distributed trace.

## Resources

`Resource` captures information about the entity for which telemetry is
recorded. For example, metrics exposed by a Kubernetes container can be linked
to a resource that specifies the cluster, namespace, pod, and container name.

`Resource` may capture an entire hierarchy of entity identification. It may
describe the host in the cloud and specific container or an application running
in the process.

Note, that some of the process identification information can be associated with
telemetry automatically by the OpenTelemetry SDK.

## Context Propagation

All of OpenTelemetry cross-cutting concerns, such as traces and metrics,
share an underlying `Context` mechanism for storing state and
accessing data across the lifespan of a distributed transaction.

See the [Context](context/README.md)

## Propagators

OpenTelemetry uses `Propagators` to serialize and deserialize cross-cutting concern values
such as `Span`s (usually only the `SpanContext` portion) and `Baggage`. Different `Propagator` types define the restrictions
imposed by a specific transport and bound to a data type.

The Propagators API currently defines one `Propagator` type:

- `TextMapPropagator` injects values into and extracts values from carriers as text.

## Collector

The OpenTelemetry collector is a set of components that can collect traces,
metrics and eventually other telemetry data (e.g. logs) from processes
instrumented by OpenTelemetry or other monitoring/tracing libraries (Jaeger,
Prometheus, etc.), do aggregation and smart sampling, and export traces and
metrics to one or more monitoring/tracing backends. The collector will allow to
enrich and transform collected telemetry (e.g. add additional attributes or
scrub personal information).

The OpenTelemetry collector has two primary modes of operation: Agent (a daemon
running locally with the application) and Collector (a standalone running
service).

Read more at OpenTelemetry Service [Long-term
Vision](https://github.com/open-telemetry/opentelemetry-collector/blob/master/docs/vision.md).

## Instrumentation Libraries

See [Instrumentation Library](glossary.md#instrumentation-library)

The inspiration of the project is to make every library and application
observable out of the box by having them call OpenTelemetry API directly. However,
many libraries will not have such integration, and as such there is a need for
a separate library which would inject such calls, using mechanisms such as
wrapping interfaces, subscribing to library-specific callbacks, or translating
existing telemetry into the OpenTelemetry model.

A library that enables OpenTelemetry observability for another library is called
an [Instrumentation Library](glossary.md#instrumentation-library).

An instrumentation library should be named to follow any naming conventions of
the instrumented library (e.g. 'middleware' for a web framework).

If there is no established name, the recommendation is to prefix packages
with "opentelemetry-instrumentation", followed by the instrumented library
name itself. Examples include:

* opentelemetry-instrumentation-flask (Python)
* @opentelemetry/instrumentation-grpc (Javascript)