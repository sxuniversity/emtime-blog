---
title: 实现 USB Composite 设备：在同一接口上模拟多个设备（含 FatFS U盘）
categories: mcu

date: 2024-10-27 09:41:19
updated: 2024-10-27 09:41:19
---

仓库：https://github.com/alambe94/I-CUBE-USBD-Composite
参考：https://blog.csdn.net/u012936480/article/details/137411833

在嵌入式开发中，USB 是一个非常重要的外设接口，它不仅可以用来调试、传输数据，还能为产品增加多样化的功能体验；而如果你希望在一个 USB 接口上同时实现多个设备功能，比如**既能当串口调试工具、又能当 U 盘存储设备，甚至还能充当 HID 控制器**，那你就需要掌握 USB Composite（复合设备） 的实现方式；

本篇博客将结合[前文实现的 FatFS 文件系统](/2024/10/26/mcu/FatFS)功能，进一步介绍如何在 STM32 平台上实现 USB Composite 设备；我们将以 **U盘 + 虚拟串口（CDC）** 为例，完整讲解如何配置多个接口；通过这一实践，你将掌握一线多用的 USB 技巧，为你的嵌入式设备赋予更多“身份”和可能性；

本文采用了 GitHub 上一个开源项目（I-CUBE-USBD-Composite）作为基础，该项目已经实现了 USB Composite 的关键功能；我们基于它，以简单快捷的方式完成了 U 盘 + CDC 的复合设备实现；

## 文件下载

