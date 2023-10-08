# STM32启动文件分析

## 启动文件简介

> 文章参考野火教程，STM32中文手册，工程启动文件

启动文件由汇编编写，是系统上电后第一时间执行的程序。主要做了以下工作：

1. 初始化堆栈指针 (__initial_sp)
2. 初始化PC指针 (Reset_Handler)
3. 初始化中断向量表 (__Vectors)
4. 配置系统时钟 (SystemInit)
5. 调用C库函数__main初始化用户堆栈，从而最终调用main函数去到C的世界。(__main)

![image-20231007205931526](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310072059638.png)

![image-20231007210012837](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310072100903.png)

## 代码分析

**Stack——栈**

![image-20231007210202973](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310072102019.png)

开辟栈的大小为0x00000400(1KB)，名字为STACK，NOINIT即不初始化，可读可写，8(2^3)字节对齐。

栈的作用是用于局部变量，函数调用，函数形参灯的开销，栈的大小不能超过内部SRAM的大小。如果编写的粗比较大，定义的局部变量很多，那么就需要修改栈的大小。如果某一天你写的程序出现莫名奇怪的错误，并进入了硬fault的时候，这是你就要考虑是不是栈不够大，溢出了。

**EQU：**宏定义的伪指令，相当于等于，类似于C中的define。（equality相等）脚本语言（-eq）

**AREA：**告诉告诉汇编器一个新的代码段或者数据段。STACK表示段名，这个可以任意命名；

**NOINIT：**表示不初始化；

**READWRITE：**表示可读可写，ALIGN=3，表示按照2……3对齐，即8字节对齐。

**SPACE：**用于分配一定大小的内存空间，单位为字节。这里指定大小等于Srack_Size.

标号__initial_sp紧挨着SPACE语句放置，表示栈的结束地址，即栈顶地址，栈是由高向低生长的。

![image-20231007211312009](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310072113065.png)

开辟堆的大小为0x00000200(512字节)，名字为HEAP，NOINIT即不初始化，可读可写，8字节对齐。__heap_base表示堆的起始地址。堆是由低向高生长的，跟栈的生长方向相反。

堆主要用来动态内存的分配，像malloc()函数申请的内存就在堆上面。这个在STM32里面用的比较少。

![image-20231007211709825](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310072117871.png)

**PRESERVE8:**指定当前文件的堆栈按照8字节对齐。（preserve）保持

**THUMB：**表示后面的指令兼容THUMB指令。THUMB是ARM以前的指令集，16bit，现在Cortex-M系列的使用都是用THUMB-2指令集，THUMB-2是32位的，兼容16位和32位的指令，是THUMB的超集。

## 向量表

![image-20231007212101646](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310072121691.png)

**定义一个数据段，名字为RESET，只读。并声明__Vectors、**

**__Vectors_End**

**和__Vectors_Size这三个标号具有全局属性，可供外部文件调用。**

**EXPORT：**声明一个标号可被外部文件调用，使标号具有全局属性。如果IAR编译器，则使用的使CLOBAL这个指令。

当内核响应一个发生的异常后，对应的异常服务例程(ESR)就会执行。为了决定ESR的入口地址，内核使用了“向量表查表机制”。这里使用一张向量表。向量表其实是一个WORD(32位整数)数组，每个下表对应一种异常，该下标元素的值则是该ESR的入口地址。向量表在地址空间中的位置是可以设置的，通过NVIC中的一个重定位寄存器来指出向量表的地址。在复位后，该寄存器的值为0.因此，在地址0(即FLASH地址0)处必须包含一张向量表，用于初始时的异常分配。要注意的是这里有个另类：0号类型并不是什么入口地址，而是给出了复位后的MSP的初值。

![image-20231007213005237](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310072130304.png)

