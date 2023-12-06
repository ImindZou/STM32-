# STM32小技巧(1) HAL自定义串口中断回调 模块化代码进行解耦

[TOC]

## 前言

**用STM32CubeMx生成初始化配置代码是十分方便的，但是在处理多个项目的时候，就会发现，自动生成的中断函数调用的都是同一个回调函数，在这个回调函数中又要根据不同的句柄来处理，不同的项目外设又有所区别，还是要花点时间来进行移植。**
**比如说，我在项目A中使用到了串口1、串口2、串口3，但是在项目B中，我只使用到了串口2，并且在项目B中，串口2的功能是项目A的串口1的功能，在这种情况下，我就需要再定义一次 HAL_UART_RxCpltCallback() ，然后移植对应的功能。当然也可以重新 define 一个宏，来对应所使用的串口，这样移植上也会快许多。但是假如，后面新来了一个需求，需要实现项目C的串口3的功能，原本只要添加对应的头文件，模块处理这些外设是最简便的，但是这时候会发现，项目C中也定义了HAL_UART_RxCpltCallback()，这就需要人工去移植这部分的代码了。**
而我就在想，项目C的串口3是已经实现的模块，且这个模块是单独的.c文件和.h文件，并且不和其他的串口耦合在一起，这样的轮子造起车子来就快了许多了。但是找遍全网，会发现，关于HAL库中串口中断的教程都是基本的实现，没有更深层次的解释了，**在学习MEMS库的过程中，看到官方的代码，在使用按钮触发外部中断的时候，竟然可以自定义外部中断的回调函数，进行一番学习之后，终于掌握了自定义中断回调的方法，不仅限于串口中断函数的回调，其他的外设都是相类似的，都是可以自定义中断的回调函数。**
高手可以直接跳过第一节的内容，直接跳到第二节看如何进行操作。
一、HAL库的中断实现
中断的原理网上到处都有，也可以参考正点原子这种开源的教程。
我们正常在STM32CubeMX配置好中断后，通过 GENERATE CODE 生成代码，在 stm32f7xx_it.c 中，就会自动生成中断的处理函数，比如 USART2_IRQHandler，可以看到串口2的全局中断仅仅调用了 HAL_UART_IRQHandler 函数，该函数用于处理UART中断请求。

```c
/**
  * @brief This function handles USART2 global interrupt.
  */
void USART2_IRQHandler(void)
{
  /* USER CODE BEGIN USART2_IRQn 0 */

  /* USER CODE END USART2_IRQn 0 */
  HAL_UART_IRQHandler(&huart2);
  /* USER CODE BEGIN USART2_IRQn 1 */

  /* USER CODE END USART2_IRQn 1 */
}

```

我们进入 HAL_UART_IRQHandler 看看里面具体的实现。这个实现虽然很长，但是逻辑比较简单。首先判断是否有错误产生，没有错误就会调用 UART_Receive_IT，用于在非阻塞模式下接收大量数据。如果在接收的过程中有错误发生，那么就会进行标志位的处理，再下面一点就可以看到 USE_HAL_UART_REGISTER_CALLBACKS 这个宏定义，这就是我们自定义中断函数的关键了。如果我们定义了 USE_HAL_UART_REGISTER_CALLBACKS，那么我们通过注册自定义的中断函数后，在这里 huart->ErrorCallback(huart) 就可以直接跳转到我们的自定义的接收异常函数了，而不是 HAL_UART_ErrorCallback(huart) 这个HAL通用的接收异常处理函数。
注：F7的板子在公司，写这个教程的时候手头只有 NUCLEO-L152RE 的开发板，使用的库是 STM32Cube_FW_L1_V1.10.2。 **STM32CubeMx生成的代码有所区别，但原理上是一样的。F767ZI 是通过 huart->RxISR(huart) 这个方式来调用中断处理函数的，最终调用的是 UART_RxISR_8BIT 这样子的函数。当然后面推出了新的固件版本，也可能导致具体的实现有所区别。**

