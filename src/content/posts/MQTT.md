---
title: MQTT
published: 2024-06-28
description: MQTT 协议详解及 SpringBoot 集成 RabbitMQ MQTT 实现即时通讯
image: ''
tags: [MQTT, 消息队列, SpringBoot]
category: '技术分享'
draft: false
lang: zh-CN
---
# MQTT协议

MQTT（Message Queuing Telemetry Transport，消息队列遥测传输协议），是一种基于发布/订阅（publish/subscribe）模式的`轻量级`通讯协议，该协议构建于`TCP/IP`协议上。MQTT最大优点在于，可以以极少的代码和有限的带宽，为连接远程设备提供实时可靠的消息服务。

![图片](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202406272212277.webp)

## 相关概念

- Publisher（发布者）：消息的发出者，负责发送消息。
- Subscriber（订阅者）：消息的订阅者，负责接收并处理消息。
- Broker（代理）：消息代理，位于消息发布者和订阅者之间，各类支持MQTT协议的消息中间件都可以充当。
- Topic（主题）：可以理解为消息队列中的路由，订阅者订阅了主题之后，就可以收到发送到该主题的消息。
- Payload（负载）；可以理解为发送消息的内容。
- QoS（消息质量）：全称Quality of Service，即消息的发送质量，主要有`QoS 0`、`QoS 1`、`QoS 2`三个等级，下面分别介绍下：
  - QoS 0（Almost Once）：至多一次，只发送一次，会发生消息丢失或重复；
  - QoS 1（Atleast Once）：至少一次，确保消息到达，但消息重复可能会发生；
  - QoS 2（Exactly Once）：只有一次，确保消息只到达一次。

## RabbitMQ启用MQTT功能

> RabbitMQ启用MQTT功能，需要先安装然后再启用插件。

```cmd
rabbitmq-plugins enable rabbitmq_mqtt
```

- 启用RabbitMQ的MQTT插件了，默认是不启用的，使用如下命令开启即可

![image-20240627230217734](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/2024%2F06%2F28%2F20240628003122.png)

- 开启成功后，查看管理控制台，我们可以发现MQTT服务运行在`1883`端口上了。

![image-20240627230331865](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/2024%2F06%2F28%2F20240628003119.png)

## MQTT客户端

![image-20240627231045986](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/2024%2F06%2F28%2F20240628003114.png)



![image-20240627232552108](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/2024%2F06%2F28%2F20240628003111.png)

![image-20240627232742979](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/2024%2F06%2F28%2F20240628003106.png)

![image-20240627232841088](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/2024%2F06%2F28%2F20240628003100.png)

## 前端直接实现即时通讯

```
rabbitmq-plugins enable rabbitmq_web_mqtt
```

- WEB端与MQTT服务进行通讯需要使用一个叫`MQTT.js`的库，项目地址：https://github.com/mqttjs/MQTT.js
- 第一个订阅主题`testTopicA`，访问地址：http://localhost:8088/page/index?topic=testTopicA
- 第二个订阅主题`testTopicB`，访问地址：http://localhost:8088/page/index?topic=testTopicB

![图片](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/2024%2F06%2F28%2F20240628003022.gif)



## 在SpringBoot中使用

> 没有特殊业务需求的时候，前端可以直接和RabbitMQ对接实现即时通讯。但是有时候我们需要通过服务端去通知前端，此时就需要在应用中集成MQTT了，接下来我们来讲讲如何在SpringBoot应用中使用MQTT。

### 依赖

```xml
<!--Spring集成MQTT-->
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-mqtt</artifactId>
</dependency>
```

### 在`application.yml`中添加MQTT相关配置

```yaml
# 应用服务 WEB 访问端口
server:
  port: 8080
# mqtt
rabbitmq:
  mqtt:
    url: tcp://localhost:1883
    username: guest
    password: guest
    defaultTopic: testTopic
```

### 代码

#### Java配置类

```java
@Data
@EqualsAndHashCode(callSuper = false)
@Component
@ConfigurationProperties(prefix = "rabbitmq.mqtt")
public class MqttConfig {
    /**
     * RabbitMQ连接用户名
     */
    private String username;
    /**
     * RabbitMQ连接密码
     */
    private String password;
    /**
     * RabbitMQ的MQTT默认topic
     */
    private String defaultTopic;
    /**
     * RabbitMQ的MQTT连接地址
     */
    private String url;
}
```

#### MQTT消息订阅者相关配置

使用`@ServiceActivator`注解声明一个服务激活器，通过`MessageHandler`来处理订阅消息

```java
@Slf4j
@Configuration
public class MqttInboundConfig {
    @Autowired
    private MqttConfig mqttConfig;

    @Bean
    public MessageChannel mqttInputChannel() {
        return new DirectChannel();
    }

    @Bean
    public MessageProducer inbound() {
        MqttPahoMessageDrivenChannelAdapter adapter =
                new MqttPahoMessageDrivenChannelAdapter(mqttConfig.getUrl(), "subscriberClient",
                        mqttConfig.getDefaultTopic());
        adapter.setCompletionTimeout(5000);
        adapter.setConverter(new DefaultPahoMessageConverter());
        //设置消息质量：0->至多一次；1->至少一次；2->只有一次
        adapter.setQos(1);
        adapter.setOutputChannel(mqttInputChannel());
        return adapter;
    }

    @Bean
    @ServiceActivator(inputChannel = "mqttInputChannel")
    public MessageHandler handler() {
        return new MessageHandler() {

            @Override
            public void handleMessage(Message<?> message) throws MessagingException {
                //处理订阅消息
                log.info("handleMessage : {}",message.getPayload());
            }

        };
    }
}
```

