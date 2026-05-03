---
title: Letter Shell：问题修复与功能扩展
categories: mcu

date: 2024-02-15 08:01:59
updated: 2024-02-15 08:01:59
---

仓库：https://github.com/NevermindZZT/letter-shell

## 前言

在之前的文章中，我介绍了 Letter Shell 的基本使用方法；在实际使用过程中，我发现 Letter Shell 存在一些小问题，也结合项目需求对其功能进行了扩展；本文将介绍这些问题的现象与修复方法，并分享新增功能的实现过程和实际用途；

## 问题修复（部分修复）

> 现象：在 Letter Shell 中，串口输入时长按退格键或 Delete 键时，终端显示存在字符残留，虽然字符已无法继续删除，但视觉上仍未完全清除；

### 这背后的常见原因：

1. 终端行为：
   - 串口终端（如 SecureCRT、TeraTerm、MobaXterm 等）长按退格时会快速发出多个 0x08 字节；
   - 这些字符本质是 “光标后退一格”，但 Letter Shell 要负责同步显示以及缓存处理；

2. Letter Shell 内部处理问题：
   - Letter Shell 会维护一个输入缓存；
   - 如果没有正确处理 “删到头” 的情况，可能会出现：
     - 显示没擦干净（字符残留）；
     - 缓存指针已到头，仍在处理退格；
     - 没有回显正确的光标移动和空格清除字符；

### 修复方法要点

1. 判断输入缓存是否为空：在 shellHandler() 中处理退格符（0x08）时，判断光标是否已在开头，防止“负向删除”（即光标越界到输入行左边界之外）;

2. 对于连续退格，要保证：
   - Shell 缓冲区更新正确；
   - 串口输出内容与终端光标同步；
   - 不要超出缓存边界；

### 修改后的完整代码

此函数整体思路是：先调整缓存，再清空并重绘整行命令，最后将光标还原至正确位置；

```c
void shellDeleteByte(Shell *shell, signed char direction)
{
    char offset = (direction == -1) ? 1 : 0;

    if ((shell->parser.cursor == 0 && direction == 1)
        || (shell->parser.cursor == shell->parser.length && direction == -1))
    {
        return;
    }
    if (shell->parser.cursor == shell->parser.length && direction == 1)
    {
        shell->parser.cursor--;
        shell->parser.length--;
        shell->parser.buffer[shell->parser.length] = 0;
        shellDeleteCommandLine(shell, 1);
    }
    else
    {
        for (short i = offset; i < shell->parser.length - shell->parser.cursor; i++)
        {
            shell->parser.buffer[shell->parser.cursor + i - 1] = 
                shell->parser.buffer[shell->parser.cursor + i];
        }
        shell->parser.length--;
        if (!offset)
        {
            shell->parser.cursor--;
            shellWriteByte(shell, '\b');
        }
        shell->parser.buffer[shell->parser.length] = 0;
        // 将光标移动到行首
        for (short i = 0; i < shell->parser.cursor + 1; i++)
        {
            shellWriteByte(shell, '\b');
        }
        // 重新打印整个命令行
        for (short i = 0; i < shell->parser.length; i++)
        {
            shellWriteByte(shell, shell->parser.buffer[i]);
        }
        // 打印一个空格，覆盖掉原来的最后一个字符
        shellWriteByte(shell, ' ');
        // 将光标移动到正确的位置
        for (short i = shell->parser.length; i > shell->parser.cursor; i--)
        {
            shellWriteByte(shell, '\b');
        }
    }
}
```

> 该函数通过整体刷新命令行缓冲区，并手动控制光标移动，确保删除行为与终端显示保持一致，尤其适用于字符中间删除的情形；
> 不过在实际使用中，仍会在某些输入场景下出现字符未被正确清除的情况，说明该方案不够完全可靠；如果你对此有更优的实现思路，欢迎联系我交流（联系方式已在主页提供）；

## 功能扩展

