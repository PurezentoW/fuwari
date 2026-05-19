---
title: SkyWalking
published: 2024-08-26
description: SkyWalking 分布式链路追踪系统安装配置与使用指南
image: ''
tags: [JAVA, 微服务, 监控]
category: '技术分享'
draft: false
lang: zh-CN
---
# SkyWalking



## 概述

SkyWalking是一个开源的可观察性平台，用于收集，分析，聚合和可视化来自本地或者云服务中的数据。即使在整个云环境中，SkyWalking也能提供一种简便的方法来维护您的分布式系统的清晰视图。它是一个现代的APM（Application Performance Monitor 应用性能监测软件），专门为基于云、容器的分布式系统而设计。

## 选型

SkyWalking提供了用于在许多不同情况下观察和监视分布式系统的解决方案，并通过agent方式，做到高性能、低损耗、无侵入性，与类似的功能组件如：Zipkin、Pinpoint、CAT相比，skywalking无论是从性能还是社区活跃度方面考虑，都具有一定的优势。

## skywalking监控维度

skywalking从三个维度提供可观察项功能，分别是：服务，服务实例，端点

服务。表示一组/一组工作负载，这些工作负载为传入请求提供相同的行为。
服务实例。服务组中的每个单独工作负载都称为实例。像pods在Kubernetes中一样，它不必是单个OS进程，但是，如果您使用agent代理，则实例实际上是一个真正的OS进程。
端点。服务中用于传入请求的路径，例如HTTP URI路径或gRPC服务类+方法签名。

## skywalking架构

从逻辑上看，skywalking分为四个部分：探针，平台后端，存储和UI。

![在这里插入图片描述](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202408262052840.png)

探针：收集数据并重新格式化以符合SkyWalking的要求（不同的探针支持不同的来源）。
平台后端：支持数据聚合，分析和流处理，涵盖跟踪，指标和日志。
存储：设备通过开放/可插入的界面存储SkyWalking数据。您可以选择现有的实现，例如ElasticSearch，H2，MySQL，TiDB，InfluxDB，或者实现自己的实现。
UI：是一个高度可定制的基于Web的界面，允许SkyWalking最终用户可视化和管理SkyWalking数据。

## 安装（Windows）

![img](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202408262052402.png)

### skywalking安装环境信息

win10+jdk 1.8+oap 10.0.1 +es7.17+ agent9.3.0

### 搭建Elasticsearch 服务

之前搭过，localhost:9200 不在赘述

![image-20240826101538649](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202408262052083.png)

### 搭建 SkyWalking OAP 服务

OAP ： 全称Observability Analysis Platform,可观测性分析平台，SkyWalking 的OAP服务，其主要责任有两个：

#### oapService

一个是负责接收 Agent 上报上来的 Trace、Metrics 等数据，交给 Analysis Core （涉及SkyWalkingOAP 中的多个模块）进行流式分析，最终将分析得到的结果写入持久化存储中。SkyWalking 可以使用ElasticSearch、H2、MySQL等作为其持久化存储，一般线上使用ElasticSearch 集群作为其后端存储。

**配置文件**：E:\Work\ElasticSearch\skywalking\apache-skywalking-apm-bin\config\application.yml

修改存储

```yaml
storage:
  selector: ${SW_STORAGE:elasticsearch} #修改为ES
  elasticsearch:
    namespace: ${SW_NAMESPACE:""}
    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:localhost:9200} #修改为ES地址
    user: ${SW_ES_USER:""} #账号
    password: ${SW_ES_PASSWORD:""} #密码
```



#### SkyWalking UI

界面发送来的查询请求，将前面持久化的数据查询出来，组成正确的响应结果返回给 UI界面进行展示。

### 建一个项目打包

```java
@RestController
public class HelloWorld {
    private static final Logger logger = org.slf4j.LoggerFactory.getLogger(HelloWorld.class );
    @GetMapping("/test")
    public String helloworld(){

        logger.info("*****info*****: call hello world");
        return "Hello World ";

    }

    @GetMapping("/test/login")
    public String login(@RequestParam(value="name",required=true) String name, @RequestParam(value="password",required=true) String pwd){
        logger.info("*****info*****: call login");
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            logger.error("error");
        }
        return name+" "+pwd;
    }
}
```

打完包得到 jar包 **export-0.0.1-SNAPSHOT.jar**

### 配置 SkyWalking Agent

Agent 运行在各个服务实例中，负责采集服务实例的 Trace 、Metrics 等数据，然后通过 gRPC方式上报给SkyWalking后端。

#### SkyWalking Agent 下载地址

