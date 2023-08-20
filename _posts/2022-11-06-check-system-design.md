---
title: 对账系统设计
author: daviswujiahao
date: 2022-11-08 11:33:00 +0800
categories: [技术]
tags: [对账,系统设计]
math: true
mermaid: true
---

[TOC]

# 背景简介

## 名词解释

- 对账

​      狭义：为了保证账簿记录的真实性、完整性、准确性，在记账以后结账之前，定期或不定期地对有关数据进行检查、核对，包含三个方面的核对工作，

1. 账证核对：是将账簿记录与记账凭证进行核对，这里是记账凭证对于财务系统来说就是订单。
2. 账账核对：是把有相互关系的多个账簿记录进行核对。有相互关系的账簿记录，包括总分类账簿间核对，明细账簿间核对等多种类型。例如：公司业务总支出和自由币、优惠券、商品采购、商家结算等支出的核对，是总分类帐薄核对；各收款账户的收支明细和总入账、总支出的核对，是明细帐薄的核对。
3. 账实核对：是各项资产物资的记录数值与实际真实数额间的核对。例如：用户购买了权益，自己的确享受了所购买的权益；购买了实物商品，真实收到了商品。

  举例说明：有家淘宝店，只卖一种杯子，一个杯子 20 元，一个月只有 1 单生意，1 单卖 1 个杯子。当用户下单购时，打印机自动打印出一份发货订单，老板根据这份纸质订单到库房拣货，然后交给快递公司发货，用户收到杯子，确认收货，20 元转入老板的账户。发货次日，老板盘点库房，发现少了一个杯子，于此同时，发现自己的账户里多了 20 元，正好是 1 个杯子的售价。这个对账过程叫**账实核对。**老板根据打印的订单对账，手里有一张纸质订单，订单总额 20 元，再看自己账户金额，多了 20 元，这个对账过程叫**账证对账。**月底对账时，发现账户只多了 10 元，于是老板翻出全部账本，发现订单账多了 20 元，快递发货账因发货减少 10 元（快递费），一计算，收入 20 元（增加），快递费 10 元（减少），即账户账应增加 10 元，这个对账叫**账账对账**。

​      **广义：某个时间段内，信息流经的多个系统间的数据核对。**

- 左/右方

  对账的双方系统的称呼，对账对的就是数据，数据源的提供方，一方叫做左方，另一方叫做右方。

- 差异

​       对账过程中左/右两方数据源的差异，分为左单边，右单边，金额不等。

​       左单边：左方账单比右方账单多出的账单记录

​       右单边：右方账单比左方账单多出的账单记录

​       金额不等：左、右两方账单文件中金额不一致的账单记录

- 聚合对账

  以对账单号聚合将金额求和后以求和的金额进行对账

- 滚动对账

  判断新产生的右单边(左单边)记录和历史产生的左单边(右单边)是否可以对平，主要解决账单记录跨天导致的差异问题。

# 为什么要对账

- 对账是信息在各个系统(尤其是整个支付系统)流转过程中，最后一道安全防线，是交易流程中重要的纠错机制。
- 避免意外和人为错误发生

​     （1）意外错误：网络稳定性、内部系统（订单、支付、风控）的健壮性。

​     （2）人为错误：人为操作失误、上游、内部恶意修改等行为。

- 对账是财务流程中重要的一环，特别是当交易量上万 / 天，人工手动对账毫无可能时，为了避免订单差错越积越多，变成糊涂账，我们需要日日结清，对账也是保证公司财务健康的必要环节。

# 对账核心逻辑

## 方案一：我要造轮子(Java版伪代码)

```java
/**
 左方文件:
 业务单号,金额,完成时间,状态
 key001, 10, 2022-11-01 00:01, 0
 key001, 11, 2022-11-01 01:01, 0
 key003, 10, 2022-11-01 02:01, 0
 key002, 10, 2022-11-01 03:01, 0
**/
inStream leftData = read("左方账单文件路径");
/**
 右方文件:
 业务单号,金额,完成时间
 key001, 10, 2022-11-01 00:01
 key002, 10, 2022-11-01 03:01
 key004, 10, 2022-11-01 03:01
**/
inStream rightData = read("右方账单文件路径");

// 根据业务单号对左/右账单文件里的账单记录进行从小到大排序
inStream leftDataSoft = sort(leftData);
inStream rightDataSoft = sort(rightData);

// 双指针遍历排完序后的左/右两方的账单文件记录
do {
  leftRow = readRow(leftDataSoft, leftIdx);
  rightRow = readRow(leftDataSoft, rightIdx);
  // 注意非空处理
  if (leftRow[0] == rightRow[0]) {
    // 两边业务单号相同
    if (leftRow[1] == rightRow[0]) {
      // leftRow 和 leftRow 为对平记录
    } else {
      // leftRow 和 leftRow 为金额不等的差异记录
    }
    leftIdx++;
    rightIdx++;
  } else if (leftRow[0] < rightRow[0]) {
    // leftRow 为左单边的差异记录
    leftIdx++;
  } else {
    // rightIdx 为右单边的差异记录
    rightIdx++;
  }
} while (leftRow != null || rightRow != null);

// 汇总对平记录、金额不等记录、左单边记录、右单边记录
```

## 方案二：我要骑自行车(shell 脚本)

