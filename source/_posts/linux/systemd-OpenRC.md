---
title: 通过 init 系统实现开机自启（以 systemd 和 OpenRC 为例）
categories: linux

date: 2025-02-12 17:10:28
updated: 2025-02-12 17:10:28
---

在 Linux 系统中，**“开机自启”** 是我们几乎绕不开的一个话题：不管是启动一个自建的服务、挂载硬盘、跑一个守护进程，还是让脚本在系统启动后自动运行，**归根结底都依赖系统使用的 init 系统**；

你可能搜索过相关方法，结果却一脸懵逼：
- 有人说往 /etc/rc.local 里加一行就行了；
- 有人建议写 .service 文件放进 systemd；
- 更有甚者，在 Alpine 这类轻量级系统里突然蹦出个 “OpenRC”——啥啊这是？

更糟的是，你跟着教程配置了一堆，结果脚本根本没执行，日志里还一点提示都没有；

我也踩过这些坑，什么 systemctl、runlevel、依赖顺序、文件权限……搞得人头大；

---

这篇博客将结合实际情况，总结 **systemd** 与 **OpenRC** 两种主流 init 系统下的开机自启方式：
- 在现代 Linux 系统中，**systemd** 已成为最主流的初始化系统和服务管理器，被包括 **Debian、Ubuntu、Fedora、CentOS 7+、Arch Linux** 在内的大多数发行版所采用；它提供了统一的服务控制方式（如 systemctl 命令）、并行启动优化、日志管理（journald）等丰富功能，适合中大型服务器、桌面系统及容器环境；
- 相比之下，**OpenRC** 是一个轻量级的 init 系统，广泛用于如 **Alpine Linux、Gentoo** 等追求极致体积与灵活性的发行版；它不依赖 systemd，使用简单的 shell 脚本管理服务，启动快速，特别适合资源受限或需要最大程度自定义的场景，如**容器、嵌入式设备和自定义 Linux 构建系统**；

目标是：**看完后你能快速写出属于自己的启动脚本，并能“优雅地”开机自启，而不是一顿 chmod +x + debug**；

## systemd

systemd是主流的 init 系统，被包括 Debian、Ubuntu、Fedora、CentOS 7+、Arch Linux 在内的大多数发行版所采用；

最近整了一个成都的LXC云服务器，7块包年，配置也“感人”：128M 内存 + 1G 硬盘；不过好在有25个可分配的V4端口，并且带宽高达1Gbps，非常适合用来跑frps以及虚拟组网等服务；

所以我就打算跑一些服务，让它们后台运行并且开机自启；但可能是因为性能或者别的一些问题，我使用 nohup + & 或 screen 都不稳定，别说开机自启了，连后台稳定运行都做不到；

以下以 frps 为例，演示如何在 systemd 下实现开机自启；

### 创建服务文件

首先，在 /etc/systemd/system/ 目录下创建一个以 .service 结尾的文件，如 frps.service；

```bash
sudo vim /etc/systemd/system/frps.service
```

建议以后统一使用比如 emtime_xxx.service 这样的命名方式，便于识别和管理；

以下是一个最小可用服务配置：

```ini
[Unit]
Description=Frp Server Service
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml
Restart=on-failure
RestartSec=5s
StandardOutput=file:/var/log/frps.log
StandardError=file:/var/log/frps-error.log

[Install]
WantedBy=multi-user.target
```

**说明：**
- After= 可根据需求改成 network-online.target，必要时加上 Requires=network-online.target，这样服务会在网络可用后启动；
- User 可根据需求改成其他用户，但要注意权限问题，比如访问低于1024的端口或者调用需要 root 权限的命令，都需要使用 root 用户；
- ExecStart 要使用绝对路径，不能写成 ./xxx.sh；
- **最后的命令不能中断或退出，否则服务会直接终止**；
- Restart=always 可用于无条件重启；

### 启用服务

创建完服务文件后，需要启用并启动服务：

```bash
# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 启动服务（frps 是服务名）
sudo systemctl start frps

# 设置开机自启
sudo systemctl enable frps
```

### 服务运行管理

