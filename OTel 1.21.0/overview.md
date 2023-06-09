# 概述

<details>
<summary>目录</summary>

<!-- toc -->

- [概述](#概述)
  - [OTel客户端架构](#otel客户端架构)
    - [API](#api)
    - [SDK](#sdk)
    - [语义规范](#语义规范)
    - [贡献包](#贡献包)
    - [版本控制与稳定性](#版本控制与稳定性)
  - [调用链信号](#调用链信号)
    - [调用链](#调用链)
    - [Spans](#spans)
    - [SpanContext](#spancontext)
    - [span间的链接](#span间的链接)
  - [指标信号](#指标信号)
    - [记录原始测量值](#记录原始测量值)
      - [Measure类](#measure类)
      - [Measurement类](#measurement类)
    - [基于预定义聚合类型记录指标](#基于预定义聚合类型记录指标)
    - [指标数据模型与SDK](#指标数据模型与sdk)
  - [日志信号](#日志信号)
    - [数据模型](#数据模型)
  - [Baggage信号](#baggage信号)
  - [Resource](#resource)
  - [上下文传播](#上下文传播)
  - [Propagators](#propagators)
  - [Collector](#collector)
  - [插桩库](#插桩库)

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
需要注意，SDK包含了额外的公共接口（public interfaces），这些公共接口并不是API的一部分，因为它们不是**横切关注点**（cross-cutting concerns）。这些公共接口被定义为**构造器**（[constructors](glossary.md#构造器)）和**插件接口**（[plugin interfaces](glossary.md#插件接口)）。
**应用所有者**使用SDK构造器；**插件作者**（[plugin authors](glossary.md#插件作者)）使用SDK插件接口。
插桩作者（[Instrumentation authors](glossary.md#插桩作者)）**不能**直接引用任何类型的SDK包，只能引用API。

### 语义规范

**语义规范**（Semantic Conventions）定义了键和值（key-value），以描述应用程序广泛使用的概念、协议和操作。
语义规范位于独立的仓库：https://github.com/zkxuchi/OpenTelemetry/tree/main/Semantic%20Conventions

OTel的collector和客户端库都应该将语义规范中的**键**与其**枚举值**自动生成为常量。
在语义规范版本稳定前，规范键值对**不能**写入程序中，而必须使用**YAML**文件作为配置来源。
每种语言的OTel实现（SDK）都应该提供该语言相应的[**代码生成器**](https://github.com/zkxuchi/OpenTelemetry/tree/main/Semantic%20Conventions#代码生成器)。

此外，语义规范中的[**保留属性**](semantic-conventions.md#保留属性)不能被使用。

### 贡献包

OTel项目也会维护与一些常见OSS（Open Source Software）项目的集成，这些OSS项目通常对可观测性有重要作用。API集成包含：web框架的插码，数据库客户端，以及消息队列等。SDK集成包含将OTel信号输出至常见分析工具或OTel存储系统的各类插件。

OTel规范要求提供OTLP exporters、TraceContext Propagators等插件，并作为SDK的一部分。
插件以及插桩程序包可选，且与SDK分离，作为**贡献包**（Contrib package）。
**API贡献包**是指仅依赖API的程序包；**SDK贡献包**是指同时依赖SDK的程序包。

术语**贡献包**特指OTel项目维护的插件与插桩的合集，不涉及第三方插件。

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
- 事件（Events）：0或多个元组（tuple），包含时间戳、名称与属性，名称必须是字符串。
- 父span标识符
- [链接](#span间的链接)（Links）：0或多个具有因果关系的其他span。
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

指标型号（Metric Signal），OTel支持记录**原始测量值**（raw measurements）与**指标**（metrics），并预定义了这些原始测量值与指标的聚合类型、[属性](./common/README.md#属性)。 

通过OTel API记录原始测量值时，用户可自定义指标的聚合算法以及属性（维度）。客户端库通常使用该方式记录原始测量值，如：gPRC的“服务端延迟（server_latency）”、“接收字节数（received_bytes）”等。随后用户通过原始测量值生成所需的聚合值类型，如：平均值、直方图等。

OTel API基于预定义聚合类型生成指标（metrics），此方式多用于采集CPU/内存用量、队列长度（queue length）等简单指标。

### 记录原始测量值

`Measure`和`Measurement`是OTel API记录原始测量值的两个主要的类（class）。`Measurement`用于列出OTel API支持的**测量值**及其附加的**上下文**信息，随后用户可定义**测量值**的聚合类型，并根据**上下文**信息定义该指标的附件**属性**。

#### Measure类

`Measure`类用于表述库记录的**值**类型，其定义了应用程序将**测量值**聚合成**指标**的模式。`Measure`由**名称**、**描述**、值的**单位**来标识。

#### Measurement类

`Measurement`用于描述`Measure`所采集到的单个值，在API中暴露为空接口，该接口在SDK中定义。

### 基于预定义聚合类型记录指标

`Metric`是所有预聚合类型指标的基类（base class），其定义了基本的**指标**特征（properties），如：名称、属性（attributes）。从`Metric`继承的类则定义了指标的聚合类型以及单个测量值（measurements）的结构。API定义了以下预聚合指标类型：

- 计数指标（Counter metric）记录瞬时测量值。计数器的值**只增不减**（可不变），且**不能**为负数，支持`double`和`long`两种字段类型。
- 计量指标（Gauge metric）记录数字值的瞬时测量值，但是其值**可增可减**，**可为负数**，也支持`double`和`long`两种字段类型。

此外，Prometheus的指标类型是OTel指标类型的**子集**，参阅[这里](https://www.timescale.com/blog/prometheus-vs-opentelemetry-metrics-a-complete-guide/)。

通过API构建`Metric`时，类型可选，SDK中则预定义了查询`Metric`值时的类型。

每种`Metric`类型都有各自的API记录所需聚合的值，且API支持推（push）、拉（pull）两种模式设置`Metric`的值。

### 指标数据模型与SDK

指标的数据模型参阅[这里](metrics/data-model.md)，模板文件[metrics.proto](https://github.com/open-telemetry/opentelemetry-proto/blob/master/opentelemetry/proto/metrics/v1/metrics.proto)。
其中定义了3种语义（Semantics）：API使用的**事件**（Event）模型；SDK和OTLP使用的传输中（in-flight）数据模型；以及一个时序（timeseries）模型用于说明导出器（exporter）应该如何解析传输中（in-flight）数据模型。

不同的导出器具有不同的功能与约束（constraints），如：可支持不同的数据类型，约束属性键值的可用字符等。OTel指标旨在具有最大的类型兼容性。所有导出器都通过OTel SDK中定义的指标生产者接口（Metric Producer interface）从指标数据模型中消费数据。

因此，指标对数据的约束最小（如：属性键值的可用字符），处理指标的代码应**避免**数据的验证和清洗，而应将数据传至后端（backend），由后端进行验证，并返回错误。

参阅[指标数据模型规范](metrics/data-model.md) 获取更多信息。

## 日志信号

### 数据模型

[日志数据模型](logs/data-model.md)定义了OTel如何解析日志（logs）与事件（events）。

## Baggage信号

除了调用链的传播机制外，OTel还提供一个简单机制用来传播键值对（name/value pairs）：`Baggage`。在同一个事务（transaction）中，`Baggage`通过前一个服务的属性（attributes）索引当前服务的事件，从而有助于建立事务间的因果关系。

虽然`Baggage`可作为其他横切关注点（cross-cutting concerns）的原型（prototype），但是此机制主要用于值传递。
`Baggage`消费这些值，并作为指标（metrics）的附加属性（attributes），例如：

- Web服务可以从发送请求的服务获取其上下文（context）
- SaaS服务可以在上下文中标注哪些API用户或token负责处理该请求
- 确定图像处理服务的错误与哪些浏览器版本相关

为了保证后端系统与OpenTracing的兼容性，当使用OpenTracing桥接（bridge）时，Baggage为作为`Baggage`进行传播。
不同标准下，新的关注点应该创建一个新的横切关注点以覆盖其用例（use-case），虽然一样采用W3C规范的编码格式，但是会使用新的HTTP Header以分布式调用链的方式传递数据。

## Resource

`Resource`用于采集遥测信号（Telemetry）的**实体**信息，如：某个k8s容器的指标，可关联其所属集群、命名空间、pod、容器名称等资源信息。
`Resource`可采集**实体**完整的层级结构，如：某个云中的主机，进程所运行的容器或应用等。
一些进程（process）的标示信息可通过OTel SDK自动与遥测信号（Telemetry）关联。

## 上下文传播

上下文传播（context propagation），OTel的所有横切关注点（如调用链、指标），共享一个底层的`Context`机制，该机制用于在分布式事务的生命周期中存储状态和访问数据。

参阅[上下文](context/README.md)。

## Propagators

OTel使用`Propagators`序列化及反序列化横切关注点的值，如：`Span`（通常仅`SpanContext`部分）与 `Baggage`。
`Propagator`类型定义了各传输协议的限制，并绑定至数据类型。

`Propagators` API当前只定义了一个`Propagator`类型：

- `TextMapPropagator` 以文本方式向`carriers`中注入或提取值。

## Collector

OTel collector是一套组件，用于接收调用链、指标、日志等遥测信号，不仅支持OTel插桩，也支持第三方的监控/调用链追踪库，如：Jaeger，Prometheus等。其支持对调用链与指标的聚合、智能采样，并输出至一个或多个后端系统（backends）。Collector还支持丰富（enrich）和转换（transform）采集到的遥测信号，如：附加属性、清洗个人信息等。

OTel collector支持两种运行模式：Agent（运行在应用本地的守护进程）、Collector（独立运行的服务）。

OTel Collector的[长期愿景](https://github.com/open-telemetry/opentelemetry-collector/blob/master/docs/vision.md)。

## 插桩库

该项目起源于以直接调用OTel API的方式，让每个应用程序及库都可开箱即用。但是许多库并为进行该集成，此外也需要一个独立的库注入这些调用，可采用诸如：接口封装（wrapping interfaces）、订阅特定的库回调、将现有的遥测信号转换为OTel数据模型等机制。

所以用于开启其他库OTel可观测性功能的库，称为[插桩库](glossary.md#插桩库)（Instrumentation Libraries）。
插桩库的命名应该与被插码库的命名语义相同，如：middleware等。

如果没有规范名称，推荐采用`opentelemetry-instrumentation`作为前缀，加上被插码库的名称，如：

* opentelemetry-instrumentation-flask (Python)
* @opentelemetry/instrumentation-grpc (Javascript)