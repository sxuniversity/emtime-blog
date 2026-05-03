---
title: 从 0 搭建 SPI Flash 文件系统：驱动、FatFS、读写与坑点
categories: mcu

date: 2024-10-26 23:15:07
updated: 2024-10-26 23:15:07
---

FatFS官网：https://elm-chan.org/fsw/ff/00index_e.html

当你用单片机做项目，代码调试靠串口、数据记录靠看屏幕、文件读写靠想象，久而久之，你会发现：**没有文件系统，生活就像裸奔，哪都能跑，就是不太方便；**

尤其是在一些**需要长时间运行、持续采集数据**的应用场景中，比如环境监测、设备日志记录、传感器数据采集等，如果没有一个可靠的文件系统来进行**数据持久化存储**，不仅开发调试麻烦，维护和升级也会变得困难重重；你总不能每次都靠串口打印几十KB甚至几MB的数据吧？

这时候你可能听说了一个神器：**FatFS**，一个轻量级的 FAT 文件系统，专为嵌入式系统设计，小巧灵活，支持 SD 卡、SPI Flash，甚至 RAMDisk；不论你用的是 STM32、GD32，还是别的 MCU 平台，都能把它“嫁接”过去；

有了文件系统，不仅可以更方便地**与电脑共享数据**（比如通过 U 盘或 SD 卡读取设备日志），还能按时间归档、分类管理信息，甚至在设备**意外断电或异常重启时保留关键数据**，提升项目的健壮性和专业程度；

那么，这篇文章就是来讲一讲：**如何在你的单片机上，成功移植 FatFS，让你的 MCU 拥有读写文件的能力；**

## FatFS 移植流程概览     

FatFS 的移植主要包括以下几个步骤：

1. 准备底层存储驱动（如 SPI Flash 驱动）
2. 实现 FatFS 所需的 diskio.c 接口函数
3. 配置 ffconf.h 以满足你的文件系统需求
4. 在主函数中初始化 FatFS，挂载文件系统
5. 实现文件的读写操作测试

接下来，我们从第一步开始，移植 SPI Flash 驱动；

## FatFS 文件系统移植到 SPI Flash

要让 FatFS 在单片机上正常工作，首先你得有一个“存储设备”能读能写；虽然 SD 卡是最常见的选择，但很多时候，SPI Flash 是更方便的一种方式：不需要外接卡座、不怕接触不良，容量也够用；

对应的驱动文件如下，将文件添加到你的工程中，驱动文件来自沁恒例程，做了一点点的补充与修改：

W25Qxx.c

