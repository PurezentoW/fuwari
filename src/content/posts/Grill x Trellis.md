---
title: 'Grill x Trellis：让 AI 真正干活的团队开发工作流'
published: 2026-06-29
description: '把 Matt Pocock 的 grill-me「动工前对齐纪律」与 Trellis 的「工程治理框架」组合，覆盖从需求盘问到稳定产出的完整生命周期，并给出可推广到团队的落地实战。'
image: 'https://cdn.wcxian.cc/img/20260629205238355.png'
tags: ['AI Coding', 'Trellis', 'grill-me', 'Claude Code', 'Agent']
category: '技术分享'
draft: false
lang: zh-CN
---
# Grill x Trellis：让 AI 真正干活的团队开发工作流

## 0. 写在前面

上一篇 [[Harness Engineering到Trellis落地实战]] 里，我聊了 Harness Engineering 的理念和 Trellis 的落地。当时我的结论是：**AI Coding 的能力 = 模型能力 × Harness 质量**。

但是还有几个场景没有考虑到：

- Spec 写得挺完整，**但 AI 一动手还是跑偏**——因为它根本没真正理解我们要做什么，只是机械地按规范堆代码；
- 一个需求丢给 Agent，它二话不说开始写，**写到一半才发现方向错了**，前面 200 轮对话全废；
- 团队里每个人用 AI 的姿势都不一样，**A 同学产出质量很高，B 同学用同一个模型却一团糟**——差距全在"人和 AI 对齐"这一步。

这些问题，光靠 Trellis 的 Spec + JSONL 注入是解决不了的。因为它们不是"规范缺失"的问题，而是**"动手前没对齐"的问题**。Trellis 默认你喂给它的需求和 PRD 是"对的"，但这个前提往往根本不成立。

