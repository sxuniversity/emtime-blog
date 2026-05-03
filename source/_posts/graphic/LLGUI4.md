---
title: 玲珑GUI-移植修改
categories: graphic
date: 2022-11-01 15:12:17
updated: 2022-11-01 15:12:17
---

玲珑GUI本身是脱离平台的，移植其实不复杂，主要就是修改 LL_Config.c 和 LL_Config.h 两个文件；本篇就来把这些点一一说明，照着配置下来，跑起来基本不成问题；

> 注意：
💡 因为这个也算是新手教程，所以我会以简单的方式来实现功能，如果有一些没提到的地方，可能就需要你自己摸索了；
💡 同时，一切的前提是**你屏幕驱动已经写好了**，玲珑GUI并没有内置任何屏幕的驱动，无法完成点亮屏幕的功能；

## 1. 创建玲珑GUI工程

要开始使用玲珑GUI，首先需要在 MDK 工程中创建一个新的 GUI 项目；以下是详细步骤：

1. **打开 MDK 工程**：
   - 启动你的 MDK（Keil）开发环境，打开或创建一个新的工程；

2. **启动 LLGUIBuilder**：
   - 在工程中启动 LLGUIBuilder 工具；
   - 在弹出的窗口中，选择与你的屏幕分辨率相匹配的选项，然后点击确定（不需要选择路径）；

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/12/6849bb79e9024.jpg"/>
</center>

> **注意**：如果没有找到与你屏幕匹配的分辨率，请继续阅读**添加分辨率**部分；如果分辨率匹配，可以忽略此部分；

### 添加分辨率

如果 LLGUIBuilder 中没有你需要的分辨率，按照以下步骤添加：

1. **关闭软件**：
   - 关闭引导窗口后，软件本体也会弹出，确保将其一并关闭；

2. **编辑配置文件**：
   - 找到玲珑GUI软件的安装目录，使用记事本打开 deviceType.ini 文件；
   - 修改 count 值以增加新的分辨率；
   - 在文件末尾，按照现有格式添加你的屏幕分辨率和方向；

   <p style="display:flex; justify-content:center; gap:20px;">
     <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/12/6849bbb42096c.jpg" width="200" />
     <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/12/6849bbb56e247.jpg" width="200" />
   </p>

3. **重新启动 LLGUIBuilder**：
   - 重新打开 LLGUIBuilder，选择新添加的分辨率；

---

### 生成代码

1. **添加新页面**：
   - 进入 LLGUIBuilder 后，右键点击 UI File，选择添加新文件；
   - 为你的第一个页面命名；

2. **设计界面**：
   - 在右侧界面上拖放控件，设计你的界面布局；

3. **生成代码**：
   - 点击绿色三角按钮生成代码（不需要设置路径，直接确认即可）；
   - 生成完成后，返回 Keil，在弹出的对话框中选择“是”以重载工程；

   <center>
     <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/12/6849bbeb8bb4f.jpg"/>
   </center>

这样，你的工程中就会包含由玲珑GUI生成的代码；

<center>
  <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/12/6849bbeacfa14.jpg"/>
</center>

## 2. 必须配置的部分

### 画点函数 llCfgSetPoint

这是最核心的一个函数，玲珑GUI的绘图基本都依赖它：

```c
void llCfgSetPoint(int16_t x, int16_t y, llColor color)
{
    // 调用你屏幕的画点函数即可
    // 如果是单色屏，可以通过判断 color 是否为黑色（或者别的颜色），来决定是否画点
    // color是uint16的变量，是RGB565的格式，可以使用RGB_CONVERT(R, G, B)的方式将颜色转为llColor的格式
    // 如果你屏幕的颜色和玲珑GUI的一样，那么就不用做什么了
    // 如果不一样，那么就需要你自己把llColor类型转为你屏幕的函数能接受的类型了

    LCD_Point(x, y, color);
}
```

### 单色矩形填充 llCfgFillSingleColor

> 注意，在测试阶段或者说你是新手，可以把USE_USER_FILL_MULTIPLE_COLORS改为0，这样使用的就是单色矩形填充