```c
#include "W25Qxx.h"

static void W25Qxx_Reset(void);

W25Qxx_Info_t W25Qxx_Info = {0};

__weak void W25Qxx_CS_Enable(void)
{

}

__weak void W25Qxx_CS_Disable(void)
{

}

// 返回1是正常
__weak uint8_t W25Qxx_ReadByte(uint8_t* RxData, uint16_t Size)
{
  (void)RxData;
  (void)Size;

  return 0;
}

// 返回1是正常
__weak uint8_t W25Qxx_WriteByte(uint8_t* TxData, uint16_t Size)
{
  (void)TxData;
  (void)Size;

  return 0;
}

__weak uint32_t W25Qxx_GetTick(void)
{
  return 0;
}

/**
  * @brief  Initializes the W25Q128FV interface.
  * @retval None
  */
uint8_t W25Qxx_Init(void)
{
  /* Reset W25Qxx */
  W25Qxx_Reset();

  return W25Qxx_GetStatus();
}

/**
  * @brief  This function reset the W25Qx.
  * @retval None
  */
static void W25Qxx_Reset(void)
{
  uint8_t cmd[2] = {RESET_ENABLE_CMD, RESET_MEMORY_CMD};

  W25Qxx_CS_Enable();
  /* Send the reset command */
  W25Qxx_WriteByte(cmd, 2);
  W25Qxx_CS_Disable();
}

/**
  * @brief  Reads current status of the W25Q128FV.
  * @retval W25Q128FV memory status
  */
uint8_t W25Qxx_GetStatus(void)
{
  uint8_t cmd[] = {READ_STATUS_REG1_CMD};
  uint8_t status;

  W25Qxx_CS_Enable();
  /* Send the read status command */
  W25Qxx_WriteByte(cmd, 1);
  /* Reception of the data */
  W25Qxx_ReadByte(&status, 1);
  W25Qxx_CS_Disable();

  /* Check the value of the register */
  if((status & W25QXX_FSR_BUSY) != 0)
  {
    return W25QXX_BUSY;
  }
  else
  {
    return W25QXX_OK;
  }
}

/**
  * @brief  This function send a Write Enable and wait it is effective.
  * @retval None
  */
uint8_t W25Qxx_WriteEnable(void)
{
  uint8_t cmd[] = {WRITE_ENABLE_CMD};
  uint32_t tickstart = W25Qxx_GetTick();

  /*Select the FLASH: Chip Select low */
  W25Qxx_CS_Enable();
  /* Send the read ID command */
  W25Qxx_WriteByte(cmd, 1);
  /*Deselect the FLASH: Chip Select high */
  W25Qxx_CS_Disable();

  /* Wait the end of Flash writing */
  while(W25Qxx_GetStatus() == W25QXX_BUSY)
  {
    /* Check for the Timeout */
    if((W25Qxx_GetTick() - tickstart) > W25QXX_TIMEOUT_VALUE)
    {
      return W25QXX_TIMEOUT;
    }
  }

  return W25QXX_OK;
}

/**
  * @brief  Read Manufacture/Device ID.
  * @param  return value address
  * @retval None
  */
void W25Qxx_Read_ID(uint8_t* ID)
{
  uint8_t cmd[4] = {READ_JEDEC_ID_CMD, 0x00, 0x00, 0x00};

  W25Qxx_CS_Enable();
  /* Send the read ID command */
  W25Qxx_WriteByte(cmd, 1);
  /* Reception of the data */
  W25Qxx_ReadByte(ID, 3);
  W25Qxx_CS_Disable();
}

void W25Qxx_IC_Check(void)
{
  uint32_t count;

  uint8_t temp_id[3];

  /* Read FLASH ID */
  W25Qxx_Read_ID(temp_id);

  W25Qxx_Info.Flash_ID = ((uint32_t)temp_id[0] << 16) | 
                         ((uint32_t)temp_id[1] << 8)  | 
                         ((uint32_t)temp_id[2]);

  W25Qxx_Info.Flash_Sector_Count = 0x00;
  W25Qxx_Info.Flash_Sector_Size  = 0x00;

  switch(W25Qxx_Info.Flash_ID)
  {
  /* W25XXX */
  case W25X10_FLASH_ID: /* 0xEF3011-----1M bit */
    count = 1;
    break;

  case W25X20_FLASH_ID: /* 0xEF3012-----2M bit */
    count = 2;
    break;

  case W25X40_FLASH_ID: /* 0xEF3013-----4M bit */
    count = 4;
    break;

  case W25X80_FLASH_ID: /* 0xEF4014-----8M bit */
    count = 8;
    break;

  case W25Q16_FLASH_ID1: /* 0xEF3015-----16M bit */
  case W25Q16_FLASH_ID2: /* 0xEF4015-----16M bit */
    count = 16;
    break;

  case W25Q32_FLASH_ID1: /* 0xEF4016-----32M bit */
  case W25Q32_FLASH_ID2: /* 0xEF6016-----32M bit */
    count = 32;
    break;

  case W25Q64_FLASH_ID1: /* 0xEF4017-----64M bit */
  case W25Q64_FLASH_ID2: /* 0xEF6017-----64M bit */
    count = 64;
    break;

  case W25Q128_FLASH_ID1: /* 0xEF4018-----128M bit */
  case W25Q128_FLASH_ID2: /* 0xEF6018-----128M bit */
    count = 128;
    break;

  case W25Q256_FLASH_ID1: /* 0xEF4019-----256M bit */
  case W25Q256_FLASH_ID2: /* 0xEF6019-----256M bit */
    count = 256;
    break;
  default:
    if((W25Qxx_Info.Flash_ID != 0xFFFFFFFF) && (W25Qxx_Info.Flash_ID != 0x00000000))
    {
      count = 16;
    }
    else
    {
      count = 0x00;
    }
    break;
  }
  count = ((uint32_t)count * 1024) * ((uint32_t)1024 / 8);

  if(count)
  {
    // 如果是内部,那么DEF_UDISK_SECTOR_SIZE是512,如果是外部,则DEF_SECTOR_SIZE是4096
    W25Qxx_Info.Flash_Sector_Count = count / DEF_SECTOR_SIZE; // DEF_SECTOR_SIZE;
    W25Qxx_Info.Flash_Sector_Size  = DEF_SECTOR_SIZE;         // DEF_SECTOR_SIZE;
    W25Qxx_Info.Flash_Page_Size = 256;                        // 全系列固定的
  }
  else
  {
//    printf ("External Flash not connected\r\n");
    //  while(1);
  }
}

/**
  * @brief  Reads an amount of data from the QSPI memory.
  * @param  pData: Pointer to data to be read
  * @param  ReadAddr: Read start address
  * @param  Size: Size of data to read
  * @retval QSPI memory status
  */
// TODO
uint8_t W25Qxx_Read(uint8_t* pData, uint32_t ReadAddr, uint32_t Size)
{
  uint8_t cmd[4];

  /* Configure the command */
  cmd[0] = READ_CMD;
  cmd[1] = (uint8_t)(ReadAddr >> 16);
  cmd[2] = (uint8_t)(ReadAddr >> 8);
  cmd[3] = (uint8_t)(ReadAddr);

  W25Qxx_CS_Enable();
  /* Send the read ID command */
  W25Qxx_WriteByte(cmd, 4);
  /* Reception of the data */
  if(W25Qxx_ReadByte(pData, Size) == 0)
  {
    return W25QXX_ERROR;
  }
  W25Qxx_CS_Disable();
  return W25QXX_OK;
}

/**
  * @brief  Writes an amount of data to the QSPI memory.
  * @param  pData: Pointer to data to be written
  * @param  WriteAddr: Write start address
  * @param  Size: Size of data to write,No more than 256byte.
  * @retval QSPI memory status
  */
uint8_t W25Qxx_Write(uint8_t* pData, uint32_t WriteAddr, uint32_t Size)
{
  uint8_t cmd[4];
  uint32_t end_addr, current_size, current_addr;
  uint32_t tickstart = W25Qxx_GetTick();

  /* Calculation of the size between the write address and the end of the page */
  current_addr = 0;

  while(current_addr <= WriteAddr)
  {
    current_addr += W25Qxx_Info.Flash_Page_Size;
  }
  current_size = current_addr - WriteAddr;

  /* Check if the size of the data is less than the remaining place in the page */
  if(current_size > Size)
  {
    current_size = Size;
  }

  /* Initialize the adress variables */
  current_addr = WriteAddr;
  end_addr = WriteAddr + Size;

  /* Perform the write page by page */
  do
  {
    /* Configure the command */
    cmd[0] = PAGE_PROG_CMD;
    cmd[1] = (uint8_t)(current_addr >> 16);
    cmd[2] = (uint8_t)(current_addr >> 8);
    cmd[3] = (uint8_t)(current_addr);

    /* Enable write operations */
    W25Qxx_WriteEnable();

    W25Qxx_CS_Enable();
    /* Send the command */
    if(W25Qxx_WriteByte(cmd, 4) == 0)
    {
      return W25QXX_ERROR;
    }

    /* Transmission of the data */
    if(W25Qxx_WriteByte(pData, current_size) == 0)
    {
      return W25QXX_ERROR;
    }
    W25Qxx_CS_Disable();
    /* Wait the end of Flash writing */
    while(W25Qxx_GetStatus() == W25QXX_BUSY)
    {
      /* Check for the Timeout */
      if((W25Qxx_GetTick() - tickstart) > W25QXX_TIMEOUT_VALUE)
      {
        return W25QXX_TIMEOUT;
      }
    }

    /* Update the address and size variables for next page programming */
    current_addr += current_size;
    pData += current_size;
    current_size = ((current_addr + W25Qxx_Info.Flash_Page_Size) > end_addr) ? (end_addr - current_addr) : W25Qxx_Info.Flash_Page_Size;
  }
  while(current_addr < end_addr);

  return W25QXX_OK;
}

/**
  * @brief  Erases the specified block of the QSPI memory.
  * @param  BlockAddress: Block address to erase
  * @retval QSPI memory status
  */
uint8_t W25Qxx_Erase_Block(uint32_t Address)
{
  uint8_t cmd[4];
  uint32_t tickstart = W25Qxx_GetTick();
  cmd[0] = SECTOR_ERASE_CMD;
  cmd[1] = (uint8_t)(Address >> 16);
  cmd[2] = (uint8_t)(Address >> 8);
  cmd[3] = (uint8_t)(Address);

  /* Enable write operations */
  W25Qxx_WriteEnable();

  /*Select the FLASH: Chip Select low */
  W25Qxx_CS_Enable();
  /* Send the read ID command */
  W25Qxx_WriteByte(cmd, 4);
  /*Deselect the FLASH: Chip Select high */
  W25Qxx_CS_Disable();

  /* Wait the end of Flash writing */
  while(W25Qxx_GetStatus() == W25QXX_BUSY)
  {
    /* Check for the Timeout */
    if((W25Qxx_GetTick() - tickstart) > W25QXX_SECTOR_ERASE_MAX_TIME)
    {
      return W25QXX_TIMEOUT;
    }
  }
  return W25QXX_OK;
}

/**
  * @brief  Erases the entire QSPI memory.This function will take a very long time.
  * @retval QSPI memory status
  */
uint8_t W25Qxx_Erase_Chip(void)
{
  uint8_t cmd[4];
  uint32_t tickstart = W25Qxx_GetTick();
  cmd[0] = CHIP_ERASE_CMD;

  /* Enable write operations */
  W25Qxx_WriteEnable();

  /*Select the FLASH: Chip Select low */
  W25Qxx_CS_Enable();
  /* Send the read ID command */
  W25Qxx_WriteByte(cmd, 1);
  /*Deselect the FLASH: Chip Select high */
  W25Qxx_CS_Disable();

  /* Wait the end of Flash writing */
  while(W25Qxx_GetStatus() != W25QXX_BUSY)
  {
    /* Check for the Timeout */
    if((W25Qxx_GetTick() - tickstart) > W25QXX_BULK_ERASE_MAX_TIME)
    {
      return W25QXX_TIMEOUT;
    }
  }
  return W25QXX_OK;
}
```

