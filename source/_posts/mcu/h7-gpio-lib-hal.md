---
title: STM32H7开发笔记(五):GPIO-输入处理-HAL库实现
categories: mcu
---

在上一篇中，我简单介绍了一下 easy_button，这一篇，我们使用 HAL 库结合 easy_button 来实现一个简单的按键输入例子。

## 工程导入

> 在这里我们依然沿用之前的工程，如果你打算新建工程，或者说没有看前面的文章，请先看 [STM32H7开发笔记(二):GPIO-HAL库实现](/2025/09/24/mcu/h7-gpio-hal/)

### 下载 easy_button

- Github仓库：https://github.com/bobwenstudy/easy_button

> 考虑到国内网络问题，部分读者可能无法访问 Github，所以我自己部署了 Gitea，将 easy_button 仓库同步到了我的 Gitea 服务器上，地址：https://git.orangetime.top/EMTime/easy_button

### 添加 easy_button 文件

进入下载好的 easy_button 路径，进入 ebtn 文件夹，将其中的所有文件复制到我们的工程目录中，在 MDK 软件中添加对应的源码文件，同时设置好头文件搜索路径，如下图：

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/11/29/6929f6af7874b.png"/>
</center>

## 代码实现

为了方便管理，我在 ebtn.c 同路径下创建了 ebtn_cb.c 和 ebtn_cb.h 文件，用于存放 easy_button 的具体实现函数。

### 初始化

easy_button 的初始化主要包含：
- 按键的时间配置参数（如消抖时间、长按时间等）
- 按键定义（区分按键）
- 实现按键状态读取函数
- 获取系统时间函数
- 实现事件回调函数

#### 时间配置

直接上代码：

```c
static const ebtn_btn_param_t default_param = EBTN_PARAMS_INIT
    (
      20,   // 按下去抖时间(ms)
      20,   // 释放去抖时间(ms)
      20,   // 点击最短时间(ms)
      300,  // 点击最长时间(ms)
      200,  // 连击间隔最大值(ms)
      500,  // 长按KEEPALIVE间隔(ms)
      10    // 最大连续点击次数
    );
```

easy_button 可以给每一个不同的按键设置不同的时间参数，这里我就定义了一个变量 default_param，之后所有的按键都使用这个参数。

#### 按键定义

```c
typedef enum
{
  USER_BUTTON1 = 0,
  USER_BUTTON_MAX,
} user_button_t;

static ebtn_btn_t btns[] =
{
  EBTN_BUTTON_INIT(USER_BUTTON1, &default_param),
};
```

用枚举来给我们的按键定义一个编号，实际上你不用枚举也是可以的，不过这样更方便管理。
然后使用 EBTN_BUTTON_INIT 宏来初始化我们的按键，第一个参数是按键编号，第二个参数是之前定义的时间参数。

#### 按键状态读取函数

```c
static uint8_t prv_btn_get_state(struct ebtn_btn* btn)
{
  switch(btn->key_id)
  {
  case USER_BUTTON1:
    return HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_SET;
  default:
    return 0;
  }
}
```

我们自己实现一个参数为 struct ebtn_btn* btn 的函数，根据按键编号来读取按键状态，我的按键按下时为高电平，所以我们判断是否为高电平，然后返回 1（表示true，按下了按键） 或者 0（表示false，按键没有被按下）。

#### 获取系统时间函数

```c
static uint32_t ebtn_user_get_tick(void)
{
  return HAL_GetTick();
}
```

这个函数很简单，直接调用 HAL 库提供的 HAL_GetTick() 函数即可，或者也可以直接获取 uwTick这个值，看你个人的习惯。

#### 实现事件回调函数

```c
static void prv_btn_event(struct ebtn_btn* btn, ebtn_evt_t evt)
{
  switch(evt)
  {
  case EBTN_EVT_ONPRESS:

    break;
  case EBTN_EVT_ONRELEASE:

    break;
  case EBTN_EVT_ONCLICK:
    if((ebtn_click_get_count(btn) == 2) && (btn->key_id == USER_BUTTON1))
    {
      HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
    }
    //printf("[BTN %d] Clicked, count=%d\r\n", btn->key_id, ebtn_click_get_count(btn));
    break;
  case EBTN_EVT_KEEPALIVE:
    if(btn->key_id == USER_BUTTON1)
    {
      HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
    }
    //printf("[BTN %d] Keepalive, cnt=%d\r\n", btn->key_id, ebtn_keepalive_get_count(btn));
    break;
  default:
    break;
  }
}
```

我们自己实现一个参数为 struct ebtn_btn* btn 和 ebtn_evt_t evt 的函数，我们根据事件类型来处理不同的按键事件，在不同的事件中通过判断按键编号来执行不同的操作。

其中：
- EBTN_EVT_ONPRESS：按键按下事件，当按键按下时触发。
- EBTN_EVT_ONRELEASE：按键释放事件，当按键释放时触发。
- EBTN_EVT_ONCLICK：按键点击事件，当按键被点击时触发，easy_button 将单击和多击进行了合并，统一都叫 EBTN_EVT_ONCLICK，在代码中可以看到，我们可以通过使用 ebtn_click_get_count() 函数来获取连续点击的次数，从而区分单击和双击以及更多的点击次数。
- EBTN_EVT_KEEPALIVE：按键长按事件，当按键被长按时持续触发，执行的周期和时间参数中的 KEEPALIVE 间隔一致，我千面写的是500，那么长按期间，每隔500ms就会触发一次 EBTN_EVT_KEEPALIVE 事件。
  - 那么如果想实现长按后只执行一次操作，应该怎么处理呢？在代码中其实你也看到了，注释掉的部分有一个函数 ebtn_keepalive_get_count，通过这个函数，我们可以获取到长按期间，keepalive 事件触发的次数，当这个次数为1时，表示长按后第一次触发 keepalive 事件，那么我们就可以执行我们想要执行的操作，之后值为2，3，4...，表示 keepalive 事件被多次触发，那么我们简单的通过 if 判断就可以不再执行长按的操作了。