```c
void HAL_UART_IRQHandler(UART_HandleTypeDef *huart)
{
  uint32_t isrflags   = READ_REG(huart->Instance->SR);
  uint32_t cr1its     = READ_REG(huart->Instance->CR1);
  uint32_t cr3its     = READ_REG(huart->Instance->CR3);
  uint32_t errorflags = 0x00U;
  uint32_t dmarequest = 0x00U;

  /* 如果没有错误发生 */
  errorflags = (isrflags & (uint32_t)(USART_SR_PE | USART_SR_FE | USART_SR_ORE | USART_SR_NE));
  if (errorflags == RESET)
  {
    /* UART处于接收模式 -------------------------------------------------*/
    if (((isrflags & USART_SR_RXNE) != RESET) && ((cr1its & USART_CR1_RXNEIE) != RESET))
    {
      UART_Receive_IT(huart);
      return;
    }
  }

  /* 如果发生一些错误 */
  if ((errorflags != RESET) && (((cr3its & USART_CR3_EIE) != RESET) || ((cr1its & (USART_CR1_RXNEIE | USART_CR1_PEIE)) != RESET)))
  {
    /* UART parity error interrupt occurred ----------------------------------*/
    if (((isrflags & USART_SR_PE) != RESET) && ((cr1its & USART_CR1_PEIE) != RESET))
    {
      huart->ErrorCode |= HAL_UART_ERROR_PE;
    }

    /* UART noise error interrupt occurred -----------------------------------*/
    if (((isrflags & USART_SR_NE) != RESET) && ((cr3its & USART_CR3_EIE) != RESET))
    {
      huart->ErrorCode |= HAL_UART_ERROR_NE;
    }

    /* UART frame error interrupt occurred -----------------------------------*/
    if (((isrflags & USART_SR_FE) != RESET) && ((cr3its & USART_CR3_EIE) != RESET))
    {
      huart->ErrorCode |= HAL_UART_ERROR_FE;
    }

    /* UART Over-Run interrupt occurred --------------------------------------*/
    if (((isrflags & USART_SR_ORE) != RESET) && (((cr1its & USART_CR1_RXNEIE) != RESET) || ((cr3its & USART_CR3_EIE) != RESET)))
    {
      huart->ErrorCode |= HAL_UART_ERROR_ORE;
    }

    /* Call UART Error Call back function if need be --------------------------*/
    if (huart->ErrorCode != HAL_UART_ERROR_NONE)
    {
      /* UART in mode Receiver -----------------------------------------------*/
      if (((isrflags & USART_SR_RXNE) != RESET) && ((cr1its & USART_CR1_RXNEIE) != RESET))
      {
        UART_Receive_IT(huart);
      }

      /* If Overrun error occurs, or if any error occurs in DMA mode reception,
         consider error as blocking */
      dmarequest = HAL_IS_BIT_SET(huart->Instance->CR3, USART_CR3_DMAR);
      if (((huart->ErrorCode & HAL_UART_ERROR_ORE) != RESET) || dmarequest)
      {
        /* Blocking error : transfer is aborted
           Set the UART state ready to be able to start again the process,
           Disable Rx Interrupts, and disable Rx DMA request, if ongoing */
        UART_EndRxTransfer(huart);

        /* Disable the UART DMA Rx request if enabled */
        if (HAL_IS_BIT_SET(huart->Instance->CR3, USART_CR3_DMAR))
        {
          CLEAR_BIT(huart->Instance->CR3, USART_CR3_DMAR);

          /* Abort the UART DMA Rx channel */
          if (huart->hdmarx != NULL)
          {
            /* Set the UART DMA Abort callback :
               will lead to call HAL_UART_ErrorCallback() at end of DMA abort procedure */
            huart->hdmarx->XferAbortCallback = UART_DMAAbortOnError;
            if (HAL_DMA_Abort_IT(huart->hdmarx) != HAL_OK)
            {
              /* Call Directly XferAbortCallback function in case of error */
              huart->hdmarx->XferAbortCallback(huart->hdmarx);
            }
          }
          else
          {
            /* Call user error callback */
#if (USE_HAL_UART_REGISTER_CALLBACKS == 1)
            /*Call registered error callback*/
            huart->ErrorCallback(huart);
#else
            /*Call legacy weak error callback*/
            HAL_UART_ErrorCallback(huart);
#endif /* USE_HAL_UART_REGISTER_CALLBACKS */
          }
        }
        else
        {
          /* Call user error callback */
#if (USE_HAL_UART_REGISTER_CALLBACKS == 1)
          /*Call registered error callback*/
          huart->ErrorCallback(huart);
#else
          /*Call legacy weak error callback*/
          HAL_UART_ErrorCallback(huart);
#endif /* USE_HAL_UART_REGISTER_CALLBACKS */
        }
      }
      else
      {
        /* Non Blocking error : transfer could go on.
           Error is notified to user through user error callback */
#if (USE_HAL_UART_REGISTER_CALLBACKS == 1)
        /*Call registered error callback*/
        huart->ErrorCallback(huart);
#else
        /*Call legacy weak error callback*/
        HAL_UART_ErrorCallback(huart);
#endif /* USE_HAL_UART_REGISTER_CALLBACKS */

        huart->ErrorCode = HAL_UART_ERROR_NONE;
      }
    }
    return;
  } /* 如果发生错误则结束 */

  /* UART 处于发送模式，准备在非阻塞模式下发送大量数据。 ---------------------------*/
  if (((isrflags & USART_SR_TXE) != RESET) && ((cr1its & USART_CR1_TXEIE) != RESET))
  {
    UART_Transmit_IT(huart);
    return;
  }

  /*  UART 处于发送结束模式，在非阻塞模式下结束传输。 -------------------------------*/
  if (((isrflags & USART_SR_TC) != RESET) && ((cr1its & USART_CR1_TCIE) != RESET))
  {
    UART_EndTransmit_IT(huart);
    return;
  }
}

```