W25Qxx.h

```c
#ifndef __W25QXX_H_
#define __W25QXX_H_

#include <stdint.h>

#define W25QXX_BULK_ERASE_MAX_TIME         250000
#define W25QXX_SECTOR_ERASE_MAX_TIME       3000
#define W25QXX_SUBSECTOR_ERASE_MAX_TIME    800
#define W25QXX_TIMEOUT_VALUE 1000

#define DEF_SECTOR_SIZE 4096

#define W25X10_FLASH_ID 0xEF3011   /* 1M bit */
#define W25X20_FLASH_ID 0xEF3012   /* 2M bit */
#define W25X40_FLASH_ID 0xEF3013   /* 4M bit */
#define W25X80_FLASH_ID 0xEF4014   /* 8M bit */
#define W25Q16_FLASH_ID1 0xEF3015  /* 16M bit */
#define W25Q16_FLASH_ID2 0xEF4015  /* 16M bit */
#define W25Q32_FLASH_ID1 0xEF4016  /* 32M bit */
#define W25Q32_FLASH_ID2 0xEF6016  /* 32M bit */
#define W25Q64_FLASH_ID1 0xEF4017  /* 64M bit */
#define W25Q64_FLASH_ID2 0xEF6017  /* 64M bit */
#define W25Q128_FLASH_ID1 0xEF4018 /* 128M bit */
#define W25Q128_FLASH_ID2 0xEF6018 /* 128M bit */
#define W25Q256_FLASH_ID1 0xEF4019 /* 256M bit */
#define W25Q256_FLASH_ID2 0xEF6019 /* 256M bit */

/* Reset Operations */
#define RESET_ENABLE_CMD          0x66
#define RESET_MEMORY_CMD          0x99

#define ENTER_QPI_MODE_CMD        0x38
#define EXIT_QPI_MODE_CMD         0xFF

/* Identification Operations */
#define READ_ID_CMD               0x90
#define DUAL_READ_ID_CMD          0x92
#define QUAD_READ_ID_CMD          0x94
#define READ_JEDEC_ID_CMD         0x9F

/* Read Operations */
#define READ_CMD                  0x03
#define FAST_READ_CMD             0x0B
#define DUAL_OUT_FAST_READ_CMD    0x3B
#define DUAL_INOUT_FAST_READ_CMD  0xBB
#define QUAD_OUT_FAST_READ_CMD    0x6B
#define QUAD_INOUT_FAST_READ_CMD  0xEB

/* Write Operations */
#define WRITE_ENABLE_CMD          0x06
#define WRITE_DISABLE_CMD         0x04

/* Register Operations */
#define READ_STATUS_REG1_CMD      0x05
#define READ_STATUS_REG2_CMD      0x35
#define READ_STATUS_REG3_CMD      0x15

#define WRITE_STATUS_REG1_CMD     0x01
#define WRITE_STATUS_REG2_CMD     0x31
#define WRITE_STATUS_REG3_CMD     0x11

/* Program Operations */
#define PAGE_PROG_CMD             0x02
#define QUAD_INPUT_PAGE_PROG_CMD  0x32

/* Erase Operations */
#define SECTOR_ERASE_CMD          0x20
#define CHIP_ERASE_CMD            0xC7

#define PROG_ERASE_RESUME_CMD     0x7A
#define PROG_ERASE_SUSPEND_CMD    0x75

/* Flag Status Register */
#define W25QXX_FSR_BUSY            ((uint8_t)0x01)    /*!< busy */
#define W25QXX_FSR_WREN            ((uint8_t)0x02)    /*!< write enable */
#define W25QXX_FSR_QE              ((uint8_t)0x02)    /*!< quad enable */

/* Status */
#define W25QXX_OK            ((uint8_t)0x00)
#define W25QXX_ERROR         ((uint8_t)0x01)
#define W25QXX_BUSY          ((uint8_t)0x02)
#define W25QXX_TIMEOUT       ((uint8_t)0x03)

typedef struct
{
  uint32_t Flash_ID;
  uint32_t Flash_Sector_Count;
  uint32_t Flash_Page_Size;
  uint16_t Flash_Sector_Size;
}W25Qxx_Info_t;

extern W25Qxx_Info_t W25Qxx_Info;

uint8_t W25Qxx_Init(void);  // 必须执行
uint8_t W25Qxx_GetStatus(void);
uint8_t W25Qxx_WriteEnable(void);
void W25Qxx_Read_ID(uint8_t* ID);
void W25Qxx_IC_Check(void); // 必须执行
uint8_t W25Qxx_Read(uint8_t* pData, uint32_t ReadAddr, uint32_t Size);
uint8_t W25Qxx_Write(uint8_t* pData, uint32_t WriteAddr, uint32_t Size);
uint8_t W25Qxx_Erase_Block(uint32_t Address);
uint8_t W25Qxx_Erase_Chip(void);

#endif
```
该驱动程序在SPI1上实现了对SPI FLASH的读写操作，包括初始化、读取ID、写入使能、写入禁用、读取状态寄存器、检查IC、擦除扇区、读取块、写入块等操作。

