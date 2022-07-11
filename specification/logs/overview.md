# OpenTelemetry 日志 总览

**状态**: [起草阶段](https://opentelemetry.io/status/)

<details>
<summary>内容大纲</summary>

<!-- toc -->

- [简介](#introduction)
- [非OpenTelemetry解决方案目前的瓶颈](#limitations-of-non-opentelemetry-solutions)
- [OpenTelemetry解决方案介绍](#opentelemetry-solution)
- [日志如何做关联](#log-correlation)
- [事件和日志](#events-and-logs)
- [传统和现在日志源](#legacy-and-modern-log-sources)
  * [系统日志](#system-logs)
  * [组件日志](#infrastructure-logs)
  * [第三方应用日志](#third-party-application-logs)
  * [老系统自身业务日志](#legacy-first-party-applications-logs)
    + [Via File or Stdout Logs](#via-file-or-stdout-logs)
    + [Direct to Collector](#direct-to-collector)
  * [新系统自身业务日志](#new-first-party-application-logs)
- [OpenTelemetry Collector](#opentelemetry-collector)
- [自动检测已存在日志行为](#auto-instrumenting-existing-logging)


<!-- tocstop -->

</details>

## 简介

目前日志遥测数据是一个很大遗留问题. 大部分开发语言都有广泛的日志库去很好支撑日志功能。
Opentelemetry 相对于在 metrics、traces 目前有清晰设计，详细定义了全新的API，完整开发了多语言的API。
但是，日志考量上会有很大的差异：一方面，我们要提供改进、更好的可观测领域集成方案，同时还要兼容好过去系统存在的不同日志库。
对于Opentelemetry 这个目标难度极大，我们也欣然接受现状挑战：
致力于让已经存在日志库，日志收集系统，和日志方案都能很好运转。

## 非OpenTelemetry解决方案目前的瓶颈

很不幸运的事，目前整合日志解决方案对于可观测信号集成是比较弱的。很典型是在链路和监控工具中，比如时间戳、源头属性等相关信息的不完整，
日志支持受到限制。想要关联起来非常脆弱，因为属性值往往写入日志，链路和指标通过不同方式采集，比如不同采集器。
采集链路和指标时，没有一个通用方式能带上日志(比如应用系统和运行的基础组件)的源头信息，同时让遥测数据精准和强有力方式关联起来。

同时，日志没有通用方式去产生和记录请求的上下文。在分布式系统中经常产生不同的系统组件日志收集是不相关的。
这里展示没有 Opentelemetry 方式做可观测流程图的现状



![Separate Collection Diagram](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/logs/img/separate-collection.png)

这里用到了不同库，不同的采集器，用到不同协议和数据模型，遥测数据采集完最后送到不同的独立后台，甚至都不知道一起工作是否正常


## OpenTelemetry 解决方案


分布式链路引进了请求上下文传播的概念

根本上讲，日志也有上下文传播的概念。日志保存下上下文标识(比如链路和Span的id或者用户自定义标识)，它就可以和链路形成丰富的关联。
而且分布式系统各个不同组件暴露日志，日志间也形成关联。这样，分布式系统日志变得非常有价值了。

这里有一种很有前景的可观测工具。标准化的metrics、traces和logs关联，支持了分布式上下文日志的传播，
统一metrics、traces和logs的源信息，提高了传统和现代系统可观测性信息的单个和组合的价值。
这是Opentelemetry 收集 logs, traces and metrics 整体的构想



![Unified Collection Diagram](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/logs/img/unified-collection.png)

我们用Opentelemetry 约定数据格式去暴露logs, traces and metrics，发送采集数据到OpenTelemetry Collector，
Collector 它可以统一方式丰富处理数据。比如，一个Kubernetes Pod 的描述，不需要应用程序单独做任何处理，
这些描述Pod的属性自动通过Collector的 k8sprocessor 就能自动添加到遥测数据中。更重要的事，三个维度数据还被关联统一起来了。
Collector 保证了 logs, traces and metrics 里包含了同样的属性名、熟悉值，这样能够准确无误描述这个Kubernetes Pod来做于哪里。
在后端就能够拿到这个Pod 清晰、准确的关联起来的Pod 数据。



## 日志如何做关联

日志可以在几个维度上与其它的可观测数据相关联： 

- 通过执行时的时间进行关联。日志、链路追踪和度量指标可以记录执行发生的时间点或时间范围。这是最基本的关联形式。
- 通过执行上下文进行关联，执行上下文也称为请求上下文。在 span 中记录执行上下文（trace、span id 以及用户定义的上下文）是一种标准做法。OpenTelemetry 通过在日志中记录 TraceId 和 SpanId 信息将这种做法扩展到日志中。通过这种方式，直接将对应于相同执行上下文的日志和跟踪关联起来。通过这种方式，还能够将涉及特定请求过程中的分布式系统的不同组件的日志关联起来。
- 通过遥测的来源进行关联，遥测的来源也称为资源上下文。 OpenTelemetry 链路追踪和度量指标内包含它们的资源信息。我们通过在日志记录中记录资源来将此做法扩展到日志中。

以上 3 个相关性可以成为强大的导航、过滤、查询和分析能力的基础。 OpenTelemetry 旨在通过上述 3 种关联的方式来记录和收集日志。

## 事件和日志

维基百科中[日志文件的定义](https://en.wikipedia.org/wiki/Log_file)：

> 在计算中，日志文件是记录操作系统或其他软件运行中发生的事件的文件。

在 OpenTelemetry 中使用的日志记录的概念与维基百科的定义一致。我们认为在可观测性领域，从数据建模的角度来看，日志记录和记录的事件之间没有重要区别。

从 OpenTelemetry 的角度来看，日志记录和事件是同一概念的不同名称。

一些产品可能想区分开收集自某些来源的事件与其他来源的日志。OpenTelemetry 认为，从数据建模的角度来看，日志记录和事件之间没有本质上的不同，其不同之处在于来源本身。

因此，产品重心在于根据数据来源做出区分，而不是试图将数据任意归类为事件与日志。

## 传统和现代日志源

区分几种传统和现代日志源很重要。首先，这直接影响我们如何准确地访问这些日志以及我们如何收集它们。其次，我们对这些日志的生成方式以及是否可以修改日志中囊括的信息有不同程度的控制。

下面我们列出了几个类别的日志，并且描述为了更好地体验可观测性解决方案，每个类别的日志可以做些什么。

### 系统日志

这些是操作系统生成且我们无法控制的日志。我们无法更改日志的格式或修改日志囊括的信息。系统格式的示例有 Syslog 和 Windows 事件日志。

系统日志在主机级别（可能是物理机、虚拟机或容器）写入，并具有预定义的格式和内容（请注意，应用程序也可能将记录写入标准的系统日志中：下面的 [第三方应用程序日志](https://github.com/open-telemetry/docs-cn/tree/main/specification/logs#third-party-application-logs) 部分介绍了这种情况）。

### 基础设施日志

由各种基础设施组件生成的日志，例如 Kubernetes 事件（如果您想知道为什么在日志上下文中讨论事件，请参阅[事件和日志](https://github.com/open-telemetry/docs-cn/tree/main/specification/logs#事件和日志)）。与系统日志一样，基础设施日志缺少请求上下文，并且可以通过资源上下文来补充节点、pod、容器等的信息。 

OpenTelemetry 收集器或其他代理可以从最常见的基础设施控制器中查询日志。

### 第三方应用程序日志

应用程序通常将日志写入标准输出、文件或其他专用介质（例如应用程序的 Windows 事件日志）。这些日志可以有许多不同的格式，包括以下变化：

- 自由的文本格式，自动化和或通过可靠的方法来解析其中的结构化数据是困难的。
- 更好地指定和有时可自定义格式，可以解析提取出结构化数据（例如 Apache 日志或 RFC5424 Syslog）。
- 正式的结构化格式（例如，具有明确定义架构的 JSON 文件或 Windows 事件日志）。

收集系统被要求有能力发现最常用的应用程序，并具有可以将这些日志结构化的解析器。与系统和基础设施日志一样，应用程序日志通常缺少请求上下文，但可以通过资源上下文来补充主机和基础设施的属性信息，以及应用程序级别的属性信息（例如应用程序名称、版本、数据库名称 — 如果它是 DBMS 等）。

OpenTelemetry 建议使用收集器的[文件日志接收器](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver)来收集应用程序日志。或者，采用日志收集代理，例如 FluentBit，可以收集日志，然后发送到 OpenTelemetry 收集器，在那里可以进一步处理和补充日志信息。

### 原第一方应用程序日志

这些是内部创建的应用程序。负责配置日志收集基础设施的人员有时能够修改这些应用程序，以更改日志的写入方式和日志囊括的信息。例如，应用程序的日志输出内容的格式可能被重新配置为 json 而不是纯文本，这样做有助于提高日志收集的可靠性。

开发人员可以手动对这些应用程序进行更重要的修改，例如将请求上下文添加到每个日志语句中，然而，由于 OpenTelemetry  在日志领域的努力，这种情况可能会越来越少。

相对于手动修改每个日志语句，我们通过一个有趣且较不费力的方式“升级”应用程序日志 —— 提供全自动或半自动检测的解决方案，这些解决方案修改应用程序使用的日志库，以自动输出请求上下文，如每个日志语句的 trace id 或 span id。如果请求使用的传播约定符合标准，则可以从传入的请求中自动提取出请求上下文，例如通过 [W3C TraceContext](https://w3c.github.io/trace-context/)。此外，从应用程序发出的请求可以被注入相同的请求上下文数据，从而导致上下文通过应用程序传播，并创造一个契机，从所有可以通过这种方式检测的应用程序收集的日志中获得完整的请求上下文。

#### Via File or Stdout Logs


#### Direct to Collector



### New First-Party Application Logs


## OpenTelemetry Collector


## Auto-Instrumenting Existing Logging



## Trace Context in Legacy Formats
