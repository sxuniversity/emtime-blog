---
title: Linux使用鼠标宏
categories: linux

date: 2023-01-03 12:49:25
updated: 2023-01-03 12:49:25
---

在 Windows 上设置鼠标宏，大多数人会依赖厂商提供的驱动或者“宏软件”；但在 Linux 上，**情况就复杂多了**，因为多数鼠标厂商**并没有提供 Linux 驱动支持**，尤其是便宜的鼠标，几乎无法期待有什么官方工具可以用；所以想靠安装“鼠标驱动”来配置宏？不现实；

不过别担心，Linux 玩的就是自由和组合，我们完全可以用现有的工具手动配置宏功能；本文就介绍一种通用的方案：**利用 xdotool + xbindkeys 实现鼠标宏功能**；

## 安装依赖工具

首先，我们需要安装两个软件：
- xdotool：模拟键盘和鼠标输入事件；
- xbindkeys：监听键盘/鼠标按键并执行命令；

安装命令如下：

```bash
sudo apt install xdotool xbindkeys
```

## 找到鼠标宏按键的“按钮编号”

接下来我们要搞清楚：**你鼠标上的某个按键，系统识别为哪个 Button**；比如侧键、DPI 切换键等；

在 Linux 中，可以使用 xev 工具来监听输入事件，它会弹出一个窗口，并输出按键或鼠标事件的详细信息：

```bash
xev
```

运行后会弹出一个小窗口，把鼠标移到窗口内，按下你想用作“宏”的鼠标键；终端里会刷出大量输出，找到类似这样的信息：

<center>
  	<img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/13/684bfdda6eb49.png"/>
</center>

<center>
  	<img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/13/684bfe12de62e.png"/>
</center>

其中 button 8 就是你按下的那个键的编号；例如：
- 我的鼠标上“向下的侧键”是 button 8；
- “向上的侧键”是 button 9；
- 有些鼠标 DPI 调节键在 xev 中根本不会触发事件（这通常意味着该按键需要厂商驱动才支持）；

> **注意**：每个鼠标的 Button 编号可能不一样，请务必用 xev 测试确认；

## 编写配置文件 .xbindkeysrc

xbindkeys 读取的配置文件路径为 ~/.xbindkeysrc，我们可以使用默认模板生成：

```bash
xbindkeys --defaults > ~/.xbindkeysrc
```

然后编辑它（有图形界面的话可以用 gedit，命令行用 nano 也可以）：

```bash
nano ~/.xbindkeysrc
```

添加如下内容，将鼠标按键与模拟的按键事件绑定：

```ini
# 鼠标按键 Button 8 对应键盘 Page_Down
"xdotool key Page_Down"
  b:8

# 鼠标按键 Button 9 对应键盘 Page_Up
"xdotool key Page_Up"
  b:9
```

当然你可以换成你想要模拟的任何键，比如 "xdotool key Ctrl+W"、"xdotool click 1" 甚至执行 shell 脚本；

## 启用并测试绑定

输入以下命令重新加载配置并启动 xbindkeys：

```bash
killall xbindkeys && xbindkeys
```

测试一下，按下鼠标的侧键，看看是否能够模拟出按下 Page_Up / Page_Down 的效果；一般来说，如果配置正确，是可以马上生效的；

## 设置开机自启（可选）

大部分基于 apt 安装的 xbindkeys，在桌面环境中会自动随图形会话启动；但如果发现重启后没有生效，可以手动设置自启：
- 放入 .xprofile 或 .xinitrc：

```bash
echo "xbindkeys &" >> ~/.xprofile
```

- 桌面环境的“启动应用程序”中添加一项 xbindkeys 即可；

## 总结

虽然 Linux 上没有“傻瓜式”的宏配置软件，但通过 xdotool + xbindkeys 这套组合，我们依然可以实现强大且自定义程度极高的鼠标宏功能 —— 无需驱动、跨桌面环境、兼容大多数 X11 系统；

这种方式虽然稍显“原始”，但胜在灵活、可控：**想绑定哪个键、执行什么动作，全部自己说了算**；

如果你用的是 **Wayland**，而不是 X11，**这套方法可能会失效**，因为 Wayland 和 X11 的输入事件机制有所不同；不过好在 Wayland 也有类似的工具，比如 **wayland-clipboard**、**wlroots** 等，可以参考本文的方法自行实现；
