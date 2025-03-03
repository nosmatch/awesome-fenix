# 聚合度量

度量（Metrics）的目的是揭示系统的总体运行状态。相信大家应该见过这样的场景：舰船的驾驶舱或者卫星发射中心的控制室，在整个房间最显眼的位置，布满整面墙壁的巨型屏幕里显示着一个个指示器、仪表板与统计图表，沉稳端坐中央的指挥官看着屏幕上闪烁变化的指标，果断决策，下达命令……如果以上场景被改成指挥官双手在键盘上飞舞，双眼紧盯着日志或者追踪系统，试图判断出系统工作是否正常。这光想像一下，都能感觉到一股身份与行为不一致的违和气息。由此可见度量与日志、追踪的差别，度量是用经过聚合统计后的高维度信息，以最简单直观的形式来总结复杂的过程，为监控、预警提供决策支持。

:::center

![](./images/mon.png)

图 10-8 Windows 系统的任务管理器界面

:::

如果你人生经历比较平淡，没有驾驶航母的经验，甚至连一颗卫星或者导弹都没有发射过，那就只好打开电脑，按`CTRL`+`ALT`+`DEL`呼出任务管理器，看看上面图 10-8 这个熟悉的界面，它也是一个非常具有代表性的度量系统。

度量总体上可分为客户端的指标收集、服务端的存储查询以及终端的监控预警三个相对独立的过程，每个过程在系统中一般也会设置对应的组件来实现，你不妨现在先翻到下面，看一眼 Prometheus 的组件流程图作为例子，图中在 Prometheus Server 左边的部分都属于客户端过程，右边的部分就属于终端过程。

