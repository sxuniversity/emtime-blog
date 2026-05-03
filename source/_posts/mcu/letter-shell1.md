---
title: Letter Shell：嵌入式轻量命令行
categories: mcu

date: 2024-02-14 20:11:19
updated: 2024-02-14 20:11:19
---

仓库：https://github.com/NevermindZZT/letter-shell

在调试嵌入式系统时，你是否也曾烦恼过：每次验证一个功能都得重新烧录？或是面对一串串串口日志，想动态改个参数却无从下手？

**Letter Shell**，一个小巧、灵活、易嵌入的命令行框架，正是为此而生；它能让你的 MCU 拥有一个“命令行终端”，不仅可以动态执行指令、查看变量、调试功能，还支持自动补全、自定义参数解析，甚至能像 Linux shell 一样扩展子命令和模块；

我将带你快速上手 Letter Shell，从集成、注册命令到实际运行效果，一步步构建属于你的嵌入式交互工具；

## 支持功能
- 命令自动补全
- 快捷键功能定义
- 命令权限管理
- 用户管理
- 支持变量读写
- 支持代理函数和参数代理解析

## 移植步骤与配置说明（基于裸机或非时间片系统）

> 有时间片轮转的情况下会更简单；

1. 添加源码文件
- 将整个 Letter Shell 源码加入工程，ext/ 文件夹可视具体需求选择性添加；
- 与 shell.c 同目录下，新建 shell_port.c 和 shell_port.h 文件并添加到工程中；

2. 修改 shell_ext.h

在 shell_ext.h 中包含 stdio.h，因为部分接口使用了 size_t 类型；

## shell_port.c 配置

### 创建 Shell 对象与缓冲区

```c
Shell shell;
char shellBuffer[512];
```

### 实现 Shell 写函数

```c
signed short shellWrite(char* data, unsigned short len) {
    User_UART1_Transmit_DMA((uint8_t*)data, len);
    return (signed short)len;
}
```

> 如果你是基于 DMA 输出的串口设备，可以用上述方式封装；

### 编写任务函数

```c
void shell_loop(void* parameter) {
    static PElement buffer = 0;
    for (;;) {
        if (FIFO_Count(&Uart1Buffer_R)) {
            buffer = FIFO_Quit(&Uart1Buffer_R);
        } else {
            buffer = 0;
        }

        if (buffer != 0) {
            for (uint16_t i = 0; i < buffer->Size; i++) {
                shellHandler(parameter, buffer->Buffer[i]);
            }
            buffer = 0;
        }

        bos_delay_ms(10); // 避免死循环卡死
    }
}

bos_task_export(task_shell, shell_loop, 2, (void*)&shell);
```

> 这里使用的是Basic OS，如果你使用的是其他操作系统，请自行修改延时部分；
> 如果你使用支持时间片轮转的操作系统，可以省略延时部分；

### 自定义按键支持（Ctrl+C）

```c
uint8_t is_CtrlC_Pressed = 0;

static void CtrlC_Interrupt(Shell* shell) {
    is_CtrlC_Pressed = 1;

    extern void myshellWritePrompt(Shell *shell, unsigned char newline);
    myshellWritePrompt(shell, 1);

    extern void myshellExec(Shell *shell);
    myshellExec(shell);
}

SHELL_EXPORT_KEY(SHELL_CMD_PERMISSION(0), 0x03000000, CtrlC_Interrupt, Ctrl_C);
```

### 添加自定义命令与变量

```c
uint8_t Var_Test = 0;

void myShellTask(void) {
    Shell* shell = shellGetCurrent();
    if (shell) {
        shellWriteString(shell, "Shell自定义程序测试");
    }
}

SHELL_EXPORT_CMD(
    SHELL_CMD_PERMISSION(0) | SHELL_CMD_TYPE(SHELL_TYPE_CMD_FUNC) | SHELL_CMD_DISABLE_RETURN,
    mytest, myShellTask, print message);

SHELL_EXPORT_VAR(
    SHELL_CMD_PERMISSION(0) | SHELL_CMD_TYPE(SHELL_TYPE_VAR_SHORT),
    varTest, &Var_Test, vartest);
```

## shell_port.h 示例

```c
#ifndef __SHELL_PORT_H_
#define __SHELL_PORT_H_

#include "main.h"

void User_Shell_Init(void);

#endif
```

## 配置宏

所有的宏都在shell_cfg.h中，可根据需求进行配置；

建议配置：
- 禁用 SHELL_TASK_WHILE，避免死循环卡死主线程（如果支持时间片轮转可忽略）；
- 配置 SHELL_GET_TICK()，启用双击检测与超时自动锁定；
- 默认用户可改为自定义标识，如 "EMTime"；
- SHELL_SHOW_INFO 可关闭以简化登录信息；

## 运行

基础工作此时已经基本完成，接下来我们只需要在main函数中调用User_Shell_Init()，启动调度运行 shell_loop 任务即可（如果没有移植 RTOS，在while中调用shell_loop即可）；

## 总结

至此，我们已经完成了 Letter Shell 的基础移植，构建出了一个功能完整、可交互的嵌入式命令行终端系统。只需几十行配置代码，便能让你的 MCU 拥有灵活的“命令行界面”，不仅能在线调试、读写变量，还能扩展快捷键与命令控制。

这意味着：
- 你无需再为测试一个函数反复烧录
- 你可以实时打印调试信息、修改参数
- 你可以为设备加上“开发者入口”，未来还能做远程调试

更重要的是，**Letter Shell 的核心功能远不止于此**。在后续内容中，我还会继续分享更多的功能扩展与使用技巧，帮助你更好地利用 Letter Shell，提升嵌入式开发的效率与体验。
