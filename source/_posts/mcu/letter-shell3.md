---
title: Letter Shell：自定义命令
categories: mcu

date: 2024-02-15 09:22:38
updated: 2024-02-15 09:22:38
---

仓库：https://github.com/NevermindZZT/letter-shell

## 前言

在完成基本交互功能和修复了一些常见问题之后，Letter Shell 已经能够稳定运行于我的嵌入式项目中；然而，项目开发中常常需要更多“定制化”的功能，比如：
- 注册自己的命令，快速调试；
- 添加快捷键，提高开发效率；
- 让 Shell 能调用带多个复杂参数的函数（支持自动解析参数），避免为每个测试场景重复编写包装函数；

好消息是，Letter Shell 本身就提供了完善的接口支持这些扩展；坏消息是，官方文档对这些内容略显简略，实际用起来还需要一点“摸索”；

这些扩展并非仅是“锦上添花”，而是将 Shell 从一个输入工具，变成一个真正实用的交互调试平台的关键；

Letter Shell 中，所有能被执行的“功能”：无论是你写的函数、绑定的快捷键、设置的变量，甚至是用户本身，在底层其实都被统一视为“命令项（Command）”；这些命令共享一套注册、查找与执行机制，使得 Shell 的扩展非常灵活；

不过在实际使用中，我们往往会将“自定义命令”专指 **将 C 函数导出为 Shell 命令的行为**，这是调试开发中最常用、最直接、最强大的手段之一；通过简单的宏定义，就可以将任意函数变成终端可调用的命令，极大提升调试效率和系统可观测性；

## 写在前面

Letter Shell 针对常用编译器都提供了宏支持，而为了保证导出的命令不会被优化掉，不同编译器还有一些额外的配置技巧：
- Keil MDK：建议在编译选项中添加 --keep shellCommand*，防止命令结构被链接器裁剪；
- GCC：需要在链接脚本（ld 文件）中显式保留命令段：

```ld
_shell_command_start = .;
KEEP (*(shellCommand))
_shell_command_end = .;
```

## 函数命令导出

函数导出是最常见的命令注册方式，适用于调试、信息打印、功能调用等场景；

宏定义原型：

```c
/**
 * @brief shell 命令定义
 *
 * @param _attr 命令属性
 * @param _name 命令名
 * @param _func 命令函数
 * @param _desc 命令描述
 */
#define SHELL_EXPORT_CMD(_attr, _name, _func, _desc) \
    const char shellCmd##_name[] = #_name; \
    const char shellDesc##_name[] = #_desc; \
    SHELL_USED const ShellCommand \
    shellCommand##_name SHELL_SECTION("shellCommand") =  \
    { \
        .attr.value = _attr, \
        .data.cmd.name = shellCmd##_name, \
        .data.cmd.function = (int (*)())_func, \
        .data.cmd.desc = shellDesc##_name \
    }
```

这行代码中将 _func 强转为 (int (*)()) 是为了兼容不同的函数签名（不管参数多少），但这样做会掩盖类型安全，有些 IDE 可能会报警告；

这种强制类型转换方式牺牲了类型检查，适合用于调试环境，但如果对类型安全有更严格要求，也可以配合包装函数使用；

使用示例：

```c
void myShellTask(void)
{
    Shell* shell = shellGetCurrent();
    if(shell)
    {
        shellWriteString(shell, "Shell自定义程序测试");
    }
}

SHELL_EXPORT_CMD(
    SHELL_CMD_PERMISSION(0) |
    SHELL_CMD_TYPE(SHELL_TYPE_CMD_FUNC) |
    SHELL_CMD_DISABLE_RETURN,
    mytest, myShellTask, print message);
```

使用 mytest 命令后，将输出 "Shell自定义程序测试"；这个机制可以非常方便地快速验证某个函数逻辑；

## 变量导出

除了函数，也可以将变量导出到 Shell 中进行读写；支持类型包括：char、short、int、字符串、指针等；

宏定义原型：

