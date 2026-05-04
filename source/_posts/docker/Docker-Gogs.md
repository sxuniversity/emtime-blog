---
title: Docker部署Gogs（个人Github）
categories: docker
date: 2024-08-16 16:29:55
updated: 2024-08-19 22:40:08
---

参考：
- https://blog.csdn.net/bbj12345678/article/details/122713883
- https://blog.csdn.net/yanzili/article/details/130615472

Gogs仓库：https://github.com/gogs/gogs

现在越来越多团队开始尝试搭建自己的 Git 服务，用于私有代码托管、版本控制以及团队协作；如果你觉得 GitLab 太重、Gitea 功能太多，那么 **Gogs** 这个轻量级的 Git 服务可能刚好适合你；

Gogs 的全名是「Go Git Service」，它由 Go 语言开发，体积小、部署快、占用资源极低，非常适合搭建在树莓派、低配机器或开发环境中自用；

这篇博客我就来分享下 **如何用 Docker 部署 Gogs**，整个流程简单高效，适合初学者和懒人党（比如我）参考；

## 部署Gogs

### 拉取 Gogs Docker 镜像

```bash
docker pull gogs/gogs
```

Gogs 提供官方维护的镜像，持续更新，推荐使用；拉取速度可能因网络而异，可考虑配置镜像加速；

### 创建挂载目录

为了数据持久化，我们提前创建挂载目录；这里以 /home/time/docker_mount/gogs_github 为例：

```bash
mkdir -p /home/time/docker_mount/gogs_github
```

这个目录将映射到容器的 /data 目录，Gogs 会把所有的配置、仓库文件、SSH 密钥等放在这里，重启、迁移都不影响；

### 启动 Gogs 容器

```bash
docker run -d \
  --name my_github \
  --restart=always \
  -p 10022:22 \
  -p 13000:3000 \
  -v /home/time/docker_mount/gogs_github:/data \
  gogs/gogs
```

解释一下：
- -p 13000:3000：Web 界面访问端口；
- -p 10022:22：Git SSH 克隆使用的端口；
- -v：挂载持久化数据目录；
- --name：容器名称为 my_github；

### 初始配置 Gogs

在浏览器中访问 http://运行服务的ip:13000，开始初始化设置；

配置项说明：
- **数据库类型**：选择 SQLite3（默认即可）；
- **应用名称**：如 EMTime's Private Repo，仅用于界面展示；
- **SSH 端口**：建议勾选“使用内置 SSH 服务器”；
- **应用 URL**：填写 http<span></span>://运行服务的ip:13000，注意不能加斜杠 /；
- **管理员账号**：填写好用户名、密码和邮箱；
- **点击“立即安装”** 即可完成；

> 安装完成后请及时记下管理员账号密码，后续登录需要用到；

### 修改配置（如关闭注册功能）

默认情况下，任何人都可以注册新账号；如果你希望私有使用，可以通过配置文件禁用注册功能：

```bash
nano /home/time/docker_mount/gogs_github/gogs/conf/app.ini
```

找到：

```ini
[service]
DISABLE_REGISTRATION = false
```

修改为：

```ini
[service]
DISABLE_REGISTRATION = true
```

保存后重启容器：

```bash
docker restart my_github
```

这样就只保留管理员创建用户的方式，更安全；

### 使用自定义域名（可选）

比如你使用了反代（如 Caddy、Nginx、frp 等）并绑定了你的域名，就可以修改 Gogs 的显示地址：

```bash
docker stop my_github
vim /home/time/docker_mount/gogs_github/gogs/conf/app.ini
```

找到：

```ini
APP_URL = http://localhost:13000
```

修改为：

```ini
APP_URL = https://git.140105.xyz
```

这里用我自己的域名为例，注意不要加斜杠 /；

保存后重启容器：

```bash
docker restart my_github
```

### 克隆仓库报 SSL 错误的解决办法（可选）

