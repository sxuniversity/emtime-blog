---
title: Linux 与 Windows 的 USB 桥梁：USBIP 远程共享
categories: linux

date: 2023-06-05 20:52:02
updated: 2023-06-05 20:52:02
---

Windows usbip 仓库：https://github.com/cezanne/usbip-win
来源：https://blog.csdn.net/OTZ_2333/article/details/124073337

## 安装前提示

在 Windows 上使用 USBIP，需要加载一个未签名的驱动（比如 usbip.sys）；默认情况下，Windows 会阻止这类驱动安装；因此，你**必须**：

> **启用测试模式（Test Mode）**，以允许安装未签名驱动；

但请务必注意：
- ❌ 很多游戏/反作弊系统（如 Easy Anti-Cheat、BattlEye）将拒绝启动，表现为游戏无法启动！
- ❌ 某些安全软件也可能报错或拒绝运行；
- 📌 桌面右下角会多一个“测试模式”的字样（虽然可以隐藏）；

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/17/6850d3cac3622.png"/>
</center>

**对于安全性的特意说明**：
- 测试模式允许加载未经过微软签名的驱动程序，这意味着任何拥有管理员权限的程序都可以安装内核级驱动；
- 如果系统感染了恶意软件，它可能会借此绕过系统签名校验机制，加载内核级后门或监控驱动；
- 某些安全软件在测试模式下也可能无法发挥完整防护作用；
- **请务必确保你的系统是安全的，再启用测试模式；**

建议在非主力游戏环境中测试，或者在虚拟机/备用系统上使用；

## Win 安装

在 Windows 上使用 USBIP 实现 USB 设备共享，步骤略显复杂，尤其涉及驱动签名与系统安全策略变更；请务必在非主力系统或测试环境中进行操作；

### 下载并解压 USBIP 工具