[AL94.I-CUBE-USBD-COMPOSITE.1.0.3.pack](https://webdisk.orangetime.top/#s/EAMFFqgT)
**提取密码: zKxR3**

链接中是一个 pack 文件，但是这个文件并不是给 MDK 使用的，而是给 STM32CubeMX 使用的，我们需要在 STM32CubeMX 中导入这个文件，作为组件库引入我们的工程中；

导入文件步骤如下：

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/25/685ae35335f95.png"/>
</center>

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/25/685ae36f1d0a5.png"/>
</center>

## 工程引入

导入完成之后，我们还需要在 CubeMX 中勾选这个组件库，选择所需的功能 ，如下：

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/25/685ae3b35928d.png"/>
</center>

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/07/04/6867acf5ea71a.png"/>
</center>

其中，Core 是必须勾选的，如果你要同时使用 CDC 和 U 盘，那么不仅 CDC_ACM 和 MSC_BOT 都需要勾选，而且还**必须**勾选 COMPISITE,否则 USB Composite 功能将无法正常使用；

在导入组件库之后，我们还需要在 CubeMX 中配置一下 USB 相关的配置，如下：

- 配置 USB 外设的中断，否则 USB 外设将无法正常工作

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/25/685ae5085d7e9.png"/>
</center>

- 配置组件库

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/25/685ae5bc20669.png"/>
</center>

博客使用 F103 作为例子，所以还需要选择最后一项；

## 代码修改

配置好之后，我们直接使用 CubeMX 生成代码即可，代码中需要修改的地方不多，主要是初始化和对应 USB 功能的逻辑，如下：

### 引入工程

```c
#include "usb_device.h"
#include "usbd_cdc_acm_if.h"

int main(void)
{
    ...
    MX_USB_DEVICE_Init();
    ...

    while(1)
    {
        ...
    }
}
```

### 功能逻辑

每一个 USB 功能都对应一个文件，比如 CDC 功能对应的是 usbd_cdc_acm_if.c，我们只需要在对应的文件中添加我们需要的逻辑即可，因为在[之前的博客](/2024/10/26/mcu/FatFS)中已经实现了 FatFS 文件系统，所以我们今天主要将重点放在 CDC 与 U 盘的处理逻辑上；

#### CDC 串口

CDC（Communication Device Class）是一种虚拟串口接口，库文件中已经封装好了基本的收发逻辑：发送数据时直接调用发送函数，接收到数据时则会自动调用接收回调函数；

> 听起来一切都很完美，但这背后其实还隐藏着一个关键问题；

答案是：**理解 USB 的传输机制，特别是数据包的处理方式**；

USB 在底层按“包（Packet）”传输，CDC 使用的传输类型为 Bulk（批量传输），**每个端点的最大数据包大小（Max Packet Size）**对于全速设备通常为 64 字节；因此，大于 64 字节的数据会被分为多个包发送；这意味着：
- 如果我们发送的数据长度小于等于 64 字节，可以一包发完；
- 如果发送的数据长度大于 64 字节，则需要拆分为多个数据包进行分批发送；

**在发送方面**，库通常已经封装好了自动拆包的逻辑；我们调用发送函数时，只需要传入需要发送的缓冲区和长度，不必关心包的边界或分段细节；

**接收就没那么轻松了；**

虽然 USB 主机会自动将数据拆分为多个包发送，但在设备端的接收回调函数中，我们**每次只能获取一包数据**，这意味着：如果一条完整的数据大于 64 字节，必须**手动将这些数据包拼接起来**，然后再统一处理；

那么，**我们该如何判断数据是否接收完毕**？

我的做法是引入了一个“**接收超时判断**”机制；每当接收到一包数据时，就刷新一次超时计时器；如果在某个预设时间内没有再接收到新的数据包，就说明数据应该已经接收完毕，可以进行处理了（算是串口空闲的简单实现思路）；

这种方法简单实用，特别适合一些非实时、包长度不确定的串口协议；

如果你希望更稳妥地判断数据边界，也可以考虑使用协议头尾、长度字段等方式进行辅助判断；但在简单应用中，“超时判断法”是一个很实用的起点；

头文件 usbd_cdc_acm_if.h 中需要添加以下内容：

```c
// 相关宏定义
#define CDC_RX_BUF_SIZE    512   // 缓冲区大小，根据实际需求调整
#define CDC_TIMEOUT_MS     6     // 超过 6ms 没数据，认为一帧结束，结合实际一包发送的时间调整

// 相关函数声明
void process_usb_frame(uint8_t* data, uint16_t len);  // 处理接收到的 USB 数据帧
void check_usb_timeout(void);                         // 检测 USB 接收超时，超时后调用 process_usb_frame
```

源文件 usbd_cdc_acm_if.c 中需要添加以下内容：

```c
// 相关变量定义
uint8_t cdc_rx_buffer[CDC_RX_BUF_SIZE];
uint16_t cdc_rx_index = 0;
uint32_t cdc_last_recv_tick = 0;
uint8_t  cdc_receiving = 0;

// 修改 CDC_Receive 函数
static int8_t CDC_Receive(uint8_t cdc_ch, uint8_t *Buf, uint32_t *Len)
{
  /* USER CODE BEGIN 6 */
  // 拼接到总缓冲区
  if(cdc_rx_index + *Len < CDC_RX_BUF_SIZE)
  {
      memcpy(&cdc_rx_buffer[cdc_rx_index], Buf, *Len);
      cdc_rx_index += *Len;
      cdc_last_recv_tick = HAL_GetTick(); // 更新时间戳
      cdc_receiving = 1;
  }
  else
  {
      // 缓冲区溢出，复位
      cdc_rx_index = 0;
      cdc_receiving = 0;
  }

  USBD_CDC_SetRxBuffer(cdc_ch, &hUsbDevice, &Buf[0]);
  USBD_CDC_ReceivePacket(cdc_ch, &hUsbDevice);
  return (USBD_OK);
  /* USER CODE END 6 */
}

// 处理 USB 数据帧
void process_usb_frame(uint8_t* data, uint16_t len)
{
    // 举例：打印字符串帧
    data[len] = '\0'; // 保证以 \0 结尾
    CDC_Transmit(0, data, len);  // 回显
}

// 检测 USB 接收超时
void check_usb_timeout(void)
{
    if(cdc_receiving)
    {
        if(HAL_GetTick() - cdc_last_recv_tick > CDC_TIMEOUT_MS)
        {
            // 超时，处理一帧数据
            process_usb_frame(cdc_rx_buffer, cdc_rx_index);
            cdc_rx_index = 0;
            cdc_receiving = 0;
        }
    }
}
```

在 main.c 的 while 循环中添加 check_usb_timeout 函数即可；

> **注意**：
> - 这套库中的用户代码部分是没有作用的，你每次使用 CubeMX 生成代码都会覆盖掉你写的代码，无论你写在注释部分或者非注释部分，重新生成之后都会消失，所以可以找到 CubeMX 存储的位置，找到库文件，手动修改库文件，这样每次生成的代码都会是有处理逻辑的代码；
> 我的存储位置如下，在 C:\Users\EMTime\STM32Cube\Repository\Packs\AL94\I-CUBE-USBD-COMPOSITE\1.0.3\Middlewares\Third_Party\COMPOSITE\App 中即可找到对应的文件，打开进行修改即可；
> 放心，USB 的逻辑部分不像别的代码一样需要频繁修改，你写好之后，大部分情况可以直接使用；

<center>
   <img src="https://emtime-picture.cn-nb1.rains3.com/2025/06/25/685bb2865f2ed.png"/>
</center>

> **注意**：虽然虚拟串口数量可以在 CubeMX 中配置，但由于 I-CUBE-USBD-COMPOSITE 库在生成代码时会覆盖相关配置，这项设置在实际工程中并不会生效；真正起作用的是头文件 AL94.I-CUBE-USBD-COMPOSITE_conf.h 中的 _USBD_CDC_ACM_COUNT 宏定义，该值才是决定虚拟串口个数的关键参数；

#### MSC U盘

因为之前已经实现了 FatFS 文件系统，所以关于文件系统的部分不过多赘述，USB 模拟 U 盘仅需实现 usbd_msc_if.c 中下面几个函数即可：

```c
int8_t STORAGE_GetCapacity(uint8_t lun, uint32_t *block_num, uint16_t *block_size)
{
  /* USER CODE BEGIN 3 */

  // 如果使用我之前博客中提供的代码，则这里可以不用自己计算扇区数和扇区大小，和我写一样的即可

  // 总扇区数，和存储容量有关，扇区数 = 总容量 / 每个扇区大小，比如 16MB SPI Flash 的扇区数 = 16M / 4096 = 16 * 1024 * 1024 / 4096 = 4096，那么这里就是4096
  *block_num  = Flash_Sector_Count;

  // 每个扇区大小，和存储类型有关，如果是 SPI FLash，则是 4096，如果是内部 FLash 或者 SD 卡，则是 512
  *block_size = Flash_Sector_Size;

  return (USBD_OK);
  /* USER CODE END 3 */
}

int8_t STORAGE_Read(uint8_t lun, uint8_t *buf, uint32_t blk_addr, uint16_t blk_len)
{
  /* USER CODE BEGIN 6 */

  // 我之前博客中提供的读取函数如下所示，改成你自己的即可
  FLASH_RD_Block_Start(blk_addr * 4096);
  FLASH_RD_Block(buf, blk_len * 4096);
  FLASH_RD_Block_End();

  return (USBD_OK);
  /* USER CODE END 6 */
}

int8_t STORAGE_Write(uint8_t lun, uint8_t *buf, uint32_t blk_addr, uint16_t blk_len)
{
  /* USER CODE BEGIN 7 */

  for (uint16_t i = 0; i < blk_len; i++)
  {
    FLASH_Erase_Sector((blk_addr + i) * 4096);
  }

  W25XXX_WR_Block((uint8_t*)buf, blk_addr * 4096, blk_len * 4096);

  return (USBD_OK);
  /* USER CODE END 7 */
}
```

> **注意**：除了和前面一样的，需要自己手动去修改库文件之外，最好把 uint8_t MSC_Storage[32*1024]; 这个变量也删除掉，因为默认库中分配了 uint8_t MSC_Storage[32*1024]; 作为虚拟存储使用，但我们使用了外部存储器（如 SPI Flash），该数组将不再使用，建议释放该部分内存资源；

至此，我们已经成功实现了一个包含 CDC 和 USB 存储的复合设备功能；如果你掌握了这部分内容，就可以自由组合 HID、Audio、WebUSB 等模块，为你的嵌入式项目打造更加丰富的功能形态；

## 总结

这篇文章带你一步步实现了一个“多合一”的 USB 设备，让一个 USB 接口同时具备虚拟串口（CDC）和 U 盘（MSC）的功能；我们从组件库的导入、CubeMX 的配置，到代码逻辑的修改与实现，完整搭建了一个 USB Composite 设备的基础框架；

在过程中，你了解了 USB 是如何以数据包方式传输信息的，也学到了接收多包数据时如何“拼接”数据，以及如何判断一帧数据是否接收完成；通过简单的“超时判断”方法，我们就能实现比较可靠的数据处理逻辑；

另外，我们还结合了之前写好的 FatFS 文件系统，让 MCU 成功模拟出一个能读写的 U 盘，这对实际项目开发很有帮助；

掌握了这些内容之后，你就可以继续添加更多 USB 功能，比如 HID 键盘、音频设备，甚至是 WebUSB，让你的设备功能更加丰富；

如果你是第一次接触 USB Composite，这篇文章可以作为一个很好的起点；希望这次的实践能帮你打下基础，也欢迎你把这些经验应用到自己的项目中，做出更多好玩的嵌入式设备！
