---
title: Basic OS：从任务定义到运行调度的全流程实践
categories: mcu

date: 2024-07-09 18:47:32
updated: 2024-07-09 18:47:32
---

仓库：https://gitee.com/event-os/basic-os

在上一篇中，我们从架构视角出发，了解了 Basic OS 的设计理念和核心机制：共享任务栈、协作式调度、基于 PendSV 的上下文切换……这些特性让它成为资源受限设备中，极具性价比的结构化选项；

但了解原理只是第一步；

真正的价值，来自它在实际项目中的表现；**这篇文章，我们将从定义任务开始，带你完整走一遍 Basic OS 的应用流程**：如何写任务？如何初始化调度器？任务之间如何协作与切换？

不需要复杂的配置，也无需外部依赖；你将看到的是一个纯净、高效、清晰可控的任务调度过程，真正让你从“裸机大 while(1)”走向结构化设计的实战落地；

## 支持平台

当前 Basic OS 默认只适配了 Cortex-M0 和 M3 内核（比如 STM32F103 或 F030），虽然不算通用，但架构非常清晰，完全可以参考其结构来适配其他内核或平台；如果你对汇编和异常向量表比较熟悉，实现并不困难；

## 准备文件

使用 Basic OS 时，最基本的工作就是把它的核心源码添加到你的工程中（文件可以去开头所说的仓库中下载）；至少需要以下几个文件：

| 文件名 | 说明 |
| ----- | ---- |
| basic_os.c | 系统核心逻辑实现 |
| basic_os.h | 	系统核心头文件 |
| cpu.c | 与平台无关的通用 CPU 支持 |
| port_mdk.s | 与平台有关的汇编端口（不同平台需对应更换） |

此外，还需要你自己实现部分**平台相关的钩子函数、SysTick中断**等逻辑，官方示例中以 cb_basic_os.c 命名此文件；你也可以照此命名，并加入自己的实现：

```c
#include "basic_os.h"

void bos_port_assert(uint32_t error_id)
{
    while (1)
    {
        bos_delay_ms(10);
    }
}

uint32_t count_idle = 0;
void bos_hook_idle(void)
{
    count_idle++;
}

void bos_hook_start(void)
{
    // 系统调度器启动后的钩子函数
    // 可选实现，用于调试或初始化用户资源
}

void SysTick_Handler(void)
{
    bos_tick(); // 系统时基更新
}
```

>  **特别注意（针对 STM32 使用 CubeMX 的情况）**
> CubeMX 默认会自动生成 SysTick_Handler 和 PendSV_Handler，而这两个函数已由 Basic OS 实现；请务必在 **NVIC 配置页面 → Code Generation 设置中取消勾选它们的生成**，避免函数重定义冲突；

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/14/684d873056671.png"/>
</center>

## 初始化 Basic OS

在你的 main.c 中，初始化 Basic OS 的代码结构非常简单：

```c
#include "basic_os.h"

int main(void)
{
    static uint8_t stack[4096]; // 为共享栈分配空间

    bsp_init();                 // 板级初始化（LED/GPIO/UART等）
    basic_os_init(stack, 4096); // 初始化调度器

    basic_os_run();            // 启动调度器（不会返回）

    while (1)
    {
        // 实际上，这一行代码永远不会被执行到
    }
}
```

初始化的关键点是提供一段连续 RAM 区域作为共享任务栈，这里用了 4096 字节，具体大小依据任务栈深度和应用复杂度可调整；

> 注意：栈区必须是静态分配（static/global），不能是局部变量或动态分配，否则可能在函数退出后被破坏，导致调度器异常崩溃；

## 定义任务：结构化的多任务运行方式

在 Basic OS 中，每个任务就是**一个函数 + 一个注册宏**；定义任务的方式非常直观，适合嵌入式初学者理解；

以下是一个典型的多任务定义示例：

```c
#include "basic_os.h"

uint32_t count_poll1 = 0;

void _func_poll_1(void *parameter)
{
    while (1)
    {
        count_poll1++;
        bos_delay_ms(10); // 每 10ms 执行一次
    }
}

bos_task_export(poll1, _func_poll_1, 2, NULL);
```

bos_task_export参数：
- 第一个参数：任务名称（可用于调试）
- 第二个参数：函数指针（真正的任务执行体）
- 第三个参数：任务优先级（同优先级按注册顺序轮转）
- 第四个参数：任务参数（通常传 NULL）

