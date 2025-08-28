---
title: Spring Cloud Alibaba 灰度发布
published: 2025-08-27
description: 'Spring Cloud Alibaba 灰度发布教程'
image: ''
tags: [Spring Cloud, Nacos, Gateway]
category: '技术分享'
draft: false 
lang: ''
---
# Spring Cloud Alibaba 灰度发布教程

本教程展示如何基于 Spring Cloud Alibaba 使用 **Nacos**（服务注册与配置中心）和 **Spring Cloud Gateway** 实现灰度发布。我们将部署两个版本的服务（1.0.0 和 1.0.1），通过网关根据请求特征（如请求头或用户 ID）将部分流量路由到新版本（1.0.1）。

## 前提条件

- Java 17 或更高版本（推荐使用 JDK 17，确保环境变量 JAVA_HOME 配置正确）。
- Maven 3.8+ 或 Gradle 7+（本教程使用 Maven 示例）。
- Spring Boot 3.x（推荐 3.0.0 或更高）。
- Nacos Server（推荐 2.3.2 或更高版本，下载并启动，参考 Nacos 官网）。
- Spring Cloud Alibaba 2023.0.1.0（确保与 Spring Boot 版本兼容）。
- IDE（如 IntelliJ IDEA 或 VS Code），用于代码编辑和调试。
- 确保本地防火墙未阻止端口 8848（Nacos 默认端口）、8080（Gateway）、8081/8082（服务端口）。
- 安装 curl 或 Postman，用于测试 API。

## 步骤 1：搭建 Nacos Server

Nacos 作为服务注册中心和配置中心

