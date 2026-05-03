---
title: 玲珑GUI-显示图片
categories: graphic

date: 2022-11-02 13:39:51
updated: 2022-11-02 13:39:51
---

## 写在前面

在前一章中我们介绍了如何移植玲珑GUI、如何配置基本绘图函数和触摸支持等内容；不过关于“**如何显示图片**”的部分，我并没有在上一章中展开讲解——主要是因为图片显示这部分内容稍微复杂一些，因此我决定单独写一章来详细说明；

如果你在使用过程中发现界面上的图片控件始终无法显示，很可能就是卡在了这个点上；

如果你有调试代码，在一路追踪之后可以发现，在llGeneralImageShow函数内部会调用llReadExFlash来读取图片数据；

> 还记得在生成代码时，玲珑GUI 会提示设置 image size 吗？实际上，它会自动对图片进行处理，并在 MDK 工程目录下的 LingLongGUI/User 文件夹中生成一个名为 image.bin 的文件；

所以我们的目标就是：无论使用什么方式，都要让单片机能够读取到 image.bin 中的数据；实现这一点的方法有很多，比如：

- 将文件存储到外部 Flash 中，在运行时读取；
- 使用工具将文件转换为数组，直接复制到代码中进行访问；
- 借助 MDK 的 FCARM 功能实现文件到数据的转换；

## 简化处理方式：将图片转为数组

之前我也说过，会用简单的方式来说问题，所以这里我用最简单的将文件转为数组来实现，我写了个Python脚本如下：

```python
import os
import sys
import struct

def read_data_from_binary_file(filename, list_data):
    f = open(filename, 'rb')
    f.seek(0, 0)
    while True:
        t_byte = f.read(1)
        if len(t_byte) == 0:
            break
        else:
            list_data.append("0x%.2x" % ord(t_byte))
    f.close()

def write_data_to_text_file(filename, list_data, data_num_per_line):
    f_output = open(filename, 'w+')
    if ((data_num_per_line <= 0) or data_num_per_line > len(list_data)):
        data_num_per_line = 16
        print('data_num_per_line out of range,use default value\n')
    f_output.write('    ')
    for i in range(0,len(list_data)):
        if ( (i != 0) and (i % data_num_per_line == 0)):
            f_output.write('\n    ')
            f_output.write(list_data[i]+', ')
        elif (i + 1)== len(list_data):
            f_output.write(list_data[i])
        else:
            f_output.write(list_data[i]+', ')
    f_output.write('\n')
    f_output.close()

def bin_2_c_array(src,dest):
    input_f = src
    output_f = dest
    data_num_per_line = 16
    list_data = []
    read_data_from_binary_file(input_f, list_data)
    write_data_to_text_file(output_f, list_data, data_num_per_line)

if __name__ == '__main__':
    bin_2_c_array(sys.argv[1], sys.argv[2])
```

在win的终端里这样调用，就可以把bin文件转成C的数组了：
```bash
python.exe ./conver.py ./image.bin ./data.txt
```

## 补充玲珑GUI函数，完成图片显示

为了简单直接，我在 LL_Config.c 文件尾部定义了如下数组，把 data.txt 中的内容复制进来：
```c
const uint8_t IMAGE[] = {
    // 复制 data.txt 中的内容
};
```
这样，图片就作为常量直接存储在 MCU 的 Flash 中了，不需要额外的外部 Flash 驱动；

然后补全LL_Config.c中的llReadExFlash函数，如下：
```c
void llReadExFlash(uint32_t addr,uint8_t* pBuffer,uint16_t length)
{
    extern const uint8_t IMAGE[];

    memcpy(pBuffer, IMAGE + addr, length);
}
```

> 注意：这里只是模拟一个从外部 Flash 读取的过程，实测完全可行；当然，如果你项目真的用了 SPI Flash 或 QSPI Flash或者其他的存储方案，那就要用实际的读写接口去替换 memcpy；如果你还添加了文件系统，那么就用f_read来读取；

这样做完，我们的图片就可以显示了；
