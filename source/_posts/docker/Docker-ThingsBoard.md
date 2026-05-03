---
title: ThingsBoard Docker 部署指南
categories: docker
date: 2025-06-21 15:48:24
# updated: 2025-06-21 15:48:24
---

官网：https://thingsboard.io
参考：http://www.ithingsboard.com/docs/user-guide/attributes

应**朋友**的要求，写一篇关于 ThingsBoard 的文章，记录一下安装与使用的过程；

近年来，**物联网平台（IoT Platform）** 在智慧农业、工业监测、智能城市等应用中扮演着越来越关键的角色；随着传感器网络的普及，单一设备的数据采集和本地显示早已无法满足需求，**集中式的远程管理、实时监控与可视化平台** 成为构建现代物联网系统的重要组成部分；

在众多开源物联网平台中，**ThingsBoard 凭借插件机制灵活、支持多种协议（如 MQTT、HTTP、CoAP），以及强大的仪表盘功能**，成为中小型物联网项目中极具性价比的选择；

如果你正在寻找一个稳定易用的物联网平台，并满足以下需求：
- 需要远程监控嵌入式/传感器设备；
- 希望快速搭建一个支持 MQTT、HTTP、CoAP 的平台；
- 苦于其他平台太重或收费昂贵……

那么，**ThingsBoard** 是你值得一试的开源解决方案；

本系列将基于实操视角，逐步记录我从零开始搭建 ThingsBoard 环境、连接设备、上传数据并实现可视化展示的过程；涉及内容包括但不限于：
- ThingsBoard 的快速部署（Docker 方式）；
- 基于 MQTT 协议的数据上报与遥控；
- Dashboard 可视化配置；
- 与嵌入式终端（如 STM32、ESP32、树莓派等）的对接实践；

## 使用 Docker 部署 ThingsBoard

ThingsBoard 提供了多种安装方式，包括 Docker、Kubernetes、本地安装包等；考虑到 ThingsBoard 的 Docker 镜像已经非常完善，且易于部署，本文将采用 Docker 方式进行安装；

### 环境依赖
在开始安装之前，请确保你的系统满足以下环境依赖：

- **Docker**：ThingsBoard 需要 Docker 来运行；请确保你的系统已经安装了 Docker；如果没有安装，可以参考 Docker 官方文档进行安装；

- **Docker Compose**：ThingsBoard 使用 Docker Compose 来管理多个容器；请确保你的系统已经安装了 Docker Compose；如果没有安装，可以参考 Docker Compose 官方文档进行安装；

### Docker compose 配置文件

进入某一个目录（你自己定义，最好和其他任何软件不产生交集），编辑 docker-compose.yml 文件：

```bash
nano docker-compose.yml
```

在文件中添加：

```yaml
services:
  postgres:
    restart: always
    image: "postgres:16"
    ports:
      - "5432"
    environment:
      POSTGRES_DB: thingsboard
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres-data:/var/lib/postgresql/data
  thingsboard-ce:
    restart: always
    image: "thingsboard/tb-node:4.0.1.1"
    ports:
      - "8080:8080"
      - "7070:7070"
      - "1883:1883"
      - "8883:8883"
      - "5683-5688:5683-5688/udp"
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"
    environment:
      TB_SERVICE_ID: tb-ce-node
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/thingsboard
    depends_on:
      - postgres

volumes:
  postgres-data:
    name: tb-postgres-data
    driver: local
```

**服务说明**：
- postgres ：用于存储 ThingsBoard 的数据；
  - image: postgres:16：使用官方维护的 PostgreSQL 16 镜像，稳定可靠；
  - ports: "5432"：数据库默认端口，未映射到宿主机，确保安全（可按需映射，可以通过 Docker 创建的网络使用容器名访问）；
  - environment：配置 PostgreSQL 的数据库信息；
    - POSTGRES_DB=thingsboard：初始化时创建名为 thingsboard 的数据库；
    - POSTGRES_PASSWORD=postgres：数据库密码为 postgres，可自行修改；
  - volumes：将数据库数据挂载到宿主机上的 tb-postgres-data 卷中，实现数据持久化，容器重启不会丢失数据（但不会直接在本地映射出来），建议定期备份 tb-postgres-data，以便灾难恢复；

