---
title: iKuai主路由 OpenWrt旁路由 设置ipv6
published: 2025-03-30
description: ''
image: 'https://cdn.wcxian.cc/img/20250828154641398.png'
tags: [iKuai, OpenWrt]
category: 网络
draft: false 
lang: ''
---
# iKuai+旁路由OpenWrt 开启IPV6

## 步骤1 光猫改桥接

### 如下图所示，将光猫改为桥接模式(由于光猫型号过多，修改过程不赘述，自行百度)

## 步骤2 iKuai 开启IPv6

### 确认外网配置获得公网 IPV6 地址

![image-20250330223800957](https://cdn.wcxian.cc/img/202503302257006.png)

![image-20250330223850390](https://cdn.wcxian.cc/img/202503302257409.png)

### 按照下图添加内网配置并配置 IPv6 DNS，配置好之后点击保存开启 IPv6 DNS 服务器。

前缀分配长度设置为60

![image-20250330223952272](https://cdn.wcxian.cc/img/202503302257846.png)

![image-20250330224032535](https://cdn.wcxian.cc/img/202503302257095.png)

## 步骤3 设置 OpenWrt

### 进入OP,点击 `网络-接口-添加新接口`

![image-20250330224145536](https://cdn.wcxian.cc/img/202503302257591.png)





![image-20250330224631311](https://cdn.wcxian.cc/img/202503302257549.png)

### 新接口名称，填写 `IPV6` (可自定义)

### 新接口的协议，选择 `DHCPv6客户端`

### 包括以下接口，选择自定义接口并填写 `@lan` (这个切记不能填错)

![image-20250330224440662](https://cdn.wcxian.cc/img/202503302257896.png)

### 高级设置

![image-20250330224543046](https://cdn.wcxian.cc/img/202503302258382.png)

### 防火墙设置

![image-20250330224843994](https://cdn.wcxian.cc/img/202503302258581.png)

## 步骤4 OP LAN 口开启IPv6

### 基本设置 `IPv6 分配长度` 修改为 `64`

### IPv6后缀 `::1`

![image-20250330224941659](https://cdn.wcxian.cc/img/202503302258124.png)

### 因为需要使用爱快的 ipv6 DHCP 服务，所以停用 OP 的

![image-20250330225004919](https://cdn.wcxian.cc/img/202503302258799.png)

![image-20250330225057183](https://cdn.wcxian.cc/img/202503302258767.png)

![image-20250330225715689](https://cdn.wcxian.cc/img/202503302258872.png)

## 步骤5 测试 IPv6

### OP已经获得 ipv6 公网IP

https://ipw.cn/

![image-20250330225441788](https://cdn.wcxian.cc/img/202503302258310.png)

### [https://www.test-ipv6.com/](https://www.drixn.com/?golink=aHR0cHM6Ly93d3cudGVzdC1pcHY2LmNvbS8=) 测试结果

![image-20250330225527024](https://cdn.wcxian.cc/img/202503302258819.png)



