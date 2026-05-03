---
title: STM32H7开发笔记(四):GPIO-按键处理引入
categories: mcu
---

在前面的文章中，我们实现了一个最简单的功能：按下按键，LED 灯亮；松开按键，LED 灭。

这个小实验虽然验证了 GPIO 已经可以工作，但如果把它用到真正的项目里，很快就会遇到新需求——

比如，我们希望一个按键可以支持：
- 单击控制 LED
- 双击切换让单片机进入低功耗模式
- 长按触发期间，LED 灯闪烁

很明显，按键不仅仅是“开关”这么简单，它可能承载多种操作事件。为了让按键在项目中表现得灵活而可靠，我们就需要对按键进行更系统的处理。

这些库的功能大同小异，都是对按键进行消抖、长按、双击等处理，本文使用了 Github 中开源的 [easy_button](https://github.com/bobwenstudy/easy_button) 库进行按键的处理。

> 和之前一样，仓库都是在 Github 中，如果你访问 Github 没有那么顺畅，可以使用我提供的链接，我将仓库同步到了我自部署的 Git 服务器上：https://git.orangetime.top/EMTime/easy_button

easy_button 是一个轻量级但功能丰富的按键处理库，作者人家也说了，核心的按键管理机制是借鉴的 [lwbtn](https://github.com/MaJerle/lwbtn)，具备以下的特点：
- 多按键支持：理论上按键数量无限制
- 灵活事件机制：支持单击、双击、多击、长按、超长按
- 组合按键支持：基于 bit_array 实现组合逻辑，无需重复扫描逻辑
- 静态/动态注册按键：可按需选择，节省代码空间
- 可配置时间参数：每个按键可独立配置消抖、长按、双击间隔等参数

基于作者个人的角度进行横向对比，结果如下：
|  | easy_button | FlexibleButton | MultiButton	 | lwbtn |
| :--- | :--- | :--- | :--- | :--- |
| 最大支持按键数 | 无限 | 32 | 无限 | 无限 |
| 按键时间参数独立配置 | 支持 | 支持 | 部分支持 | 支持 |
| 单个按键RAM Size(Bytes) | 20(ebtn_btn_t) | 28(flex_button_t) | 44(Button) | 48(lwbtn_btn_t) |
| 支持组合按键 | 支持 | 不支持 | 不支持 | 不支持 |
| 支持静态注册(可以省 Code Size) | 支持 | 不支持 | 不支持 | 支持 |
| 支持动态注册 | 支持 | 支持 | 支持 | 不支持 |
| 点击最大次数 | 无限 | 无限 | 2 | 无限 |
| 长按种类 | 无限 | 1 | 1 | 无限 |
| 批量扫描支持 | 支持 | 不支持 | 不支持 | 不支持 |

可以看到，easy_button 在保证了代码体积最小化的同时，还提供了非常全面且灵活的按键处理功能，因此本文选择它作为按键处理的方案。

## 主要代码介绍

> 本来想着写一些源码解释，以及相关的一些说明的，但是发现我写出来之后，只有我能看懂，在我表述的过程中，又丢失了很多细节，所以还是主要介绍一下用到的部分，然后直接上代码吧。

### 事件类型

easy_button 仅保留了 4 种核心类型，可以满足大部分按键需求：

```c
typedef enum
{
    EBTN_EVT_ONPRESS = 0x00,
    EBTN_EVT_ONRELEASE,
    EBTN_EVT_ONCLICK,
    EBTN_EVT_KEEPALIVE,
} ebtn_evt_t;
```

其中：

- EBTN_EVT_ONPRESS 为按下事件，当按键按下的时候就会触发
- EBTN_EVT_ONRELEASE 为抬起事件，当按键松开的时候就会触发
- EBTN_EVT_ONCLICK 为点击事件，easy_button 将单击和双击以及更多次点击事件合并为一种类型，只要你是按下并抬起，就会记录为一次点击事件，至于如何区分，这个放在应用的时候再讲
- EBTN_EVT_KEEPALIVE 为长按事件，当按键按下并持续一段时间，就会触发，easy_button 也将长按做了和点击事件同样的处理，根据配置的时间，会记录长按的次数；结合记录的次数，我们可以独立实现进入长按时的功能，和持续长按时的功能

### 按键时长配置

easy_button 可以为每个按键提供不同的时间配置，基于这些配置，easy_button 不仅实现了软件消抖，同时还提供了更灵活的长按/多击自定义方案，每个按键都可以根据应用场景和物理电路特性配置为不同的参数：

```c
typedef struct ebtn_btn_param
{
    uint16_t time_debounce;
    uint16_t time_debounce_release;
    uint16_t time_click_pressed_min;
    uint16_t time_click_pressed_max;
    uint16_t time_click_multi_max;
    uint16_t time_keepalive_period;
    uint16_t max_consecutive;
} ebtn_btn_param_t;
```

其中：

- time_debounce 为按下消抖时间，防止按下时的按键抖动
- time_debounce_release 为松开消抖时间，防止松开时的按键抖动
- time_click_pressed_min 为单击按下最小时间，高于这个时间，才能算一次有效点击
- time_click_pressed_max 为单击按下最大时间，高于这个时间，通常会视为长按
- time_click_multi_max 为多击最大时间间隔，用于检测双击、三击等多击事件
- time_keepalive_period 为长按事件触发间隔，按键持续按下时，周期性触发长按事件的间隔时间
- max_consecutive 为连续点击最大次数，也就是多击上限，比如最多支持 5 次连续点击，超出 5 次后，系统会立即通知应用层进行处理

## 总结

easy_button 的使用其实非常的简单，接下来两篇文章，我会分别根据 HAL 库和 libopencm3 库，分别使用 easy_button 实现一个按键处理示例，通过这个示例，你可以了解到 easy_button 的使用方法，以及如何结合按键处理实现一些实际的功能。
