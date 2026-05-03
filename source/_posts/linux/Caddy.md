---
title: Caddy 反向代理
categories: linux

date: 2024-08-19 22:19:03
updated: 2024-08-19 22:19:03
---

Caddy官网：https://caddyserver.com

## 为什么我开始用 Caddy

最初接触 Caddy，是因为它在社区中被广泛推荐为一款“自动 HTTPS 一把梭”的轻量级 Web 服务器；只要你有个域名、端口没被封，Caddy 理论上能自动申请并续签 HTTPS 证书，全程无需手动操作，真正实现开箱即用；

不过，**实话实说，我并没能成功实现自动 HTTPS**；
我这台小水管部署在没有公网 IP 的局域网环境中，Let's Encrypt 验证域名所有权时无法访问到我的服务器，自然无法成功签发证书；翻了一些文档后确认，这种内网场景确实不适合自动签证；

但我使用 Caddy 的**真正目的其实并不是自动 HTTPS**，而是简洁地实现反向代理；我的需求是把一些局域网服务（比如 Web 控制面板、API 服务、博客页面）通过统一的域名访问，而不是记一堆不同端口；

相比 Nginx 那一堆配置、手动写 location、reload 重载，**Caddy 用 Caddyfile 几行就搞定**，还能自动帮我处理 header、压缩、缓存等常用设置；
虽然 HTTPS 自动签发失败，但 Caddy 依然凭借其优秀的反向代理能力和极简配置体验让我留下它；

## 安装和基本使用

### 安装 Caddy（推荐官方一键脚本）

```bash
curl -fsSL https://get.caddyserver.com | bash -s personal
```

或者使用包管理器，如在 Debian/Ubuntu 下：
```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/caddy.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy.list
sudo apt update
sudo apt install caddy
```

### 编写配置文件

默认配置文件路径为 /etc/caddy/Caddyfile，也可以自己创建一个 Caddyfile 并通过命令指定；

```bash
sudo nano /etc/caddy/Caddyfile
```

### 启动 Caddy

编辑完配置后，使用 systemd 启动 Caddy：

```bash
sudo systemctl restart caddy
sudo systemctl enable caddy
```

或者直接在本地调试运行：

```bash
caddy run --config /path/to/Caddyfile
```

## 我的 Caddy 配置详解

### 配置监听端口（http_port）

```bash
{
	http_port 8080
}
```

因为 80 端口经常被占用（尤其内网小主机上），我将默认 HTTP 监听端口改为了 8080，更灵活些；

### 反代局域网里的服务

```bash
你的域名 {
    # 你的域名证书
	tls cert.crt cert.key

    # 本地服务
	reverse_proxy 127.0.0.1:34718
}
```

这个配置用于访问运行在本机 34718 端口的服务；

域名解析的是我在虚拟组网中的主机地址，因此组网内的设备都能通过域名来访问该服务；

这里的证书我是使用Certd申请的，详情见：[Docker部署Certd，实现HTTPS证书自动申请与更新](/2024/08/19/docker/Docker-Certd)；

### 反代局域网中的https服务

```bash
你的域名 {
    reverse_proxy https://192.168.0.1:1234 {
        transport http {
            # 跳过 TLS 验证，避免证书错误导致Caddy反代失败
            tls_insecure_skip_verify
        }
    }

    # 你的证书
    tls cert.crt cert.key
}
```

这里反向代理的是别的设备的https服务；

Caddy 默认验证目标服务的 TLS 证书，由于我这里是自签的，就使用 tls_insecure_skip_verify 跳过了验证，避免反代失败；

### 本地静态页面（端口访问）

```bash
:21824 {
	tls emtime.crt emtime.key
	root * /home/time/doc/blog/public
	file_server

	handle_errors {
		@404 {
			expression {http.error.status_code} == 404
		}
		rewrite @404 /404.html
		file_server
	}
}
```

这是一个纯静态服务配置，我用来部署 Hexo 生成的博客页面；

Caddy 会监听 21824 端口，并将 /home/time/doc/blog/public 目录作为静态资源根目录；配置中还加了自定义的 404.html 页面处理逻辑；

因为没有公网 IP，这个页面更多是在局域网访问时使用；

### 本地静态页面（绑定域名）

```bash
blog.emtime.top {
	tls emtime.crt emtime.key
	root * /home/time/doc/blog/public
	file_server

	handle_errors {
		@404 {
			expression {http.error.status_code} == 404
		}
		rewrite @404 /404.html
		file_server
	}
}
```

这个配置和上面几乎一致，只是绑定了 blog.emtime.top 这个域名，可能正是**你现在正在访问的这个博客**的域名（我还有别的域名也同样部署了同样的博客）；

> 因为我的小水管没有公网 IP，这个域名是通过 frp 穿透暴露出来的；
> 这里简单说一下，我使用的是frp的方法，创建HTTPS隧道，选择本机的443端口进行映射，域名通过cname的方法解析到frp的域名即可；

### 为什么我最终选择了 Caddy 而不是 Nginx？

其实我也试过 Nginx，作为一个功能成熟的老牌服务器，它确实非常灵活；但对我这种轻量化内网部署场景来说，Caddy 的优势太明显了：
- Nginx 配置冗长，一写就是几十行；
- Caddy 的配置语言天然支持反向代理，几行就能完成；
- Caddy 默认自带 TLS（即使我没成功用自动签发）；
- Caddy 自带静态文件服务、自动压缩、常用 header，非常适合个人小服务；
- 部署只需要一个二进制，不依赖太多系统组件，非常适合局域网小主机部署；

虽然 Caddy 的自动 HTTPS 功能在我的场景下并没有成功使用，但它的反向代理能力和极低的学习成本，已经让我在日常部署中离不开它了；

## 总结

主要就是说了说我如何在没有公网的局域网环境下使用 Caddy 来构建统一访问入口，反代各种服务、博客页面和管理面板；

如果你也有类似的需求，不妨试试 Caddy，即使不能一把梭 HTTPS，它依然是一个非常轻便好用的反代工具；
