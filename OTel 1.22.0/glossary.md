# 术语表

术语表（glossary）文档定义了OTel规范中的术语（terms）。此外一些基础术语可参阅[概述](overview.md)文档。

<details>
<summary>目录</summary>

<!-- toc -->

- [术语表](#术语表)
  - [用户角色](#用户角色)
    - [应用负责人](#应用负责人)
    - [库作者](#库作者)
    - [插桩作者](#插桩作者)
    - [插件作者](#插件作者)
  - [通用](#通用)
    - [信号](#信号)
    - [包](#包)
    - [ABI兼容性](#abi兼容性)
    - [带内与带外数据](#带内与带外数据)
    - [手动插桩](#手动插桩)
    - [自动插桩](#自动插桩)
    - [遥测SDK](#遥测sdk)
    - [构造器](#构造器)
    - [SDK插件](#sdk插件)
    - [导出器库](#导出器库)
    - [插桩构建库](#插桩构建库)
    - [插桩库](#插桩库)
    - [插桩作用域](#插桩作用域)
    - [追踪器与计量器名称](#追踪器与计量器名称)
    - [执行单元](#执行单元)
  - [日志](#日志)
    - [日志记录](#日志记录)
    - [日志](#日志-1)
    - [嵌入式日志](#嵌入式日志)
    - [独立日志](#独立日志)
    - [日志属性](#日志属性)
    - [结构化日志](#结构化日志)
    - [平面文件日志](#平面文件日志)
    - [日志输出器/桥接器](#日志输出器桥接器)

<!-- tocstop -->

</details>

## 用户角色

### 应用负责人

应用负责人（Application Owner）是应用或服务的维护者，负责OTel SDK生命周期内的配置和管理。

### 库作者

库作者（Library Author）是应用依赖的共享库的维护者，该共享库以实现OTel插桩为目标。

### 插桩作者

插桩作者（Instrumentation Author）是基于OTel API所编写的OTel插桩的维护者，这些插桩代码通常存在于程序代码，共享库或插桩库中。

### 插件作者

插件作者（Plugin Author）是基于OTel SDK插件接口所编写的插件的维护者。

## 通用

### 信号

OTel基于信号（Signals）或遥测（telemetry）类型构建。
指标（metrics）、日志（logs）、调用链（traces）和baggage都是信号的例子。
每种信号都代表一组连续且独立的功能。
每种信号有独立的生命周期，并定义其自身稳定性级别。

### 包

OTel规范中，术语**包**（Packages）表示一组代表单一依赖的代码，可独立于其他包导入程序中。在某些语言中可能会使用其它术语表示此概念，如“模块”。需要注意的是，在一些语言中，术语**包**代表其他概念。

### ABI兼容性

ABI（application binary interface）是一个接口，其定义了在机器代码级别上，应用组件之间的交互性，如：应用程序执行文件与共享对象库编译后的二进制文件之间的交互。
ABI兼容性意味着一个库的新编译版本可以正确地连接到一个目标可执行文件，而不需要重新编译该执行文件。

在一些提供机器代码的语言中，ABI兼容性很重要，而另一些语言则不太关心此需求。

### 带内与带外数据

> 在通讯学中，**带内信令**（in-band signaling）是指使用与音视频等用户数据相同的频段（band）或通道（channel）发送控制信息。与之相对的是**带外信令**，其使用不同的通道或独立的网络发送控制信息。参与[Wikipedia](https://en.wikipedia.org/wiki/In-band_signaling)。

OTel中，我们将分布式系统内，作为业务消息的一部分在组件之间传递的数据，称为**带内数据**（in-band data）。例如，调用链或baggages以http headers的形式放入http请求中。这些数据通常不包含遥测信号，但是会用于关联合并各组件产生的遥测信号。
遥测信号自身则作为**带外数据**（out-of-band data），并以专用消息从应用程序发送，其通常由后台任务异步传输，而非在业务逻辑的关键路径上进行传输。
指标（metrics）、日志（logs）、调用链（traces）都是作为**带外数据**发送至后端系统。

### 手动插桩

手动插桩（Manual Instrumentation）是指基于OTel API人工编码，如[调用链API](trace/api.md)，[指标API](metrics/api.md)或其它API，从用户代码或共享框架（如MongoDB、Redis等）采集遥测信号。

### 自动插桩

自动插桩（Automatic Instrumentation）是指不需要用户修改程序代码的方式采集遥测信号。此方式因编程语言而异，示例包含：代码操作（编译期间或运行时中）、猴子补丁（monkey patching）以及使用eBPF程序。

同义词: *Auto-instrumentation*.

### 遥测SDK

遥测（Telemetry）SDK是实现*OTel API*的库。

更多请查看[库指南](library-guidelines.md#sdk实现)以及[库资源语义规范](resource/semantic_conventions/README.md#遥测sdk)。

### 构造器

构造器（Constructors）是应用负责人用来初始化和配置OTel SDK以及贡献包的公共代码。例如：配置对象、环境变量以及构建器（builders）。

### SDK插件

插件是扩展OTel SDK的库，常见的插件接口有：`SpanProcessor`、`Exporter`和`Sampler`接口等。

### 导出器库

导出器库（Exporter library）是实现`Exporter`接口的SDK插件，并将遥测信号发送至消费者。

### 插桩构建库

插桩构建库（Instrumented Library）是用于采集遥测信号（调用链、指标、日志）的库。
对OTel API的调用即可由**插桩构建库**自己完成，也可由其它[插桩库](#instrumentation-library)完成。

例如：`org.mongodb.client`。

### 插桩库

插桩库（Instrumentation Library）是指为[插桩构建库](#插桩构建库)提供插桩功能的库。
**插桩构建库**和**插桩库**可以是同一个库，如果该库内置了OTel插桩代码。

详细定义及命名指南请参阅[概览](overview.md#插桩库)。

例如：`io.opentelemetry.contrib.mongodb`。

同义词：*Instrumenting Library*。

### 插桩作用域

插桩作用域（Instrumentation Scope）是应用代码中关联要发送的遥测信号的逻辑单元。通常由开发者来决定合理的插桩作用域，常规使用[插桩库](#插桩库)的作用域，但是也有其他常用的作用域，如使用一个模块、包或类作为插桩作用域。

如果代码单元有版本，则用**名称+版本**定义插桩作用域，否则省略**版本**，只是用**名称**。名称或名称+版本必须可唯一标识发送遥测信号的代码单元，通常使用该代码单元的**全限定名**（fully qualified name）保证唯一性（全限定库/类名）。

插桩作用域用于获取一个[`Tracer`或`Meter`](#追踪器与计量器名称)。

插桩作用域可以有附加的属性提供额外的作用域信息，例如：可通过属性指定插桩库存放源码的仓库URL。由于作用域是一个构建时（build-time）概念，所以作用域的属性在运行时中不可修改。

### 追踪器与计量器名称

追踪器与计量器名称（Tracer Name / Meter Name）指创建一个新的`Tracer`或`Meter`时的`name`和`version`（可选）参数，详见[使用追踪器](trace/api.md#tracerprovider)/[使用计量器](metrics/api.md#meterprovider)
每对`name`和`version`参数标识一个[插桩作用域](#插桩作用域)，例如：应用程序中发送遥测信号的[插桩库](#插桩库)或其他的代码单元。

### 执行单元

执行单元（Execution Unit）是一个通用术语，代指顺序代码执行的最小单元，用于多任务的不同概念中，例如：线程（threads）、协程（coroutines）或纤程（fibers）。

## 日志

### 日志记录

日志记录（Log Record）是事件（event）的记录，通常包含一个时间戳用于标识事件何时发生，以及其他描述事件发生于何地、何事的数据。

同义词: *日志条目（Log Entry）*.

### 日志

日志代指日志记录（log records）的合集，有时也会使用日志代指单条日志记录，因此，需要结合上下文小心使用，以免引起歧义。
Sometimes used to refer to a collection of Log Records. May be ambiguous, since
people also sometimes use `Log` to refer to a single `Log Record`, thus this
term should be used carefully and in the context where ambiguity is possible
additional qualifiers should be used (e.g. `Log Record`).

### 嵌入式日志

嵌入式日志（Embedded Log）即为嵌入在[span](trace/api.md#span)中的日志，通常在span的[事件](trace/api.md#添加事件)列表中。

### 独立日志

独立日志（Standalone Log）指单独记录的，没有嵌入在span中的日志。

### 日志属性

日志属性（Log Attributes）指日志条目中的键值对（Key/value pairs）。

### 结构化日志

结构化日志（Log Attributes）指以某种结构格式记录的日志，可区分日志中的不同元素（时间戳、各属性等），例如：*Syslog协议* ([RFC 5424](https://tools.ietf.org/html/rfc5424))即定义了一种结构化日志格式。

### 平面文件日志

平面文件日志（Flat File Logs）即为日志记录在文本中，通常每行为一条日志条目（也可能是多条）。目前行业标准没有规范结构化日志（日JSON文件）是否为平面文件日志。所以需要自行约定。

### 日志输出器/桥接器

日志输出器/桥接器（Log Appender / Bridge）是一个应用组件，其使用[Log Bridge API](./logs/bridge-api.md)将现有的日志API转换为OTel协议。这两个术语可互换使用，均指将数据转换为OTel协议的组件，但是在日志领域中，通常成为输出器（Appender）。