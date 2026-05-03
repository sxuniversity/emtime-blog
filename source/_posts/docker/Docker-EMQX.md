---
title: Docker部署MQTT服务器
categories: docker
date: 2024-08-20 17:49:12
updated: 2024-08-20 17:49:12
---

参考：
- https://blog.csdn.net/mftang/article/details/136585110
- https://mftang.blog.csdn.net/article/details/136601795

官网：https://www.emqx.com/zh

在物联网系统中，消息队列是设备通信的核心，而 MQTT 作为轻量、高效的通信协议，已经成为行业默认标准；说到 MQTT Broker，大多数人第一时间想到的就是 EMQX：一款高性能、开源且支持企业级功能的 MQTT 消息服务器；

> **注意**：企业级功能或者商用需要付费使用；

不过 EMQX 的部署一直被人觉得“偏重”，各种配置繁琐、界面复杂，特别是初学者常常被一堆文档劝退；其实用 Docker 部署 EMQX，整个过程不仅轻量而且非常优雅；只要几条命令，就可以在本地快速跑起来，甚至连可视化管理界面都顺带搭好了；

本篇文章会一步步带你通过 Docker 部署 EMQX，适合刚接触 MQTT 或想自建消息服务的小伙伴；

> 顺带一提，EMQ 还提供了一个更轻量的版本： [NanoMQ](https://nanomq.io/zh)，专为嵌入式 Linux 设备设计，如果你后续在做边缘侧 MQTT 应用，也可以考虑这个版本；

## Docker 部署 EMQX

部署方式选择的是官方提供的 开源版 + 自托管 + Docker 镜像：

下载地址（可查看所有平台选项）：https://www.emqx.io/zh/downloads

直接使用如下命令完成镜像拉取与启动：

```bash
# 拉取 EMQX 开源版镜像（版本 5.7.2）,如果想看别的版本可以去docker hub查看
docker pull emqx/emqx:5.7.2

# 启动容器并映射常用端口
docker run -d --name emqx \
  -p 1883:1883 \
  -p 8883:8883 \
  -p 8083:8083 \
  -p 8084:8084 \
  -p 18083:18083 \
  emqx/emqx:5.7.2
```

## 常用端口说明

| 端口号 | 协议说明 |
| ----------- | ----------- |
| 1883 | MQTT over TCP（最常用端口）|
| 8883 | MQTT over SSL/TLS（加密传输） |
| 8083 | MQTT over WebSocket |
| 8084 | MQTT over WSS（加密 WebSocket） |
| 18083 | HTTP Dashboard & REST API |
| 4370 / 5370 | 集群通信端口（本次没用到） |
| 5683 | CoAP端口（本次没用到） |

## 访问管理后台（Dashboard）

由于 EMQX 的网页管理后台需要通过 18083 端口访问，为了方便管理，我通过本地 Caddy 做了 HTTPS 反向代理，这样就可以通过我自己的域名访问了：mqtt.140105.xyz

> 如果你没做 Caddy 代理，直接访问宿主机的 http://localhost:18083 也是一样的；

默认账号密码是：
- 用户名：admin
- 密码：public

首次登录建议立即修改密码，避免风险；

## 客户端认证设置

在 EMQX 中，可以通过**密码认证**来控制哪些客户端可以连接；

操作路径：
Dashboard 面板 -> **访问控制** -> **客户端认证**

配置流程如下：
- 新建一个认证方式：
  - 类型选择：Password-Based
  - 认证数据源：内置数据库
  - 用户名类型：username
  - 加密算法、加盐策略使用默认配置即可

- 添加用户账号：
  - 用户名：emtime
  - 密码：你自己的常用密码

## 客户端连接测试

### 桌面客户端推荐

建议使用官方出品的 MQTT 客户端：[MQTTX](https://mqttx.app/zh)

配置方式如下：
- 地址填写你自己的域名或本机地址，如：mqtt.140105.xyz
- 端口填写 1883（TCP MQTT）
- 用户名 / 密码 填写你刚才设置的认证信息

点击连接后，右上角绿色表示连接成功，就可以自己进行订阅 / 发布测试了；

### Web 客户端（可选）

你也可以用官方提供的 Web 客户端工具：https://mqttx.app/web-client

不过我个人测试的时候没连上，可能是网络或证书的问题，也可以使用 Dashboard 自带的测试功能尝试；

## 总结

这套部署流程非常适合个人或小型项目使用，Docker 启动简单、配置界面直观，而且支持多种认证机制，非常灵活；后续如果你有更复杂的使用场景（如 TLS 加密、集群部署、ACL 访问控制等），EMQX 都能胜任；
