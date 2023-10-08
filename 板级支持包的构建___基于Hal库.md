# 板级支持包的构建（基于Hal库）

[TOC]



> 每日一句：当一件事足够重要的时候，即使胜算不大，也要去做。
>
> **When something is important enough, even if the odds are against it, do it.**

![img](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310071714202.gif)

今天的目标是学习以下板级支持包的构建，板级支持包其实在学标准库的时候就已经接触过了，大家应该也很熟悉了了，唯一的区别点就是hal跟标准库的区别。

> 首先弄懂一下概念：

## 什么是板级支持包？

- **板级支持包（BSP）（Board Support Package）是介于主板硬件和操作系统中与驱动程序之间的一层，一般认为它属于操作系统的一部分，主要对是西安操作系统的支持，为上层的驱动程序提供访问硬件设备寄存器的函数包，使之能够更好地运行于硬件主板间。**
- ![image-20231007173459121](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310071734192.png)
- 构建板级支持包既要了解底层驱动的逻辑也要理解用户用户应用层逻辑，这样才可以实现对应的硬件接口。

## STM32CubeMX构建板级支持包

1. 初始化GPIO口
    - ![image-20231007174020748](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310071740833.png)
2. 初始化板级支持包
    - ![image-20231007174819384](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310071748452.png)
3. 支持包说明
    - **步骤跟标准库差不多，官方文档也有提供一些参考例程**
    - **打开stm32f1xx_hal_gpio.c**
    - ![image-20231007175321105](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310071753183.png)
    - **注意这些都是引用stm32f1xx.h作为中间文（支持包）实现的效果，引用其他的具体根据需求引用具体文件。**
    - ![image-20231007175647841](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310071756903.png)
    - **文件说明。**
    - ![image-20231007175851938](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310071758003.png)
4. **构建驱动程序**
    - ![image-20231007180118494](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310071801547.png)

## 编写用户层程序

![image-20231007180252814](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310071802874.png)

![image-20231007180646232](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310071806281.png)

## 实验现象

<img src="https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310071808262.gif" style="zoom:50%;" />

![img](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310071811784.gif)