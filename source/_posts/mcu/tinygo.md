---
title: TinyGo开发单片机
categories: mcu

date: 2023-07-08 17:07:24
updated: 2023-07-08 17:07:24
---

官网：https://tinygo.org

在单片机开发领域，**C 语言依然是不可动摇的主力军**；它生态成熟、性能稳定、资源开销小，是嵌入式工程师绕不开的基础工具；几乎所有主流的芯片厂商和开发工具链，默认都以 C 语言为核心进行支持；

不过，对于习惯现代语言开发体验的程序员来说，“单片机只能写 C”往往显得有些局限；有没有可能，用一门**更高级、语法更友好、开发体验更现代的语言**来完成嵌入式开发？这就是 TinyGo 想要回答的问题；

Go 语言以简洁、**并发友好**、构建快速著称；相比 C，**它避免了野指针、缓冲区溢出等经典问题**，协程（goroutine）和通道（channel）让**并发写法天然支持**；再加上现代化的标准库、优秀的构建系统、开箱即用的工具链，Go 已经成为后端服务开发领域的“明星选手”；

TinyGo 则是一个将 Go 语言带入嵌入式世界的编译器项目；它对 Go 的语法和运行时做了精简，针对资源受限的 MCU 和 WebAssembly 场景做了大量优化；你仍然可以写几乎原汁原味的 Go 代码，但最终编译出来的是可以直接烧录进单片机的二进制程序；

> 当然，话说回来，TinyGo 目前**仍算不上成熟完善**；
> 一方面，它在语法和标准库支持方面还存在一定限制，部分 Go 的常规语法或第三方库在 TinyGo 中并不完全兼容；调试体验也不如传统 C 工具链那样成熟顺手；有时你甚至需要阅读 LLVM 编译错误或查阅芯片文档，才能定位一个低级问题；
> 另一方面，它对单片机平台的支持还比较“基础”——很多常用外设（如 DMA、CAN、低功耗模式、USB 等）要么没有封装，要么封装得不够细致，你可能不得不查阅芯片手册，甚至手动修改底层寄存器；
> 
> 这也就意味着，如果你想要进行复杂的、性能敏感的嵌入式开发，TinyGo 可能还远不如成熟的 C 生态来得省心、稳定、受控；
> 但站在另一个角度来看，如果只是做一些基础的 GPIO 控制、串口通信、I2C/SPI 设备测试等原型验证，TinyGo 还是非常够用的，而它带来的高可读性、快速编译体验、包管理机制等优势也能大大降低开发门槛；

它不完美，但确实有趣；如果你愿意尝试，TinyGo 是一个值得探索的方向；

这篇博客将记录我使用 TinyGo 开发单片机项目的完整过程：从开发环境搭建、编译部署，到代码结构设计、避坑技巧；无论你是 Go 开发者，想“下沉”做些硬件探索；还是嵌入式开发者，对 C 语言以外的方案感兴趣，希望本文都能给你带来一些启发与帮助；

## 环境搭建

### 安装 TinyGo

