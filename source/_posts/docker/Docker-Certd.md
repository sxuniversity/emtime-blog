---
title: Docker部署Certd，实现HTTPS证书自动申请与更新
categories: docker
date: 2024-08-19 22:19:47
updated: 2024-08-19 22:19:47
---

Certd官网：https://certd.docmirror.cn

在现代网站服务中，HTTPS 已成为保障通信安全的基础组件；为了简化证书申请与更新流程，Certd 提供了自动化的证书管理能力，并支持多家主流证书颁发机构；

本文将详细介绍如何通过 Docker 快速部署 Certd，实现 HTTPS 证书的自动申请与更新；

## 安装并启动服务

```bash
# 创建用来存放Certd相关的
mkdir certd

cd certd

wget https://gitee.com/certd/certd/raw/v2/docker/run/docker-compose.yaml

nano docker-compose.yaml

docker conmpose up -d

# 停止并删除容器，不用怕数据，数据已经挂载到宿主机了
#docker compose down
```

我自己的docker-compose.yaml文件如下：

```yaml
version: '3.3' # 兼容旧版docker-compose
services:
  certd:
    # 镜像                                                  #  ↓↓↓↓↓ ---- 镜像版本号，建议改成固定版本号,例如：certd:1.29.0
    image: registry.cn-shenzhen.aliyuncs.com/handsfree/certd:latest
    container_name: certd # 容器名
    restart: always # 自动重启
    volumes:
      #   ↓↓↓↓↓ -------------------------------------------------------- 数据库以及证书存储路径,默认存在宿主机的/data/certd/目录下，【您需要定时备份此目录，以保障数据容灾】
      #                                                                  只要修改冒号前面的，冒号后面的/app/data不要动
      - ./:/app/data
    ports: # 端口映射
      #  ↓↓↓↓ ---------------------------------------------------------- 如果端口有冲突，可以修改第一个7001为其他不冲突的端口号，第二个7001不要动
      - "7001:7001"
      #  ↓↓↓↓ ---------------------------------------------------------- https端口，可以根据实际情况，是否暴露该端口
      - "7002:7002"
    #↓↓↓↓ -------------------------------------------------------------- 如果出现getaddrinfo ENOTFOUND错误，可以尝试设置dns
    dns:
      - 223.5.5.5
      - 223.6.6.6
    
#    dns:
#      - 223.5.5.5      # 阿里云公共dns
#      - 223.6.6.6
#       # ↓↓↓↓ --------------------------------------------------------- 如果你服务器在腾讯云，可以用这个替换上面阿里云的公共dns
#      - 119.29.29.29  # 腾讯云公共dns
#      - 182.254.116.116
#       # ↓↓↓↓ --------------------------------------------------------- 如果你服务器部署在国外，可以用这个替换上面阿里云的公共dns
#      - 8.8.8.8       # 谷歌公共dns
#      - 8.8.4.4
#    extra_hosts:
#        # ↓↓↓↓ -------------------------------------------------------- 这里可以配置自定义hosts，外网域名可以指向本地局域网ip地址
#      - "localdomain.com:192.168.1.3"
#        #         ↓↓↓↓ ------------------------------------------------ 直接使用主机的网络，如果网络问题实在找不到原因，可以尝试打开此参数
#    network_mode: host
    labels:
      com.centurylinklabs.watchtower.enable: "true"
#    ↓↓↓↓ -------------------------------------------------------------- 启用ipv6网络，还需要把下面networks的注释放开
#    networks:
#      - ip6net
    environment:
#     设置环境变量即可自定义certd配置
#     配置项见： packages/ui/certd-server/src/config/config.default.ts
#     配置规则： certd_ + 配置项, 点号用_代替
#                                    #↓↓↓↓ ----------------------------- 如果忘记管理员密码，可以设置为true，重启之后，管理员密码将改成123456，然后请及时修改回false
      - certd_system_resetAdminPasswd=false

#     默认使用sqlite文件数据库，如果需要使用其他数据库，请设置以下环境变量
#     注意： 选定使用一种数据库之后，不支持更换数据库；
#     数据库迁移方法：1、使用新数据库重新部署一套，然后将旧数据同步过去，注意flyway_history表的数据不要同步
#                                    #↓↓↓↓ ----------------------------- 使用postgresql数据库，需要提前创建数据库
#      - certd_flyway_scriptDir=./db/migration-pg                        # 升级脚本目录
#      - certd_typeorm_dataSource_default_type=postgres                  # 数据库类型
#      - certd_typeorm_dataSource_default_host=localhost                 # 数据库地址
#      - certd_typeorm_dataSource_default_port=5433                      # 数据库端口
#      - certd_typeorm_dataSource_default_username=postgres              # 用户名
#      - certd_typeorm_dataSource_default_password=yourpasswd            # 密码
#      - certd_typeorm_dataSource_default_database=certd                 # 数据库名

#                                    #↓↓↓↓ ----------------------------- 使用mysql数据库，需要提前创建数据库 charset=utf8mb4, collation=utf8mb4_bin
#      - certd_flyway_scriptDir=./db/migration-mysql                     # 升级脚本目录
#      - certd_typeorm_dataSource_default_type=mysql                     # 数据库类型， 或者 mariadb
#      - certd_typeorm_dataSource_default_host=localhost                 # 数据库地址
#      - certd_typeorm_dataSource_default_port=3306                      # 数据库端口
#      - certd_typeorm_dataSource_default_username=root                  # 用户名
#      - certd_typeorm_dataSource_default_password=yourpasswd            # 密码
#      - certd_typeorm_dataSource_default_database=certd                 # 数据库名

#         ↓↓↓↓ ---------------------------------------------------------  自动升级，上面certd的版本号要保持为latest
#  certd-updater:  # 添加 Watchtower 服务
#    image: containrrr/watchtower:latest
#    container_name: certd-updater
#    restart: unless-stopped
#    volumes:
#      - /var/run/docker.sock:/var/run/docker.sock
#    # 配置 自动更新
#    environment:
#      - WATCHTOWER_CLEANUP=true            # 自动清理旧版本容器
#      - WATCHTOWER_INCLUDE_STOPPED=false   # 不更新已停止的容器
#      - WATCHTOWER_LABEL_ENABLE=true       # 根据容器标签进行更新
#      - WATCHTOWER_POLL_INTERVAL=600       # 每 10 分钟检查一次更新


#    ↓↓↓↓ -------------------------------------------------------------- 启用ipv6网络，还需要把上面networks的注释放开
#networks:
#  ip6net:
#    enable_ipv6: true
#    ipam:
#      config:
#        - subnet: 2001:db8::/64
```

