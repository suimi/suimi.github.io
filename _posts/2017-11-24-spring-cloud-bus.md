---
layout: post
title: Spring Cloud BUS
tags: ["spring cloud","spring cloud bus"]
categories: ["MQ","spring cloud"]
---
* TOC
{:toc}

# 刷新机制

1. application id
- 类`ContextIdApplicationContextInitializer`定义了具体的id格式
- 默认为`<spring.application.name>.<active profiles>.<server.port>`
- 该id将作为刷新时的destination使用(eg. `/bus/refresh?destination=customers:dev:9000`).
- 事件origin id为实例application id, destination id支持匹配模式,也是与application id 匹配

2. bus/refresh端点

    收到refresh时间后,发布`RefreshRemoteApplicationEvent`事件,指定事件源id,目标id(调用时可指定)
    ```shell
    curl -X POST -d destination=<destination> http://localhost:8081/bus/refresh
    eg: curl -X POST -d destination=bus-rabbitmq:rabbitmq:8081 http://localhost:8081/bus/refresh
    ```

    ```java
    class RefreshBusEndpoint
    @RequestMapping(value = "refresh", method = RequestMethod.POST)
        @ResponseBody
        @ManagedOperation
        public void refresh(
                @RequestParam(value = "destination", required = false) String destination) {
            publish(new RefreshRemoteApplicationEvent(this, getInstanceId(), destination));
        }
    ```

3. 事件监听

    - 刷新事件注册及刷新
    ```java
    class BusAutoConfiguration
    @Bean
            @ConditionalOnProperty(value = "spring.cloud.bus.refresh.enabled", matchIfMissing = true)
            public RefreshListener refreshListener(ContextRefresher contextRefresher) {
                return new RefreshListener(contextRefresher);
            }
    ```
    ```java
    class RefreshListener
        @Override
        public void onApplicationEvent(RefreshRemoteApplicationEvent event) {
            Set<String> keys = contextRefresher.refresh();
            log.info("Received remote refresh request. Keys refreshed " + keys);
        }
    ```
    - 消息发布. 如果是当前实例发布的并且不是ack事件,则会向out channel 发布该事件消息

    ```java
    class BusAutoConfiguration
    @EventListener(classes = RemoteApplicationEvent.class)
        public void acceptLocal(RemoteApplicationEvent event) {
            if (this.serviceMatcher.isFromSelf(event)
                    && !(event instanceof AckRemoteApplicationEvent)) {
                this.cloudBusOutboundChannel.send(MessageBuilder.withPayload(event).build());
            }
        }
    ```

4. 消息监听.
    - 如果是ack事件并且bus.trace开启且是自己发布的事件,发布该事件
    - 如果是目标是自己,发布刷新事件，有`RefreshListener`做刷新处理,参见事件监听
    ```java
    class BusAutoConfiguration
    @StreamListener(SpringCloudBusClient.INPUT)
        public void acceptRemote(RemoteApplicationEvent event) {
            if (event instanceof AckRemoteApplicationEvent) {
                if (this.bus.getTrace().isEnabled() && !this.serviceMatcher.isFromSelf(event)
                        && this.applicationEventPublisher != null) {
                    this.applicationEventPublisher.publishEvent(event);
                }
                // If it's an ACK we are finished processing at this point
                return;
            }
            if (this.serviceMatcher.isForSelf(event)
                    && this.applicationEventPublisher != null) {
                if (!this.serviceMatcher.isFromSelf(event)) {
                    this.applicationEventPublisher.publishEvent(event);
                }
                if (this.bus.getAck().isEnabled()) {
                    AckRemoteApplicationEvent ack = new AckRemoteApplicationEvent(this,
                            this.serviceMatcher.getServiceId(),
                            this.bus.getAck().getDestinationService(),
                            event.getDestinationService(), event.getId(), event.getClass());
                    this.cloudBusOutboundChannel
                            .send(MessageBuilder.withPayload(ack).build());
                    this.applicationEventPublisher.publishEvent(ack);
                }
            }
            if (this.bus.getTrace().isEnabled() && this.applicationEventPublisher != null) {
                // We are set to register sent events so publish it for local consumption,
                // irrespective of the origin
                this.applicationEventPublisher.publishEvent(new SentApplicationEvent(this,
                        event.getOriginService(), event.getDestinationService(),
                        event.getId(), event.getClass()));
            }
        }
    ```


# RabbitMQ
## Destination
- 配置属性:`spring.cloud.bus.destination`，是topic名称,默认为springCloudBus
- 与刷新时的`destination`不是一个概念(eg:`/bus/refresh?destination=customers:9000`)

## Queue
客户端启动时会创建对应Queue,queue名为`<Destination>.anonymous.<uuid>` eg: `springCloudBus.anonymous.t4cuHSE6RfKRYvPvrgfbUg`,并且绑定到对应topic,绑定路由为`#`
```java
class RabbitExchangeQueueProvisioner
@Override
	public ConsumerDestination provisionConsumerDestination(String name, String group, ExtendedConsumerProperties<RabbitConsumerProperties> properties) {
		boolean anonymous = !StringUtils.hasText(group);
		String baseQueueName = anonymous ? groupedName(name, ANONYMOUS_GROUP_NAME_GENERATOR.generateName())
				: groupedName(name, group);
		...
	}
```
```java
class Base64UrlNamingStrategy

@Override
		public String generateName() {
			UUID uuid = UUID.randomUUID();
			ByteBuffer bb = ByteBuffer.wrap(new byte[16]);
			bb.putLong(uuid.getMostSignificantBits())
			  .putLong(uuid.getLeastSignificantBits());
			// TODO: when Spring 4.2.4 is the minimum Spring Framework version, use encodeToUrlSafeString() SPR-13784.
			return this.prefix + Base64Utils.encodeToString(bb.array())
									.replaceAll("\\+", "-")
									.replaceAll("/", "_")
									// but this will remain
									.replaceAll("=", "");
		}
```

# Kafka
## Destination
- 配置属性:`spring.cloud.bus.destination`，是topic名称,默认为springCloudBus
- 与刷新时的`destination`不是一个概念(eg:`/bus/refresh?destination=customers:9000`)

## Group
客户端创建消费者时, 会默认设置group id`anonymous.<uuid>` eg:`anonymous.213c19b4-aa26-4c84-a814-c4ff5a335e18`
```java
class KafkaMessageChannelBinder extends AbstractMessageChannelBinder

@Override
	@SuppressWarnings("unchecked")
	protected MessageProducer createConsumerEndpoint(final ConsumerDestination destination, final String group,
			final ExtendedConsumerProperties<KafkaConsumerProperties> extendedConsumerProperties) {

		boolean anonymous = !StringUtils.hasText(group);
		Assert.isTrue(!anonymous || !extendedConsumerProperties.getExtension().isEnableDlq(),
				"DLQ support is not available for anonymous subscriptions");
		String consumerGroup = anonymous ? "anonymous." + UUID.randomUUID().toString() : group;
		...
	}
```