**然后我们看看没有接收错误发生时，调用的函数 UART_Receive_IT(huart)。这个首先根据数据位的长度来进行接收数据，（9bit和少于9bit的处理是不一样的），串口数据就保存在 huart->pRxBuffPtr，每接收一个数据，那么 huart->RxXferCount 就减一，直至为0，这时候就会触发接收完成的中断了。至于需要接收几个字节来触发中断，就取决于初始化时候的设置了。**
**huart->pRxBuffPtr变为0后，会先关闭中断，如果使能了USE_HAL_UART_REGISTER_CALLBACKS ，就会调用 huart->RxCpltCallback(huart)，我们就可以通过注册中断处理函数来跳转到我们自定义的函数，而不是HAL库本身的接收完成中断函数HAL_UART_RxCpltCallback(huart)。**而如果我们使能了USE_HAL_UART_REGISTER_CALLBACKS，却没有去注册自定义函数，最终调用的还是HAL库本身的中断处理函数，这是因为在 HAL_UART_Init 初始化中会调用函数 UART_InitCallbacksToDefault，这个函数会将所有的回调函数初始化为其默认值。
**注：接收完成后，这个会自动将关闭中断，接收完指定的字节之后，还需要继续接收的话就需要再次打开中断了。**

```c
static HAL_StatusTypeDef UART_Receive_IT(UART_HandleTypeDef *huart)
{
  uint16_t *tmp;

  /* 检查接收进程是否正在进行 */
  if (huart->RxState == HAL_UART_STATE_BUSY_RX)
  {
    if (huart->Init.WordLength == UART_WORDLENGTH_9B)
    {
      tmp = (uint16_t *) huart->pRxBuffPtr;
      if (huart->Init.Parity == UART_PARITY_NONE)
      {
        *tmp = (uint16_t)(huart->Instance->DR & (uint16_t)0x01FF);
        huart->pRxBuffPtr += 2U;
      }
      else
      {
        *tmp = (uint16_t)(huart->Instance->DR & (uint16_t)0x00FF);
        huart->pRxBuffPtr += 1U;
      }
    }
    else
    {
      if (huart->Init.Parity == UART_PARITY_NONE)
      {
        *huart->pRxBuffPtr++ = (uint8_t)(huart->Instance->DR & (uint8_t)0x00FF);
      }
      else
      {
        *huart->pRxBuffPtr++ = (uint8_t)(huart->Instance->DR & (uint8_t)0x007F);
      }
    }

    if (--huart->RxXferCount == 0U)
    {
      /* 禁止UART数据寄存器不为空的中断 */
      __HAL_UART_DISABLE_IT(huart, UART_IT_RXNE);

      /* 禁用UART奇偶校验错误中断 */
      __HAL_UART_DISABLE_IT(huart, UART_IT_PE);

      /* 禁用UART错误中断：（帧错误，噪声错误，溢出错误） */
      __HAL_UART_DISABLE_IT(huart, UART_IT_ERR);

      /* Rx进程完成，将huart-> RxState还原为Ready */
      huart->RxState = HAL_UART_STATE_READY;

#if (USE_HAL_UART_REGISTER_CALLBACKS == 1)
      /* 调用注册的Rx完成回调函数 */
      huart->RxCpltCallback(huart);
#else
      /*调用旧版弱Rx完成回调函数*/
      HAL_UART_RxCpltCallback(huart);
#endif /* USE_HAL_UART_REGISTER_CALLBACKS */

      return HAL_OK;
    }
    return HAL_OK;
  }
  else
  {
    return HAL_BUSY;
  }
}

```

