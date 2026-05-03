---
title: STM32H7开发笔记(三):GPIO-libopencm3库实现
categories: mcu
---

> 以为 H7 的 libopencm3 开发也和 F1、F4 用起来一样简单，结果发现还是有些不一样的，自己先小折腾了半天，差点刚开坑就结束了。

在上一篇文章中我们使用 HAL 库实现了 STM32H7 的 GPIO 控制功能，本文我们使用 libopencm3 库来实现同样的功能，有关于 libopencm3 的介绍可以参考我前面写的[libopencm3 开发STM32体验笔记](/2023/07/20/mcu/libopencm3)。

> 强烈推荐大家看一下这篇文章，才好理解 libopencm3 的使用方式，否则可能很难快速上手。

本文代码仓库：[stm32h7-libopencm3](https://git.orangetime.top/EMTime/stm32h7-libopencm3)，不想看我啰嗦的，可以直接拉代码，目录结构清晰，还写了详细注释。

> 本文对应于仓库中的 1gpio 文件夹

## 开发环境简单说明
- **系统**：Ubuntu 24.04 LTS（没啥特别的，装好 git、make 等基础工具即可）。
- **开发工具链**：arm-none-eabi-gcc
  - apt 里有老版本。
  - ARM 官网可下最新版：[ARM GNU Toolchain](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads)。
  - 我用的是 14.3 版本。装完记得把路径加到环境变量。
- **构建工具**：[xmake](https://xmake.io/)
  - 上手比 cmake 直观，能生成 makefile、ninja 等多种配置。
  - 我这边用 xmake 生成了一份 makefile，习惯 make 的同学也能直接用。
- **调试工具、烧录工具等**：参考我之前的那篇 libopencm3 笔记。

## 工程结构

工程目录结构如下：

### 工程与 libopencm3 的结构

```txt
./
├── libopencm3/           # 库源码
├── libopencm3-examples/  # 官方示例
└── stm32h7/              # 我自己的项目
    └── 1gpio
```

- libopencm3/：库源码，首次拉取后要进目录 make 一下，生成库文件和头文件。
- libopencm3-examples/：示例仓库，可惜目前还没 STM32H7 的例子。
- stm32h7/：我自己的代码。建议你要玩 F1/F4 的话，也建个同级目录，比如 stm32f1/，方便区分不同型号。

> libopencm3 与 libopencm3-examples 两个目录是 libopencm3 的官方仓库，为了方便大家拉取，我也将这两个仓库同步到了我自己部署的 Gitea 中。
> 地址分别是：
> - [libopencm3](https://git.orangetime.top/EMTime/libopencm3)
> - [libopencm3-examples](https://git.orangetime.top/EMTime/libopencm3-examples)。

### 工程内部目录结构

```txt
./
├── bin/                # 编译产物（elf、hex、bin）
├── build/              # 临时文件
├── user/               # 用户代码
│   ├── inc/            # 头文件
│   │   ├── gpio.h
│   │   ├── main.h
│   │   └── systick.h
│   └── src/            # 源文件
│       ├── gpio.c
│       ├── main.c
│       └── systick.c
├── cortex-m-generic.ld # 链接脚本
├── makefile
└── xmake.lua
```

- bin/ 目录用于存放编译生成的可执行文件，如果使用 xmake，则会一并生成 hex、bin 文件。
- build/ 目录用于存放编译生成的临时文件。

> 保持源代码和生成物分离是一个好习惯，能避免 Git 仓库污染。

- user/ 目录用于存放用户代码，我按照 HAL 的目录结构，将代码分成了 inc 和 src 两个目录，其中 inc 目录用于存放头文件，src 目录用于存放源文件。之后随着功能的增多，我还会增加 lib 等目录，分别用于存放相关的库文件。
- cortex-m-generic.ld 是链接脚本，用于指定程序的内存布局。
- makefile 是 make 的构建文件，这里我是使用 xmake 生成的，可以编译生成 elf 文件，但没有添加 hex、bin 文件的生成，如果需要生成 hex、bin 文件，可以自行添加。
- xmake.lua 是 xmake 的构建文件，用于指定项目的构建规则，我的工程也是使用 xmake 进行构建的。

## 功能实现

用 libopencm3 写 GPIO，需要先准备几个“地基”：
- 系统时钟配置
- SysTick 延时
- GPIO 初始化与操作

顺序差不多就是这样的，我们一步一步来。

### 时钟配置

我的核心板外部晶振为 **25 MHz**，芯片是 **STM32H743VIT6**，最大主频为 480 MHz，需要配置 PLL 把晶振倍频到 480 MHz，再分频给各个总线和外设。

CubeMX 的时钟树很直观，所以我通常用它来进行对照，libopencm3 的结构体配置和 CubeMX 的参数差不多是一一对应的：

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/09/28/68d81189b6f1d.png"/>
</center>

在 libopencm3 中，有一个结构体定义如下，分别对应了我们图片中的倍频、分频因子：

```c
struct rcc_pll_config {
	enum rcc_osc sysclock_source;     /**< SYSCLK source input selection. */
	uint8_t pll_source;               /**< RCC_PLLCKSELR_PLLSRC_xxx value. */
	uint32_t hse_frequency;           /**< User specified HSE frequency, 0 if none. */
	struct pll_config {
		uint8_t divm;                   /**< Pre-divider value for each PLL. 0-64 integers. */
		uint16_t divn;                  /**< Multiplier, 0-512 integer. */
		uint8_t divp;                   /**< Post divider for PLLP clock. */
		uint8_t divq;                   /**< Post divider for PLLQ clock. */
		uint8_t divr;                   /**< Post divider for PLLR clock. */
	} pll1, pll2, pll3;               /**< PLL1-PLL3 configurations. */
	uint8_t core_pre;                 /**< Core prescaler  note: domain 1. */
	uint8_t hpre;                     /**< HCLK3 prescaler note: domain 1. */
	uint8_t ppre1;                    /**< APB1 Peripheral prescaler note: domain 2. */
	uint8_t ppre2;                    /**< APB2 Peripheral prescaler note: domain 2. */
	uint8_t ppre3;                    /**< APB3 Peripheral prescaler note: domain 1. */
	uint8_t ppre4;                    /**< APB4 Peripheral prescaler note: domain 3. */
	uint8_t flash_waitstates;         /**< Latency Value to set for flahs. */
	enum pwr_vos_scale voltage_scale; /**< LDO/SMPS Voltage scale used for this frequency. */
	enum pwr_sys_mode power_mode;     /**< LDO/SMPS configuration for device. */
	uint8_t smps_level;               /**< If using SMPS, voltage level to set. */
};
```

我们对其进行逐项解释：

- sysclock_source：系统时钟源选择，对应上面图片中的 System Clock Mux 部分，一般来说，我们都是选择 PLL 作为系统时钟源。可选参数如下：
  - RCC_PLL：选择 PLL 作为系统时钟源，这个是常用的配置。
  - RCC_HSE：选择 HSE 作为系统时钟源。
  - RCC_HSI：选择 HSI 作为系统时钟源。
  - 其实还有一些参数，但我觉得不能用在系统时钟源的选择上，所以就不一一列举了。

- pll_source：PLL 时钟源选择，对应图片中的 PLL Clock Mux 部分，我们通常选择 HSE 作为 PLL 时钟源，可选参数如下：
  - RCC_PLLCKSELR_PLLSRC_HSE：选择 HSE 作为 PLL 时钟源，有外部晶振的情况下，我们通常选择这个配置。
  - RCC_PLLCKSELR_PLLSRC_CSI：选择 CSI 作为 PLL 时钟源，CSI 是比较新的单片机（如 H7、U5）引入的 4 MHz 的内部时钟源，功耗低，但精度不高，如果说有低功耗方面的需求，可以考虑这个配置。
  - RCC_PLLCKSELR_PLLSRC_HSI：选择 HSI 作为 PLL 时钟源，HSI 是一个 64 MHz 的内部时钟源，单片机上电默认就是这个时钟源，在没有外部晶振的情况下，我们通常选择这个配置（精度也不是很高）。

- hse_frequency：HSE 频率，是一个 uint32_t 的变量，在配置的时候我们直接填写 HSE 的频率即可（如 25000000U），如果选择的是 HSI，则填写 0。

- pll1、pll2、pll3：PLL 配置，对应图片中 PLL Source Mux 出来之后分叉出来的三条线，这三条线大同小异，我们以 pll1 为例，逐项解释：
  - divm：PLL 预分频因子，对应图片中的 DIVM1，图片里面是几，我们赋值的值就是几，比如图片中是 5，我们赋值 5 即可。
  - divn：PLL 倍频因子，对应图片中的 DIVN1，图片里面是几，同理，图片中是 192，我们就是赋值 192 即可。
  - divp、divq、divr：PLL 后分频因子，根据 pqr 即可在锁相环上输出三个不同的时钟，对应图片中的 DIVP1、DIVQ1、DIVR1，图片里面是几，我们赋值几即可。
    - divp：该通道的输出常用于系统时钟。
    - divq：该通道的输出常用于 USB、SDMMC、ETH 等外设。
    - divr：该通道的输出常用于 SPI、ADC、DAC 等外设。

- core_pre：核心时钟预分频因子，对应图片中的 D1CPRE Prescaler，决定了核心时钟的频率，可选参数如下：
  - RCC_D1CFGR_D1CPRE_BYP：不进行分频，对应于图片中的 1 分频。
  - RCC_D1CFGR_D1CPRE_DIV2：2 分频。
  - RCC_D1CFGR_D1CPRE_DIV4：4 分频。
  - RCC_D1CFGR_D1CPRE_DIV8：8 分频。
  - RCC_D1CFGR_D1CPRE_DIV16：16 分频。
  - ...：其他分频因子。

- hpre、ppre1、ppre2、ppre3、ppre4：和核心时钟预分频因子类似，其意义在上面展示结构体 rcc_pll_config 的代码中标有注释，在图片中都分别有对应的配置可以参考，要注意图片中类似于 D2PPRE1 这样的名字，在 libopencm3 中，对应的参数是 RCC_D2CFGR_D2PPRE_DIV2，要注意区分。

- flash_waitstates：闪存等待周期，这个在图片中还真没对应的，可以生成一份同样配置的 MDK 工程，在 SystemClock_Config 函数中，有如下代码：

```c
  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_4) != HAL_OK)
  {
    Error_Handler();
  }

```

可以看到函数的参数是 FLASH_LATENCY_4，这个参数对应于 libopencm3 中的 FLASH_ACR_LATENCY_4WS，所以我们赋值为 FLASH_ACR_LATENCY_4WS 即可。

- voltage_scale：电压缩放，也是新系列的单片机中出现的东西，缩放影响内核电压与最高主频，0 为高档位，档位越高，功耗越大，但主频越高。同样在 SystemClock_Config 函数中，有如下代码：

```c
  __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE0);

  while(!__HAL_PWR_GET_FLAG(PWR_FLAG_VOSRDY)) {}

```

可以看到函数的参数是 PWR_REGULATOR_VOLTAGE_SCALE0，这个参数对应于 libopencm3 中的 PWR_VOS_SCALE_0，所以我们赋值为 PWR_VOS_SCALE_0 即可。

- power_mode：电源模式，同样是新系列带来的新功能，可以选择 LDO 或者 SMPS，LDO 简单易用，但功耗较大，SMPS 功耗低，但需要外部电路支持。同样在 SystemClock_Config 函数中，有如下代码：

```c
  HAL_PWREx_ConfigSupply(PWR_LDO_SUPPLY);
```

我们和 MDK 工程保持一致，所以也是选择 LDO 的供电方式，对应于 libopencm3 中的 PWR_SYS_LDO，所以我们赋值为 PWR_SYS_LDO 即可。

- smps_level：如果选择的是 SMPS，则需要设置电压等级，libopencm3 也提供了对应的选项，如 PWR_CR3_SMPSLEVEL_VOS，我没没有使用 SMPS，所以这里设置为什么都没关系。

在配置完参数之后，我们调用 rcc_clock_setup_pll(&pll_config); 即可进行系统时钟的初始化，该函数会根据我们配置的参数，设置到寄存器中，完成系统时钟的初始化。

> 对于时钟的初始化来说，执行顺序其实是一个值得考虑的事情，先初始化什么，后初始化什么，都是有考量的，不过 libopencm3 已经做好这一步了，如果你对执行的顺序有所疑惑，可以查看 libopencm3 源码，在libopencm3/lib/stm32/h7/rcc.c中，可以看到 rcc_clock_setup_pll 函数的执行顺序，对于理解时钟的初始化过程，有很大的帮助。

> 使用第三方库，需要多看源码，多找定义，多翻阅文档，多动手实践，才能熟练使用。

### SysTick 延时

众所周知，HAL 库的 HAL_Delay 是基于 SysTick 实现的，延时/计时对于单片机程序来说，是一个非常重要的功能，所以 SysTick 定时器的初始化，也是我们必须要做的。

因为 SysTick 是 ARM 内核提供的，并不是 STM32 系列单片机独有的，所以在初始化思路上来说，你可以借鉴任何一个 ARM 内核处理器的相关代码，我这里还是以 CubeMX 生成的代码作为参考，进行滴答定时器的初始化。

在 HAL_Init 这个函数中，有如下代码：

```c
  if(HAL_InitTick(TICK_INT_PRIORITY) != HAL_OK)
  {
    return HAL_ERROR;
  }

```

这个函数实现了 SysTick 的初始化，主要包含了如下几步：
- SysTick 时钟源选择。
- SysTick 重装值设置。
- SysTick 计数值清零。
- SysTick 中断优先级设置。
- SysTick 中断使能。
- SysTick 使能。

对应于 libopencm3 中的代码如下：

```c
void systick_init(uint32_t ticks)
{
  systick_set_clocksource(STK_CSR_CLKSOURCE_AHB);

  systick_set_reload((rcc_get_bus_clk_freq(RCC_CPUCLK) / ticks) - 1UL);

  systick_clear();

  nvic_set_priority(NVIC_SYSTICK_IRQ, 15);

  systick_interrupt_enable();
  systick_counter_enable();
}

```

SysTick 可以正常工作之后，我们写好中断和延时函数：

```c
volatile uint32_t systick = 0;

// 进入低功耗模式
static inline __attribute__((always_inline)) void __WFI(void)
{
  __asm volatile("wfi");
}

void user_delay_ms(uint32_t ms)
{
  uint32_t start = systick;
  while (systick -  start < ms)
  {
    __WFI();
  }
}

// SysTick 定时器中断处理函数
void sys_tick_handler(void)
{
  systick++;
}

```

这样就能愉快用 user_delay_ms() 进行毫秒级延时了。

### 补充：延时时间展示

那么我们这样初始化之后，到底对不对呢，能不能实现延时的功能呢？我写了一个简单的程序，在无限循环中，每隔 100ms，翻转一次 PE3 引脚，如下：

```c
  while (1)
  {
    user_delay_ms(100);
    gpio_toggle(GPIOE, GPIO3);
  }
```

然后我们使用逻辑分析仪，采集 PE3 引脚的电平，如下：

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/11/08/690f09e4d4fb9.png"/>
</center>

可以看到，PE3 引脚高电平+低电平的持续时间为200.021 ms，结合逻辑分析仪和定时器本身的误差，我们可以认为，这个延时是准确的。

## GPIO

### GPIO 初始化

绕了一大圈才回到主题，接下来的事情反而很简单，我们只需要像标准库一样，进行时钟使能，然后配置引脚的模式即可，libopencm3 中，GPIO 的配置，需要分两步进行，第一步是配置引脚的模式，第二步是配置引脚的属性。

这一部分函数名就可以表达出其功能，所以直接上代码：

```c
void user_gpio_setup(void)
{
  rcc_periph_clock_enable(RCC_GPIOE);
  gpio_mode_setup(GPIOE, GPIO_MODE_OUTPUT, GPIO_PUPD_NONE, GPIO3);
  gpio_set_output_options(GPIOE, GPIO_OTYPE_PP, GPIO_OSPEED_2MHZ, GPIO3);

  gpio_clear(GPIOE, GPIO3);

  PWR_CR1 |= PWR_CR1_DBP;
  rcc_periph_clock_enable(RCC_GPIOC);
  gpio_mode_setup(GPIOC, GPIO_MODE_INPUT, GPIO_PUPD_PULLDOWN, GPIO13);
}

```

基本思路就是使能时钟->配置引脚模式->配置引脚属性，然后就可以使用对应的引脚了。

但是，在 STM32 中，PC13 - PC15 这三个引脚是比较特殊的，有以下一些需要注意的地方：
- 他们属于低速I/O，最大只能工作在 2MHz。
- 驱动能力弱，一般用作输入或者低速输出（所以核心板用这个引脚作为按键还是有一定合理性的）。
- 在 STM32H7 中，这三个引脚属于备份域（Backup Domain），为了防止程序误操作，也为了保持主电源掉电之后的可靠性，单片机在默认上电的时候是禁止访问备份域的，所以我们需要在初始化的时候，先使能备份域的访问权限，也就是在代码中，先执行 PWR_CR1 |= PWR_CR1_DBP; 这一行代码。

> 可能你会问，为什么在使用 HAL 库的时候没有这样的问题，因为 HAL 库在 SystemClock_Config 函数中，调用了 HAL_RCC_OscConfig，而这个函数中进行了备份域的写使能：PWR->CR1 |= PWR_CR1_DBP;，所以在使用 HAL 库的时候，不需要手动写使能备份域的代码。

### GPIO 读写

和上一篇的代码逻辑一致，当按键按下的时候，进行 LED 的翻转，代码如下：

```c
int main(void)
{
  ...

  user_gpio_setup();

  while (1)
  {
    if(gpio_get(GPIOC, GPIO13))
    {
      user_delay_ms(20);
      if(gpio_get(GPIOC, GPIO13))
      {
        gpio_toggle(GPIOE, GPIO3);
      }
      while(gpio_get(GPIOC, GPIO13));
    }
  }

  return 0;
}

```

如果要直接控制电平，可以使用 gpio_set 和 gpio_clear 函数，如 gpio_set(GPIOE, GPIO3); 和 gpio_clear(GPIOE, GPIO3);。

对于读取函数来说，虽然看起来没有什么问题，但是要注意 gpio_get 函数返回的并不是电平的逻辑值，而是包含所选引脚状态的位掩码。例如：
- 当 PC13 引脚为高电平时，gpio_get(GPIOC, GPIO13) 返回 0x2000（即 1 << 13），而不是单纯的 1。
- 当 PC13 为低电平时，返回值才是 0。

因此，在读取引脚电平的时候，应当判断它是不是 0，才能保证引脚电平的正确性。

## 总结

这一篇文章还是比较啰嗦了，本来只是想点亮一个灯，结果为了把地基打稳，写了大半篇。

用 libopencm3 开发 STM32H7，最大的心得是：
- 要习惯自己写初始化，不像 HAL 那样一口气帮你包好。
- 多看源码，多对照 CubeMX，少走弯路。
- 习惯把工程目录分层，方便后面扩展。
