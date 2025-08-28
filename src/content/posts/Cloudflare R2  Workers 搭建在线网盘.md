---
title: Cloudflare R2 Workers 搭建在线网盘
published: 2025-07-30
description: ''
image: 'https://cdn.wcxian.cc/img/20250828153336785.png'
tags: [Cloudflare,Workers]
category: '技术分享'
draft: false 
lang: ''
---
# 利用 Cloudflare R2 Workers 搭建在线网盘



## 0.前言

Cloudflare 是一家提供网站加速和安全服务的公司。主要功能包括：**CDN 加速** 、**DDoS 防护**、**DNS 服务**、**反向代理** 等功能。

我们访问一些网站的时候，经常可以看到这样一个人机验证页面，这就是Cloudflare提供的免费服务之一。



![img](https://cdn.wcxian.cc/img/20250725232201487.png)

它可以免费CDN加速，同时也可以隐藏我们服务器的真实ip地址，还能免费开启攻击防御。



## 1.Cloudflare R2

对象存储相信很多人都有所耳闻，例如腾讯云 COS、阿里云 OSS、七牛云 KODO 等。

那么，为什么我们还要使用 Cloudflare R2 呢？答案在于它拥有独特的优势：不限访问流量。

对于对象存储来说，流量费用是一项不小的支出。Cloudflare 直接免除流量费用，堪称慈善大佛。

| 类型     | 免费额度           | 超额付费          |
| -------- | ------------------ | ----------------- |
| 存储     | 10 GB-月/月        | $0.015/GB/月      |
| A 类操作 | 100 万个请求 / 月  | $4.50/100万个请求 |
| B 类操作 | 1000 万个请求 / 月 | $0.36100万个请求  |
| 出口流量 | 全球免费①          | 全球免费①         |

Cloudflare R2 主要靠存储容量收费，10G 以内是免费的，对于写写博客做个图床完全是够用的。就算是收费也不贵，100G 差不多也就 10 大洋多点。A 类操作和 B 类操作是指一些读写操作，应该没有多少网站能超过这个数。总的来说，的确够实惠。

## 2.账户配置

### 注册账号

首先你得有个 Cloudflare 账号

[随时随地连接、保护、构建 | Cloudflare](https://www.cloudflare.com/zh-cn/)

### 添加订阅

和其他 CF 免费服务不同，**R2 超出免费额度后不是自动停用等待下个月额度恢复，而是扣钱按量付费**，因此需要绑定一个支付手段。因为我之前在 CF 购买域名添加了 paypal (绑定招行的人民币借记卡)，因此可以直接使用 paypal。（实际上给的免费额度也十分充裕，对于个人乃至一大坨人使用的图床已经足够了）

![image-20250725210753490](https://cdn.wcxian.cc/img/20250725232208133.png)

### 创建 R2 储存桶

前往 Cloudflare R2 新建一个 R2 储存桶。

![image-20250725140047125](https://cdn.wcxian.cc/img/20250725232211333.png)

设置你的存储桶名，名称随意设置，然后点击创建。

![image-20250725140239298](https://cdn.wcxian.cc/img/20250725232214506.png)

- 存储桶名称：自己填写
- 位置：**亚太地区** 或 **北美洲西部** (实际速度差不多)
- 默认存储类：标准（不能选**不频繁访问**，没有免费额度）

### 通过r2.dev子域名访问

**（第一种方法，不推荐使用，可直接忽略此步骤）** 在储存桶设置中 R2.dev 子域 允许公开访问，复制公共储存桶 URL。 如开启 R2 存储桶的公开读写权限！你的存储资源有可能会被恶意刷取，造成资源付费。

开启了会获得一个公共 R2.dev 存储桶 URL

``` 
https://pub-*****.r2.dev
```

![image-20250725142731633](https://cdn.wcxian.cc/img/20250725232217583.png)

### 使用自定义域访问

**（第二种方法，推荐使用）** 使用R2的 自定义域名 您的存储桶的内容将可以通过该域公开访问。连接的网站还可以受益于 Cloudflare 功能，如机器人管理、访问和缓存。

![image-20250725143106417](https://cdn.wcxian.cc/img/20250725232220433.png)

![image-20250725143142794](https://img2.wcxian.cc/img/2025/07/3f6996a3807903f9d38a74dfb3941457.png)

## 3.创建 Cloudflare Pages 站点

### Fork 仓库

在 Github Fork 项目

[willow-god/FlareDrive-R2: 😁基于cloudflare-r2实现的网盘，可以实现文件分享功能](https://github.com/willow-god/FlareDrive-R2)

### 创建站点

![image-20250725140904848](https://cdn.wcxian.cc/img/20250725232223603.png)

![image-20250725143340327](https://cdn.wcxian.cc/img/20250725232226686.png)

前往 Cloudflare Pages，新建站点，选择连接到 Git，并选择刚刚 Fork 的仓库。点击开始设置。

![image-20250725143450092](https://cdn.wcxian.cc/img/20250725232229438.png)

### 环境变量

在 Cloudflare Pages 项目中，进入 **Settings → Environment Variables** 添加以下变量：

| 变量名         | 示例值                                                | 是否必要 | 说明                                           |
| -------------- | ----------------------------------------------------- | -------- | ---------------------------------------------- |
| `PUBURL`       | `https://pub-kdsjfhlasnwiuweia4387rfho85tnof4.r2.dev` | ✅ 必填   | R2 公共存储桶地址                              |
| `admin:123456` | `*`                                                   | ✅ 必填   | 管理员账号，格式为 `用户名:密码`               |
| `GUEST`        | `public/`                                             | ❌ 可选   | 游客写入的默认目录                             |
| `user1:123456` | `user1/,shared/`                                      | ❌ 可选   | 普通用户及其可写入目录，支持多个目录，格式一致 |

项目名称可自行修改，其它项目保持默认设置。点击环境变量并添加如下变量，自用只需要设置个管理员账号就行。

![image-20250725165256006](https://cdn.wcxian.cc/img/20250725232234254.png)

### 部署成功

![image-20250725143854632](https://cdn.wcxian.cc/img/20250725232237436.png)

### 绑定 R2 储存桶

前往 Pages > cloudflare-r2-oss > 设置 > 函数 > R2 储存桶绑定，绑定对应的 R2 储存桶，变量名称为 BUCKET。

![image-20250725144801141](https://cdn.wcxian.cc/img/20250725232240361.png)

### 重新部署

在部署页面点击重新部署按钮，完成设置后即可正式使用。

![image-20250725144928314](https://cdn.wcxian.cc/img/20250725232242913.png)

### 设置自定义域名

因为我的域名就在cloudflare,所以可以直接一键设置域名，设置完就自动有https

![image-20250725145105169](https://cdn.wcxian.cc/img/20250725232245556.png)

## 4.在线网盘

这下就可以直接访问网盘主页了

https://pan.wcxian.cc/

[/ - 优雅的 Cloudflare R2 网盘文件库](https://pan.wcxian.cc/)

![image-20250725212130895](https://cdn.wcxian.cc/img/20250725232248612.png)


<iframe width="100%" height="468" src="https://img.wcxian.cc/video/bandicam%202025-07-25%2022-47-32-184.mp4" title="YouTube video player" frameborder="0" allowfullscreen></iframe>



## 5.超过 300 MB 大文件上传

超过 300 MB 的文件仅支持通过 S3 兼容性 API 或 Workers 进行上传。
因此，建议采用 PicList 方案进行设置，详情请参考本篇博文 PicList - 一款高效的云存储和图片托管平台管理工具。

![image-20250727210951564](https://cdn.wcxian.cc/img/20250727212242288.png)

![image-20250727210542347](https://cdn.wcxian.cc/img/20250727212245877.png)

![image-20250727212101268](https://cdn.wcxian.cc/img/20250727212248644.png)

图床配置名=随意设置
应用密钥ID=访问密钥 ID
应用密钥=机密访问密钥
桶名=你R2存储桶创建的名称
文件路径={fileName}.{extName}
自定义节点=客户端使用管辖权地特定的终结点

## 6.Cloudflare R2 + Workers 方案的独特优势：

- **极低成本：** R2核心优势是无出站流量费，这意味着大量数据访问（例如视频播放、图片加载）不会产生高额的CDN费用。存储费用本身也极具竞争力。
- **全球边缘加速：** Cloudflare的CDN网络遍布全球，Workers在边缘运行，数据存储在R2后，可以通过Workers在最近的边缘节点进行处理和分发，实现极低的延迟。对于无人商店的“快进”体验至关重要。
- **高可扩展性：** 无需关心服务器扩容，R2和Workers天生具备弹性伸缩能力，可以应对访问量的瞬间爆发。
- **内置安全：** 利用Cloudflare的DDoS防护、WAF等能力，为网盘服务提供高级安全保护。Workers可以实现细粒度的访问控制和鉴权。
- **开发简洁：** Workers使用JavaScript/TypeScript开发，生态成熟，门槛相对较低，可以快速迭代和部署。
- **易于集成：** R2兼容S3 API，可以方便地与现有工具和SDK集成。Workers可以暴露标准的API接口供其他系统调用。

参考资料：

> [Cloudflare R2对象存储搭建高速免费图床完全指南几年前，当我第一次把域名扔进Cloudflare的怀抱时，仿佛 - 掘金](https://juejin.cn/post/7471968780865093659)
>
> [Cloudflare R2 + Workers：在线网盘 使用 Cloudflare Pages 快速部署教程 | 周润发博客](https://blog.zrf.me/p/Cloudflare-R2-oss/)
>
> [Cloudflare R2 对象存储白嫖指南：10G存储+免流量费，打造免费图床](https://blog.wpjam.com/article/cloudflare-r2/)