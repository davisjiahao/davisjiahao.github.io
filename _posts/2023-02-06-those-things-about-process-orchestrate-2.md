---
title: 流程编排那些事-第二部-流程编排的简单实现
author: daviswujiahao
date: 2023-02-06 11:33:00 +0800
categories: [技术]
tags: [流程编排,开源调研]
math: true
mermaid: true
---

[TOC]

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