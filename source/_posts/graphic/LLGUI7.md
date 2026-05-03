---
title: 玲珑GUI-逻辑控制
categories: graphic

date: 2022-11-02 21:44:41
updated: 2022-11-02 21:44:41
---

除了图形界面上的显示布局，图形库的另一个核心任务，就是响应用户的操作——比如点击按键、输入文字等；这一部分，也就是逻辑控制的实现，是让“界面活起来”的关键；

这篇文章我们就来看看如何在玲珑GUI中实现控件事件的绑定，并完善图形界面的交互逻辑；

## 信号与事件绑定

在如图所示的地方，可以看到，有发送者，信号等东西；

<center>
  <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/12/684aa2ea78ee2.png"/>
</center>

这一页中最关键的部分，就是“连接”的概念；简单理解，它就像是在某个控件的某个动作和你自己写的函数之间架了一座桥；具体来说：
- 发送者：触发事件的控件，比如点击按键，那么按键就是发送者；
- 信号：发送者发送的事件，比如按下按键，那么按键发送的就是按下事件；
- 接收者：接收事件的控件，如果你的目标只是执行一段逻辑代码（比如设置某个变量），而不是操作其他控件，可以把接收者选为background；
- 动作函数：点击右边的小“重载”按钮，它会自动帮你生成一个带有发送者/接收者信息的函数名称；
- 代码：说实话，不太明白这里应该设置啥，所以我也没有动这里；
- 强制重写：勾选之后你之前的这个函数的内容就会被覆盖掉；

> 注意：如果你在生成代码后修改了发送者/接收者，原来的函数不会自动删除，需要你手动处理一下代码，避免冗余；

## 添加一个事件连接

点击「添加连接」，你就可以创建一个完整的信号-动作响应链；如下图：

<center>
  <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/12/684aa5a19339c.png"/>
</center>

<center>
  <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/12/684aa6c0438f0.png"/>
</center>

你可以看到，不同的控件支持不同的信号类型（这部分建议大家多点一点，每个控件支持的信号都列得很清楚）；比如 Button 支持按下、抬起、自锁状态下的切换等；

信号逻辑设置完成后，点击生成代码，我们就可以在 Keil 工程中看到对应的函数定义了；

<center>
  <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/12/684aa9d3846c4.png"/>
</center>

## 动作函数解析

在上面的图中可以看到，发送者就是Button0，接收者就是Text0，在代码中可以对Button0控件和Text0控件进行操作；

下面是一个典型的由事件绑定自动生成的函数示例：

```c
bool ui_mainAction_Button_0_pressed_Text_0(llConnectInfo info)
{
  //sender Button 0
  //receiver Text 0
  //return true -> stop signal transmission
  uint8_t display[10] = {0};

  // 因为info.sender就是Button0，所以这里直接强制转换类型并获取其键值
  sprintf((char*)display, "%d", ((llButton *)(info.sender))->keyValue);

  // 注意，这里使用的是p开头的函数，因为info.receiver是一个指针而不是控件ID，所以这里就用p开头的函数对控件进行操作
  pTextSetText(info.receiver, display);

  return false;
}
```

这个函数的意思非常直观：
- info.sender 是触发事件的控件（这里是 Button0）；
- info.receiver 是接收事件的控件（这里是 Text0）；
- 利用 sender 获取控件属性（比如键值），然后用 receiver 来更新显示内容；
- 使用 pTextSetText() 是因为 info.receiver 是一个指针；如果你用的是控件ID，那么就要用 nTextSetText()；
- 如果返回了true，那么信号就不会再传递了，简单理解的话就是这个函数就不会再触发了

通过玲珑GUI提供的逻辑控制功能，我们可以非常方便地将用户操作和代码逻辑进行绑定，实现真正可交互的嵌入式图形界面；

> 如果你熟悉 Qt 的 signal-slot 模型，那么玲珑GUI的“连接机制”你会感到非常熟悉；
> 如果你是第一次接触，建议多试试不同的信号和控件组合，慢慢你就能熟练掌握控件间的交互逻辑；