我将与单片机硬件相关的函数都使用弱定义进行了声明，这样在移植的时候，只需要在别的文件中实现硬件操作即可，不需要修改其他文件。比如，你可以在 spi.c 中实现以下函数：

```c
void W25Qxx_CS_Enable(void)
{
  HAL_GPIO_WritePin(SPI1_CS_GPIO_Port, SPI1_CS_Pin, GPIO_PIN_RESET);
}

void W25Qxx_CS_Disable(void)
{
  HAL_GPIO_WritePin(SPI1_CS_GPIO_Port, SPI1_CS_Pin, GPIO_PIN_SET);
}

// 返回1是正常
uint8_t W25Qxx_ReadByte(uint8_t* RxData, uint16_t Size)
{
  return (HAL_SPI_Receive(&hspi1, RxData, Size, 0xFF) == HAL_OK);
}

// 返回1是正常
uint8_t W25Qxx_WriteByte(uint8_t* TxData, uint16_t Size)
{
  return (HAL_SPI_Transmit(&hspi1, TxData, Size, 0xFF) == HAL_OK);
}

uint32_t W25Qxx_GetTick(void)
{
  return HAL_GetTick();
}
```

> 执行FLASH_IC_Check函数之后，函数会根据返回的芯片 ID，设置Flash_Type、Flash_ID、Flash_Sector_Count、Flash_Sector_Size等变量，以便后续操作使用；

