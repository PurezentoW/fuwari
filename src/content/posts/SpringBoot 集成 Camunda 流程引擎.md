---
title: SpringBoot 集成 Camunda 流程引擎
published: 2024-01-31
description: SpringBoot 集成 Camunda 工作流引擎，实现业务流程自动化与 BPMN 建模实战
image: ''
tags: [SpringBoot, Camunda, 工作流]
category: '技术分享'
draft: false
lang: zh-CN
---
# SpringBoot 集成 Camunda 流程引擎



## 什么是工作流？

> **ChatGPT**:
>
> 工作流可以理解为一种规定了工作或业务流程的方式。就像我们在日常生活中完成任务一样，工作流定义了任务的顺序、执行者和相关的规则。
>
> 想象一下你要组织一个活动，比如举办一个生日派对。你可以按照一定的步骤来完成这个任务，比如确定日期和地点、发送邀请、购买食物和饮料、布置场地等。这些步骤就构成了一个工作流，每个步骤都有特定的执行者和依赖关系。
>
> 在工作环境中，工作流可以帮助组织管理各种业务流程，如审批文件、处理订单、解决问题等。通过定义工作流，可以明确每个环节的责任人、任务执行的顺序和条件。这样可以提高工作效率，减少错误和延迟，并且可以追踪任务的进展和监控整个流程。
>
> 工作流管理系统是一种软件工具，它可以帮助组织创建、执行和监控工作流。它提供了一个简单易用的界面，让用户可以方便地设计和配置工作流程。同时，系统还能自动分配任务、跟踪任务的执行情况，并提供实时的状态和报告，帮助组织更好地管理业务流程。



## Java工作流开源框架

目前主流的开源框架,它们都源自于jbpm

- Activiti

- Camunda

- Flowable

工作流程图这块其实有一个统一的标准，那就是 BPMN。BPMN 全称是 Business Process Model and Notation，中文译作业务流程模型和标记法。

这是一套图形化表示法，用图形来表示业务流程模型。BPMN 最初由业务流程管理倡议组织（BPMI, Business Process Management Initiative）开发，BPMI 于 2005 年与对象管理组织（OMG, Object Management Group）合并，并于 2011 年 1 月 OMG 发布 2.0 版本，同时改为现在的名称。

一句话，就是流程图这块有一个特别古老的规范，那就是 BPMN，而我们前面所说的无论是 Activiti、Flowable 还是 Camunda，都是支持这个规范的，所以呢，无论你使用哪一个流程引擎，都可以使用同一套流程图。

![image-20240124233117495](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202401310945246.png)

一个流程图中主要包含四方面的内容：

- **事件**

首先在一个流程图中应该有开始事件和结束事件，也就是上图的两个圆圈。

- **连线**

连线就是将事件、任务、网关等连在一起的线条。

- **任务**

用户任务：表示人工要介入做的事情。比如同意与否，或者输入一些参数，要让人工完成任务，就需要一个表单系统，让人工输入数据，或者显示数据给人看，这也是为什么用户任务和表单系统结合在一起的原因，用户任务需要用户向引擎提交一个完成任务的动作，否则流程会暂停在这里等待。

服务任务：表示机器自动做的事情。调用服务的任务，这个服务可以是一个 Spring JavaBean，也可以是一个远程 REST 服务，流程会自动执行服务任务。

- **网关**

​	互斥网关

​	相容网关

​	事件网关

​	并行网关





## 概念

- **流程（PROCESS）**: 通过工具建模最终生成的BPMN文件，里面有整个流程的定义
- **流程实例（Instance）**：流程启动后的实例
- **流程变量（Variables）**：流程任务之间传递的参数
- **任务（TASK）**：流程中定义的每一个节点
- **流程部署**：将之前流程定义的.bpmn文件部署到工作流平台

## 核心组件

- **Process Engine**-流程引擎
- **Web Applicatons**- 基于web的管理页面



## Camunda

> Camunda 是一个基于 Java 的框架，支持用于工作流和流程自动化的 BPMN、用于案例管理的 CMMN 和用于业务决策管理的 DMN。



github地址：https://github.com/camunda/camunda-bpm-platform

### API

#### ProcessEngine

为流程引擎，可以通过他获取相关service，里面集成了很多相关service

#### RepositoryService

此服务提供用于管理和操作部署和流程定义的操作，使用camunda的第一要务

#### RuntimeService

运行相关，启动流程实例、删除、搜索等

#### TaskService