```c
/**
 * @brief shell 变量定义
 *
 * @param _attr 变量属性
 * @param _name 变量名
 * @param _value 变量值（地址）
 * @param _desc 变量描述
 */
#define SHELL_EXPORT_VAR(_attr, _name, _value, _desc) \
    const char shellCmd##_name[] = #_name; \
    const char shellDesc##_name[] = #_desc; \
    SHELL_USED const ShellCommand \
    shellVar##_name SHELL_SECTION("shellCommand") =  \
    { \
        .attr.value = _attr, \
        .data.var.name = shellCmd##_name, \
        .data.var.value = (void *)_value, \
        .data.var.desc = shellDesc##_name \
    }
```

> **注意**：_value 必须是变量的地址，除非是字符串（数组本身即指针）；如果变量是只读的，可以加上 SHELL_CMD_READ_ONLY 属性；

使用示例：

```c
int varInt = 0;
SHELL_EXPORT_VAR(SHELL_CMD_PERMISSION(0)|SHELL_CMD_TYPE(SHELL_TYPE_VAR_INT), varInt, &varInt, test);

char str[] = "test string";
SHELL_EXPORT_VAR(SHELL_CMD_PERMISSION(0)|SHELL_CMD_TYPE(SHELL_TYPE_VAR_STRING), varStr, str, test);

Log log;
SHELL_EXPORT_VAR(SHELL_CMD_PERMISSION(0)|SHELL_CMD_TYPE(SHELL_TYPE_VAR_POINT), log, &log, test);
```

上述变量在终端中直接输入变量名（如 varInt），即可显示或修改值（除非是只读变量）；

## 按键定义导出

除了函数和变量，也可以绑定终端中的某些按键（如 Tab、Ctrl+C）来触发特定功能；

宏定义原型：

```c
/**
 * @brief shell 按键定义
 *
 * @param _attr 按键属性
 * @param _value 按键键值（大端序）
 * @param _func 按键函数
 * @param _desc 描述信息
 */
#define SHELL_EXPORT_KEY(_attr, _value, _func, _desc) \
    const char shellDesc##_value[] = #_desc; \
    SHELL_USED const ShellCommand \
    shellKey##_value SHELL_SECTION("shellCommand") =  \
    { \
        .attr.value = _attr|SHELL_CMD_TYPE(SHELL_TYPE_KEY), \
        .data.key.value = _value, \
        .data.key.function = (void (*)(Shell *))_func, \
        .data.key.desc = shellDesc##_value \
    }
```

按键值说明，按键值为按键发送的字节序列，按 **大端模式** 表示；例如：
- Tab 键发送 0x0B：键值为 0x0B000000
- 方向上箭头发送序列 0x1B 0x5B 0x41：键值为 0x1B5B4100

使用示例（绑定 Ctrl+C）：

```c
uint8_t is_CtrlC_Pressed = 0;

static void CtrlC_Interrupt(Shell* shell)
{
    // 标记 Ctrl+C 被按下，用于其他程序中判断
    is_CtrlC_Pressed = 1;

    // 下面两行是标准行为：换行 + 触发下一轮命令解析
    extern void myshellWritePrompt(Shell *shell, unsigned char newline);
    myshellWritePrompt(shell, 1);
    
    extern void myshellExec(Shell *shell);
    myshellExec(shell);
}

SHELL_EXPORT_KEY(SHELL_CMD_PERMISSION(0), 0x03000000, CtrlC_Interrupt, Ctrl_C);
```

这样在终端中按下 Ctrl+C 时，即可自动执行 CtrlC_Interrupt() 中定义的行为；

> 提示：可以使用串口调试工具（如 SecureCRT）开启“十六进制查看模式”，按下目标键并记录其字节序列，以获取精确键值；

## 总结

Letter Shell 的命令导出机制设计得非常灵活统一，使用 .shellCommand 段 + 宏展开的方式，让用户可以零成本地将已有的函数、变量或按键操作暴露到终端；这一机制不仅极大提高了调试效率，也为后期维护和动态控制提供了便利；

只需根据不同平台配置保留 .shellCommand 段（如通过 --keep shellCommand* 或 KEEP(*(.shellCommand))），即可确保这些命令在最终镜像中生效；
