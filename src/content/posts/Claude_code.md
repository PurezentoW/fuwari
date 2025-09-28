---
title: Claude Code
published: 2025-09-28
description: 'Claude Code介绍和使用'
image: 'https://cdn.wcxian.cc/img/20250928150000109.png'
tags: [Claude Code,MCP,Vibe Code]
category: '技术分享'
draft: false 
lang: ''
---
# Claude Code

## 0. Vibe Coding

近几年，AI 编程工具快速迭代：

- **GitHub Copilot** —— 开启了“代码补全”的新纪元；
- **Cursor** —— 在 IDE 中融入 AI 助手，支持上下文感知；
- **Claude Code** —— 更进一步，直接作为**终端全栈 AI 开发伙伴**，从需求分析到部署全流程覆盖。

与此同时，**Vibe Coding** 概念由 Andrej Karpathy 于 2025 年提出，强调通过自然语言交互“让 AI 写代码”，开发者只需表达意图，而不必过度关注实现细节。这种模式迅速流行，成为 AI 编程浪潮中的新趋势。

Claude Code 正是在这一背景下应运而生，它不是简单的“代码补全”，而是一个能帮你完成整个功能模块甚至系统的智能开发代理。

## 1. Claude Code 简介

### Claude Code 的独特优势

- **端到端开发**：只需描述项目需求，Claude Code 即可生成前端界面、后端接口、数据库结构、测试用例和部署脚本的完整解决方案。
- **简洁高质量输出**：生成的生产级代码简练、规范，堪比高级开发者的工作成果。
- **无调用限制**：支持复杂多步骤任务，无 API 调用次数限制。
- **Vibe Coding 理念**：受 Andrej Karpathy 的“Vibe Coding”概念启发，Claude Code 让开发者专注于高层创意，AI 负责实现细节。

> **“这不是传统编程——我只是看看东西，说说东西，运行东西，复制粘贴东西，大部分都能工作。”**
> — Andrej Karpathy，描述 Vibe Coding 方式。

**Claude Code** 是 Anthropic 推出的终端 AI 编程工具。它直接在命令行运行，通过自然语言指令即可完成 **代码编写、调试、项目管理** 等任务。

### 与其他工具的对比

| 工具           | 核心能力                    | 局限性           | Claude Code 的优势                     |
| -------------- | --------------------------- | ---------------- | -------------------------------------- |
| GitHub Copilot | 智能补全、注释转代码        | 缺乏全局上下文   | Claude Code 有全局记忆与上下文         |
| Cursor         | IDE 集成、项目感知          | 主要局限于编辑器 | Claude Code 直接在终端运行，跨环境通用 |
| Claude Code    | 全流程 AI 开发（需求→部署） | 仍需人工验证结果 | 类全栈工程师，支持 200K 上下文         |

一句话总结：**Claude Code 不是助手，而是 AI 开发伙伴。**

## 2. 详细介绍

#### **核心能力矩阵**

Claude Code 通过强大的功能集重新定义了 AI 辅助开发体验，以下是其核心功能及新增内容的详细介绍：

| **功能**                 | **描述**                                                     |
| ------------------------ | ------------------------------------------------------------ |
| **记忆机制**             | 支持长期记忆（CLAUDE.md）、会话记忆和跨会话上下文保持，确保项目一致性。可以通过 CLAUDE.md 文件存储项目规范、历史任务记录和上下文信息，减少重复输入。 |
| **工具调用**             | 支持文件读写、Shell 命令执行、Web 搜索等工具调用。新增支持直接调用外部 API（如 Docker、Kubernetes）以实现自动化部署和容器管理。 |
| **代理式搜索**           | 按需搜索代码库，无需全量索引。新增功能包括智能代码补全建议、跨文件引用解析，以及基于语义的代码片段推荐，优化大型项目中的代码检索效率。 |
| **多层级上下文管理**     | 支持工程级、本地级和全局级配置管理。新增支持动态上下文切换，可根据任务需求自动调整上下文优先级（如优先使用项目级配置或全局默认设置）。 |
| **可视化能力**           | 自动生成流程图、架构图和 ASCII 图，便于理解项目结构。新增支持 UML 类图、时序图生成，以及交互式代码依赖图，便于团队协作和代码审查。 |
| **Git 集成**             | 自动生成提交信息，支持 git add/commit 自动化操作。新增功能包括自动分支管理、冲突检测与解决建议，以及基于代码变更的智能提交日志生成。 |
| **代码优化与重构**       | 自动识别代码中的冗余、低效或潜在 bug，并提供优化建议。支持一键重构代码以符合指定编码规范（如 PEP8、ESLint），提升代码可读性和性能。 |
| **多语言支持与框架适配** | 支持主流编程语言和框架，新增对新兴技术栈（如 Rust、Svelte、Django）的适配，提供开箱即用的模板和最佳实践。 |
| **自动化测试生成**       | 根据代码逻辑自动生成单元测试、集成测试和端到端测试用例。新增支持测试覆盖率分析和自动生成 Mock 数据，简化测试流程。 |