虽然有些屏幕只需要画点也能撑起来界面，但如果你是彩屏，或者要支持某些控件的高效渲染，还是建议实现这个：

```c
void llCfgFillSingleColor(int16_t x0, int16_t y0, int16_t x1, int16_t y1, llColor color)
{
    // 填充矩形区域
    // 如果你是用循环画点的方式来实现的，那么可以简单注意一下循环里的 <=，否则容易出现“最后一行没画”的情况
    // 有时候需要注意，你的填充函数可能是起始坐标+对应长宽的形式，所以需要自己计算一下，如下：
    LCD_Fill(x0, y0, x1 - x0 + 1, y1 - y0 + 1, color); 
}
```

### 设置屏幕分辨率

有时候上位机选择了分辨率，但代码里并不会自动同步，需要手动在 LL_Config.h 中修改两个宏：

```c
#define LL_MONITOR_WIDTH  240
#define LL_MONITOR_HEIGHT 320
```

这两个值应与你的 LCD 实际分辨率一致，且与 LLGUIBuilder 中的方向一致（竖屏/横屏），否则坐标映射容易出问题；

---

## 3. 可选配置部分

### 触摸输入 llCfgClickGetPoint

如果你的项目用不上触摸屏，可以直接返回 false；如果要用触摸功能，则需要实现这个函数：

```c
bool llCfgClickGetPoint(int16_t *x, int16_t *y)
{
    bool touchState = false;

    // 下面这三句根据实际情况填写，按同样的触摸函数来说，一句话就能获得状态和按下的x，y坐标了
    touchState = 你自己的触摸函数，返回值为真则有触摸发生;
     *x = 从你的触摸函数中获得的x;
     *y = 从你的触摸函数中获得的y;

    if((touchState != 0)&&(*x != 0xffff)&&(*y != 0xffff))
    {
        touchState = true;
    }
    else
    {
        touchState = false;
        *x = 0xffff;
        *y = 0xffff;
    }
    return touchState;
}
```

### RTC接口

玲珑GUI内置了和时间相关的控件；如果你实现了对应的 RTC 接口函数，那么这些控件就能自动获取系统时间并更新显示；也就是说，你不需要手动刷新它们，对应的部分会自己刷新；

#### llGetRtc 读取时间

读取当前时间，GUI 会通过这个函数获取系统时间；你需要将年、月、日、时、分、秒、星期依次写入 readBuf：

> ▪ 读 buf 的内容格式：年(2字节) 月 日 时 分 秒 周（共 8 字节）

```c
// yy yy mm dd hh mm ss ww
void llGetRtc(uint8_t *readBuf)
{
    uint16_t year;
    uint8_t month, day, hour, minute, second, week;

    bm8563_readDate(&year, &month, &day);
    bm8563_readTime(&hour, &minute, &second);
    week = bm8563_getWeek(year, month, day);

    readBuf[0] = year >> 8;
    readBuf[1] = year;
    readBuf[2] = month;
    readBuf[3] = day;
    readBuf[4] = hour;
    readBuf[5] = minute;
    readBuf[6] = second;
    readBuf[7] = week;
}
```

---

#### llSetRtc 设置时间

当你在时间控件中修改了时间（比如通过一个日期选择器设置新时间），GUI 会调用这个函数，把用户设置的值传给底层；

> ▪ 写 buf 的内容格式：年(2字节) 月 日 时 分 秒（共 7 字节）

```c
// yy yy mm dd hh mm ss
void llSetRtc(uint8_t *writeBuf)
{
    bm8563_writeDate((writeBuf[0] << 8) + writeBuf[1], writeBuf[2], writeBuf[3]);
    bm8563_writeTime(writeBuf[4], writeBuf[5], writeBuf[6]);
}
```

---

> RTC 模块使用哪一款由你自行决定；上面的示例以 BM8563 为例，你也可以替换为其他 RTC 芯片如 DS1302、DS3231 或系统内部 RTC 等；
你可能还需要考虑设置时区、夏令时或系统时间同步等问题，具体取决于你的 RTC 模块与项目需求；
