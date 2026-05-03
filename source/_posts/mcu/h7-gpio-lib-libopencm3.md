---
title: STM32H7开发笔记(六):GPIO-输入处理-libopencm3库实现
categories: mcu
---

在上一篇中，我们使用 HAL 库实现了 easy_button 的按键处理，这一篇我们使用 libopencm3 库实现同样的功能。

大体上思路和 HAL 库实现是一样的，只是使用 libopencm3 库的 API 来实现。

本文代码仓库：[stm32h7-libopencm3](https://git.orangetime.top/EMTime/stm32h7-libopencm3)，不想看我啰嗦的，可以直接拉代码，目录结构清晰，还写了详细注释。

> 本文对应于仓库中的 2gpio-lib 文件夹

## 工程导入

在这一次的代码中，我进行了一些管理上的变更，也算是一个新的工程，所以可以新建一个文件夹，我来引导大家一步一步来。

文件夹还是需要创建在和 stm32h7-libopencm3 同级的目录下，我们可以从之前的工程中复制 cortex-m-generic.ld 以及 user 文件夹到当前工程下，这些都是和之前保持一致的。

### 下载 easy_button

- Github仓库：https://github.com/bobwenstudy/easy_button

> 考虑到国内网络问题，部分读者可能无法访问 Github，所以我自己部署了 Gitea，将 easy_button 仓库同步到了我的 Gitea 服务器上，地址：https://git.orangetime.top/EMTime/easy_button

### 添加 easy_button 文件

我们要新增第三方库，这些库的源码，我建议单独放到一个文件夹中，这样方便管理，所以新建一个 lib 文件夹，在里面再创建 ebtn 文件夹，我们将下载的 easy_button 仓库中的所有文件复制到这个文件夹中。

### xmake 配置

我主要修改了规则的位置和对于库的依赖，整体的配置如下：

```lua
-- 工程名
set_project("stm32h7")

-- 定义工具链，确定位置和名称
toolchain("arm-none-eabi")
    set_kind("standalone")
    set_sdkdir("/home/time/doc/mybin/arm-none-eabi")
toolchain_end()

-- 让工程使用我们定义的工具链
set_toolchains("arm-none-eabi")

-- 设置平台与架构
set_plat("MCU")
set_arch("ARM Cortex-M7")

-- 设置编译优化等级与编译模式
set_optimize("none")
set_symbols("debug")

-- 设置架构 & FPU
add_cxflags("-mthumb", "-mcpu=cortex-m7", "-mfpu=fpv5-d16", "-mfloat-abi=hard", {force = true})
add_asflags("-mthumb", "-mcpu=cortex-m7", "-mfpu=fpv5-d16", "-mfloat-abi=hard", {force = true})
add_ldflags("-mthumb", "-mcpu=cortex-m7", "-mfpu=fpv5-d16", "-mfloat-abi=hard", {force = true})

-- 设置全局头文件路径位置 (libopencm3)
add_includedirs("../../libopencm3/include")

-- 设置全局链接库 (libopencm3)
add_linkdirs("../../libopencm3/lib")
add_links("opencm3_stm32h7")

-- 设置全局额外链接选项
add_ldflags("-T./cortex-m-generic.ld", "--static", "-nostartfiles", "-Wl,--gc-sections", {force = true})
add_syslinks("c", "gcc", "nosys")

-- 设置全局宏定义
add_defines("STM32H7")

-- 设置全局用户头文件搜索路径
add_includedirs("user/inc")

-- 目标程序
target("gpio-lib")
    -- 设置目标类型为二进制，就是编译输出可执行文件
    set_kind("binary")
    add_files("user/src/*.c")
    set_targetdir("$(projectdir)/bin")

    -- 依赖于 lib 目标，这个目标在后面定义
    add_deps("lib")

    on_load(function (target)
        target:set("filename", target:name() .. ".elf")
        -- 生成 map 文件并带 cref，显示 cross reference
        target:add("ldflags",
            "-Wl,-Map=" .. path.join(target:targetdir(), target:name() .. ".map") .. ",-cref",
            {force = true}
        )
    end)

    -- 生成额外文件
    after_build(function (target)
        local elf = target:targetfile()
        local bindir = target:targetdir()
        local name = target:name()

        -- 常见格式
        os.execv("arm-none-eabi-objcopy", {"-Obinary", elf, path.join(bindir, name .. ".bin")})
        os.execv("arm-none-eabi-objcopy", {"-Oihex",   elf, path.join(bindir, name .. ".hex")})
        os.execv("arm-none-eabi-objcopy", {"-Osrec",   elf, path.join(bindir, name .. ".srec")})
        os.execv("arm-none-eabi-objdump", {"-S", elf}, {stdout = path.join(bindir, name .. ".list")})

        -- 🔥 符号表 (对应 MDK Image Symbol Table)
        os.execv("arm-none-eabi-nm", {"-n", elf}, {stdout = path.join(bindir, name .. ".sym")})

        -- 🔥 段大小统计 (对应 MDK Image component sizes)
        os.execv("arm-none-eabi-size", {"-B", elf}, {stdout = path.join(bindir, name .. ".size")})
    end)

    on_clean(function (target)
        os.rm("build")
        os.rm("bin")
    end)

-- lib 目标，负责整合所有第三方库 (libopencm3除外)
target("lib")
    -- 设置目标类型为phony，即伪目标，不会生成文件，仅负责将不同的目标整合到一起
    set_kind("phony")

    -- 依赖于 easybutton 目标，之后有新的目标，都可以添加到这里
    add_deps("lib-ebtn")

target("lib-ebtn")
    -- 设置目标类型为静态库，依赖于他的目标会链接这个库
    set_kind("static")
    add_includedirs("lib/ebtn", {public = true})
    add_files("lib/ebtn/*.c")
```

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
    return gpio_get(GPIOC, GPIO13) == GPIO13;
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
  return systick;
}
```

在前面 libopencm3 的文章中，我们初始化了 SysTick，并且实现了对应的延时功能，在这里我们包含 systick.h 就可以直接使用 systick 变量，获取系统运行的时间（毫秒）。

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
      gpio_toggle(GPIOE, GPIO3);
    }
    //printf("[BTN %d] Clicked, count=%d\r\n", btn->key_id, ebtn_click_get_count(btn));
    break;
  case EBTN_EVT_KEEPALIVE:
    if(btn->key_id == USER_BUTTON1)
    {
      gpio_toggle(GPIOE, GPIO3);
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

## 总结

通过使用 easy_button 库，我们可以很方便的实现按键的点击、长按、连击等功能，并且可以很方便的扩展到多个按键，只需要在 ebtn_btn_t 数组中添加按键对象即可，非常方便。