- thingsboard-ce：ThingsBoard 的核心服务；
  - image: thingsboard/tb-node:4.0.1.1：指定 ThingsBoard CE 版本为 4.0.1.1，可根据需求更新为最新版本；
  - ports：映射到宿主机的端口，如下：
    - 8080:8080：Web UI 访问端口（默认使用 http://localhost:8080 打开平台）
    - 7070:7070：用于 WebSocket 和服务间通信
    - 1883:1883：MQTT 协议端口（设备上报数据）
    - 8883:8883：MQTT over TLS 加密端口
    - 5683-5688:5683-5688/udp：CoAP 协议端口（适用于轻量设备）
  - logging：限制日志大小防止磁盘爆满，每个日志文件最大 100MB，最多保留 10 个文件；
  - environment：配置 ThingsBoard 的环境变量；
    - TB_SERVICE_ID=tb-ce-node：设置服务 ID，适用于集群或多节点部署；
    - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/thingsboard：指定数据库连接字符串，注意这里的 postgres 为 Compose 内部服务名（自动 DNS），无需写成 IP 地址；
  - depends_on: postgres：确保在启动 thingsboard-ce 之前，postgres 服务已经启动；

- volumes：数据持久化，用于保存 PostgreSQL 数据的宿主机挂载卷，避免容器重启后数据丢失；

> **注意**：端口配置中，冒号前面的是宿主机端口，后面的端口是容器内端口；如果需要修改端口，注意只能修改宿主机端口，容器内端口不能修改；
> 若设备运行在非 Docker 主机，请确保防火墙/路由规则允许连接映射到的 MQTT/CoAP/Web 端口；

### 初始化数据库与加载系统资源

在首次启动 ThingsBoard 服务之前，我们需要先 **初始化数据库的表结构** 并 **加载系统内置资源**；这一步非常关键，否则平台无法正常运行；

ThingsBoard 官方镜像已经内置了初始化命令，可以通过一条 Docker Compose 命令完成：

```bash
docker compose run --rm \
  -e INSTALL_TB=true \
  -e LOAD_DEMO=true \
  thingsboard-ce
```

该命令会：
- 启动一个临时的 thingsboard-ce 容器；
- 执行内置的数据库初始化流程；
- 结束后自动退出并删除临时容器（因为加了 --rm 参数）；

## 启动与登录平台

### 启动平台

完成数据库初始化后，我们即可正式启动 ThingsBoard 平台服务；

使用以下命令启动所有容器，并实时查看 ThingsBoard 的运行日志：

```bash
docker compose up -d && docker compose logs -f thingsboard-ce
```

这条命令做了两件事：
- docker compose up -d：在后台启动所有定义在 docker-compose.yml 中的服务（包括 PostgreSQL 和 ThingsBoard 容器）；
- docker compose logs -f thingsboard-ce：持续输出 thingsboard-ce 容器的运行日志，方便我们第一时间看到启动进度和是否成功；

> 如果你看到 Started ThingsBoardServerApplication 或类似提示，说明平台已经成功启动；
> 如果不想看日志，可以按下 Ctrl C 退出日志输出（容器本身不会停止，ThingsBoard 服务仍然会在后台运行）；
> 如果你想重新查看运行日志，可以随时使用 docker compose logs -f thingsboard-ce；

### 登录平台

默认情况下，ThingsBoard 会监听本地的 8080 端口；你可以在浏览器中访问：

```txt
http://localhost:8080
```

或者在非本地部署时，替换为你的主机 IP 地址：

```txt
http://{your-host-ip}:8080
```

打开网页后，即可看到 ThingsBoard 的登录界面；

首次登录系统，你可以使用以下默认账户：

| 角色 | 用户名 | 密码 |
| ---- | ----- | ---- |
| 系统管理员 | 	sysadmin@thingsboard.org | sysadmin |
| 租户管理员 | 	tenant@thingsboard.org | tenant |
| 客户用户（演示） | 	customer@thingsboard.org | customer |

> 建议登录后尽快修改各个账户的密码，以保证平台安全；

## ThingsBoard 使用

### 添加设备

其实我们第一步应该是创建租户和租户管理员账号的，但是平台默认提供了三个账号，我们直接使用即可；

登录租户管理员账号后，点击左侧导航栏的 **设备** 选项，进入设备管理页面即可添加设备；

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/21/685691260679f.png"/>
</center>

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/21/685692da81ca1.png"/>
</center>