### 加入到代码逻辑中

上面的代码，只是分别定义了按键的时间参数、按键定义、按键状态读取函数、获取系统时间函数、事件回调函数，但是并没有将他们关联起来，更没有实现按键事件的执行，所以我们还需要一些函数，将他们关联起来，并且实现真正的逻辑执行，如下：

```c
void ebtn_user_init(void)
{
  ebtn_init(btns,
            EBTN_ARRAY_SIZE(btns),
            NULL, 0,                    // 无组合键
            prv_btn_get_state,
            prv_btn_event);
}

void ebtn_user_process(void)
{
  ebtn_process(ebtn_user_get_tick());
}
```

ebtn_user_init 就是 easy_button 的初始化函数，第一个参数是之前定义的按键数组，包含了按键 ID和按键时间参数，第二个参数是按键数组的长度，第三个参数是组合键数组，第四个参数是组合键数组的长度，这里我们不需要组合键，所以都设置为 NULL 和 0，第五个参数是按键状态读取函数，第六个参数是事件回调函数。

ebtn_user_process 是用于周期性调用的函数，进行按键的状态读取、消抖、状态判断、事件执行等，它可以在定时器中断中调用，也可以在主循环中调用，如果你使用 RTOS，那么还可以单独开一个线程，调用这个函数，我这里就放在主循环中调用了。

### 预期效果

对于已经定义的 USER_BUTTON1，我通过按键状态读取函数将 PC11 与其关联起来，然后在事件回调函数中，实现了点击事件和长按事件，当双击按钮时，LED 灯会闪烁，当长按按钮时，LED 灯会每隔500ms闪烁一次，直到松开按键。

## 完整代码

### ebtn_cb.c

```c
include "ebtn_cb.h"

/* ---------------- 按钮参数配置 ---------------- */
static const ebtn_btn_param_t default_param = EBTN_PARAMS_INIT
    (
      20,   // 按下去抖时间(ms)
      20,   // 释放去抖时间(ms)
      20,   // 点击最短时间(ms)
      300,  // 点击最长时间(ms)
      200,  // 连击间隔最大值(ms)
      500,  // 长按KEEPALIVE间隔(ms)
      10    // 最大连续点击次数
    );

/* ---------------- 按钮ID定义 ---------------- */
typedef enum
{
  USER_BUTTON1 = 0,
  USER_BUTTON_MAX,
} user_button_t;

/* ---------------- 按钮对象 ---------------- */
static ebtn_btn_t btns[] =
{
  EBTN_BUTTON_INIT(USER_BUTTON1, &default_param),
};

/* ---------------- 获取按键状态 ---------------- */
static uint8_t prv_btn_get_state(struct ebtn_btn* btn)
{
  switch(btn->key_id)
  {
  case USER_BUTTON1:
    // 如果按下时为高电平
    return HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_SET;
  default:
    return 0;
  }
}

/* ---------------- 事件回调函数 ---------------- */
static void prv_btn_event(struct ebtn_btn* btn, ebtn_evt_t evt)
{
  switch(evt)
  {
  case EBTN_EVT_ONPRESS:

    break;
  case EBTN_EVT_ONRELEASE:

    break;
  case EBTN_EVT_ONCLICK:
    if((ebtn_click_get_count(btn) == 2) && (btn->key_id == USER_BUTTON1))
    {
      HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
    }
    //printf("[BTN %d] Clicked, count=%d\r\n", btn->key_id, ebtn_click_get_count(btn));
    break;
  case EBTN_EVT_KEEPALIVE:
    if(btn->key_id == USER_BUTTON1)
    {
      HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
    }
    //printf("[BTN %d] Keepalive, cnt=%d\r\n", btn->key_id, ebtn_keepalive_get_count(btn));
    break;
  default:
    break;
  }
}

/* ---------------- 系统时间 ---------------- */
static uint32_t ebtn_user_get_tick(void)
{
  return HAL_GetTick();
}

/* ---------------- 初始化函数 ---------------- */
void ebtn_user_init(void)
{
  ebtn_init(btns,
            EBTN_ARRAY_SIZE(btns),
            NULL, 0,                    // 无组合键
            prv_btn_get_state,
            prv_btn_event);
}

/* ---------------- 周期处理函数 ---------------- */
void ebtn_user_process(void)
{
  ebtn_process(ebtn_user_get_tick());
}
```

### ebtn_cb.h

```c
#ifndef __EBTN_CB_H_
#define __EBTN_CB_H_

#include "gpio.h"
#include "ebtn.h"

#ifdef __cplusplus
extern "C" {
#endif

void ebtn_user_init(void);
void ebtn_user_process(void);

#ifdef __cplusplus
}
#endif

#endif
```

### main.c

```c

...
#include "ebtn_cb.h"
...

int main(void)
{
  ...
  ebtn_user_init();
  ...
  while (1)
  {
    ...
    ebtn_user_process();
    HAL_Delay(5);
  }
}
```

## 总结

通过使用 easy_button 库，我们可以很方便的实现按键的点击、长按、连击等功能，并且可以很方便的扩展到多个按键，只需要在 ebtn_btn_t 数组中添加按键对象即可，非常方便。
