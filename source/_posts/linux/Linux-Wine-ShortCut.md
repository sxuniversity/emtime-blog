---
title: Linux向Wine应用传递快捷键
categories: linux

date: 2023-01-03 15:03:11
updated: 2023-01-03 15:03:11
---

众所周知，在 Linux 下跑 Windows 软件（比如用 Wine 打游戏、用一些 Windows 独占工具）已经不是什么稀奇事；但很快你可能会遇到一个让人抓狂的小问题：
> “我想在Linux运行的时候使用快捷键直接调用Wine 里的程序，想整点快捷键自动化操作，结果发现 Linux 上的热键根本传不进去！”

是的，直接按键/发送快捷键在宿主系统上没问题，但一到 Wine 里就经常失效，不管是自定义快捷键、窗口焦点切换，还是模拟复杂组合键，通通都失灵了；

不用急，这其实并不是快捷键失效，而是你发送的按键根本没传到 Wine 的窗口里；我们只需要用到一把熟悉的“瑞士军刀”：xdotool！

虽然 xdotool 本身是为 X11 桌面环境设计的，但只要掌握正确的用法，它一样能**准确地把键盘事件“打”进 Wine 应用窗口**；本篇博客就来手把手讲讲怎么做到这一点，附带实际例子，让你的自动化宏在 Wine 里也能畅通无阻；

## 工具准备

我们仍然使用经典组合：xdotool + 系统快捷键绑定功能；

```bash
sudo apt install xdotool
```

## 思路解析

我们需要实现的是这样一件事：
- 找到正在运行的 Wine 应用窗口（比如 WeChat.exe）；
- 激活该窗口（确保它获得焦点）；
- 发送模拟按键事件，例如 Ctrl + Alt + W；

关键步骤就是用 xdotool 搜索窗口并发送按键；

## 编写快捷键脚本

我们先写一个简单的脚本，命名为 open_wechat.sh（存放位置自行决定，比如 ~/scripts/open_wechat.sh）：

```bash
#!/bin/sh
# 找到窗口标题中包含 "WeChat.exe" 的 Wine 应用窗口，并发送 Ctrl+Alt+W 快捷键

WINDOW_ID=$(xdotool search --name "WeChat.exe" | head -n 1)

if [ -n "$WINDOW_ID" ]; then
    xdotool windowactivate "$WINDOW_ID"
    xdotool key --window "$WINDOW_ID" ctrl+alt+w
else
    echo "未找到 WeChat.exe 的窗口"
fi
```

别忘了给脚本加上可执行权限：

```bash
chmod +x ~/scripts/open_wechat.sh
```

## 设置系统快捷键调用脚本

在 Ubuntu/Gnome 等图形界面系统下，你可以打开系统设置 → 键盘 → 自定义快捷键，自定义一个快捷键，比如 Ctrl+Shift+W 来执行这个脚本：
- 名称：打开微信快捷键;
- 命令：/home/你的用户名/scripts/open_wechat.sh
- 快捷键：自己定义，比如 Ctrl + Shift + W;

## 测试效果

确保 Wine 中的 WeChat 已经在运行，然后按下你设置的快捷键，如果一切正常，微信窗口应该被激活并执行你发送的 Ctrl+Alt+W 快捷键动作；

> 注意事项：
> - 窗口名称（如 "WeChat.exe"）必须 完全匹配标题，区分大小写；如果不确定可以用 xdotool search --onlyvisible --name . 来列出所有当前窗口标题；
> - 如果你想发送其它组合键（比如 Ctrl+Shift+T），直接修改 xdotool key 后面的参数即可；
> - 某些 Wine 程序可能使用多级窗口或虚拟桌面，你也可以尝试使用 xdotool windowfocus 或 xdotool windowraise；

## 总结

用 xdotool 这把万能工具，我们不仅可以自动化 Linux 原生应用，也可以让 **Wine 下的“Windows 应用”听从你的指令**；不需要复杂配置，不需要改注册表，也不需要驱动，只要你能找到窗口，就能让它“动起来”；

下次如果你想做个“自动聊天脚本”或者“连点助手”，别忘了还有 xdotool 这把好用的“键盘魔法棒”！
