---
title: Linux安装N系列显卡驱动
categories: linux

date: 2023-01-01 11:17:02
updated: 2023-01-01 11:17:02
---

在 Linux 下使用 NVIDIA 显卡（即 N 卡）常常会遇到一系列问题，例如亮度不可调、多屏无法正常工作、性能下降等，大多数问题的根源在于系统默认使用的是开源驱动（如 nouveau），功能和性能都远逊于官方闭源驱动；

当然，部分发行版（如 Ubuntu）提供图形化的驱动管理器，可以自动安装适配的驱动版本，但为了获得更稳定和更高性能的体验，尤其是在深度学习、图形加速等场景下，**手动安装官方驱动仍然是值得掌握的技能**；

## 安装前准备

### 从官网下载NVIDIA显卡驱动

<center>
  <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/13/684b15049701c.png"/>
</center>

> 注意事项：不要选择 Notebook 版本的驱动（即便你用的是笔记本），桌面版稳定性更好；

### 安装编译工具

```bash
sudo apt install gcc
```

### 屏蔽开源驱动（nouveau 等）

编辑内核模块黑名单文件：

```bash
sudo nano /etc/modprobe.d/blacklist.conf
```

在文件末尾添加如下内容：

```txt
blacklist vga16fb
blacklist nouveau 
blacklist rivafb
blacklist rivatv
blacklist nvidiafb
```

### 更新 initramfs 并重启

```bash
sudo update-initramfs -u
sudo reboot
```

## 安装驱动

### 进入图形界面之外的终端

按下Ctrl Alt F1（或者F2，F3等等）进入tty终端，并停止当前显示管理器：

```bash
# 根据使用的显示管理器选择
sudo service gdm3 stop     # GDM 用户
sudo service lightdm stop  # LightDM 用户
```

### 运行安装程序

```bash
sudo chmod +x ./Nvidia-xxx.xx.run
sudo ./Nvidia-xxx.xx.run
```

安装过程中会询问多个选项，推荐如下配置：

- 是否要内核模块签名：是，Sign the kernel module
- 是否生成密钥对：是，Generate a new key pair
- 是否删除密钥：否，No
- 是否重新构建xxx：是，Rebuild
- 是否自动更新配置文件（备份原配置）：Yes

## 驱动加载与 MOK 密钥导入

### 加载 NVIDIA 驱动模块

```bash
sudo modprobe nvidia
```

### 导入内核模块密钥（MOK）

```bash
sudo mokutil --import /usr/share/nvidia/nvidia*.der
```

接着：

```bash
sudo reboot
```

系统重启后会进入一个蓝色的 MOK 管理界面，按照以下步骤操作：
- 选择Enroll MOK
- 选择Continue
- 选择Yes
- 输入密码
- 选择Reboot

### 验证驱动是否生效

重启后，在终端中运行：

```bash
nvidia-smi
```

如果看到显卡状态信息说明驱动安装成功；如果没有，可以尝试：

```bash
sudo modprobe nvidia
nvidia-smi
```

## 安装后调整

### 亮度调节不可用的修复方法

编辑 X11 配置文件：

```bash
sudo nano /etc/X11/xorg.conf
```

在 Section "Device" 下添加：

```txt
Option "RegistryDwords" "EnableBrightnessControl=1"
```

保存后重启：

```bash
sudo reboot
```

### 使用 DKMS 自动维护驱动模块

每次内核更新后，NVIDIA 驱动常常失效；我们可以通过 DKMS 机制使驱动自动随内核更新进行重新构建：

```bash
sudo apt install dkms
sudo dkms install -m nvidia -v 470.74
```

其中 470.74 是驱动版本号，可通过查看 /usr/src/ 下的目录确定版本：

```bash
ls /usr/src/
# 例如会看到：nvidia-470.74/
```

> 版本记得要打全，比如：535.113.01

## 总结

本文介绍了如何在 Linux 系统中手动安装官方 NVIDIA 显卡驱动，并涵盖：
- 驱动安装前的准备；
- 正确的禁用开源驱动方法；
- .run 安装文件的执行流程；
- MOK 密钥导入与验证；
- 系统更新后通过 DKMS 保持驱动可用；
- 常见问题如亮度调节无法使用的解决方法；

相比发行版自带的驱动管理器，这种方式虽然略复杂，但可控性更强，兼容性更好，适用于需要稳定 GPU 支持的开发场景，特别是深度学习、CUDA 编程等任务；