可选择 default 或 thermostat 模板，这些模板预配置了 MQTT、HTTP 与 CoAP 通信协议，便于多种方式测试连通性（在我当前使用的版本中，MQTT 功能运行最稳定，CoAP反而无法正常使用）；

分配给客户，我选择了公开，这个决定了用户账户可以查看的设备；

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/21/68569408ddfbd.png"/>
</center>

我们设置 MQTT 凭据，要注意，每一个设备的凭据不能相同，凭据是设备与平台通信的唯一标识；

### 测试设备

#### 连接平台

我们可以使用客户端，或者 Linux 命令行工具，来测试设备与平台之间的连通性，这里我使用 MQTTX 工具进行测试；

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/22/6857f09cde319.png"/>
</center>

选择你运行的 Docker ThingsBoard 的机器地址，端口选择 Docker 配置映射出来的端口（默认是 1883），填入在 ThingsBoard 中设备的凭据，然后点击连接即可连接到平台；

#### 订阅、发布与平台设备配置

在说订阅与发布之前，我们先来看一下 ThingsBoard 的属性，ThingsBoard 分为客户端属性、服务端属性和最新属性值，如下：

| 属性类型 | 来源方向 | 设备是否可写 | 平台是否可写 | 是否自动推送到设备 | 描述 |
|--------------|--------------|----------------|----------------|-----------------------|------|
| 客户端属性 | 设备 → 平台 | ✅ 是 | ✅ 可初始化（仅首次） | ❌ 否 | 设备主动上报的信息，例如设备型号、固件版本、运行状态，平台只做查看，不会主动推送 |
| 服务端属性 | 平台 → 平台侧应用 | ❌ 否 | ✅ 是 | ❌ 否 | 平台保存的属性集合，设备无法获取 |
| 共享属性 | 平台 → 设备 | ❌ 否 | ✅ 是 | ✅ 是 | 平台设置的共享配置项，支持平台→设备自动推送，也可由设备主动请求（如目标温度、模式等） |
| 最新属性值（概念） | 双向 | ✅ 是 | ✅ 是 | ❌ 否（需主动查询） | 并非单独属性类型，而是平台维护的当前属性快照，用于展示或通过 API 获取 |

在平台点击设备，进入设备详情页面，可以看到设备的属性，如下：

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/23/68583b1340fff.png"/>
</center>

接下来我就根据上面的表格，来演示一下 ThingsBoard 的属性与设备之间的交互；

##### 服务端属性

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/23/6858fdcfae2a3.png"/>
</center>

服务端属性通常是由平台维护、设备通过请求获取的一类静态或配置属性；它常用于存储平台侧记录的设备参数，例如部署位置、设备负责人、安装时间等；

- 平台侧应用获取服务端属性

因为服务端属性仅能通过平台侧应用进行读写，所以无法使用 MQTT 订阅的方式获取，只能通过 API 获取，如下：

1. 获取 JWT 令牌

```bash
curl -X POST https://iot.140105.xyz/api/auth/login   -H "Content-Type: application/json"   -d '{"username":"tenant@thingsboard.org","password":"tenant"}' 
```

以我的平台为例，将地址替换为你的平台地址，用户名和密码替换为你的租户管理员账号和密码，即可获取 JWT 令牌，返回示例如下（返回的信息比较长，注意仔细比对）：

```json
{
    "token": "xxxxx",
    "refreshToken": "xxxxx"
}
```

2. 获取设备服务端属性

使用上一步获得的 JWT 令牌，通过下面的方法获取设备的服务端属性：

```bash
curl -X GET 'https://iot.140105.xyz/api/plugins/telemetry/DEVICE/4d453110-4e91-11f0-aa54-d1bcfb4dde7b/values/attributes/SERVER_SCOPE'   -H 'Content-Type: application/json'   -H 'X-Authorization: Bearer JWT 令牌'
```

其中 DEVICE 后面的 4d453110-4e91-11f0-aa54-d1bcfb4dde7b 是设备的 ID，在平台网页点击设备，网址链接中可以看到，JWT 令牌就是上一步获取的 token，返回结果如下；

```json
[
    {
        "lastUpdateTs": 1750619760822,
        "key": "lastConnectTime",
        "value": 1750619760822
    },
    {
        "lastUpdateTs": 1750619852183,
        "key": "test",
        "value": "1"
    },
    {
        "lastUpdateTs": 1750621040988,
        "key": "lastDisconnectTime",
        "value": 1750621040987
    },
    {
        "lastUpdateTs": 1750621041491,
        "key": "lastActivityTime",
        "value": 1750621040963
    },
    {
        "lastUpdateTs": 1750621684069,
        "key": "inactivityAlarmTime",
        "value": 1750621684069
    },
    {
        "lastUpdateTs": 1750621684069,
        "key": "active",
        "value": false
    }
]
```

