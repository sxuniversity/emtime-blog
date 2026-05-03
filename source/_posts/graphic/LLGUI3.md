---
title: 玲珑GUI-工程设置
categories: graphic
date: 2022-11-01 11:27:31
updated: 2022-11-01 11:27:31
---

# 工程设置相关问题及解决方案

**先把这些注意事项看一遍，省得半路出问题不知道咋整**

- **使用 AC5 和 AC6 编译器的注意事项**

  - **AC5 编译器**  
    - 需要添加编译选项 --no-multibyte-chars；  
    - 出现 #94-D: the size of an array must be greater than zero 时，确保启用 GNU 扩展；  
    - 出现 error:#8: missing closing quote 错误时：  
      - 在编译选项中添加 --locale=english；  
      - 或者将文件编码保存为 **UTF-8 with BOM** 格式；  
    - 消除编译警告使用参数：
      ```
      --diag_suppress=1278,128,111,68,174,188,177
      ```

  - **AC6 编译器（作者推荐使用AC6）**  
    - C 标准选择 gnu99；  
    - 消除警告使用参数：
      ```
      -Wno-unused-value -Wno-excess-initializers
      ```

参数添加在图片下方的Misc Controls框即可
<center>
  <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/12/6849bb2080aaf.jpg" alt="插件截图" width="400">
</center>

- **控件显示不全**  
  - 调整配置文件（cfg 文件）的内存大小；  
  - 调整启动文件中的堆栈大小，例如统一改成 0x1000；

- **Win10 下 Keil 闪退问题**  
  - 在“此电脑”右键 → 管理 → 计算机管理 → 服务与应用程序 → 服务 → 找到“Windows许可证管理器服务”；  
  - 将启动类型设置为“自动”，并手动启动该服务；

- **llGuiInit() 初始化分配内存时硬件错误**  
  - 进入 D:\WinSoft\LLGUIBuilder\LingLongGUI 目录打开终端，执行 git pull 更新源码；  
  - 重新创建一个新的工程，不建议在已有工程上直接操作，重新创建更容易搞；
