---
title: 服务系统设计
author: daviswujiahao
date: 2022-08-01 11:33:00 +0800
categories: [技术]
tags: [服务中心,系统设计]
math: true
mermaid: true
---
[TOC]


# 产品方案说明书

  为更好的服务保险用户，提升用户 NPS，现以礼品金 & 积分兑换等服务形式，为用户提供有价值，可用性高的服务，用以提升用户体验，不断提高用户粘性，从而提高用户的复购率。
# 系统功能模块图

![未命名绘图](https://github.com/davisjiahao/davisjiahao.github.io/blob/main/_data/assets/img/service-center.png)

# 库存中心

## 交互时序图

### 库存缓存交互时序图

#### ~~方案1(缓存可用兑换码)~~

```mermaid
sequenceDiagram
    participant xxlJob as 定时任务
    participant store as 库存中心
    participant config as 服务配置
    participant redis as redis缓存存储
    participant mysql as mysql持久化存储
xxlJob->>store: 定时任务(每天凌晨2点)触发库存缓存更新
activate store
store->>config: 获取兑换码发放渠道为【库存下发】的处于已上线的服务列表serviceIds
config->>store: 响应：serviceIds
loop serviceId in serviceIds
    par 并行更新服务库存缓存
        store->>mysql: 根据serviceId查询对应服务库存下【已生效&未过期&可使用状态】的兑换码列表(按失效时间从小到大排序) redeemCodeList(item: ${code}_${endTime})
        mysql->>store: 返回兑换码列表redeemCodeList(item: ${code}_${endTime})
        store->>redis: Lua脚本(LTRIM minera:store_${serviceId} 1 0 // 清空 LPUSH store_${serviceId} redeemCodeList} // 重新导入)
        redis->>store: 缓存结果
    end
end
store-)xxlJob: 定时任务执行结束
deactivate store
```

#### 方案2(缓存可用兑换码总数)

```mermaid
sequenceDiagram
    participant xxlJob as 定时任务
    participant store as 库存中心
    participant config as 服务配置
    participant redis as redis缓存存储
    participant mysql as mysql持久化存储
xxlJob->>store: 定时任务(每天凌晨2点)触发库存缓存更新
activate store
store->>config: 获取兑换码发放渠道为【库存下发】的处于已上线的服务列表serviceIds
config->>store: 响应：serviceIds
loop serviceId in serviceIds
    par 并行更新服务库存缓存
        store->>mysql: 根据serviceId查询对应服务库存下【已生效&未过期&可使用状态】的兑换码总数
        mysql->>store: 返回兑换码总数codeSum
        store->>redis: 执行 [SET minera:store_${serviceId}_sum ${codeSum}]
        redis->>store: 缓存结果
    end
end
store-)xxlJob: 定时任务执行结束
deactivate store
```

方案3(无需缓存) 创建库存账户表(创建时机：首次创建库存或者创建服务)，直接加锁核销

### 库存核销交互时序图

#### 方案1-(缓存可用兑换码)

```mermaid
sequenceDiagram
    participant service as 服务中心
    participant config as 服务配置
    participant store as 库存中心
    participant redis as redis缓存存储
    participant mysql as mysql持久化存储
    participant mq as ddmq
service->>config: 查询已上架服务配置(入参：serviceId)
config->>service: 服务配置serviceConfig
opt serviceConfig 兑换码发放渠道为【库存下发】
    activate store
    service->>store: 服务使用(入参：serviceId、amount、endTimeThreshold、orderId)
    store->>redis: Lua脚本(for ${code}_${endTime} in LPOP minera:store_${serviceId}, if endTime > endTimeThreshold return code} // 重新导入)
    redis->>store: 响应：codeList(兑换码数组)
    opt codeList.size() != amount
        note over store,mysql: 【缓存兑换码(子流程复用)&获取兑换码】
        store->>redis: 缓存兑换码(子流程复用)&获取兑换码
        redis->>store: 响应：codeList(兑换码数组)
    end
    alt codeList.size() != amount
        store->>service: 响应:ERROR(库存不足!)
    else
       store->>mysql: 插入库存订单记录(唯一健: orderId+serviceId)
       mysql->>store: 插入结果
       alt 重复插入
          store->>mysql: 根据库存订单查询核销明记录获取券码列表
          mysql->>store: 券码列表
          store->>service: 响应:SUCCESS(核销成功)和码值列表
          store--)mq: 库存核销成功的消息
       else 插入失败(网络原因)
           note over store, mysql: 暂时不回滚 redis中的券码库存，不会导致超发就行
           store->>service: 响应:ERROR(核销失效)
       else 插入成功
           activate store
           store-->mysql: 批量插入库存核销明细记录，并更新codeList对应的兑换码记录状态为已使用
           mysql-->store: 数据库记录操作结果
           alt 均是重复插入或者均插入成功
              store->>service: 响应:SUCCESS(核销成功)和码值列表
              store--)mq: 库存核销成功的消息
           else
              store->>service: 响应:ERROR(核销失效)
       end
       deactivate store   
       end
    end
    deactivate store
end
```

#### 方案2(缓存可用兑换码总数)

```mermaid
sequenceDiagram
    participant service as 服务中心
    participant config as 服务配置
    participant store as 库存中心
    participant redis as redis缓存存储
    participant mysql as mysql持久化存储
    participant mq as ddmq
service->>config: 查询已上架服务配置(入参：serviceId)
config->>service: 服务配置serviceConfig
opt serviceConfig 兑换码发放渠道为【库存下发】
    activate store
    service->>store: 服务使用(入参：serviceId、amount、endTimeThreshold、orderId)
    store->>redis: Lua脚本(codeSum=`GET minera:store_${serviceId}_sum` <br/> if codeSum<amount <br/> {`SET minera:store_${serviceId}_sum codeSum-amount` <br/> `return 'OK'`} <br/> else { `return 'NO'`})
    redis->>store: 响应：flag
    alt flag != OK
       store->>service: 响应:ERROR(库存不足!)
    else
       store->>mysql: 插入库存订单记录(唯一健: orderId+serviceId)
       mysql->>store: 插入结果
       alt 重复插入
          store->>mysql: 根据库存订单查询核销明记录获取券码列表
          store--)mq: 库存核销成功的消息
       else 插入失败(网络原因)
           note over store, mysql: 暂时不回滚 redis中的总库存，不会导致超发就行
           store->>service: 响应:ERROR(核销失效)
       else 插入成功
           note over store,mysql: 修改兑换码状态应该异步，同步效率太低
           store->>mq: 库存核销操作消息
           store->>service: 响应:SUCCESS(核销成功)
       end
       activate store
       mq--)store: 库存核销操作消息
       note over store, mysql: 这里不能直接update兑换码状态(通过serviceId更新接近锁全表了)，先暂时用分布式锁串行处理,但是mq消费需要限流保护一波
       store-->redis: redis.lock(minera:${serviceId}_lock, 3s)
       store-->mysql: 查找满足条件的券码
       store-->mysql: 批量插入库存核销记录，并更新codeList对应的兑换码记录状态为已使用
       mysql-->store: 数据库记录操作结果
       alt 均是重复插入或者均插入成功
          store--)mq: 库存核销成功的消息
          store-->mysql: 更新库存订单记录为成功
       else
           store-->mysql: 更新库存订单记录为失败
       end
       deactivate store
    end
    deactivate store
end
```

方案3(无需缓存) 创建库存账户表(创建时机：首次创建库存或者创建服务)，直接加锁核销

### ~~方案对比~~

| ~~方案描述~~                   | ~~实现成本~~ | ~~并发性(券码状态更新)~~ | ~~是否会超发~~ |
| ------------------------------ | ------------ | ------------------------ | -------------- |
| ~~方案一(缓存可用兑换码)~~     | ~~较高~~     | ~~较高~~                 | ~~否~~         |
| ~~方案二(缓存可用兑换码总数)~~ | ~~较低~~     | ~~低~~                   | ~~否~~         |

~~考虑到系统稳定性，暂定采用方案二~~

### 库存核增交互时序图

```mermaid
sequenceDiagram
    participant service as 服务中心
    participant config as 服务配置
    participant store as 库存中心
    participant redis as redis缓存存储
    participant mysql as mysql持久化存储
service->>config: 查询已上架服务配置(入参：serviceId)
config->>service: 服务配置serviceConfig
opt serviceConfig 兑换码发放渠道为【库存下发】
    service->>store: 库存核增(入参：serviceId、codeList、orderId)
    store->>mysql: 直接插入(insert ignore)库存核增订单记录(action=核增、status=INIT)
    mysql->>store: 响应：insertRes
    alt insertRes = 0
      store->>service: 幂等响应：SUCCESS【核增成功】
    else insertRes = 1
      store->>mysql: begin 事务
      store->>mysql: 批量修改codeList对应的兑换码记录状态为未使用
      store->>redis: 这里更新redis库存缓存(在头部插入券码或者在总数上累加)
      store->>mysql: 修改库存核增订单记录状态为SUCCESS
      mysql->>store: commit 事务
      store->>service: 幂等响应：SUCCESS【核增成功】
    end
end
```



### 库存采购交互时序图

```mermaid
sequenceDiagram
    participant mis
    participant store as 库存中心
    participant gift as 文件存储
    participant redis as redis缓存存储
    participant mysql as mysql持久化存储
mis->>store: 生成兑换码(serviceId, type=csv或者gen)
activate store
alt type=csv
    store->>gift: 获取csv文件
    store->>mysql: 插入库存采购记录(type、csv url、status=INIT)
    store->>store: 多线程解析csv的兑换码文件
    store->>mysql: 批量插入兑换码库存记录(status=未使用，采购批次号)
    store->>mysql: 修改库存采购记录状态和统计结果(status=SUCCESS, 总条数，成功条数，失败条数)
    store->>redis: 调用库存刷新子流程
else type=gen
    store->>gift: 批量生成兑换码值
    store->>mysql: 插入库存采购记录(type、status=INIT)
    store->>mysql: 批量插入兑换码库存记录(status=未使用，采购批次号)
    store->>mysql: 修改库存采购记录状态和统计结果(status=SUCCESS, 总条数，成功条数，失败条数)
    store->>redis: 调用库存刷新子流程
end
deactivate store
```

### 库存过期时序图

```mermaid
sequenceDiagram
    participant xxlJob as 定时任务
    participant store as 库存中心
    participant redis as redis缓存存储
    participant mysql as mysql持久化存储
xxlJob->>store: 库存过期定时任务(每天零点1分)
store->>mysql: 根据id分页查询处于未使用并且过期时间小于当前时间的库存兑换码记录
store->>mysql: 根据id依次更新对应库存兑换码记录的状态为已过期
store->>redis: 调用库存刷新子流程
store->>xxlJob: 库存过期完成
```



### 库存监控时序图

```mermaid
sequenceDiagram
    participant xxlJob as 定时任务
    participant store as 库存中心
    participant config as 服务配置
    participant redis as redis缓存存储
    participant mysql as mysql持久化存储
xxlJob->>store: 库存监控定时任务(每天早上9点)
store->>config: 获取兑换码下发渠道为渠道发放的已上线服务列表 serviceIds
loop serviceId in serviceIds
     store->>redis: 获取redis缓存中的serviceId对应库存总数 codeSum1
     store->>mysql: 获取mysql中对应serviceId并且处于生效中&可使用的库存总数 codeSum2
end
opt min(codeSum1, codeSum2) < 10000
   store->>store: 发送报警邮件&发送DC 
end
```

### 库存查询时序图

```mermaid
sequenceDiagram
    participant mis
    participant store as 库存中心
    participant config as 服务配置
    participant gift as gift文件存储
    participant mysql as mysql持久化存储
note over mis, store: 接口1-查询服务库存聚合列表
mis->>store: 查询服务库存聚合列表
store->>config: 分页查询满足搜索条件的serviceIds
store->>mysql: 查询库存记录表(group by serviceId,status where serviceId in serviceIds)
mysql->>store: 服务库存聚合列表
store->>mis: 查询服务库存聚合列表

note over mis, store: 接口2-查询服务库存详情信息
mis->>store: 查询服务库存详情信息(入参：serviceIds)
store->>mysql: 查询库存记录表(group by serviceId,status where serviceId in serviceIds)<br/>获取库存聚合信息
store->>config: 根据serviceIds查询服务配置信息
store->>store: 聚合库存聚合信息和服务配置信息
store->>mis: 服务库存聚合聚合信息

note over mis, store: 接口3-查询服务库存采购列表信息
mis->>store: 查询服务库存详情信息(入参：serviceId)
store->>mysql: 查询库存采购表获取列表信息
store->>mis: 库存采购列表信息

note over mis, store: 接口4-查询并下载服务库存明细csv文件
mis->>store: 查询服务库存详情信息(入参：serviceId、采购批次号)
store->>mysql: 分页查询服务库存明细记录
store->>store: 写明细记录到本地临时磁盘文件
store->>gift: 上传csv到gift
store->>mis: csv 的gift下载链接
```

## 数据库模型

#### 商品库存账户表

|   字段名    | 字段类型 |        默认值         |    是否主键     |                             备注                             |    示例    |
| :---------: | :------: | :-------------------: | :-------------: | :----------------------------------------------------------: | :--------: |
|     id      |  bigint  |           0           |       是        |                           自增主键                           |     0      |
|     sku     |  String  |          ''           | 否<br/>唯一索引 | 分类：手机<br/>spu：苹果 6<br/>sku：土豪金 128g 苹果 6 <br/>目前直接取值serviceId |   12345    |
|  sku_name   |  String  |          ''           |       否        |                           sku名称                            | '重疾绿通' |
| useable_cnt |   Int    |           0           |       否        |                         当前可用数量                         |            |
|   status    |   int    |           0           |       否        |                           商品状态                           |   0-正常   |
| create_time | datetime |  '1971-01-01 00:00'   |       否        |                           创建时间                           |            |
| update_time | datetime | '1971-01-01 00:00:00' |       否        |                           修改时间                           |            |

#### 商品库存明细表[store_detail_record]

暂时不分表，后续可以通过 serial_number分表

|      字段名       | 字段类型 |        默认值         |               是否主键               |                             备注                             |    示例    |
| :---------------: | :------: | :-------------------: | :----------------------------------: | :----------------------------------------------------------: | :--------: |
|        id         |  bigint  |           0           |                  是                  |                           自增主键                           |     0      |
| purchase_batch_id |  bigint  |           0           |               普通索引               |                          采购批次id                          |    123     |
|     sku_name      |  Strng   |          ''           |                  否                  |                           sku名称                            | '重疾绿通' |
|        sku        |  String  |          ''           |  sku_serial_number<br/>联合唯一索引  | 分类：手机<br/>spu：苹果 6<br/>sku：土豪金 128g 苹果 6 <br/>目前直接取值serviceId |   12345    |
|   serial_number   |  String  |          ''           |  sku_serial_number<br/>联合唯一索引  |               商品序列号<br />**<u>加密</u>**                |    1234    |
| serial_number_id  |  bigint  |          ''           |          唯一索引(业务主键)          |                         商品序列号id                         |   12345    |
|       extra       |   text   |                       |                  否                  |                         其它额外信息                         |            |
|      status       |   int    |           0           |                  否                  |         状态<br/>0-未下发<br/>1-已下发<br/>2-已过期          |            |
|  effective_time   | datetime | '1971-01-01 00:00:00' | invalid_time_effective_time 普通索引 |                           生效时间                           |            |
|   invalid_time    | datetime | '1971-01-01 00:00:00' | invalid_time_effective_time 普通索引 |                           失效时间                           |            |
|    create_time    | datetime | '1971-01-01 00:00:00' |                  否                  |                           创建时间                           |            |
|    update_time    | datetime | '1971-01-01 00:00:00' |                  否                  |                           修改时间                           |            |

#### 商品库存采购批次表[store_purchase_record]

|      字段名       | 字段类型 |        默认值         | 是否主键 |                             备注                             |                 示例                 |
| :---------------: | :------: | :-------------------: | :------: | :----------------------------------------------------------: | :----------------------------------: |
|        id         |  bigint  |           0           |    是    |                           自增主键                           |                  0                   |
| purchase_batch_id |  bigint  |           0           | 业务主键 |                          采购批次id                          |                 123                  |
|   purchase_type   |   int    |           0           |    否    |            采购方式<br />0-csv导入<br />1-自生成             |                  1                   |
|     sku_name      |  Strng   |          ''           |    否    |                   sku名称<br />serviceName                   |              '重疾绿通'              |
|        sku        |  String  |          ''           |    否    | 分类：手机<br/>spu：苹果 6<br/>sku：土豪金 128g 苹果 6 <br/>serviceId |                12345                 |
|      status       |   int    |           0           |    否    |           状态<br/>0-初始化-<br/>1-成功<br/>2-失败           |                  0                   |
|      all_cnt      |   int    |           0           |    否    |                          应导总条数                          |                12345                 |
|    success_cnt    |   int    |           0           |    否    |                        实际导入成功数                        |                 123                  |
|       extra       |   text   |                       |    否    |                         其它额外信息                         | {"csvUrl":"https://123456789/a.csv"} |
|     operator      |  string  |          ''           |    否    |                            操作人                            |                                      |
|    create_time    | datetime | '1971-01-01 00:00:00' |    否    |                           创建时间                           |                                      |
|    update_time    | datetime | '1971-01-01 00:00:00' |    否    |                           修改时间                           |                                      |

### 商品库存操作流水表[store_flow_record]

暂时不分表，后续可以通过 serial_number分表

|      字段名       | 字段类型 |        默认值         |                         是否主键                         |                             备注                             |    示例    |
| :---------------: | :------: | :-------------------: | :------------------------------------------------------: | :----------------------------------------------------------: | :--------: |
|        id         |  bigint  |           0           |                            是                            |                           自增主键                           |     0      |
|  order_record_id  |  bigint  |           0           |                       业务唯一主键                       |                         业务唯一主键                         |     0      |
| source_identifier |   int    |           0           | source_trade_no_source_identifier_sku <br />联合唯一索引 |                          业务方标识                          |    123     |
|  source_trade_no  |  strng   |          ''           | source_trade_no_source_identifier_sku <br />联合唯一索引 |                     业务方编号，用于幂等                     | '重疾绿通' |
|     sku_name      |  Strng   |          ''           |                            否                            |                   sku名称<br />serviceName                   | '重疾绿通' |
|        sku        |  string  |          ''           | source_trade_no_source_identifier_sku <br />联合唯一索引 | 分类：手机<br/>spu：苹果 6<br/>sku：土豪金 128g 苹果 6 <br/>目前直接取值serviceId |   12345    |
|    action_type    |   int    |           0           |                            否                            |              操作类型：<br />0-核销<br />1-核增              |     0      |
|      amount       |   int    |           0           |                            否                            |                             数量                             |    1234    |
|       extra       |   text   |                       |                            否                            |                         其它额外信息                         |            |
|      status       |   int    |           0           |                            否                            |           状态<br/>0-初始化<br/>1-成功<br/>2-失败            |            |
|    create_time    | datetime | '1971-01-01 00:00:00' |                            否                            |                           创建时间                           |            |
|    update_time    | datetime | '1971-01-01 00:00:00' |                            否                            |                           修改时间                           |            |

### 商品库存操作流水明细表[store_flow_detail_record]

暂时不分表，后续可以通过 serial_number分表

|         字段名         | 字段类型 |        默认值         |                         是否主键                         |                             备注                             |    示例    |
| :--------------------: | :------: | :-------------------: | :------------------------------------------------------: | :----------------------------------------------------------: | :--------: |
|           id           |  bigint  |           0           |                            是                            |                           自增主键                           |     0      |
|    order_record_id     |  bigint  |           0           |                         普通索引                         |                      库存操作流水记录id                      |     0      |
| order_record_detail_id |  bigint  |           0           |                       业务唯一主键                       |                    库存操作流水明细记录id                    |            |
|   source_identifier    |   int    |           0           | source_trade_no_source_identifier_sku <br />联合普通索引 |                          业务方标识                          |    123     |
|    source_trade_no     |  strng   |          ''           | source_trade_no_source_identifier_sku <br />联合普通索引 |                     业务方编号，用于幂等                     | '重疾绿通' |
|        sku_name        |  Strng   |          ''           |                            否                            |                   sku名称<br />serviceName                   | '重疾绿通' |
|          sku           |  string  |          ''           | source_trade_no_source_identifier_sku <br />联合普通索引 | 分类：手机<br/>spu：苹果 6<br/>sku：土豪金 128g 苹果 6 <br/>目前直接取值serviceId |   12345    |
|      action_type       |   int    |           0           |                            否                            |              操作类型：<br />0-核销<br />1-核增              |     0      |
|    serial_number_id    |  bigint  |           0           |                         普通索引                         |                         商品序列号id                         |    1234    |
|         extra          |   text   |                       |                            否                            |                         其它额外信息                         |            |
|         status         |   int    |           0           |                            否                            |           状态<br/>0-初始化<br/>1-成功<br/>2-失败            |            |
|      create_time       | datetime | '1971-01-01 00:00:00' |                            否                            |                           创建时间                           |            |
|      update_time       | datetime | '1971-01-01 00:00:00' |                            否                            |                           修改时间                           |            |

#### 实体关系图

```mermaid
erDiagram
    store_purchase_record o{--|| store_detail_record : generate
    store_flow_detail_record ||--o{ store_flow_record : generate
    store_flow_detail_record ||--|| store_detail_record : generate
   
```

#### 兑换码状态流转图

```mermaid
stateDiagram-v2
    未下发 --> 已过期: 过期操作
    未下发 --> 已下发: 核销
    已下发 --> 未下发: 核增
```







## 外部能力

#### 接口能力

##### 库存核销

- 入参：sku(serviceId)、source_identifier(库存中心分配)、source_trade_no(业务单号)、amount(核销数量)
- 出参：核销结果(成功/失败)，核销的兑换码通过消息异步返回

##### 库存核增

- 入参：sku(serviceId)、source_identifier(库存中心分配)、source_trade_no(业务单号)、serial_number_ids(核增的商品库存序列码id)
- 出参：核增结果(成功/失败)

##### 库存查询

- 入参：skuList
- 出参：各个商品可用库存数量数组

##### 库存聚合信息列表查询

- 入参：服务配置查询相关参数(服务名称、服务id、服务code、供应商)、分页参数
- 出参：各个商品库存总数、过期总数、已下发总数、未下发总数聚合信息数组

##### 库存采购操作列表记录查询

- 入参：服务id
- 出参：对应服务商品的库存采购操作记录列表

##### 库存兑换码明细导出

- 入参：serviceId(服务id) || purchase_batch_id(采购批次id)
- 出参：对应库存记录列表的csv文件的gift 下载地址

##### 采购库存(生成兑换码)

- 入参：采购类型(csv 导入||自生成)、csv文件git路径(或者生成数量)、库存生效&&失效时间
- 出参：csv 导入(导入结果明细)、自生成(成功||失败)

#### 消息能力

##### 核销成功消息

- 触发时机：兑换码库存核销成功(一个兑换码一条消息)
- 消息字段：sku(serviceId)、source_identifier(库存中心分配)、source_trade_no(业务单号)、serial_number_id、serial_number

## 外部依赖

### 接口依赖

#### 服务配置中心

##### 分页查询服务配置

- 入参：服务id、供应商、兑换码下发渠道等，分页参数
- 出参:  服务配置数组

## 工期(10.5PD)

### 前期准备 2PD

#### 接口文档(1.5PD)

- 库存核销
- 库存核增
- 库存查询
- 库存聚合信息列表查询
- 库存采购操作列表记录查询
- 库存兑换码明细导出
- 采购库存(生成兑换码)

#### 项目搭建(0.5PD)

### 开发(8.5PD)

- 库存缓存定时任务  1.5PD
- 库存过期定时任务  1PD

- 库存核销  1.5PD
- 库存核增  1PD
- 库存查询  0.25PD
- 库存聚合信息列表查询 0.25PD
- 库存采购操作列表记录查询 0.5PD
- 库存兑换码明细导出 1PD
- 采购库存(生成兑换码) 1.5PD



# 服务中心

## 系统交互时序图

### 服务发放/兑换交互时序图

```mermaid
sequenceDiagram
    participant up as 积分商城||活动中心||服务商城
    participant service as 服务中心
    participant config as 服务配置
    participant store as 库存中心
    participant gateway as 服务网关
    participant mysql as mysql持久化存储
up->>service: 积分兑换服务||命中活动发放礼品||保单承保, 入参：serviceId、caller、sourceTradeNo、sumAmount(总金额)、count(数量)、服务生效有效时间
service->>config: 查询服务配置(入参：serviceId)
config->>service: 服务配置serviceConfig
note over service,config: 不存在或者未上架提示错误
service-->service: 前置校验(sumAmount/count == 服务销售金额)
note right of service: 如果限制兑换次数，则查询用户是否有处于生效状态的订单记录，如果有则返回【已有可用服务】
note right of service: 如果兑换码发放渠道为库存发放，则查询该服务当前可用库存总数，如果当前库存总数 < count,返回【当前库存不足】
service-->mysql: 插入下单流水记录(status=init)
note over service,mysql: 进行幂等处理，根据下单流水id查找主/子订单信息返回给up
note over service,service: 先扣减库存，再创建订单
opt 兑换码下发节点为服务下单
  alt 发放渠道为三方调用
     note over service,gateway: 三方下发库存gateway会保证幂等
     note over service,gateway: 这里需要一个码一个码发放
     service-->gateway: 发放兑换码
     gateway-->service: 发放结果(成功/失败)，兑换码需要异步返回
  else 发放渠道为库存下发
     note over service,store: 库存下发是幂等的，可以重试，券码消息也会重新触发
     note over service,store: 这里支持批量发放
     service-->gateway: 发放兑换码
     gateway-->service: 发放结果(成功/失败)，兑换码需要异步返回
  end
end 
note over service,service: 需要插入count个主订单
loop count:
    service-->service: 计算服务主/子订单初始状态 initStatus
    note over service,config: 如果需要激活，initStatus=未激活；否则如果已生效，initStatus=已生效；否则如果待生效，则为待生效
    service-->mysql: 插入服务主订单记录(status=initStatus)
    service-->mysql: 批量插入服务子订单记录(status=initStatus)
end
service-->mysql: 修改下单流水记录状态(status=success)
```

#### 服务激活交互时序图



```mermaid
sequenceDiagram
    participant shop as 服务商城
    participant service as 服务中心
    participant config as 服务配置
    participant store as 库存中心
    participant gateway as 服务网关
    participant mysql as mysql持久化存储
    participant mq as ddmq
shop->>service: 激活服务, 入参：子订单id，serviceId
service->>config: 查询服务配置(入参：serviceId)
config->>service: 服务配置serviceConfig
note over service,config: 不存在或者未上架提示错误
opt 服务配置为无需激活
   service->>shop: 响应：ERROR【无需激活】
end
opt 兑换码下发节点为服务激活
  alt 发放渠道为三方调用
     note over service,gateway: 三方下发库存gateway会保证幂等
     service->>gateway: 发放兑换码
     gateway->>service: 发放结果(成功/失败)，兑换码需要异步返回
     gateway->>mq: 兑换码兑换结果消息
  else 发放渠道为库存下发
     note over service,store: 库存下发是幂等的，可以重试，券码消息也会重新触发
     service->>store: 发放兑换码
     store->>service: 发放结果(成功/失败)，兑换码需要异步返回
     store->>mq: 兑换码兑换结果消息
  end
end 
service->>shop: 响应：SUCCESS【激活成功】
```

### 服务使用交互时序图

```mermaid
sequenceDiagram
    participant shop as 服务商城
    participant service as 服务中心
    participant config as 服务配置
    participant store as 库存中心
    participant gateway as 服务网关
    participant mysql as mysql持久化存储
shop->>service: 使用服务, 入参：subOrderId，serviceId，其它参数(收货地址等)
service->>config: 查询服务配置(入参：serviceId)
config->>service: 服务配置serviceConfig
note over service,config: 不存在则提示错误
opt 服务配置为无需核销
   service->>shop: 响应：无需手动使用
end
service->>mysql: 插入服务核销单记录(status=进行中)
note over service, mysql: 重复插入幂等处理
service->>mysql: 更新子订单状态为【锁定中】
service->>mysql: 根据子订单状态更新主订单状态
service->>gateway: 下单
service->>shop: 响应：SUCCESS【下单成功】
```

### 服务回退交互时序图

```mermaid
sequenceDiagram
    participant up as 积分商城
    participant service as 服务中心
    participant config as 服务配置
    participant store as 库存中心
    participant mysql as mysql持久化存储
up->>service: 服务兑换积分(serviceId、sourceTradeNo、count(数量)、subOrderIds)
service->>config: 查询服务配置(入参：serviceId)
config->>service: 服务配置serviceConfig
note over service,config: 不存在则提示错误
opt 服务类型不为活动礼品
   service->>up: 该服务不支持回退
end
note over service, mysql: 先修改服务订单状态，再回退库存
service->>mysql: 修改subOrderIds对应子订单状态为已归还
service->>mysql: 修改subOrderIds对应子订单的主订单状态为已归还
service->>store: 库存回退
service->>shop: 响应：SUCCESS【回退成功】
```

### 

### 服务兑换码回调交互时序图

```mermaid
sequenceDiagram
    participant shop as 服务商城
    participant service as 服务中心
    participant config as 服务配置
    participant store as 库存中心
    participant gateway as 服务网关
    participant mysql as mysql持久化存储
    participant mq as ddmq
mq->>service: 兑换码兑换结果消息(serviceId、兑换码编号、subOrderId)
service->>config: 查询服务配置(入参：serviceId)
config->>service: 服务配置serviceConfig
alt 兑换码发放成功
   service->>mysql: 更新服务子订单的兑换码字段
else
   service->>service: 暂时先不考虑发放失败的服务回滚，直接先报警处理
end

```



### 服务核销回调交互时序图

```mermaid
sequenceDiagram
    participant service as 服务中心
    participant config as 服务配置
    participant store as 库存中心
    participant gateway as 服务网关
    participant mysql as mysql持久化存储
    participant mq as ddmq
mq->>service: 核销结果消息(duid、serviceId、兑换码编号、sourceTradeNo)
service->>config: 查询服务配置(入参：serviceId)
config->>service: 服务配置serviceConfig
note over service,mysql: 核销回调前可能不会和供应商有任何交互，有可能只能通过兑换码进行匹配
service-->mysql: 根据兑换码编号或者sourceTradeNo查找服务核销记录
alt 不存在核销记录
  service->>mysql: 直接插入核销状态为已使用的核销记录
else 存在核销记录
  service->>mysql: 更新核销记录的额外信息字段，进行merge操作
end
```



### 服务过期交互时序图

```mermaid
sequenceDiagram
    participant xxl as xxlJob
    participant service as 服务中心
    participant config as 服务配置
    participant store as 库存中心
    participant gateway as 服务网关
    participant mysql as mysql持久化存储

xxl->>service: 服务过期定时任务(每天00:01)
service->>mysql: 多线程分表分页扫描处于生效中&&失效时间小于当前时间的主订单记录列表
service->>mysql: 更新主订单记录状态为已失效
service->>mysql: 判断主订单下处于生效中状态的子订单的状态为已失效
```



### 服务单查询交互时序图

```mermaid
sequenceDiagram
    participant up as mis/资产中心/服务商城
    participant service as 服务中心
    participant config as 服务配置
    participant store as 库存中心
    participant gateway as 服务网关
    participant mysql as mysql持久化存储
    participant es as 搜索引擎
mysql--)es: 服务主订单记录、核销单记录数据同步
note over up,es: 接口1-分页查询主订单记录
up->>service: 分页查询主订单记录(入参：服务配置、用户信息)
service->>config: 条件查询所有的服务配置
alt 参数不包含duid
  service->>es: 分页条件查询主订单记录
else
  service->>mysql: 分页条件查询主订单记录
end
service->>up: 主订单列表记录

note over up,es: 接口2-查询主订单详情
up->>service: 查询主订单详情(duid、orderId)
service->>mysql: 查询主订单详情(duid、orderId)
service->>up: 主订单记录详情(额外信息字段)

note over up,es: 接口3-查询主订单下子订单列表
up->>service: 查询主订单下子订单列表(duid、orderId)
service->>mysql: 查询主订单下子订单列表(duid、orderId)
service->>up: 子订单列表

note over up,es: 接口4-分页查询核销列表记录
up->>service: 分页查询核销列表记录(入参：服务配置、用户信息)
service->>config: 条件查询所有的服务配置
alt 参数不包含duid
  service->>es: 分页查询核销列表记录
else
  service->>mysql: 分页查询核销列表记录
end
service->>up: 核销列表记录
```

### ~~服务单线下核销交互时序图~~



```mermaid
sequenceDiagram
    participant mis as mis
    participant service as 服务中心
    participant config as 服务配置
    participant mysql as mysql持久化存储
    participant gift as gift文件存储
    participant mq as ddmq消息队列

mis->>service: 上传csv进行线下核销(csv gift下载路径，serviceId)
service->>config: 查询服务配置(入参：serviceId)
opt 服务配置不支持线下核销
   service->>mis:ERROR(该服务不支持线下核销)
end
service->>mysql: 插入线下核销操作批次记录(status=INIT, 记录应该处理总数，当前处理总数)
service->>gift: 下载核销csv文件
service->>service: 解析核销csv文件，格式解析，获取券码列表
service->>mq: 发送核销消息(自产自销，复用已有核销消息回调处理子流程，每行数据需要有唯一标识运营同学需要自己添加)
note over service, mq: 每异步处理一条，累加已处理总数，当本批次都已经处理完成，则修改该批次状态为处理成功
service->>mysql: 插入线下核销操作批次记录(status=SUCCESS)
```

### 供应商跳转链接参数获取

```mermaid
sequenceDiagram
    participant service as 服务中心
    participant gateway as 服务网关

service->>gateway: 获取H5跳转链接的外部额外参数map, 入参：链接类型&兑换码&子订单号
gateway->>service: 额外跳转链接的额外参数map

```





### ~~服务单线下核销批次记录查询~~

单表查询，省略

### 账户注销

已有功能，按规范接入即可，带duid的表均需要进行归档处理

## 数据模型

### 数据库表模型

#### 服务订单操作流水记录表(service_order_flow_record_{0,1023})

按duid 分表 

|      字段名       | 字段类型 |        默认值         |                           是否主键                           |                             备注                             |    示例    |
| :---------------: | :------: | :-------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :--------: |
|        id         |  bigint  |           0           |                              是                              |                           自增主键                           |     0      |
|  flow_record_id   |  bigint  |           0           |                         唯一业务主键                         |                         业务唯一主键                         |            |
|       duid        |  bigint  |           0           | duid_source_trade_no_source_identifier_sku <br />联合唯一索引 |                             duid                             |     0      |
| source_identifier |   int    |           0           | duid_source_trade_no_source_identifier_sku <br />联合唯一索引 |                          业务方标识                          |    123     |
|  source_trade_no  |  strng   |          ''           | duid_source_trade_no_source_identifier_sku <br />联合唯一索引 |                     业务方编号，用于幂等                     | '重疾绿通' |
|     sku_name      |  Strng   |          ''           |                              否                              |                   sku名称<br />serviceName                   | '重疾绿通' |
|        sku        |  string  |          ''           | duid_source_trade_no_source_identifier_sku <br />联合唯一索引 | 分类：手机<br/>spu：苹果 6<br/>sku：土豪金 128g 苹果 6 <br/>目前直接取值serviceId |   12345    |
|       count       |   int    |           0           |                              否                              |                    本次下单/退还服务次数                     |            |
|    action_type    |   int    |           0           |                              否                              |              操作类型：<br />0-下单<br />1-退还              |     0      |
|       extra       |   text   |                       |                              否                              |                         其它额外信息                         |            |
|      status       |   int    |           0           |                              否                              |           状态<br/>0-初始化<br/>1-成功<br/>2-失败            |            |
|    create_time    | datetime | '1971-01-01 00:00:00' |                              否                              |                           创建时间                           |            |
|    update_time    | datetime | '1971-01-01 00:00:00' |                              否                              |                           修改时间                           |            |

#### 服务主订单记录表(service_order_{0,1023})

按duid 分表 

|         字段名         | 字段类型 |        默认值         |                         是否主键                          |                             备注                             |    示例    |
| :--------------------: | :------: | :-------------------: | :-------------------------------------------------------: | :----------------------------------------------------------: | :--------: |
|           id           |  bigint  |           0           |                            是                             |                           自增主键                           |     0      |
|        order_id        |  bigint  |           0           |                       唯一业务主键                        |                         业务唯一主键                         |            |
|          duid          |  bigint  |           0           | duid_source_trade_no_source_identifier_sku <br />普通索引 |                             duid                             |     0      |
|          uid           |  bigint  |           0           |                            否                             |                             uid                              |            |
|          role          |   int    |           0           |                            否                             |                             role                             |            |
|   source_identifier    |   int    |           0           | duid_source_trade_no_source_identifier_sku <br />普通索引 |                          业务方标识                          |    123     |
|    source_trade_no     |  strng   |          ''           | duid_source_trade_no_source_identifier_sku <br />普通索引 |                     业务方编号，用于幂等                     | '重疾绿通' |
|        sku_name        |  Strng   |          ''           |                            否                             |                   sku名称<br />serviceName                   | '重疾绿通' |
|          sku           |  string  |          ''           | duid_source_trade_no_source_identifier_sku <br />普通索引 | 分类：手机<br/>spu：苹果 6<br/>sku：土豪金 128g 苹果 6 <br/>目前直接取值serviceId |   12345    |
|       all_count        |   Int    |           0           |                            否                             |              可兑换使用总次数<br />-1：不限次数              |            |
|         extra          |   text   |                       |                            否                             |                         其它额外信息                         |            |
|         status         |   int    |           0           |                            否                             |    1-已回收<br /> 2-待使用 <br />3-已使用 4<br />-已失效     |            |
| service_effective_time | datetime | '1971-01-01 00:00:00' |                            否                             |                         服务生效时间                         |            |
|  service_invalid_time  | datetime | '1971-01-01 00:00:00' |                            否                             |                         服务失效时间                         |            |
| useable_effective_time | datetime | '1971-01-01 00:00:00' |                            否                             |                        可兑换生效时间                        |            |
| useable_effective_time | datetime | '1971-01-01 00:00:00' |                            否                             |                        可兑换失效时间                        |            |
|      create_time       | datetime | '1971-01-01 00:00:00' |                            否                             |                           创建时间                           |            |
|      update_time       | datetime | '1971-01-01 00:00:00' |                            否                             |                           修改时间                           |            |

#### 服务子订单记录表(service_sub_order_{0,1023})

按duid 分表 

|      字段名       | 字段类型 |        默认值         |                           是否主键                           |                             备注                             |    示例    |
| :---------------: | :------: | :-------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :--------: |
|        id         |  bigint  |           0           |                              是                              |                           自增主键                           |     0      |
|   sub_order_id    |  bigint  |                       |                         唯一业务主键                         |                         唯一业务主键                         |            |
|     order_id      |  bigint  |           0           |                           普通索引                           |                         业务唯一主键                         |            |
|       duid        |  bigint  |           0           | duid_source_trade_no_source_identifier_sku <br />联合普通索引 |                             duid                             |     0      |
|        uid        |  bigint  |           0           |                              否                              |                             uid                              |            |
|       role        |   int    |           0           |                              否                              |                             role                             |            |
| source_identifier |   int    |           0           | duid_source_trade_no_source_identifier_sku <br />联合普通索引 |                          业务方标识                          |    123     |
|  source_trade_no  |  strng   |          ''           | duid_source_trade_no_source_identifier_sku <br />联合普通索引 |                          业务方编号                          | '重疾绿通' |
|     sku_name      |  Strng   |          ''           |                              否                              |                   sku名称<br />serviceName                   | '重疾绿通' |
|        sku        |  string  |          ''           | duid_source_trade_no_source_identifier_sku <br />联合唯一索引 | 分类：手机<br/>spu：苹果 6<br/>sku：土豪金 128g 苹果 6 <br/>目前直接取值serviceId |   12345    |
|   serial_number   |  String  |          ''           |                  serial_number<br/>普通索引                  |               商品序列号<br />**<u>加密</u>**                |    1234    |
|       extra       |   text   |                       |                              否                              |                         其它额外信息                         |            |
|      status       |   int    |           0           |                              否                              | 1、已回收<br />2、未激活（服务配置为需要激活，用户未激活时） <br />3、待生效 <br />4、已生效 <br />5、已使用 <br />6、锁定中（核销下单后，有一段时间才能返回核销结果，此时服务锁定中，禁止再下单） <br />7、已失效（服务兑换有效期结束且用户未兑换服务 or 服务有效期结束 or 保单状态变更时） |            |
|    create_time    | datetime | '1971-01-01 00:00:00' |                              否                              |                           创建时间                           |            |
|    update_time    | datetime | '1971-01-01 00:00:00' |                              否                              |                           修改时间                           |            |

#### 服务使用记录表(service_use_record_{0,1023})

按duid 分表 

|      字段名       | 字段类型 |        默认值         |                           是否主键                           |                             备注                             |    示例    |
| :---------------: | :------: | :-------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :--------: |
|        id         |  bigint  |           0           |                              是                              |                           自增主键                           |     0      |
|   use_record_id   |  bigint  |           0           |                         唯一业务主键                         |                         业务唯一主键                         |            |
|       duid        |  bigint  |           0           |  duid_source_trade_no_source_identifier_sku <br />普通索引   |                             duid                             |     0      |
|        uid        |  bigint  |           0           |                              否                              |                             uid                              |            |
|       role        |   int    |           0           |                              否                              |                             role                             |            |
| source_identifier |   int    |           0           |  duid_source_trade_no_source_identifier_sku <br />普通索引   |                          业务方标识                          |    123     |
|  source_trade_no  |  strng   |          ''           |  duid_source_trade_no_source_identifier_sku <br />普通索引   |                     业务方编号，用于幂等                     | '重疾绿通' |
|        sku        |  string  |          ''           | duid_source_trade_no_source_identifier_sku <br />联合唯一索引 | 分类：手机<br/>spu：苹果 6<br/>sku：土豪金 128g 苹果 6 <br/>目前直接取值serviceId |   12345    |
|   serial_number   |  String  |          ''           |                  serial_number<br/>普通索引                  |               商品序列号<br />**<u>加密</u>**                |    1234    |
|       extra       |   text   |                       |                              否                              |                         其它额外信息                         |            |
|      status       |   int    |           0           |                              否                              | 进行中（比如拖车服务，下了核销单后，需要一段时间才能完成服务） <br />已完成 <br />已取消（当服务单下单后，用户没有用，取消了） |            |
|    create_time    | datetime | '1971-01-01 00:00:00' |                              否                              |                           创建时间                           |            |
|    update_time    | datetime | '1971-01-01 00:00:00' |                              否                              |                           修改时间                           |            |

#### ~~服务线下核销批次记录表~~

### 状态流转图

![主子订单状态流转图](/Users/didi/Downloads/clipboard_image_1665717365080.png)

#### 主订单状态流转

| 子订单状态 | 判断条件                | 主订单状态 |
| :--------- | :---------------------- | :--------- |
| 已回收     | ALL 子订单 = 已回收状态 | 已回收     |
| 未激活     | 子订单未激活个数≥1      | 待使用     |
| 待生效     | 子订单待生效个数≥1      | 待使用     |
| 已生效     | 子订单已生效个数≥1      | 待使用     |
| 已使用     | 子订单已使用个数≥1      | 已使用     |
| 锁定中     | 子订单锁定中个数≥1      | 已使用     |
| 已失效     | ALL 子订单 = 已失效状态 | 已失效     |

#### 子订单状态流转图

```mermaid
stateDiagram-v2
    * --> 未激活: 服务下单&&需要激活
    * --> 待生效: 服务下单&&不需要激活&&还未生效
    * --> 生效中: 服务下单&&不需要激活&&已生效
    生效中 --> 已使用: 收到兑换码消息&&无需核销&&非保单类服务
    未激活 --> 已使用: 收到激活成功消息&&无需核销
    未激活 --> 待生效: 收到激活成功消息&&还未生效
    未激活 --> 生效中: 收到激活成功消息&&当前服务生效中
    未激活 --> 已失效: 收到激活成功消息&&当前服务已失效
    待生效 --> 生效中: 当前服务已生效
    生效中 --> 锁定中: 非不限次数使用&&服务使用中
    锁定中 --> 已使用: 非不限次数使用&&核销完成
    生效中 --> 已失效: 当前服务已失效
    生效中 --> 已回收: 退货
```

#### 核销单状态流转图

```mermaid
stateDiagram-v2
    * --> 进行中: 服务使用
    * --> 已完成: 服务直接核销
    * --> 已取消: 暂不考虑
    进行中 --> 已完成: 服务核销
    进行中 --> 已取消: 暂不考虑
    
```



## 外部能力

### service能力

- 批量兑换/发放服务

  入参：serviceId、唯一业务单号、sumAmount(服务总金额)、sumCount(服务总数量)

  响应：兑换结果

- 激活服务

  入参：serviceId、subOrderId

  响应：激活结果

- 批量退还服务

  入参：serviceId、subOrderIdList

  响应：退还结果

- 获取三方供应商H5额外链接参数

  入参：serviceId、兑换码、subOrderId

  响应：链接参数Map

### 接口能力

- 4 个基本查询接口

  入参：serviceId、兑换码、subOrderId

  响应：链接参数Map

### 消息能力

 无，暂无需求

## 外部依赖

### 接口依赖

####  服务配置中心

- 条件查询所有服务配置

  入参：服务配置字段

  响应：服务配置列表

####  库存中心

- 库存核销
- 库存核增
- 服务可用库存查询

####  服务网关

- 兑换码获取
- 服务使用
- H5链接额外参数获取

### 消息依赖

####  库存中心

- 兑换码发放结果消息

####  服务网关

- 兑换码发放结果消息
- 核销回调消息

## 排期(9.5PD)

### 前期准备 0.5PD

#### 接口文档(10.5PD)

- 4 个基本查询接口 0.5PD
- 库存核增
- 库存查询
- 库存聚合信息列表查询
- 库存采购操作列表记录查询
- 库存兑换码明细导出
- 采购库存(生成兑换码)

#### 项目搭建(0PD)

### 开发(10PD)

- 批量兑换/发放服务  1.5PD
- 激活服务 0.5PD
- 批量退还服务 1.5PD
- 获取三方供应商H5额外链接参数  0.5PD
- 兑换码发放回调消息处理  1PD
- 核销回调消息处理  1.5PD
- 订单过期 1PD
- 4个基本查询接口 1.5PD
- 账户注销处理(1PD)

# 服务商城

##  数据模型

### 服务类页面配置模型

#### 字典配置

##### 文本字体大小

| 标识(key) | 描述     | value | 效果示例图片 |
| --------- | -------- | ----- | ------------ |
| Key1      | 通用字体 | 5号   | http://1.img |
| Key2      | 5号字体  | 5     | http://1.img |
| Key3      | 6号字体  | 6     | http://1.img |

##### 文本颜色

| 标识(key) | 描述     | value | 效果示例图片 |
| --------- | -------- | ----- | ------------ |
| Key1      | 通用颜色 | black | http://1.img |
| Key2      | 5号字体  | red   | http://1.img |
| Key3      | 6号字体  | blue  | http://1.img |

##### 字段取值配置

| 标识(key) | 描述                    | value               | 来源标识 |
| --------- | ----------------------- | ------------------- | -------- |
| Key1      | 主订单号                | ${data.orderId}     | Func1    |
| Key2      | 保险公司名称            | ${data.companyName} | Func1    |
| Key3      | 生效日期-yyyyMMdd HH:ss | ${data.startTime}   | Func2    |

#### 控件配置

##### 动态文本[DynTitle]

+Font{360.font, }+积分

| 字段名     | 字段类型 | 是否必填 | 备注                                           | 示例值           |
| ---------- | -------- | -------- | ---------------------------------------------- | ---------------- |
| id         | int      | 是       | 控件类型                                       | 1                |
| titleType  | int      | 否       | 文本类型<br />1-普通文本，默认<br />2-动态取值 | 2                |
| titleValue | String   | 是       | 普通文本，或者字段取值标识                     | ”你好“<br />Key1 |
| fontSize   | String   | 否       | 字体大小                                       | Key1             |
| fontColor  | String   | 否       | 字体颜色                                       | Key1             |

##### 动态链接[DynLink]

| 字段名    | 字段类型            | 是否必填 | 备注                                           | 示例值           |
| --------- | ------------------- | -------- | ---------------------------------------------- | ---------------- |
| id        | int                 | 是       | 控件类型                                       | 1                |
| title     | String              | 是       | 链接文本                                       |                  |
| type      | int                 | 否       | 文本类型<br />1-普通链接，默认<br />2-动态取值 | 2                |
| preFixUrl | String              | 是       | 固定前缀域名                                   | ”你好“<br />Key1 |
| params    | map<String，string> | 否       | 动态参数取值，来源于字段取值配置表             |                  |

服务按钮(ServiceButtun)

| 字段名             | 字段类型              | 是否必填 | 备注                                                         | 示例值           |
| ------------------ | --------------------- | -------- | ------------------------------------------------------------ | ---------------- |
| id                 | int                   | 是       | 控件类型                                                     | 2                |
| actionType         | int                   | 是       | 按钮类型<br />1-跳转链接<br />2-动态获取链接并跳转<br />3-置换积分-接口调用<br />4-复制券码并跳转链接<br />5-复制券码并动态跳转链接<br />6-电话预约<br />7-兑换服务-接口调用<br />8-发放券码-接口调用<br />9-激活服务-接口调用 | 2                |
| actionValue        | String/List<DynTitle> | 否       | url(链接类)或者接口标识(接口类)或者弹窗文本(电话预约类)      |                  |
| buttunTitle        | List<DynTitle>        | 是       | 按钮文案<br />动态文本                                       | ”你好“<br />Key1 |
| titleTopOverButtun | List<DynTitle>        | 是       | 按钮上方文案<br />动态文本                                   | ”你好“<br />Key1 |
| 图标字段           |                       |          |                                                              |                  |
| buttunImg          | String                | 是       | 按钮图片                                                     |                  |
| toastBeforeAction  | List<DynTitle>        | 否       | 按钮动作触发前Toast提示文本<br />动态文本                    | ”你好“<br />Key1 |
| toastAfterAction   | List<DynTitle>        | 否       | 按钮动作触发后Toast提示文本<br />动态文本                    | ”你好“<br />Key1 |

#### 服务类组件

#####  图片组件(Image)

| 字段名 | 字段类型 | 是否必填 | 备注     | 示例值 |
| ------ | -------- | -------- | -------- | ------ |
| id     | int      | 是       | 组件类型 | 45     |
| imgUrl | String   | 是       | 图片url  |        |
| order  | int      | 是       | 顺序     |        |

##### 标题组件(newTitle)

| 字段名 | 字段类型       | 是否必填 | 备注                            | 示例值 |
| ------ | -------------- | -------- | ------------------------------- | ------ |
| id     | int            | 是       | 组件类型                        | 46     |
| order  | int            | 是       | 顺序                            | 顺序   |
| title  | List<DynTitle> | 是       | titleType为动态文本时，需要解析 |        |

##### 通用悬底按钮区(CommonButtonArea)

| 字段名          | 字段类型            | 是否必填 | 备注             | 示例值 |
| --------------- | ------------------- | -------- | ---------------- | ------ |
| id              | int                 | 是       | 组件类型         | 47     |
| order           | int                 | 是       | 顺序             | 顺序   |
| titleLeftButton | List<DynTitle>      | 否       | 按钮右侧展示文案 | 1      |
| buttons         | List<ServiceButtun> | 是       | 按钮列表         |        |

##### 服务头图(serviceTopImg)

| 字段名    | 字段类型       | 是否必填 | 备注             | 示例值 |
| --------- | -------------- | -------- | ---------------- | ------ |
| id        | int            | 是       | 组件类型         | 47     |
| order     | int            | 是       | 顺序             | 顺序   |
| iconImg   | String         | 是       | 标题icon 图片url |        |
| img       | String         | 是       | 背景图           |        |
| title     | List<DynTitle> | 是       | 主标题           |        |
| subTitle  | List<DynTitle> | 是       | 子标题           |        |
| detailUrl | DynUrl         | 是       | 查看详情url      |        |

##### 服务使用结果(serviceUseRes)

|  字段名  |         字段类型          | 是否必填 |       备注       | 示例值 |
| :------: | :-----------------------: | :------: | :--------------: | :----: |
|    id    |            int            |    是    |     组件类型     |   47   |
|  order   |            int            |    是    |       顺序       |        |
|  title   |      List<DynTitle>       |    是    |      主标题      |        |
| subTitle |      List<DynTitle>       |    是    |      子标题      |        |
|  fields  | List<Map<String，String>> |    是    | 字段lable和value |        |

##### 券码展示(CodeShow)

| 字段名           | 字段类型 | 是否必填 | 备注     | 示例值 |
| ---------------- | -------- | -------- | -------- | ------ |
| id               | int      | 是       | 组件类型 | 47     |
| order            | int      | 是       | 顺序     |        |
| 是否需要复制券码 |          |          |          |        |

##### 服务使用记录列表(ServiceUseRecord)

| 字段名 | 字段类型                  | 是否必填 | 备注             | 示例值 |
| ------ | ------------------------- | -------- | ---------------- | ------ |
| id     | int                       | 是       | 组件类型         | 47     |
| order  | int                       | 是       | 顺序             |        |
| title  | String                    | 是       | 标题             |        |
| fields | List<Map<String，String>> | 是       | 字段lable和value |        |

#### 服务类页面类型

##### 服务详情页

标题组件(newTitle)

N(N>=1)个图片组件(Image)

##### 服务兑换页

标题组件(newTitle)

N(N>=1)个图片组件(Image)

通用悬底按钮区(CommonButtonArea)

##### 服务使用结果页

标题组件(newTitle)

服务结果头图

服务使用结果(serviceUseRes)

券码展示组件

服务使用记录组件

通用悬底按钮区(CommonButtonArea)

### 核心逻辑

#### 页面配置查询

先从页面配置系统中查询页面配置，聚合查询积分&服务配置&服务子订单详情信息组装为context，解析页面配置模块的各个组件为freeMake，根据context解析页面配置的freeMake为响应json，返回给前端

备选方案：json path取值一个个模块解析替换

## 外部能力

### 接口能力

1-服务页面配置查询

2-服务兑换

4-服务激活

5-服务下单

6-动态获取H5参数

### 消息能力

无

## 外部依赖

### 接口依赖

#### 积分商城

查询用户积分信息

#### 服务中心

查询用户子订单信息

#### 服务配置中心

计算服务起止期

计算服务兑换起止期

获取服务配置

### 消息依赖

保单状态变更消息：发放/过期保单类服务订单

## 排期 9.5PD

### 前期准备 2.5PD

#### 接口文档(2.5PD)

- 服务页面配置查询 1PD
- 服务兑换  0.5PD
- 服务激活  0.25PD
- 服务下单  0.5PD
- 获取动态跳转链接   0.25PD

#### 项目搭建(0PD)

### 开发(7PD)

- 服务页面配置查询 (组件、控件抽象、服务编排) 3PD
- 服务兑换  1PD
- 服务激活  0.5PD
- 服务下单  1 PD
- 获取动态跳转链接  0.5PD
- 保单状态变更消息处理  1 PD