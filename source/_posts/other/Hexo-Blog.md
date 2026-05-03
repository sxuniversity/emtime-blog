---
title: 使用Hexo搭建个人博客
categories: other

date: 2025-02-22 09:47:18
updated: 2025-02-22 09:47:18
---

在信息过载的时代，拥有一个属于自己的博客，不仅能整理思路、沉淀经验，还能在需要时快速复用曾经解决过的问题；而 Hexo，作为一款基于 Node.js 的轻量静态博客框架，以其部署简单、主题丰富、生成速度快的优势，成为了许多开发者的首选；

我最开始只是想记录一下技术笔记，后来发现 Markdown 写作 + Hexo 渲染这个组合用起来非常舒服：格式统一、可定制性强、部署自由（可以部署在 GitHub Pages、VPS、自建服务器等）；于是就干脆一步步搭建了一个完整的 Hexo 博客站点；

这篇文章就是整个过程的完整记录：从搭建环境、初始化项目、使用主题，到部署上线、开启 HTTPS、绑定域名等等，涵盖我实战中遇到的所有关键环节；希望能帮到你，也为自己留个备忘；

## 环境准备（Ubuntu 系统）

在正式搭建 Hexo 博客之前，需要先安装一些基础环境工具；由于我使用了 **FRP 实现内网穿透部署 Hexo**，所以本篇不涉及 GitHub Pages 或 Git 相关内容，仅保留最必要的部分；

### 安装 Node.js 和 npm

Ubuntu 默认的软件源中 Node.js 版本可能比较老，推荐使用官方的 NodeSource 源进行安装；这里以 Node.js LTS 版本（长期支持版）为例：

```bash
# 安装 Node.js LTS 版本（包含 npm）
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
```

验证安装是否成功：

```bash
node -v    # 输出版本号表示安装成功
npm -v
```

### 安装 Hexo 命令行工具

```bash
sudo npm install -g hexo-cli
```

验证安装是否成功：

```bash
hexo -v    # 输出版本号表示安装成功
```

### 可选：Git（非必须）

如果你后续打算将博客托管到 GitHub Pages，或使用 Git 做版本管理，也可以顺手安装 Git：

```bash
sudo apt install -y git
```

但如果你和我一样是用 FRP 暴露本地服务到公网，那完全可以跳过这一步；

##  Hexo 博客项目初始化

在完成环境安装后，就可以开始初始化 Hexo 博客项目了；Hexo 会在指定目录生成整个博客所需的目录结构，包括文章源文件、配置文件和静态资源目录等；

### 创建博客项目文件夹

选择一个你喜欢的目录作为博客的根目录，例如：

```bash
mkdir -p ~/my_blog
cd ~/my_blog
```

### 初始化博客项目

在项目目录中初始化 Hexo，会自动生成 scaffolds/、source/、themes/ 等基础目录：

```bash
hexo init
npm install
```

执行完后目录结构大致如下：

```bash
my_blog/
├── _config.yml        # Hexo 配置文件
├── package.json       # 项目依赖描述
├── scaffolds/         # 文章模板
├── source/            # 文章和静态资源存放目录
├── themes/            # 博客主题目录
└── node_modules/      # npm 安装的依赖包
```

### 启动本地服务器

完成初始化后，Hexo 自带的本地服务器就可以直接启动，用于查看博客内容是否正确渲染：

```bash
hexo server
```

启动成功后，浏览器访问 http<span></span>://localhost:4000，即可看到博客的默认页面；

> 别急！别看到有端口就急着用FRP进行映射，因为hexo server 本质上只是一个轻量级本地预览服务，虽然可以通过 FRP 等工具映射到公网进行访问，但要注意以下几点：
> - 它不是为长期稳定服务设计的，性能和安全性不适合生产环境；
> - 它的渲染速度比生产环境要慢很多，不适合用来做性能测试；
> 一旦关闭终端或重启机器，服务就会中断；
> 没有缓存机制、日志控制、访问控制等功能，暴露在公网风险较高；
>
> 所以，在正式部署之前，还有一些操作要做；

## 使用 Hexo 生成静态博客 + Caddy 反向代理

当你完成博客内容撰写后，就可以使用 Hexo 将其生成静态页面，供 Caddy 托管：

### 生成静态页面

在 Hexo 项目的根目录下运行：

```bash
hexo clean && hexo generate
```

或简写为：

```bash
hexo clean && hexo g
```

这会将博客的 HTML、CSS、JS 等静态资源统一输出到 public/ 目录中；这个目录就是你最终需要部署的网页内容；

### 使用 Caddy 托管博客内容

假设你希望通过 https<span></span>://blog.example.com 来访问这个博客，并且你的 public/ 目录路径为 /home/time/hexo-blog/public，Caddy 的配置大致如下（具体细节见我之前写的 [Caddy 文章](/2024/08/19/linux/Caddy)）：

```caddyfile
blog.example.com {
    root * /home/time/hexo-blog/public
    file_server
}
```

保存配置后，重新加载 Caddy 即可：

```bash
sudo caddy reload
```

> 注意：每次修改文章或主题后，记得重新执行 hexo g 生成静态文件；

这样，就已经将博客反向代理到本机的443端口并且开启了https，绑定了域名；

## 使用FRP实现公网访问

在某些情况下，我们的服务器部署在家里或内网环境中，并没有公网 IP；这时可以借助 FRP（Fast Reverse Proxy）进行内网穿透，让外部用户也能访问部署在本地的 Hexo 博客；

国内有很多提供FRP的服务，如果你有自己的公网服务器，也可以自己搭建FRP服务端（但如果有公网服务器，谁会在内网部署服务呢）；

> 需要注意的是：我国通过自定义域名实现公网访问博客是需要备案的；根据工信部规定，域名用于网站服务必须完成 ICP 备案，否则将无法解析或被阻断；

因为 FRP 平台和服务太多了，我也没法一一列举，这里只介绍一个通用的 FRP 映射配置方式：

### 将本地博客服务映射到公网域名

假设你本地的 Hexo 博客是通过 Caddy 启动的（前文所述），那么监听端口就是在 443 端口，我们希望通过公网访问 https<span></span>://blog.example.com；

那么配置大体如下：

```ini
[common]
server_addr = your.server.ip  # 替换成 FRP 服务端地址
server_port = 7000            # 替换成你服务端配置的端口

[hexo-blog]
type = tcp
local_port = 443
custom_domains = blog.example.com
```

然后将blog.example.com域名解析为 FRP 服务端地址，这样就可以实现公网的博客访问了；

## 总结

至此，一个完整的 Hexo 博客就搭建完成了：从安装环境、初始化项目、选择部署方式，再到通过 Caddy 提供 HTTPS 支持，最后结合 FRP 将内网服务映射到公网访问，整个过程虽然步骤不少，但每一步其实都非常清晰；而一旦搭好，后续的维护只需要写好文章、执行 hexo g，几秒钟就能上线更新，非常高效；

当然，这只是一个最基础的博客框架，还有很多可以优化的地方，比如：

- 主题定制：Hexo 主题丰富，你可以根据自己的喜好选择合适的主题，甚至自己动手定制主题；
- 文章分类：Hexo 默认没有分类功能，但可以通过插件实现；
- 评论系统：Hexo 默认没有评论功能，但可以通过插件实现；
- SEO 优化：Hexo 默认没有 SEO 优化功能，但可以通过插件实现；

博客就像是一个属于自己的技术“树洞”，**写给别人看是传播，写给自己看是成长**；希望你也能在这个过程中收获成就感，把更多思考沉淀为内容，持续构建自己的知识体系；
