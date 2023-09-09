---
title: 流程编排那些事
author: daviswujiahao
date: 2023-01-06 11:33:00 +0800
categories: [技术]
tags: [流程编排,开源调研]
math: true
mermaid: true
---

[TOC]

# 背景介绍

## 何为编排

在计算机科学和软件工程中，"编排"是指协调和管理多个独立组件或服务，以实现特定的业务流程或工作流程。它涉及到在一个整体系统中将各个组件或服务按照预定的顺序和方式进行调度、执行和交互。编排通常用于构建分布式系统、集成不同的服务或微服务，并实现复杂的业务逻辑和流程。编排的目标是通过协调和组织各个组件或服务之间的交互，以实现特定的功能和业务需求。
## 编制(Orchestration)VS编排(Choreography)

对于编排的概念定义，业界其实有过激烈的讨论，其中讨论得一个很有意思的点是编制(Orchestration)和编排(Choreography)的区别，本文汇总了一些关于这方面的讨论和大家进行分享，感兴趣的可以参考 [编配和编排的定义之争](https://www.infoq.cn/news/2008/09/Orchestration/)。
编制的英文是 Orchestration，本意是乐队指挥，在演出的时候，乐队由指挥来统一的进行指挥和控制。编排的英文是 Choreography，本意是舞蹈编舞，舞蹈表演通常是舞蹈演员对外部感应作出响应，比如音乐的响应，并且需要与舞伴的行动和表情进行配合。
单从字面看 Orchestration为管弦乐曲，由一人负责指挥与调控各个乐师，而Choreography为舞蹈，各个舞蹈者多为对等关系，没有统一的协调者。事实上也是如此，Orchestration需要类似ESB的服务总线来统一管理调度各个服务间的通讯，而Choreography更强调的是各服务自治，各自自己去调用需要的服务，相对而言更去中心化。

![编制(Orchestration)VS编排(Choreography)](https://ask.qcloudimg.com/http-save/yehe-1475574/g10voqcq7r.jpeg)

编制（Orchestration）强调的是通过一个可执行的中心流程来协同内部及外部的服务交互。通过中心流程来控制总体的目标，涉及的操作，服务调用顺序，其就像好像交通信号灯，控制着车辆什么时候可以通行。而编排（Choreography）强调的是协作，通过消息的交互序列来控制各个部分资源的交互。参与交互的资源都是对等的，没有集中的控制，其就好像是环岛，没有集中的控制，只有一系列的规则来指明车辆在接近十字路口的时候必须要等待，直到有空间进入环岛环绕系统，然后寻找适当的时候离开。

![编制(Orchestration)VS编排(Choreography)](https://ask.qcloudimg.com/http-save/yehe-1475574/stru92o58o.jpeg)

编制（Orchestration）是有中心化调度服务以协调各组件通信的方式，以用户注册流程为例：

![用户注册流程编制](../_data/assets/img/ms-services-invoke-register-orchestration.png)
各服务间都通过ESB进行数据通信，用户注册后会向ESB发送调用活动服务的用户注册接口，ESB会将对应的请求路由给活动服务，活动服务收到请求并处理后又向ESB发送调用卡券服务生成卡券，调用积分服务生成积分，ESB路由此请求给卡券服务和积分服务。我们可以在ESB中做集中式的权限管控、日志处理等增强操作，这是它很大的优势，但也带来了不少的问题，存在中心化的ESB，所有请求都要经由ESB，所以在性能、扩展性、灵活性上会比较差。

编排（Choreography）则是去中心化的、点对点的通信的方式，还是以注册为例：
![用户注册流程编排](../_data/assets/img/ms-services-invoke-register-orchestration.png)
所有的请求都是直接调用对应的服务，没有ESB，这带来的直接好处是性能的提升，但是存在一定的服务耦合，比如用户服务就需要感知到卡券服务及活动服务，活动服务需要感知到卡券服务，但是这点其实**可以通过事件驱动架构来优化**。

当然，**刻意的区分 “编制” 与 “编排” 并没有多大价值**，**重要的是保证各参与方对所用的术语有共同的理解**，业内也并未普遍区分“编制” 与 “编排”的区别，并以编排来泛指了“编制” 与 “编排”这两种流程设计模式，故本文中后续内容的编排采用业内普遍看法，代指为“编制” 与 “编排”。

## 长流程VS短流程

流程有长流程和短流程之分:

- 长流程是指包含人工活动的流程，流程的完成时间因为人的因素会在一个较大范围内波动；
- 短流程指的是不包含人工活动的流程，在流程启动后会在一个较短的预期时间内完成。
  
**本文介绍的流程编排主要是短流程相关的编排**。

## 编排的类型

### 流程编排（Process Orchestration）

流程编排是指在一个系统或平台中，通过编排多个组件（如任务、操作、触发器等）来实现特定的业务流程。这些组件之间可以通过逻辑连接方式相互关联，以达到流程自动化和任务自动执行的目的。
流程编排一般用于自动化任务和流程，可以有效提高工作效率和业务执行的准确性。常见的应用场景包括数据处理、自动化测试、CI/CD（持续集成 / 持续交付）、工作流等。

### 服务编排（Service Orchestration）

服务编排是指在服务级别对多个独立的服务进行协调和管理，以实现特定的业务流程或工作流程。它涉及到定义和管理服务之间的依赖关系、交互方式和执行顺序，以满足业务需求。服务编排通常用于构建分布式系统或微服务架构，将各个服务按照业务流程组织起来，并通过编排引擎或工作流引擎来实现服务之间的交互和协调。

### 容器编排（Container Orchestration）

容器编排是指在容器级别对多个容器进行协调和管理，以实现高效的部署、伸缩和运维。它涉及到管理容器的创建、启动、停止、伸缩、网络配置等，以实现容器化应用程序的管理和调度。容器编排工具（如 Kubernetes）提供了自动化容器管理的能力，使得在分布式环境中大规模运行和管理容器化应用程序变得更加简单和可靠。

### 应用编排（Application Orchestration）

应用编排是指在应用程序级别对多个组件、服务或模块进行协调和管理，以实现特定的应用程序功能或业务流程。它涉及到在应用程序中定义和管理组件之间的依赖关系、交互方式和执行顺序。应用编排通常用于构建复杂的企业应用程序，协调不同的服务、模块和组件来实现整体功能。

应用编排、服务编排和容器编排是针对不同层级的编排实践。应用编排和服务编排关注在应用程序或服务级别的组件协调和管理，而容器编排关注在容器级别的部署、伸缩和运维。这些编排实践都旨在提高系统的可靠性、可伸缩性和效率，并简化复杂系统的管理和运维工作。而流程编排的概念则是可大可小，从大的方面来说，活动即流程，应用、服务以及容器都属于流程的一部分，则可以认为流程编排包含了应用编排、服务编排和容器编排；从小的方面来说，则可以认为服务里的函数执行也算是流程编排，粒度比服务编排更低。

## 工作流引擎VS流程引擎VS规则引擎VS决策引擎

- 流程引擎就是 “业务过程的部分或整体在计算机应用环境下的自动化”，它主要解决的是 “使在多个参与者之间按照某种预定义的规则传递文档、信息或任务的过程自动进行，从而实现某个预期的业务目标，或者促使此目标的实现”。通俗的说，流程就是多种业务对象在一起合作完成某件事情的步骤，把步骤变成计算机能理解的形式就是流程引擎。
- 工作流引擎和流程引擎是同一个意思，其是对工作流程及其各操作步骤之间业务规则的抽象、概括描述。在计算机中，工作流属于计算机支持的协同工作（CSCW）的一部分。工作流概念起源于生产组织和办公自动化领域，是针对日常工作中具有固定程序活动而提出的一个概念，目的是通过将工作分解成定义良好的任务或角色，按照一定的规则和过程来执行这些任务并对其进行监控，达到提高工作效率、更好的控制过程、增强对客户的服务、有效管理业务流程等目的。即工作流主要解决的问题是为了实现某个业务目标，利用计算机在多个参与者之间按某种预定规则自动传递文档、信息或者任务。工作流引擎是一个预先编码的脚本，它考虑了工作流设计，即任务应该如何从一个阶段流向另一个阶段，并执行该步骤。工作流引擎是嵌入在软件中的代码，用于将任务从一个阶段推送到另一个阶段。
- 规则引擎
业务规则引擎可以理解为程序中的一组条件，如果满足所有条件，则执行相应的程序代码。它是关于设置一个软件在特定参数内的行为准则。规则引擎的优点是，它允许非技术性软件用户根据其业务需求更改软件行为，而无需更改底层代码。业务规则引擎根据大量的信息数据做出快速可靠的决策，通常这些数据对于人类大脑来说太大了，无法处理。业务规则引擎是一个更广泛的概念中的一部分，它的范围甚至超出了工作流管理。规则引擎无法控制编排任务，但它们根据特定条件为推断决策指南。同时，它还可用于在给定条件下模拟工作流的过程。

工作流引擎和业务规则引擎都允许非技术性的最终用户在运行时更改流程行为，而无需更改代码。但它们的不同之处多于相似之处。如上所述，它们的工作模式和目的有着根本的不同。下面列出了工作流引擎和业务规则引擎之间的一些其他区别：
![规则引擎VS流程引擎](../_data/assets/img/%E8%A7%84%E5%88%99%E5%BC%95%E6%93%8E%E5%92%8C%E6%B5%81%E7%A8%8B%E5%BC%95%E6%93%8E%E5%AF%B9%E6%AF%94.png)

- 决策引擎
决策，指决定的策略或办法。是人们为各种事件出主意、做决定的过程。它是一个复杂的思维操作过程，是信息搜集、加工，最后作出判断、得出结论的过程。而决策引擎是指企业针对其客户提供个性化服务的决策平台，这些个性化服务决策包括：风险决策、精确营销决策等。
决策引擎就是把商业规则转换成商业决策，在决策引擎之上可以开发出各种不同的解决方案。

规则引擎是一个工具，本身是不带规则的，规则需要人为输入，可单独将规则从系统剥离出来放到规则引擎平台单独进行执行管理。具有一定智能化的使用价值，可以按照需求来进行规则的配置、执行、管理，不同的行业都可以配置出属于自己不同的规则平台。
而决策引擎，就是已经包含了很多的规则、决策条件，具备了对规则的决策能力，如风控决策引擎，就是在金融行业的风险控制环节进行决策的。

# 流程编排从0到1

了编排的概念之后，如果要实现一个流程编排的最小集，改如何去实现呢？一般来说，比较完善的编排应该由下面几个部分组成：

1. 流程定义：编排定义了业务流程或工作流程的步骤、顺序和条件。它描述了系统中各个组件或服务之间的依赖关系和交互方式。
2. 调度和协调：编排负责根据流程定义，调度和协调各个组件或服务的执行。它确保在正确的时间和顺序下启动和运行每个组件，以满足整体流程的要求。
3. 数据传递和转换：编排负责管理数据在各个组件或服务之间的传递和转换。它确保正确的数据被传递给需要的组件，并处理数据格式转换和映射等。
4. 错误处理和故障恢复：编排需要处理可能发生的错误和故障情况。它可以包括错误处理、重试机制、故障恢复策略等，以确保整个流程的可靠性和稳定性。
最小集应该是实现其中的1、2、3点。针对第1、2点，流程的定义可以认为是执行链的定义，如果不考虑流程中的并行流程，可以使用责任链模式来进行流程的定义和串联执行；针对第3点则可以定义简易的上下文字段来进行各个执行节点间数据的传输。基于以上前提，我们可以通过如下方式来实现一个简易的流程编排框架。

## 上下文的定义

```Java
public class Context implements Serializable {

    private final Map<Class<?>,Object> CONTEXT = new ConcurrentHashMap<>();

    public <T> T get(Class<T> clazz) {
        return (T) CONTEXT.get(clazz);
    }

    public void put(Object obj) {
        if(null == obj) {
            return;
        }
        CONTEXT.put(obj.getClass(),obj);
    }
}
```

## 任务节点的定义

```Java
public interface ActionTask {

    // 判断是否可以执行
    bool canProcess();

    // 执行业务逻辑
    void process(Context context);

}
```

## 责任链节点的定义

```Java

public class FlowNode {
    // 本节点执行任务
    private ActionTask task;
    // 后续节点任务，大于1则并发执行
    private List<FlowNode> nextNodes = Lists.newArrayList();

    public FlowNode(ActionTask task) {
        this.task = task;
    }

    // 责任链执行
    public void process(Context context) {
        if (!task.canProcess()) {
            return;
        }
        task.process();
        for (FlowNode nextNode : nextNodes) {
            nextNode.process();
        }
    }

    public void addNextNodes(FlowNode node) {
        nextNodes.add(node);
    }
}
```

## 触发流程

```Java
public static Main {

    public static void main(String[] args) {

        // 1、定义流程节点
        // 开始节点
        FlowNode startNode = new FlowNode(new StartTask());
        // 活动中心节点
        FlowNode activityNode = new FlowNode(new ActivityTask());
        // 库存中心节点
        FlowNode storeNode = new FlowNode(new StoreTask());
         // 订单中心节点
        FlowNode orderNode = new FlowNode(new OrderTask());

        // 构造流程
        startNode.addNextNodes(activityNode);
        startNode.addNextNodes(storeNode);
        storeNode.addNextNodes(orderNode);

        // 链式执行流程
        startNode.process(new Context());
        
    }
}
```



# 开源编排框架调研

## Zeebe

### 基本介绍

[Zeebe](https://docs.camunda.io/docs/0.26/components/zeebe/deployment-guide/getting-started/)是一个用于微服务编排的开源工作流引擎。在Camunda基于BPMN2.0工作流基础上衍生出来的，设计很灵活，不需要依赖后端的存储，支持复制、分片（借鉴了kafka），支持同步和异步发起流程，它提供了：

- 可见性 (visibility)：Zeebe 提供能力展示出企业工作流运行状态，包括当前运行中的工作流数量、平均耗时、工作流当前的故障和错误等；
- 工作流编排 (workflow orchestration)：基于工作流的当前状态，Zeebe 以事件的形式发布指令 (command)，这些指令可以被一个或多个微服务消费，确保工作流任务可以按预先的定义流转；
- 监控超时 (monitoring for timeouts) 或其他流程错误：同时提供能力配置错误处理方式，比如有状态的重试或者升级给运维团队手动处理，确保工作流总是能按计划完成。
为了应对超大规模，Zeebe 支持：

- 横向扩容 (horizontal scalability)：Zeebe 支持横向扩容并且不依赖外部的数据库，相反的，Zeebe 直接把数据写到所部署节点的文件系统里，然后在集群内分布式的计算处理，实现高吞吐；
- 容错 (fault tolerance)：通过简单配置化的副本机制，确保 Zeebe 能从软硬件故障中快速恢复，并且不会有数据丢失；
- 消息驱动架构 (message-driven architecture)：所有工作流相关事件被写到只追加写的日志 (append-only log) 里；
- 发布 - 订阅交互模式 (publish-subscribe interaction model)：可以保证连接到 Zeebe 的微服务根据实际的处理能力，自主的消费事件执行任务，同时提供平滑流量和背压的机制；
- BPMN2.0 标准 (Visual workflows modeled in ISO-standard BPMN 2.0)：保证开发和业务能够使用相同的语言协作设计工作流；
- 语言无关的客户端模型 (language-agnostic client model)：可以使用任何编程语言构建 Zeebe 客户端。

Zeebe架构主要包含 4 大组件：client, gateway, brokers 以及 exporters:
![Zeebe架构图](../_data/assets/img/zeebe-architecture.png)

- Client
嵌入到应用程序 (执行业务逻辑的微服务) 的库,可以完全独立于 Zeebe 扩缩容，用于跟 Zeebe 集群连接通信。客户端通过基于 HTTP/2 协议的 gRPC 与 Zeebe gateway 连接, 可用于：
  - 部署流程
  - 处理业务逻辑
  - 可以集成到各种语言项目中进行使用
  Client 中，执行单独任务的单元叫 JobWorker。
JobWorker定期请求某种类型的工作（即轮询）。此间隔和请求的作业数量可在 Zeebe 客户端中配置。如果请求类型的一项或多项作业可用，则 Zeebe会将激活的作业流式传输给工作人员。接收到作业后，JobWorker执行它们，并根据作业是否可以成功完成，为每个作业发回完成或失败命令。
许多JobWorker可以请求相同的工作类型，以扩大加工规模。在这种情况下，Zeebe 确保每个作业仅发送给其中一个JobWorker。此类作业被视为已激活，直到作业完成、失败或作业激活超时。
![JobWorker](../_data/assets/img/zeebe-job-workers-graphic.png)

- Gateway：
  - zeebe集群的统一入口点，将请求转发给代理
  - 网关是无状态的，可以根据需要添加网关，以实现负载平衡和高可用性
- Broker：
Broker 是分布式的流程引擎，维护运行中流程实例的状态。Brokers 可以分区以实现横向扩容、副本以实现容错。通常情况下，Zeebe 集群都不止一个节点。broker 不包含任何业务逻辑，它只负责：
  - 处理客户端发送的指令
  - 存储和管理运行中流程实例的状态
  - 分配任务给 job workers
Brokes 形成一个对等网络 (peer-to-peer)，这样集群不会有单点故障。集群中所有节点都承担相同的职责，所以一个节点不可用后，节点的任务会被透明的重新分配到网络中其他节点。

- Exporter: 提供 Zeebe 内状态变化的事件流。这些事件流数据有很多潜在用处，包括但不限于：
  - 监控当前流程的执行状态
  - 分析历史工作流数据，用于审计、商业使用
  - Zeebe异常追踪

- 额外的扩展组件：
  - Modeler：一个创建和编辑BPMN可视化工具。可以远程运行、部署BPMN文件
  - Operate：一个监控Zeebe流程执行的可视化工具
  - Tasklist：是一个在Zeebe中处理用户任务的工具。您可以过滤、声明和完成用户任务。

### 代码示例

```Java

// 1、创建和gateway交互的client
final String gatewayAddress = System.getenv("ZEEBE_ADDRESS");
final ZeebeClient client =
            ZeebeClient.newClientBuilder()
                .gatewayAddress(gatewayAddress)
                .build();

// 2、使用client部署工作流程
final DeploymentEvent deployment = client.newDeployCommand()
            .addResourceFromClasspath("order-process.bpmn")
            .send()
            .join();    

// 3、创建工作流实例
final Map<String, Object> data = new HashMap<>();
        data.put("orderId", 31243);
        data.put("orderItems", Arrays.asList(435, 182, 376));
final WorkflowInstanceEvent wfInstance = client.newCreateInstanceCommand()
    .bpmnProcessId("order-process")
    .latestVersion()
    .variables(data) // data 为流程实例上下文参数
    .send()
    .join();

// 4、创建JobWorker接收类型为“payment-service”的执行任务
// “payment-service” 为“order-proces”流程中的任务节点
final JobWorker jobWorker = client.newWorker().jobType("payment-service")
    .handler((jobClient, job) ->
    {
        final Map<String, Object> variables = job.getVariablesAsMap();

        System.out.println("Process order: " + variables.get("orderId"));
        double price = 46.50;
        System.out.println("Collect money: $" + price);

        final Map<String, Object> result = new HashMap<>();
        result.put("totalPrice", price);

        jobClient.newCompleteCommand(job.getKey())
            .variables(result)
            .send()
            .join();
    })
    .fetchVariables("orderId")
    .open();
```

### 总结

由上可知，Zeebe 是有中心控制的编制(Orchestration)模式，进行微服务编排时，各个微服务引入Client包后，由Broker进行统一的协调调度，来完成单个流程实例的执行。其为超大规模分布式微服务而设计，可通过横向扩容来应对高并发场景的挑战；同时基于状态机的方式记录了单个流程实例的执行情况，为流程执行提供了可监控的抓手。但是由于是中心化设计，流程发起方通过向Zeebe发起流程执行的请求，而由Zeebe间接取进行流程图中各个Job的分发和结果的收集聚合，不可避免存在一些性能损耗。

## Conductor

### 简介

netflix conductor 是基于 JAVA 语言编写的开源流程引擎，用于架构基于微服务的流程。它具备如下特性：

- 允许创建复杂的业务流程，流程中每个独立的任务都是由一个微服务所实现。
- 基于 JSON DSL 创建工作流，对任务的执行进行编排。
- 工作流在执行的过程中可见、可追溯。
- 提供暂停、恢复、重启等多种控制模型。
- 提供一种简单的方式来最大限度重用微服务。
- 拥有扩展到百万流程并发运行的服务能力。
- 通过队列服务实现客户端与服务端的分离。
- 支持 HTTP 或其他 RPC 协议进行数据传送

### 整体架构
![Alt text](../_data/assets/img/conductor.webp)
主要分为几个部分：

- Orchestrator: 负责流程的流转调度工作；
- Management/Execution Service: 提供流程、任务的管理更新等操作；
- TaskQueues: 任务队列，Orchestrator 解析出来的待执行 Task 会放到队列中；
- Worker: 任务执行 worker，从 TaskQueues 中获取任务，通过 Execution Service 更新任务状态与结果数据；
- Database: 元数据 & 运行时数据库，用于保存运行时的 Workflow、Task 等状态信息，以及流程任务定义的等原信息；
- Index: 索引数据库，用于存储执行历史；

### 基本概念

#### Task

Task 是最小执行单元，承载了一段执行逻辑，如发送 HTTP 请求等。

- System Task：被 conductor 服务执行，这些任务的执行与引擎在同一个 JVM 中。
- Worker Task：被 worker 服务执行，执行与引擎隔离开，worker 通过队列获取任务后，执行并更新结果状态到引擎。Worker 的实现是跨语言的，其使用 Http 协议与 Server 通信。
conductor 提供了若干内置 SystemTask:

   - 功能性 Task：
   - HTTP：发送 http 请求
   - JSON_JQ_TRANSFORM：jq 命令执行，一般用户 json 的转换，具体可见 jq 官方文档
   - KAFKA_PUBLISH: 发布 kafka 消息
  
- 流程控制 Task：
  
  - SWITCH（原 Decision）：条件判断分支，类似于代码中的 switch case
  - FORK：启动并行分支，用于调度并行任务
  - JOIN：汇总并行分支，用于汇总并行任务
  - DO_WHILE：循环，类似于代码中的 do while
  - WAIT：一直在运行中，直到外部时间触发更新节点状态，可用于等待外部操作
  - SUB_WORKFLOW：子流程，执行其他的流程
  - TERMINATE：结束流程，以指定输出提前结束流程，可以与 SWITCH 节点配合使用，类似代码中的提前return语句
 - 自定义 Task：
   - 对于 System Task，Conductor 提供了 WorkflowSystemTask 抽象类，可以自定义扩展实现。
   - 对于 Worker Task，可以实现 conductor 的 client Worker 接口实现执行逻辑。

#### Workflow

Workflow 由一系列需要执行的 Task 组成，conductor 采用 json 来描述 Task 的流转关系。
除基本的顺序流程外，借助内置的 SWITCH、FORK、JOIN、DO_WIHLE、TERMINATE 任务，还能实现分支、并行、循环、提前结束等流程控制。

#### Input&Output

Task 的输入是一种映射，其作为工作流实例化的一部分或某些其他 Task 的输出。允许将来自工作流或其他 Task 的输入 / 输出作为随后执行的 Task 的输入。
Task 有自己的输入和输出，输入输出都是 jsonobject 类型。
Task 可以引用其他 Task 的输入输出，使用 ${taskxxx.output} 的方式引用。引用语法为 json-path，除最基础的 ${taskxxx.output} 的值解析方式外，还支持其他复杂操作，如过滤等，具体见 json-path 语法。
启动 Workflow 时可以传入流程的输入数据，Task 可以通过 ${workflow.input} 的方式引用。
Task 实现原子操作的处理以及流程控制操作，Workflow 定义描述 Task 的流转关系，Task 引用 Workflow 或者其它 Task 的输入输出。通过这些机制，conductor 实现了 JSON DSL 对流程的描述。


## kstry
### 简介

[kstry](http://kstry.cn/doc)是流程编排框架、并发框架、微服务业务整合框架。可以将原本存在于代码中错综复杂的方法调用关系以可视化流程图的形式更直观的展示出来，并提供了将所见的方法节点加以控制的配置手段。框架不能脱离程序执行之外存在，只能在方法与方法的调用中生效和使用，比如某个接口的一次调用。不会像Activiti、Camunda等任务流框架一样，脱离程序执行之外将任务实例存储和管理。不同使用场景中，因其发挥作用的不同，可以理解成不同的框架，Kstry 是：
- 【流程编排框架】提供所见（ 流程图示 ）即所得（ 代码执行 ）的可视化能力，可自定义流程协议，支持流程配置的热部署
- 【并发框架】可通过极简操作将流程从串行升级到并行，支持任务拆分、任务重试、任务降级、子任务遍历、指定流程或任务的超时时间
- 【微服务业务整合框架】支持自定义指令和任务脚本，可负责各种基础能力组件的拼装
- 【轻量的 TMF2.0 (opens new window)框架】可以通过以下三个步骤来满足同一接口下各类业务对实现功能的不同诉求：抽象能力资源、定义并将抽象出的能力资源授权给业务角色、同一流程的不同场景可分别匹配不同角色再将其下的能力资源任务加以执行
kstry的核心逻辑
![kstry](../_data/assets/img/kstry.png)

### 使用示例

- 编写组件代码
  
``` Java
@TaskComponent(name = "goods")
public class GoodsService {
    
	@NoticeResult
    @TaskService(name = "init-base-info")
    public GoodsDetail initBaseInfo(@ReqTaskParam(reqSelf = true) GoodsDetailRequest request) {
        return GoodsDetail.builder().id(request.getId()).name("商品").build();
    }
}

```

- 定义流程图
![Alt text](../_data/assets/img/image-20211211151733111.png)


- 执行流程

```Java

StoryRequest<GoodsDetail> req = ReqBuilder.returnType(GoodsDetail.class).startId("kstry-demo-goods-show").request(request).build();
TaskResponse<GoodsDetail> fire = storyEngine.fire(req);
if (fire.isSuccess()) {
    return fire.getResult();
}
```



## LiteFlow

### 简介

[LiteFlow](https://liteflow.cc/pages/5816c5/)是一个非常强大的现代化的规则引擎框架，融合了编排特性和规则引擎的所有特性。可用于复杂的组件化业务的编排领域，独有的 DSL 规则驱动整个复杂业务，并可实现平滑刷新热部署，支持多种脚本语言规则的嵌入。帮助系统变得更加丝滑且灵活。

LiteFlow 适用于拥有复杂逻辑的业务，比如说价格引擎，下单流程等，这些业务往往都拥有很多步骤，这些步骤完全可以按照业务粒度拆分成一个个独立的组件，进行装配复用变更。使用 LiteFlow，你会得到一个灵活度高，扩展性很强的系统。因为组件之间相互独立，也可以避免改一处而动全身的这样的风险。

### 使用示例
- 定义组件
  
```Java
@LiteflowComponent("b")
public class Bnode extends NodeComponent {

    @Override
    public void process() throws Exception {
        Book contextBean = this.getContextBean(Book.class);
        contextBean.setName("my");
        System.out.println(">>>>>>>Bnode 被调用了。。。。。"+ this.getRequestData()  +"oooooooooooooooooooooooo" + contextBean);
    }
}

@LiteflowComponent("c")
public class Cnode extends NodeComponent {

    @Override
    public void process() throws Exception {
        System.out.println(">>>>>>>Cnode 被调用了。。。。。"+ this.getRequestData()  +">>>>" +this.getContextBean(Book.class));
    }
}

```

- 编写规则文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE flow PUBLIC  "liteflow" "liteflow.dtd">
<flow>
    <chain name="chain1">
        THEN(a, b, c);
    </chain>
</flow>
```

- 执行流程
```Java
@SpringBootTest
class SimonkingLiteflowApplicationTests {

    @Resource
    private FlowExecutor flowExecutor;

    @Test
    void contextLoads() {
        Book book = new Book();
        book.setName("wsss:" + Thread.currentThread().getName());
        LiteflowResponse response = flowExecutor.execute2Resp("chain1", "arg****************", book);
        System.out.println("response:" + JSON.toJSONString(response));
    }
}
```


## Compileflow
[compileflow](https://github.com/alibaba/compileflow/blob/master/README_CN.md) 是一个非常轻量、高性能、可集成、可扩展的流程引擎。
compileflow Process 引擎是淘宝工作流 TBBPM 引擎之一，是专注于纯内存执行，无状态的流程引擎，通过将流程文件转换生成 java 代码编译执行，简洁高效。当前是阿里业务中台交易等多个核心系统的流程引擎。
compileflow 能让开发人员通过流程编辑器设计自己的业务流程，将复杂的业务逻辑可视化，为业务设计人员与开发工程师架起了一座桥梁。
功能列表:

- 高性能：通过将流程文件转换生成 java 代码编译执行，简洁高效。
- 丰富的应用场景：在阿里巴巴中台解决方案中广泛使用，支撑了导购、交易、履约、资金等多个业务场景。
- 可集成：轻量、简洁的设计使得可以极其方便地集成到各个解决方案和业务场景中。
- 完善的插件支持：流程设计目前有 IntelliJ IDEA、Eclipse 插件支持，可以在流程设计中实时动态生成 java 代码并预览，所见即所得。
- 支持流程设计图导出 svg 文件和单元测试代码。
- 支持基于 Java 反射和 Spring 容器的代码触发


# 流程编排在保险业务的应用
![流程编排在页面配置中的应用](../_data/assets/img/流程编排在页面配置中的应用.png)
基于LiteFlow二次开发的全声明式组件编排的实现

## 声明式组件定义

```Java
// 声明式组件父类
public abstract class BaseFlowNodeToContext extends NodeComponent {

    @Override
    public void process() throws Exception {
        PageContextH5 context = this.getContextBean(PageContextH5.class);
        context.getContext().put(getNodeId(), JSON.toJSON(handle(context)));
    }

    /**
     * 节点处理
     * @param contextH5
     * @return
     */
    protected abstract Object handle(PageContextH5 contextH5);
}

// 方法注解，标识为声明式组件
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface ParamLiteFlowComponent {
    String name() default "";
}

// class注解，标识为包含声明式组件的bean
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Component
public @interface ParamLiteFlowComponents {

    String prefix() default "";

    @AliasFor(annotation = Component.class, attribute = "value")
    String value() default "";

    @AliasFor(annotation = Component.class, attribute = "value")
    String id() default "";
}


// 为业务处理函数添加声明式组件注解
@ParamLiteFlowComponents
@HttpApiClient(name = "service-config", interceptor = CommonHttpApiInterceptor.INTERCEPTOR_NAME)
public interface ServiceConfigApiService {
    /**
     * 获取服务配置详情
     * @param serviceId
     * @return
     */
    @ParamLiteFlowComponent(name = "getServiceConfig")
    @GetMapping("/v1/service/detailByServiceId")
    ServiceConfigDetailDTO detailByServiceId(@RequestParam("serviceId") Long serviceId);

    /**
     *
     * @param serviceId
     * @return
     */
    @ParamLiteFlowComponent(name = "getProductListByServiceId")
    @GetMapping("/v1/service/getProductListByServiceId")
    List<ProductBaseInfo> getProductListByServiceId(@RequestParam("serviceId") Long serviceId);
}
```

## 声明式组件自注册

使用反射获取spring容器中被ParamLiteFlowComponents 注解标识的bean，并获取被ParamLiteFlowComponent注解标识的函数，注册为声明式组件

``` Java
@Configuration
public class ParamNodeBeanFactory {
    @PostConstruct
    public void registerComponent() {
        Map<String, Object> beansWithAnnotation = BeanUtil.getBeanFactory().getBeansWithAnnotation(ParamLiteFlowComponents.class);
        for (Map.Entry<String, Object> entry : beansWithAnnotation.entrySet()) {
            List<Method> methods = Arrays.stream(entry.getValue().getClass().getDeclaredMethods()).
                    filter(e -> Modifier.isPublic(e.getModifiers())).
                    collect(Collectors.toList());
            ParamLiteFlowComponents annotation = AnnotationUtils.findAnnotation(entry.getValue().getClass(), ParamLiteFlowComponents.class);
            for (Method method : methods) {
                ParamLiteFlowComponent flowComponent = AnnotationUtils.findAnnotation(method, ParamLiteFlowComponent.class);
                if (flowComponent == null) {
                    continue;
                }
                String nodeId = annotation.prefix() + (StringUtils.isEmpty(flowComponent.name()) ? method.getName() : flowComponent.name());
                FlowBus.addSpringScanNode(nodeId, new BaseFlowNodeToContext() {
                    @SneakyThrows
                    @Override
                    protected Object handle(PageContextH5 contextH5) {
                        List<Object> jsonArray = contextH5.getFlowParas().get(getNodeId());
                        if (jsonArray == null || jsonArray.size() == 0) {
                            return method.invoke(entry.getValue());
                        }
                        return method.invoke(entry.getValue(), jsonArray.toArray());
                    }
                });
            }
        }
    }

}

```

## 流程配置文件

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<flow>
    <nodes>
        <node id="parseGetServiceConfigParam" name="解析getServiceConfig的入参" type="script" language="groovy">
            <![CDATA[
                def serviceId = Long.valueOf(pageContextH5.requestPageDetail.productId);
                pageContextH5.flowParas.put("getServiceConfig", [serviceId]);
            ]]>
        </node>
        <node id="parseShowInvalidTimeParam" name="解析showInvalidTime的入参" type="script" language="groovy">
            <![CDATA[
                def useValidPeriodUnit = pageContextH5.context.getServiceConfig.serviceInfo.useValidPeriodUnit;
                def useValidPeriod = pageContextH5.context.getServiceConfig.serviceInfo.useValidPeriod;
                pageContextH5.flowParas.put("showInvalidTime", [useValidPeriodUnit, useValidPeriod]);
            ]]>
        </node>
        <node id="parseQuerySubOrderParam" name="解析querySubOrder的入参" type="script" language="groovy">
            <![CDATA[
                def subOrderId = pageContextH5.requestPageDetail.subOrderId;
                def user = pageContextH5.userInfo;
                def querySubOrderDTO = new com.xiaoju.manhattan.neptune.domain.dto.QuerySubOrderDTO(subOrderId:subOrderId, user:user);
                pageContextH5.flowParas.put("querySubOrder", [querySubOrderDTO]);
            ]]>
        </node>
        <node id="parseGetServiceUseRecordParam" name="解析getServiceUseRecord的入参" type="script" language="groovy">
            <![CDATA[
                def subOrderId = pageContextH5.requestPageDetail.subOrderId;
                def user = pageContextH5.userInfo;
                def queryServiceUseRecordDTO = new com.xiaoju.manhattan.neptune.domain.dto.QueryServiceUseRecordDTO(subOrderId:subOrderId, user:user);
                pageContextH5.flowParas.put("queryUseRecordList", [queryServiceUseRecordDTO]);
            ]]>
        </node>
    </nodes>

    <!-- 服务兑换结果页 -->
    <chain name="showServiceResultFlow">
        THEN(
        THEN(parseGetServiceConfigParam, getServiceConfig),
        WHEN(
            THEN(parseShowInvalidTimeParam, showInvalidTime),
            THEN(parseQuerySubOrderParam, querySubOrder),
            THEN(parseGetServiceUseRecordParam, queryUseRecordList)
             ),
        generateRequestId
        );
    </chain>
</flow>
```

### 执行流程

```Java
Map<String, OBject> context = Map.newHashMap();
LiteflowResponse liteflowResponse = flowExecutor.execute2Resp("flowName.xml", null, context);
```


# 参考文献

- [服务都微了，编排怎么整？](https://cloud.tencent.com/developer/article/1080878)
- [一文读懂微服务编排利器 —Zeebe](https://www.infoq.cn/article/lh7wh4x2i1zcx8mikoho)
- [Orchestration vs Choreography](https://camunda.com/blog/2023/02/orchestration-vs-choreography/)
- [编制与协同设计](https://gudaoxuri.gitbook.io/microservices-architecture/wei-fu-wu-hua-zhi-ji-shu-jia-gou/services-invoke)
- [编排的概念以及应用编排，服务编排和容器编排的区别](https://developer.aliyun.com/article/1270564)
- [工作流可以赋能的几个技术方向](https://zhuanlan.zhihu.com/p/445880346)
- [流程编排引擎技术方向](https://juejin.cn/s/%E6%B5%81%E7%A8%8B%E7%BC%96%E6%8E%92%E5%BC%95%E6%93%8E%E6%8A%80%E6%9C%AF%E6%96%B9%E5%90%91)