## 移植 FatFS

### 实现 diskio.c 接口

主要就是需要编写 diskio.c 文件，实现以下函数（可以直接复制）：

```c
/*-----------------------------------------------------------------------*/
/* Low level disk I/O module SKELETON for FatFs     (C)ChaN, 2019        */
/*-----------------------------------------------------------------------*/
/* If a working storage control module is available, it should be        */
/* attached to the FatFs via a glue function rather than modifying it.   */
/* This is an example of glue functions to attach various exsisting      */
/* storage control modules to the FatFs module with a defined API.       */
/*-----------------------------------------------------------------------*/

#include "ff.h"			/* Obtains integer types */
#include "diskio.h"		/* Declarations of disk functions */

#include "W25Qxx.h"

/* Definitions of physical drive number for each drive */
#define DEV_SPIFLASH 0

/*-----------------------------------------------------------------------*/
/* Get Drive Status                                                      */
/*-----------------------------------------------------------------------*/

DSTATUS disk_status(
  BYTE pdrv		/* Physical drive nmuber to identify the drive */
)
{
  DSTATUS stat;
  uint8_t result;

  switch(pdrv)
  {
  case DEV_SPIFLASH :
    result = W25Qxx_GetStatus();

    // translate the reslut code here
    switch(result)
    {
    case W25QXX_OK:
      stat = STA_NOINIT & (~STA_NOINIT);
      break;

    case W25QXX_ERROR:
      stat = STA_NOINIT;
      break;

    case W25QXX_BUSY:
      stat = STA_NOINIT;
      break;

    case W25QXX_TIMEOUT:
      stat = STA_NOINIT;
      break;
    }
    return stat;

  }
  return STA_NOINIT;
}



/*-----------------------------------------------------------------------*/
/* Inidialize a Drive                                                    */
/*-----------------------------------------------------------------------*/

DSTATUS disk_initialize(
  BYTE pdrv				/* Physical drive nmuber to identify the drive */
)
{
  DSTATUS stat;
  uint8_t result;

  switch(pdrv)
  {
  case DEV_SPIFLASH :
    result = W25Qxx_Init();
    W25Qxx_IC_Check();

    // translate the reslut code here
    switch(result)
    {
    case W25QXX_OK:
      stat = STA_NOINIT & (~STA_NOINIT);
      break;

    case W25QXX_ERROR:
      stat = STA_NOINIT;
      break;

    case W25QXX_BUSY:
      stat = STA_NOINIT;
      break;

    case W25QXX_TIMEOUT:
      stat = STA_NOINIT;
      break;
    }
    return stat;

  }
  return STA_NOINIT;
}



/*-----------------------------------------------------------------------*/
/* Read Sector(s)                                                        */
/*-----------------------------------------------------------------------*/

DRESULT disk_read(
  BYTE pdrv,		/* Physical drive nmuber to identify the drive */
  BYTE* buff,		/* Data buffer to store read data */
  LBA_t sector,	/* Start sector in LBA */
  UINT count		/* Number of sectors to read */
)
{
  DRESULT res;
  uint8_t result;

  switch(pdrv)
  {
  case DEV_SPIFLASH :
    // translate the arguments here

    result = W25Qxx_Read(buff, sector * W25Qxx_Info.Flash_Sector_Size, count * W25Qxx_Info.Flash_Sector_Size);

    // translate the reslut code here
    switch(result)
    {
    case W25QXX_OK:
      res = RES_OK;
      break;

    case W25QXX_ERROR:
      res = RES_ERROR;
      break;

    case W25QXX_BUSY:
      res = RES_NOTRDY;
      break;

    case W25QXX_TIMEOUT:
      res = RES_ERROR;
      break;
    }
    return res;

  }

  return RES_PARERR;
}



/*-----------------------------------------------------------------------*/
/* Write Sector(s)                                                       */
/*-----------------------------------------------------------------------*/

#if FF_FS_READONLY == 0

DRESULT disk_write(
  BYTE pdrv,			/* Physical drive nmuber to identify the drive */
  const BYTE* buff,	/* Data to be written */
  LBA_t sector,		/* Start sector in LBA */
  UINT count			/* Number of sectors to write */
)
{
  DRESULT res;
  int result;

  switch(pdrv)
  {
  case DEV_SPIFLASH :
    // translate the arguments here
    for(UINT i = 0; i < count; i++)
    {
      W25Qxx_Erase_Block((sector + i) * W25Qxx_Info.Flash_Sector_Size);
    }
    result = W25Qxx_Write((uint8_t*)buff, sector * W25Qxx_Info.Flash_Sector_Size, count * W25Qxx_Info.Flash_Sector_Size);

    // translate the reslut code here
    switch(result)
    {
    case W25QXX_OK:
      res = RES_OK;
      break;

    case W25QXX_ERROR:
      res = RES_ERROR;
      break;

    case W25QXX_BUSY:
      res = RES_NOTRDY;
      break;

    case W25QXX_TIMEOUT:
      res = RES_ERROR;
      break;
    }
    return res;

  }

  return RES_PARERR;
}

#endif


/*-----------------------------------------------------------------------*/
/* Miscellaneous Functions                                               */
/*-----------------------------------------------------------------------*/

DRESULT disk_ioctl(
  BYTE pdrv,		/* Physical drive nmuber (0..) */
  BYTE cmd,		/* Control code */
  void* buff		/* Buffer to send/receive control data */
)
{
  DRESULT res = RES_OK;

  switch(pdrv)
  {
  case DEV_SPIFLASH :
    switch(cmd)
    {
    case GET_SECTOR_COUNT://将驱动器上可用扇区的数目返回到buff指向的DWORD变量中
    {
      *(DWORD*)buff = W25Qxx_Info.Flash_Sector_Count;
      break;
    }
    case GET_SECTOR_SIZE://将媒体的扇区大小返回到buff指向的WORD变量中
    {
      *(WORD*)buff = W25Qxx_Info.Flash_Sector_Size; //类型是WORD的类型,每个扇区是4096的大小,这里同时还需要修改MAX_SS的值
      break;
    }
    case GET_BLOCK_SIZE://将闪存介质的擦除块大小(以扇区为单位)返回到buff指向的DWORD变量中
    {
      *(DWORD*)buff = 1; //每次擦除的大小是1个扇区,因为单位是扇区
      break;
    }
    }
    // Process of the command for the RAM drive

    return res;

  }

  return RES_PARERR;
}
```