```asm
__Vectors       DCD     __initial_sp               ; Top of Stack
                DCD     Reset_Handler              ; Reset Handler
                DCD     NMI_Handler                ; NMI Handler
                DCD     HardFault_Handler          ; Hard Fault Handler
                DCD     MemManage_Handler          ; MPU Fault Handler
                DCD     BusFault_Handler           ; Bus Fault Handler
                DCD     UsageFault_Handler         ; Usage Fault Handler
                DCD     0                          ; Reserved
                DCD     0                          ; Reserved
                DCD     0                          ; Reserved
                DCD     0                          ; Reserved
                DCD     SVC_Handler                ; SVCall Handler
                DCD     DebugMon_Handler           ; Debug Monitor Handler
                DCD     0                          ; Reserved
                DCD     PendSV_Handler             ; PendSV Handler
                DCD     SysTick_Handler            ; SysTick Handler

                ; External Interrupts
                DCD     WWDG_IRQHandler            ; Window Watchdog
                DCD     PVD_IRQHandler             ; PVD through EXTI Line detect
                DCD     TAMPER_IRQHandler          ; Tamper
                DCD     RTC_IRQHandler             ; RTC
                DCD     FLASH_IRQHandler           ; Flash
                DCD     RCC_IRQHandler             ; RCC
                DCD     EXTI0_IRQHandler           ; EXTI Line 0
                DCD     EXTI1_IRQHandler           ; EXTI Line 1
                DCD     EXTI2_IRQHandler           ; EXTI Line 2
                DCD     EXTI3_IRQHandler           ; EXTI Line 3
                DCD     EXTI4_IRQHandler           ; EXTI Line 4
                DCD     DMA1_Channel1_IRQHandler   ; DMA1 Channel 1
                DCD     DMA1_Channel2_IRQHandler   ; DMA1 Channel 2
                DCD     DMA1_Channel3_IRQHandler   ; DMA1 Channel 3
                DCD     DMA1_Channel4_IRQHandler   ; DMA1 Channel 4
                DCD     DMA1_Channel5_IRQHandler   ; DMA1 Channel 5
                DCD     DMA1_Channel6_IRQHandler   ; DMA1 Channel 6
                DCD     DMA1_Channel7_IRQHandler   ; DMA1 Channel 7
                DCD     ADC1_2_IRQHandler          ; ADC1 & ADC2
                DCD     USB_HP_CAN1_TX_IRQHandler  ; USB High Priority or CAN1 TX
                DCD     USB_LP_CAN1_RX0_IRQHandler ; USB Low  Priority or CAN1 RX0
                DCD     CAN1_RX1_IRQHandler        ; CAN1 RX1
                DCD     CAN1_SCE_IRQHandler        ; CAN1 SCE
                DCD     EXTI9_5_IRQHandler         ; EXTI Line 9..5
                DCD     TIM1_BRK_IRQHandler        ; TIM1 Break
                DCD     TIM1_UP_IRQHandler         ; TIM1 Update
                DCD     TIM1_TRG_COM_IRQHandler    ; TIM1 Trigger and Commutation
                DCD     TIM1_CC_IRQHandler         ; TIM1 Capture Compare
                DCD     TIM2_IRQHandler            ; TIM2
                DCD     TIM3_IRQHandler            ; TIM3
                DCD     TIM4_IRQHandler            ; TIM4
                DCD     I2C1_EV_IRQHandler         ; I2C1 Event
                DCD     I2C1_ER_IRQHandler         ; I2C1 Error
                DCD     I2C2_EV_IRQHandler         ; I2C2 Event
                DCD     I2C2_ER_IRQHandler         ; I2C2 Error
                DCD     SPI1_IRQHandler            ; SPI1
                DCD     SPI2_IRQHandler            ; SPI2
                DCD     USART1_IRQHandler          ; USART1
                DCD     USART2_IRQHandler          ; USART2
                DCD     USART3_IRQHandler          ; USART3
                DCD     EXTI15_10_IRQHandler       ; EXTI Line 15..10
                DCD     RTC_Alarm_IRQHandler        ; RTC Alarm through EXTI Line
                DCD     USBWakeUp_IRQHandler       ; USB Wakeup from suspend
                DCD     TIM8_BRK_IRQHandler        ; TIM8 Break
                DCD     TIM8_UP_IRQHandler         ; TIM8 Update
                DCD     TIM8_TRG_COM_IRQHandler    ; TIM8 Trigger and Commutation
                DCD     TIM8_CC_IRQHandler         ; TIM8 Capture Compare
                DCD     ADC3_IRQHandler            ; ADC3
                DCD     FSMC_IRQHandler            ; FSMC
                DCD     SDIO_IRQHandler            ; SDIO
                DCD     TIM5_IRQHandler            ; TIM5
                DCD     SPI3_IRQHandler            ; SPI3
                DCD     UART4_IRQHandler           ; UART4
                DCD     UART5_IRQHandler           ; UART5
                DCD     TIM6_IRQHandler            ; TIM6
                DCD     TIM7_IRQHandler            ; TIM7
                DCD     DMA2_Channel1_IRQHandler   ; DMA2 Channel1
                DCD     DMA2_Channel2_IRQHandler   ; DMA2 Channel2
                DCD     DMA2_Channel3_IRQHandler   ; DMA2 Channel3
                DCD     DMA2_Channel4_5_IRQHandler ; DMA2 Channel4 & Channel5
__Vectors_End

__Vectors_Size  EQU  __Vectors_End - __Vectors
```