如果你的 HTTPS 使用的是自签名证书，Git 拉取可能提示 SSL 验证失败，可以关闭 SSL 验证（注意，仅建议在局域网或受信网络使用）：

```bash
git config --global http.sslVerify false
```

如果使用 Let's Encrypt 的证书则无需担心，Git 会自动验证；

## 为 Gogs 配置 SSH

在部署好 Gogs 之后，如果每次拉取和推送都输入账号密码，不仅麻烦还不安全；这时候我们可以通过 SSH 公钥认证 来实现更安全、无密码的 Git 操作；

这篇文章将一步步带你完成 Gogs 的 SSH 配置过程，适用于 Linux 和 Windows（使用 MobaXterm 等终端工具）环境；

### 生成 SSH 密钥对

在终端中输入以下命令，按提示操作：

```bash
ssh-keygen -t rsa -C "git@git.140105.xyz"
```

这里：
- -t rsa 表示使用 RSA 加密算法；
- -C 根据你自己之前部署的情况进行修改，这里以我自己的域名为例；
- 默认会将密钥保存在 ~/.ssh/id_rsa，你也可以选择自定义，例如 id_rsa_gogs；

Gogs 同样支持 ed25519 等更现代的算法，参考其界面提示或 GitHub 教程进行生成即可；

### 设置权限（非常重要）

确保私钥不可被其他用户读取，防止泄露：

```bash
chmod 600 ~/.ssh/id_rsa_gogs
chmod 644 ~/.ssh/id_rsa_gogs.pub
```

###  配置 SSH 连接

编辑 SSH 配置文件：

```bash
nano ~/.ssh/config
```

添加以下内容：

```ini
Host 随便取个名字
    HostName git.140105.xyz
    User git
    Port 10022
    IdentityFile ~/.ssh/id_rsa_gogs
    PreferredAuthentications publickey
```

解释一下这些配置：
- Host 是你之后在 git clone 命令中使用的别名；
- HostName 是实际的服务器地址；
- User 为固定的 git，Gogs 默认就是用这个账户名；
- Port 是你容器映射出来的 SSH 端口（上文我们用的是 10022）；
- IdentityFile 指向你刚刚生成的私钥；
- PreferredAuthentications 指定只使用公钥认证；

> 如果你是在 Windows 使用 MobaXterm 或类似终端，~/.ssh/config 位置在 C:\Users\<用户名>\.ssh\config，内容相同；

### 添加 SSH 公钥到 Gogs

登录 Gogs 网站 → 点击右上角头像 → 用户设置 → SSH 公钥 → 添加密钥；
- 名称随意，例如 My laptop SSH key；
- 内容粘贴 id_rsa_gogs.pub 中的全部内容；

这样 Gogs 就认识你的电脑了；

### 测试 SSH 连接

执行以下命令：

```bash
ssh -T git@git.emtime.top -p 10022
```

如果成功，会提示类似：

```bash
Hi EMTime! You've successfully authenticated, but Gogs does not provide shell access.
```

表示你已经通过公钥成功认证了！

### 克隆仓库的方式说明

使用 SSH 克隆仓库有两种方式：

```bash
# 推荐方式（更清晰直观）
git clone ssh://git@git.emtime.top:10022/EMTime/test.git

# 简写方式（Gogs 默认提供），这种写法虽然更短，但有些 Git 客户端或未配置 SSH 时可能会失败
git@git.emtime.top:EMTime/test.git
```

## 总结

部署一个轻量级的 Git 仓库服务其实非常简单，Gogs + Docker 就是最实用、门槛最低的选择之一：
- 简单易用，不依赖外部数据库；
- 支持 SSH、Web、组织管理；
- Docker 容器化部署，迁移方便；
- 可与 Caddy、Frp 等无缝结合，快速公网访问；

非常适合用于自用项目托管、内网代码管理、小型团队开发协作等场景；