### 修改 ffconf.h 配置

```c
#define FF_MAX_SS		4096  // 使用的是SPI FLash,所以这个需要修改为4096

#define FF_USE_MKFS		1     // 这个需要修改为1启用格式化的功能

#define FF_CODE_PAGE	936   // 可以设置成936,增加对中文的支持

#define FF_FS_NORTC		1     // 这个需要设置成1,就是先不搞RTC相关的日期功能
```

## 使用

main.c 文件中大体如下操作即可

```c
#include "ff.h"

FATFS FsObject;

FIL fp;

static BYTE work_buffer[4096];

int main(void)
{
  FRESULT result;
  result=f_mount(&FsObject,"0:",1); // 这个0:就是路径,和#define FF_VOLUMES 1有关,设定为1则路径只有0:

  // 如果是13,则表明没有格式化,我们进行格式化
  // 如果是11,则表明数量对不上,需要去改设备个数,也就是FF_VOLUMES
  if(result == 13)
  {
    MKFS_PARM Format = {FM_FAT32, 0, 0, 0, 0};  // 为了兼容,可以改为FM_ANY,对16MB来说,FAT16最合适
    result = f_mkfs("0:", &Format, work_buffer, sizeof(work_buffer));

    result=f_mount(&FsObject,"0:",1);  // 再次挂载,其实还可以进行判断
  }

//  result = f_open(&fp, "0:test.txt", FA_OPEN_ALWAYS|FA_WRITE|FA_READ);

//  UINT test;
//  result = f_write(&fp,"test1234",sizeof("test1234"),&test);
//  f_close(&fp);

  
  result = f_open(&fp, "0:test.txt", FA_OPEN_ALWAYS|FA_WRITE|FA_READ);

  UINT test;
  uint8_t read[20];
  result=f_read(&fp, read, f_size(&fp), &test);

  while(1)
  {
    
  }

  return 0;
}
```

