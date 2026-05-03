---
title: OpenMV引出的QT排错
categories: linux

date: 2023-09-01 20:35:09
updated: 2023-09-01 20:35:09
---

参考：https://blog.csdn.net/LOVEmy134611/article/details/107212845

今天在 Ubuntu 上更新了 OpenMV IDE 后，突然发现软件打不开了；

Linux 用久了，第一个反应自然是：**用终端启动看看错误信息**；

在终端中进入 OpenMV 安装目录，执行启动脚本：

```bash
./openmvide.sh
```

果不其然，终端中输出了一堆报错信息，看起来像是 Qt 插件加载失败的问题：

<center>
  <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/13/684b02910cc72.png"/>
</center>

看起来像是QT的问题，那就进一步查看更详细的信息吧；

## 开启 Qt 插件调试模式

为了获取更详细的错误信息，我们可以设置环境变量来启用 Qt 插件的调试输出：

```bash
export QT_DEBUG_PLUGINS=1
./openmvide.sh
```

再次运行后，输出了类似这样的提示信息：

<center>
  <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/13/684b03004ac8b.png"/>
</center>

这里明确指出：Qt 无法加载 libqxcb.so 插件，是因为它依赖的某些库找不到；

## 检查插件依赖库

我们前往提示中指向的路径，查看该插件（libqxcb.so）所依赖的共享库：

```bash
cd /home/time/Doc/1software/openmvide/lib/Qt/plugins/platforms/
ldd libqxcb.so
```

能看到下面这样的结果：

<center>
  <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/13/684b0378cf859.png"/>
</center>

可以看到，某些共享库（如 libxcb-xxx.so）提示为 not found，说明这些库在系统中尚未安装；

## 安装缺失依赖

这一步其实就是“缺啥补啥”；

对于大多数 Qt XCB 相关插件问题，可以尝试安装如下依赖包：

```bash
sudo apt install libxcb-xinerama0
```

如果还有其他依赖缺失，终端里也会继续提示，然后我们继续安装即可，比如：

```bash
sudo apt install libxkbcommon-x11-0
sudo apt install libxcb-icccm4
sudo apt install libxcb-image0
```

总之，**看报错、查缺失、逐个安装**，直到所有 ldd 输出的依赖都变成 => /usr/lib/... 而非 not found 为止

## 再次启动软件

重新运行 OpenMV IDE：

```bash
./openmvide.sh
```

此时应该能够正常打开了；

## 总结

Qt 程序在 Linux 上偶尔会遇到因为环境或依赖缺失导致的启动问题，尤其是那些 不使用系统 Qt、而是自己打包了一份 Qt 的软件；这类问题可以通过：
- 终端运行，查看错误信息；
- 启用 QT_DEBUG_PLUGINS=1 环境变量，获取详细调试日志；
- ldd 检查缺失的动态链接库；
- apt install 安装对应依赖；

来一步步定位并解决；

虽然过程有点繁琐，但也算是一次熟悉 Qt 插件系统和 Linux 动态链接机制的好机会；
