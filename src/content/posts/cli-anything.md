---
title: cli-anything
published: 2026-04-24
description: cli-anything 适配禅道实现cli化
image: 'https://cdn.wcxian.cc/img/20260424190535379.png'
tags: [CLI,ZenTao,CLI-Anything]
category: '技术分享'
draft: false 
lang: zh-CN
---
# CLI Anything 把任意软件变成 AI 可调用工具

CLI 这个计算机世界里最古老的交互方式，正在迎来一轮新的爆发。以前大家提到 CLI，第一反应往往是程序员、脚本、终端、批处理；但在 Agent 时代，CLI 的角色已经不一样了。它不再只是给人类开发者准备的工具，而开始变成一种非常适合 AI 使用的软件接口。

最近这段时间，越来越多的大厂和开源项目都开始重新加码 CLI。原因并不复杂。GUI 是给人设计的，强调视觉、拖拽、按钮和菜单；而 CLI 天然就是文本输入、结构化参数、明确输出、可组合调用。对大模型来说，后者往往比前者更容易理解，也更容易稳定执行。

![image.png](https://cdn.wcxian.cc/img/20260424181621256.png)


也正因为这样，最近出现了一类非常有意思的项目，不是去重新发明一个新的协议，而是**把原本没有命令行界面的软件，重新组织成 CLI**。`CLI Anything` 就是这里面讨论度非常高的代表。

这篇文章主要讲三件事：

- 为什么 CLI 在 Agent 时代更好用
- `CLI Anything` 是怎么把任意软件 CLI 化的
- 怎么用 `CLI Anything` 生成你想要的软件 CLI，并通过 `cli-anything-zentao` 这个真实案例看它到底能落地到什么程度

## 一句话总结

在 Agent 时代，CLI 的价值不只是“可以在终端里操作软件”，而是**它天然更适合作为 AI 的操作接口**。而 `CLI Anything` 做的事情，本质上是把原本依赖 GUI 的软件能力，重构成一套 Agent 更容易理解、调用和编排的命令系统。

## 为什么 CLI 在 Agent 时代更好用

CLI 全称是 Command Line Interface ，其实就是命令行界面。CLI 跟我们平时熟悉的图形界面(GUI)，代表了两种不同的交互逻辑。 如果我想把这个视频的前 5 秒切分出来，使用 GUI 的方式是先导入剪辑软件，找到对应的时间点切分，最后再把视频导出出来。使用 CLI 的方式则是打开命令行窗口，执行这一个命令就搞定了。

很多人第一次接触 Agent，会默认觉得 GUI 才是更先进、更自然的交互方式，因为人类平时用的就是 GUI。但如果把使用者换成 AI，这个判断往往就要反过来。

### 1. CLI 更接近 AI 的“母语”

大模型在训练时天然接触过大量代码、脚本、命令行、配置文件和终端输出。所以对 AI 来说，像下面这种输入是非常自然的：

```bash
ffmpeg -i input.mp4 -t 00:00:5 -c copy part1.mp4
```

这条命令背后表达的是一组清晰的意图：输入文件、截取时长、输出文件。它没有按钮，没有弹窗，也没有隐藏状态。Agent 不需要猜用户应该点哪个菜单，只需要理解参数含义。

而 GUI 恰恰相反。它的很多信息隐藏在页面布局、视觉层级和交互状态里，对人类很友好，对 AI 却不一定友好。

### 2. CLI 天然自解释，Agent 可以边用边学

CLI 有一个非常重要的优势，就是几乎所有设计良好的命令行工具都支持 `--help`。这意味着 Agent 不需要在一开始就把所有工具定义、所有参数格式、所有调用示例一次性学完。它可以在需要时再查。

比如面对一个陌生 CLI，Agent 完全可以这样逐步学习：

```bash
gh --help
gh issue --help
gh issue list --help
```

这种学习方式和 Skills 的渐进式披露思路其实是一样的。好处很明显：

- 不需要把全部工具 schema 一次性注入上下文
- Token 开销更低
- 学习路径更自然
- 参数调用更不容易出错

这也是为什么很多 Agent 在使用 CLI 时，稳定性往往比直接操作复杂 GUI 更高。

### 3. CLI 同时对人和 AI 友好

这一点在真实工程里非常关键。

如果 Agent 调用一个工具失败了，而这个工具刚好是 CLI，那么人类开发者可以直接把同一条命令复制到终端里复现问题、看报错、改参数、继续排查。整个人机协作过程会非常顺畅。

但如果整个操作只存在于 Agent 内部的黑盒调用里，人类往往很难快速复盘到底发生了什么。

CLI 在这里的价值，不只是可执行，而是**可复现、可观察、可排错**。

### 4. CLI 天然适合组合和自动化

CLI 的另一个核心优势，是它天然适合进入脚本和流水线。

比如同样是查询某个系统里的数据，如果这个系统提供 CLI，那么后续你就可以：

- 直接输出 JSON
- 用 `jq` 或 PowerShell 做二次筛选
- 通过管道把多个命令拼成流水线
- 接入定时任务、CI、AI 工作流

这类能力不是额外“补上去”的，而是 CLI 的原生优势。

### 5. CLI 和 GUI、MCP 各自适合的场景并不一样

从工程角度看，这三种能力暴露方式并不是非此即彼。

| 方式 | 更适合谁     | 优势                             | 局限                           |
| ---- | ------------ | -------------------------------- | ------------------------------ |
| GUI  | 人类用户     | 直观、低门槛、适合视觉化操作     | 不利于自动化，Agent 调用成本高 |
| CLI  | 人类 + Agent | 自解释、可复现、可组合、易脚本化 | 需要好的命令抽象               |
| MCP  | Agent        | 协议化、标准化、便于平台托管     | 对人类不够直接，调试链路更重   |

所以不是 CLI 一定取代 GUI 或 MCP，而是到了 Agent 时代，CLI 的价值第一次被大规模重新看见了。

![image (2)](https://cdn.wcxian.cc/img/20260424161430275.png)

## CLI Anything 是什么

![image-20260424190511584](https://cdn.wcxian.cc/img/20260424190517329.png)

理解了为什么 CLI 重要，再来看 `CLI Anything` 就很自然了。

`CLI Anything` 的目标很直接：**让原本只有 GUI 的软件，也能被转换成 Agent 可调用的 CLI 工具。**

它不是简单录制点击脚本，也不是让浏览器去机械模仿人类操作。它真正做的事情，更接近下面这条链路：

1. 分析目标项目源码和现有能力边界。
2. 找出哪些操作适合抽象成命令。
3. 规划命令树、参数和输出结构。
4. 生成 CLI 代码、帮助文档和测试。
5. 让这个软件最终既能被人类手动调用，也能被 Agent 自动调用。

换句话说，`CLI Anything` 的重点不是“终端化界面”，而是**重新组织软件能力的暴露方式**。

## 为什么这种思路很重要

很多企业软件不是没有能力，而是它们的能力都被封装在页面里。

比如一个系统明明支持：

- 创建记录
- 查询列表
- 编辑状态
- 指派负责人
- 导出数据

但这些操作都散落在不同页面、弹窗和菜单里。对人类来说这很正常，因为 GUI 更直观；但对 Agent 来说，这种交互链路会很长，很脆弱，也很难复用。

如果把这些动作重新抽象成 CLI，整个交互模式就变了。原本要打开页面、切换模块、点击按钮、填写表单的过程，最后可能只需要一条命令：

```bash
some-cli task create --project 7 --name "修复登录问题"
```

从这个角度看，`CLI Anything` 更像是在做一件很底层、但很有价值的事情：**把软件从“只能被人点”变成“也能被 AI 调”。**

## 怎么用 CLI Anything 生成你想要的软件 CLI

[HKUDS/CLI-Anything: "让所有软件都能被 Agent 驱动"](https://github.com/HKUDS/CLI-Anything)

![image.png|697](https://cdn.wcxian.cc/img/20260424181858024.png)


### 1. 准备环境

根据现有使用方式，前提通常有两个：

- 电脑上安装好 Python
- 选择一个支持插件和 Agent 工作流的 AI 编程工具，比如 Claude Code

在 Claude Code 里，可以先安装 `CLI Anything` 插件：

```bash
/plugin marketplace add HKUDS/CLI-Anything
/plugin install cli-anything
```

安装完成后，重启对应的 Agent 工具。

![PixPin_2026-04-16_10-39-45](https://cdn.wcxian.cc/img/20260424150942002.jpg)

### 2. 准备目标软件源码

`CLI Anything` 的输入不是一个随便的网址，而通常是一个目标软件的源码目录。所以你需要先把想要 CLI 化的项目源码准备到本地。

例如：

```bash
git clone <目标项目仓库>
cd <目标项目目录>
```

更适合拿来做 CLI 化的项目，一般有这些特征：

- 领域对象清晰
- 业务动作稳定
- 有明确输入输出
- 不强依赖复杂拖拽或纯视觉化交互

像项目管理系统、内容管理系统、数据查询工具、配置管理后台，这类都比复杂设计软件更适合先做 CLI 化。

[禅道 12.5.3 版本发布，主要重构年度汇总功能 - 禅道下载 - 禅道项目管理软件](https://www.zentao.net/download/zentaopms12.5.3-80319.html#2)

我这边使用了我们在用的12.5.3版本，实际最新的已经到22了，支持MCP和RESTful

![image-20260424151633780](https://cdn.wcxian.cc/img/20260424151635084.png)

### 3. 执行 CLI Anything 的核心命令

准备好源码之后，就可以执行最关键的一步：

```bash
/cli-anything:cli-anything ./你的项目目录
```

后面跟的就是你要 CLI 化的软件源码路径。

在这个阶段，Agent 会开始执行一整套相对复杂的流程，包括源码分析、命令设计、实现、测试和文档生成。你可以把它理解成“让 Agent 帮你给一个项目补出 CLI 层”。

![解析禅道源码](https://cdn.wcxian.cc/img/20260424161441311.jpg)

### 4. 等待生成结果并检查交付物

![PixPin_2026-04-16_11-24-59](https://cdn.wcxian.cc/img/20260424161444940.jpg)

生成完成后，通常不会只有一个可执行文件，而是一整套交付内容。以常见产物来看，一般会包括：

- `README.md`：说明生成出来的 CLI 怎么使用
- `HARNESS.md`、`TEST.md`、`SOP.md` 一类文件：约束 Agent 如何编写和验证代码
- `cli_anything/` 目录：真正的 CLI 代码实现

这里可以顺便多解释一句 `HARNESS.md`，因为很多人第一次看到这个文件名时，其实并不知道它的价值在哪里。

如果按字面理解，harness 可以把它看成一个“执行框架”或者“操作护栏”。它的作用通常不是写业务代码本身，而是告诉 Agent：这个项目应该怎么进入、命令应该怎么跑、任务应该按什么约束执行、产出要满足什么边界。

换句话说，`HARNESS.md` 更像是给 AI 准备的工程运行说明书。它会把一些原本只能靠人类工程师口头补充的上下文，提前固化成文档，比如：

- 先从哪些命令开始了解项目
- 修改代码前后要做哪些检查
- 哪些目录或能力是核心入口
- 遇到失败时应该怎样排查和回退

这件事非常重要。因为 Agent 最大的问题之一，从来不是“不会写代码”，而是容易在陌生项目里走偏：不知道入口，不知道边界，不知道验证方式，最后写出来的东西看似能跑，实际却不符合项目约束。

`HARNESS.md` 的价值就在这里。它不是面向最终用户的功能文档，而是面向 Agent 的执行上下文。你可以把它理解成：CLI Anything 不只是帮你生成一个 CLI，还顺手帮你把“AI 以后应该如何继续维护和使用这个 CLI”这套规则也沉淀了下来。

![image (1)](https://cdn.wcxian.cc/img/20260424161450554.png)

也就是说，`CLI Anything` 交付的不是“一个命令”，而是“一个可持续维护的 CLI 工程骨架”。

### 5. 安装生成出来的 CLI

生成完成后，通常还需要在项目目录里把这个 Python 包安装起来：

```bash
pip install -e .
```

安装完成以后，你就可以像使用正常 CLI 一样去执行它。

### 6. 先用 --help 学习，再让 Agent 调用

这是最关键的一步，也是我觉得最符合 Agent 工作方式的一步。

不要一上来就让 Agent 盲调复杂命令。更好的做法是先让它执行：

```bash
your-cli --help
your-cli subcommand --help

PS D:\My Project\zentaopms-zentaopms_12.5.3_20210108> cli-anything-zentao --help
Usage: cli-anything-zentao [OPTIONS] COMMAND [ARGS]...

  cli-anything-zentao: CLI harness for ZentaoPMS.

Options:
  --json         Output as JSON
  --server TEXT  ZentaoPMS server URL (e.g., http://pms.example.com)
  --help         Show this message and exit.

Commands:
  action     Action history and activity log commands.
  auth       Authentication commands.
  branch     Branch management commands.
  bug        Bug management commands.
  build      Build management commands.
  context    Set current context (product, project).
  dept       Department management commands.
  doc        Document management commands.
  estimate   Work estimate (effort) tracking commands.
  export     Export data to JSON or CSV.
  module     Module management commands.
  my         My dashboard - tasks, bugs, stories assigned to me.
  plan       Product plan management commands.
  product    Product management commands.
  project    Project management commands.
  release    Release management commands.
  report     Report and statistics commands.
  story      Story/requirement management commands.
  task       Task management commands.
  testcase   Test case management commands.
  testsuite  Test suite management commands.
  testtask   Test task management commands.
  todo       Todo management commands.
  user       User management commands.
  workload   Workload statistics and reporting.

```

让 Agent 先学会命令结构、参数格式和输出方式，再去执行真正任务。这样准确率会高很多。

### 7. 把生成出来的 CLI 接回 Agent 工作流

当一个软件成功被 CLI 化之后，它的价值才刚刚开始体现。接下来你就可以让 Agent 去做这些事：

- 自动查询数据
- 自动创建、编辑、关闭对象
- 批量导出结果
- 结合脚本做数据汇总
- 接进更大的自动化工作流

到了这一步，这个软件的能力就不再只是“页面里的按钮”，而是真正变成了一层可以被 AI 和脚本复用的接口。

## cli-anything-zentao

### 为什么禅道很适合CLI

禅道这类项目管理系统非常适合做 CLI 化，原因很简单：

- 它的领域对象很清晰：产品、项目、需求、任务、Bug、测试用例
- 它的业务动作很稳定：创建、查询、编辑、指派、关闭、解决、导出
- 它的数据结构相对明确，不依赖复杂视觉交互

这种软件一旦 CLI 化，收益会很直接，因为它天然就带有大量查询、批处理和自动化场景。

### 生成出来的 cli-anything-zentao，已经覆盖了什么

从 `GUIDE.md` 来看，`cli-anything-zentao` 已经不是演示级别，而是具备了非常完整的命令面。

它覆盖了：

- 认证管理：登录、登出、认证状态
- 产品管理：列表、详情、创建、编辑、删除
- 项目管理：列表、详情、成员、创建、编辑、删除
- 需求管理：查询、创建、变更、审核、关闭、删除
- 任务管理：查询、创建、编辑、开始、完成、暂停、重启、关闭、取消、指派
- Bug 管理：查询、创建、编辑、解决、关闭、重新激活、指派、删除
- 测试用例、发布、构建、待办、测试任务、模块、用户、工时统计、数据导出

如果只看这个覆盖范围，它其实已经很接近一个完整的“禅道命令行工作台”了。

### 它为什么适合给 Agent 使用

我觉得这份生成结果最有价值的，不只是功能多，而是它具备了几个对 Agent 特别友好的设计。

#### 1. 命令树清晰

整体采用的是非常典型的“实体 + 动作”结构：

- `product list`
- `project info`
- `story create`
- `task assign`
- `bug resolve`

这种结构对 Agent 非常友好，因为它容易被 `--help` 学习，也容易被人类理解。

#### 2. 支持上下文

它支持设置当前产品和项目上下文：

```bash
cli-anything-zentao context product 14
cli-anything-zentao context project 7
```

设置之后，很多命令就不需要重复传 `--product` 和 `--project` 参数了。对于连续工作的 Agent 来说，这一点很实用。

#### 3. 支持 JSON 输出

它支持全局 `--json` 参数：

```bash
cli-anything-zentao --json task list --project 7
cli-anything-zentao --json bug list --product 14
```

这意味着它不只是一个给人看的终端工具，同时也是一个可被程序稳定消费的结构化接口。

#### 4. 支持 REPL 模式

它还支持交互式 REPL：

```bash
cli-anything-zentao
```

这对于连续操作非常方便。Agent 可以在一个会话里持续工作，不必每次都重新初始化上下文。

### 用它可以做哪些实际工作

如果把 `cli-anything-zentao` 接给 Agent，最直接的用法至少有三类。

#### 登录认证

首次使用需要登录，登录后认证信息会自动保存，后续无需重复登录。

```bash
cli-anything-zentao --server http://你的禅道地址 auth login -u 用户名 -p 密码
```

**示例：**

```powershell
##登录
PS D:\My Project\zentaopms-zentaopms_12.5.3_20210108> cli-anything-zentao auth login -u openclaw -p ****

  Login successful
  ────────────────
  status: authenticated
  server: http://zentao.ikuaijin.vip
  account: openclaw
##登录持久化 
PS D:\My Project\zentaopms-zentaopms_12.5.3_20210108> cli-anything-zentao auth status

  Connection Status
  ─────────────────
  authenticated: True
  server: http://zentao.ikuaijin.vip
  account: openclaw
  context: no context
```

#### 产品管理

**列出所有产品**

```bash
cli-anything-zentao product list
```

**输出示例：**

```powershell
##产品列表
PS D:\My Project\zentaopms-zentaopms_12.5.3_20210108> cli-anything-zentao product list

  Products
  ────────
  14: 快进商店
  13: 快进go! APP
  12: 快进商户  APP
  11: 快进商店 小程序
  10: 神灯运营平台
##产品详情
PS D:\My Project\zentaopms-zentaopms_12.5.3_20210108> cli-anything-zentao product info 14

  Product #14
  ───────────
  title: 快进商店-需求列表
  products:
    {"14": "快进商店", "13": "快进go! APP", "12": "快进商户  APP", "11": "快进商店 小程序", "10": "神灯运营平台  "}
  modulePairs:
    []
  productID: 14
  ···
```

#### 项目管理

```shell
# 列出全部
cli-anything-zentao project list

# 按状态筛选
cli-anything-zentao project list --status doing
```

#### 需求管理

```powershell
# 列出某个产品的需求（默认只显示未关闭的）
cli-anything-zentao story list --product <产品ID>

# 列出所有需求（包括已关闭）
cli-anything-zentao story list --product <产品ID> --type all
```

#### 任务管理

```powershell
# 列出某项目的全部任务
cli-anything-zentao task list --project <项目ID>
```

![image-20260424155006797](https://cdn.wcxian.cc/img/20260424155014804.png)

```powershell
##查询周报
cli-anything-zentao task list --project 7 --assigned-to wcxian --begin-date 2026-04-13 --end-date 2026-04-17 --date-field finishedDate
```

![image-20260424155153436](https://cdn.wcxian.cc/img/20260424161502551.png)

#### BUG管理

```powershell
cli-anything-zentao bug list --product <产品ID> --begin-date 2026-04-12 --end-date 2026-04-18 --date-field openedDate
```

![image-20260424155420076](https://cdn.wcxian.cc/img/20260424161505300.png)

### 接入Openclaw

```powershell
## 安装到 PATH
cd {项目目录}/agent-harness && pip install -e .

##或者打包成whl,到别的环境安装
pip install setuptools wheel build
python -m build

##构建完成后，会在 dist/ 目录下生成 .whl 文件，例如：
my_project-0.1.0-py3-none-any.whl
```

第一版跑出来我直接把

cli_anything_zentao-1.0.0-py3-none-any.whl

丢给openclaw,因为是第一版，运行起来不如api的那么齐全

后面就撤销了，cli版需要重新优化skill

## 不是所有软件都适合 CLI 化

虽然 `CLI Anything` 很有价值，但这件事也不能说得太绝对。

我觉得更适合 CLI 化的软件，通常满足这几个条件：

- 领域模型清晰
- 业务动作稳定
- 输入输出结构化
- 不强依赖复杂视觉编辑

如果一个软件核心价值就是拖拽设计、实时所见即所得、强图形交互，那 CLI 很难完全替代 GUI。

所以 `CLI Anything` 更适合做的是：把那些本来就有稳定业务动作的软件能力，从 GUI 里“解放出来”。

## 我对 CLI Anything 的判断

如果只看概念，`CLI Anything` 很容易被理解成一个很酷的 Demo；但如果看 `cli-anything-zentao` 这样的真实生成结果，它更像是一种很有潜力的软件能力重构方式。

它做的事情，本质上是：

- 把原本依赖页面的人类操作，重构成稳定命令
- 把原本难以进入自动化流程的软件能力，重新暴露给 Agent 和脚本
- 让软件第一次真正具备“既服务人类，也服务 AI”的统一接口层

这件事的价值，不在于“以后是不是不用网页了”，而在于很多原本只能靠人点按钮的软件，现在开始真正具备了被 Agent 调用的可能性。



## 总结

CLI 之所以在 Agent 时代重新变得重要，不是因为它比 GUI 更先进，而是因为它更适合作为 AI 的工作接口。它自解释、可复现、易组合、便于脚本化，也更适合人和 Agent 协同排错。

`CLI Anything` 则把这个趋势往前推了一步。它尝试做的，不只是生成一个命令行工具，而是把原本只存在于 GUI 里的软件能力，重新组织成一套可以被 Agent 调用的命令系统。

而 `cli-anything-zentao` 这个案例说明，这件事并不只是停留在概念层面。对于领域对象清晰、业务动作稳定的软件，CLI 化确实是有机会落地的。

如果以后越来越多的软件同时提供 GUI、API、CLI、MCP 四种能力暴露方式，我一点都不会意外。因为在 Agent 时代，软件不只是给人用的，也要开始认真考虑怎么给 AI 用了。