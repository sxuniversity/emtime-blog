---
title: Linux 配置 SSH 服务与免密登录全攻略
categories: linux

date: 2023-01-01 17:23:00
updated: 2023-01-01 17:23:00
---

在日常 Linux 使用与服务器维护中，**SSH（Secure Shell） 是最基础也是最重要的远程登录方式**；无论是登录云主机、远程树莓派，还是调试嵌入式设备，SSH 几乎都是首选；相比传统的 telnet，SSH 不仅加密安全、稳定高效，还支持文件传输、端口转发等多种强大功能；

对于刚接触 Linux 的朋友来说，图形界面是个不错的起点：桌面图标、可视化设置、终端窗口一应俱全，能有效降低学习门槛；随着对系统的深入了解，你会逐渐意识到：真正高效、灵活的操作方式，往往都依赖于 SSH 和命令行，尤其是在没有显示器的服务器或嵌入式设备上，SSH 几乎就是你与设备之间唯一的桥梁；

不过，SSH 虽强，配置过程中仍有不少容易踩的坑：服务未开启、端口被防火墙拦截、密钥认证失败、连接提示拒绝……这些问题在生产环境中尤为关键，一旦疏忽，就可能导致远程失联，甚至“锁死”机器；

这篇文章将系统梳理 Linux 下配置 SSH 的全过程，从安装服务、修改配置文件，到开启密钥认证与常见故障排查，帮助你一步步搭建一个安全可靠的远程访问环境，为你真正摆脱图形界面打下坚实基础；

## 阅读前建议掌握的基础

虽然这篇文章会尽可能地**从零开始讲解配置过程**，但如果你已经具备以下基础，会更容易理解和操作：
- 会打开一个终端窗口，能基本使用命令行；
- 了解 Linux 文件结构和权限（如 /etc、~/.ssh、chmod 等）；
- 能够使用 sudo 或已具备 root 权限；
- 理解“客户端”和“服务端”的概念，知道 SSH 是远程连接的一种方式；