所有围绕任务相关的操作，如完成、分发、认领等

#### HistoryService

提供引擎搜集的历史数据服务

#### IdentityService

用户相关，实际中用不太到



## Springboot集成

#### 依赖

```xml
		<dependency>
            <groupId>org.camunda.bpm.springboot</groupId>
            <artifactId>camunda-bpm-spring-boot-starter</artifactId>
            <version>7.18.0</version>
        </dependency>
        <dependency>
            <groupId>org.camunda.bpm.springboot</groupId>
            <artifactId>camunda-bpm-spring-boot-starter-rest</artifactId>
            <version>7.18.0</version>
        </dependency>
        <dependency>
            <groupId>org.camunda.bpm.springboot</groupId>
            <artifactId>camunda-bpm-spring-boot-starter-webapp</artifactId>
            <version>7.18.0</version>
        </dependency>
```

#### 数据库

```yaml
server:
  port: 8080

# camunda登录信息配置
camunda.bpm:
  admin-user:
    id: admin  #用户名
    password: 123456  #密码
    firstName: yu
  filter:
    create: All tasks

# mysql连接信息
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/camunda
    username: root
    password: root
    type: com.mysql.cj.jdbc.MysqlDataSource
```

#### 启动

![image-20240124232231169](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202401242322397.png)

自动生成表

![image-20240124232448198](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202401242324627.png)

- **ACT_ID_**

这部分表示用户模块，配置文件里面的用户，信息就在此模块

- **ACT_HI_**

表示流程历史记录

- `act_hi_actinst`：执行的活动历史
- `act_hi_taskinst`：执行任务历史
- `act_hi_procinst`：执行流程实例历史
- `act_hi_varinst`：流程变量历史表
- **ACT_RE_**

表示流程资源存储

- `act_re_procdef`：流程定义存储
- `act_re_deployment`: 自动部署，springboot每次启动都会重新部署，生成记录
- **ACT_RU_**

表示流程运行时表数据，流程结束后会删除

- `act_ru_execution`：运行时流程实例
- `act_ru_task`：运行时的任务
- `act_ru_variable`：运行时的流程变量
- **ACT_GE_**

流程通用数据

- `act_ge_bytearray`：每次部署的文件2进制数据，所以如果文件修改后，重启也没用，因为重新生成了记录，需要清掉数据库，或者这个表记录

#### 绘制流程图

![image-20240124233117495](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202401310945007.png)

直接上传到服务上

![image-20240124233331899](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202401310945633.png)

#### 测试

```java
@RestController
public class TaskController {

    @Autowired
    private TestTaskService testTaskService;

    /**
     * 开启流程
     */
    @PostMapping("/start/process")
    public void startProcess(){
        testTaskService.startProcess();
    }
    /**
     * 查询流程定义
     *
     * @return {@link List}<{@link ProcessDefinition}>
     */
    @PostMapping("/find/process")
    public List<ProcessDefinition> findProcess(){
        return testTaskService.findProcesses();
    }
    /**
     * 查询任务
     *
     * @return {@link List}<{@link Task}>
     */
    @PostMapping("/task")
    public List<Task> findTasks(){
        return testTaskService.findTasks();
    }
}
```

```java
@Service
public class TestTaskService {

    @Autowired
    private TaskService taskService;
    @Autowired
    private RuntimeService runtimeService;
    @Autowired
    private RepositoryService repositoryService;

    /**
     * 启动过程
     */
    public void startProcess() {
        ProcessInstance instance = runtimeService.startProcessInstanceByKey("Process_1dcaeol");
        System.out.println(instance.toString());
    }

    public List<ProcessDefinition> findProcesses() {
        return repositoryService.createProcessDefinitionQuery().list();
    }

    public List<Task> findTasks() {
        return taskService.createTaskQuery().list();
    }
```

调用开启流程

![image-20240124233614046](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202401310946178.png)

Camunda TaskList这边会有任务

![image-20240124233650732](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202401310946964.png)

### 资料参考：

[Activiti、Flowable和Camunda选型和对比 - hanease - 博客园 (cnblogs.com)](https://www.cnblogs.com/hanease/p/17270299.html)

[SpringBoot 集成 Camunda 流程引擎，实现一套完整的业务流程 (qq.com)](https://mp.weixin.qq.com/s/8w1S7Fwqxq_ghlE-z7d5BQ)

[极简 Java 工作流概念入门 - 掘金 (juejin.cn)](https://juejin.cn/post/7136085418054778893)
