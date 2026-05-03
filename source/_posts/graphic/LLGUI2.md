---
title: 玲珑GUI-软件安装
categories: graphic
date: 2022-11-01 10:34:12
updated: 2022-11-01 10:34:12
---

# LLGUIBuilder 软件安装说明

## 前提条件
- 需要安装 MDK（Keil）；
- 软件在[玲珑GUI仓库](https://gitee.com/gzbkey/LingLongGUI)的tools下面，**登录账号**下载即可；

## 安装步骤

1. **下载安装包**
   - 将软件解压或安装到任意目录（不必安装到 Keil 的安装目录），例如：
     ```
     D:\WinSoft\LLGUIBuilder
     ```

2. **运行软件**
   - 双击打开 LLGUIBuilder.exe；

3. **插件自动添加**
   - 安装成功后，软件会尝试自动添加 Keil 插件；
   - 如果未成功自动添加，需要手动添加插件；

4. **手动添加插件到 Keil（如果Keil的Tools中没有LLGUIBuilder则需要这样做）**
   - 打开 Keil，依次进入：
     ```
     Tools -> Customize Tools Menu...
     ```
   - 添加新的工具项：
     - **Name**: LLGUIBuilder
     - **Command**: D:\WinSoft\LLGUIBuilder\LLGUIBuilder.exe（根据你安装的目录来）
     - **Arguments**: keil "#P" "!H" "#X"

5. **插件设置说明**
   - **Run Independent** 选项表示程序独立运行；
   - 如果插件无法启动，先取消勾选该选项启动一次，再勾选回来；
   - 如果不勾选该选项，则必须关闭 Builder 软件后才能继续操作 Keil；

<center>
  <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/12/6849b63848b12.jpg" width="400" />
</center>

<!-- <p align="center">
  <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/12/6849b63848b12.jpg" alt="插件截图" width="400">
</p> -->