## 常见问题与调试技巧

### Flash 相关问题

- **Q: FLASH_ReadID() 返回 0xFFFFFF 或 0x000000？**  
  A: SPI Flash 没有接好，或者 SPI 接口初始化未正确完成；建议检查：
  - SPI 时钟、模式是否与 Flash 兼容；
  - Flash 供电是否稳定；
  - CS 引脚是否正确拉低后开始通信；

- **Q: Flash 容量识别不对？**  
  A: FLASH_IC_Check() 中只处理了常见型号，若你使用的是不在列表内的型号，请根据 datasheet 添加对应的 JEDEC ID 和容量；

### 挂载与格式化相关

- **Q: f_mount 返回 FR_NO_FILESYSTEM（13），怎么解决？**  
  A: 说明当前设备上没有可识别的文件系统；应使用 f_mkfs 对 Flash 进行格式化，完成后再调用 f_mount 重新挂载；

- **Q: f_mount 返回 FR_INVALID_DRIVE（11）？**  
  A: 说明 FatFS 的卷编号配置有问题，请检查 ffconf.h 中 FF_VOLUMES 是否 >= 你的逻辑盘号，比如 f_mount(..., "0:", 1) 表示你至少得设置 #define FF_VOLUMES 1；

### FAT 文件系统格式相关

- **Q: f_mkfs() 返回 FR_INVALID_PARAMETER？**  
  A: f_mkfs 参数设置不当，建议使用如下方式初始化：

```c
MKFS_PARM fs_param = {FM_ANY, 0, 0, 0, 0};
f_mkfs("0:", &fs_param, work_buffer, sizeof(work_buffer));
```

- **Q: 格式化完文件系统容量很小（比如识别为1MB）？**  
  A: 可能是扇区大小未正确返回，检查 disk_ioctl() 中 GET_SECTOR_SIZE 和 GET_SECTOR_COUNT 是否准确计算，是否符合你实际 Flash 容量；

### 读写文件异常