TinyGo 的安装非常简单，官方提供了预编译好的安装包，适用于 Windows、macOS 以及大多数主流 Linux 发行版，你可以直接前往 [TinyGo 官网安装页](https://tinygo.org/getting-started/install) 选择适合你平台的方式进行安装；

我使用的是Linux，有Linux和Docker安装两种方式，为了更方便地与串口、文件系统等资源打交道，我直接选择了Linux安装方式，下载后安装，然后将tinygo添加到环境变量即可；

```bash
# 下载 TinyGo 的 DEB 安装包（以 v0.37.0 为例）
wget https://github.com/tinygo-org/tinygo/releases/download/v0.37.0/tinygo_0.37.0_amd64.deb

# 安装 TinyGo
sudo dpkg -i tinygo_0.37.0_amd64.deb
```

安装完成后，**建议检查 tinygo 命令是否已正确添加到环境变量中**；如果运行 tinygo 提示“命令未找到”，可手动编辑~/.bashrc文件添加路径：

```bash
# 直接 export 可以临时生效
export PATH=$PATH:/usr/local/bin
```

确认安装是否成功：

```bash
# 直接 export 可以临时生效
tinygo version
```

若能成功输出版本信息，如 tinygo version 0.37.0 linux/amd64 (using go version go1.24.2 and LLVM version 19.1.2)，说明 TinyGo 已正确安装；

### 支持的芯片与平台

安装完成后，推荐浏览 TinyGo 官方文档中维护的芯片支持列表：[Supported Microcontrollers](https://tinygo.org/docs/reference/microcontrollers)

这个页面详细列出了目前 TinyGo 支持的主控平台（如 Arduino、STM32、ESP32、RP2040 等）以及它们所支持的外设功能（如 GPIO、PWM、I2C、SPI、UART 等）；需要注意，不同芯片的支持程度不一，某些平台仅支持基础外设；

> **建议在开始项目之前先查一查你的开发板是否被 TinyGo 正式支持**，否则可能会遇到开发难度较大的兼容性问题；

### 使用 VSCode 进行 TinyGo 开发

虽然 TinyGo 可以直接通过命令行构建和烧录，但在 VSCode 中配合插件使用，不仅能享受自动补全、跳转、语法检查等功能，还能更方便地进行调试与项目管理，提升整体开发效率，尤其是在开发较复杂的项目时更为明显；

在 VSCode 中搜索并安装以下插件：

| 插件名称 | 说明 |
| ------- | ---- |
| Go (by Go Team at Google) | 	TinyGo 同样使用 Go 语言语法，因此这个插件是基础，提供语法高亮、跳转、补全等功能 |
| TinyGo for VSCode（可选） | 社区维护的插件，提供对 TinyGo 项目的构建与烧录支持（但维护不频繁，可视情况使用） |
| Cortex-Debug（如你的开发板支持 SWD 调试） | 如果你使用支持 SWD 的板子，比如 STM32、RP2040 等，可以配合 GDB 调试 |

插件安装后，VSCode 会自动识别 .go 文件并激活相应语言服务，使 TinyGo 的项目开发体验更接近现代 IDE；

> **注意**：当打开 .go 文件的时候，VSCode 会提示安装一批 Go 相关工具（如格式化工具、lint 工具、静态分析等），建议全部安装，有助于保持代码整洁并及时发现问题；不过国内用户可能会遇到下载速度慢的问题，建议配置 Go 的代理源或切换网络；

## ESP8266开发

在完成 TinyGo 环境配置与 VSCode 插件安装后，我们就可以正式编写代码、上传到开发板并运行了；以下是我基于 NodeMCU（ESP8266）的完整开发流程记录，适合初学者参考；

### 初始化项目

和标准的 Go 项目一样，TinyGo 同样需要使用 Go Module 管理依赖；所以，在项目目录下执行以下命令进行初始化：

```bash
go mod init esp8266
```

这样可以避免未来可能出现的包管理混乱，并使构建过程更规范；

### VSCode 开发板选择

在安装了 TinyGo 插件之后，**VSCode 会在底部状态栏提供目标开发板型号选择**，我们需要在这里将目标设为 nodemcu 或 esp8266，二者都能成功编译上传：

> 建议直接选择 nodemcu，对应官方文档中的 esp8266 target 配置；

### 编写代码

在项目根目录下创建 main.go 文件，以下是点亮 GPIO 的示例代码，分别控制了 D1、D2、D4 三个引脚进行闪烁；

```go
package main

import (
	"machine"
	"sync"
	"time"
)

var D1, D2, D4 = machine.D1, machine.D2, machine.D4

// 虽然 init() 方法方便初始化，但由于每个包的 init 执行顺序不固定，实际开发中并不推荐用于重要初始化流程
func init() {
	D1.Configure(machine.PinConfig{Mode: machine.PinOutput})
	D2.Configure(machine.PinConfig{Mode: machine.PinOutput})
	D4.Configure(machine.PinConfig{Mode: machine.PinOutput})
}

func main() {
	var wg sync.WaitGroup // 用于优雅退出（虽然这类项目通常不会退出）

	wg.Add(1)
	go gpio(&wg, D1)

	wg.Add(1)
	go gpio(&wg, D2)

	wg.Add(1)
	go gpio1(&wg)

	wg.Wait()
}

func gpio(wg *sync.WaitGroup, gpiox machine.Pin) {
	defer wg.Done()
	for {
		gpiox.High()
		time.Sleep(time.Second)
		gpiox.Low()
		time.Sleep(time.Second)
	}
}

func gpio1(wg *sync.WaitGroup) {
	defer wg.Done()
	for {
		D4.High()
		time.Sleep(500 * time.Millisecond)
		D4.Low()
		time.Sleep(500 * time.Millisecond)
	}
}
```

> Go 的代码缩进遵循 gofmt 规范，即使你尝试使用制表符或空格，格式也会被自动格式化；

### 编译与烧录

ESP8266 使用的是非标准的烧录方式，因此直接执行：

```bash
tinygo flash -target=esp8266 .
```

**是无法成功烧录的**，你会遇到连接超时、模式不对等问题；
经过查阅官方 issue 和文档，目前推荐的方式是：

#### Step 1：安装 esptool 工具（烧录核心）

```bash
pip3 install esptool
```

#### Step 2：使用 TinyGo 构建 ELF 文件

```bash
tinygo build -o image.elf -target=nodemcu ./
```

#### Step 3：转换为 BIN 镜像

```bash
esptool elf2image image.elf
```

执行完成后，会生成多个 .bin 文件，包含了程序所需的片段；

#### Step 4：烧录程序到 ESP8266

```bash
esptool.py --port /dev/ttyUSB0 write_flash 0x00000 image.elf-0x00000.bin -ff 80m -fm dout
 0x8000 image2.bin
```

> 这里 /dev/ttyUSB0 是我的串口设备路径，视你所用操作系统和串口驱动情况不同，可能是 /dev/ttyUSB1、COM3 等；

执行完成后，**LED 就会开始按预期规律闪烁，说明程序成功运行在 ESP8266 上了！**

## 总结

TinyGo 为我们打开了一个全新的可能性：用现代的 Go 语言，写运行在单片机上的程序；虽然它还处于不断完善的阶段，功能覆盖不如传统 C 工具链全面，但在 GPIO 控制、简单通讯、外设测试等轻量应用中已经足够稳定实用；

从这次 NodeMCU（ESP8266）开发程来看：
- 环境搭建并不复杂，按照官方文档一步步操作即可；
- 通过 VSCode 插件，可以获得接近 IDE 的开发体验；
- 即使不支持一键烧录，我们依然可以结合 esptool 手动构建、烧写；
- 最终程序运行效果与预期一致，开**发体验非常“Go风格”**，简洁、并发友好、可维护性高；

当然，也要明确 TinyGo 的边界：
- 外设支持还不完善，复杂功能（如低功耗、USB、DMA）使用门槛较高；'
- 调试工具较少，出问题时需要更多动手能力和经验支撑；
- 生态偏小众，很多问题需要翻 issue 或看源码解决；

但正因为它还在发展中，**也意味着这是一个值得探索、能够参与建设的生态**；对于 Go 开发者来说，TinyGo 是硬件世界的桥梁；对于嵌入式开发者来说，它提供了不一样的表达方式；

如果你也厌倦了反复写 while(1){}，不妨试试 for {}；**TinyGo，值得一试**；
