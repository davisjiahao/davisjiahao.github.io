---
title: 流程编排那些事-最终曲-全声明式组件编排
author: daviswujiahao
date: 2023-05-06 11:33:00 +0800
categories: [技术]
tags: [流程编排,二次开发]
math: true
mermaid: true
---

[TOC]

本文将讲述基于LiteFlow二次开发的全声明式组件编排的实现

# 声明式组件定义

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

# 声明式组件自注册

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

# 流程配置文件

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

## 执行流程

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
