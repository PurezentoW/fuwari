---
title: Redisson 延迟队列
published: 2024-02-25
description: 基于 Redisson 的分布式延迟队列实现，含完整 SpringBoot 集成代码与原理分析
image: ''
tags: [Redis, Redisson, 消息队列, SpringBoot]
category: '技术分享'
draft: false
lang: zh-CN
---
# Redisson 延迟队列



## 使用场景

1、下单成功，30分钟未支付。支付超时，自动取消订单

2、订单签收，签收后7天未进行评价。订单超时未评价，系统默认好评

3、下单成功，商家5分钟未接单，订单取消

4、配送超时，推送短信提醒

5、三天会员试用期，三天到期后准时准点通知用户，试用产品到期了

6、**用户进店后按照顺序播放音频**

......

对于延时比较长的场景、实时性不高的场景，我们可以采用任务调度的方式定时轮询处理。如：xxl-job。

今天我们讲解**延迟队列**的实现方式，而延迟队列有很多种实现方式，普遍会采用如下等方式，如：

- 1.如基于RabbitMQ的队列ttl+死信路由策略：通过设置一个队列的超时未消费时间，配合死信路由策略，到达时间未消费后，回会将此消息路由到指定队列

  （暴力开门和卡门）

- 2.基于RabbitMQ延迟队列插件（rabbitmq-delayed-message-exchange）：发送消息时通过在请求头添加延时参数（headers.put("x-delay", 5000)）即可达到延迟队列的效果。

- 3.使用redis的zset有序性，轮询zset中的每个元素

  `Redis`的数据结构`Zset`，可以实现延迟队列的效果，主要利用它的`score`属性，`redis`通过`score`来为集合中的成员进行从小到大的排序。

  ![image-20240225165633946](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202402251656466.png)

- 4.使用redis的key的过期通知策略，设置一个key的过期时间为延迟时间，过期后通知客户端(此方式依赖redis过期检查机制key多后延迟会比较严重；Redis的pubsub不会被持久化，服务器宕机就会被丢弃)。

- Redisson 延时队列



****

## 一、先整合实践一下

### 1、引入 Redisson 依赖并配置redis

```xml
        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson-spring-boot-starter</artifactId>
            <version>3.10.5</version>
        </dependency>
```

```yaml
  redis:
    # 地址
    host: 127.0.0.1
    # 端口，默认为6379
    port: 6379
    # 数据库索引
    database: 0
    # 密码
    password: 123456
    # 连接超时时间
    timeout: 10s
```



### 2、创建 RedissonConfig 配置

```java
/**
 * Redission配置类
 */
@Slf4j
@Configuration
public class RedissionConfig {
    private final String REDISSON_PREFIX = "redis://";
    private final RedisProperties redisProperties;

    public RedissionConfig(RedisProperties redisProperties) {
        this.redisProperties = redisProperties;
    }

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        String url = REDISSON_PREFIX + redisProperties.getHost() + ":" + redisProperties.getPort();
        // 这里以单台redis服务器为例
        config.useSingleServer()
            .setAddress(url)
            .setPassword(redisProperties.getPassword())
            .setDatabase(redisProperties.getDatabase())
            .setPingConnectionInterval(2000);
        config.setLockWatchdogTimeout(10000L);
        try {
            return Redisson.create(config);
        } catch (Exception e) {
            log.error("RedissonClient init redis url:[{}], Exception:", url, e);
            return null;
        }
    }
}
```

### 3、封装 Redis 延迟队列工具类