__Vectors为向量表起始地址，

__Vectors_End为向量表结束地址，两个相减即可算出向量表的大小。向量表从FLASH的0开始放置，以4字节为一个单位，地址为0存放的是栈顶指针（SP），0x04存放的是复位程序地址，以此类推。从代码上看，向量表中存放的都是中断服务函数的函数名，可我们知道C语言中函数名就是一个地址。

**DCD：**分配一个或多个以字为单位的内存，以四字节对齐，并要求初始化这些内存。在向量表中DCD分配了一堆内存，并以ESR的入口地址初始化它们。

## 复位程序

![image-20231007214241980](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310072142031.png)

定义一个名为.text的代码段，只读。

![image-20231007214317874](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310072143923.png)

复位子程序是系统上电后执行的第一个程序，调用StstemInit函数初始化系统时钟，然后调用C库函数__main，最终调用main函数去到C的世界。

**WEAK：**表示弱定义，如果外部文件优先定义了该标号则首先引用该标号，如果外部文件没有声明也不会出错。这里表示复位子程序可以由用户在其他文件重新实现，这里并不是唯一的。

**IMPORT：**表示该标号来自外部文件，跟C语言中的EXTERN关键字类似。这里表示SysyemInit和__mian这两个函数均来自外部文件。

SysremInit()是一个标准库函数，在system_stm32f103xe.c这个库文件中定义。主要作用是配置系统时钟，这里调用这个函数之后，单片机的系统时钟被配置为72M。

__main是一个标准C库函数，主要作用是初始化用户堆栈，并在函数的最后嗲用main函数去到C的世界。扎尔就是为什么我们写的程序都有一个main函数的原因。

**LDR   BLX    BX**  是CM4内核的指令，上网查，加载指令的，跳转指令，具体作用见下表：

![image-20231007215222618](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310072152663.png)

## 中断服务程序

在启动文件里面已经帮我们写好了所有中断服务函数，跟我们平时写的中断服务函数不一样的就是这些中断服务函数都是空的，真正的中断服务程序需要我们在外部的C文件里面重新实现，这里只是提前占了一个位置而已。

**如果我们在使用某个外设的时候，开启了某个中断，但是又忘记编写配套中断服务函数或函数名写错，当中断来临时，程序就会跳转到启动文件预先写好的空的终端服务程序中，并且在这个空函数中无限循环，即程序就死在这里。**

