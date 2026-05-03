---
title: Letter Shell：自定义函数参数解析
categories: mcu

date: 2024-02-15 10:50:19
updated: 2024-02-15 10:50:19
---

仓库：https://github.com/NevermindZZT/letter-shell

## 前言

在嵌入式项目中，将函数导出为 Shell 命令是一项非常实用的能力；然而，随着调试和控制需求的增加，我们往往希望调用的函数能够接受多个参数，甚至是不同类型的复杂参数，而不仅仅是简单地触发执行；

为了解决这一需求，Letter Shell 提供了“函数签名（Function Signature）”机制；通过为函数指定参数类型，Shell 能够在命令行中自动完成字符串到对应类型的转换，从而支持整型、浮点、字符串、指针等多种参数的解析；这个机制让 Shell 不再只是“命令触发器”，而是一个支持复杂交互的真正调试接口；

本节将介绍函数签名机制的使用方式、参数类型的定义方式以及注意事项，帮助你实现更强大、更灵活的 Shell 命令调用功能；

## 函数签名引入

从 3.2.x 版本开始，Letter Shell 引入了 **函数签名（Function Signature）** 的概念；以往 Shell 调用函数时，会根据输入参数“猜测”目标类型并尝试转换；这个机制在简单命令中还能应付，但在类型较复杂或参数较多的情况下，错误率明显升高，一旦类型猜错，轻则输出异常，重则直接跑飞；

于是，现在你可以选择开启一个新的宏：

```c
SHELL_USING_FUNC_SIGNATURE
```

开启后，Shell 会依据你提供的“函数签名”字符串，对参数进行明确且严格的类型解析，从而彻底摆脱以往“猜类型”的粗放方式；

一旦参数类型和签名不匹配，Shell 会自动停止命令执行，这也算是多了一层保护；毕竟，错误执行一个函数远比拒绝执行带来的代价大得多；

### 基本类型支持

函数签名是一个字符串，用来声明函数每个参数的类型；以下是当前版本支持的基本类型签名：

| 类型 | 签名 |
| ---- | ---- |
| char（字符） | c |
| char（数字） | q |
| short（数字） | h |
| int（数字） | i |
| char*（字符串） | s |
| void*（任意指针） | p |

函数的返回值不参与签名声明，仅声明参数类型即可；

```c
int func(int a, char *b, char c);
```

对应的函数签名是：

```c
"isc"
```

### 使用方法

在使用 SHELL_EXPORT_CMD 宏导出命令时，只需要在最后添加 .data.cmd.signature = "..." 即可，示例代码如下：

```c
void shellFuncSignatureTest(int a, char *b, char c)
{
    printf("a = %d, b = %s, c = %c\r\n", a, b, c);
}

SHELL_EXPORT_CMD(SHELL_CMD_PERMISSION(0)|SHELL_CMD_TYPE(SHELL_TYPE_CMD_FUNC),
funcSignatureTest, shellFuncSignatureTest, test function signature,
.data.cmd.signature = "isc");
```

就这么简单，你只需声明一次签名，Shell 便会在每次命令调用时，自动进行对应的参数解析与校验；

### 自定义参数类型解析

函数签名不仅支持基础类型，还支持 结构体等自定义类型 的解析；自定义签名格式为：

```ini
L<类型名>;
```

比如自定义一个结构体：

```c
typedef struct {
    int a;
    char *b;
} TestStruct;
```

函数签名如下：

```c
"iLTestStruct;s"
```

则代表函数的参数为：int、TestStruct* 和 char*；

### 注册解析器

要让 Shell 正确解析 TestStruct，我们需要写一个解析器函数，并通过宏注册：

```c
int testStructParser(char *string, void **param)
{
    TestStruct *data = malloc(sizeof(TestStruct));
    data->b = malloc(16);
    if (sscanf(string, "%d %s", &(data->a), data->b) == 2)
    {
        *param = (void *)data;
        return 0;
    }
    return -1;
}
```

如果解析过程中涉及动态分配内存，推荐再提供一个清理函数：

```c
int testStructCleaner(void *param)
{
    TestStruct *data = (TestStruct *)param;
    free(data->b);
    free(data);
    return 0;
}
```

注册方式如下：

```c
SHELL_EXPORT_PARAM_PARSER(0, LTestStruct;, testStructParser, testStructCleaner);
```

如果不需要清理（比如结构体全是值类型），可以把最后一个参数设为 NULL；