```java
/**
 * 分布式延时队列工具类
 */
@Slf4j
@Component
//@ConditionalOnBean({RedissonClient.class})
public class RedisDelayQueueUtil {

    @Resource
    private RedissonClient redissonClient;

    /**
     * 添加延迟队列
     *
     * @param value     队列值
     * @param delay     延迟时间
     * @param timeUnit  时间单位
     * @param queueCode 队列键
     * @param <T>
     */
    public <T> boolean addDelayQueue(@NonNull T value, @NonNull long delay, @NonNull TimeUnit timeUnit, @NonNull String queueCode) {
        if (StringUtils.isBlank(queueCode) || Objects.isNull(value)) {
            return false;
        }
        try {
            RBlockingDeque<Object> blockingDeque = redissonClient.getBlockingDeque(queueCode);
            RDelayedQueue<Object> delayedQueue = redissonClient.getDelayedQueue(blockingDeque);
            delayedQueue.offer(value, delay, timeUnit);
            //delayedQueue.destroy();
            log.info("(添加延时队列成功) 队列键：{}，队列值：{}，延迟时间：{}", queueCode, value, timeUnit.toSeconds(delay) + "秒");
        } catch (Exception e) {
            log.error("(添加延时队列失败) {}", e.getMessage());
            throw new RuntimeException("(添加延时队列失败)");
        }
        return true;
    }

    /**
     * 获取延迟队列
     *
     * @param queueCode
     * @param <T>
     */
    public <T> T getDelayQueue(@NonNull String queueCode) throws InterruptedException {
        if (StringUtils.isBlank(queueCode)) {
            return null;
        }
        RBlockingDeque<Map> blockingDeque = redissonClient.getBlockingDeque(queueCode);
        RDelayedQueue<Object> delayedQueue = redissonClient.getDelayedQueue(blockingDeque);
        T value = (T) blockingDeque.poll();
        return value;
    }
    /**
     * 删除指定队列中的消息
     *
     * @param o 指定删除的消息对象队列值(同队列需保证唯一性)
     * @param queueCode 指定队列键
     */
     public boolean removeDelayedQueue(@NonNull Object o, @NonNull String queueCode) {
        if (StringUtils.isBlank(queueCode) || Objects.isNull(o)) {
            return false;
        }
        RBlockingDeque<Object> blockingDeque = redissonClient.getBlockingDeque(queueCode); RDelayedQueue<Object> delayedQueue = redissonClient.getDelayedQueue(blockingDeque); boolean flag = delayedQueue.remove(o);
        //delayedQueue.destroy();
        return flag;
    }
}
```

### 4、创建延迟队列业务枚举

```java
/**
 * 延迟队列业务枚举
 */
@Getter
@NoArgsConstructor
@AllArgsConstructor
public enum RedisDelayQueueEnum {

    ORDER_PAYMENT_TIMEOUT("ORDER_PAYMENT_TIMEOUT","订单支付超时，自动取消订单", "orderPaymentTimeout"),
    ORDER_TIMEOUT_NOT_EVALUATED("ORDER_TIMEOUT_NOT_EVALUATED", "订单超时未评价，系统默认好评", "orderTimeoutNotEvaluated");

    /**
     * 延迟队列 Redis Key
     */
    private String code;

    /**
     * 中文描述
     */
    private String name;

    /**
     * 延迟队列具体业务实现的 Bean
     * 可通过 Spring 的上下文获取
     */
    private String beanId;

}
```

### 5、定义延迟队列执行器

```java
/**
 * 延迟队列执行器
 */
public interface RedisDelayQueueHandle<T> {

    void execute(T t);

}
```

### 6、创建枚举中定义的Bean,实现延迟队列执行器

**OrderPaymentTimeout：订单支付超时延迟队列处理类**

```java
/**
 * 订单支付超时处理类
 */
@Component
@Slf4j
public class OrderPaymentTimeout implements RedisDelayQueueHandle<Map> {
    @Override
    public void execute(Map map) {
        log.info("(收到订单支付超时延迟消息) {}", map);
        // TODO 订单支付超时，自动取消订单处理业务...

    }
}
```

**OrderTimeoutNotEvaluated：订单超时未评价延迟队列处理类**

```java
/**
 * 订单超时未评价处理类
 */
@Component
@Slf4j
public class OrderTimeoutNotEvaluated implements RedisDelayQueueHandle<Map> {
    @Override
    public void execute(Map map) {
        log.info("(收到订单超时未评价延迟消息) {}", map);
        // TODO 订单超时未评价，系统默认好评处理业务...

    }
}
```

### 7、创建延迟队列消费线程，项目启动完成后开启