## 二、使用方法

原理性的代码解析已经在前面一章节解析了，这一章节就是讲要如何实现，其他的都不难，重点是要知道 **USE_HAL_UART_REGISTER_CALLBACKS** 这个宏，其他的不懂的地方就可以根据这个线索来一步一步的追踪了。

图2.1 设置串口参数

![在这里插入图片描述](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310260844788.png)

图2.2 使能串口中断并设置优先级

![在这里插入图片描述](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310260845798.png)

**前两步都是设置串口中断的设置，按照常规来设置就可以了，重点了如何使能 USE_HAL_UART_REGISTER_CALLBACKS 这个宏，这个宏在 Project Manager 中的 Advanced Settings 右边的 Register Callback 中，选择 UART 为 ENABLE ,注意不是 USART，别选错了。**
**当然也可以在生成的项目中搜索 USE_HAL_UART_REGISTER_CALLBACKS 手工改成1，但是假如后续又通过STM32CubeMX修改配置的话，再生成代码，会将这里的配置覆盖过去，如果没注意这个点的就很容易出错，建议还是在这里修改比较稳妥一些。**
![在这里插入图片描述](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310260845535.png)

**代码生成之后，默认是没有打开串口中断的，通过 HAL_UART_RegisterCallback() 自定义回调函数，再开启中断就完成了我们的目的。**
**我们可以专门在一个.c文件来自定义接收完成的回调函数并打开中断，在main函数中调用这个初始化函数就可以了。这样就能和其他串口解耦，移植起来就很方便了。**比如我需要指定一个串口来使用 Letter Shell( github上的开源项目，我的文章中也有这个教程)，那么我就写一个 shell_port.c 文件，仅仅需要通过宏来指定使用shell 的串口句柄，其他的都不需要修改，直接添加这个 c文件，在main函数中使用 userShellInit() 来初始化，就能直接使用这个扩展模块了，相比于其他的方式，这个移植过程可以说是相当简便省事了（其他的形式需要在 usart.c 中定义串口接收的数组，再定义接收的个数，再在中断中移植自己的功能，再在main函数中打开串口中断，每建立一个新的项目都需要这么做，尤其时间长了之后，要配置串口中断的函数的时候就容易忘记函数名称，真的很烦）。

