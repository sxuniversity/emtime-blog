---
title: 小小调度器：轻量任务调度的艺术
categories: mcu

date: 2023-04-12 18:34:07
updated: 2023-04-12 18:34:07
---

参考：
- http://www.51hei.com/bbs/dpj-132959-1.html
- https://www.armbbs.cn/forum.php?mod=viewthread&tid=110648
- https://bbs.eeworld.com.cn/thread-501913-1-1.html

仓库：
- https://github.com/smset028/xxddq
- https://github.com/fxyc87/xxddq<span></span>（改版）

## 为什么我们需要一个“小小调度器”？

在写单片机程序时，大多数人一开始的代码结构可能都是这样的：

```c
int main(void)
{
    while(1)
    {
        // 任务1
        Task1();
        // 任务2
        Task2();
        // 任务3
        Task3();
        // 任务4
        Task4();
        // 任务5
        Task5();
    }
}
```

逻辑写得多了，while(1) 越来越臃肿，各种“定时执行”“条件轮询”杂糅在一起，程序**可维护性越来越差**；一旦需求变动，整个结构可能都要大改；尤其在要同时处理多个任务（比如串口通信、传感器采样、OLED 刷新）时，开发者常常要手动“计算延时”或“管理状态机”，费时费力，极易出错；

你可能会想：“我是不是需要一个 RTOS？”

确实，RTOS（如 FreeRTOS）能够很好地解决这些问题，提供任务调度、消息队列、定时器等机制，**把逻辑切分成更小、更独立的任务单元**；

但问题也来了：
- **RTOS 体积较大**，很多小内存 MCU（如 STM32F030、STC89C51）资源捉襟见肘；
- **学习成本不低**，任务优先级、中断嵌套、栈溢出等“高级”概念对初学者不太友好；
- 实际项目中我们也不一定要“完整的 OS”，**我们只是想让几个任务按时执行，彼此别卡着就行了**；

于是，一些老工程师自己写了“小小调度器”（xxddq）：

> 不是 RTOS，但解决了很多和 RTOS 类似的问题；

它的核心思想很简单：用定时器节拍+任务列表，实现轻量级任务调度，让任务“看起来像多线程”一样轮流运行；没有堆栈切换、没有抢占机制，结构清晰、开销极低、好调试、易移植；你甚至可以在几十字节的 SRAM 里跑一个“小小调度器”，同时调度多个任务而不慌；

> 虽说这是一个很“土”的调度器，但它实在太实用了；

### 它是怎么做到“调度”的？

小小调度器的核心思想其实很朴素：

> 每个任务都是一个**“会记住上次运行位置”的函数**；

这看起来像是“魔法”，但其实它只是用了 C 语言里的一个“老把戏”：**状态机 + switch-case + 宏封装**；

每个任务在运行时，会记住自己上一次运行的位置（哪一行），下一次就从那一行继续开始执行；中间你可以设置它延时、等待某个条件，甚至等待另一个子任务运行结束，**就像一个简化版的协程一样**；

调度器本身做的事情也很简单：
- 每个任务都有一个“定时器”
- 每次轮到它运行时，看它要不要延时
- 如果不延时，就进入它的任务函数

于是多个任务轮流执行，看起来就像是“多线程”一样，其实不过就是用状态机模拟出来的非抢占式任务调度；

### 来看看它的全部源码

我们就从 2.0 简易版开始看起，整套调度器只用了一个 .h 文件，不依赖任何库，写法清晰，非常适合用作学习或实际项目中的任务调度基础模块：

```c
#ifndef __XXDDQ_H_
#define __XXDDQ_H_

#define TimeDef         unsigned short
#define LineDef         unsigned char

#define END             ((TimeDef)-1)
#define LINE            ((__LINE__%(LineDef)-1)+1)

#define me  (*cp)
#define TaskFun(TaskName)     TimeDef TaskName(C_##TaskName *cp) {switch(me.task.lc){default:
#define Exit            do { me.task.lc=0; return END; }  while(0)
#define Restart         do { me.task.lc=0; return 0; }  while(0)
#define EndFun          } Exit; }

#define WaitX(ticks)      do { me.task.lc=LINE; return (ticks); case LINE:;} while(0)
#define WaitUntil(A)      do { while(!(A)) WaitX(0);} while(0)

#define UpdateTimer(TaskVar) do { if((TaskVar.task.timer!=0)&&(TaskVar.task.timer!=END)) TaskVar.task.timer--; }  while(0)
#define RunTask(TaskName,TaskVar) do { if(TaskVar.task.timer==0) TaskVar.task.timer=TaskName(&(TaskVar)); }  while(0)

#define CallSub(SubTaskName,SubTaskVar) do { WaitX(0);SubTaskVar.task.timer=SubTaskName(&(SubTaskVar)); \
                  if(SubTaskVar.task.timer!=END) return SubTaskVar.task.timer;} while(0)

#define Class(type) typedef struct C_##type C_##type; struct C_##type
Class(task)
{
    TimeDef timer;
    LineDef lc;
};

#endif

```