- **Q: 文件写入后读出来的数据不对，乱码或者全 0？**  
  A: 可能原因
  - 写入之前没有正确擦除扇区；
  - 写操作未对齐页写入（W25系列对页写入有要求）；
  - 写入数据后未调用 f_close() 或 f_sync()，导致未刷新缓存到 Flash；
  - diskio.c 中 FLASH_Erase_Sector() 和 W25XXX_WR_Block() 地址未正确计算；

- **Q: 写文件成功了，但再次打开文件内容变空？**  
  A: 注意写模式是否是 FA_CREATE_ALWAYS，该模式会每次打开都清空内容；如果想保留内容，改为 FA_OPEN_ALWAYS | FA_WRITE 并调用 f_lseek(&fp, f_size(&fp)) 跳到末尾再写；

### 文件系统行为与配置相关

- **Q: 文件名太长无法识别？**  
  A: 默认 FatFS 禁用长文件名（LFN），需在 ffconf.h 中配置：

```c
#define FF_USE_LFN    1
#define FF_MAX_LFN    64
```

- **Q: 中文文件名乱码？**  
  A: 请设置正确的代码页，例如：

```c
#define FF_CODE_PAGE 936  // 简体中文 GBK
```

- **Q: 同一个文件写入后再读读取不到内容？**  
  A: 若写入后未关闭文件或调用 f_sync()，FatFS 可能未刷新数据到底层 Flash，建议：

```c
f_write(...);
f_sync(&fp);  // 确保数据落盘
```

### 运行异常 / 稳定性问题

- **Q: 写入操作中系统卡死或死循环？**  
  A: Flash 的写入/擦除是阻塞操作，部分芯片擦除单个扇区可能耗时几十毫秒；建议你：
  - 在写函数中加入 watchdog 喂狗机制；
  - 考虑用非阻塞 Flash 驱动 + 文件系统缓存策略来优化；

- **Q: Flash 写入过程中掉电，数据损坏？**  
  A: 推荐使用 FatFS 的事务机制，例如在写入文件时增加 f_sync()，或使用 FAT 的备用区功能（需高级配置）；另外也可以借助 CRC 校验机制保证文件有效性；

### 调试技巧

- **建议在初始化完成后先跑一个简单的读写测试函数**：

```c
f_open(&fp, "test.txt", FA_CREATE_ALWAYS | FA_WRITE);
f_write(&fp, "hello", 5, &bw);
f_close(&fp);

f_open(&fp, "test.txt", FA_READ);
f_read(&fp, buf, 5, &br);
f_close(&fp);
```

- **使用串口或者 RTT 等工具打印中间步骤结果**，比如挂载结果、读写返回值、实际读出的内容，有助于快速定位问题；

- **调试期间建议将所有错误码都打印出来对应 FR_XXX 含义**，便于对照 FatFS 源码中的错误枚举；

## 总结

本篇博客详细介绍了如何将 FatFS 移植到 SPI Flash，并通过 W25Q128 实现文件读写功能；从驱动实现、FatFS 配置、文件操作到问题排查，整个流程强调的是「实用」与「稳定」，希望对你的嵌入式项目有所帮助；

值得注意的是，**SPI Flash 天生具备“写前擦除”“页擦除/块擦除”的特性，且写入寿命有限**（一般每个扇区约 10 万次擦写）；这意味着在频繁写入场景下，Flash 容易出现写坏、性能衰减等问题；

为此，建议关注以下几点：
- **磨损均衡（Wear Leveling）**：FatFS 本身不具备磨损均衡机制，如果使用 SPI Flash 存储频繁变更的数据（如日志、数据库），需要在应用层实现“循环覆盖”或“动态地址映射”来避免单点反复擦写；
- **避免频繁格式化和 f_open/f_write/f_close 操作循环**，应尽可能复用文件句柄，按需 flush 写入；
- **设置合适的缓存机制**，如启用 sector 缓冲，减少物理擦写次数；
- **建议定期备份重要数据**，并在系统初始化时进行 Flash 健康检查（可利用空闲位、标志位判断 Flash 是否写满或擦损）；
- **日志/配置文件等** 建议使用固定格式（如简化版 TLV）写入，便于恢复和分析；

最后，虽然 FatFS 的结构设计优雅轻量，但在用它搭配 SPI Flash 构建嵌入式文件系统时，我们仍需深入理解底层 Flash 的行为特性，并结合自身项目场景做出相应调整和优化；

下一步，我计划基于 **USB Composite（复合设备）** 实现 STM32 同时具备串口调试和 **模拟U盘功能**，通过 USB MSC 协议挂载 FatFS 文件系统，让用户能够在电脑端直接读写 SPI Flash 中的数据，这将进一步提升系统的易用性和可扩展性，敬请期待！
