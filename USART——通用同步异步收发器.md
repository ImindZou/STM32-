# USART——通用同步异步收发器

[TOC]

> 每日语录：有些人不喜欢变化，但如果另一个选择是灾难，你需要拥抱变化。
>
> Some people don't like change, but if the alternative is disaster, you need to embrace change.

## UART通用同步异步收发器时产生协议信号的器件

- 通用异步收发器，英文全称Universal Asynchronous Receiver/Transmitter，简称UART。
    - STM32上的USART外设剋实现同步传输功能，所以外设名为USART，比UART多了个S，即Synchronous（同步）。
- **UART器件主要用来产生相关接口的协议信号，如RS232/RS485**等串行接口标准规范和总线标准规范，要使这些接口传输数据，就要按照接口规定的协议信号发送数据。所以UART器件广泛用于窗口通信中，扮演者着传输器的角色。

## 相关协议简介及RS232介绍

- #### 协议概念

    - 波特率（协调数据以什么速率进行发送）
        - 本章主要讲解串口异步通讯，异步通讯中由于没有时钟信号(如前面讲解的DB9接口中是没有时钟信号的)，所以两个通讯设备之间摇曳的好波特率，即每个码元的长度，以便于对信号进行解码。常见的波特率为4800、9600、115200等。
    - 通讯的起始和停止信号
        - 串口通讯的一个数据包从起始信号开始，知道结束信号结束。数据包的起始信号有一个逻辑0的数据为表示，而数据包的调制信号可由0.5、1、1.5、或2个逻辑1的数据为表示，只要双方约定移植即可。
    - 有效数据
        - 在数据包的起始位子厚紧接着就是要成熟的数据主题内容，也成为有效数据，有效数据的长度常被约定为5、6、7或8位长。
    - 数据校验
        - 在有效数据之后，有一个可选的数据校验位。由于数据通信相对比较容易受到外部干扰导致传输数据出现偏差，可以在传输过程加上校验位来解决这个问题。校验方法有奇校验(odd)/偶校验(even)/0校验(space)/1校验(mark)以及无校验（noparity）。

- #### RS-232标准

    - **一般开发板上使用的电平标准与通讯使用的电平标准不同，如TTL标准及RS-232标准。**

        - 电平标准是数据1和数据0的表达方式，是传输线缆中人为规定的电压与数据的对应关系，串口常用的电平标准有如下三种：

            - TTL电平：+3.3V或+5V表示1，0V表示0

            - RS232电平：-3~-15V表示1，+3~+15V表示0

            - RS485电平：两线压差+2~+6V表示1，-2~-6V表示0（差分信号）

        - **因为控制器一般使用TTL电平标准，所以常常会使用MA3232芯片对TTL及RS-232电平的信号进行互相转换。**

        - #### RS-232标准数据传输协议层

            - 串口通讯的数据包由发送设备通过自身的TXD接口传输到接收设备的RXD接口。在串口通讯的协议层中，规定了数据包的内容，它由起始位、主题数据、校验位以及停止位组成，通过双方的数据包格式要约定才能正常收发数据，其组成如图。
            - ![image-20231009225439768](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310092254833.png)

## STM32上的UART外设功能

**对比F1-H7差异**

- ![image-20231009235606197](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310092356313.png)
    - **F1/F2/F7**
- ![image-20231010000000117](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310100000375.png)
    - **H7**
    - FIFO（Fist Intput Fist Out 先进先出）这是个缓冲区，可配合DMA使用。

## USART使用代码讲解

- 
- 简单收发