---
title: Skills 在 Spring Boot 中的应用
published: 2026-02-11
description: Anthropic Skills 为 Spring Boot 应用带来了全新的 AI 能力扩展方式。本文详细介绍 Skills 的概念、架构设计、实现方式及实践案例。
image: https://cdn.wcxian.cc/img/20260211102752855.png
tags: [Spring Boot, AI, Spring AI, Anthropic, Claude, Skills]
category: 技术开发
draft: false
lang: zh-CN
---

# Skills 在 Spring Boot 中的应用

## 什么是 Skills

**Skills**（技能）是 Anthropic 在 **2025年10月**推出的革命性功能，它是一个**技能封装系统**，由模块化、可定制的指令包组成，旨在增强 Claude AI 执行专业任务的能力，它本质上是给 AI Agent 的**专业技能包**或**结构化说明书**。

**核心理念**：将特定的工作流、专业领域知识或行业规范封装成定制化的 Skill。

**对比优势**：

- **相较于 System Prompt**：Skills 更有组织性，避免了长提示词带来的逻辑混乱。
- **相较于 MCP**：Skills 结构更轻量，且显著降低了 **Token 消耗**，因为它是按需加载的。

### 核心特性

![PixPin_2026-01-25_13-48-58](https://cdn.wcxian.cc/img/20260125143135908.png)

Skills 可以理解为包含以下内容的**文件夹**：

- 📜 元数据 (Metadata) 类比书的"目录"。这部分始终加载在 AI 的提示词中，体积小，Token 消耗低。

- 📝 指令（Instructions）类比书的"正文"。仅当 AI 判定需要该技能时，才会读取详细指令。
- 📦 资源（Resources）类比书的"附录"。包含 Python 脚本、外部数据或模板，由 AI 在执行过程中调用。

比如这是一个需求分析的技能目录

![PixPin_2026-01-25_13-39-37](https://cdn.wcxian.cc/img/20260125143141433.png)

这是skill.md的内容

![PixPin_2026-01-25_14-29-17](https://cdn.wcxian.cc/img/20260125143144841.png)

这些技能可以被 Claude **动态发现和加载**，分为两种类型：

| 类型 | 说明 | 示例 |
| :--- | :--- | :--- |
| **预构建技能** | 由 Anthropic 官方管理和维护 | 文档格式化、代码审查、数据分析 |
| **自定义技能** | 用户根据自己的需求创建 | 企业流程、个人工作流、领域专业知识 |

### 与 MCP 的关系

Skills 是继 **MCP（Model Context Protocol）**之后的又一重要创新。如果说 MCP 解决了"AI 如何访问外部工具"的问题，那么 Skills 则解决了"AI 如何快速掌握专业技能"的问题。

**对比分析：**

```
MCP（模型上下文协议）
├── 作用：让 AI 能够访问数据库、API、文件系统
└── 定位：工具调用能力的扩展

Skills（技能系统）
├── 作用：让 AI 快速掌握领域知识和工作流程
└── 定位：专业能力的封装和复用
```

**两者配合使用**，可以构建出真正懂业务、能干活的 AI 智能体。

---

## 使用场景

Skills 在实际业务中的应用场景非常广泛：

### 1. 企业文档处理
- **品牌规范应用**：自动将公司品牌指南应用到文档
- **合同审查**：按照法务部门的标准审查合同条款
- **技术文档生成**：生成符合公司规范的 API 文档

### 2. 业务流程自动化
- **客户服务**：构建企业级智能客服系统
- **数据分析**：数据库查询和数据可视化
- **报告生成**：自动生成周报、月报等业务报告

### 3. 开发提效
- **代码审查**：按照团队规范进行代码审查
- **测试用例生成**：根据业务逻辑生成测试用例
- **API 文档编写**：自动生成和维护 API 文档

### 4. 个人工作流
- **邮件分类**：按个人规则自动分类邮件
- **笔记整理**：将会议记录整理成结构化文档
- **学习助手**：创建个性化的学习计划

## 实践

这边使用skills有很多种方式，可以使用trae，cusor，codex之类，我这边直接使用claude code(之前的文档有介绍，可以去看一下)。

直接在你的项目目录把skiil文件夹放进去，C:\Users\Purezento\Desktop\财报分析\.claude\skills\技能文件夹

### 场景一：深度的财报数据分析与可视化

下载了一个anthropic官方的一个财报分析技能 **analyzing-financial-statements**

[claude-cookbooks/skills/custom_skills/analyzing-financial-statements at main · anthropics/claude-cookbooks](https://github.com/anthropics/claude-cookbooks/tree/main/skills/custom_skills/analyzing-financial-statements)

这里面他包含了一个技能文档和2个python脚本

![PixPin_2026-01-25_14-25-17](https://cdn.wcxian.cc/img/20260125143150642.png)

我找了一个腾讯2025年中期报告，让他分析一下

https://static.www.tencent.com/uploads/2025/08/26/00175ef813605d01873b4a533131aed0.pdf

![image-20260211102740636](https://cdn.wcxian.cc/img/20260211102752855.png)

使用Claude Code ，直接让他分析，会自己根据你的提示词调用skills，也可以显性的调用用**/analyzing-financial-statements**

![PixPin_2026-01-25_14-39-05](https://cdn.wcxian.cc/img/20260125150237015.png)

出现了这个skill就代表他开始使用技能了

这边他创建了2个python脚本来计算

![PixPin_2026-01-25_14-41-27](https://cdn.wcxian.cc/img/20260125150240005.png)

![PixPin_2026-01-25_14-43-52](https://cdn.wcxian.cc/img/20260125150242155.png)

最终生成一个md文档

![PixPin_2026-01-25_15-01-32](https://cdn.wcxian.cc/img/20260125150245108.png)

### 场景二：使用UI UX Pro Max Skills 设计一个天气卡片

[UI/UX Pro Max - AI-Powered Design Intelligence](https://www.uupm.cc/)

![image-20260211104206297](https://cdn.wcxian.cc/img/20260211104233076.png)

![image-20260211104447344](https://cdn.wcxian.cc/img/20260211110814958.png)

```shell
# 全局安装 CLI 工具
npm install -g uipro-cli

# 进入你的项目目录
cd /path/to/your/project

# 为指定 AI 助手安装技能
uipro init --ai claude      # Claude Code
uipro init --ai cursor      # Cursor
uipro init --ai windsurf    # Windsurf
uipro init --ai antigravity # Antigravity (.agent + .shared)
uipro init --ai copilot     # GitHub Copilot
uipro init --ai kiro        # Kiro
uipro init --ai all         # 所有 AI 助手
```

这时候就会在你的项目目录下生成skills文件夹，包含提示词和python脚本

![image-20260211110709421](https://cdn.wcxian.cc/img/20260211110819386.png)

```
你是一位就职于苹果公司的顶级前端工程师。请创建一个包含CSS和JavaScript的HTML文件，用于生成动画天气卡片。卡片需要以不同动画效果直观展示以下天气状况：

风力（如：飘动的云朵、摇曳的树木或风线）
降雨（如：下落的雨滴、形成的水洼）
晴天（如：闪耀的光线、明亮的背景）
下雪（如：飘落的雪花、积雪效果）

所有天气卡片需要并排显示，背景采用深色设计。所有HTML、CSS和JavaScript代码都需包含在这个单一文件中。JavaScript部分需包含切换不同天气状态的功能（例如通过函数或按钮组），以演示每种天气的动画效果。

将前端显示效果优化得更精致流畅，打造出价值20元/月的精品天气应用既视感。
```

![image-20260211105131650](https://cdn.wcxian.cc/img/20260211110822545.png)

![image-20260211105900047](https://cdn.wcxian.cc/img/20260211110825693.png)

生成的效果

![PixPin_2026-02-11_11-04-39](https://cdn.wcxian.cc/img/20260211110828315.gif)

---

## 一、环境准备

### 1.1 技术栈要求

**核心依赖版本：**

```xml
<properties>
    <java.version>17</java.version>
    <spring-boot.version>3.4.4</spring-boot.version>
    <spring-ai.version>1.0.0-M6</spring-ai.version>
</properties>
```

**关键点说明：**
- ✅ **JDK 17+**：Skills 需要 Java 17 或更高版本
- ✅ **Spring Boot 3.x**：需要 Spring Boot 3.x 版本
- ✅ **Spring AI 1.0.0-M6**：这是支持 Agent Skills 的稳定版本

### 1.2 Maven 依赖配置

```xml
<dependencies>
    <!-- Spring AI 核心依赖 -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-core</artifactId>
        <version>${spring-ai.version}</version>
    </dependency>

    <!-- Spring AI OpenAI 集成（兼容 DeepSeek 等） -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
        <version>${spring-ai.version}</version>
    </dependency>

    <!-- Spring AI Anthropic 集成 -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-anthropic-spring-boot-starter</artifactId>
        <version>${spring-ai.version}</version>
    </dependency>
</dependencies>
```

**添加 Milestone 仓库（因为 Spring AI 还在快速迭代中）：**

```xml
<repositories>
    <repository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>
```

### 1.3 application.yml 配置

```yaml
spring:
  ai:
    # Anthropic 配置（使用 Claude 模型）
    anthropic:
      chat:
        enabled: true
        options:
          model: claude-sonnet-4-20250514  # 或者 claude-opus-4-20250514
      api-key: ${ANTHROPIC_API_KEY}  # 环境变量配置

    # 或者使用 DeepSeek（更经济的选择）
    openai:
      chat:
        enabled: true
        options:
          model: deepseek-chat  # DeepSeek 支持 Tools 调用
      base-url: https://api.deepseek.com
      api-key: ${DEEPSEEK_API_KEY}

    # Agent Skills 配置
    agent:
      skills-enabled: true  # 启用 Skills 功能
      skills-directory: classpath:skills/  # Skills 文件目录
```

---

## 二、Skills 的实现方式

### 2.1 架构设计

Spring AI 与 Skills 的集成遵循以下架构：

```
┌─────────────────────────────────────────────────────┐
│                   Spring Boot 应用                   │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌─────────────────────────────────────────────┐  │
│  │         ChatClient (AI 对话客户端)            │  │
│  └──────────────┬──────────────────────────────┘  │
│                 │                                   │
│  ┌──────────────▼──────────────────────────────┐  │
│  │      ToolCallbackProvider (工具注册)         │  │
│  └──────────────┬──────────────────────────────┘  │
│                 │                                   │
│  ┌──────────────▼──────────────────────────────┐  │
│  │         Skills (技能封装层)                  │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │  │
│  │  │ Skill 1  │  │ Skill 2  │  │ Skill N  │  │  │
│  │  │ (数据分析)│  │(文档生成)│  │(代码审查)│  │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  │  │
│  └───────┼────────────┼────────────┼──────────┘  │
│          │            │            │              │
│  ┌───────▼────────────▼────────────▼──────────┐  │
│  │      底层工具实现（数据库、API、脚本等）      │  │
│  └─────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### 2.2 Skills 目录结构

创建 Skills 目录结构：

```
src/main/resources/
└── skills/
    ├── data-analysis/
    │   ├── skill.md          # 技能定义文件
    │   ├── tools.py          # Python 脚本（可选）
    │   └── schema.json       # 数据结构定义（可选）
    │
    ├── document-generator/
    │   ├── skill.md
    │   ├── template.md       # 文档模板
    │   └── examples/         # 示例文档
    │
    └── code-reviewer/
        ├── skill.md
        ├── checkstyle.xml    # 代码规范配置
        └── rules/            # 审查规则
```

---

## 三、实现一个 Skills：数据分析助手

### 3.1 创建 Skill 定义文件

**文件路径：** `src/main/resources/skills/data-analysis/skill.md`

```markdown
# Data Analysis Skill

## 触发条件
当用户需要以下操作时，自动激活此技能：
- 查询数据库中的用户数据
- 生成统计报表
- 分析业务趋势
- 导出数据到 Excel

## 可用工具

### 1. queryUsers
- 描述：根据条件查询用户信息
- 参数：name（用户姓名，可选）、department（部门，可选）
- 返回：用户列表

### 2. generateReport
- 描述：生成数据统计报表
- 参数：startDate、endDate、reportType（daily/weekly/monthly）
- 返回：报表数据

### 3. exportToExcel
- 描述：将数据导出为 Excel 文件
- 参数：data、filename
- 返回：文件下载链接

## 执行流程
1. 理解用户需求
2. 选择合适的工具
3. 执行查询或分析
4. 格式化输出结果
```

### 3.2 实现 Java 工具类

**创建数据查询工具：**

```java
package com.example.skills.tools;

import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

import java.util.List;
import java.util.Map;

/**
 * 数据分析技能工具类
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class DataAnalysisTools {

    private final UserService userService;
    private final ReportService reportService;

    /**
     * 查询用户信息
     */
    @Tool(name = "queryUsers", description = "根据条件查询用户信息")
    public List<Map<String, Object>> queryUsers(
            @ToolParam(description = "用户姓名（模糊查询）") String name,
            @ToolParam(description = "部门名称") String department
    ) {
        log.info("执行用户查询：name={}, department={}", name, department);

        // 调用业务服务查询数据
        List<Map<String, Object>> users = userService.queryUsers(name, department);

        log.info("查询到 {} 条用户记录", users.size());
        return users;
    }

    /**
     * 生成统计报表
     */
    @Tool(name = "generateReport", description = "生成数据统计报表")
    public Map<String, Object> generateReport(
            @ToolParam(description = "开始日期，格式：yyyy-MM-dd") String startDate,
            @ToolParam(description = "结束日期，格式：yyyy-MM-dd") String endDate,
            @ToolParam(description = "报表类型：daily/weekly/monthly") String reportType
    ) {
        log.info("生成报表：startDate={}, endDate={}, type={}", startDate, endDate, reportType);

        // 生成报表数据
        Map<String, Object> report = reportService.generateReport(startDate, endDate, reportType);

        log.info("报表生成成功");
        return report;
    }

    /**
     * 导出数据到 Excel
     */
    @Tool(name = "exportToExcel", description = "将数据导出为 Excel 文件")
    public String exportToExcel(
            @ToolParam(description = "要导出的数据列表") List<Map<String, Object>> data,
            @ToolParam(description = "文件名称（不含扩展名）") String filename
    ) {
        log.info("导出数据到 Excel：filename={}, 数据量={}", filename, data.size());

        // 导出 Excel
        String downloadUrl = reportService.exportToExcel(data, filename);

        log.info("Excel 导出成功：{}", downloadUrl);
        return downloadUrl;
    }
}
```

### 3.3 注册 Skills 到 ChatClient

**创建 Skills 配置类：**

```java
package com.example.skills.config;

import com.example.skills.tools.DataAnalysisTools;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.tool.MethodToolCallbackProvider;
import org.springframework.ai.tool.ToolCallbackProvider;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import lombok.extern.slf4j.Slf4j;

/**
 * Skills 配置类
 */
@Slf4j
@Configuration
public class SkillsConfig {

    /**
     * 注册数据分析技能
     */
    @Bean
    public ToolCallbackProvider dataAnalysisSkillProvider(DataAnalysisTools dataAnalysisTools) {
        log.info("开始注册数据分析技能...");

        MethodToolCallbackProvider provider = MethodToolCallbackProvider.builder()
                .toolObjects(dataAnalysisTools)
                .build();

        // 打印注册的工具信息
        for (var tool : provider.getToolCallbacks()) {
            log.info("已注册工具：{} - {}",
                    tool.getToolDefinition().name(),
                    tool.getToolDefinition().description());
        }

        return provider;
    }

    /**
     * 创建数据分析 ChatClient
     */
    @Bean
    public ChatClient dataAnalysisChatClient(
            ChatClient.Builder builder,
            ToolCallbackProvider dataAnalysisSkillProvider
    ) {
        return builder
                .defaultSystem("""
                        你是一个专业的数据分析助手。
                        你可以帮助用户：
                        1. 查询数据库中的用户信息
                        2. 生成各类统计报表
                        3. 将数据导出为 Excel 文件

                        请使用简洁、专业的语言与用户交流。
                        当提供数据时，请用表格或列表的形式展示，确保易读性。
                        """)
                .defaultTools(dataAnalysisSkillProvider)
                .build();
    }
}
```

### 3.4 创建 API 接口

```java
package com.example.skills.controller;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.web.bind.annotation.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

/**
 * 数据分析 API 控制器
 */
@Slf4j
@RestController
@RequestMapping("/api/data-analysis")
@RequiredArgsConstructor
public class DataAnalysisController {

    private final ChatClient dataAnalysisChatClient;

    @PostMapping("/chat")
    public String chat(@RequestBody String userMessage) {
        log.info("用户请求：{}", userMessage);

        String response = dataAnalysisChatClient.prompt()
                .user(userMessage)
                .call()
                .content();

        log.info("AI 响应：{}", response);
        return response;
    }
}
```

---

## 四、功能验证

### 4.1 启动项目

启动 Spring Boot 应用，观察日志输出：

![img](https://cdn.wcxian.cc/img/20260125154021828.png)

### 4.2 测试案例

**测试1：查询用户信息**

**请求：**
```json
POST /api/data-analysis/chat

{
  "userMessage": "帮我查询技术部的所有用户"
}
```

**AI 响应：**

![PixPin_2026-01-25_15-48-40](https://cdn.wcxian.cc/img/20260125160249313.png)

![img](https://cdn.wcxian.cc/img/20260125160253021.png)

**测试2：生成报表**

**请求：**
```json
POST /api/data-analysis/chat

{
  "userMessage": "生成2025年1月的月度报表"
}
```

**AI 响应：**

![img](https://cdn.wcxian.cc/img/20260125160257380.png)

**测试3：导出 Excel**

**请求：**
```json
POST /api/data-analysis/chat

{
  "userMessage": "把技术部用户导出成Excel"
}
```

**AI 响应：**

![PixPin_2026-01-25_15-50-13](https://cdn.wcxian.cc/img/20260125160300353.png)

导出的表格

![PixPin_2026-01-25_15-51-35](https://cdn.wcxian.cc/img/20260125160303391.png)

### 4.3 执行流程图

```
用户输入
    │
    ├─→ Claude 理解意图
    │       │
    │       ├─→ 判断需要哪个 Skill
    │       │
    │       └─→ 选择合适的 Tool
    │               │
    │               ├─→ Tool 1: queryUsers
    │               ├─→ Tool 2: generateReport
    │               └─→ Tool 3: exportToExcel
    │                       │
    │                       └─→ 执行 Java 方法
    │                               │
    │                               ├─→ 查询数据库
    │                               ├─→ 调用业务服务
    │                               └─→ 返回结果
    │                                       │
    └─→ Claude 格式化输出
            │
            └─→ 返回给用户
```

---

## 五、高级用法：多 Skills 组合

### 5.1 创建多个技能

在同一个应用中可以定义多个独立的 Skills：

```
skills/
├── data-analysis/      # 数据分析技能
│   └── skill.md
├── document-writer/    # 文档编写技能
│   └── skill.md
└── code-reviewer/      # 代码审查技能
    └── skill.md
```

### 5.2 多 Skills 配置

```java
@Configuration
public class MultiSkillsConfig {

    @Bean
    public ToolCallbackProvider dataAnalysisSkillProvider(DataAnalysisTools tools) {
        return MethodToolCallbackProvider.builder()
                .toolObjects(tools)
                .build();
    }

    @Bean
    public ToolCallbackProvider documentWriterSkillProvider(DocumentWriterTools tools) {
        return MethodToolCallbackProvider.builder()
                .toolObjects(tools)
                .build();
    }

    @Bean
    public ChatClient multiSkillChatClient(
            ChatClient.Builder builder,
            @Qualifier("dataAnalysisSkillProvider") ToolCallbackProvider dataSkills,
            @Qualifier("documentWriterSkillProvider") ToolCallbackProvider docSkills
    ) {
        return builder
                .defaultSystem("""
                        你是一个全能的 AI 助手，具备以下能力：

                        1. 数据分析：查询数据、生成报表、导出 Excel
                        2. 文档编写：生成 API 文档、编写技术方案
                        3. 代码审查：检查代码质量、提供优化建议

                        根据用户的需求，自动选择合适的技能完成任务。
                        """)
                .defaultTools(dataSkills, docSkills)
                .build();
    }
}
```

### 5.3 技能路由策略

当有多个 Skills 时，Claude 会自动判断使用哪个技能：

```java
// 示例：不同的用户请求会自动路由到不同的技能

"帮我查询销售数据"      → Data Analysis Skill
"生成 API 文档"        → Document Writer Skill
"审查这段代码"         → Code Reviewer Skill
"分析用户增长趋势"     → Data Analysis Skill
"编写接口使用说明"     → Document Writer Skill
```

---

## 六、技术要点解析

### 6.1 Skills vs 传统方法

| 对比维度 | 传统方式 | Skills 方式 |
| :--- | :--- | :--- |
| **能力封装** | 分散在各个 Service 中 | 集中在 Skill 文件夹中 |
| **AI 理解** | 需要大量 Prompt | 通过 skill.md 自动理解 |
| **复用性** | 依赖代码层面 | 跨项目、跨语言复用 |
| **维护性** | 代码和知识分离 | 统一管理 |
| **可移植性** | 绑定具体实现 | 抽象定义，可换底层实现 |

### 6.2 最佳实践

**1. Skill 定义原则**
- ✅ **单一职责**：一个 Skill 只做好一件事
- ✅ **清晰命名**：Tool 名称要见名知意
- ✅ **详细描述**：Description 要准确说明功能
- ✅ **参数验证**：对输入参数进行校验

**2. 目录组织**
```
skills/
└── {skill-name}/
    ├── skill.md          # 必需：技能定义
    ├── schema.json       # 可选：数据结构
    ├── tools.py          # 可选：脚本工具
    ├── templates/        # 可选：模板文件
    └── examples/         # 可选：示例数据
```

**3. 性能优化**
- 使用连接池管理数据库连接
- 大数据量查询使用分页
- 缓存频繁访问的数据
- 异步处理耗时操作

### 6.3 与 Spring AI 的深度集成

**Spring AI 对 Skills 的支持包括：**

1. **自动发现**：自动扫描 skills 目录
2. **动态加载**：运行时动态加载技能
3. **工具注册**：通过 `@Tool` 注解自动注册
4. **跨模型支持**：定义一次，支持多个 LLM

---

## 七、优势与局限

### 优势

| 优势 | 说明 |
| :--- | :--- |
| **快速上手** | 只需创建 skill.md 文件，无需复杂配置 |
| **标准化** | 统一的技能定义格式，便于团队协作 |
| **可扩展** | 支持自定义工具和脚本 |
| **跨模型**  | 支持 OpenAI、Anthropic、DeepSeek 等 |
| **Java 生态** | 与 Spring Boot 深度集成，适合企业应用 |

### 局限

| 局限 | 解决方案 |
| :--- | :--- |
| **模型依赖** | 需要支持 Tool Calls 的模型 | 选择 Claude、DeepSeek 等支持的模型 |
| **调试困难** | AI 调用过程不够透明 | 添加详细日志，使用 trace 工具 |
| **成本考量** | Claude API 调用成本较高 | 使用 DeepSeek 等替代方案 |

### 适用场景

✅ **推荐使用：**
- 企业内部工具集成
- 数据分析和报表生成
- 文档自动化生成
- 代码审查和质量检查

❌ **不推荐使用：**
- 简单的 CRUD 操作（直接写代码更高效）
- 对实时性要求极高的场景
- 需要精确控制的业务逻辑

---

## 八、未来展望

### 8.1 Skills 生态发展

随着 Anthropic 和 Spring AI 的持续投入，Skills 生态将迎来：

1. **更多预构建技能**：官方提供更多开箱即用的技能
2. **技能市场**：社区驱动的技能分享平台
3. **可视化编辑器**：无需编写 Markdown 即可创建技能
4. **性能优化**：更高效的技能加载和执行机制

### 8.2 企业级应用

Skills 在企业中的应用方向：

- **知识库集成**：结合 RAG 技术，构建企业知识问答系统
- **流程自动化**：将复杂的业务流程封装为 Skills
- **多智能体协作**：不同 Skills 之间的协同工作
- **私有化部署**：在企业内网部署专属 Skills

---

## 九、总结

Anthropic Skills 为 Spring Boot 应用带来了全新的 AI 能力扩展方式：

### 核心价值

1. **从通用到专业**：让 AI 快速掌握领域知识
2. **从代码到配置**：通过 skill.md 即可定义能力
3. **从单体到生态**：构建可复用的技能市场
4. **从概念到实践**：Spring AI 提供完整的 Java 支持

Skills 让 AI 不再是一个通用的聊天机器人，而是一个**懂业务、能干活的专业助手**。这正是 AI 从"玩具"走向"工具"的关键一步。

---

## 参考资料

> [通过API 使用Agent Skills - Claude 官方文档](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide)
>
> [Spring AI Agentic Patterns: Agent Skills - Spring 官方博客](https://spring.io/blog/2026/01/13/spring-ai-generic-agent-skills)
>
> [深入解析：LLM - Spring AI × Anthropic Skills - 博客园](https://www.cnblogs.com/ljbguanli/p/19477949)
>
> [掌握Claude Skills：2025年新手必备指南](https://help.apiyi.com/claude-skills-beginners-guide-2025.html)
>
> [Extend Claude with skills - Claude Code Docs](https://code.claude.com/docs/en/skills)
>
> [Building Effective Agents with Spring AI - Spring Blog](https://spring.io/blog/2025/01/21/spring-ai-agentic-patterns)
>
> [A Practical Guide to Building AI Agents With Java and Spring AI - Dev.to](https://dev.to/yuriybezsonov/a-practical-guide-to-building-ai-agents-with-java-and-spring-ai-part-1-create-an-ai-agent-4f4a)