直到我后来用了 [Matt Pocock 的 `grill-me`](https://github.com/mattpocock/skills/tree/main/skills/productivity/grill-me)，才发现缺的那块拼图是什么。

这篇文章要讲的就是这个主题——**把 `grill-me` 的"动工前对齐纪律"和 Trellis 的"工程治理框架"组合起来，形成一套可以推广到团队的完整开发工作流**。主要分三部分：

- **`grill-me` 是什么**，它那套"灵魂拷问"解决了 Trellis 没解决的"动手前对齐"问题
- **两者为什么天生一对**，`grill-me` 做完对齐后，正好把接力棒交给 Trellis 的任务生命周期
- **团队落地实战**，从需求到上线的完整流程，以及怎么推广到团队

---

## 1. 一句话总结

**`grill-me` 负责"想清楚"，Trellis 负责"做稳定"。** 一个让 AI 在动手前把你盘问到无路可退、彻底搞清楚你要什么；一个让 AI 在动手后被规范和上下文精准约束、保证产出一致。两者结合，才是 AI 从"玩具"走向"团队生产力"的完整答案。

---

## 2. 先回答一个关键问题：AI 写代码，到底为什么总翻车

在讲 `grill-me` 之前，我们得先把"AI 翻车的原因"拆清楚。因为 `grill-me` 和 Trellis 恰好各自命中了一半。

Matt Pocock（TypeScript 圈的大佬，Total TypeScript 作者）在介绍他这套 Skills 时，点出了 Agent 开发最致命的一个翻车场景——**Misalignment（没对齐）**：

> "你以为它懂了，它以为它懂了，结果它做的根本不是你要的。"

他还引了《程序员修炼之道》里那句经典的话：

> "No-one knows exactly what they want."（没人真的知道自己想要什么。）

这恰恰是 Trellis 完全没覆盖的环节。Trellis 的强项是"规范注入"——它假设你给的需求是对的，然后约束 AI 按规范产出。**但需求本身对不对，Trellis 不管。**

![一个需求从想法到上线，会翻两次车](https://cdn.wcxian.cc/img/20260628143427507.png)

**`grill-me` 专治第一次翻车，Trellis 专治第二次翻车。** 这就是它们天生一对的根本原因。

---

## 3. grill-me 是什么：动工前的"灵魂拷问"

### 3.1 一句话定位

[`grill-me`](https://github.com/mattpocock/skills)（GitHub: `mattpocock/skills`）是 Matt Pocock 那套 "Skills for Real Engineers" 里的一个轻量 Skill。整套 Skills 的口号是 "not vibe coding"（拒绝凭感觉编码），而 `grill-me` 是其中**最纯粹、最核心、也最受欢迎的一个**。

![[mattpocock](https://github.com/mattpocock)](https://cdn.wcxian.cc/img/20260628151244146.png)

**Grill 的字面意思就是"烤架"——引申为"严加盘问"。**

它做的事可以用一句话概括：**在 AI 动手写任何代码之前，先让它反过来盘问你，一个问题一个问题地问，沿着决策树的每一个分支往下钻，直到每一个含糊的地方都被钉死。**

`grill-me` 这个 Skill 本身极其精简——它底层调用的 `/grilling`，核心 Prompt 只有这么几行：

```text
Interview me relentlessly about every aspect of this plan
until we reach a shared understanding. Walk down each branch
of the design tree, resolving dependencies between decisions
one-by-one. For each question, provide your recommended answer.

Ask the questions one at a time, waiting for feedback on each
question before continuing.

If a question can be answered by exploring the codebase,
explore the codebase instead.
```

翻译过来就是**四条铁律**：

1. **逐个问，别一次抛一堆**——一次一个问题，等你回答了再问下一个。Matt 原话："Asking multiple questions at once is bewildering"（一次问一堆会让人懵）；
2. **每个问题都给推荐答案**——不是干巴巴地问，而是带着 AI 自己的判断问，你来拍板；
3. **沿着决策树往下钻**——不是随机提问，而是按决策之间的依赖关系一个个解决，前面的不钉死，后面的没法谈；
4. **能看代码就别问人**——能从代码库里查到的，AI 自己去查，别拿这种问题烦你。

### 3.2 为什么这招这么管用

你可能会问：我自己写 PRD 不也是在澄清需求吗？为什么要让 AI 来盘问我？

因为人有一个致命弱点：**我们以为知道自己想要什么，其实并不知道。**

当你自己写需求文档时，你会本能地跳过那些"感觉很明显"的细节。但 AI 不知道你的"感觉"，它只能根据你写下来的字面意思干活。**那些被你跳过的含糊地带，就是 AI 翻车的雷区。**

让 AI 反过来盘问你，本质上是**强迫你把脑子里的隐性知识显性化**。每被问住一次，就排掉一颗雷。

举个真实场景。需求是"做一个积分系统"，你觉得自己说得很清楚了。但 `/grill-me` 会这么问你（每个问题都带着它的推荐答案）：

```text
AI: 我们要做的"积分"，是指可消费的虚拟货币，
    还是不可消费的等级积分？
    我倾向前者——因为你说的是"积分系统"，通常隐含可兑换。
    请确认。

你:  对，是可消费的。

AI: 那积分会被消耗在哪些场景？充值？兑换商品？打赏？
    我推荐先只支持"兑换商品"，其他后续迭代。
    这样切片最小，能最快验证。

你:  兑换商品 + 打赏，这两个都要。

AI: "打赏"是指用户之间转移积分对吧？
    那并发场景下，如果 A 和 B 同时给 C 打赏，
    C 的余额该怎么算？是按事务先后扣减，
    还是允许短暂超扣再用对账兜底？
    我推荐事务先后扣减——简单，且不会超扣。
    但这会要求打赏走强一致性路径。
    请确认。

你:  ……（被问住了，这确实没想过）
```

注意第三问——**这就是 `grill-me` 的价值所在**。你原本以为"做一个积分系统"很清楚，但并发打赏的扣减语义这种关键细节，你不被问就根本想不起来。而一旦没想清楚，AI 自己脑补一个实现，上线后就是 bug。

### 3.3 它和 Trellis 的 brainstorm 有什么不一样

熟悉 Trellis 的同学可能会问：Trellis 不是也有 `/trellis-brainstorm` 做需求澄清吗？为什么要再引入 `grill-me`？

两者定位确实重叠，但**纪律强度不一样**：

| 维度           | Trellis `/trellis-brainstorm`           | `grill-me`（grilling）                     |
| :------------- | :-------------------------------------- | :----------------------------------------- |
| **核心动作**   | 引导式提问，收敛 MVP 范围               | **无路可退的盘问**，决策树每个分支钉死     |
| **提问节奏**   | 一次可以问几个高价值问题                | **严格一次一个**，答完再问下一个           |
| **推荐答案**   | 有，但偏引导                            | **每个问题都带 AI 的明确推荐**，你只需拍板 |
| **能否看代码** | 可以                                    | **优先看代码**，能查到的绝不问人           |
| **产物**       | PRD、task.json（和 Trellis 任务流绑定） | **只产出"对齐共识"**，不绑定任何后续流程   |

说白了，`/trellis-brainstorm` 是**温和的需求收敛**，目标是"把 MVP 圈出来"；`grill-me` 是**严苛的逐点拷问**，目标是"把每个含糊处逼到非黑即白"。

我的建议是**两者都能用，按需求复杂度选**：

- 简单需求、边界清晰 → 直接用 Trellis 的 brainstorm，快；
- 复杂需求、或者你隐约觉得"哪里没想透" → 上 `grill-me`，盘问到清爽为止。

---

## 4. 为什么 grill-me 和 Trellis 天生一对

讲完两者，我们把它摊开来看它们怎么接力。

![微信图片_20260502085908_55_1955](https://cdn.wcxian.cc/img/20260625161458380.jpeg)

### 4.1 一张图看清分工

![一个需求的完整生命周期](https://cdn.wcxian.cc/img/20260628150924927.png)

**一句话**：`grill-me` 管的是"动工前"，Trellis 管的是"动工后"。中间的交接点，就是盘问结束后凝练出的那份**对齐共识**（你可以把它直接写成 PRD 喂给 Trellis）。

### 4.2 流程的接力点

`grill-me` 的妙处在于：**它本身很轻，只产出"对齐共识"，不绑定任何后续流程。** 这反而让它能完美地嵌进 Trellis 的任务生命周期，当那个"动手前"的对齐环节：

![grilli-me -> Trellis 接力](https://cdn.wcxian.cc/img/20260628150933564.png)

**关键洞察**：`grill-me` 把"需求到底对不对"这件事在动手前解决掉，Trellis 接手时拿到的就是一个"已经对齐过、没有含糊地带"的需求。Trellis 的 Spec 约束"代码长什么样"，`grill-me` 保证"需求本身没问题"——两者叠加，AI 才既不会跑偏，又不会乱写。

### 4.3 为什么这个组合比单用任何一个都强

| 场景     | 只用 Trellis         | 只用 grill-me                  | **grill-me + Trellis**              |
| :------- | :------------------- | :----------------------------- | :---------------------------------- |
| 需求含糊 | 盲写，跑偏才发现     | 盘问清楚，但写到一半换会话就乱 | **先盘问清楚，再稳定执行**          |
| 跨会话   | JSONL 注入保规范     | 不管跨会话                     | **规范 + 意图都不丢**               |
| 多人协作 | Worktree 隔离        | 不管多人                       | **每人盘问各自需求，Spec 共享兜底** |
| Bug 反复 | /break-loop 沉淀经验 | 不管沉淀                       | **盘问防新坑 + break-loop 治旧坑**  |
| 团队推广 | Spec 是共享基座      | 纯个人习惯，难推广             | **Spec 共享 + 盘问纪律统一**        |

单用任何一个都有明显的洞。**组合起来，从"想清楚"到"做稳定"全链路没有断点。**

---

## 5. 团队落地实战：一个完整需求的完整流程

光讲理论不够，这边我走一遍完整的团队流程。假设我们要做一个**"用户积分系统"**的需求。

### 5.1 第 0 步：一次性环境准备

#### 安装 grill-me

在项目根目录跑（支持 Claude Code、Cursor、Codex 等）：

```bash
npx skills@latest add mattpocock/skills
```

![PixPin_2026-06-28_15-14-34](https://cdn.wcxian.cc/img/20260628151453951.png)

选 Skill 时，**只勾 `grill-me` 就够了**（它属于 productivity 类，不需要跑 `/setup-matt-pocock-skills` 那套 issue tracker 配置，门槛很低）。

装完后，它会作为一个 slash command 出现，用 `/grill-me` 触发。

#### 接入 Trellis

```bash
trellis init -u yourname
```

这步我上一篇讲过，会在项目里生成 `.trellis/` 目录（spec / tasks / workspace）。

**两者的配置互不冲突**：`grill-me` 是个运行时 Skill，不往仓库里写任何配置文件；Trellis 写 `.trellis/`。各管各的。

### 5.2 第 1 步：盘问对齐（grill-me 主场）

现在来了一个需求："做一个用户积分系统"。**别急着让 AI 写代码，先盘问。**

```text
/grill-me
```

接下来你会经历一场"灵魂拷问"。AI 会一个一个地问你，沿着决策树往下钻。大致会是这样的节奏（每个问题都带着它的推荐答案，你只需确认或修正）：

![PixPin_2026-06-28_14-55-01](https://cdn.wcxian.cc/img/20260628151644367.png)

![image-20260629203717604](https://cdn.wcxian.cc/img/20260629203726623.png)

**这一步结束，你得到的是一份"被盘问得清清楚楚、所有含糊地带都钉死"的需求共识。**

![image-20260629203822749](https://cdn.wcxian.cc/img/20260629203833369.png)

这一步是 Trellis 从来没帮你做过的，而它恰恰是团队里 A 同学和 B 同学产出质量差距的根源——A 同学脑子清楚、自己能把需求想透，B 同学需要被逼着才能想透。**`grill-me` 把 A 同学的"想清楚"能力，变成了一套所有人都能照着走的纪律。**

### 5.3 第 2 步：把对齐共识写成 PRD（交接给 Trellis）

`grill-me` 本身不产出文件，它只产出"对话里的共识"。盘问完之后，你需要把这份共识固化成 Trellis 能消费的 `prd.md`。

两种做法：

- **手写**：照着盘问记录，把决策整理成 PRD（适合关键需求，你想自己把关）；
- **让 AI 综合**：直接在同一会话里说"把刚才盘问的所有结论整理成一份 PRD"，AI 会综合整个对话生成。

![PixPin_2026-06-28_15-04-28](https://cdn.wcxian.cc/img/20260628151720125.png)

PRD 长这样：

```markdown
# 营销活动报表（限时满减 / 限时折扣）

> 本 PRD 记录需求与验收标准。技术设计见 `design.md`，执行计划见 `implement.md`（复杂任务，二者在 `task.py start` 前补齐）。所有决策来自一轮 grill-me 对齐会议，已逐项确认。

---

## 一、背景

现有营销域只有「活动管理」（增删改查）和活动详情页的 3 个汇总数（订单数 / 成交额 / 优惠额，来自 T+1 汇总表 `kj_promotion_statistics`）。**缺少跨活动横向对比和单活动纵向复盘能力**，运营无法回答"这批活动哪个效果好""某个活动里哪些商品卖得最好"。

本任务为「限时折扣」(type=1) 和「满减活动」(type=2) 两种活动补齐**活动报表**，分两层：跨活动汇总对比 + 单活动下钻复盘。

---

## 二、对齐决策（grill-me 共识，9 项）

---

## 三、功能需求

---

## 四、接口清单（6 个）

---

## 五、关键约束（口径与一致性）

---

## 六、改动范围（预计 ~12 文件，全部新增、零侵入现有逻辑）

---

## 七、验收标准

**功能**

**口径**

**工程**

**性能**


---

## 八、明确不做（v2 backlog）

---

## 九、依赖与风险

---

## 十、后续步骤（交接给 Trellis 执行流程）

```

**注意：PRD 里的每一条决策，都是刚才被 `grill-me` 钉死过的，不是凭感觉写的。** 这就是盘问的价值——你的 PRD 从"我以为我想清楚了"变成"我真的被逼着想清楚了"。

### 5.4 第 3 步：进入 Trellis 任务流（Trellis 主场）

PRD 写好后，接力棒正式交给 Trellis。这一步开始就是我上一篇讲过的流程：

```text
trellis start  # 或 /trellis:continue
```

Trellis 会自动：

1. **创建 `task.json`**，记录任务元数据（状态、负责人、分支）；
2. **创建 Git Worktree**，把这个任务隔离到独立分支，多人/多 Agent 并行不踩脚；
3. **注入 `implement.jsonl`** 定义的所有 Spec 文件——Controller 规范、错误处理规范、日志规范、返回格式规范……全部喂给 AI；
4. **创建 Journal**，开始记录本次会话。

**到这一步，AI 同时被两样东西托着：**

- `grill-me` 盘问出来的**对齐共识**（写进了 PRD，AI 知道"要做什么"）；
- Trellis 的 **Spec**（AI 知道"代码长什么样"）。

**意图 + 规范，一起喂给 AI。** 这就是为什么组合比单用强——单用 Trellis，AI 知道"怎么写"但不知道"写什么是对的"；单用 `grill-me`，AI 知道"写什么"但没有规范兜底。**两个一起，意图和规范都齐了。**

### 5.5 第 4 步：实现 + 质量检查 + 归档（Trellis 主场）

实现、检查、归档这几步全是 Trellis 的强项，我上一篇详细讲过，这里简单带过：

```bash
# 编码时，Trellis 已注入 Spec，AI 自动按规范写
# （不用再重复说"用 MyBatis-Plus""日志用 Slf4j""返回 Result<T>"——都在 Spec 里了）

/check-backend    # Spec 合规检查：目录结构、异常处理、日志、返回格式逐项过
/finish-work      # 触发 AI Code Review + Spec 合规
/record-session   # 归档，生成 Journal
```

如果上线后发现 bug，用 Trellis 的 `/break-loop` 做根因分析——它会分析"为什么之前的修复失败了"，然后把经验沉淀回 Spec。**下次换个人、换个会话，不会再踩同一个坑。**

---

## 6. 怎么推广到团队

上面讲的是个人怎么用。但你的目标是**推广到团队**，这才是难点。我把推广分成三个阶段。

### 6.1 第一阶段：建立共享基座（1-2 周）

团队推广的第一步，不是让所有人学会所有 Skill，而是**建立共享的知识基座**。对这套组合来说，共享基座只有一个——**Trellis 的 Spec**：

```text
仓库根目录
└── .trellis/
    └── spec/               ← 技术规范（团队共建，随仓库版本化）
        ├── shared/
        │   └── index.md    ← 跨技术栈通用规范
        └── backend/
            ├── index.md
            ├── database-guidelines.md
            ├── error-handling.md
            └── ...
```

> 这里要特别说明：Matt 那套 Skills 里还有个 `grill-with-docs` 会额外沉淀 `CONTEXT.md`（业务术语）和 ADR（架构决策）。**但我们这次只用 `grill-me`，它不产出任何文件**，所以团队共享基座就只有 Trellis 的 Spec 这一份。这样反而更简单——团队只需要维护一个 `.trellis/spec/`，心智负担低，推广阻力小。

**Trellis 官方文档把这一步叫"团队共享标准"**：Spec 随仓库一同版本化，个人总结的规则与流程可以直接成为整个团队的基础设施。

这一步要做的：

- 拉一次全团队的工作坊，把团队现有的编码规范迁移进 `.trellis/spec/`；
- 约定 Spec 的分层结构（shared / backend / frontend）；
- 让每个人 `trellis init` 接入，跑通第一个需求。

**这一步的价值是"拉高全员基线"**——以后不管谁用 AI，喂进去的规范基座都是一样的。这是推广能成立的地基。

### 6.2 第二阶段：统一工作流（2-4 周）

基座建好后，约定一套**所有人都遵守的工作流**。核心就两条纪律：

```text
团队统一工作流（建议贴在团队 wiki）

任何非 trivial 的需求，都必须走这条主线：

  1. /grill-me           ← 动手前盘问对齐（不许跳过）
  2. 把共识写成 prd.md    ← 固化盘问结果
  3. trellis start        ← 进 Worktree，注入 Spec
  4. implement            ← AI 按对齐需求 + Spec 写代码
  5. /check-backend       ← Spec 合规检查
  6. /finish-work         ← 归档

trivial 的小修小改（改个文案、调个配置）可以跳过 grill-me。
判断标准：这个改动有含糊地带吗？有就别跳盘问。
```

推广时最常见的两个反对意见，我都帮你准备好反驳：

- **"盘问太慢了，直接写不是更快？"**
  → 反驳：你省下的是盘问的 30 分钟，赔上的是返工的 3 小时。盘问是把返工成本前置。而且 `grill-me` 每个问题都带推荐答案，你大多数时候只需点头，没你想的那么慢。

- **"我们团队用 Cursor / Codex，不是 Claude Code，能用吗？"**
  → 这正是这套组合能推广的命门，见下一节。

### 6.3 第三阶段：不挑平台，这才是能推广的关键

这一点必须单独拎出来讲，因为它是**能不能推广成功的命门**。

团队里一定有人用 Claude Code，有人用 Cursor，有人用 Codex，有人用 Gemini。如果工作流绑死一个平台，推广必败。

**而 `grill-me` 和 Trellis 恰好都不挑平台：**

| 维度           | grill-me                                                     | Trellis                                                      |
| :------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **跨平台**     | 遵循 `.agents/skills/` 约定（agentskills.io 规范），任何兼容的 Agent 都能装 | 官方支持 **16 个平台**（Claude Code、Cursor、Codex、Gemini、OpenCode…） |
| **本质**       | 一个 Markdown 写的 Skill 文件                                | 一套 `.trellis/` 目录结构（Markdown + JSON）                 |
| **平台差异**   | 几乎没有，Skill 就是纯文本指令                               | `trellis init` 为每个平台写入各自配置目录，核心概念跨平台一致 |
| **团队共享物** | 无（grill-me 不写文件，纯运行时纪律）                        | `.trellis/spec/`（纯 Markdown，随仓库版本化）                |

**这意味着：团队里 A 用 Claude Code、B 用 Cursor，他们读的是同一份 Spec、同一份 PRD，盘问走的是同一套 `grill-me` 纪律。** 换工具不影响产出，这才是能推广的基础。

Trellis 官方文档的原话：

> 同一套 Trellis 结构覆盖 16 个 AI coding 平台。`.trellis/` 下的核心概念跨平台一致，差异仅在于 command、skill、sub-agent、hook 的交付方式。

```text
┌──────────────────────────────────────────────────────────┐
│                  团队的共享层（和平台无关）              │
│                                                          │
│              .trellis/spec/  （技术规范）                │
│                    │                                     │
│                    │  + grill-me 纪律（运行时，不挑平台  │
│                    │                                     │
│         任何人、任何平台都能用                           │
│                    │                                     │
│     ┌──────────┬───┴────────────┬───────────┐            │
│     ▼          ▼                ▼           ▼            │
│  Claude     Cursor           Codex       Gemini          │
│  Code                                        等 16 个    │
└──────────────────────────────────────────────────────────┘
```

**注意一个细节**：`grill-me` 是运行时纪律，它不往仓库里写东西，所以它本身不需要"团队共享"——每个人在自己机器上装一份就行。**真正需要团队共享的只有 `.trellis/spec/`**，而它是纯 Markdown，跟着 git 走。这让推广变得异常干净。

### 6.4 团队落地检查清单

最后给一份可直接用的团队落地清单：

- [ ] `.trellis/spec/` 覆盖了团队的主要技术栈，且随仓库版本化
- [ ] 团队 wiki 写明了统一工作流（6 步主线）
- [ ] 每个成员都在自己机器上装好了 `grill-me` 和 Trellis
- [ ] 至少跑通一个完整需求（从 grill-me 到 finish-work）
- [ ] PR 模板里加一句："本需求是否经过 grill-me 盘问对齐？"
- [ ] 新人 onboarding 文档里有这套流程的说明
- [ ] 定期 review Spec 是否需要更新（项目演进后 Spec 要同步）

---

## 7. 优势与局限

### 7.1 优势

| 优势                 | 说明                                                         |
| :------------------- | :----------------------------------------------------------- |
| **覆盖完整生命周期** | grill-me 管"想清楚"，Trellis 管"做稳定"，从 idea 到 ship 无断点 |
| **不挑平台**         | 两者都跨平台，团队 heterogeneous 工具栈也能统一              |
| **分工干净**         | grill-me 只管对齐不写文件，Trellis 只管治理，互不污染        |
| **上手门槛低**       | grill-me 一个 Skill 文件，Trellis 一条 `trellis init`，没有复杂的 issue tracker 配置 |
| **知识可沉淀**       | PRD（需求）+ Spec（规范）+ Journal（经验）三重沉淀，全在 Trellis 侧 |
| **对齐可复现**       | 盘问纪律让"需求澄清"不再是玄学，新人也能产出高质量 PRD       |
| **渐进式采用**       | 可以先只用 grill-me 盘问，或先只用 Trellis 的 Spec，逐步叠加 |

### 7.2 局限

| 局限                  | 说明                            | 缓解方式                                               |
| :-------------------- | :------------------------------ | :----------------------------------------------------- |
| **盘问有 token 成本** | 一次 grill-me 会消耗不少 token  | 大需求才盘问，小修小改跳过；每个问题带推荐答案，回答快 |
| **依赖团队纪律**      | 有人不遵守工作流就会破坏一致性  | 把工作流写进 PR 模板/CI，流程化约束                    |
| **盘问质量看人**      | 被盘问者自己答错了，AI 也救不了 | 关键决策盘问时拉上熟悉业务的人一起                     |
| **Spec 维护成本**     | 项目演进后 Spec 需要同步更新    | 把 `/update-spec` 作为日常工作流的一部分               |
| **小任务收益低**      | 改一行配置也走完整流程是浪费    | trivial 改动用"跳过 grill-me + 跳过 Trellis task"模式  |
| **Skill 生态在迭代**  | Matt 的 Skills 还在频繁更新     | `grill-me` 本身极简稳定，锁版本即可                    |

### 7.3 什么时候不该用

别为了用而用。以下场景，这套组合反而拖累：

- **一次性脚本、原型** —— 盘问和 Spec 都是负担，直接 vibe coding；
- **单人小项目** —— Trellis 的 Worktree/Task 状态机是过度设计，光用 grill-me 盘问就够了；
- **需求极度明确的小改动** —— 改个文案、调个阈值，别走六步流程；
- **团队对 AI 还很陌生** —— 先让每个人用好单会话 Claude Code，再谈团队工作流。

---

## 8. grill-me、Trellis 与同类方案的对比

把这套组合放进更大的生态里看：

![grill-me、Trellis 与同类方案对比](https://cdn.wcxian.cc/img/20260628152341993.png)

注意最后一行：**单用 grill-me 在"团队可推广"上是弱的**——因为它是纯个人运行时纪律，不沉淀任何共享物。**只有和 Trellis 结合，盘问纪律才有了团队共享的 Spec 基座托着，才能推广。** 这就是组合的意义。

---

## 9. 总结

回到开头的问题：**为什么团队里 AI 用得好坏差距这么大？**

我的答案是：因为大多数人只装备了"让 AI 写代码"的能力，没有装备"让 AI 想清楚"和"让 AI 做稳定"的能力。

这三个能力，对应三件事：

1. **想清楚** —— `grill-me` 的盘问纪律，让需求在动手前就被钉死，所有含糊地带逼到非黑即白；
2. **做稳定** —— Trellis 的 Spec + JSONL + Worktree，让产出跨会话、跨人、跨模型保持一致；
3. **持续进化** —— Trellis 的 Journal + `/break-loop`，让每一次开发和修 bug 都变成知识沉淀，Spec 越用越厚。

把这三个关键词记下来：**结构化、可沉淀、可复现。**

最后，关于"推广到团队"，我最想强调的一点是：**不要追求所有人用同一个平台，而要追求所有人共享同一套 Spec 基座 + 同一套盘问纪律。** `grill-me` 是运行时纪律，每人装一份；`.trellis/spec/` 是纯 Markdown，跟着仓库走。AI 工具年年换，但这些规范和纪律是团队真正的资产。

毕竟，AI 编程进入团队时代后，比拼的已经不是谁用的模型更强，而是**谁的团队把模型驾驭得更系统**。

```text
  团队 AI 生产力 = 模型能力 × 对齐纪律 × 工程基座

  模型能力大家在追，对齐纪律靠 grill-me 统一，
  工程基座——Spec——才是你能主动构建、长期复利的部分。
```

---



![image-20260625165343661](https://cdn.wcxian.cc/img/20260625165347314.png)

![image-20260625165405379](https://cdn.wcxian.cc/img/20260625165408603.png)

## 参考资料

> [Matt Pocock Skills — Skills for Real Engineers](https://github.com/mattpocock/skills)
>
> [grill-me — 仓库内 Skill 定义](https://github.com/mattpocock/skills/tree/main/skills/productivity/grill-me)
>
> [grilling — 盘问底层循环的 Skill 定义](https://github.com/mattpocock/skills/tree/main/skills/productivity/grilling)
>
> [Trellis 多平台与团队配置 — 官方文档](https://docs.trytrellis.app/zh/advanced/multi-platform)
>
> [Trellis 附录 F：FAQ — 官方文档](https://docs.trytrellis.app/zh/advanced/appendix-f)
>
> [Trellis GitHub 仓库（中文 README）](https://github.com/mindfold-ai/Trellis/blob/main/README_CN.md)
>
> [Trellis 安装与快速上手 — 官方文档](https://docs.trytrellis.app/zh/start/install-and-first-task)
>
> [Harness Engineering 到 Trellis 落地实战（本系列上一篇文章）](Harness%20Engineering到Trellis落地实战.md)
>
> [The Pragmatic Programmer — David Thomas & Andrew Hunt](https://www.amazon.co.uk/Pragmatic-Programmer-Anniversary-Journey-Mastery/dp/B0833F1T3V)