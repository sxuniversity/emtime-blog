---
title: 小小调度器：轻量任务调度的应用
categories: mcu

date: 2023-04-12 19:12:56
updated: 2023-04-12 19:12:56
---

参考：
- http://www.51hei.com/bbs/dpj-132959-1.html
- https://www.armbbs.cn/forum.php?mod=viewthread&tid=110648
- https://bbs.eeworld.com.cn/thread-501913-1-1.html

仓库：
- https://github.com/smset028/xxddq
- https://github.com/fxyc87/xxddq<span></span>（改版）

前面我们从理论层面详细介绍了“小小调度器”的设计思路和实现原理；现在，到了把它真正用起来的阶段；在嵌入式开发的世界里，资源永远都是最宝贵的，尤其是在一些低端或者老旧的单片机上，内存和处理能力都非常有限；此时，一个轻量、高效且易用的任务调度方案尤为重要；

“小小调度器”恰恰在这方面表现得非常出色；它的设计极为精简，每个任务只需要保存两个状态变量：一个 unsigned short 用作定时器计数器，和一个 unsigned char 用来记录任务当前执行的代码行位置；换句话说，**一个任务的 RAM 占用最小仅为 3 个字节**，这在嵌入式系统中实在是太“香”了，不仅节省了宝贵的内存空间，也让系统能够同时管理更多任务；

这种极简设计，使得“小小调度器”在资源受限的环境下，依然能够稳定、可靠地执行多任务调度，满足绝大多数实际需求；无论是传感器采样、串口通信，还是屏幕刷新，都可以轻松实现任务之间的有序切换，既避免了复杂的状态机设计，也避免了繁重的 RTOS 资源开销；

## 准备工作

将 xxddq 文件添加到 Keil 工程的包含路径中；

因为调度器使用状态机宏的方式，大部分任务实际上是死循环（但函数声明仍有返回值）；如果不想看到烦人的告警，可以这样处理，在魔术棒 -> C/C++ 编译器 -> Misc Controls 添加:

```bash
--diag_suppress=128,111
```

## 任务结构定义

每个任务都需要定义一个包含 C_task 成员的结构体；例如：

```c
Class(LED0Task)
{
    C_task task;    // 必须包含
    uint8_t i;      // 用户自定义变量
}
led0 = {
    .i = 0
};
```

说明：
- C_task task; 是必须包含的，调度器会根据这个成员来管理任务；
- 用户自定义变量可以任意添加，但注意不要与调度器内部使用的变量重名；

## 任务函数定义

任务函数使用 TaskFun 和 EndFun 包围，如下：

```c
TaskFun(LED0Task)
{
    while(1)
    {
        if(me.i == 0)
        {
            GPIO_WriteBit(GPIOA, GPIO_Pin_10, Bit_RESET);
            me.i = 1;
        }
        else if(me.i == 1)
        {
            GPIO_WriteBit(GPIOA, GPIO_Pin_10, Bit_SET);
            me.i = 0;
        }
        WaitX(500);  // 必须手动主动释放CPU，否则该任务永远占用CPU
    }
}
EndFun
```

> 如果任务不需要延时，也需要使用 WaitX(0); 来释放 CPU 控制权；

## 任务调度与更新机制

### 外设中任务时间更新

考虑到每个任务变量是文件内定义的，我们采用“曲线救国”的办法：在每个外设对应的 .c 文件中定义更新函数，如：

```c
void GPIOTask_Update(void)
{
    UpdateTimer(led0);
    UpdateTimer(key0);
}
```

### 外设中任务运行函数

同样，为了更好的模块化，每个外设文件中也定义运行函数：

```c
void GPIOTask_Run(void)
{
    RunTask(LED0Task, led0);
    RunTask(KEY0Task, key0);
}
```

### 更新函数在 SysTick 中调用

```c
void SysTick_Handler(void)
{
    Ticks++;  // 可选全局计时器

    GPIOTask_Update();  // 定时更新所有任务计时器
}
```

### 在主循环中执行任务

```c
int main(void)
{
    // 初始化代码...

    while(1)
    {
        GPIOTask_Run();  // 执行各任务逻辑
    }
}
```

## 总结

“小小调度器”使用极简的宏和状态机思想，实现了对多个任务的低成本管理，非常适合内存紧张的 MCU 项目；通过合理组织代码结构，使用更新和执行函数对任务状态进行控制，你可以在不引入 RTOS 的情况下，实现类似多任务调度的效果；

这套方法虽然看似“原始”，但在资源受限系统中却非常实用；尤其适用于裸机开发场景中，让 MCU 稳定地跑多个任务而不会陷入状态机地狱或时间片混乱；

> **注意**:由于“小小调度器”本身内部的任务调度机制依赖于 C 语言的 switch-case 结构进行任务状态切换，因此在**任务函数内部**不要再嵌套使用 switch-case；这可能会导致状态跳转逻辑冲突，甚至出现难以排查的行为错误；如果确实需要实现复杂的分支逻辑，建议改用 if-else 或将该任务拆分为多个子任务分别管理，以保持调度系统的稳定性与可维护性；
