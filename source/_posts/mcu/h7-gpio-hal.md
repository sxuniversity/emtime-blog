---
title: STM32H7开发笔记(二):GPIO-HAL库实现
categories: mcu
---

在单片机的世界里，“点亮一盏 LED”几乎是所有开发的第一步。它不仅是传统意义上的“Hello World”，更是验证开发环境是否正确搭建、工具链能否正常工作的基本手段。

在 STM32H7 上，GPIO（通用输入输出口）依然是最基础、最常用的外设。无论是后续的串口通信、外设控制，还是复杂的总线接口，几乎都离不开 GPIO 的支撑。

本文将从 **HAL 库** 的角度出发，介绍如何完成 GPIO 的初始化与配置，并通过点亮开发板上的 LED 来完成第一个小实验。这也是我们深入 STM32H7 开发的起点。

## CubeMX 配置

意法半导体为 STM32H7 提供了两套官方开发库：**HAL 库** 和 **LL 库**。 
- HAL 库：官方推荐，封装更高，开发效率更快；
- LL 库：更接近寄存器，性能可控，但门槛更高。

本文使用 HAL 库来实现 GPIO 的配置和控制。

> 顺带一提，HAL 和 LL 库是可以混用的，但需要你有一定的寄存器基础，否则很容易出问题。我后续还会写 libopencm3 的使用笔记，它和 LL 思路类似，可以作为参考。

新建一个 CubeMX 工程，选择 **STM32H743VIT6** 作为目标芯片。创建时会提示 MPU（内存保护单元）的配置，暂时不用管，直接点击 “Yes” 即可。

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/09/25/68d4269cc8360.png"/>
</center>

### 配置时钟

在左侧的 **System Core → RCC** 中进入时钟配置，选择合适的时钟源，并配置相应的时钟，如下所示：

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/09/25/68d427d955624.png"/>
</center>

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/09/25/68d42823e4531.png"/>
</center>

### 配置 GPIO

在本篇文章中，我们会同时演示 GPIO **输出**（点亮 LED）和 **输入**（读取按键）。

在我的核心板上：
- **LED** 接在 PE3
- **按键** 接在 PC13

对应的原理图如下：

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/09/25/68d42a776216b.png"/>
</center>

这里 PE3 并不是直接接 LED，而是通过一个三极管来驱动。简单来说：
- 当 PE3 输出高电平时，三极管导通，LED 亮；
- 当 PE3 输出低电平时，三极管截止，LED 灭。

配置效果如下图所示：

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/09/25/68d4cb6bbee29.png"/>
</center>

> 小技巧：看到这种三极管驱动电路时，不用慌，直接看三极管箭头方向——引脚电流和箭头方向一致时，三极管导通，LED 就会亮。

在 CubeMX 中还可以给引脚加上 **User Label**，这样在生成代码时会有 LED_GPIO_Port 和 LED_Pin 这样的宏，方便后续编程。

对于输入引脚 PC13，电路中串了一个电阻和 ESD 保护器件。ESD 可以忽略，把它当作导线即可。限流电阻保证安全。
但需要注意的是：**按键松开时，PC13 会悬空（电平不确定）**，所以我们在 CubeMX 里要把它配置为 **下拉输入**。

配置效果如下：

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/09/25/68d4cbd832ff9.png"/>
</center>

## 代码编写

配置完外设后，生成对应的工程。我这里使用 **MDK**，你也可以选择其他 IDE。

在 main.c 中，我们添加简单的按键检测和 LED 控制逻辑：

```c
#include "main.h"
#include "gpio.h"

...

int main(void)
{
  ...

  /* USER CODE BEGIN WHILE */
  while (1)
  {
    if(HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin) == GPIO_PIN_SET)
    {
      HAL_Delay(20);
      if(HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin) == GPIO_PIN_SET)
      {
        HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
      }
      while(HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin) == GPIO_PIN_SET);
    }
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
}
```

这段代码实现了一个最经典的“按键控制 LED”功能：

- 按下按键 → LED 状态反转（亮 ↔ 灭）
- 松开按键后才能继续检测（避免长按一直触发）

当然，如果需要直接控制电平，也可以用 HAL_GPIO_WritePin：

```c
HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET);   // 点亮 LED
HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_RESET); // 熄灭 LED
```

## 总结

GPIO 是单片机开发的起点，也是后续一切外设驱动的基础。

- 输出方面：常见用法是点灯，比如在多线程或异步场景下，周期性闪烁的 LED 就能作为“心跳指示灯”，帮助我们快速判断设备是否运行正常。
- 输入方面：最常见的就是按键，不仅能作为人机交互入口，还可以用外部中断的方式捕获事件，这部分我会在后续文章中单独展开。