> 在 Linux Shell 中，如果你在终端中键入了一长串命令，但突然不想执行，也不想一个一个退格删除，只需要轻轻按下 Ctrl+C，整行输入就会被取消，重新回到提示符；这种“强制销毁当前输入”的行为，对嵌入式调试来说也非常实用；
> Letter Shell 默认的 Ctrl+C 实现主要用于中断长时间运行的命令，但并不具备清除当前行的效果；接下来，我们就来实现这一功能；

### 实现思路

1. 使用 SHELL_EXPORT_KEY 注册一个 Ctrl+C 快捷键；
2. 在回调函数中：
   - 将 is_CtrlC_Pressed 标记设为 1（供其他任务判断是否中断）；
   - 重新输出提示符（myshellWritePrompt）；
   - 清空当前行缓存（myshellExec 直接执行空行）；
3. 由于 Letter Shell 的内部函数是 static 的，我们自行实现 myshellWritePrompt 和 myshellExec 两个版本；

### Ctrl+C 注册代码（port 文件中）

```c
/* Ctrl+C 注册，键值为 0x03 */
uint8_t is_CtrlC_Pressed = 0;

static void CtrlC_Interrupt(Shell* shell)
{
    // 设置中断标志位，可供其他长运行命令判断
    is_CtrlC_Pressed = 1;

    extern void myshellWritePrompt(Shell *shell, unsigned char newline);
    myshellWritePrompt(shell, 1); // 输出新的一行提示符

    extern void myshellExec(Shell *shell);
    myshellExec(shell); // 清除当前行输入
}

SHELL_EXPORT_KEY(SHELL_CMD_PERMISSION(0), 0x03000000, CtrlC_Interrupt, Ctrl_C);
```

### 添加 myshellWritePrompt 与 myshellExec 函数（shell.c）

由于原函数是 static，我们避免直接修改 Letter Shell 源码，而是新建两个函数：

myshellWritePrompt：打印提示符

```c
void myshellWritePrompt(Shell *shell, unsigned char newline)
{
    if (shell->status.isChecked)
    {
        if (newline)
        {
            shellWriteString(shell, "\r\n");
        }
        shellWriteString(shell, shell->info.user->data.user.name);
        shellWriteString(shell, ":");
        shellWriteString(shell, shell->info.path ? shell->info.path : "/");
        shellWriteString(shell, "$ ");
    }
    else
    {
        shellWriteString(shell, shellText[SHELL_TEXT_PASSWORD_HINT]);
    }
}
```

myshellExec：清空当前命令缓冲区
```c
void myshellExec(Shell *shell)
{
    if (shell->parser.length == 0)
    {
        return;
    }

    shell->parser.buffer[shell->parser.length] = 0;

    if (shell->status.isChecked)
    {
        // 此处不添加到历史、不调用命令，仅清空缓冲区
        shellParserParam(shell); 
        shell->parser.length = shell->parser.cursor = 0;

        if (shell->parser.paramCount == 0)
        {
            return;
        }
    }
    else
    {
        shellCheckPassword(shell);
    }
}
```

> **注意**：该功能依赖串口工具正确发送 0x03（Ctrl+C）字符，建议使用支持该行为的终端，如 MobaXterm、TeraTerm 等；

### 扩展建议：基于 is_CtrlC_Pressed 的中断任务设计

> 通过 is_CtrlC_Pressed 标志，用户可以在长时间运行的命令中主动检查该变量，实现软中断功能；例如：

```c
while (running) {
    if (is_CtrlC_Pressed) {
        break;
    }
    do_some_heavy_task();
}
```

> **注意**：此中断机制适用于循环任务或长延时任务的主动中断，不适用于系统层级的任务终止；

## 总结

本文基于 Letter Shell 在实际嵌入式项目中的使用经验，修复了输入过程中因退格处理不完整导致的字符残留问题，并新增 Ctrl+C 清除当前输入行的功能，提升了交互体验与调试效率；

在功能扩展的同时，尽量保持对原始源码的非侵入性，便于后续升级与维护；Letter Shell 作为一款轻量级命令行工具，其灵活的可扩展性值得深入挖掘；

如果你也在使用它，欢迎交流更多改进思路；
