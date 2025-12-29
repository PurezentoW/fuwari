---
title: n8n
published: 2025-12-29
description: 'n8n 工作流教程'
image: 'https://cdn.wcxian.cc/img/20251229150438223.png'
tags: [n8n,AI]
category: '技术分享'
draft: false 
lang: ''
---
# n8n

---

## 项目概述

### 什么是 n8n

**n8n**（发音为"n-eight-n"）是一个开源的**工作流自动化工具**，可以通过可视化的拖拽界面，将不同的应用和服务连接起来，实现自动化任务。它的设计理念是"工作流即代码"，让非程序员也能轻松构建复杂的自动化流程。

[n8n-io/n8n: Fair-code workflow automation platform with native AI capabilities. Combine visual building with custom code, self-host or cloud, 400+ integrations.](https://github.com/n8n-io/n8n)

![image-20251224170531334](https://cdn.wcxian.cc/img/20251224183127206.png)

![image-20251224183516343](https://cdn.wcxian.cc/img/20251224183519287.png)

### n8n 的核心特性

✅ **开源免费**: MIT 许可证，支持私有化部署
✅ **可视化设计**: 直观的拖拽式界面，无需编程基础
✅ **1000+ 集成**: 支持超过 1000 种第三方服务集成
✅ **灵活扩展**: 支持自定义节点和代码执行
✅ **AI 友好**: 内置 LangChain 节点，深度集成大模型能力
✅ **部署简单**: 支持 Docker 一键部署

---

## 为什么选择 n8n

### 与主流平台对比

| 平台        | 部署方式  | 第三方集成 | AI 支持 | 价格      | 开源 |
| ----------- | --------- | ---------- | ------- | --------- | ---- |
| **n8n**     | 私有化/云 | 1000+      | 原生    | 免费/付费 | ✅    |
| **Zapier**  | 云服务    | 7000+      | 一般    | $144/月起 | ❌    |
| **Coze**    | 云服务    | 375+       | 强      | 套餐制    | ❌    |
| **Dify.ai** | 私有化/云 | 40+        | 强      | 免费/付费 | ✅    |

### n8n 的独特优势

#### 1. 部署灵活性

- **私有化部署**: 数据完全自主可控
- **Docker 支持**: 一行命令即可启动
- **跨平台**: 支持 Windows、macOS、Linux

#### 2. 强大的扩展能力

```javascript
// 支持在 Code 节点中直接编写 JavaScript 或 Python
const items = $input.all();
return items.map(item => {
  return {
    json: {
      // 自定义处理逻辑
    }
  };
});
```

#### 3. AI 深度集成

- **LangChain 可视化**: 无需编写代码即可构建 AI Agent
- **多模型支持**: OpenAI、Claude、Gemini、本地模型等
- **灵活的提示词管理**: 支持动态变量和模板

![image-20251224183526025](https://cdn.wcxian.cc/img/20251224183527798.png)

### n8n 适合做什么？

n8n 适合一切关于自动化的事情，包括在自动化中使用 AI。

这里有一些官方的例子，比如：

![image-20251224183536482](https://cdn.wcxian.cc/img/20251224183538703.png) ![image-20251224183558949](https://cdn.wcxian.cc/img/20251224183601618.png)



<img src="https://cdn.wcxian.cc/img/20251224183614547.png" alt="image-20251224183611204" style="zoom:62%;" /> ![image-20251224183633611](https://cdn.wcxian.cc/img/20251224183635443.png)



---

## n8n 架构设计

### 整体架构

![生成 n8n 架构图](https://cdn.wcxian.cc/img/20251224183647376.png)

### 核心概念

#### 1. Workflow（工作流）

工作流是由多个节点组成的自动化流程，定义了数据如何在各个节点之间流转和处理。

#### 2. Node（节点）

节点是工作流的基本组成单元，每个节点代表一个具体的操作或功能。

#### 3. Connection（连接）

连接定义了节点之间的数据流转关系，包括数据映射和传递规则。

---



## 快速开始

n8n 分为两个版本：

1. [云服务版](https://n8n.partnerlinks.io/5ke1e3gdv6jj)：官方提供的云服务版本，这个是开箱即用，注册账号购买会员即可开始使用，这里不做说明

2. [自托管版](https://github.com/n8n-io/n8n)：官方在 Github 开源的版本

### 安装部署

#### Docker 部署（推荐）

```bash
# 拉取镜像
docker pull n8nio/n8n

# 启动容器，因为我这边是在虚拟机里部署的，所以N8N_DIAGNOSTICS_ENABLED和WEBHOOK_URL需要配置一下
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -e N8N_DIAGNOSTICS_ENABLED=false \
  -e WEBHOOK_URL=http://192.168.159.131:5678/ \
  -v /home/purezento/data:/home/node/.n8n-files \
  n8nio/n8n
```

| 环境变量         | 值                     | 备注                                                         |
| ---------------- | ---------------------- | ------------------------------------------------------------ |
| N8N_Port         | 5678                   | Docker端口                                                   |
| N8N_PROTOCOL     | https                  | 开启 https 访问                                              |
| N8N_HOST         | 你的域名               | ㅤ                                                           |
| N8N_WEBHOOK      | https://你的域名:5680/ | n8n 用于接 Web hook 的地址，和你从外网访问 n8n 的地址是一个，要带协议头。 |
| GENERIC_TIMEZONE | Asia/Shanghai          | 设置 n8n 的默认时区                                          |

### 首次访问

1. 打开浏览器访问 `http://localhost:5678`
2. 创建管理员账号
3. 进入工作流编辑界面

![image-20251224183705932](https://cdn.wcxian.cc/img/20251224183732696.png)

这边会给你的邮箱发送一个密钥，可以解锁部分付费功能



---

## 核心功能详解

![image-20251224183935334](https://cdn.wcxian.cc/img/20251224183939799.png)

### 1. 触发器（Triggers）

触发器是工作流的起点，决定了工作流何时执行。

#### 常见触发器类型

| 触发器类型   | 使用场景                    | 配置参数            |
| ------------ | --------------------------- | ------------------- |
| **Manual**   | 手动执行工作流              | 无                  |
| **Schedule** | 定时执行（Cron 表达式）     | Cron 表达式         |
| **Webhook**  | HTTP 请求触发               | URL 路径、HTTP 方法 |
| **Chat**     | 聊天消息触发（AI 节点专用） | 聊天平台配置        |

#### 示例：定时触发器

```yaml
# 每天早上 8 点执行
cron: "0 8 * * *"

# 每小时执行一次
cron: "0 * * * *"

# 每 30 分钟执行一次
cron: "*/30 * * * *"
```

### 2. 数据处理

#### 数据读取节点

| 节点名称          | 功能                 | 使用场景             |
| ----------------- | -------------------- | -------------------- |
| **HTTP Request**  | 发送 HTTP 请求       | 调用 REST API        |
| **Read Binary**   | 读取二进制文件       | 处理图片、PDF 等文件 |
| **Google Sheets** | 读取 Google 表格数据 | 数据导入导出         |
| **MySQL**         | 执行 SQL 查询        | 数据库操作           |

#### 数据转换节点

| 节点名称      | 功能               | 使用场景     |
| ------------- | ------------------ | ------------ |
| **Code**      | 执行自定义代码     | 复杂数据处理 |
| **Set**       | 设置和修改字段值   | 数据格式化   |
| **IF**        | 条件判断           | 分支逻辑     |
| **Merge**     | 合并多个数据流     | 数据聚合     |
| **Split Out** | 拆分数组为多行数据 | 批量数据处理 |

### 3. AI 集成

#### LangChain 节点架构

```
┌──────────────────────────────────────────────────────┐
│                   AI Agent 节点                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐    │  │
│  │  │  Model   │  │  Memory  │  │   Tool   │    │  │
│  │  │  (模型)   │  │  (记忆)   │  │  (工具)   │    │  │
│  │  └──────────┘  └──────────┘  └──────────┘    │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

#### 支持的 AI 模型

- **OpenAI**: GPT-4、GPT-3.5 等
- **Anthropic**: Claude 3 系列
- **Google**: Gemini Pro
- DeepSeek
- **本地模型**: Ollama、LM Studio

---

## 实战案例

### 案例 1：网页数据爬取

#### 需求描述

当我们打开网页版的即刻(比如：@[评论尸](https://okjk.co/oenysg)），进入某一个用户的主页，可以查看他最新的 10 条动态。

#### 第一步：如何触发程序运行

所有的程序运行都需要一个条件触发，当你打开一个画布，选择新增第一个节点时，系统会询问你希望通过什么方式来触发这个程序的运行。

![img](https://cdn.wcxian.cc/img/20251224184133342)

这些选项的含义分别是：

- 在 n8n 上面点击一下运行按钮，触发程序运行；

- 外界 API 发来的命令，比如 Telegram 收到一个消息；

- 每天/每小时固定一个时间执行;

- 通过 HTTP 请求触发任务；

- 由另一个 Workflow 触发这个任务；

- 由聊天框消息触发这个程序运行（AI 对话类节点专用）；

- 文件变更/Email/等等；

因为这是一个示例节点，所以我们只需要选择第一个选项即可。

所以我们的第一个节点如下图所示：

![image-20251224185244846](https://cdn.wcxian.cc/img/20251224185246874.png)

#### 第二步：如何读取网页的内容？

**Step 1: HTTP Request 节点**

```yaml
方法: GET
URL: https://m.okjike.com/users/xxx
```

![image-20251224184518805](https://cdn.wcxian.cc/img/20251224184534533.png)

**Step 2: Extract HTML Content 节点**

```javascript
// 提取规则
Extraction Values:
  - Key: 正文
    CSS Selector: div.jsx-3930310120.wrap
    Return Array: true

  - Key: 日期
    CSS Selector: div.subtitle
    Return Array: true
```

![image-20251224184602477](https://cdn.wcxian.cc/img/20251224184606663.png)

**Step 3: Split Out 节点**

```javascript
// 将数组拆分为多行数据
Field to Split Out: 正文, 日期
```

![PixPin_2025-12-24_18-48-36](https://cdn.wcxian.cc/img/20251224184945485.png)

**Step 4: Export to Excel 节点**

```javascript
// 导出字段
Fields to Export: date, content
File Name: jike_dynamic.xlsx
```

![image-20251224185007998](https://cdn.wcxian.cc/img/20251224185010868.png)



![img](https://cdn.wcxian.cc/img/20251224185029967)

#### 工作流

![img](https://cdn.wcxian.cc/img/20251224185050374)

---

### 案例 2：AI 工单分类

#### 需求描述

使用大模型自动分析工单内容，判断是否属于预定义的问题之一。

这边我导了一份我们的工单excel给他，分析一下工单内容，然后判断一下是什么问题

![img](https://cdn.wcxian.cc/img/20251224185111895)

核心是这个循环

![img](https://cdn.wcxian.cc/img/20251224185133236)

#### 提示词设计

```text
我需要你帮忙做一个工单的分类，这里有非常多的工单。
我需要你根据工单里面的客服和客户的聊天内容。帮忙判断我给你的工单属于哪类问题，问题种类有：设备问题、数据问题、需求问题、资金问题、其他
只需要输出问题的分类和工单编号即可，例如“【数据问题】、【设备问题】”
请不要输出除此以外的任何内容，也不需要做任何的解释性说明。
面这个是你需要分类的工单
工单编号： {{ $json['工单编号'] }}
工单内容： {{ $json['问题描述'] }}
```

#### 配置要点

1. **循环处理**: n8n 会自动遍历表格中的每一行数据
2. **参数传递**: 使用 `{{变量名}}` 语法引用上游节点的数据
3. **结果回写**: 使用唯一标识符（如工单编号）关联更新结果

![image-20251224181542347](https://cdn.wcxian.cc/img/20251224185142185.png)

### 案例 3：多系统协同工作流（MCP 协议）

#### 需求描述

在实际业务中，我们经常需要协调多个外部系统完成复杂的业务流程。例如：

**业务场景**：门店每天需要从多个数据源同步数据，并进行智能分析和决策：

1. **Google Sheets**：供应商提供的商品价格表（每日更新）
2. **MySQL 数据库**：门店的销售流水和库存数据
3. **外部 HTTP API**：天气数据、物流信息等
4. **AI 分析**：根据多源数据生成每日运营报告


---

## 高级技巧

### 1. 隐式参数传递

n8n 的一个重要设计理念是**隐式参数传递**，这意味着：

- **跨节点引用**: 可以引用任意前序节点的变量
- **自动关联**: 系统自动维护数据的唯一关联关系
- **简化开发**: 无需手动传递中间结果

```javascript
// 示例：在第 5 个节点引用第 1 个节点的数据
{{ $node["HTTP Request"].json["id"] }}
```

### 2. 错误处理

#### 错误处理策略

```javascript
// 在节点中启用错误输出
Continue On Fail: true

// 使用 IF 节点判断错误
{{ $json.error }}
```

#### 重试机制

```yaml
# HTTP Request 节点重试配置
Retry On Fail: true
Max Tries: 3
Wait Between Tries: 1000ms
```

### 3. 性能优化

#### 批量处理

```javascript
// 使用 Split In/Out 节点优化批量操作
Split In:
  Batch Size: 100
  Options:
    Reset: false
```

#### 并发控制

```yaml
# 执行策略配置
Execution Order: Parallel (并行执行)
Max Concurrency: 10 (最大并发数)
```

---

## 最佳实践

### 1. 工作流设计原则

#### KISS 原则（Keep It Simple, Stupid）

- ❌ 避免过度复杂的工作流
- ✅ 拆分为多个小工作流
- ✅ 使用子工作流复用逻辑

#### DRY 原则（Don't Repeat Yourself）

```javascript
// ✅ 推荐：使用变量统一管理
{{ $vars.apiEndpoint }}/users

// ❌ 避免：硬编码重复
https://api.example.com/v1/users
https://api.example.com/v1/products
```

### 2. 安全建议

#### 凭证管理

- ✅ 使用 Credentials 统一管理敏感信息
- ❌ 避免在工作流中硬编码 API Key
- ✅ 定期轮换密钥

#### 权限控制

```yaml
# 最小权限原则
API Scopes:
  - read:user       # 仅读取用户信息
  - write:orders    # 仅写入订单数据
```

### 3. 监控与告警

#### 执行日志

```bash
# 查看执行日志
Settings → Executions → 选择执行记录 → 查看 Logs
```

![img](https://cdn.wcxian.cc/img/20251224185147091)

#### 性能监控

```javascript
// 获取执行统计
Workflow Settings → Save Execution Progress: Enabled
```

---

### 具体联动场景

#### 场景 1：多渠道消息通知与告警（Java 触发 n8n）

- 流程：
    1. Java 后端检测到异常（如：某个货架传感器失联、有人在店内存留超过1小时）。
    2. Java 发送 Webhook 到 n8n，携带设备 ID 和异常类型。
    3. n8n 分发逻辑：
        - 如果是“设备故障”，发 Slack 给技术团队。
        - 如果是“疑似偷盗/安全隐患”，发 Telegram 给安保，并触发语音播报 API。
        - 如果是“冷柜温度异常”，给店主拨打告警电话（Twilio 插件）。

#### 场景 2：智能营销与会员触达（n8n + Java API）

- 流程：
    1. n8n 每天凌晨定时启动，查询 Java 后端的数据库，筛选出“过去 7 天进店超过 3 次但昨天没来”的会员。
    2. n8n 调用外部优惠券平台 API 生成专属折扣码。
    3. n8n 回填折扣码到 Java 后端的优惠券表。
    4. n8n 通过微信公众号或短信平台发送个性化文案给用户。

#### 场景 3：供应链与自动补货（n8n 协调 Java 与 外部系统）

- 流程：
    1. Java 后端检测到某款饮料库存低于 20 瓶。
    2. Java 触发 n8n，n8n 首先检查 Google Sheets 或 ERP 里的采购合同。
    3. n8n 自动向供应商发送采购邮件，附带 Java 系统生成的订单 PDF。
    4. 供应商回复“已发货”邮件后，n8n 提取邮件中的物流单号，通过 Java API 更新门店系统的“预计到货状态”。

#### 场景 4：门店环境的“策略中心”（n8n 决策，Java 执行）

- 流程：
    1. n8n 定时获取天气 API（如：今天最高气温 35℃）。
    2. n8n 根据策略判断：如果温度 > 30℃，调用 Java 后端的 IoT 控制接口，提前将店内空调调低 2 度。
    3. n8n 结合“店内当前人数”（由 Java 后端提供数据），动态调整店内背景音乐的音量或灯光亮度。

#### 场景 5：复杂的对账与财务报表

- 流程：
    1. Java 后端只负责记录原始流水。
    2. n8n 每月 1 号提取 Java 数据库中的流水、支付宝/微信支付导出的对账单。
    3. n8n 在工作流中进行比对，标记差异项。
    4. n8n 将最终生成的 Excel 表格上传到企业网盘，并生成一张财务总结图片发到老板微信。

---

## 舆情分析系统

![image-20251229143310408](https://cdn.wcxian.cc/img/20251229143317547.png)





![image-20251229143823151](https://cdn.wcxian.cc/img/20251229145146715.png)

![PixPin_2025-12-29_14-45-24](https://cdn.wcxian.cc/img/20251229145158927.png)

[final_report_研究近3年的中国无人值守商店经营情况_20251217_081307.html](D:\Downloads\final_report_研究近3年的中国无人值守商店经营情况_20251217_081307.html)

![image-20251229145644906](https://cdn.wcxian.cc/img/20251229145648147.png)

## 参考资料

- [n8n 官方文档](https://docs.n8n.io/)
- [n8n GitHub 仓库](https://github.com/n8n-io/n8n)
- [n8n 社区论坛](https://community.n8n.io/)
- [(60 封私信 / 80 条消息) n8n本地Excel操作指南 - 知乎](https://zhuanlan.zhihu.com/p/81931281916)https://m.okjike.com/originalPosts/6944fbc9e48f1bd7abc4c1c7)
- [制作第一个与大模型配合的 n8n 程序 | 简单易懂的现代魔法 - n8n 中文使用教程](https://n8n.akashio.com/article/the-first-n8n-program)

---

## 总结

n8n 作为一款强大的工作流自动化工具，具有以下核心价值：

1. **降低技术门槛**: 可视化界面让非程序员也能构建自动化流程
2. **灵活的扩展性**: 支持 1000+ 集成和自定义代码
3. **AI 友好**: 深度集成 LangChain，助力 AI 应用开发
4. **开源可控**: 支持私有化部署，数据安全有保障

通过 n8n，我们可以将繁琐的手工操作转化为自动化的工作流，极大地提升工作效率。在 AI 时代，n8n 更是成为了连接大模型与各种服务的桥梁，让每个人都能轻松打造属于自己的"自动化魔法"。