---
title: Cursor Gitee-MCP来写一个贪吃蛇游戏
published: 2025-03-27
description: 'Cursor+Gitee-MCP 是一个基于 Gitee 和 MCP 的开源项目，旨在通过 Gitee 和 MCP 来实现代码托管和项目管理。'
image: 'https://cdn.wcxian.cc/img/20250828154958279.png'
tags: [AI, Gitee, MCP]
category: 技术分享
draft: false 
lang: ''
---
# Cursor + Gitee MCP

### MCP（Model Context Protocol）

#### 1. **MCP是什么？**

MCP（模型上下文协议）是一种让AI大语言模型（如ChatGPT、Claude）与外部数据源、工具无缝交互的“万能翻译协议”。它像一座桥梁，帮助AI理解并操作你的本地文件、数据库、日历甚至第三方服务（如GitHub、邮件系统），而无需手动编写代码。

![MCP 架构图](https://cdn.wcxian.cc/img/20250327104106398.png)

总共分为了下面五个部分：

- MCP Hosts: Hosts 是指 LLM 启动连接的应用程序，像 Cursor, Claude Desktop、[Cline](https://github.com/cline/cline) 这样的应用程序。
- MCP Clients: 客户端是用来在 Hosts 应用程序内维护与 Server 之间 1:1 连接。
- MCP Servers: 通过标准化的协议，为 Client 端提供上下文、工具和提示。
- Local Data Sources: 本地的文件、数据库和 API。
- Remote Services: 外部的文件、数据库和 API。

#### 2. **为什么需要MCP？**

**传统AI的局限性**：

- **信息盲区**：AI无法直接访问你的本地文件（如Excel）、私有数据库（如订单记录）或实时数据（如天气）。
- **操作受限**：AI不能直接执行复杂任务（如订机票、发邮件），只能生成文本建议，需要用户手动操作。

**MCP的作用**：

- **打通信息壁垒**：让AI像人类一样“看到”你的文件、日历或数据库。
- **自动执行任务**：AI可通过MCP直接调用工具完成任务，无需用户手动操作。

------

### 通俗例子：用MCP订机票

假设你想让AI助手订一张明天飞上海的机票：

1. **传统方式**：
   - 你需手动打开日历查空闲时间，再去航空公司网站搜航班，最后复制航班信息让AI写邮件确认。
   - **痛点**：步骤繁琐，需多次切换工具。
2. **MCP方式**：
   - **查日历**：AI通过MCP直接读取你的本地日历，确认明天有空。
   - **搜航班**：AI调用航空公司API，筛选出合适的航班。
   - **发确认邮件**：AI通过邮箱接口自动发送确认邮件。
   - **全程自动化**：用户只需说“帮我订明天飞上海的机票”，剩下的由MCP协调完成。

------

### MCP的核心特点

- **万能插头**：一个协议连接所有工具（如GitHub、日历、邮箱），无需为每个工具单独开发接口。
- **动态发现**：AI能自动识别新接入的工具，就像手机自动连接新耳机。
- **安全控制**：所有交互加密，且可设置权限（如禁止AI访问敏感文件）。

### 实际应用场景

1. 编程助手
   - AI通过MCP读取GitHub代码仓库，自动修复Bug或合并分支，而不仅限于编辑当前文件[10]。
2. 数据分析
   - AI直接访问数据库，生成实时销售报表，无需手动导出数据[3][5]。
3. 日程管理
   - AI根据邮件内容自动调整会议时间，并同步到你的日历[7][9]。

### MCP Server、Function Call

- Function Call

![Image](https://cdn.wcxian.cc/img/202503272145961.png)

- MCP Server

![Image](https://cdn.wcxian.cc/img/202503272147748.png)



### **Cursor 和 Gitee MCP 做个贪吃蛇游戏**

#### 环境准备

##### 安装mcp-gitee

- 注册 Gitee 账号：[gitee.com/signup](https://link.juejin.cn/?target=https%3A%2F%2Fgitee.com%2Fsignup)
- 创建私人令牌（需`projects`、`pull_requests`、`issues`、`notes`、`groups`等权限）

![image-20250326105011271](https://cdn.wcxian.cc/img/20250327104120498.png)

- 安装 Gitee MCP Server

项目地址：[mcp-gitee: mcp-gitee 是 Gitee 的模型上下文协议 (MCP) 服务器实现。它提供了一组与 Gitee 的 API 交互的工具，允许 AI 助手管理仓库、Issue、Pull Request等。](https://gitee.com/oschina/mcp-gitee)

因为这个是go写的，所以我们这里还需要安装一下golang的环境

[The Go Programming Language](https://go.dev/)

我这边是windows，直接安装完以后，他会自己添加到PATH

![image-20250326105415160](https://cdn.wcxian.cc/img/20250327104125964.png)

用go安装mcp-gitee

```shell
# golang >= 1.23 
go install gitee.com/oschina/mcp-gitee@latest
```

```shell
# 查询一下是否安装成功
mcp-gitee --version
Gitee MCP Server
Version: 0.1.6
```

##### 安装cursor

- 安装 Cursor ：[www.cursor.com/](https://link.juejin.cn/?target=https%3A%2F%2Fwww.cursor.com%2F)
- 配置 Cursor MCP Server

打开设置 `Settings` - `MCP` - `Add new global MCP server`

```json
{
  "mcpServers": {
    "gitee": {
      "command": "mcp-gitee",
      "env": {
        "GITEE_API_BASE": "https://gitee.com/api/v5",
        "GITEE_ACCESS_TOKEN": "<your personal access token>"
      }
    }
  }
}
```



到这里可能会用不上，具体不知道什么原因，好像是cursor的bug

![image](https://cdn.wcxian.cc/img/20250327104130246.png)



解决方法就是升级cursor，升级go，配置环境变量，重启之类的，反正最后突然就好了

[MCP Servers No tools found - Bug Reports - Cursor - Community Forum](https://forum.cursor.com/t/mcp-servers-no-tools-found/49094)

看到刚才添加的 GiteeMCP 前有绿色圆点即代表开启成功

![image-20250326110408594](https://cdn.wcxian.cc/img/20250327104133160.png)

测试一下，在 Cursor Chat 里输入「Gitee 上有什么通知」，如果 MCP tool 正确调用，说明配置完成。

![image-20250326113746385](https://cdn.wcxian.cc/img/20250327104136001.png)

按一下run tool

![image-20250326113821995](https://cdn.wcxian.cc/img/20250327104138408.png)

查询到信息

#### 做一个贪吃蛇游戏

- 创建 Gitee 仓库

![image-20250326114755199](https://cdn.wcxian.cc/img/20250327104143877.png)

- 批量创建 Issue

![image-20250326115047689](https://cdn.wcxian.cc/img/20250327104147959.png)

- 实现issues

![image-20250326120536575](https://cdn.wcxian.cc/img/20250327104150977.png)

- 提交代码

![image-20250326134746773](https://cdn.wcxian.cc/img/20250327104154202.png)

- 解决bug和feature

![image-20250326135922035](https://cdn.wcxian.cc/img/20250327104157140.png)

- 运行游戏

![image-20250326134647050](https://cdn.wcxian.cc/img/20250327104159758.png)

### MCP在智能零售中的应用场景

#### 1. **智能客服与实时问答**

MCP可连接零售企业的订单系统、库存数据库和产品知识库，让AI客服助手实时获取最新数据。例如：

- 客户询问“我的订单什么时候送达？”时，AI通过MCP直接查询物流系统，返回实时物流信息[1][3]。
- 用户咨询“XX商品是否有促销活动？”，AI调用促销数据库MCP服务器，结合当前活动规则生成准确回答[1][10]。

#### 2. **个性化推荐系统**

通过MCP整合客户行为数据、历史购买记录和实时库存信息：

- AI分析用户在APP内的浏览路径，通过MCP调取库存量、同类商品评价等数据，生成“根据您的喜好，推荐搭配A商品的B配件”等精准推荐[3][10]。
- 结合天气数据（通过MCP接入气象API），推荐应季商品（如雨天推送雨伞促销）[4][8]。

#### 3. **库存管理与智能补货**

MCP打通销售系统、供应链数据和仓储IoT设备：

- AI实时监控货架传感器数据（通过MCP连接IoT平台），当某商品库存低于阈值时，自动生成补货工单并发送给仓库管理系统[2][9]。
- 结合节假日销售趋势（通过MCP访问历史销售数据库），预测未来需求量并调整采购计划[1][6]。

#### 4. **动态定价与促销优化**

通过MCP接入竞争对手价格监测数据、成本波动信息：

- AI每小时抓取竞品价格（调用价格爬虫MCP服务），自动调整本店商品定价以保持竞争力[4][7]。
- 根据滞销商品库存情况（通过MCP读取仓储数据），生成“买二送一”等智能促销方案[3][10]。

#### 5. **线下门店智能导购**

MCP连接店内AR设备、电子价签和会员系统：

- 顾客扫描商品二维码时，AI通过MCP调取商品详情视频、用户评价，并在AR眼镜上展示3D使用演示[2][8]。
- 导购机器人通过MCP查询会员等级和积分，主动提示“您的积分可兑换XX礼品”[1][6]。

### MCP Servers

[punkpeye/awesome-mcp-servers: A collection of MCP servers.](https://github.com/punkpeye/awesome-mcp-servers)

[MCP Servers](https://mcp.so/)

[cline/mcp-marketplace: This is the official repository for submitting MCP servers to be included in Cline's MCP Marketplace. If you’ve built an MCP server and want it to be discoverable and easily installable by millions of developers using Cline, submit your server here.](https://github.com/cline/mcp-marketplace)

![image-20250327104040147](https://cdn.wcxian.cc/img/20250327104203563.png)

### 参考资料：

> [不再混淆了！一文揭秘MCP Server、Function Call与Agent的核心区别_人工智能_咔咔学姐kk-AI Agent技术社区](https://agent.csdn.net/67d8ccec1056564ee2463822.html)

> [一文看懂：MCP(大模型上下文协议) - 知乎](https://zhuanlan.zhihu.com/p/27327515233)

> [一行代码不用写，用 Cursor + Gitee MCP 做个贪吃蛇游戏Gitee 正式发布官方 MCP Server - 掘金](https://juejin.cn/post/7483158015192072226)

> [对话即服务：Spring Boot整合MCP让你的CRUD系统秒变AI助手-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2506434)