---
title: Spring Cloud Stream
tags: ["spring cloud","spring cloud stream"]
categories: ["MQ","spring cloud"]
---
* TOC
{:toc}


### 绑定器
通过定义绑定器作为中间层，实现了应用程序与消息中间件细节之间的隔离。通过向应用程序暴露统一的Channel通过，是的应用程序不需要再考虑各种不同的消息中间件的实现。当需要升级消息中间件，或者是更换其他消息中间件产品时，我们需要做的就是更换对应的Binder绑定器而不需要修改任何应用逻辑 。

目前只提供了RabbitMQ和Kafka的Binder实现
### 消息分区(partitions)
Spring Cloud Stream对给定应用的多个实例之间分隔数据予以支持。在分隔方案中，物理交流媒介（如：代理主题）被视为分隔成了多个片（partitions）。一个或者多个生产者应用实例给多个消费者应用实例发送消息并确保相同特征的数据被同一消费者实例处理。
Spring Cloud Stream对分割的进程实例实现进行了抽象。使得Spring Cloud Stream 为不具备分区功能的消息中间件（RabbitMQ）也增加了分区功能扩展。
#### 消费者分区
```
#开启消费分区
spring.cloud.stream.bindings.<channelName>.consumer.partitioned=true
#实例数量
spring.cloud.stream.instanceCount=2
#实例索引
spring.cloud.stream.instanceIndex=1
```
#### 生产者分区
输出绑定被配置为通过设置其唯一的一个partitionKeyExpression或partitionKeyExtractorClass属性以及其partitionCount属性来发送分区数据.例如：
```
#分区键
spring.cloud.stream.bindings.<channelName>.producer.partitionKeyExpression=payload.id
#分区数量
spring.cloud.stream.bindings.<channelName>.producer.partitionCount=2
```
### Spring integration支持
#### @ServiceActivator 和 @InboundChannelAdapter
@ServiceActivator注解 和 @StreamListener 都实现了对消息的监听，ServiceActivator 没有内置消息转换，需要自己实现转换
@StreamListener 不需要自己实现，只需要在配置文件增加spring.cloud.stream.bindings.input.content-type=application/json 属性(默认支持json，json格式的可以不用配置)