```bash
# 查看服务状态
sudo systemctl status frps

# 查看运行输出
tail -f /var/log/frps.log
tail -f /var/log/frps-error.log

# 停止 / 重启服务
sudo systemctl stop frps
sudo systemctl restart frps

# 禁用开机自启
sudo systemctl disable frps

# 查看服务日志
sudo journalctl -u frps -f
sudo journalctl -u frps -n 50 --no-pager

# 验证服务文件语法
sudo systemd-analyze verify /etc/systemd/system/frps.service
```

## OpenRC

OpenRC 是一个轻量级的 init 系统，广泛用于 Alpine Linux、Gentoo 等发行版；

还是以frps为例，演示如何在 OpenRC 系统（如 Alpine）中实现开机自启；

### 安装 OpenRC（如未预装）

```bash
apk add openrc --no-cache
```

### 编写启动脚本

在 OpenRC 中，服务通过位于 /etc/init.d/ 的脚本来管理；我们可以手动编写一个脚本来启动 frps：

```bash
sudo nano /etc/init.d/frps
```

内容如下所示：

```bash
#!/sbin/openrc-run

name="frps"
description="Frp server"

command="/home/alpine/app/frp/frps"
command_args="-c /home/alpine/app/frp/frps.toml"
pidfile="/run/${RC_SVCNAME}.pid"

output_log="/var/log/frps.log"
error_log="/var/log/frps.err"

depend() {
    after sshd
    need net
}

start_pre() {
    # 确保 /run 目录存在，并可写入 PID 文件
    checkpath --directory /run --owner root:root

    # 创建或检查日志文件
    checkpath --file --mode 0644 /var/log/frps.log /var/log/frps.err
}

command_background="yes"
```

**说明：**
- command= 和 command_args= 指定要执行的命令及参数；
- pidfile= 指定 PID 文件的路径，通常建议放在 /run；
- output_log= 和 error_log= 可指定标准输出和错误输出文件路径；
- depend() 中声明依赖项，如网络（need net）和 SSH；
- start_pre() 在启动服务前执行一些准备工作，如创建所需目录或文件；
- command_background="yes" 表示服务以后台方式运行；

### 添加可执行权限

OpenRC 启动脚本必须具有执行权限：

```bash
sudo chmod +x /etc/init.d/frps
```

### 管理服务

```bash
# 启动服务
sudo rc-service frps start

# 停止服务
sudo rc-service frps stop

# 查看服务状态
sudo rc-service frps status

# 设置开机启动
sudo rc-update add frps

# 取消开机启动
sudo rc-update del frps

# 查看所有开机自启服务
sudo rc-update show
```

### 提示与建议

- **日志调试**：OpenRC 本身不会自动保存日志，建议你手动查看日志文件或将日志写入 /var/log 中的文件（如上所示），确保问题可定位；
- **后台守护进程注意事项**：确保你的命令不会自行退出（即不会直接跑完就结束），否则 OpenRC 会认为服务失败；
- **目录权限问题**：某些系统（特别是 Alpine）在 /run 下的临时目录可能会重置，建议服务脚本中始终用 checkpath 创建并赋权；
- **写 PID 的必要性**：OpenRC 通过 pidfile 判断服务状态，漏写 PID 文件可能导致服务状态判断异常；

## 总结

无论是主流发行版中功能强大的 systemd，还是轻量简洁、广泛用于容器的 OpenRC，它们都承担着 Linux 系统中“启动与管理服务”的关键角色；

这篇文章从两个角度出发，分别介绍了如何：
- 编写 systemd 的 .service 文件，实现服务注册、日志管理、异常重启等；
- 使用 OpenRC 编写 init 脚本，借助 checkpath 和 command_background 等机制实现后台运行和日志落地；

> 在实际使用中，你只需要记住：
> - systemd 写的是配置文件，放在 /etc/systemd/system/；
> - OpenRC 写的是可执行脚本，放在 /etc/init.d/；
> - 二者都可以通过 enable 或 rc-update add 实现“开机自启”；

最后，无论你是跑一个简单的守护进程，还是想在嵌入式设备上部署自己的服务，只要搞清楚系统用的是什么 init 系统，**再结合本文中的方法，一般都能跑通**（别再满世界搜 /etc/rc.local 了，那已经是“老黄历”了）；

希望本文能帮你绕过“服务起不来没日志还没报错”的痛苦，优雅地把脚本跑起来；
