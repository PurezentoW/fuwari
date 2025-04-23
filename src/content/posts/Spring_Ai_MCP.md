---
title: Spring_Ai_MCP.md
published: 2025-04-23
description: 'springboot 整合MCP实现数据库查询'
image: ''
tags: [AI, Spring, MCP]
category: 技术分享
draft: false 
lang: ''
---


# Spring Boot整合MCP实现数据库查询

## Spring AI MCP 简介

Spring AI MCP 为模型上下文协议提供 Java 和 Spring 框架集成。它使 Spring AI 应用程序能够通过标准化的接口与不同的数据源和工具进行交互，支持同步和异步通信模式。

![img](https://cdn.wcxian.cc/img/20250423220003834.png)

### Spring AI MCP 采用模块化架构，包括以下组件：

- Spring AI 应用程序：使用 Spring AI 框架构建想要通过 MCP 访问数据的生成式 AI 应用程序
- Spring MCP 客户端：MCP 协议的 Spring AI 实现，与服务器保持 1:1 连接
- MCP 服务器：轻量级程序，每个程序都通过标准化的模型上下文协议公开特定的功能
- 本地数据源：MCP 服务器可以安全访问的计算机文件、数据库和服务
- 远程服务：MCP 服务器可以通过互联网（例如，通过 API）连接到的外部系统

### 核心组件

1. MCP Client，与 MCP 集成的关键，提供了与本地文件系统进行交互的能力。
2. Function Callbacks，Spring AI MCP 的 function calling 声明方式。
3. Chat Client，Spring AI 关键组件，用于 LLM 模型交互、智能体代理。

## 一、环境准备

### 1.1 依赖配置

```xml
    	<!-- JDK Spring Spring ai相关版本-->
        <properties>
            <java.version>17</java.version>
            <spring-ai.version>1.0.0-M6</spring-ai.version>
            <spring-boot.version>3.4.4</spring-boot.version>
        </properties>

        <!-- Spring AI 核心依赖 -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-core</artifactId>
            <version>${spring-ai.version}</version>
        </dependency>
        <!-- Spring AI OpenAI 集成依赖-->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
            <version>${spring-ai.version}</version>
        </dependency>
        <!-- Spring AI Spring Boot 自动配置依赖-->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-spring-boot-autoconfigure</artifactId>
            <version>${spring-ai.version}</version>
        </dependency>
        <!-- Spring AI MCP Server 相关依赖 -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-mcp-server-webmvc-spring-boot-starter</artifactId>
            <version>${spring-ai.version}</version>
        </dependency>
```

```xml
<!--因为这些包比较新，如果下不到，就要配置这些仓库-->
<repositories>
    <repository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
    <repository>
        <id>spring-snapshots</id>
        <name>Spring Snapshots</name>
        <url>https://repo.spring.io/snapshot</url>
        <releases>
            <enabled>false</enabled>
        </releases>
    </repository>
    <repository>
        <name>Central Portal Snapshots</name>
        <id>central-portal-snapshots</id>
        <url>https://central.sonatype.com/repository/maven-snapshots/</url>
        <releases>
            <enabled>false</enabled>
        </releases>
        <snapshots>
            <enabled>true</enabled>
        </snapshots>
    </repository>
</repositories>
```

**关键点**

- SpringBoot 3
- SpringAi 1.0.0-M6 (其他版本好像有一些奇怪的问题，这个版本是正常的)
- JDK 17 +

### 1.2 核心配置，需使用deepseek-chat模型，其他模型暂不支持工具调用

```yaml
# Anthropic 聊天服务配置
spring:
  ai:
    anthropic:
      chat:
        enabled: false  # 禁用 Anthropic 聊天服务

    # MCP 服务配置
    mcp:
      server:
        enabled: true       # 启用 MCP 服务
        name: mcp-demo-server  # 设置 MCP 服务名称为 mcp-demo-server
        version: "1.0.0"      # 设置 MCP 服务版本为 1.0.0
        type: SYNC         # 设置 MCP 服务类型为同步(SYNC)模式
        sse-message-endpoint: /mcp/message  # 设置 SSE(服务器发送事件)消息端点
        stdio: false       # 禁用标准输入输出模式

    # OpenAI 相关配置
    openai:
      chat:
        enabled: true     # 启用 OpenAI 聊天服务
        options:
          model: deepseek-chat  # 设置聊天模型为 deepseek-chat
      base-url: https://api.deepseek.com  # 设置 OpenAI API 的基础 URL(此处指向 deepseek 的 API)
      api-key: sk-2290e6*********dd225  # 设置 OpenAI API 密钥(部分隐藏)
      embedding:
        enabled: false  # 禁用 OpenAI 的嵌入功能

```

------

## 二、MCP Server配置

### 2.1 工具函数实现

#### 2.1.1 用户查询工具

```java
@Service
@Slf4j
public class SysUserService {

    @Autowired
    private SysUserMapper sysUserMapper;


    public SysUserService() {
        System.out.println("SysUserService 实例已创建");
    }

    @Tool(name = "queryUserListByName", description = "根据用户姓名模糊查询用户信息")
    public List<SysUserVo> queryUserListByName(@ToolParam(description = "用户姓名") String name) {
        LambdaQueryWrapper<SysUser> wrapper = Wrappers.lambdaQuery();
        wrapper.like(SysUser::getNickName, name);
        List<SysUser> sysUsers = sysUserMapper.selectList(wrapper);
        List<SysUserVo> sysUserVos = BeanUtil.copyToList(sysUsers, SysUserVo.class);
        return sysUserVos;
    }

}
```

#### 2.1.2 商品查询工具

```java
@Service
public class KjGoodsService  {
    @Autowired
    private KjGoodsMapper kjGoodsMapper;

    @Tool(name = "queryGoodsListByName", description = "根据商品名称和商户名称查询商品信息")
    public List<KjGoods> queryGoodsListByName(@ToolParam(description = "商品名称") String goodsName,@ToolParam(description = "商户名称") String merchantName) {
        if (merchantName != null) {
            return kjGoodsMapper.queryGoodsListByName(goodsName, merchantName);
        }
        return null;
    }



    @Tool(name = "queryGoodsListByPage", description = "根据条件分页查询商品信息")
    public IPage<GoodsVo> queryGoodsListByPage(@ToolParam(description = "分页信息") Page<KjGoods> page, @ToolParam(description = "商品名称") String goodsName, @ToolParam(description = "商户名称") String merchantName){
        IPage<KjGoods> kjGoodsIPage = kjGoodsMapper.queryGoodsListByPage(page, goodsName, merchantName);
        List<KjGoods> kjGoods = kjGoodsIPage.getRecords();
        Page<GoodsVo> result = new Page<>();
        List<GoodsVo> goodsVos = BeanUtil.copyToList(kjGoods, GoodsVo.class);
        result.setRecords(goodsVos);
        return result;
    }

}
```



------



### 2.2 工具回调注册

注意这边因为注册了2个，所以需要设置别名，如果是一个的话就不用设置

```java
@Configuration
@Slf4j
public class McpServerConfig {


    /**
     * 用户工具回调接口
     *
     * @param sysUserService sys用户服务
     * @return {@link ToolCallbackProvider }
     */
    @Bean
    public ToolCallbackProvider sysUserToolCallbackProvider(SysUserService sysUserService) {
        log.info("开始注册工具回调，sysUserService类型: {}", sysUserService.getClass().getName());

        MethodToolCallbackProvider provider = MethodToolCallbackProvider.builder()
                .toolObjects(sysUserService)
                .build();
        provider.getToolCallbacks();
        if (provider.getToolCallbacks().length == 0) {
            log.warn("没有找到任何可注册的工具方法！请检查sysUserService的方法是否满足条件");
        } else {
            Arrays.stream(provider.getToolCallbacks())
                    .forEach(tool -> log.info("注册的工具: {} 描述 ：{})",
                            tool.getToolDefinition().name(),
                            tool.getToolDefinition().description()));
        }

        return provider;
    }

    /**
     * 商品工具回调接口
     *
     * @param kjGoodsService kj商品服务
     * @return {@link ToolCallbackProvider }
     */
    @Bean
    public ToolCallbackProvider goodsToolCallbackProvider(KjGoodsService kjGoodsService) {
        log.info("开始注册工具回调，kjGoodsService: {}", kjGoodsService.getClass().getName());

        MethodToolCallbackProvider provider = MethodToolCallbackProvider.builder()
                .toolObjects(kjGoodsService)
                .build();
        provider.getToolCallbacks();
        if (provider.getToolCallbacks().length == 0) {
            log.warn("没有找到任何可注册的工具方法！请检查kjGoodsService的方法是否满足条件");
        } else {
            Arrays.stream(provider.getToolCallbacks())
                    .forEach(tool -> log.info("注册的工具: {} 描述 ：{})",
                            tool.getToolDefinition().name(),
                            tool.getToolDefinition().description()));
        }

        return provider;
    }

}
```







------



## 三、ChatClient 配置

### 3.1 客户端构建

```java
@Configuration
public class ChatClientConfig {


    /**
     * 配置ChatClient，注册系统指令和工具函数
     */
    @Bean
    public ChatClient userChatClient(ChatClient.Builder builder, @Qualifier("sysUserToolCallbackProvider") ToolCallbackProvider toolCallbackProvider) {
        return builder
                .defaultSystem("你是一个用户信息管理助手，可以帮助用户查询用户信息。" +
                        "你可以根据用户姓名模糊查询用户信息、根据条件分页查询用户信息。" +
                        "回复时，请使用简洁友好的语言，并将用户信息整理为易读的格式。")
                // 注册工具方法
                .defaultTools(toolCallbackProvider)
                .build();
    }
    /**
     * 配置ChatClient，注册系统指令和工具函数
     */
    @Bean
    public ChatClient goodsChatClient(ChatClient.Builder builder, @Qualifier("goodsToolCallbackProvider") ToolCallbackProvider toolCallbackProvider) {
        return builder
                .defaultSystem("你是一个商品信息管理助手，可以帮助用户查询商品信息。" +
                        "你可以根据商品姓名模糊查询商品信息、根据条件分页查询商品信息。" +
                        "回复时，请使用简洁友好的语言，并将商品信息整理为易读的格式。")
                // 注册工具方法
                .defaultTools(toolCallbackProvider)
                .build();
    }
}
```

------

## 四、API接口实现

### 4.1 控制器层

```java
@RestController
@RequestMapping("/api/chat")
public class ChatController {

    @Autowired
    private ChatClient userChatClient;
    @Autowired
    private ChatClient goodsChatClient;


    @PostMapping("/user")
    public ResponseEntity<ChatResponse> chat(@RequestBody ChatRequest request) {
        try {
            // 创建用户消息
            String userMessage = request.getMessage();

            // 使用流式API调用聊天
            String content = userChatClient.prompt()
                    .user(userMessage)
                    .call()
                    .content();

            return ResponseEntity.ok(new ChatResponse().setContent(content));
        } catch (Exception e) {
            e.printStackTrace();
            return ResponseEntity.ok(new ChatResponse("处理请求时出错: " + e.getMessage()));
        }
    }

    @PostMapping("/goods")
    public ResponseEntity<ChatResponse> goodsChat(@RequestBody ChatRequest request) {
        try {
            // 创建用户消息
            String userMessage = request.getMessage();

            // 使用流式API调用聊天
            String content = goodsChatClient.prompt()
                    .user(userMessage)
                    .call()
                    .content();

            return ResponseEntity.ok(new ChatResponse().setContent(content));
        } catch (Exception e) {
            e.printStackTrace();
            return ResponseEntity.ok(new ChatResponse("处理请求时出错: " + e.getMessage()));
        }
    }
}
```

------

###  启动后会显示注册的工具

![image-20250423180556833](https://cdn.wcxian.cc/img/20250423220022381.png)

## 五、功能验证

### 5.1 测试案例

**示例1：帮我查询叫王xx的人**

![image-20250423183435757](https://cdn.wcxian.cc/img/20250423220025923.png)

**示例2：技术部有少个人，查到前2个人的信息给我**

![image-20250423183653190](https://cdn.wcxian.cc/img/20250423220028884.png)

**示例3：帮我查询快进商店体验店的可口可乐500ml多少钱**

![image-20250423184129949](https://cdn.wcxian.cc/img/20250423220326949.png)

**示例4：帮我查询快进商店体验店的可乐第二页，每页5条数据**

![image-20250423220248024](https://cdn.wcxian.cc/img/20250423220329719.png)

**示例5：帮我查询快进商店体验店的有多少种可乐**

![image-20250423215921085](https://cdn.wcxian.cc/img/20250423220332930.png)

代码流程

![image-20250423184447352](https://cdn.wcxian.cc/img/20250423220335696.png)

![image-20250423184541828](https://cdn.wcxian.cc/img/20250423220338009.png)

------

## 六、技术要点解析

1. **工具动态注册机制**
   通过`MethodToolCallbackProvider`实现Spring Bean方法的自动发现，运行时动态注册工具函数
2. **国产模型适配**
   通过修改base-url实现对DeepSeek的适配，保持与OpenAI API兼容
3. **上下文管理**
   `defaultSystem`指令确保大模型始终遵守预设的业务规则

##  参考资料：

> [SpringBoot整合MCP，支持使用国产大模型DeepSeek，通过MCP让DeepSeek来帮你查询数据库 - 掘金](https://article.juejin.cn/post/7489720690884624393)
>
> [对话即服务：Spring Boot整合MCP让你的CRUD系统秒变AI助手-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2506434)
>
> [把SpringBoot项目改造成MCP Server非常简单 - 知乎](https://zhuanlan.zhihu.com/p/1892863789579351119)
>
> [Grok](https://grok.com/?referrer=website)