> **注意**：token 是有有效期的，如果过期了，需要重新获取；
> 以上的演示方式，均在命令行中完成，可以使用其他编程语言，如 Python、Java、Go 等作为 curl 的替代；

- 平台侧应用设置服务端属性

我们同样使用 JWT 令牌，通过下面的方法设置设备的服务端属性：‘

```bash
curl -v 'https://iot.140105.xyz/api/plugins/telemetry/DEVICE/4d453110-4e91-11f0-aa54-d1bcfb4dde7b/SERVER_SCOPE'   -H 'Content-Type: application/json'   -H 'X-Authorization: Bearer JWT 令牌' \
-H 'content-type: application/json' \
--data-raw '{"test":"2"}'
```

其中最后一行的 test 是你要设置的属性键名，2 是你要设置的属性值，运行之后平台对应设备的服务端属性就会发生变化；

##### 客户端属性

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/23/6858fa649f16b.png"/>
</center>

客户端属性常用于平台的分组管理、版本控制、规则引擎判断等场景，是设备与平台之间状态同步的基础结构之一；

- 设备发送数据到平台

客户端属性的主题是 v1/devices/me/attributes，通过表格我们能看到，这个是由设备发送到平台的，所以我们使用 MQTTX 工具模拟设备，向平台发布一条消息，如下：

```json
{
  "serialNumber": "SN-001-ABC",
  "firmwareVersion": "v1.2.5",
  "model": "X200"
}
```

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/23/68583efd769a6.png"/>
</center>

平台可以根据设备上报的属性，在仪表盘中展示设备信息，也可以在规则链库中根据属性值进行判断，如下：

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/23/6858414f20529.png"/>
</center>

如上图即为在仪表盘中显示设备信息；

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/23/68584392e86b5.png"/>
</center>

如上图即为设置规则链库，当版本信息小于脚本中所配置的版本时，便会触发提醒；

- 设备从平台获取数据

设备可以主动请求平台侧的客户端属性，请求主题为 v1/devices/me/attributes/request/&lt;request_id&gt;，其中 &lt;request_id&gt; 是请求 ID，可以任意指定，订阅主题可以是 v1/devices/me/attributes/request/&lt;request_id&gt; 或者 v1/devices/me/attributes/response/+，发送的数据如下：

```json
{
  "clientKeys": "firmwareVersion,serialNumber"
}
```

其中， firmwareVersion 和 serialNumber 是设备在 ThingsBoard 中配置的客户端属性；

在主题 v1/devices/me/attributes/request/1 或者 v1/devices/me/attributes/response/+ 中会得到从平台返回的数据，如下：

```json
{
  "client": {
    "serialNumber": "SN-001-ABC",
    "firmwareVersion": "v1.2.5"
  }
}
```

对 json 进行解析，即可得到平台返回的属性值；

> 在 ThingsBoard 中，每个设备都有唯一的连接凭据，因此平台能够识别出请求来源的具体设备；即使多个设备同时使用相同的 request_id 发起属性请求，平台也能正确分发并返回各自的响应结果，不会互相影响；
> 但如果你使用一个设备代理器，代理器会使用相同的凭据，那么平台会认为所有的请求都来自同一个设备，导致响应混乱，这时候就需要你对代理的设备进行额外的代码编写，给每个被代理的设备分配不同的请求 ID，实现设备之间的请求隔离；

- 平台侧应用获取数据

平台侧可以和设备一样使用 MQTT 获取数据，也可以使用 API 获取数据，如下：

```bash
curl -X GET 'https://iot.140105.xyz/api/plugins/telemetry/DEVICE/4d453110-4e91-11f0-aa54-d1bcfb4dde7b/values/attributes/CLIENT_SCOPE'   -H 'Content-Type: application/json'   -H 'X-Authorization: Bearer JWT 令牌'
```

和服务端属性获取的方式大同小异，只是将 SERVER_SCOPE 替换为 CLIENT_SCOPE 即可，运行之后结果如下：