## 3. 安装指南

不管如何，先安装 Claude Code 再说，安装步骤如下：

#### **安装 Node.js**

- 从 nodejs.org 下载并安装最新版本的 Node.js，按照安装向导完成设置。https://nodejs.org/zh-cn/download
- 使用nvm安装 [coreybutler/nvm-windows: A node.js version management utility for Windows. Ironically written in Go.](https://github.com/coreybutler/nvm-windows)

![image-20250922214123857](https://cdn.wcxian.cc/img/20250928133521464.png)

#### **安装 Claude Code**

打开终端，运行以下命令全局安装 Claude Code：

```bash
npm install -g @anthropic-ai/claude-code
```

![image-20250928133612731](https://cdn.wcxian.cc/img/20250928135907072.png)

```bash
claude
```

![image-20250922214434687](https://cdn.wcxian.cc/img/20250928133754894.png)

Claude 访问比较麻烦，收费也比较贵，所以需要引入其他的工具

#### 配置 Claude Code Router (CCR)

**Claude Code Router** 就是这样一款强大的 Claude Code 开源代理工具，它可以将 Claude Code 的请求**路由到不同的大模型**，并支持自定义任何请求。

主要功能如下：

- **模型路由**：根据您的需求将请求路由到不同的模型（比如：后台任务、思考、长上下文）。
- **多提供商支持**：支持 OpenRouter、DeepSeek、Ollama、Gemini、Volcengine 和 SiliconFlow 等各种模型提供商。
- **请求/响应转换**：使用转换器为不同的提供商自定义请求和响应。
- **动态模型切换**：在 Claude Code 中使用 `/model` 命令动态切换模型。
- **GitHub Actions 集成**：在您的 GitHub 工作流程中触发 Claude Code 任务。
- **插件系统**：使用自定义转换器扩展功能。

[musistudio/claude-code-router](https://github.com/musistudio/claude-code-router)

![img](https://cdn.wcxian.cc/img/20250928133807123.png)

- **安装 CCR**：克隆并设置 CCR 仓库：

```bash
git clone https://github.com/musistudio/claude-code-router.git
cd claude-code-router
npm install
```

- **配置 API 访问**：在 ModelScope 注册账号并绑定阿里云，获取每日 2000 次免费 API 调用权限。获取 API 密钥并在 CCR 配置中设置。

- **示例配置文件**：在 CCR 目录下创建 config.json 文件：

```json
{
  "apiKey": "您的-modelscope-api-key",
  "defaultModel": "claude-4",
  "providers": ["ModelScope", "OpenRouter", "DeepSeek"]
}
```

- **运行 CCR**：启动路由器：

  ```bash
  node index.js
  ```

- 这边我使用一个配置脚本

```bash
npx zcf
```

![image-20250922215055121](https://cdn.wcxian.cc/img/20250928133841871.png)

![image-20250922215218853](https://cdn.wcxian.cc/img/20250928133846188.png)

配置CCR模型，这边我用了一个魔搭的api，每天有2000次免费的api调用

https://modelscope.cn/

注册一个账号，绑定一下阿里云，就能使用每日2000次的调用

![image-20250922220010810](https://cdn.wcxian.cc/img/20250928133900817.png)

![image-20250922215354266](https://cdn.wcxian.cc/img/20250928133906198.png)

#### 启动

```shell
claude
```

启动页面，这边就是通过代理接入我们配置的模型

![image-20250922220306579](https://cdn.wcxian.cc/img/20250928133917825.png)



## 4. 使用

### 初始化项目（CLAUDE.md ）

CLAUDE.md 文件是 Claude Code 的核心特性之一，充当项目专属的上下文指南，记录编码规范、Shell 命令、测试流程和项目特定说明，确保 AI 输出与项目需求高度一致。

- 在项目目录下运行以下命令生成 CLAUDE.md 文件：

  ```bash
  claude /init
  ```

- 生成的模板文件可自定义，包含以下内容：

    - 项目架构和依赖项。
    - 编码规范（例如命名、格式）。
    - 常用工作流（例如测试、部署脚本）。

- 示例 CLAUDE.md 内容：

  ```markdown
  # 项目指南
  ## 概述
  - 项目：用户认证系统
  - 语言：JavaScript, Node.js
  - 框架：Express, React
  
  ## 编码规范
  - 变量名使用 camelCase。
  - 遵循 RESTful API 规范。
  
  ## 常用命令
  - 启动服务：`npm run start`
  - 运行测试：`npm test`
  ```

你可以在项目根和子目录创建多个 `CLAUDE.md`，为每个上下文提供个性化配置。

| 文件路径                   | 作用                                                 |
| -------------------------- | ---------------------------------------------------- |
| 项目根目录/CLAUDE.md       | 团队共享的项目级配置，提交至 Git 供所有成员使用      |
| 项目根目录/CLAUDE.local.md | 个人本地覆盖配置，通常加入 .gitignore 避免影响他人   |
| 父目录/CLAUDE.md           | 在 Monorepo 结构中自动继承的上级配置（递归向上查找） |
| 子目录/CLAUDE.md           | 针对特定子模块/功能的独立配置（优先于父级配置加载）  |
| ~/.claude/CLAUDE.md        | 用户全局默认配置，适用于所有 Claude 会话的基线设定   |

![image-20250923112917998](https://cdn.wcxian.cc/img/20250928133935380.png)

**这边是我新建的一个空的springboot项目，直接生成的文档**

![image-20250923113346491](https://cdn.wcxian.cc/img/20250928133943694.png)

### 生成PRD

**先让他生成一份PRD**

```text
/generate-prp INITIAL.md
```

```bash
本项目是要做一个热点新闻的后端项目，通过采集网络上的新闻或者资讯热榜，做一个数据源汇总，你可以参考https://momoyu.cc/
这个项目，我的热榜目前包括知乎热榜，微博热搜，什么值得买热门，这些数据源会在我们服务器每半小时采集一次记录下来，你需要帮我先出一个需求文档
```

### MCP 调用

通过前面设置的MCP，他会去网页上搜索

期间我发现他使用了mysql，这时候我打算让他换成postgresql，直接esc，然后让他改成postgresql

![image-20250923162617999](https://cdn.wcxian.cc/img/20250928135516185.png)

### 使用验证机制

- PRP 中自带测试任务
- AI 会迭代执行直到全部通过
- 确保首次生成就是可用代码

![image-20250923170609654](https://cdn.wcxian.cc/img/20250928135521386.png)

修改完成后，自动校验

![image-20250923171344204](https://cdn.wcxian.cc/img/20250928135527101.png)

### 修改框架

修改为mybatis-plus

![image-20250923171246164](https://cdn.wcxian.cc/img/20250928135531752.png)



![image-20250923192357628](https://cdn.wcxian.cc/img/20250928135535734.png)

### 新增需求

这边我让他去爬微博热榜

```bash
再帮我实现微博热搜的数据采集，地址是https://s.weibo.com/top/summary?cate=realtimehot
```

![image-20250923192517456](https://cdn.wcxian.cc/img/20250928135541713.png)

但是微博热搜有反爬虫，所以我找了一段开源代码给他参考,这边他黏贴代码就会变成代码块

```bash
微博热搜的采集，你看下这个实现，帮我处理到我们的项目里面[Pasted text #1 +87 lines]
```

后面我让他又爬了csdn和什么值得买

![image-20250926110350669](https://cdn.wcxian.cc/img/20250928135550105.png)

### 生成前端项目

```bash
本项目是一个热榜聚合的网页前端，我要使用Next.js作为我的框架，目前我有知乎热榜，CSDN热榜，微博热搜等等，帮我实现这个前端框架，接口目前使用是这个http://localhost:8080/api/v1/news/latest，用参数区分，你可以参考这个页面来帮我做一个[Image #1]
```

这里使用他的另一个能力，就是可以支持粘贴图片，可以让Claude根据图片来完成任务，上传后的图片不会直接显示出来，而是会用`[Image #id]`的占位符替代。

![image-20250926110614424](https://cdn.wcxian.cc/img/20250928135558314.png)



## 5. 效果展示

<iframe width="100%" height="468" src="https://cdn.wcxian.cc/img/20250928135727930.mp4" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

生成新的文档

![image-20250926190027911](https://cdn.wcxian.cc/img/20250928135753965.png)





Token消耗

![image-20250926183445789](https://cdn.wcxian.cc/img/20250928135806951.png)





## 6.其他使用方法

### 智能权限管理

Claude Code默认很谨慎，不会随便执行危险操作，但你可以精确控制它的权限。
权限级别

只读模式：只能读文件，不能修改编辑模式：可以编辑文件执行模式：可以运行命令完全权限：啥都能干（慎用！）
实用的权限配置

```bash
# 查看当前权限
/permissions

# 允许编辑特定类型文件
/permissions add "Edit(*.js,*.ts,*.jsx,*.tsx)"

# 允许运行特定命令
/permissions add "Bash(npm run *)"
/permissions add "Bash(git *)"

# 禁止危险操作
/permissions deny "Bash(rm -rf *)"
/permissions deny "Bash(curl *)"
```

### 深度思考模式

当遇到复杂问题时，你可以让Claude进入”深度思考模式”，它会像人类专家一样深入分析问题。
三种思考级别

基础思考：think

```
"帮我优化这个函数的性能，think"
```

深度思考：think hard

```
"重构整个用户系统架构，think hard"
```

极限思考：ultrathink

```
"设计一个支持千万级用户的分布式系统，ultrathink"
```

### 上下文压缩

Claude Code 提供了一个 `/compact` 压缩命令：

![img](https://cdn.wcxian.cc/img/20250928135811999.png)

它会清除对话历史记录，但保留上下文中的摘要。

这样做的好处是：

- **减少对话上下文大小**：当对话历史变得很长时，使用 `/compact` 可以压缩对话内容，减少令牌使用量。
- **手动压缩控制**：虽然 Claude Code 默认在上下文超过 95% 容量时自动压缩（可通过 `/config` 开启/关闭自动压缩），但你可以使用 `/compact` 手动触发压缩。

### Git集成 – 版本管理自动化

Claude Code和Git的集成做得特别好，基本实现了版本管理的自动化。

**GitHub App功能**

```
# 安装GitHub集成
/install-github-app
```

安装后Claude可以：

- 自动分析PR差异
- 智能生成提交信息
- 解决代码冲突
- 管理Issues和PR

#### 实用Git操作

**智能提交：**

```
"帮我提交代码，用合适的commit message"
```

Claude会分析你的修改，生成类似这样的提交信息：

```
feat(auth): 新增用户登录状态持久化

- 使用localStorage保存token
- 添加自动登录检查
- 修复登录过期跳转问题
```

**分支管理：**

```
"创建功能分支：用户头像上传"
# 自动执行：git checkout -b feature/user-avatar-upload

"合并到主分支并推送"
# 自动处理合并和推送流程
```

### 消息队列系统 – 批量任务处理

你可以一口气给Claude安排多个任务，它会智能排序并按顺序执行。

**示例：**

```
# 连续输入多个任务
"修复用户注册页面的表单验证bug"
"给商品列表添加分页功能" 
"优化图片加载性能"
"更新API文档"
"跑一遍完整测试"
```

Claude会：

1. 分析任务依赖关系
2. 智能排序执行顺序
3. 自动处理任务切换
4. 需要你确认时会暂停等待

## 7. 优势与局限

### 优势

- 上下文能力强（200K 窗口）
- 端到端交付（从需求到部署）
- 类全栈 AI 工程师角色

### 局限

- 模型依赖度高 → 输出仍需人工验证
- 上下文长任务时可能有延迟
- 对复杂业务逻辑，仍需人工主导

### 适用场景

- 快速原型开发
- 学习新框架
- MVP 验证
- 团队高效协作

------

## 8. 未来展望

- **AI 编程标准化**：Claude Code + RAG 框架，让项目上下文管理更智能。
- **与 DevOps 融合**：未来可能直接接管 CI/CD 流程。
- **AI 工程师新角色**：开发者将更多成为“需求表达者”与“AI 管理者”。

------

## 9. 总结

Claude Code 带来的革新在于：

- **从工具到伙伴** —— 不只是补全，而是全流程开发代理；
- **从片段到系统** —— 交付模块甚至完整系统；
- **从一次性到持续** —— CLAUDE.md 提供项目级记忆，让交互更高效。

它不仅仅是一个编程助手，更是未来开发模式的预演 —— **一个随时待命的全栈 AI 工程师**。



参考资料

[Claude Code：AI编程的深度体验与实践 - 葡萄城技术团队 - 博客园](https://www.cnblogs.com/powertoolsteam/p/19024395)

[Vibe Coding - 五步提升Claude Code开发效率_claude怎么辅助写代码-CSDN博客](https://blog.csdn.net/yangshangwei/article/details/149406359)

[Claude Code使用教程：下载安装完全指南 2025最新版-小木研习社](https://www.xmpick.com/claude-code-download-install-guide/)

[实用指南：Claude Code 的核心能力与架构解析 - yfceshi - 博客园](https://www.cnblogs.com/yfceshi/p/19038024)