### 调度器背后的“小聪明”：状态保存与 OOC 思想

小小调度器虽然没有堆栈切换、没有抢占调度，看起来只是用宏拼凑出来的玩具，但它背后的原理和设计思路却非常巧妙，值得深入讲讲；

#### 状态保存：让函数“暂停”再“接着运行”

调度器里每个任务结构体（如 task）都有一个 lc 字段，表示**当前运行到哪一行**；
执行过程中，WaitX() 宏会将 lc 设置为当前行号，然后 return 暂停任务；
下一次调用时，任务会 switch(me.task.lc)，跳转回这个位置继续执行；

就像这样：

```c
#define WaitX(ticks) do { me.task.lc=LINE; return (ticks); case LINE:; } while(0)
```

这其实是一个**简化版的协程（Coroutine）**；你可以写出看起来是“顺序执行”的代码，其实是每次走一步、等一会儿，任务之间交替运行，互不阻塞；

#### 调度核心：定时器 + 状态机

调度器每轮执行时，检查任务的 timer 是否为 0：

```c
#define UpdateTimer(TaskVar) do { \
    if ((TaskVar.task.timer != 0) && (TaskVar.task.timer != END)) \
        TaskVar.task.timer--; \
} while(0)

#define RunTask(TaskName, TaskVar) do { \
    if (TaskVar.task.timer == 0) \
        TaskVar.task.timer = TaskName(&(TaskVar)); \
} while(0)
```

这就是“定时器驱动的调度”：
- timer 大于 0，表示任务还在等待
- timer 递减至 0，就可以运行了
- 运行后，任务可能再次 WaitX() 暂停，设置新的 timer

于是你就得到了一个极其轻量、但功能完整的非抢占式调度器；

#### 面向对象的影子：Object-Oriented C（OOC）

再来看看这个宏：

```c
#define Class(type) typedef struct C_##type C_##type; struct C_##type
```

这是在用 C 语言模拟“类”；再结合：

```c
#define me  (*cp)
```

你会发现你写任务函数的时候就像在写“类的成员函数”一样：

```c
TaskFun(MyTask)
{
    // 可以通过 me.timer / me.lc 访问自己的状态
    ...
}
EndFun
```

每个任务都是 struct 的一个实例，带有自己的状态字段；
调度器通过传入指针 cp，让每个任务自己保存状态、自己运行；

甚至还有类似“调用另一个对象”的操作：

```c
#define CallSub(SubTaskName, SubTaskVar) \
    do { \
        WaitX(0); \
        SubTaskVar.task.timer = SubTaskName(&(SubTaskVar)); \
        if (SubTaskVar.task.timer != END) \
            return SubTaskVar.task.timer; \
    } while(0)
```

就像一个对象调用了另一个对象的方法，还能“等待它完成”；

这种写法属于典型的 **面向对象的 C 写法（OOC）**，它让调度器：
- 拥有了“类与实例”的基本结构；
- 支持任务封装、子任务调用；
- 具备良好的扩展性和移植性；

## 总结

“小小调度器”虽不是真正意义上的 RTOS，却通过简单而巧妙的设计，解决了许多单片机多任务调度的痛点；它用定时器节拍和任务列表实现轻量级、非抢占式调度，借助状态机和宏封装巧妙保存任务运行状态，实现了类似协程的“函数暂停与继续”效果；与此同时，采用面向对象的 C 语言写法，让任务结构清晰、易于扩展和维护；整体代码简洁无依赖，适合资源受限的 MCU 环境，兼具实用性和学习价值；

后续内容将着重介绍如何在项目中实际使用“小小调度器”，包括任务定义、调度循环及典型应用场景，帮助读者快速上手并灵活应用；希望这套“小小调度器”能为你的单片机开发带来便利，也欢迎大家在实践中提出宝贵建议，共同完善这一轻量级调度方案；
