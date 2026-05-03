---
title: 玲珑GUI-杂项补充
categories: graphic

date: 2022-11-03 10:13:11
updated: 2022-11-03 10:13:11
---

## 页面跳转

使用 LLJumpPage(页面ID) 进行页面切换，其中页面ID通常在 LL_User.h 文件中定义，例如：
```c
#define PAGE_UI_MAIN    0
```

## 除了控件API，别的都可以在如图所示的地方找找

<center>
  <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/12/684ac3b48a89a.png"/>
</center>

## 图片库

有时候你不仅仅需要图片控件显示图片，可能还需要切换，但是你发现想要切换的图片并不在image.bin中，这时候就需要把图片放到图片库中，这样image.bin就会有你想要的图片的数据了

<center>
  <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/12/684ac4c51d037.png"/>
</center>