#### MQTT消息发布者相关配置

使用`@ServiceActivator`注解声明一个服务激活器，通过`MessageHandler`来发布消息

```java
@Configuration
public class MqttOutboundConfig {

    @Autowired
    private MqttConfig mqttConfig;

    @Bean
    public MqttPahoClientFactory mqttClientFactory() {
        DefaultMqttPahoClientFactory factory = new DefaultMqttPahoClientFactory();
        MqttConnectOptions options = new MqttConnectOptions();
        options.setServerURIs(new String[] { mqttConfig.getUrl()});
        options.setUserName(mqttConfig.getUsername());
        options.setPassword(mqttConfig.getPassword().toCharArray());
        factory.setConnectionOptions(options);
        return factory;
    }

    @Bean
    @ServiceActivator(inputChannel = "mqttOutboundChannel")
    public MessageHandler mqttOutbound() {
        MqttPahoMessageHandler messageHandler =
                new MqttPahoMessageHandler("publisherClient", mqttClientFactory());
        messageHandler.setAsync(true);
        messageHandler.setDefaultTopic(mqttConfig.getDefaultTopic());
        return messageHandler;
    }

    @Bean
    public MessageChannel mqttOutboundChannel() {
        return new DirectChannel();
    }
}
```

#### 添加MQTT网关，用于向主题中发送消息

```java
@Component
@MessagingGateway(defaultRequestChannel = "mqttOutboundChannel")
public interface MqttGateway {
    /**
     * 发送消息到默认topic
     */
    void sendToMqtt(String payload);

    /**
     * 发送消息到指定topic
     */
    void sendToMqtt(String payload, @Header(MqttHeaders.TOPIC) String topic);

    /**
     * 发送消息到指定topic并设置QOS
     */
    void sendToMqtt(@Header(MqttHeaders.TOPIC) String topic, @Header(MqttHeaders.QOS) int qos, String payload);
}
```

#### 测试接口

```java
@RestController
@RequestMapping("/mqtt")
public class MqttController {

    @Autowired
    private MqttGateway mqttGateway;

    /**
     * sendToDefaultTopic
     * @param payload
     */
    @PostMapping("/sendToDefaultTopic")
    public void sendToDefaultTopic(String payload) {
        mqttGateway.sendToMqtt(payload);
    }

    /**
     * sendToTopic
     * @param payload
     * @param topic
     */
    @PostMapping("/sendToTopic")
    public void sendToTopic(String payload, String topic) {
        mqttGateway.sendToMqtt(payload, topic);
    }
}
```

![image-20240628001431812](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/2024%2F06%2F28%2F20240628003034.png)

![image-20240628001712633](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/2024%2F06%2F28%2F20240628003049.png)

## MQTT和Websocket的区别

MQTT（Message Queuing Telemetry Transport）和WebSocket都是在物联网（IoT）和实时Web应用中广泛使用的协议，但它们的设计目的、工作方式以及应用场景有所不同。

**MQTT**:

1. **设计目的**: MQTT最初是为低带宽、高延迟或不可靠的网络连接而设计的，特别适用于资源受限的设备之间的通信，比如传感器网络。
2. **协议层级**: MQTT运行在传输层之上，可以使用TCP/IP作为其传输协议。它是一个轻量级的消息协议，非常适合于机器对机器（M2M）和物联网通信。
3. **发布/订阅模型**: MQTT采用发布/订阅模式，允许设备（客户端）向一个主题（Topic）发布消息，其他订阅了该主题的设备可以接收到这些消息。这种模式减少了点对点连接的需求，提高了效率。
4. **节电与带宽优化**: MQTT支持QoS（Quality of Service）等级，确保消息的可靠传递，同时允许在不可靠的网络环境中优化带宽使用。
5. **应用场景**: 常用于远程监控、智能城市、工业自动化、农业环境监测等物联网领域。

**WebSocket**:

1. **设计目的**: WebSocket是一种在单个TCP连接上进行全双工通信的协议，旨在提供浏览器与服务器间的低延迟、双向实时通信，以实现实时Web应用。
2. **协议层级**: WebSocket建立在HTTP协议之上，通过一个HTTP握手升级为WebSocket连接，之后的数据交换不再使用HTTP，而是使用自己的帧格式。
3. **双向通信**: WebSocket支持服务器主动向客户端推送数据，无需客户端发起请求，实现了真正的双向通信。
4. **灵活性**: WebSocket本身不规定消息格式，开发者可以根据需要选择JSON、XML或其他格式来封装数据。
5. **应用场景**: 常用于在线聊天、协作编辑、实时游戏、股票交易、体育赛事直播等需要低延迟交互的Web应用。

**总结**:

- 如果你的应用场景侧重于低功耗、远程设备通信和需要在不稳定网络环境下运作，MQTT可能是更好的选择。
- 对于需要在Web浏览器和服务器之间实现高效、双向实时通信的应用，WebSocket则更为合适。



> [还在用WebSocket实现实时消息推送？试试MQTT吧，真香！](https://mp.weixin.qq.com/s/IWaqLlPZydHnowZHautVHA)
>
> [MQTT和Websocket的区别是什么 – PingCode](https://docs.pingcode.com/ask/38860.html)
>
> [MQTT和Websocket的区别-CSDN博客](https://blog.csdn.net/p1279030826/article/details/122235670)