```asm
; Dummy Exception Handlers (infinite loops which can be modified)

NMI_Handler     PROC
                EXPORT  NMI_Handler                [WEAK]
                B       .
                ENDP
HardFault_Handler\
                PROC
                EXPORT  HardFault_Handler          [WEAK]
                B       .
                ENDP
MemManage_Handler\
                PROC
                EXPORT  MemManage_Handler          [WEAK]
                B       .
                ENDP
BusFault_Handler\
                PROC
                EXPORT  BusFault_Handler           [WEAK]
                B       .
                ENDP
UsageFault_Handler\
                PROC
                EXPORT  UsageFault_Handler         [WEAK]
                B       .
                ENDP
SVC_Handler     PROC
                EXPORT  SVC_Handler                [WEAK]
                B       .
                ENDP
DebugMon_Handler\
                PROC
                EXPORT  DebugMon_Handler           [WEAK]
                B       .
                ENDP
PendSV_Handler  PROC
                EXPORT  PendSV_Handler             [WEAK]
                B       .
                ENDP
SysTick_Handler PROC
                EXPORT  SysTick_Handler            [WEAK]
                B       .
                ENDP

Default_Handler PROC

                EXPORT  WWDG_IRQHandler            [WEAK]
                EXPORT  PVD_IRQHandler             [WEAK]
                EXPORT  TAMPER_IRQHandler          [WEAK]
                EXPORT  RTC_IRQHandler             [WEAK]
                EXPORT  FLASH_IRQHandler           [WEAK]
                EXPORT  RCC_IRQHandler             [WEAK]
                EXPORT  EXTI0_IRQHandler           [WEAK]
                EXPORT  EXTI1_IRQHandler           [WEAK]
                EXPORT  EXTI2_IRQHandler           [WEAK]
                EXPORT  EXTI3_IRQHandler           [WEAK]
                EXPORT  EXTI4_IRQHandler           [WEAK]
                EXPORT  DMA1_Channel1_IRQHandler   [WEAK]
                EXPORT  DMA1_Channel2_IRQHandler   [WEAK]
                EXPORT  DMA1_Channel3_IRQHandler   [WEAK]
                EXPORT  DMA1_Channel4_IRQHandler   [WEAK]
                EXPORT  DMA1_Channel5_IRQHandler   [WEAK]
                EXPORT  DMA1_Channel6_IRQHandler   [WEAK]
                EXPORT  DMA1_Channel7_IRQHandler   [WEAK]
                EXPORT  ADC1_2_IRQHandler          [WEAK]
                EXPORT  USB_HP_CAN1_TX_IRQHandler  [WEAK]
                EXPORT  USB_LP_CAN1_RX0_IRQHandler [WEAK]
                EXPORT  CAN1_RX1_IRQHandler        [WEAK]
                EXPORT  CAN1_SCE_IRQHandler        [WEAK]
                EXPORT  EXTI9_5_IRQHandler         [WEAK]
                EXPORT  TIM1_BRK_IRQHandler        [WEAK]
                EXPORT  TIM1_UP_IRQHandler         [WEAK]
                EXPORT  TIM1_TRG_COM_IRQHandler    [WEAK]
                EXPORT  TIM1_CC_IRQHandler         [WEAK]
                EXPORT  TIM2_IRQHandler            [WEAK]
                EXPORT  TIM3_IRQHandler            [WEAK]
                EXPORT  TIM4_IRQHandler            [WEAK]
                EXPORT  I2C1_EV_IRQHandler         [WEAK]
                EXPORT  I2C1_ER_IRQHandler         [WEAK]
                EXPORT  I2C2_EV_IRQHandler         [WEAK]
                EXPORT  I2C2_ER_IRQHandler         [WEAK]
                EXPORT  SPI1_IRQHandler            [WEAK]
                EXPORT  SPI2_IRQHandler            [WEAK]
                EXPORT  USART1_IRQHandler          [WEAK]
                EXPORT  USART2_IRQHandler          [WEAK]
                EXPORT  USART3_IRQHandler          [WEAK]
                EXPORT  EXTI15_10_IRQHandler       [WEAK]
                EXPORT  RTC_Alarm_IRQHandler        [WEAK]
                EXPORT  USBWakeUp_IRQHandler       [WEAK]
                EXPORT  TIM8_BRK_IRQHandler        [WEAK]
                EXPORT  TIM8_UP_IRQHandler         [WEAK]
                EXPORT  TIM8_TRG_COM_IRQHandler    [WEAK]
                EXPORT  TIM8_CC_IRQHandler         [WEAK]
                EXPORT  ADC3_IRQHandler            [WEAK]
                EXPORT  FSMC_IRQHandler            [WEAK]
                EXPORT  SDIO_IRQHandler            [WEAK]
                EXPORT  TIM5_IRQHandler            [WEAK]
                EXPORT  SPI3_IRQHandler            [WEAK]
                EXPORT  UART4_IRQHandler           [WEAK]
                EXPORT  UART5_IRQHandler           [WEAK]
                EXPORT  TIM6_IRQHandler            [WEAK]
                EXPORT  TIM7_IRQHandler            [WEAK]
                EXPORT  DMA2_Channel1_IRQHandler   [WEAK]
                EXPORT  DMA2_Channel2_IRQHandler   [WEAK]
                EXPORT  DMA2_Channel3_IRQHandler   [WEAK]
                EXPORT  DMA2_Channel4_5_IRQHandler [WEAK]

WWDG_IRQHandler
PVD_IRQHandler
TAMPER_IRQHandler
RTC_IRQHandler
FLASH_IRQHandler
RCC_IRQHandler
EXTI0_IRQHandler
EXTI1_IRQHandler
EXTI2_IRQHandler
EXTI3_IRQHandler
EXTI4_IRQHandler
DMA1_Channel1_IRQHandler
DMA1_Channel2_IRQHandler
DMA1_Channel3_IRQHandler
DMA1_Channel4_IRQHandler
DMA1_Channel5_IRQHandler
DMA1_Channel6_IRQHandler
DMA1_Channel7_IRQHandler
ADC1_2_IRQHandler
USB_HP_CAN1_TX_IRQHandler
USB_LP_CAN1_RX0_IRQHandler
CAN1_RX1_IRQHandler
CAN1_SCE_IRQHandler
EXTI9_5_IRQHandler
TIM1_BRK_IRQHandler
TIM1_UP_IRQHandler
TIM1_TRG_COM_IRQHandler
TIM1_CC_IRQHandler
TIM2_IRQHandler
TIM3_IRQHandler
TIM4_IRQHandler
I2C1_EV_IRQHandler
I2C1_ER_IRQHandler
I2C2_EV_IRQHandler
I2C2_ER_IRQHandler
SPI1_IRQHandler
SPI2_IRQHandler
USART1_IRQHandler
USART2_IRQHandler
USART3_IRQHandler
EXTI15_10_IRQHandler
RTC_Alarm_IRQHandler
USBWakeUp_IRQHandler
TIM8_BRK_IRQHandler
TIM8_UP_IRQHandler
TIM8_TRG_COM_IRQHandler
TIM8_CC_IRQHandler
ADC3_IRQHandler
FSMC_IRQHandler
SDIO_IRQHandler
TIM5_IRQHandler
SPI3_IRQHandler
UART4_IRQHandler
UART5_IRQHandler
TIM6_IRQHandler
TIM7_IRQHandler
DMA2_Channel1_IRQHandler
DMA2_Channel2_IRQHandler
DMA2_Channel3_IRQHandler
DMA2_Channel4_5_IRQHandler
                B       .		"无限循环

                ENDP
```