[Nacos 快速开始 | Nacos 官网](https://nacos.io/docs/latest/quickstart/quick-start/)

![image-20250825161754056](https://cdn.wcxian.cc/img/20250827215717662.png)

1. **下载并启动 Nacos**：

    - 从 Nacos 官网下载页面 下载 Nacos Server。

    - 解压到本地目录

      3.* 版本需要设置一个密钥，2.* 不用设置，我这边是最新的3.0.3稳定版，所以要设置一个32位的密钥

      ```properties
      ##"D:\My Project\Spring-Cloud-HDFB\nacos\conf\application.properties"
      nacos.core.auth.plugin.nacos.token.secret.key=HTzELaWk1NS2LLCTyrsH6VeTOLTpdfg6CcpLwDvRGSw=
      ```

    - 进入 `bin` 目录，运行单机模式启动命令：

        - Linux/Mac：`sh startup.sh -m standalone`
        - Windows：`cmd startup.cmd -m standalone`

      ![image-20250825164351922](https://cdn.wcxian.cc/img/20250827215555596.png)

    - 启动后，检查日志（`logs/start.out`）确认无错误，端口 8848 已监听。

    - 访问 Nacos 控制台：`http://localhost:8080`，默认用户名/密码为 `nacos/nacos`。如果无法访问，检查防火墙或端口占用。

      ![image-20250825164517354](https://cdn.wcxian.cc/img/20250827215602342.png)

2. **配置 Nacos**：

    - 在 Nacos 控制台左侧菜单，选择 “命名空间” &gt; “新建命名空间”，创建命名空间 ID 为 `gray-release`，名称为 “Gray Release”（可选，但推荐用于隔离环境）。如果不创建，将使用默认命名空间 `public`。
    - 注意：如果使用自定义命名空间，在后续服务和 Gateway 配置中需添加 `namespace: gray-release`。
    - 在 “服务管理” &gt; “服务列表” 页面，确保页面加载正常（如果为空，这是正常的，因为还未注册服务）。

## 步骤 2：创建服务（新旧版本）

我们将创建两个版本的服务实例，模拟 1.0.0（稳定版）和 1.0.1（新版）。重点确保服务正确注册到 Nacos。

### 2.1 创建服务项目

#### 官方推荐版本对应关系（核心）

| Spring Cloud Alibaba Version  | Spring Cloud Version       | Spring Boot Version | **Nacos Client Version** | 重要说明                            |
| :---------------------------- | :------------------------- | :------------------ | :----------------------- | :---------------------------------- |
| **2022.0.0.0** *（最新）*     | 2022.0.x                   | 3.0.x ~ 3.1.x       | **2.2.7.RELEASE**        | 基于 Spring Cloud 2022（"Kilburn"） |
| **2021.0.5.0** *（推荐稳定）* | 2021.0.x                   | 2.6.x ~ 2.7.x       | **2.2.7.RELEASE**        | 基于 Spring Cloud 2021（"Jubilee"） |
| **2.2.10-RC1**                | Spring Cloud Hoxton.SR12   | 2.3.x ~ 2.4.x       | 2.2.7.RELEASE            | 基于 Spring Cloud Hoxton            |
| **2021.1**                    | Spring Cloud 2020.0.x      | 2.4.x ~ 2.6.x       | 2.2.7.RELEASE            | 旧版，不推荐新项目使用              |
| **2.2.7.RELEASE**             | Spring Cloud Hoxton.SR9    | 2.3.x ~ 2.4.x       | 2.2.7.RELEASE            | 旧版稳定                            |
| **2.1.4.RELEASE**             | Spring Cloud Greenwich.SR6 | 2.1.x               | 1.4.2                    | **非常旧**，仅维护                  |

| Nacos Server Version | Recommended Nacos Client Version |
| -------------------- | -------------------------------- |
| **3.0.x**            | **2.2.3 - 2.5.1**                |
| 2.2.x                | 2.2.1 - 2.2.3                    |
| 2.1.x                | 2.1.2                            |
| 2.0.x                | 2.0.4                            |

####  创建一个 Spring Boot 项目：

使用 Spring Initializr（https://start.spring.io）或 (https://start.aliyun.com/)

- 项目名称：`my-service-v1`（用于 1.0.0），后续复制为 `my-service-v2`（1.0.1）。
- 依赖：Spring Web、Spring Cloud Alibaba Nacos Discovery。
- 添加以下依赖到 `pom.xml`（确保版本一致）：

```xml
  <properties>
        <java.version>17</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <spring-boot.version>3.0.2</spring-boot.version>
        <spring-cloud-alibaba.version>2022.0.0.0</spring-cloud-alibaba.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
            <exclusions>
                <!-- 排除 Spring Cloud Alibaba 默认绑定的低版本 Nacos 客户端（2.2.x） -->
                <exclusion>
                    <groupId>com.alibaba.nacos</groupId>
                    <artifactId>nacos-client</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!-- 手动引入与服务端匹配的 Nacos 客户端 2.5.1 -->
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-client</artifactId>
            <version>2.5.1</version> <!-- 与你的 Nacos 服务端版本一致 -->
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

#### 版本 1.0.0 配置

`src/main/resources/application.yml`：

```yaml
server:
  port: 8081
spring:
  application:
    name: my-service-v1  # 服务名称，必须一致以便网关发现
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848  # Nacos 地址
#        namespace: gray-release  # 如果创建了自定义命名空间，使用此配置；否则删除此行，使用默认 public
        metadata:
          version: 1.0.0  # 元数据，用于版本区分
app:
  version: 1.0.0
logging:
  level:
    com.alibaba.nacos: DEBUG  # 启用 Nacos 注册日志，便于调试

```

#### 版本 1.0.1 配置

`src/main/resources/application.yml`：

```yaml
server:
  port: 8082
spring:
  application:
    name: my-service-v2  # 服务名称，必须一致以便网关发现
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848  # Nacos 地址
#        namespace: gray-release  # 如果创建了自定义命名空间，使用此配置；否则删除此行，使用默认 public
        metadata:
          version: 1.0.1  # 元数据，用于版本区分
app:
  version: 1.0.1
logging:
  level:
    com.alibaba.nacos: DEBUG  # 启用 Nacos 注册日志，便于调试

```

- 将端口改为 8082。
- 将 metadata.version 和 app.version 改为 1.0.1。
- 确保 namespace 配置一致。

#### 创建 REST 控制器

`TestController.java`（在同一包下）：

```java
@RestController
@RequestMapping("/api")
public class TestController {
    @Value("${app.version}")
    private String version;

    @GetMapping("/test")
    public String test() {
        return "Version: " + version;
    }
}
```

#### **启动并验证注册**

- 在 IDE 中运行 `MyServiceApplication`（或命令行 `mvn spring-boot:run`）。

- 检查控制台日志：搜索 "Nacos" 相关输出，例如：

    - "nacos registry, DEFAULT_GROUP my-service-v1 192.168.159.1:8081 register finished"
    - 如果看到错误，如 "ConnectException"，检查 Nacos 是否启动、地址是否正确、网络连接。

  ![image-20250825170759011](https://cdn.wcxian.cc/img/20250827215611735.png)

- 打开 Nacos 控制台 &gt; “服务管理” &gt; “服务列表”，搜索 `my-service`：

    - 应看到服务名称 `my-service-v1`，点击详情查看实例列表（IP:端口，如 127.0.0.1:8081）。
    - 在实例详情中，查看 “元数据” 列，应有 `version: 1.0.0`。

  ![image-20250825170943848](https://cdn.wcxian.cc/img/20250827215615839.png)



#### 重复上述步骤启动版本 1.0.1，确认两个实例均注册成功

![image-20250825171138686](https://cdn.wcxian.cc/img/20250827215624205.png)

## 步骤 3：创建 Spring Cloud Gateway

网关负责根据请求特征路由到新旧版本，并通过 Nacos 动态管理路由配置。

### 创建 Gateway 项目

#### 配置依赖

- 添加依赖到 `pom.xml`（类似服务项目，但添加 Gateway 和 Nacos Config）：

```xml
    <properties>
        <java.version>17</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <spring-boot.version>3.0.2</spring-boot.version>
        <spring-cloud-alibaba.version>2022.0.0.0</spring-cloud-alibaba.version>
        <spring-cloud.version>2022.0.1</spring-cloud.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
            <exclusions>
                <!-- 排除 Spring Cloud Alibaba 默认绑定的低版本 Nacos 客户端（2.2.x） -->
                <exclusion>
                    <groupId>com.alibaba.nacos</groupId>
                    <artifactId>nacos-client</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!-- 手动引入与服务端匹配的 Nacos 客户端 2.5.1 -->
        <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-client</artifactId>
            <version>2.5.1</version> <!-- 与你的 Nacos 服务端版本一致 -->
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

#### 配置 `application.yml`

```yaml
server:
  port: 8083
spring:
  application:
    name: gateway
  config:
    import: "optional:nacos:gateway.yaml"
  cloud:

    nacos:
      config:
        server-addr: localhost:8848
        file-extension: yaml
        name: gateway
        extension-configs:
          - data-id: gateway.yaml
            group: DEFAULT
            refresh: true
      discovery:
        server-addr: localhost:8848
logging:
  level:
    com.alibaba.nacos: DEBUG
    org.springframework.cloud.gateway: DEBUG  # 启用 Gateway 日志
    org.springframework.cloud.loadbalancer: TRACE  # 更详细的负载均衡日志

# 开启健康检查端点
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: always
```

#### Nacos新增配置

在 Nacos 控制台 &gt; “配置管理” &gt; “配置列表”，新建配置：

- Data ID: `gateway.yaml`
- Group: `DEFAULT_GROUP`
- 格式: YAML
- 内容：

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: false  # 必须为false
      routes:
      - id: gray_route
        uri: lb://my-service-v2
        predicates:
        - Path=/api/**
        - Weight=group1, 50 #使用权重路由
        #- Header=gray-version, 1.0.1  # 精确匹配header        
        metadata:
          version: 1.0.1
      - id: stable_route
        uri: lb://my-service-v1
        predicates:
        - Path=/api/**
        - Weight=group1, 50
        metadata:
          version: 1.0.0
```

- 保存后，确认配置发布成功。

#### 创建自定义 Predicate

`GrayRoutePredicateFactory.java`

```java
import org.springframework.cloud.gateway.handler.predicate.AbstractRoutePredicateFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;

import java.util.function.Predicate;

@Component
public class GrayRoutePredicateFactory extends AbstractRoutePredicateFactory<GrayRoutePredicateFactory.Config> {
    public GrayRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return exchange -> {
            String userId = exchange.getRequest().getHeaders().getFirst("user-id");
            if (userId != null) {
                return Math.abs(userId.hashCode()) % 100 < config.getGrayPercentage();
            }
            return false;
        };
    }

    public static class Config {
        private int grayPercentage = 0;

        public int getGrayPercentage() {
            return grayPercentage;
        }

        public void setGrayPercentage(int grayPercentage) {
            this.grayPercentage = grayPercentage;
        }
    }
}
```

#### **启动并验证**

- 运行 Gateway，检查日志：确认从 Nacos 加载配置（如 "Pulling config from Nacos"），并注册到 Nacos。

- 在 Nacos 服务列表中，应看到 `gateway` 服务。

- 如果配置未加载，检查 application.yml 中的 config.server-addr 和 namespace。

  ![image-20250825180823671](https://cdn.wcxian.cc/img/20250827215637510.png)

## 步骤 4：测试灰度发布

1. 确保 Nacos 运行，服务 1.0.0 和 1.0.1 已启动并注册。

2. 启动 Gateway。

3. 测试命令：

    - 稳定版：`curl http://localhost:8083/api/test`（预期响应 "Version: 1.0.0"）

    - 灰度版：`curl -H "gray-version: 1.0.1" http://localhost:8083/api/test`（预期 "Version: 1.0.1"）

    - 动态分配：`curl -H "user-id: 123" http://localhost:8083/api/test`（根据哈希，部分请求到 1.0.1）

      ```yaml
      - Weight=group1, 50
      ##基于权重
      Administrator@KJSD MINGW64 /d/My Project/Spring-Cloud-HDFB/nacos/conf
      $ curl http://localhost:8083/api/test
      Version: 1.0.1
      Administrator@KJSD MINGW64 /d/My Project/Spring-Cloud-HDFB/nacos/conf
      $ curl http://localhost:8083/api/test
      Version: 1.0.0
      
      
      ```

      ![PixPin_2025-08-27_21-48-58](https://cdn.wcxian.cc/img/20250827215646568.gif)

      ```yaml
      - Header=gray-version, 1.0.1
      ##基于header,精确匹配header
      $ curl -H "gray-version: 1.0.1" http://localhost:8083/api/test
      Version: 1.0.1
      
      - name: Gray
        args:
          grayPercentage: 50
      ##基于user-id hash
      Administrator@KJSD MINGW64 /d/My Project/Spring-Cloud-HDFB/nacos/conf
      $ curl -H "user-id: 123" http://localhost:8083/api/test
      Version: 1.0.0
      Administrator@KJSD MINGW64 /d/My Project/Spring-Cloud-HDFB/nacos/conf
      $ curl -H "user-id: 1" http://localhost:8083/api/test
      Version: 1.0.1
      ```

      ![PixPin_2025-08-27_21-53-07](https://cdn.wcxian.cc/img/20250827215651089.gif)

      ```yaml
      routes:
      - id: cookie_gray
        uri: lb://my-service2
        predicates:
        - Path=/api/**
        - Cookie=version, 1.0.1  # 基于Cookie值分配
        filters:
        - AddResponseHeader=X-Version, 1.0.1
      
      routes:
      - id: query_param_gray
        uri: lb://my-service2
        predicates:
        - Path=/api/**
        - Query=version, 1.0.1  # 基于URL参数分配
        filters:
        - AddResponseHeader=X-Version, 1.0.1
      
      routes:
      - id: host_gray
        uri: lb://my-service2
        predicates:
        - Path=/api/**
        - Host=gray.example.com  # 基于域名分配
        filters:
        - AddResponseHeader=X-Version, 1.0.1
      
      routes:
      - id: method_gray
        uri: lb://my-service2
        predicates:
        - Path=/api/**
        - Method=POST,PUT  # 只对特定HTTP方法灰度
        filters:
        - AddResponseHeader=X-Version, 1.0.1
      
      routes:
      - id: time_gray
        uri: lb://my-service2
        predicates:
        - Path=/api/**
        - Between=2024-01-01T00:00:00.000+08:00, 2024-12-31T23:59:59.000+08:00 #基于时间段的分配
        filters:
        - AddResponseHeader=X-Version, 1.0.1
      
      routes:
      - id: ip_gray
        uri: lb://my-service2
        predicates:
        - Path=/api/**
        - RemoteAddr=192.168.1.1/24  # 特定IP段
        filters:
        - AddResponseHeader=X-Version, 1.0.1
      
      routes:
      - id: complex_gray
        uri: lb://my-service2
        predicates:
        - Path=/api/**
        - Header=gray-version, 1.0.1
        - Cookie=user-type, vip       # AND条件：必须同时满足
        - Method=GET,POST
        filters:
        - AddResponseHeader=X-Version, 1.0.1
      
      routes:
      - id: canary_phase1
        uri: lb://my-service2
        predicates:
        - Path=/api/**
        - Weight=canary-group, 5  # 第一阶段：5%流量
        filters:
        - AddResponseHeader=X-Version, 1.0.1
      
      - id: canary_phase2  
        uri: lb://my-service2
        predicates:
        - Path=/api/**
        - Weight=canary-group, 20 # 第二阶段：20%流量
        filters:
        - AddResponseHeader=X-Version, 1.0.1
      
      - id: canary_phase3
        uri: lb://my-service2
        predicates:
        - Path=/api/**
        - Weight=canary-group, 50 # 第三阶段：50%流量
        filters:
        - AddResponseHeader=X-Version, 1.0.1
      
      ```

      ```java
      // 自定义谓词工厂
      public class BusinessGrayPredicateFactory extends AbstractRoutePredicateFactory<Config> {
          
          @Override
          public Predicate<ServerWebExchange> apply(Config config) {
              return exchange -> {
                  // 基于用户ID哈希
                  String userId = exchange.getRequest().getHeaders().getFirst("user-id");
                  if (userId != null) {
                      int hash = userId.hashCode() % 100;
                      return hash < config.percentage; // 百分比控制
                  }
                  
                  // 基于设备类型
                  String device = exchange.getRequest().getHeaders().getFirst("device-type");
                  return "ios".equals(device); // 只对iOS设备
                  
                  // 基于地理位置等其他业务逻辑
              };
          }
      }
      
      ```



4. **动态调整**：在 Nacos 修改 `gateway.yaml`（如 Gray=50），保存发布。Gateway 日志显示刷新，测试验证新比例。



## 步骤 5：监控与回滚

1. **监控**：在服务 `pom.xml` 添加 Actuator 依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

- 更新 `application.yml`：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,info
```

- 测试：`curl http://localhost:8081/actuator/health`（{"status":"UP"}）

  ![image-20250825195208357](https://cdn.wcxian.cc/img/20250827215658826.png)

2. **回滚**：在 Nacos 修改 `gateway.yaml`，移除 gray_route，保存。Gateway 自动刷新，流量全回旧版。

3. **日志增强**：在控制器添加日志：

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
// ... 
private static final Logger log = LoggerFactory.getLogger(TestController.class);

@GetMapping("/test")
public String test() {
    log.info("Request handled by version: {}", version);
    return "Version: " + version;
}
```

## 注意事项

- **常见注册失败原因**：命名空间不匹配、Nacos 地址错误、依赖冲突、端口占用。优先检查日志。
- **生产环境**：启用 Nacos 集群模式，添加认证；使用 Sentinel 集成限流。
- **扩展**：如果需要权重-based 路由，可结合 Nacos 的权重配置（在服务实例编辑权重）。
- **调试技巧**：使用 IDE 断点在 NacosClientService 中调试注册过程。

## 其他实现方式

1. 基于 Nginx 的灰度发布

   ```nginx
   http {
       upstream my_service {
           server 127.0.0.1:8081; # 1.0.0
           server 127.0.0.1:8082; # 1.0.1
       }
   
       split_clients "${remote_addr}" $version {
           10% 1.0.1; # 10% 流量到新版本
           *   1.0.0; # 其余到旧版本
       }
   
       server {
           listen 80;
           location /api/ {
               proxy_pass http://my_service;
               proxy_set_header X-Version $version;
           }
       }
   }
   ```

2. 基于 Kubernetes 的灰度发布

   **适用场景**：适合容器化部署的 Spring Boot 应用，结合 Kubernetes 的 Service 和 Ingress 或服务网格（如 Istio）实现流量分发。

   **实现思路**：

    - 使用 Kubernetes 部署新旧版本的 Spring Boot 应用，通过标签（labels）区分版本。
    - 配置 Kubernetes Service 或 Ingress 规则，根据请求特征（如请求头、路径）分发流量。
    - 结合 Istio 或 Linkerd 等服务网格实现更精细的流量控制（如权重分配、请求头匹配）。

3. 基于 Spring Cloud Zuul 的灰度发布

   **适用场景**：适用于已使用 Spring Cloud 生态（但非 Alibaba）的项目，Zuul 1.x 或 2.x 作为网关实现灰度发布。

   **实现思路**：

    - 使用 Zuul 网关替代 Spring Cloud Gateway，结合 Eureka 或 Nacos 进行服务发现。

    - 配置 Zuul 路由规则，根据请求头或参数分发流量到不同版本。

    - 使用 Zuul 过滤器实现自定义分流逻辑。

      ```yaml
      server:
        port: 8080
      spring:
        application:
          name: zuul-gateway
        cloud:
          nacos:
            discovery:
              server-addr: localhost:8848
      zuul:
        routes:
          my-service:
            path: /api/**
            serviceId: my-service
      ```

4. 基于 Sentinel 的灰度发布

   **适用场景**：在 Spring Cloud Alibaba 生态中，结合 Sentinel（流量控制和限流框架）实现灰度发布，适合需要流量保护的场景。

   **实现思路**：

    - 使用 Sentinel 定义流量规则，根据请求参数或用户 ID 分配到新旧版本。
    - 结合 Nacos 存储规则，动态调整。

## 总结

本教程提供详细步骤，确保服务注册成功。通过 Nacos 的动态配置，实现无重启灰度发布。