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
    - [Telemetry SDK](#telemetry-sdk)
    - [Constructors](#constructors)
    - [SDK Plugins](#sdk-plugins)
    - [Exporter Library](#exporter-library)
    - [Instrumented Library](#instrumented-library)
    - [Instrumentation Library](#instrumentation-library)
    - [Instrumentation Scope](#instrumentation-scope)
    - [Tracer Name / Meter Name](#tracer-name--meter-name)
    - [Execution Unit](#execution-unit)
  - [Logs](#logs)
    - [Log Record](#log-record)
    - [Log](#log)
    - [Embedded Log](#embedded-log)
    - [Standalone Log](#standalone-log)
    - [Log Attributes](#log-attributes)
    - [Structured Logs](#structured-logs)
    - [Flat File Logs](#flat-file-logs)
    - [Log Appender / Bridge](#log-appender--bridge)

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

### Telemetry SDK

Denotes the library that implements the *OpenTelemetry API*.

See [Library Guidelines](library-guidelines.md#sdk-implementation) and
[Library resource semantic conventions](resource/semantic_conventions/README.md#telemetry-sdk).

### Constructors

Constructors are public code used by Application Owners to initialize and configure the OpenTelemetry SDK and contrib packages. Examples of constructors include configuration objects, environment variables, and builders.

### SDK Plugins

Plugins are libraries which extend the OpenTelemetry SDK. Examples of plugin interfaces are the `SpanProcessor`, `Exporter`, and `Sampler` interfaces.

### Exporter Library

Exporters are SDK Plugins which implement the `Exporter` interface, and emit telemetry to consumers.

### Instrumented Library

Denotes the library for which the telemetry signals (traces, metrics, logs) are gathered.

The calls to the OpenTelemetry API can be done either by the Instrumented Library itself,
or by another [Instrumentation Library](#instrumentation-library).

Example: `org.mongodb.client`.

### Instrumentation Library

Denotes the library that provides the instrumentation for a given [Instrumented Library](#instrumented-library).
*Instrumented Library* and *Instrumentation Library* may be the same library
if it has built-in OpenTelemetry instrumentation.

See [Overview](overview.md#instrumentation-libraries) for a more detailed definition and naming guidelines.

Example: `io.opentelemetry.contrib.mongodb`.

Synonyms: *Instrumenting Library*.

### Instrumentation Scope

A logical unit of the application code with which the emitted telemetry can be
associated. It is typically the developer's choice to decide what denotes a
reasonable instrumentation scope. The most common approach is to use the
[instrumentation library](#instrumentation-library) as the scope, however other
scopes are also common, e.g. a module, a package, or a class can be chosen as
the instrumentation scope.

If the unit of code has a version then the instrumentation scope is defined by
the (name,version) pair otherwise the version is omitted and only the name is
used. The name or (name,version) pair uniquely identify the logical unit of the
code that emits the telemetry. A typical approach to ensure uniqueness is to use
fully qualified name of the emitting code (e.g. fully qualified library name or
fully qualified class name).

The instrumentation scope is used to obtain a
[Tracer or Meter](#tracer-name--meter-name).

The instrumentation scope may have zero or more additional attributes that provide
additional information about the scope. For example for a scope that specifies an
instrumentation library an additional attribute may be recorded to denote the URL of the
repository URL the library's source code is stored. Since the scope is a build-time
concept the attributes of the scope cannot change at runtime.

### Tracer Name / Meter Name

This refers to the `name` and (optional) `version` arguments specified when
creating a new `Tracer` or `Meter` (see
[Obtaining a Tracer](trace/api.md#tracerprovider)/[Obtaining a Meter](metrics/api.md#meterprovider)).
The name/version pair identifies the
[Instrumentation Scope](#instrumentation-scope), for example the
[Instrumentation Library](#instrumentation-library) or another unit of
application in the scope of which the telemetry is emitted.

### Execution Unit

An umbrella term for the smallest unit of sequential code execution, used in different concepts of multitasking. Examples are threads, coroutines or fibers.

## Logs

### Log Record

A recording of an event. Typically the record includes a timestamp indicating
when the event happened as well as other data that describes what happened,
where it happened, etc.

Synonyms: *Log Entry*.

### Log

Sometimes used to refer to a collection of Log Records. May be ambiguous, since
people also sometimes use `Log` to refer to a single `Log Record`, thus this
term should be used carefully and in the context where ambiguity is possible
additional qualifiers should be used (e.g. `Log Record`).

### Embedded Log

`Log Records` embedded inside a [Span](trace/api.md#span)
object, in the [Events](trace/api.md#add-events) list.

### Standalone Log

`Log Records` that are not embedded inside a `Span` and are recorded elsewhere.

### Log Attributes

Key/value pairs contained in a `Log Record`.

### Structured Logs

Logs that are recorded in a format which has a well-defined structure that allows
to differentiate between different elements of a Log Record (e.g. the Timestamp,
the Attributes, etc). The *Syslog protocol* ([RFC 5424](https://tools.ietf.org/html/rfc5424)),
for example, defines a `structured-data` format.

### Flat File Logs

Logs recorded in text files, often one line per log record (although multiline
records are possible too). There is no common industry agreement whether
logs written to text files in more structured formats (e.g. JSON files)
are considered Flat File Logs or not. Where such distinction is important it is
recommended to call it out specifically.

### Log Appender / Bridge

A log appender or bridge is a component which bridges logs from an existing log
API into OpenTelemetry using the [Log Bridge API](./logs/bridge-api.md). The
terms "log bridge" and "log appender" are used interchangeably, reflecting that
these components bridge data into OpenTelemetry, but are often called appenders
in the logging domain.