### 函数导出 + 自定义参数组合示例

```c
void shellParamParserTest(int a, TestStruct *data, char *c)
{
    uint8_t temp[100] = {0};
    Shell *shell = shellGetCurrent();

    sprintf((char *)temp, "a = %d, data->a = %d, data->b = %s, c = %s\r\n", a, data->a, data->b, c);
    shellWriteString(shell, (const char *)temp);
}

SHELL_EXPORT_CMD_SIGN(SHELL_CMD_PERMISSION(0)|SHELL_CMD_TYPE(SHELL_TYPE_CMD_FUNC),
paramParserTest, shellParamParserTest, test function signature and param parser,
iLTestStruct;s);
```

使用示例，终端中输入命令如下：

```bash
paramParserTest 1 "12 2a" "apple"
```

解释如下：
- 参数1：int → 1
- 参数2：TestStruct* → data->a = 12, data->b = "2a"
- 参数3：char* → "apple"

这里用双引号包住结构体参数是为了清晰分割，但实际输入中引号可选（Shell 会智能处理）；

## 数组参数解析引入

从 Letter Shell 3.2.3 版本开始，函数签名机制进一步增强，新增了 **数组参数** 的支持，适用于如 int[]、char *[]、自定义结构体数组等情况；

### 启用方法

要启用数组参数解析功能，需要：

1. 开启宏开关：SHELL_SUPPORT_ARRAY_PARAM
2. 配置内存分配函数：确保宏 SHELL_MALLOC 和 SHELL_FREE 被定义，指向你项目中的内存分配函数（如使用 FreeRTOS，可指向 pvPortMalloc 和 vPortFree，裸机系统下可绑定 malloc / free 或其他实现）；

### 数组签名语法

在原有类型签名前加 [ 表示这是一个数组类型参数；例如：

| 类型 | 签名 |
| ---- | ---- |
| int[] | [i |
| TestStruct*[] | [LTestStruct; |

完整签名可组合使用，如：

```c
"i[i[LTestStruct;"
```

表示的类型分别为：
- int
- int []（整型数组）
- TestStruct*[]（结构体指针数组）

### 示例代码

```c
int shellArrayTest(int a, int *b, TestStruct **datas)
{
    int i;
    printf("a = %d, b = %p, datas = %p\r\n", a, b, datas);
    for (i = 0; i < shellGetArrayParamSize(b); i++)
    {
        printf("b[%d] = %d\r\n", i, b[i]);
    }
    for (i = 0; i < shellGetArrayParamSize(datas); i++)
    {
        printf("datas[%d]->a = %d, datas[%d]->b = %s\r\n", i, datas[i]->a, i, datas[i]->b);
    }
    return 0;
}

SHELL_EXPORT_CMD_SIGN(SHELL_CMD_PERMISSION(0)|SHELL_CMD_TYPE(SHELL_TYPE_CMD_FUNC),
arrayTest, shellArrayTest, test array param parser,
i[i[LTestStruct;);
```

使用示例，终端中输入命令如下：

```c
arrayTest 12 [65, 89, 45] ["100 hello", "56 world"]
```

含义解析如下：
- 参数1：int → 12
- 参数2：int [] → {65, 89, 45}
- 参数3：TestStruct []（结构体数组）→
  - datas[0]->a = 100, datas[0]->b = "hello"
  - datas[1]->a = 56, datas[1]->b = "world"

> **提示**：数组用中括号括起来，元素之间用英文逗号分隔；结构体数组的每个元素用双引号包裹，格式与结构体解析规则一致；

## 总结

通过引入函数签名机制，Letter Shell 显著提升了命令行参数解析的可靠性与灵活性，使其不仅仅是一个简单的命令触发器，更是一个具备强交互能力的调试入口；无论是基础类型、指针类型，还是结构体乃至结构体数组，用户都可以借助清晰的签名字符串和自定义解析器，实现类型安全且表达力丰富的参数传递；

此外，函数签名机制还为 Shell 使用者提供了更明确的接口约束，有效避免了因类型错误导致的崩溃或异常；配合内存管理机制、解析器注册机制和数组支持，Letter Shell 已具备构建嵌入式调试框架所需的高级能力；

对于追求高可靠性和高可维护性的嵌入式项目来说，掌握并使用好函数签名功能，将为 Shell 接口的设计与扩展带来极大的便利和安全保障；