如果使用 Windows 进行 Linux 的连接，建议先安装一个 Linux 终端模拟器，如 [Windows Terminal](https://github.com/microsoft/terminal)、[MobaXterm](https://mobaxterm.mobatek.net/) 或 [Xshell](https://www.netsarang.com/zh/xshell/)，这样在后续操作中，你可以更方便地复制粘贴命令和文件；

## 安装并配置 SSH 服务

想要通过 SSH 远程登录 Linux，第一步就是在系统中安装并配置好 SSH 服务端；以下操作以 Debian 系为例进行说明；

### 安装基础工具和 SSH 服务

先更新一下软件源并安装必要工具包：

```bash
sudo apt update
sudo apt install sudo net-tools nano openssh-server
```

其中：
- sudo 是非 root 用户提升权限常用的命令；
- net-tools 包含 ifconfig 等网络工具，便于查看 IP 地址；
- nano 是轻量级终端编辑器，用来修改配置文件；
- openssh-server 是 SSH 服务端，安装这个才能让其他设备通过 SSH 登录本机；

### 修改 SSH 配置文件

配置文件路径位于 /etc/ssh/sshd_config，用 nano 打开它：

```bash
sudo nano /etc/ssh/sshd_config
```

> **注意**：这里我们配置的是 SSH 服务端的设置文件 /etc/ssh/sshd_config，这是为了让其他设备能连接到这台机器；而 ~/.ssh/config 是客户端连接用的，跟远程登录进来的规则无关，会在后面详细介绍；

以下是常用的修改项，建议按需调整：

> 在 nano 中可以使用 Ctrl W 搜索关键字，下面的配置项中，你可以按下搜索，然后输入关键字，快速找到对应行；
> 编辑完之后，Ctrl S 保存文件，Ctrl X 退出编辑器；

```bash
# 允许使用 root 账户登录（不推荐用于线上，但开发阶段可用）
PermitRootLogin yes

# 控制是否启用 Pluggable Authentication Modules，禁用后可能会导致一些系统级认证特性（如 faillock, login.defs，失败登录次数限制等）失效
UsePAM yes

# 启用基于公钥的登录方式（即后续可以用 ssh key 登录）
PubkeyAuthentication yes

# 启用密码登录（如果你希望用用户名+密码方式登录）
PasswordAuthentication yes

# 修改默认端口（可选，如果你希望避开默认的22端口）
Port 9000
```

> **强烈建议**：生产环境中请设置 PermitRootLogin no 和 PasswordAuthentication no，避免被暴力破解！
> **UsePAM 说明**：如果你了解 PAM 并确实想禁用它，可以将该项设置为 no；一般情况下，建议保持默认值 yes；
> **注意**：如果你修改了 SSH 端口（如上例改为 9000），后续使用 SSH 登录时需要加上 -p 9000 参数，例如：

```bash
ssh root@192.168.1.100 -p 9000
```

### 重启 SSH 服务使配置生效

```bash
sudo service ssh stop
sudo service ssh start

#或者可以使用
sudo systemctl restart ssh
```

#### 设置 root 密码（如启用 root 登录）

如果你开启了 root 登录权限，记得设置 root 密码：

```bash
passwd root
```

根据提示输入并确认你要设置的密码；

### 测试 SSH 登录

配置完成后，你可以用以下命令测试 SSH 是否能正常工作：

```bash
ssh root@192.168.1.100
```

其中 192.168.1.100 是你的 Linux 机器的 IP 地址，如果一切正常，你会看到类似下面的提示：

```bash
The authenticity of host '192.168.1.100 (192.168.1.100)' can't be established.
ECDSA key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.1.100' (ECDSA) to the list of known hosts.
root@192.168.1.100's password: 
Welcome to Ubuntu 20.04.5 LTS (GNU/Linux 5.10.0-23-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri 2023-01-06 10:00:00 CST

  System load:  0.0               Processes:               91
  Usage of /:   11.6% of 19.56GB   Users logged in:         0
  Memory usage: 15%               IP address for eth0:      192.168.1.100
  Swap usage:   0%                IP address for eth1:      192.168.1.101

  Graph this data and manage this system at:
    https://landscape.canonical.com

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

0 packages can be updated.
0 updates are security updates.

Last login: Fri Jan  6 10:00:00 2023 from 192.168.1.101
```

## 配置 SSH 密钥认证（推荐）

到目前为止，你应该已经可以通过用户名和密码登录你的 Linux 机器了；但为了更高的安全性，我**强烈推荐你使用 SSH 密钥对进行登录认证**，而不是每次都输入密码；

### 为什么用密钥更好？
- 密钥认证方式几乎无法被暴力破解；
- 登陆过程无需输密码，自动又安全；
- 可以关闭密码登录，从根本上杜绝暴力破解风险；
- 可以配合 Git、远程同步等工具一并使用，一劳永逸；

### 生成 SSH 密钥对

如果你还没有生成过密钥，可以在**客户端机器**（即你用来连接远程服务器的电脑）上执行：

```bash
ssh-keygen
```

执行后，终端会提示你选择保存路径、是否设置密码等，直接按回车使用默认设置即可；

```bash
Generating public/private rsa key pair.
Enter file in which to save the key (/home/yourname/.ssh/id_rsa):  ← 回车
Enter passphrase (empty for no passphrase):  ← 回车
Enter same passphrase again:  ← 回车
```

如果你按提示一路回车，系统会默认把密钥保存到：
- **私钥**：~/.ssh/id_rsa（请务必保管好，**不要泄露或上传**）
- **公钥**：~/.ssh/id_rsa.pub（可以发送到服务器）

> **注意**：
> - 如果你以前生成过密钥，它会提醒你是否覆盖已有的密钥；
> - ~ 表示你的用户主目录，例如 /home/yourname/（如果是Win，那么会在用户目录的隐藏文件夹.ssh中，如果是其它工具生成的，那么请找工具对应目录）；
> - .ssh 是一个隐藏文件夹，用于存放 SSH 相关文件；
> - 文件名字不一定是 id_rsa，如果你之前生成过密钥，可能会是 id_ed25519、id_ecdsa 等，但后缀名一定是 .pub，然后和 pub 文件名字一样的就是对应的私钥文件；

### 把公钥拷贝到服务器

使用如下命令将本地公钥发送到目标 Linux 服务器：

```bash
ssh-copy-id -i ~/.ssh/mykey.pub user@your-server-ip -p 9000
```

也可以手动拷贝：

```bash
cat ~/.ssh/id_rsa.pub
```

将内容复制后，新建一行，把内容追加到目标机器的下列文件末尾中：

```bash
~/.ssh/authorized_keys
```

注意权限设置：

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 测试 SSH 密钥登录

配置完成后，你可以用以下命令测试 SSH 是否能正常工作：

```bash
ssh user@your-server-ip -p 22
```

如果一切顺利，不需要输入密码就可以登录了！

### 进阶：关闭密码登录（可选）

当你验证密钥登录无误后，可以选择**禁用密码登录**，让系统只能通过密钥访问，从而进一步提升安全性：

编辑 /etc/ssh/sshd_config：

```bash
# 编辑之前先备份，防止出错
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

sudo nano /etc/ssh/sshd_config
```

找到 PasswordAuthentication，改为 no：

```bash
PasswordAuthentication no
```

重启 SSH 服务使配置生效：

```bash
sudo service ssh stop
sudo service ssh start

#或者可以使用
sudo systemctl restart ssh
```

> **注意**：别在无法通过密钥登录的情况下贸然关闭密码登录，否则可能会锁死自己！

## 配置 SSH 客户端（可选）

如果你有多个 SSH 服务器，每次连接都要输入一长串命令，比如：

```bash
ssh root@192.168.1.10 -p 9000
```

时间一久难免繁琐，容易输错；其实，我们可以通过配置 SSH 客户端，让它自动识别要连接的服务器，大大简化操作；

### 编辑 SSH 配置文件

SSH 客户端会读取你主目录下的 ~/.ssh/config 文件，你可以通过以下命令进行编辑：

```bash
nano ~/.ssh/config
```

如果这个文件不存在，可以直接新建；

### 示例配置

假设你有一台服务器的 IP 是 192.168.1.10，端口是 9000，用户名是 root，你可以像这样写入一段配置：

```txt
Host myserver
    HostName 192.168.1.10
    Port 9000
    User root
    IdentityFile ~/.ssh/id_rsa

Host test-server
    HostName 192.168.1.12
    User root
```

其中：
- Host 是你自定义的别名，可以随便取，比如 myserver；
- HostName 是服务器的 IP 地址或域名；
- Port 是 SSH 端口，默认是22（这里是自定义了）；
- User 是用户名；
- IdentityFile 是你的私钥文件路径，默认是 ~/.ssh/id_rsa；

这样以后你只需要运行：

```bash
ssh myserver
```

> 配置好后，在命令行中输入 ssh my 然后按 Tab 键，可以自动补全服务器别名，非常方便；

有了这个配置文件之后，管理多台服务器也能井井有条；如果你不想每次都敲命令行连接服务器，这一步非常值得设置一下；

## 补充

很多用户配置完 SSH 后连不上，其实是端口被 ufw 或 firewalld 拦了，可以在测试 SSH 登录前修改防火墙规则，或者直接关闭防火墙；

例如 UFW：

```bash
sudo ufw allow 22  # 或你自定义的端口如 sudo ufw allow 9000
```

## 总结

SSH 是连接 Linux 世界的重要桥梁，从远程登录、文件传输到系统管理，它几乎无处不在；本文系统梳理了从安装 SSH 服务、配置基础项、启用密钥认证，到客户端简化连接配置的完整流程，帮助你一步步搭建起一个安全、高效、可靠的远程访问环境；

对于新手来说，理解配置文件结构、掌握基本命令操作，是迈向高效 Linux 使用的关键一步；而对于有一定经验的用户，启用密钥认证、关闭密码登录、设置客户端别名等进阶操作，则是进一步提升安全性与使用体验的核心所在；

如果你能按照本文的流程完成配置，未来在管理远程服务器、调试嵌入式设备或参与自动化部署时，将更加得心应手；希望这篇全攻略能为你的 Linux 学习之路打下坚实的基础；