> bos_task_export() 本质上是通过宏将任务描述信息放入 .bos_tasks 特定段，系统初始化时会自动扫描这段数据进行任务注册；所以bos_task_export()并不需要被调用，只需要在代码中这样导出（注册）任务即可；
> Basic OS 是协作式调度器，不支持抢占式调度；**只有当当前任务主动 yield 或 delay 时，调度器才会切换到其他任务**；优先级高的任务在有多个可运行任务时会优先执行，但**不会打断正在运行的任务**；

我们可以定义多个任务，观察它们是否能按预期并发执行、互不干扰：

```c
void _func_poll_2(void *parameter) { while (1) { count_poll2++; bos_task_yield(); } }
void _func_poll_3(void *parameter) { while (1) { count_poll3++; bos_task_yield(); } }
void _func_poll_4(void *parameter) { while (1) { count_poll4++; bos_task_yield(); } }
void _func_poll_5(void *parameter) { while (1) { count_poll5++; bos_task_yield(); } }
void _func_poll_6(void *parameter) { while (1) { count_poll6++; bos_delay_ms(10); } }

bos_task_export(poll2, _func_poll_2, 2, NULL);
bos_task_export(poll3, _func_poll_3, 2, NULL);
bos_task_export(poll4, _func_poll_4, 2, NULL);
bos_task_export(poll5, _func_poll_5, 2, NULL);
bos_task_export(poll6, _func_poll_6, 2, NULL);
```

你还可以添加一些控制硬件的任务，例如通过 GPIO 控制 LED 闪烁：

```c
#include "gpio_led.h"

void _func_led1(void *parameter)
{
    while (1)
    {
        LED1_ON();
        bos_delay_ms(500);
        LED1_OFF();
        bos_delay_ms(500);
    }
}

bos_task_export(task_led1, _func_led1, 2, NULL);
```

## 软件定时器

在某些实际场景中，仅靠 bos_delay_ms() 实现任务延时是远远不够的；比如，我们希望任务可以“预约”一段时间后执行一次，或中途暂停、恢复某个定时行为，这时就需要更强大的软件定时器机制；

Basic OS 内置了轻量级的软件定时器接口，能够提供灵活、可控的定时事件触发逻辑，避免我们手动维护繁琐的计数器状态；

这一部分的使用非常简单，核心只需要掌握以下 5 个函数：
- bos_timer_export(_name, _func, _oneshoot, _para)：导出（注册）一个定时器
  - _name：定时器名称
  - _func：定时器回调函数
  - _oneshoot：是否是一次性定时器，true 表示是一次性定时器，false 表示是周期性定时器
  - _para：定时器回调函数的参数

相当于给调度器声明“我有这么一个定时器”，必须在使用前调用一次，推荐在 main() 的初始化部分完成；

示例用法：

```c
void _func_poll_7(void *para);

bos_timer_export(my_timer, _cb_timer_tick, false, NULL);

void _func_poll_7(void *para)
{

}
```

- int16_t bos_timer_get_id(const char *name)：从软件定时器名称获取 ID，返回定时器 ID 或者错误代码

- void bos_timer_start(uint16_t timer_id, uint32_t period)：启动一个定时器，period 单位是毫秒，可以和get_id配合使用

- void bos_timer_stop(uint16_t timer_id)：停止一个定时器，可以和get_id配合使用

- void bos_timer_reset(uint16_t timer_id, uint32_t period)：重置一个定时器，可以和get_id配合使用

## 总结

Basic OS 虽然定位轻量，但其设计并不简陋；从任务注册、共享栈调度，到钩子机制与软件定时器，它具备了一个结构化操作系统的核心要素；本文从创建工程出发，完整梳理了 Basic OS 的使用流程，让我们真正从裸机的 while(1) 循环中走出，迈入结构化的多任务世界；

当然，Basic OS 并非万能，它不支持抢占式调度，也没有复杂的同步机制，但也正是这种“克制”，让它在资源受限的嵌入式设备中成为一种高性价比的选择；

如果你正在开发一个不复杂但希望清晰可维护的项目，Basic OS 值得一试；下一步，我们可以探索它与外设协同、低功耗运行、任务通信等更高级的实战应用；