```shell

 左方文件: left.csv.2022-11-01
 业务单号,金额,完成时间,状态
 key001, 10, 2022-11-01 00:01, 0
 key001, 11, 2022-11-01 01:01, 0
 key003, 10, 2022-11-01 02:01, 0
 key002, 10, 2022-11-01 03:01, 0

 右方文件: right.csv.2022-11-01
 业务单号,金额,完成时间
 key001, 10, 2022-11-01 00:01
 key002, 10, 2022-11-01 03:01
 key004, 10, 2022-11-01 03:01
 
 # 拼接唯一键：业务单号+金额，清晰为新文件
 awk -F ', ' '{printf "%s_%s\n",$1,$2}' left.csv.2022-11-01 > left_uniq
 awk -F ', ' '{printf "%s_%s\n",$1,$2}' right.csv.2022-11-01 > right_uniq
 
 # 排序
 sort left_uniq > left_uniq_sort
 sort right_uniq > right_uniq_sort
 
 # 文件对比
 comm left_uniq_sort right_uniq_sort > result.csv
 # result.csv：
 # 第一列为 左单边(或者金额不一致)
 # 第二列为有单边(或者金额不一致)
 # 第三列为对平数据
```

## 方案三：我要坐飞机(hive)

```sql
 /**
 左方文件: left.csv.2022-11-01
 业务单号,金额,完成时间,状态
 key001, 10, 2022-11-01 00:01, 0
 key001, 11, 2022-11-01 01:01, 0
 key003, 10, 2022-11-01 02:01, 0
 key002, 10, 2022-11-01 03:01, 0

 右方文件: right.csv.2022-11-01
 业务单号,金额,完成时间
 key001, 10, 2022-11-01 00:01
 key002, 10, 2022-11-01 03:01
 key004, 10, 2022-11-01 03:01
**/

# csv -> mysql/hive
# left_data_table/right_data_table:
# keyNo, amount, extra
left.csv.2022-11-01 >> left_data_table
right.csv.2022-11-01 >> right_data_table

# 执行对账sql，获取对账结果
insert into check_result 
(
  select
    t1.keyNo as leftKeyNo,
    t1.amount as leftAmount,
    t2.keyNo as rightKeyNo,
    t2.amount as rightAmount
from
    (
        select
            keyNo as keyNo,
            sum(amount) as amount
        from
            left_data_table
        group by keyNo
    ) as t1 
    full outer join 
    (
        select
            keyNo as keyNo,
            sum(amount) as amount
        from
            right_data_table
        group by keyNo
    ) as t2
    on
    t1.keyNo = t2.keyNo
 );
 
 # 对平结果
 select * from check_result where leftKeyNo != null && rightKeyNo != null && leftAmount = rightAmount;
 # 左单边
 select * from check_result where leftKeyNo != null && rightKeyNo = null;
 # 有单边
 select * from check_result where leftKeyNo = null && rightKeyNo != null;
 # 金额不等
 select * from check_result where leftKeyNo != null && rightKeyNo != null && leftAmount != rightAmount;
```

## 方案对比

| 方案名称   | 实现复杂度 | 是否可并行 | 执行效率 | 支持聚合对账 | 额外存储成本 | 依赖基础设施 |
| :--------- | ---------- | ---------- | -------- | ------------ | ------------ | ------------ |
| 纯Java代码 | 高         | 否         | 低       | 支持         | 否           | 否           |
| shell脚本  | 中         | 否         | 中       | 不支持       | 否           | 否           |
| hive       | 低         | 是         | 高       | 支持         | 是           | 是           |

# 对账平台

## 需求

对账的过程：触发对账任务 ->  拉取左/右两边对账数据  -> 执行对账产生结果(对平&&差异数据) -> 对账结果处理

- 任务什么时候执行？拉取哪几个数据源的数据？
- 对账方数据源的获取方式，有哪些字段，是否需要过滤，是否需要进行字段处理
- 产生差异怎么办？
- 对平数据怎么处理？

## 功能模块图

![未命名绘图.drawio](https://github.com/davisjiahao/davisjiahao.github.io/blob/main/_data/assets/img/功能模块图.png?raw=true)  

### 任务管理

- 任务调度：定时调度执行对账任务
- 任务配置：调度配置，数据源配置、差异管理配置、对平数据管理配置
- 任务监控：异常任务执行记录监控报警(执行失败/执行超时/漏执行)

### 数据源管理

- 字段映射：各业务账单数据的字段映射和解析
- 清洗配置：各业务账单数据的过滤配置和
- 数据拉取：根据各业务账单数据类型(sftp/hive)，拉取账单数据

### 对平数据管理

- 数据存储：持久化存储对平数据
- 结果通知：每次对账任务产生的对平汇总数据的通知
- 数据推送：接口/mq通知每次对平数据的获取方式

### 差异管理

- 滚动对账：对账过程中新产生的差异记录和历史差异记录的对账校验，尝试自动修复历史差异
- 差异回调：接口/mq通知每次产生的差异记录
- 差异回调修单：接口通知每次产生的差异记录，被调用方返回正确的账单数据，进行对账校验，尝试自动修复差异
- 手动修单：人工补上传有问题那方的账单数据，进行差异修复
- 差异通知：每次对账任务产生的差异汇总数据的通知
- 差异监控：当日扫描T-2日还未对平的差异记录，进行报警处理

## 现有系统架构图

![架构图](https://github.com/davisjiahao/davisjiahao.github.io/blob/main/_data/assets/img/架构图.png?raw=true)

## 现有系统核心逻辑实现

### 对账核心

```java
// shell + java代码
// awk数据过滤和清洗 -> 重定向合并多个对账文件 -> sort命令排序 -> Java代码双指针进行对账
```

### 分阶段流程编排

将对账流程抽象为【数据准备】【对账处理】【差异处理】【结果处理】4个阶段，每个阶段再细分为多个处理节点，各个处理节点可配置多个处理插件，进行实际的处理流程。

![对账流程](https://github.com/davisjiahao/davisjiahao.github.io/blob/main/_data/assets/img/对账流程.png?raw=true)

# 讨论

对账流程的优化？
是否有必要自己实现一套流程编排代码？
任务调度框架的选择？