```json
[
    {
        "lastUpdateTs": 1750613652969,
        "key": "serialNumber",
        "value": "SN-001-ABC"
    },
    {
        "lastUpdateTs": 1750613652969,
        "key": "model",
        "value": "X200"
    },
    {
        "lastUpdateTs": 1750613652969,
        "key": "firmwareVersion",
        "value": "v1.2.5"
    }
]
```

##### 共享属性

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/23/68593b5ac0ca6.png"/>
</center>

共享属性是平台侧应用给设备分发数据的一种方式，共享属性可以用于平台侧应用之间的数据共享（也可以用于平台向设备发送数据）；

> 和客户端属性的区别是，客户端属性是设备主动请求平台返回数据，而共享属性是当属性发生变化或者新建属性时候，平台会主动向设备发送数据（当然，也可以主动请求）；

- 设备从平台请求数据

和客户端属性一样，设备可以主动请求平台侧的共享属性，请求主题为 v1/devices/me/attributes/request/&lt;request_id&gt;，其中 &lt;request_id&gt; 是请求 ID，可以任意指定，订阅主题可以是 v1/devices/me/attributes/request/&lt;request_id&gt; 或者 v1/devices/me/attributes/response/+，发送的数据如下：

```json
{
  "sharedKeys": "test"
}
```

其中， test 是你要获取的属性键名；

在主题 v1/devices/me/attributes/request/&lt;request_id&gt; 或者 v1/devices/me/attributes/response/+ 中会得到从平台返回的数据，如下：

```json
{
  "shared": {
    "test": "1"
  }
}
```

对 json 进行解析，即可得到平台返回的共享属性值；

- 属性修改/变动自动推送

无论在平台侧应用，还是在平台本身，我们对共享属性进行增、删、改操作，设备都会收到平台**自动**推送的属性值，如下：

1. 增和改

```json
{
  "test": "2"
}
```

2. 删

```json
{
  "deleted": [
    "test"
  ]
}
```

- 平台侧应用获取数据

平台侧应用获取共享属性的方式和服务端属性大同小异，只是将 SERVER_SCOPE 替换为 SHARED_SCOPE 即可，如下所示,在此不过多赘述：

```bash
curl -X GET 'https://iot.140105.xyz/api/plugins/telemetry/DEVICE/4d453110-4e91-11f0-aa54-d1bcfb4dde7b/values/attributes/SHARED_SCOPE'   -H 'Content-Type: application/json'   -H 'X-Authorization: Bearer JWT 令牌'
```

- 平台侧应用设置数据

平台侧应用设置共享属性的方式和服务端属性大同小异，只是将 SERVER_SCOPE 替换为 SHARED_SCOPE 即可，如下所示,在此不过多赘述：

```bash
curl -v 'https://iot.140105.xyz/api/plugins/telemetry/DEVICE/4d453110-4e91-11f0-aa54-d1bcfb4dde7b/SHARED_SCOPE'   -H 'Content-Type: application/json'   -H 'X-Authorization: Bearer JWT 令牌' \
-H 'content-type: application/json' \
--data-raw '{"test":"2"}'
```

> 用此方法进行数据的更新，设备也会自动收到平台推送的数据；
> 如果设备离线，可能会错过重要的数据更新，所以建议在设备启动之后订阅共享属性，并且每次连接后请求属性的最新值；

> 除了属性，ThingsBoard 还提供了遥测（Telemetry）数据，和属性的区别是遥测多了时间序列相关的功能，可以用于长时间的数据对比分析等，展开的话内容太多，在此不过多赘述，可以在这里查看：[遥测数据](http://www.ithingsboard.com/docs/user-guide/telemetry)；

## 总结

本文记录了从零开始使用 Docker 快速部署 ThingsBoard 平台的全过程，并详细演示了平台与设备之间通过 MQTT 协议进行数据交互的方式，特别是客户端属性、服务端属性、共享属性三者的使用与差异；借助 ThingsBoard 丰富的功能，我们可以轻松实现设备的远程配置、状态监测与规则联动，为物联网系统的构建打下坚实基础；

如果你正在开发或运营一个嵌入式项目，或正在寻找一套支持 MQTT/HTTP/CoAP 且功能全面的物联网平台，相信 ThingsBoard 能为你提供一个清晰的起点；

后续我将继续记录更多实战经验，包括遥测数据上传与可视化、规则链自动化处理、集成邮件/钉钉/短信报警机制等内容，欢迎持续关注；