```java
/**
 * 启动延迟队列监测扫描
 */
@Slf4j
@Component
public class RedisDelayQueueRunner implements CommandLineRunner {
    @Autowired
    private RedisDelayQueueUtil redisDelayQueueUtil;
    @Autowired
    private ApplicationContext context;
    @Autowired
    private ThreadPoolTaskExecutor ptask;

    ThreadPoolExecutor executorService = new ThreadPoolExecutor(3, 5, 30, TimeUnit.SECONDS,
        new LinkedBlockingQueue<Runnable>(1000),new ThreadFactoryBuilder().setNameFormat("order-delay-%d").build());

    @Override
    public void run(String... args) throws Exception {
        ptask.execute(() -> {
            // 无限循环监测延迟队列
            while (true){
                try {
                     // 获取所有延迟队列的枚举值
                    RedisDelayQueueEnum[] queueEnums = RedisDelayQueueEnum.values();
                    // 遍历枚举值
                    for (RedisDelayQueueEnum queueEnum : queueEnums) {
                        // 获取延迟队列中的值
                        Object value = redisDelayQueueUtil.getDelayQueue(queueEnum.getCode());
                        // 如果值不为空，则执行处理操作
                        if (value != null) {
                            // 获取处理器对象并执行
                            RedisDelayQueueHandle<Object> redisDelayQueueHandle = (RedisDelayQueueHandle<Object>)context.getBean(queueEnum.getBeanId());
                            // 在线程池中执行处理操作
                            executorService.execute(() -> {redisDelayQueueHandle.execute(value);});
                        }
                    }
                    // 控制循环速度为每次 500 毫秒
                    TimeUnit.MILLISECONDS.sleep(500);
                } catch (InterruptedException e) {
                    log.error("(Redission延迟队列监测异常中断) {}", e.getMessage());
                }
            }
        });
        log.info("(Redission延迟队列监测启动成功)");
    }
}
```



### 8、创建一个测试接口，模拟添加延迟队列

```java
/**
 * 延迟队列测试
 */
@RestController
public class RedisDelayQueueController {

    @Autowired
    private RedisDelayQueueUtil redisDelayQueueUtil;

    @PostMapping("/addQueue")
    public void addQueue() {
        Map<String, String> map1 = new HashMap<>();
        map1.put("orderId", "100");
        map1.put("remark", "订单支付超时，自动取消订单");

        Map<String, String> map2 = new HashMap<>();
        map2.put("orderId", "200");
        map2.put("remark", "订单超时未评价，系统默认好评");

        // 添加订单支付超时，自动取消订单延迟队列。为了测试效果，延迟10秒钟
        redisDelayQueueUtil.addDelayQueue(map1, 10, TimeUnit.SECONDS, RedisDelayQueueEnum.ORDER_PAYMENT_TIMEOUT.getCode());

        // 订单超时未评价，系统默认好评。为了测试效果，延迟20秒钟
        redisDelayQueueUtil.addDelayQueue(map2, 20, TimeUnit.SECONDS, RedisDelayQueueEnum.ORDER_TIMEOUT_NOT_EVALUATED.getCode());
    }

}
```

启动 SpringBoot 项目，用 PostMan 调用接口添加延迟队列 通过 Redis 客户端可看到两个延迟队列已添加成功

![image-20240225162047378](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202402251620631.png)

查看 IDEA 控制台日志可看到延迟队列已消费成功

![image-20240225162444640](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202402251625986.png)

## 小结一下

这两个队列中，带有timeout关键字的那条是一个ZSet集合，另一个是普通的List。

- 客户端启动，redisson先订阅一个key，同时 BLPOP key 0 无限监听一个阻塞队列（等里面有数据了就返回）。
- 当有数据put时，redisson先把数据放到一个zset集合（按延时到期时间的时间戳为分数排序），同时发布上面订阅的key，发布内容为数据到期的timeout，此时客户端进程开启一个延时任务，延时时间为发布的timeout。
- 客户端进程的延时任务到了时间执行，从zset分页取出过了当前时间的数据，然后将数据rpush到第一步的阻塞队列里。然后将当前数据从zset移除，取完之后，又执行 BLPOP key 0 无限监听一个阻塞队列。这一部分的逻辑，客户端会发送一个lua脚本给到服务端去操作：org.redisson.RedissonDelayedQueue源码里面的pushTaskAsync函数有lua脚本内容。

- 上一步客户端监听的阻塞队列返回取到数据，回调到 RBlockingQueue 的 take方法。于是，我们就收到了数据。

这种设计利用了 Redis 的发布订阅机制和有序集合的特性，结合延时任务和阻塞队列的处理方式，实现了高效的延迟队列功能。这种设计可以确保任务按照延时时间顺序被处理，同时保证了任务的可靠性和高性能。





参考笔记：

> [基于redis,redisson的延迟队列实践 - 字节悦动 - 博客园 (cnblogs.com)](https://www.cnblogs.com/better-farther-world2099/articles/15216447.html)
>
> [Redisson 延时队列 原理 详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/343811173)