HAL_UART_RegisterCallback()
这个函数的第一个参数是 huart ，要配置的串口句柄。
这个函数的第二个参数是 CallbackID ，不同的外设所支持的回调函数ID不同，对于串口来说，HAL_UART_CallbackIDTypeDef 列出了所支持的各个功能ID，我们要在串口接收完成后调用我们自定义的ID，使用的就是 HAL_UART_RX_COMPLETE_CB_ID 。
这个函数的第三个参数是 pCallback ，我们自定义的回调函数，但是这个函数有要求，参照 pUART_CallbackTypeDef 的定义。返回应该是void，并且需要一个参数 UART_HandleTypeDef *huart。

typedef void (*pUART_CallbackTypeDef)(UART_HandleTypeDef *huart); // pointer to an UART callback function

**注：这里设置的是每接收一个字节就触发中断，但是HAL库本身处理串口中断标志位后，会关闭中断，所以我们就需要 HAL_UART_Receive_IT 再次开启中断。**

```c
#include "shell_port.h"
#include "usart.h"

Shell shell;
char shellBuffer[512];

#define userShellhuart     			huart2	//shell 使用到的串口句柄
#define SHELL_UART_REC_LEN_ONCE 	1   	//串口单次接收的个数
uint8_t uartRecBuffer[SHELL_UART_REC_LEN_ONCE];		//串口接收的buffer


/**
 * @brief 用户shell写
 * 
 * @param data 数据
 */
void userShellWrite(char data)
{
  HAL_UART_Transmit(&userShellhuart,(uint8_t *)&data, 1,1000);
}


/**
 * @brief 用户shell读
 * 
 * @param data 数据
 * @return char 状态
 */
signed char userShellRead(char *data)
{
  if(HAL_UART_Receive(&userShellhuart,(uint8_t *)data, 1, 0) == HAL_OK)
  {
      return 0;
  }
  else
  {
      return -1;
  }
}

/**
 * @brief 自定义函数串口接收完成中断 RxCpltCallback
 */
void ShellRxCpltCallback(UART_HandleTypeDef *huart)
{
  shellHandler(&shell, uartRecBuffer[0]);   //命令行处理函数
  HAL_UART_Receive_IT(huart, (uint8_t *)uartRecBuffer, SHELL_UART_REC_LEN_ONCE);//使能串口中断：标志位UART_IT_RXNE，并且设置接收缓冲以及接收缓冲接收最大数据量
}

/**
 * @brief 注册串口接收中断到自己的自定义函数
 */
void ShellReceiveCallbackRemap(void)
{
  HAL_UART_RegisterCallback(&userShellhuart,HAL_UART_RX_COMPLETE_CB_ID,ShellRxCpltCallback);    //注册串口接收中断到自己的自定义函数
  HAL_UART_Receive_IT(&userShellhuart, (uint8_t *)uartRecBuffer, SHELL_UART_REC_LEN_ONCE);      //使能串口中断：标志位UART_IT_RXNE，并且设置接收缓冲以及接收缓冲接收最大数据量
}


/**
 * @brief 用户shell初始化
 * 
 */
void userShellInit(void)
{
  shell.write = userShellWrite;         //指定串口写入的函数
  shell.read = userShellRead;           //指定串口读取的函数
  ShellReceiveCallbackRemap();          //注册串口接收中断到自己的自定义函数
  shellInit(&shell, shellBuffer, 512);
  
}

```

