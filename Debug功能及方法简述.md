# Debug功能及方法简述

[TOC]

## 为什么要学会Debug（调试）

- 编写一个大的工程程序，在99%的情况下是需要经历过调试之后才能达到语气效果的。所以掌握调试的方法十分重要。
- 梳理程序的运行状态
- 因为有Bug。

## 常见的Debug方法

- ### 硬件调试：

    - 通过LED灯、分明其，能使调试人员感知到的器件，利用其交互性，进行调试。

- ### 打印调试：

    - 使用串口等，能将数据消息发送出来的方式，追踪程序运行状态。

- ### 调试器调试

    - 如果设备支持硬件内部运行状态的追踪，优先选择使用调试器调试。大多数的单片机都支持调试器调试，如STM32，可以使用DAP工具调试或者ST-Link等调试工具。

## Debug工具

- 一般情况下，集成IDE都会自带Debug工具，如：KEIL、IAR、CubeIDE、Eclipse等等。
- 这些集成IDE中自带的Debug工具，都会有可视化的界面，让Debug工作变得更为便捷高效。
- 这些Debug工具的主要功能都是大同小异的，所以重点讲解MDK-KEIL的Debug工具。

# 调试

> 忘记带DAP了，使用虚拟串口来模拟一下开启Debug。

![image-20231009150111573](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091501678.png)

- #### 下图选项是对调试进行定位处理，就比如要定位到main（）主函数里边就可以勾选到这个选项


![image-20231009152323590](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091523646.png)

## 执行单元

![image-20231009152840056](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091528100.png)

### 1、Reset

- #### **Reset按钮就是对应的复位功能，点击后系统触发硬件中断异常跳转到中断向量表里边查询复位中断，从反汇编文件里可以看出来，系统复位后是接着导入了__mian跟SystemInit的库。**

    - ![image-20231009152856967](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091528023.png)

- 回去要做一下run按钮，没带开发板跑不了。

### 2、Step One Line

- #### 步入语句，就是在原来进入函数里边逐句执行

    - ![image-20231009160219564](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091602606.png)

### 3、Step Over

- #### 跳过函数，也就是进入函数后，跳过那些逐句执行的部分

    - ![image-20231009160340678](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091603714.png)

- #### 步过语句，就是跳出函数，不进入具体函数内部，但执行相应部分

    - ![image-20231009160745243](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091607294.png)

### 4、Run to Cursor Line 

- #### 执行到光标处行（cursor游标）

    - ![image-20231009160940460](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091609498.png)

    - #### 蓝色光标为当前光标指向行，黄色光标是当前程序运行的行，它是可以往回调的，前后都可以。

        - ![image-20231009161020274](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091610317.png)

        - #### 点击Cursor后就跳转到对应的行执行刚刚指向的语句了。

            - ![image-20231009161237401](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091612451.png)
        

## 设置断点

![image-20231009194634340](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091946384.png)

1. **插入断点**
2. **使能/使能断点**
3. **使能/使能所有断点**
4. **杀死所有断点**

- **观察断点位置，我们可以清楚地看到程序程序在全速运行的情况下遇到断点就直接停止了，并且在特定的位置寄存器的值发生了一些改变，我们可以借助变化的现象跟自己预想的程序进行对比，看是否符合程序运行的预期，如果不符合就可能是哪里写错了，就可以借此调试修正我们的程序。**
    - ![image-20231009194112671](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091941781.png)

> ## **注意：结束Debug时务必将所有断点清除！KEIL软件在高版本有Bug，在退出Debug时，若有断点存在，会导致KEIL软件崩溃。**

# 调试工具

![image-20231009195109938](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091951982.png)

- #### 第一个工具跟gdb差不多，手动设置断点功能

    - ![image-20231009195315652](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091953695.png)

- #### 反汇编窗口

    - ![image-20231009195429746](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091954788.png)

- #### 标志窗口：函数/变量类型/函数参数/位置

    - ![image-20231009195741179](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310091957235.png)

- #### 寄存器窗口：指示CPU工作状态（具体参考CM3权威指南）

    - ![image-20231009200006252](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310092000296.png)

- #### 调用栈窗口：跟栈的原理一样，先入后出原则

    - ![image-20231009201013770](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310092010816.png)

- #### 监视窗口：监控变量的值

    - ![image-20231009205820371](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310092058436.png)

- #### Memory窗口：监控内存，双击还可直接修改内存

    - ![image-20231009210211186](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310092102238.png)

- #### 系统窗口：可查看内核与外设的状态

    - ![image-20231009211150610](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310092111662.png)

    - #### 内核中断表：E使能/P挂起/A响应

        - ![image-20231009211234002](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310092112046.png)

    - #### 外设：打开参考手册查看对应寄存器位进行调试排错

        - ![image-20231009211550834](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310092115879.png)

