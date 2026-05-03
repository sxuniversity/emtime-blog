---
title: Docker部署密码管理工具
categories: docker
date: 2024-08-18 23:24:31
updated: 2024-08-20 17:49:01
---

参考：https://ohdmire.github.io/posts/bitwarden
仓库：https://github.com/dani-garcia/vaultwarden

在数字化生活中，密码管理已经不再是“记几个数字”那么简单；各种网站、应用动辄要求复杂度极高的密码，安全意识一提升，每个账户还不能用同一个密码；这时候，一个好用的密码管理工具就显得尤为重要；

Bitwarden 是一款非常优秀的开源密码管理器，而 Vaultwarden 则是它的轻量级“克隆版”——资源占用小，部署简单，并且还有一些特有的功能，非常适合自建；更重要的是，自部署意味着你的数据真正掌握在你自己手里；

这篇文章会带你一步步部署 Vaultwarden，构建一个属于你自己的私有密码管理系统，不依赖任何第三方服务，无需高性能服务器，即便是树莓派也能轻松运行；适合注重隐私、安全、掌控感的你；

## 几个关键点先说清楚：
- **Docker 镜像不是官方的**：使用的是 vaultwarden/server，优点是轻量、解锁 Bitwarden 高级功能，适合个人部署使用；
- **必须使用 HTTPS**：Vaultwarden 不支持在 HTTP 下登录或注册，所以部署过程必须搭配 HTTPS 服务，最好申请证书；

## Docker 部署过程

部署本体其实非常简单，以下是完整命令流程：

```bash
# 拉取镜像
docker pull vaultwarden/server:latest

# 准备本地挂载目录，保存 Vaultwarden 的 SQLite 数据
mkdir -p /home/time/doc/codefile/password

# 启动容器
docker run -d \
  --name password \
  -v /home/time/doc/codefile/password/:/data/ \
  --restart unless-stopped \
  -p 6468:80 \
  vaultwarden/server:latest
```

这个服务会监听在本地的 6468 端口，**页面是能打开的，但不能登录或注册**，因为没有 HTTPS；

## HTTPS 配置说明

Vaultwarden 的前端 强制要求 HTTPS 访问，这是很多人部署时最容易卡壳的地方；我这边采用了 Caddy 来做反向代理，部署逻辑简单、证书配置灵活，比较适合个人自部署的场景；

- Caddy反向代理：我使用的是 Caddy，配置非常简洁；具体原理和部署过程可以参考我之前写的这篇博客：[Caddy反向代理配置详解](/2024/08/19/linux/Caddy)；

- Caddy 配置示例：

```caddyfile
你的域名 {
    # 你的域名证书
	tls cert.crt cert.key

    # 反向代理到 Vaultwarden 容器所在端口
	reverse_proxy 127.0.0.1:6468
}
```

- 证书来源：我没有使用 Caddy 自带的自动证书申请功能，而是通过 [Certd](/2024/08/19/docker/Docker-Certd) 生成了泛域名证书，统一管理更方便；证书配置好后，Caddy 只需要加载即可；

## 密码导入和使用体验

Vaultwarden 兼容 Bitwarden 全套客户端，包括桌面应用、网页端和浏览器插件；首次访问你部署的网页服务后，注册一个账号并登录即可使用；

以下是浏览器插件界面截图：

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/14/684c6e6229c19.png"/>
</center>

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/14/684c6e815e796.png"/>
</center>

初次使用时，我直接将 Chrome 导出的 .csv 密码文件导入进去，整个流程没有踩坑，页面提示也很直观；导入成功后，插件会识别你常用的网站并提供填充密码功能，体验类似 Chrome 自带的密码管理，多了主密码保护等功能；

## 总结

总的来说，Vaultwarden 是一个轻量、实用、易于自建的密码管理工具，适合追求数据自主和隐私保护的用户；虽然它在易用性和云同步体验上不如 1Password 或 Chrome 自带的密码管理器那么“顺手”，但它胜在部署自由、功能完整，尤其对于技术用户来说，掌控自己的数据就是最大的安心；

如果你已经有了自己的服务器资源（哪怕是树莓派），并具备基本的 Docker 和 HTTPS 配置能力，Vaultwarden 完全值得一试；配合 Caddy 做反向代理，搭配 Certd 管理泛域名证书，部署流程也没有想象中复杂；

也许它不够“傻瓜式”，但作为一名稍微动手能力强一点的用户，这种“折腾”背后的安全感和成就感，是任何云服务都给不了的；

如果你和我一样，厌倦了各种服务把账号密码握在手里，不妨给 Vaultwarden 一个机会；