### 访问 Certd 面板

部署完成后，可通过以下地址访问 Certd：
- http://your_server_ip:7001
- https://your_server_ip:7002

```pgsql
默认账号密码：
admin/123456
```

首次登录后请务必修改管理员密码；

## 配置自动签发证书

Certd 支持多种 DNS API 接入方式（阿里云、腾讯云、Cloudflare等），根据你的域名服务商选择配置方法；详细配置方式可以参考官方文档的 DNS API 配置指南；本文以不需要 API 的 CNAME 方式、腾讯云与CloudFlare三个例子进行说明。

1. 在管理页面中点击左边的证书自动化流水线->创建证书流水线，其中域名你自己填写你的域名，同时支持通配符，比如你的域名是abc.abc，那么可以填写abc.abc与*.abc.abc，这样申请得到的证书可以匹配到你的主域名与所有的二级子域名，比较方便（但不代表安全，如果你有额外精力的话，还是推荐你不同的域名单独申请证书）。

2. 设置通知邮件，如果没有配置邮箱的话，需要先去配置一下，在管理页面中点击左边的设置->通知设置->添加，其中通知类型选择电子邮件，收件人邮箱填写自己的邮箱，如果没有配置邮件服务器则需要点击页面中的“配置邮件服务器”进行配置，具体配置方法可以看[官方文档](https://certd.docmirror.cn/guide/use/email/#qq%E9%82%AE%E7%AE%B1%E9%85%8D%E7%BD%AE)。

3. 域名验证方式，这一部分请看下面CNAME方式、腾讯云以及Cloudflare。

**CNAME方式**

- CNAME是最简单的方式，我们在`域名验证方式`中选择`CNAME代理验证`，在其下方的`域名验证配置`中，会出现对应的信息，我们需要关注的是`主机记录`与`请设置CNAME记录`对应的内容，我们将其复制下来，一会要用。
- 进入你的域名管理页面，创建新解析，类型选择CNAME，名称填写`主机记录`对应的内容，目标或者值填写`请设置CNAME记录`对应的内容，然后创建即可。
- 等待几分钟让DNS解析生效，然后回到certd的这个页面，点击右边的`点击验证`按钮，验证通过之后即可进行下一步操作.

**腾讯云**

- 腾讯云与Cloudflare使用的都是DNS直接验证方式，所以我们需要更改`域名验证方式`为DNS直接验证，`DNS解析服务商`选择腾讯云，然后点击`DNS授权解析`。
- 然后我们添加授权，名称你自己取，其中`secretId`和`secretKey`需要去腾讯云的[密钥管理](https://console.cloud.tencent.com/cam/capi)创建，我们新建密钥即可，但要注意secretKey只会**出现一次**，我们要把他保存下来填写到certd对应的地方。授权添加完成之后我们可以点击测试，如果返回正常的结果，那么就是创建成功了，我们点击确定保存即可。

**Cloudflare**

- 前面的操作和腾讯云一致，只是我们把服务商改为Cloudflare即可。
- 接下来进行令牌获取，访问[用户API令牌](https://dash.cloudflare.com/profile/api-tokens)页面，点击创建令牌->创建自定义令牌。
  - 权限部分点击添加更多，我们一共需要两个权限，第一个选择`区域-区域-编辑`，第二个是`区域-DNS-编辑`。
  - 区域资源可以默认，也可以自定义，比如改为`包括-特定区域-你的单个域名`。
- 然后我们一直下一步，直到创建好令牌即可，在下面的页面中，复制我们的令牌信息，填写到certd对应的位置中，测试并保存后，在授权中选择我们新建的授权即可。

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2026/05/05/69f9da13a33c1.png"/>
</center>

4. 证书颁发机构，现在国内的很多服务器，在经历了一些原因之后，都开始封禁非大陆的连接了，所以如果你的服务器在国内，我推荐使用litessl，反过来，则选择默认的Let's Encrypt即可。因为Let's Encrypt不需要授权，所以接下来主要说一下litessl。

- 进入[FreeSSL官网](https://freessl.cn/)，根据引导注册一个账号
- 进入[ACME账号管理](https://freessl.cn/automation/eab-manager)页面，点击新增EAB，然后取个名字，保存好这个信息，也是**只显示一次**的
- 然后在certd的EAB授权页面新增一个授权即可，内容填写上面步骤保存的信息

5. 然后我们点击确定即可创建好对应的证书流水线，我们点击手动触发即可进行证书的申请，如果失败了，可以点击证书申请任务，能看到日志，交给AI判断就好，按一般的经验来说，失败个两三次，然后就成功了。

6. 申请好证书，我们可以给caddy或者Nginx使用，在流水线证书详情中点击编辑，在证书申请阶段后面添加任务->添加步骤->搜索ssh，可以选择部署证书到SSH主机，证书格式一般选择pem/crt即可，主机登录配置，就是那几样，地址端口用户名密码或者密钥，在此不过多赘述，值得说明的是`后置命令`，我们可以执行比如systemctl restart caddy这样的命令，重启需要证书才能运行的服务，这样在自动化流程中，就不需要我们去操心了。
