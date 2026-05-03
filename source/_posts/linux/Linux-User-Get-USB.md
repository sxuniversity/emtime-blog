---
title: Linux普通用户直接使用USB设备
categories: linux

date: 2023-10-14 20:00:39
updated: 2023-10-14 20:00:39
---

参考：
- https://blog.csdn.net/u012306391/article/details/88716014
- https://blog.csdn.net/u012572552/article/details/84955924

平时在 Linux 下调试 USB 设备，比如烧录器、串口设备、OpenMV 之类的小工具，很多时候我们会遇到一个老生常谈的问题：**“设备插上了，但普通用户却没有权限访问”**；

一开始你可能会试着加个 sudo 临时解决，但总不能每次都 sudo 启动图形界面应用或者命令行工具吧？更别说很多 IDE、调试器、自动化脚本根本不支持以 root 身份运行；

我也一样，当时我问自己：
- 是不是要加用户组？
- 是不是要写 udev 规则？
- 到底哪些权限决定了 USB 设备的可访问性？

本篇就从实际例子出发，整理一套稳定、可复用的方法，解决普通用户在 Linux 下访问 USB 设备权限不足的问题；适用于开发板、USB串口、摄像头、OpenMV、ST-Link、FTDI、CH340 等等；

希望你看完后不再因为一个“Permission denied”而反复插拔、重启、叠加 sudo 咒语；

## 实际情况：Sipeed SLogic 无法被普通用户访问

正巧前几天买了矽速的逻辑分析仪 SLogic，照着文档一顿操作，最后普通用户打开上位机仍然是无法操作；

很明显，这是USB设备默认权限的问题；

### 找出 USB 设备的 VID/PID

插上设备后，使用如下命令查看设备信息：

```bash
lsusb
```

输出如下：

<center>
  <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/13/684b11a7f0839.png"/>
</center>

说明这就是我们要处理的设备，记住**Vendor ID 是 359f，Product ID 是 0300**；

### 创建 udev 规则文件

我们可以通过自定义规则的方式，给该设备赋予普通用户访问权限；

编辑规则文件：

```bash
sudo nano /etc/udev/rules.d/slogic.rules
```

写入以下内容（**请将 time 替换成你当前登录用户名**）：

```bash
SUBSYSTEMS=="usb", ATTRS{idVendor}=="359f", ATTRS{idProduct}=="0300", GROUP="time", MODE="0666"
```

如果你嫌麻烦、设备多，也可以使用通配符规则（**不推荐生产环境使用**）：

```bash
SUBSYSTEMS=="usb", ATTRS{idVendor}=="*", ATTRS{idProduct}=="*", GROUP="time", MODE="0666"
```

### 重载 udev 规则并重启

```bash
sudo udevadm control --reload
sudo reboot
```

重启后，插上设备再打开上位机，普通用户应该就能正常使用了，无需 sudo；

## 总结

- 设备插上识别了，但权限不足 → 查 lsusb，写规则；
- 写规则时锁定 VID 和 PID，设置 GROUP 和 MODE；
- 通配符虽然方便，但建议调试时使用，生产环境谨慎配置；
- 一次设置，长期受益，不再为权限反复折腾；

如果你和我一样，在 Linux 会连接各种USB设备，不妨写几条规则统一配置，调试环境会瞬间清爽不少；