[Prometheus](https://prometheus.io/)在度量领域的统治力虽然还暂时不如日志领域中 Elastic Stack 的统治地位那么稳固，但在云原生时代里，基本也已经能算是事实标准了，接下来，笔者将主要以 Prometheus 为例，介绍这三部分组件的总体思路、大致内容与理论标准。

## 指标收集

指标收集部分要解决两个问题：“如何定义指标”以及“如何将这些指标告诉服务端”， 如何定义指标这个问题听起来应该是与目标系统密切相关的，必须根据实际情况才能讨论，其实并不绝对，无论目标是何种系统，都是具备一些共性特征。确定目标系统前我们无法决定要收集什么指标，但指标的数据类型（Metrics Types）是可数的，所有通用的度量系统都是面向指标的数据类型来设计的：

- **计数度量器**（Counter）：这是最好理解也是最常用的指标形式，计数器就是对有相同量纲、可加减数值的合计量，譬如业务指标像销售额、货物库存量、职工人数等等；技术指标像服务调用次数、网站访问人数等都属于计数器指标。
- **瞬态度量器**（Gauge）：瞬态度量器比计数器更简单，它就表示某个指标在某个时点的数值，连加减统计都不需要。譬如当前 Java 虚拟机堆内存的使用量，这就是一个瞬态度量器；又譬如，网站访问人数是计数器，而网站在线人数则是瞬态度量器。
- **吞吐率度量器**（Meter）：吞吐率度量器顾名思义是用于统计单位时间的吞吐量，即单位时间内某个事件的发生次数。譬如交易系统中常以 TPS 衡量事务吞吐率，即每秒发生了多少笔事务交易；又譬如港口的货运吞吐率常以“吨/每天”为单位计算，10 万吨/天的港口通常要比 1 万吨/天的港口的货运规模更大。
- **直方图度量器**（Histogram）：直方图是常见的二维统计图，它的两个坐标分别是统计样本和该样本对应的某个属性的度量，以长条图的形式表示具体数值。譬如经济报告中要衡量某个地区历年的 GDP 变化情况，常会以 GDP 为纵坐标，时间为横坐标构成直方图来呈现。
- **采样点分位图度量器**（Quantile Summary）：分位图是统计学中通过比较各分位数的分布情况的工具，用于验证实际值与理论值的差距，评估理论值与实际值之间的拟合度。譬如，我们说“高考成绩一般符合正态分布”，这句话的意思是：高考成绩高低分的人数都较少，中等成绩的较多，将人数按不同分数段统计，得出的统计结果一般能够与正态分布的曲线较好地拟合。
- 除了以上常见的度量器之外，还有 Timer、Set、Fast Compass、Cluster Histogram 等其他各种度量器，采用不同的度量系统，支持度量器类型的范围肯定会有差别，譬如 Prometheus 支持了上面提到五种度量器中的 Counter、Gauge、Histogram 和 Summary 四种。

对于“如何将这些指标告诉服务端”这个问题，通常有两种解决方案：**拉取式采集**（Pull-Based Metrics Collection）和**推送式采集**（Push-Based Metrics Collection）。所谓 Pull 是指度量系统主动从目标系统中拉取指标，相对地，Push 就是由目标系统主动向度量系统推送指标。这两种方式并没有绝对的好坏优劣，以前很多老牌的度量系统，如[Ganglia](<https://en.wikipedia.org/wiki/Ganglia_(software)>)、[Graphite](https://graphiteapp.org/)、[StatsD](https://github.com/statsd/statsd)等是基于 Push 的，而以 Prometheus、[Datadog](https://www.datadoghq.com/pricing/)、[Collectd](https://en.wikipedia.org/wiki/Collectd)为代表的另一派度量系统则青睐 Pull 式采集（Prometheus 官方解释[选择 Pull 的原因](https://prometheus.io/docs/introduction/faq/#why-do-you-pull-rather-than-push?)）。Push 还是 Pull 的权衡，不仅仅在度量中才有，所有涉及客户端和服务端通讯的场景，都会涉及该谁主动的问题，上一节中讲的追踪系统也是如此。

:::center
![](./images/prometheus_architecture.png)
图 10-9 Prometheus 组件流程图（图片来自[Prometheus 官网](https://github.com/prometheus/prometheus)）
:::

一般来说，度量系统只会支持其中一种指标采集方式，因为度量系统的网络连接数量，以及对应的线程或者协程数可能非常庞大，如何采集指标将直接影响到整个度量系统的架构设计。Prometheus 基于 Pull 架构的同时还能够有限度地兼容 Push 式采集，是因为它有 Push Gateway 的存在，如图 10-9 所示，这是一个位于 Prometheus Server 外部的相对独立的中介模块，将外部推送来的指标放到 Push Gateway 中暂存，然后再等候 Prometheus Server 从 Push Gateway 中去拉取。Prometheus 设计 Push Gateway 的本意是为了解决 Pull 的一些固有缺陷，譬如目标系统位于内网，通过 NAT 访问外网，外网的 Prometheus 是无法主动连接目标系统的，这就只能由目标系统主动推送数据；又譬如某些小型短生命周期服务，可能还等不及 Prometheus 来拉取，服务就已经结束运行了，因此也只能由服务自己 Push 来保证度量的及时和准确。

由推和拉决定该谁主动以后，另一个问题是指标应该以怎样的网络访问协议、取数接口、数据结构来获取？如同计算机科学中其他这类的问题类似，一贯的解决方向是“定义规范”，应该由行业组织和主流厂商一起协商出专门用于度量的协议，目标系统按照协议与度量系统交互。譬如，网络管理中的[SNMP](https://en.wikipedia.org/wiki/Simple_Network_Management_Protocol)、Windows 硬件的[WMI](https://en.wikipedia.org/wiki/Windows_Management_Instrumentation)、以及此前提到的 Java 的[JMX](https://en.wikipedia.org/wiki/Java_Management_Extensions)都属于这种思路的产物。但是，定义标准这个办法在度量领域中就不是那么有效，上述列举的度量协议，只在特定的一块小块领域上流行过。原因一方面是业务系统要使用这些协议并不容易，你可以想像一下，让订单金额存到 SNMP 中，让基于 Golang 实现的系统把指标放到 JMX Bean 里，即便技术上可行，这也不像是正常程序员会干的事；另一方面，度量系统又不会甘心局限于某个领域，成为某项业务的附属品。度量面向的是广义上的信息系统，横跨存储（日志、文件、数据库）、通讯（消息、网络）、中间件（HTTP 服务、API 服务），直到系统本身的业务指标，甚至还会包括度量系统本身（部署两个独立的 Prometheus 互相监控是很常见的）。所以，上面这些度量协议其实都没有成为最正确答案的希望。

既然没有了标准，有一些度量系统，譬如老牌的 Zabbix 就选择同时支持了 SNMP、JMX、IPMI 等多种不同的度量协议，另一些度量系统，以 Prometheus 为代表就相对强硬，选择任何一种协议都不去支持，只允许通过 HTTP 访问度量端点这一种访问方式。如果目标提供了 HTTP 的度量端点（如 Kubernetes、Etcd 等本身就带有 Prometheus 的 Client Library）就直接访问，否则就需要一个专门的 Exporter 来充当媒介。

Exporter 是 Prometheus 提出的概念，它是目标应用的代表，既可以独立运行，也可以与应用运行在同一个进程中，只要集成 Prometheus 的 Client Library 便可。Exporter 以 HTTP 协议（Prometheus 在 2.0 版本之前支持过 Protocol Buffer，目前已不再支持）返回符合 Prometheus 格式要求的文本数据给 Prometheus 服务器。

得益于 Prometheus 的良好社区生态，现在已经有大量各种用途的 Exporter，让 Prometheus 的监控范围几乎能涵盖所有用户所关心的目标，如表 10-2 所示。绝大多数用户都只需要针对自己系统业务方面的度量指标编写 Exporter 即可。

:::center

表 10-2 常用 Exporter

:::

| 范围      | 常用 Exporter                                                                              |
| --------- | ------------------------------------------------------------------------------------------ |
| 数据库    | MySQL Exporter、Redis Exporter、MongoDB Exporter、MSSQL Exporter 等                        |
| 硬件      | Apcupsd Exporter，IoT Edison Exporter， IPMI Exporter、Node Exporter 等                    |
| 消息队列  | Beanstalkd Exporter、Kafka Exporter、NSQ Exporter、RabbitMQ Exporter 等                    |
| 存储      | Ceph Exporter、Gluster Exporter、HDFS Exporter、ScaleIO Exporter 等                        |
| HTTP 服务 | Apache Exporter、HAProxy Exporter、Nginx Exporter 等                                       |
| API 服务  | AWS ECS Exporter， Docker Cloud Exporter、Docker Hub Exporter、GitHub Exporter 等          |
| 日志      | Fluentd Exporter、Grok Exporter 等                                                         |
| 监控系统  | Collectd Exporter、Graphite Exporter、InfluxDB Exporter、Nagios Exporter、SNMP Exporter 等 |
| 其它      | Blockbox Exporter、JIRA Exporter、Jenkins Exporter， Confluence Exporter 等                |

顺便一提，前文提到了一堆没有希望成为最终答案的协议，一种名为[OpenMetrics](https://github.com/OpenObservability/OpenMetrics)的度量规范正在从 Prometheus 的数据格式中逐渐分离出来，有望成为监控数据格式的国际标准，最终结果如何，要看 Prometheus 本身的发展情况，还有 OpenTelemetry 与 OpenMetrics 的关系如何协调。

## 存储查询

指标从目标系统采集过来之后，应存储在度量系统中，以便被后续的分析界面、监控预警所使用。存储数据对于计算机软件来说是司空见惯的操作，但如果用传统关系数据库的思路来解决度量系统的存储，效果可能不会太理想。举个例子，假设你建设一个中等规模的、有着 200 个节点的微服务系统，每个节点要采集的存储、网络、中间件和业务等各种指标加一起，也按 200 个来计算，监控的频率如果按秒为单位的话，一天时间内就会产生超过 34 亿条记录，这很大概率会出乎你的意料之外：

:::center

> 200（节点）× 200（指标）× 86400（秒）= 3,456,000,000（记录）
> :::

大多数这种 200 节点规模的系统，本身一天的业务发生数据都远到不了 34 亿条，建设度量系统，肯定不能让度量反倒成了业务系统的负担，可见，度量的存储是需要专门研究解决的问题。至于如何解决，让我们先来观察一段 Prometheus 的真实度量数据，如下所示：

```json
{
	// 时间戳
	"timestamp": 1599117392,
	// 指标名称
	"metric": "total_website_visitors",
	// 标签组
	"tags": {
		"host": "icyfenix.cn",
		"job": "prometheus"
	},
	// 指标值
	"value": 10086
}
```

观察这段度量数据的特征：每一个度量指标由时间戳、名称、值和一组标签构成，除了时间之外，指标不与任何其他因素相关。指标的数据总量固然是不小的，但它没有嵌套、没有关联、没有主外键，不必关心范式和事务，这些都是可以针对性优化的地方。事实上，业界早就已经存在了专门针对该类型数据的数据库了，即“时序数据库”（Time Series Database）。

:::quote 额外知识：时序数据库

时序数据库用于存储跟随时间而变化的数据，并且以时间（时间点或者时间区间）来建立索引的数据库。

时序数据库最早是应用于工业（电力行业、化工行业）应用的各类型实时监测、检查与分析设备所采集、产生的数据，这些工业数据的典型特点是产生频率快（每一个监测点一秒钟内可产生多条数据）、严重依赖于采集时间（每一条数据均要求对应唯一的时间）、测点多信息量大（常规的实时监测系统均可达到成千上万的监测点，监测点每秒钟都在产生数据）。

时间序列数据是历史烙印，具有不变性,、唯一性、有序性。时序数据库同时具有数据结构简单，数据量大的特点。

:::

写操作，时序数据通常只是追加，很少删改或者根本不允许删改。针对数据热点只集中在近期数据、多写少读、几乎不删改、数据只顺序追加这些特点，时序数据库被允许做出很激进的存储、访问和保留策略（Retention Policies）：

- 以[日志结构的合并树](https://en.wikipedia.org/wiki/Log-structured_merge-tree)（Log Structured Merge Tree，LSM-Tree）代替传统关系型数据库中的[B+Tree](https://en.wikipedia.org/wiki/B%2B_tree)作为存储结构，LSM 适合的应用场景就是写多读少，且几乎不删改的数据。
- 设置激进的数据保留策略，譬如根据过期时间（TTL）自动删除相关数据以节省存储空间，同时提高查询性能。对于普通数据库来说，数据会存储一段时间后就会被自动删除这种事情是不可想象的。
- 对数据进行再采样（Resampling）以节省空间，譬如最近几天的数据可能需要精确到秒，而查询一个月前的时，只需要精确到天，查询一年前的数据，只要精确到周就够了，这样将数据重新采样汇总就可以极大节省了存储空间。

时序数据库中甚至还有一种并不罕见却更加极端的形式，叫作[轮替型数据库](https://en.wikipedia.org/wiki/RRDtool)（Round Robin Database，RRD），以环形缓冲（在“[服务端缓存](/architect-perspective/general-architecture/diversion-system/cache-middleware.html)”一节介绍过）的思路实现，只能存储固定数量的最新数据，超期或超过容量的数据就会被轮替覆盖，因此也有着固定的数据库容量，却能接受无限量的数据输入。

Prometheus 服务端自己就内置了一个强大时序数据库实现，“强大”并非客气，近几年它在[DB-Engines](https://db-engines.com/en/ranking/time+series+dbms)的排名中不断提升，目前已经跃居时序数据库排行榜的前三。该时序数据库提供了名为 PromQL 的数据查询语言，能对时序数据进行丰富的查询、聚合以及逻辑运算。某些时序库（如排名第一的[InfluxDB](https://en.wikipedia.org/wiki/InfluxDB)）也会提供类 SQL 风格查询，但 PromQL 不是，它是一套完全由 Prometheus 自己定制的数据查询[DSL](https://en.wikipedia.org/wiki/Domain-specific_language)，写起来风格有点像带运算与函数支持的 CSS 选择器。譬如要查找网站`icyfenix.cn`访问人数，会是如下写法：

```
// 查询命令：
total_website_visitors{host=“icyfenix.cn”}

// 返回结果：
total_website_visitors{host=“icyfenix.cn”,job="prometheus"}=(10086)
```

通过 PromQL 可以轻易实现指标之间的运算、聚合、统计等操作，在查询界面中往往需要通过 PromQL 计算多种指标的统计结果才能满足监控的需要，语法方面的细节笔者就不详细展开了，具体可以参考[Prometheus 的文档手册](https://prometheus.io/docs/prometheus/latest/querying/basics/)。

最后补充说明一下，时序数据库对度量系统来说是很合适的选择，但并不是说绝对只有用时序数据库才能解决度量指标的存储问题，Prometheus 流行之前最老牌的度量系统 Zabbix 用的就是传统关系数据库来存储指标。

## 监控预警

指标度量是手段，最终目的是做分析和预警。界面分析和监控预警是与用户更加贴近的功能模块，但对度量系统本身而言，它们都属于相对外围的功能。与追踪系统的情况类似，广义上的度量系统由面向目标系统进行指标采集的客户端（Client，与目标系统进程在一起的 Agent，或者代表目标系统的 Exporter 等都可归为客户端），负责调度、存储和提供查询能力的服务端（Server，Prometheus 的服务端是带存储的，但也有很多度量服务端需要配合独立的存储来使用的），以及面向最终用户的终端（Backend，UI 界面、监控预警功能等都归为终端）组成。狭义上的度量系统就只包括客户端和服务端，不包含终端。

按照定义，Prometheus 应算是处于狭义和广义的度量系统之间，尽管它确实内置了一个界面解决方案“Console Template”，以模版和 JavaScript 接口的形式提供了一系列预设的组件（菜单、图表等），让用户编写一段简单的脚本就可以实现可用的监控功能。不过这种可用程度，往往不足以支撑正规的生产部署，只能说是为把度量功能嵌入到系统的某个子系统中提供了一定便利。在生产环境下，大多是 Prometheus 配合 Grafana 来进行展示的，这是 Prometheus 官方推荐的组合方案，但该组合也并非唯一选择，如果要搭配 Kibana 甚至 SkyWalking（8.x 版之后的 SkyWalking 支持从 Prometheus 获取度量数据）来使用也都是完全可行的。

良好的可视化能力对于提升度量系统的产品力十分重要，长期趋势分析（譬如根据对磁盘增长趋势的观察判断什么时候需要扩容）、对照分析（譬如版本升级后对比新旧版本的性能、资源消耗等方面的差异）、故障分析（不仅从日志、追踪自底向上可以分析故障，高维度的度量指标也可能自顶向下寻找到问题的端倪）等分析工作，既需要度量指标的持续收集、统计，往往还需要对数据进行可视化，才能让人更容易地从数据中挖掘规律，毕竟数据最终还是要为人类服务的。

除了为分析、决策、故障定位等提供支持的用户界面外，度量信息的另一种主要的消费途径是用来做预警。譬如你希望当磁盘消耗超过 90%时给你发送一封邮件或者是一条微信消息，通知管理员过来处理，这就是一种预警。Prometheus 提供了专门用于预警的 Alert Manager，将 Alert Manager 与 Prometheus 关联后，可以设置某个指标在多长时间内达到何种条件就会触发预警状态，触发预警后，根据路由中配置的接收器，譬如邮件接收器、Slack 接收器、微信接收器、或者更通用的[WebHook](https://en.wikipedia.org/wiki/Webhook)接收器等来自动通知用户。