[Apache Download Mirrors](https://www.apache.org/dyn/closer.cgi/skywalking/java-agent/9.3.0/apache-skywalking-java-agent-9.3.0.tgz)

监控的jar 与 oap 和es在同一台机器上

配置文件：E:\Work\ElasticSearch\skywalking\skywalking-agent\config\agent.config

```properties
agent.service_name=${SW_AGENT_NAME:export}
```

### 启动springboot jar

java -javaagent:E:\Work\ElasticSearch\skywalking\skywalking-agent\skywalking-agent.jar -jar E:\Work\ElasticSearch\skywalking\test-jar\export-0.0.1-SNAPSHOT.jar

如果想更改oap的启动端口，可以这样执行

java -Dserver.port=9191  -javaagent:E:\Work\ElasticSearch\skywalking\skywalking-agent\skywalking-agent.jar -jar E:\Work\ElasticSearch\skywalking\test-jar\export-0.0.1-SNAPSHOT.jar

![image-20240826105852240](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202408262052966.png)

### 启动SkyWalking UI 服务

前提：oap所在的机器上，必须配置环境变量 JAVA_HOME。

先后启动\apache-skywalking-apm-bin\bin 下的

oapService.sh

webappService.sh

也可以直接启动

startup.sh

进入页面，在浏览器中输入 http://127.0.0.1:8080   进入Skywalking OAP页面
![image-20240826110236192](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202408262053442.png)

## 链路跟踪

```curl
GET http://localhost:8080/test
```

![image-20240826112110653](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202408262053866.png)

### UI界面

SkyWalking UI的包括：仪表盘、拓扑图、追踪、性能分析、日志、告警六个功能区。功能区下方为指标对象，SW的监控对象分为 服务、端点和实例三种。右下角为时间区，用来设定统计指标的时间域（所有的指标展示都依赖于这个时间范围）。点击右上"自动"按钮可以开启自动刷新模式；其余空间为指标盘展示区，用来展示各种指标信息。

这里着重介绍下 SkyWalking 中最基础的三个概念：

- **服务(Service)** ：表示对请求提供相同行为的一系列或一组工作负载。在使用 Agent 或 SDK 的时候，你可以定义服务的名字。如果不定义的话，SkyWalking 将会使用应用名称上定义的名字，为了和告警服务联动，这里推荐大家配置成应用中心中的应用名。

  > 这里，我们可以看到 应用的**服务**为 `"export"`，这是在agent 环境变量 `SW_AGENT_NAME` 中所定义的。

- **端点(Endpoint)** ：方法级别。如请求的接口路径，定时任务的方法，服务间的RPC远程调用等。

  > 这里，我们可以看到应用的一个**端点**，为 API 接口 `/test`。

- **实例(Service Instance)** ：上述的一组工作负载中的每一个工作负载称为一个实例。就像 Kubernetes 中的 pods 一样, 服务实例未必就是操作系统上的一个进程。但当你在使用 Agent 的时候, 一个服务实例实际就是操作系统上的一个真实进程。

  > 这里，我们可以看到 Spring Boot 应用的**实例**为 `{进程UUID}@{hostname}`，由 Agent 自动生成。

### 仪表盘

仪表盘中我们主要关注APM(Application Performance Management)应用性能管理、Database数据库。仪表盘中包含了服务的各种性能指标。

主要的性能指标有：

- *ApdexScore* ： 性能指数，Apdex(Application Performance Index)是一个国际通用标准，Apdex 是用户对应用性能满意度的量化值。它提供了一个统一的测量和报告用户体验的方法，把最终用户的体验和应用性能作为一个完整的指标进行统一度量，其中最高为1最低为0；

- *ResponseTime*：响应时间，即在选定时间内，服务所有请求的平均响应时间(*ms*)；

- *Throughput*: 吞吐量，即在选定时间内，每分钟服务响应的请求量(*cpm*)

- *SLA*: service level agreement，服务等级协议，SW中特指每分钟内响应成功请求的占比。

  大盘中会列出以上指标的当前的平均值，和历史走势。 ![image.png](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202408262053264.webp)

比如Slow Endpoints(ms)指标列出了选中时间域内响应耗时最久的方法，**左侧数字在这里代表响应时间**。

![image.png](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202408262053658.webp) 而Global Response Latency(响应延迟)折线图中，**P75=530代表75％的响应延迟在530毫秒内**。

**Instance(实例)指标中，除常见的吞吐量、响应时间等指标外，还会给出当前实例运行的JVM信息，如堆栈使用情况，GC次数和耗时等**。

Database中可以查看项目中使用数据库的响应时间，请求压力以及慢SQL。指标含义都很容易理解。

### 拓扑图

拓扑图用来展示服务和服务之间的依赖关系。SW会根据请求数据，自动探测出服务的依赖关系。**点击服务，会显示当前服务的部分指标。点击依赖线上的圆点，会显示服务之间的依赖情况，如每分钟吞吐量，平均延迟时间等。**

![image-20240826204404101](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202408262053067.png)

### 追踪

当用户发现服务SLA降低，或者某个具体的端口响应时间上扬明显，可以使用追踪功能查询具体的请求记录。

- 最上方为搜索区，用户可以指定搜索条件，如隶属于哪个服务、哪个实例、哪个端口，或者请求是成功还是失败，端口名支持模糊查询；
- 调用链上每一次调用称为一个跨度，每个跨度的耗时和执行结果都会被列出（默认是列表，也可选择树形结构和表格的形式）；
- 如果有步骤失败，该步骤会标记为红色。
- 点击跨度，会显示跨度详情，如果有异常发生，异常的种类、信息和堆栈都会被自动捕获，如果跨度为数据库操作，执行的SQL也会被自动记录。
- 如果项目关联了日志，可以点击右上角的查看日志查看当前接口对应的日志。

### 性能剖析

追踪功能展示出的跨度是服务调用粒度的，如果要看应用实时的堆栈信息，可以选择性能剖析功能。 在性能剖析页面新建任务后，SW将开始采集应用的实时堆栈信息。采样结束后，用户点击分析即可查看具体的堆栈信息。

1. 点击跨度右侧的"查看"，可以看到调用链的具体详情；
2. 跨度目录下方是SW收集到的具体进程堆栈信息和耗时情况。

需要提醒的是，性能剖析功能因为要实时高频率收集服务的JVM堆栈信息，对于服务本身有一定的性能消耗，只适用于耗时端点的行为分析。

新建任务的参数：

- 服务：需要分析的服务
- 端点：链路监控中端点的名称，即链路追踪中端点的完整路径。如下图中的`{GET}/owind/api/fuseQuery/fuseQueryAll` ![image.png](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202408262053385.webp)
- 监控时间：采集数据的开始时间
- 监控持续时间：监控采集多长时间
- 起始监控时间：多少秒后进行采集
- 监控间隔：多少秒采集一次
- 最大采集数：最大采集多少样本



### 日志

日志 本项目以logback作为日志框架，这里以就以logback为例集成skywalking。其它如log4j2等步骤类似。 首先在项目中添加依赖(其它日志框架需添加的依赖自行搜索)：

```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-logback-1.x</artifactId>
    <version>8.5.0</version>
</dependency>
```

在logback-spring.xml配置文件中增加skywalking appender

```xml
<appender name="SKYWALKING" class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.log.GRPCLogClientAppender">
    <!-- 日志输出编码 -->
    <encoder>
        <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n
        </pattern>
        <charset>UTF-8</charset> <!-- 设置字符集 -->
    </encoder>
</appender>
<springProfile name="local">
    <root level="info">
        <appender-ref ref="SKYWALKING"/>
        <appender-ref ref="INFO_FILE"/>
        <appender-ref ref="WARN_FILE"/>
        <appender-ref ref="ERROR_FILE"/>
    </root>
</springProfile>
```

在项目A对应agent的agent/config/agent.config文件下添加如下配置

```ini
plugin.toolkit.log.grpc.reporter.server_host=${SW_GRPC_LOG_SERVER_HOST:XXX.XXX.XXX.XXX}
plugin.toolkit.log.grpc.reporter.server_port=${SW_GRPC_LOG_SERVER_PORT:11800}
plugin.toolkit.log.grpc.reporter.max_message_size=${SW_GRPC_LOG_MAX_MESSAGE_SIZE:10485760}
plugin.toolkit.log.grpc.reporter.upstream_timeout=${SW_GRPC_LOG_GRPC_UPSTREAM_TIMEOUT:30}
```

其中server_host为SkyWalking server端地址。 重启A服务后，即可看到A服务的日志信息。在追踪中也查看跨度的相关日志信息了。

### 告警

SkyWalking通过Webhook配置告警，在Server端alarm-settings.yml文件中配置具体的告警策略及要推送的WebHook。

```yaml
--告警策略示例--
rules:
  # Rule unique name, must be ended with `_rule`.
  service_resp_time_rule:
    metrics-name: service_resp_time
    op: ">"
    threshold: 1000
    period: 10
    count: 3
    silence-period: 5
    message: 过去十分钟内有3分钟服务 {name} 平均响应时间超过1秒
```

- **Rule name：** 规则名称，也是在告警信息中显示的唯一名称。必须以`_rule`结尾，前缀可自定义
- **Metrics name：** 度量名称，取值为oal文件夹下脚本中的度量名，目前只支持`long`、`double`和`int`类型。详见Official OAL script
- **Include names：** 该规则作用于哪些实体名称，比如服务名，终端名（可选，默认为全部）
- **Exclude names：** 该规则作不用于哪些实体名称，比如服务名，终端名（可选，默认为空）
- **Threshold：** 阈值
- **OP：** 操作符，目前支持 `>`、`<`、`=`
- **Period：** 多久告警规则需要被核实一下。这是一个时间窗口，与后端部署环境时间相匹配
- **Count：** 在一个Period窗口中，如果values超过Threshold值（按op），达到Count值，需要发送警报
- **Silence period：** 在时间N中触发报警后，在TN -> TN + period这个阶段不告警。 默认情况下，它和Period一样，这意味着相同的告警（在同一个Metrics name拥有相同的Id）在同一个Period内只会触发一次
- **message：** 告警消息

配置告警规则后，即可看到规则对应的告警信息。如要实现推送钉钉，企业微信等，则还需配置对应的WebHook。



[SkyWalking - 使用指南SkyWalking8.5.0 - 使用指南 Agent部署 配置方式 UI界面介绍 - 掘金](https://juejin.cn/post/7068550705308499975)

[可视化监控应用性能-skywalking微服务体系下需要对各个服务进行监控，或者在对服务进行压测的时候都需要观察服务性能 - 掘金](https://juejin.cn/post/6993890910081646629)