前往[仓库](https://github.com/cezanne/usbip-win)中或可靠来源，下载 USBIP Windows 压缩包，并将其解压到你希望的目录中；

### 导入驱动签名证书

找到解压目录中的 usbip_test.pfx 文件，双击导入证书；**注意：需要导入两次**，分别导入到以下两个位置：
- **受信任的根证书颁发机构**
- **受信任的发布者**

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/17/6850dc5cae855.png"/>
</center>

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/17/6850dcb36a424.png"/>
</center>

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/17/6850dcf930a30.png"/>
</center>

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/17/6850dd2a1ffc3.png"/>
</center>

操作完成后，USBIP 所用驱动才能被系统接受；

### 启用测试模式（重大安全变更）

使用管理员权限打开 PowerShell，执行以下命令启用测试模式：

```powershell
bcdedit.exe /set TESTSIGNING ON
```

如需日后关闭测试模式，可使用：

```powershell
bcdedit.exe /set TESTSIGNING OFF
```

虽然已经提到过了，但是，**请务必确保你的系统是安全的，再启用测试模式；请勿在生产环境或主力电脑上启用此功能！**

### 配置网络与防火墙

USBIP 默认使用 **TCP 端口 3240**；一般在局域网中无需额外配置防火墙；但若连接失败，可尝试：
- 在控制面板中放行 3240 端口的入站规则；
- 确保客户端能 ping 通服务端 IP 地址；

### 更新 usb.ids（可选但推荐）

在软件目录下找到 usb.ids 文件，可用[最新版](http://www.linux-usb.org/usb.ids)替换它（官网经常无法直接下载，可手动复制内容粘贴保存）；

什么是 usb.ids？
- 它记录了所有 USB 设备的 ID 与对应厂商/产品名称；
- 类似网络中的“DNS”系统，可将数值型 ID 映射为人类可读的设备名称；
- 若你的设备是新型号，可能在文件中找不到名称，显示为 unknown product，但不影响实际功能；

### 客户端驱动安装

在完成服务端配置之后，还需在客户端执行 VHCI 驱动安装，步骤如下：
- 以管理员权限打开 PowerShell，切换到 USBIP 目录；
- 执行以下命令安装客户端驱动：

```powershell
./usbip.exe install
```

如后续不再使用，可通过以下命令卸载：

```powershell
./usbip.exe uninstall
```

> 可以将 USBIP 可执行文件目录添加到系统环境变量中，因此在后续使用中可以直接执行 usbip 命令，而无需切换目录或添加 ./ 前缀；

## Win 配置

### 服务端操作（共享 USB 设备）

1. 打开 PowerShell 并进入工具目录，以管理员身份打开 PowerShell，并 cd 到之前解压 USBIP 的目录下，例如：

```powershell
cd "C:\Path\To\usbip_win"
```

2. 列出本机可共享的 USB 设备

执行以下命令，查看所有被识别到的 USB 设备：

```powershell
.\usbip.exe list -l
```

输出内容中，每个设备前会显示一个类似 1-3 或 1-253 的编号（busid）；这个 **busid 非常关键**，稍后绑定设备时要用到；

例如：

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/17/6850e0819ea79.png"/>
</center>

- 每个-就是一个设备，比如最后面的那个 1-253 就是我们的 CH340，这个 busid 要记住；
- 这个 1-253 要记住；

3. 将设备绑定到 USBIP 服务

执行如下命令进行绑定（**注意：绑定后该 USB 设备将不再能被本地系统使用**）：

```powershell
.\usbip.exe bind -b 1-253
```

如需解绑设备，执行：

```powershell
.\usbip.exe unbind -b 1-253
```

4. 启动 USBIP 服务

启动 USBIP 服务，监听来自网络的 USB 连接请求：

```powershell
.\usbipd.exe -d -4
```

如果你想使用自定义端口（非默认的 TCP 3240），可以指定端口号：

```powershell
.\usbipd.exe -d -4 -tPORT
```

例如将端口设置为 4000：

```powershell
.\usbipd.exe -d -4 -t4000
```

> **注意**：绑定之后如果不想使用了，一定要先关闭usbip进程，再解绑设备，不能直接拔掉插在 Win 上的 USB 设备，否则会导致设备无法被解绑，也无法被 Win 再次识别

### 客户端操作（连接远程 USB 设备）

假设此时客户端机器（即你希望使用该 USB 设备的另一台电脑）已经完成了 VHCI 驱动安装；

1. 查看远程 USB 设备列表

执行以下命令，查看服务端设备列表：

```powershell
./usbip.exe list --remote [服务端IP地址]
```

2. 将远程设备挂载到本机

使用以下命令将远程设备附加到当前机器：

```powershell
./usbip.exe attach --remote=[服务端IP] --busid=[busid]
```

示例：


```powershell
./usbip.exe attach --remote=192.168.1.100 --busid=1-253
```

挂载成功后，设备会在你的设备管理器中像本地 USB 一样出现，可以正常使用；

3. 查看已挂载的远程设备

要查看已绑定的远程设备：


```powershell
./usbip.exe port
```

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/17/6850e2ba825d7.png"/>
</center>

4. 卸载远程设备

若想卸载设备，使用 port 输出中 Port 后的编号（如上示例中框起来的部分）：

```powershell
./usbip.exe detach -p [端口] 
```

## Linux 安装

以我使用的Ubuntu为例，安装usbipd；

### 安装 USBIP 所需工具

Ubuntu 官方源已经集成了对应的内核工具包；直接执行以下命令即可完成安装：


```bash
sudo apt install linux-tools-`uname -r` linux-cloud-tools-`uname -r` linux-tools-common
```

> uname -r 会自动填入当前正在运行的内核版本，确保安装的工具与内核匹配；
> 如果安装失败，则可以尝试安装通配版本：

```bash
sudo apt install linux-tools-generic linux-cloud-tools-generic
```

### 加载 USBIP 所需内核模块

安装完成后，需手动加载相关的内核模块；以下命令适用于大多数情况：

```bash
sudo modprobe usbip-core
sudo modprobe vhci-hcd
sudo modprobe usbip_host
```

这些模块分别负责：
- usbip-core：USBIP 的核心通信协议；
- vhci-hcd：客户端使用的虚拟 USB 主机控制器（Virtual Host Controller）；
- usbip_host：服务端使用，用于暴露本地 USB 设备；

> **注意**：这些模块在每次重启后会被清除，需要我们进行额外配置以实现开机自动加载；

### 设置开机自动加载内核模块（推荐）

为避免每次开机都要手动加载模块，我们可以将模块名写入系统的模块加载配置中：

```bash
sudo nano /etc/modules-load.d/modules.conf
```

在文件中添加如下内容（若文件不存在会自动创建）：

```txt
# USBIP modules for automatic load at boot
usbip-core
vhci-hcd
usbip_host
```

保存退出后，系统将在每次启动时自动加载这三个模块；

### 配置防火墙（服务端）

USBIP 默认监听 TCP 的 3240 端口，若你计划让其他设备访问此服务，需放行该端口；

如果你在使用**虚拟机测试**，没必要客气，**直接关掉防火墙最省事**：

```bash
sudo ufw disable
```

如果你在**物理机使用（尤其是公网环境）**，建议**只放行所需端口**，谨慎为上：

```bash
sudo ufw allow 3240/tcp
```

> 你也可以根据实际情况进一步限制来源 IP，提升安全性；

## Linux 配置

> 提示：以下操作**均需 root 权限**，建议全程添加 sudo，或者先执行 sudo -i 切换为 root 用户；

### 服务端操作

1. 查看本地可用的 USB 设备

```bash
sudo usbip list -l
```

该命令会列出所有当前接入的 USB 设备及其 BusID；每个设备都以一个 busid 表示，例如 1-1.3；

> 如果你看到以下错误信息，不用慌：

```bash
usbip: error: failed to open /usr/share/hwdata//usb.ids
```

这是因为系统中缺失了 USB 设备 ID 数据库文件 usb.ids；解决方法如下：

```bash
sudo mkdir -p /usr/share/hwdata/
sudo nano /usr/share/hwdata/usb.ids
```

然后前往 https://www.linux-usb.org/usb.ids 下载最新版的 usb.ids，将内容粘贴进去保存即可；

2. 绑定本地设备到 USBIP 服务

```bash
sudo usbip bind -b [busid]
```

例如，要绑定刚刚查到的 1-1.3，就执行：

```bash
sudo usbip bind -b 1-1.3
```

> 一旦绑定成功，该 USB 设备**将无法在本地继续使用**，系统会将它“移交”给网络；

如需解绑设备，执行：

```bash
sudo usbip unbind -b [busid]
```

3. 启动 USBIP 服务

```bash
sudo usbipd -d -4
```

这条命令会在 IPv4 模式下启动 USBIP 服务进程，默认监听 TCP 3240 端口；如果你需要修改端口，可以这样：

```bash
sudo usbipd -d -4 -t 4000
```

### 客户端操作

1.  查看服务端可共享的 USB 设备

```bash
sudo usbip list --remote [服务端IP]
```

和之前一样，记录下你想要连接的 busid；

2. 将远程设备绑定到本地

```bash
sudo usbip attach --remote=[服务端IP] --busid=[busid]
```

例如：

```bash
sudo usbip attach --remote=192.168.1.10 --busid=1-1.3
```

执行成功后，远程 USB 设备会被虚拟挂载到本地系统中，表现得和本地 USB 设备完全一样；

3. 查看本机 USB 设备（含远程挂载）

```bash
lsusb
```

如果设备成功挂载，你会在 lsusb 输出中看到新设备的条目；

4. 查看当前已连接的远程 USB 设备

```bash
sudo usbip port
```

会显示所有已经通过 USBIP 挂载的远程设备信息，包括它们使用的“端口号”；

5. 卸载远程设备

```bash
sudo usbip detach -p [端口号]
```

端口号可以通过上一步 usbip port 命令查看，例如：

```bash
sudo usbip detach -p 0
```

## 总结

通过 USBIP，我们可以打破物理距离的限制，在局域网中共享 USB 设备，让 Windows 和 Linux 系统之间实现真正的“USB 互通”。虽然设置过程略显繁琐，尤其是 Windows 下的测试模式和驱动导入步骤，但一旦配置完成，远程使用 USB 设备将变得和本地使用几乎无异。建议在测试环境中充分调试后再应用到实际生产中，特别要注意安全性和稳定性问题。希望这篇指南能为你的跨平台 USB 使用带来一些启发和帮助。