**B:**跳转到一个标号。这里跳转到一个‘ . ’，即无限循环。

## 用户堆栈初始化

**ALIGN：**对指令或数据存放的地址进行对齐，后面会跟一个立即数。缺省表示4个字节对齐。

```
 				ALIGN

;*******************************************************************************
; User Stack and Heap initialization
;*******************************************************************************
                 IF      :DEF:__MICROLIB
                
                 EXPORT  __initial_sp
                 EXPORT  __heap_base
                 EXPORT  __heap_limit
                
                 ELSE
                
                 IMPORT  __use_two_region_memory
                 EXPORT  __user_initial_stackheap
                 
__user_initial_stackheap

                 LDR     R0, =  Heap_Mem
                 LDR     R1, =(Stack_Mem + Stack_Size)
                 LDR     R2, = (Heap_Mem +  Heap_Size)
                 LDR     R3, = Stack_Mem
                 BX      LR

                 ALIGN

                 ENDIF

                 END
```

首先判断是否定义了__MICROLIB，如果定义了这个宏则赋予标号

__initial_sp(栈顶指针）

__heap_base(堆起始地址）

__heap_limit(堆结束地址)全局属性，可供外部文件调用。有关这个宏我们在KEIL里配置，见下图/然后堆栈初始化就由C库函数

__main来完成。

![image-20231007220535192](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310072205261.png)

如果没有定义__MICROLIB,则插入标号

__use_two_region_memory,这个函数需要用户自己实现，具体要实现成什么样，可在KEIL的帮助文档里查询到：

然后声明标号__user_initial_stackheap 具有全局属性，可供外部文件调用，并实现这个标号的内
容。
IF,ELSE,ENDIF ：汇编的条件分支语句，跟C 语言的if ,else 类似
END ：文件结束。