[源工程](https://download.csdn.net/download/qq_38942623/85022269)

代码很简单，只要看 shell_port.c 就可以了。

项目中使用到了 Letter Shell ，不懂的同学可以看下我的这篇文章。
[STM32CubeMX Nucleo F767ZI 教程(3) 串口调试工具 Letter Shell](https://blog.csdn.net/qq_38942623/article/details/113097226)

[mcujackson/letter-shell: letter shell (github.com)](https://github.com/mcujackson/letter-shell)

【上面是github开源的链接】

![image-20231026084648454](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310260846510.png)

# STM32小技巧(2) STM32CubeProgrammer解除读保护的方法

## 简述

STM32CubeProgrammer（STM32CUBEPROG）为任意环境下的STM32微控制器编程提供了
一个一体化的软件工具：多操作系统，图形用户界面或命令行界面，支持多种连接选择（JTAG、SWD、USB、UART），采用手动操作或通过脚本自动操作。

很多情况下，我们为了程序安全，都会在烧录时，使能读保护功能，这样别人就无法通过SWD/JTAG接口访问程序了。之前的STM32 ST-LINK Utility 或者 Jlink 的Segger 软件都能很方便的找到解除读保护的办法，但是新一代的烧录程序STM32CubeProgrammer就很隐蔽了，本文就是介绍了如何通过STM32CubeProgrammer进行加密和解密。

但是请注意，取消读保护后，将对Flash和备份SRAM执行全部擦除，也就无法读出之前的程序。

开启读保护
为了方便测试，这里还是要先说明如何使用STM32CubeProgrammer开启读保护。在进行读保护设置之前，先了解一下读保护的等级，
经评论区提醒。这里完善下说法，像 STM32F1 只有启用与不启用读保护这两种选项。
而STM32F4 的读保护有三个等级，可参考手册（RM0090 等）。
STM32F1 的读保护就是对应着 级别0 和级别1。下文通过 STM32CubeProgrammer 的 RDP (Read Protection) 寄存器操作可以看到区别。

可对 Flash 中的用户区域实施读保护，以防不受信任的代码读取其中的数据。读保护分三个 级别，具体定义如下：

```c
● 级别 0：无读保护
将 0xAA 写入读保护选项字节 (RDP) 时，读保护级别即设为 0。此时，在所有自举配置
（用户 Flash 自举、调试或从 RAM 自举）中，均可执行对 Flash 或备份 SRAM 的读/
写操作（如果未设置写保护）。

● 级别 1：使能读保护
这是擦除选项字节后的默认读保护级别。将任意值（分别用于设置级别 0 和级别 2 的
0xAA 和 0xCC 除外）写入 RDP 选项字节时，即激活读保护级别 1。设置读保护级别
1 后：
— 在连接调试功能或从 RAM 或系统存储器自举时，不能对 Flash 或备份 SRAM 进行
访问（读取、擦除、编程）。读请求将导致总线错误。
— 从 Flash 自举时，允许通过用户代码对 Flash 和备份 SRAM 进行访问（读取、擦
除、编程）。

激活级别 1 后，如果将保护选项字节 (RDP) 编程为级别 0，则将对 Flash 和备份 SRAM
执行全部擦除。因此，在取消读保护之前，用户代码区域会清零。批量擦除操作仅擦除
用户代码区域。包括写保护在内的其它选项字节将不受影响。OTP 区域不受批量擦除操
作的影响，同样保持不变。只有在已激活级别 1 并请求级别 0 时，才会执行批量擦除。
当提高保护级别 (0->1、1->2、0->2) 时，不会执行批量擦除。

● 级别 2：禁止调试/芯片读保护
将 0xCC 写入 RDP 选项字节时，可激活读保护级别 2。设置读保护级别 2 后：
— 级别 1 提供的所有保护均有效。
— 不再允许从 RAM 或系统存储器自举。
— JTAG、SWV（单线查看器）、ETM 和边界扫描处于禁止状态。
— 用户选项字节不能再进行更改。
— 从 Flash 自举时，允许通过用户代码对 Flash 和备份 SRAM 进行访问（读取、擦
除、编程）。
存储器读保护级别 2 是不可更改的。激活级别 2 后，保护级别不能再降回级别 0 或级别 1。
注意：激活级别 2 后，将永久性禁止 JTAG 端口（相当于 JTAG 熔断）。这样，将无法执行边
界扫描。意法半导体无法对设为保护级别 2 的器件做失效分析。

```

级别2特别狠，无法降级，以后只能在用户代码中使用外设（Uart、USB、ETH等）获取数据进行在线升级，所以STM32CubeProgrammer 或者 Segger加密都只使用到级别1，这样用户才能取消读保护。

如果在 Reference manual 的 Option bytes 明确指出有 Level 2 则说明有。像 F1 的手册中，没有提到，这时候就需要看 PM0075 Programming manual，但是这里面也只说明了处于保护与未保护的状态。
还有一种办法进行确认，就是通过 STM32CubeProgrammer 连接设备后，通过查看 Option bytes 的RDP寄存器，F4 有三个等级可选，F1 的话，只有处于与不处于读保护的状态，经过测试，使能读保护之后，可以通过 STLINK 重新取消读保护，也就是对应着保护级别1。

正常情况下，在STM32CubeProgramer通过ST-LINK连接设备后，读取到空片的数据全为0xFF,我们随便下载一个LED的测试程序，下载进去。
![在这里插入图片描述](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310260849290.png)

![在这里插入图片描述](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310260849790.png)

此时断电重启设备，或者在MCU Core选项卡中选择复位再运行，就能看到程序在运行了。

![在这里插入图片描述](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310260849795.png)

这时候我们点击 Memory & File edition，点击Read，可以正常读出程序。

![在这里插入图片描述](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310260849512.png)

接下来要进行读保护的操作。切换到 Option bytes 选项卡中，可以看到Read Out Protection 中RDP是没有勾选状态的，这表示flash memory不处于读保护的状态，然后我们要勾选它，并点击下方的Apply，然后软件就会提示选项字已经写入，我们就已经使能读保护了。

![在这里插入图片描述](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310260850929.png)

上图是 F1 的 RDP 寄存器，可以看到只有 Unchecked 和 Checked 两个选项。下图是 F4 的 RDP寄存器。可以有三个保护等级可选。

![在这里插入图片描述](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310260850405.png)

这时候切换回Memory &File edition 选项卡，进行读操作，就会被提示读取数据失败，说明我们的读保护已经生效了。

![在这里插入图片描述](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310260850926.png)

### 取消读保护

在进行读保护前，我们需要了解一下连接选项卡中的Mode的各个选项的含义。在文件UM2237中有说明，此处进行摘录。

```
模式：
– 正常：使用“Normal”连接模式时，目标是休眠的，随后挂起。使用“ResetMode”
选项来选择复位的方式
– 复位下连接：“Connect Under Reset”模式允许在执行指令之前使用复位向量捕获
连接到目标。这在很多情况下是很有用的，例如当目标包含了禁用JTAG/SWD引脚
的代码时。
– 热插拔：“Hot Plug”模式下可以在不停机或复位的情况下连接到目标。这对于在
应用运行时更新RAM地址或IP寄存器非常有用。

复位模式：
– 软件系统复位：通过Cortex-M应用中断和复位控制寄存器（AIRCR）来复位除调
试以外的所有STM32组件。
– 硬件复位：通过nRST引脚来复位STM32器件。JTAG连接器（引脚15）的RESET
引脚应连接到器件复位引脚。
– 内核复位： 通过应用中断和复位控制寄存器（AIRCR）仅将内核Cortex-M复位。

```

我们使能读保护后，也就无法通过JTAG/SWD接口访问内部flash，这对应的就是第二种情况了，所以mode我们就要选择“Connect Under Reset”，而复位模式根据情况进行选择，我使用的是SWD接口，所以选择的是"Software reset"。

![在这里插入图片描述](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310260850235.png)

这时候切换到Option bytes选项卡中，取消勾选 Read Out Protection 中的RDP寄存器，然后点击Apply。然后软件提示

"Option Bytes successfully programmed"说明我们成功取消了读保护。

![在这里插入图片描述](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310260850295.png)

然后我们就能正常的擦除烧写程序了。

![在这里插入图片描述](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310260851289.png)

![image-20231026085140172](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310260851228.png)