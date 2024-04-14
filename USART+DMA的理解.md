# USART+DMA的深层理解

[TOC]

# 一、浅谈DMA

结合今天遇到的问题，对USART+DMA的一些理解。

> ### 问题：
>
> 配置DMA的时候，配置的是循环处理，没有打开FIFP模式，导致了只有一个缓冲区，用于存储串口接收到的数据，导致数据发送的时候，存进DMA的时候基本上都是看运气，如果数据帧能够对上对应的指令，那么这个程序就可以刚好运行，程序没反应，而且还有可能就是你发一次就是对不上的时候，直接就发不过去了。
>
> 问题1：数据帧出错
>
> 问题2：接收不到

首先我们回归手册。

## 1、DMA功能描述

> DMA控制器和Cortex™-M3核心共享系统数据总线，执行直接存储器数据传输。当CPU和DMA同时访问相同的目标(RAM或外设)时，DMA请求会暂停CPU访问系统总线达若干个周期，总线仲裁器执行循环调度，以保证CPU至少可以得到一半的系统总线(存储器或外设)带宽。

可见，DMA跟RAM紧密关联，而CPU读取指令又是从RAM中读取的，对RAM中的指令一个一个地进行读取，而DMA在这方面刚好可以帮上CPU的忙，就是预先把RAM中的指令给读取出来，给CPU腾出时间去执行其他指令，需要的时候再到DMA里取出指令，这很大限度地释放了CPU的压力，但是DMA的指令有时候是不准确的，那就需要我们CPU多读取几次了，然后进行一些数据清洗，迭代出可用的正确指令。

## 2、DMA处理

> 在发生一个事件后，外设向DMA控制器发送一个请求信号。DMA控制器根据通道的优先权处理请求。当DMA控制器开始访问发出请求的外设时，DMA控制器立即发送给它一个应答信号。当从DMA控制器得到应答信号时，外设立即释放它的请求。一旦外设释放了这个请求，DMA控制器同时撤销应答信号。如果有更多的请求时，外设可以启动下一个周期。

# 二、解决过程

## 问题1：数据帧出错

一番找寻后，发现是DMA在普通模式下，发送完数据DMA通道就自动关闭了，需要手动打开通道，普通模式下的DMA还是比较精准的，只要数据过滤的好，基本上另外一端发送，已发送就可以收到正确指令了。

而循环模式下的DAM就是一个连续的数组，也就是说，如果说这个数据帧单片机来不及处理，它可能就只收到了处理到一半的数据，后面的数据就留在DMA缓冲区里了，然后你后面再发数据的话就要对上会比较困难。因为他是连续的，能不能对上纯就看运气。

刚刚说的那个连续说的有一点不准确哈，它的连续模式是基于你实际发送数据帧的长度的，我的工程里定义的是5个校验位，所以在缓冲区最多出现五个，所以数据位越长越不好校准，因为没开FIFO模式，还没试过，但感觉开起来也差不多，所以个人感觉如果是一些需要比较多校验位的验证，还是选用普通模式比较好，它每发完一次都对DMA缓冲区进行初始化，只要数据清洗那边没问题了，基本上一次就可以接收到正确的数据帧。

![循环模式DMA数据帧](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312280035459.png)

而普通模式下的DMA就每次发完数据后都会对原来的数组进行初始化，偶尔会有CPU处理不过来发送一半的情况，但一般情况下再发一次基本上就对了。

### 普通模式下的DMA缓冲区

基本上发一次就好了。

![普通模式DMA数据帧](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312280031207.png)

> ### 这些问题都是结合DMA的硬件特性进行解决的。有时候可能是代码没问题，而是配置出了问题。

## 问题2：接收不到

> #### 这个问题结合前面的DMA处理进行说明。

我们可以把上面的概念总结为一个时间到来的时候，外设向DMA发送请求信号，把外设信息存入DMA。DMA控制器转而对相应的外设发送信号，把DMA的数据发送到外设信号中去。这是不是跟串口通信的原理很像，就是这样的，但本质上有区别，硬件区别，只能说是把概念抽象了一下。

如何解决呢？

直接找库函数：

![image-20231228005224501](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312280052808.png)

在1431有一个中断函数，进去看一下。

根据描述，我们很快得知这是一个开启DMA发送模式的库函数。基于这个原理我们来看一下接收的库函数，就很容易理解为什么接收不到东西了。

![image-20231228005443467](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312280054849.png)

DMA接收函数，里边有对应的开启DMA接收的函数，我们进去看一下

![DMA接收函数](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312280059240.png)

最后一个细节，这个窗口接收的DMA函数，在普通模式下使用后，DMA通道就自动关闭了，需要我们手动打开，这个关闭的通道就是我们外设到内存的一个通道。因为通道关闭了我们进不去内存，导致接收不到数据，这也是屏幕打印不到数据的原因。

```c
/**
  * @brief  Start Receive operation in DMA mode.
  * @note   This function could be called by all HAL UART API providing reception in DMA mode.
  * @note   When calling this function, parameters validity is considered as already checked,
  *         i.e. Rx State, buffer address, ...
  *         UART Handle is assumed as Locked.
  * @param  huart UART handle.
  * @param  pData Pointer to data buffer (u8 or u16 data elements).
  * @param  Size  Amount of data elements (u8 or u16) to be received.
  * @retval HAL status
  */
HAL_StatusTypeDef UART_Start_Receive_DMA(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size)
{
  uint32_t *tmp;

  huart->pRxBuffPtr = pData;
  huart->RxXferSize = Size;

  huart->ErrorCode = HAL_UART_ERROR_NONE;
  huart->RxState = HAL_UART_STATE_BUSY_RX;

  /* Set the UART DMA transfer complete callback */
  huart->hdmarx->XferCpltCallback = UART_DMAReceiveCplt;

  /* Set the UART DMA Half transfer complete callback */
  huart->hdmarx->XferHalfCpltCallback = UART_DMARxHalfCplt;

  /* Set the DMA error callback */
  huart->hdmarx->XferErrorCallback = UART_DMAError;

  /* Set the DMA abort callback */
  huart->hdmarx->XferAbortCallback = NULL;

  /* Enable the DMA stream */
  tmp = (uint32_t *)&pData;
  HAL_DMA_Start_IT(huart->hdmarx, (uint32_t)&huart->Instance->DR, *(uint32_t *)tmp, Size);

  /* Clear the Overrun flag just before enabling the DMA Rx request: can be mandatory for the second transfer */
  __HAL_UART_CLEAR_OREFLAG(huart);

  /* Process Unlocked */
  __HAL_UNLOCK(huart);

  if (huart->Init.Parity != UART_PARITY_NONE)
  {
    /* Enable the UART Parity Error Interrupt */
    ATOMIC_SET_BIT(huart->Instance->CR1, USART_CR1_PEIE);
  }

  /* Enable the UART Error Interrupt: (Frame error, noise error, overrun error) */
  ATOMIC_SET_BIT(huart->Instance->CR3, USART_CR3_EIE);

  /* Enable the DMA transfer for the receiver request by setting the DMAR bit
  in the UART CR3 register */
  ATOMIC_SET_BIT(huart->Instance->CR3, USART_CR3_DMAR);

  return HAL_OK;
}
```

EN是开启通道，NDT是数据量，最大存放65536个bit。

这是发送前的数据，发送后程序还在照常运行，只不过DMA未给你反应。

![DMA属性](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312280117133.png)

数据发送后，我们发现通道关闭了，这时候你再怎么发送他都不理你了，这时候解决的一个方法就是重新打开通道。

![数据发送后](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312280121301.png)

手动打开通道后，我们无论怎么发送DAM的通道都是打开的，这是DMA普通模式下的一个特性，循环模式会自动打开通道。

![手动打开通道后](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312280124359.png)

DAM的库函数是这么设计的，如果DMA所附属的外设属于BUSY的状态下，他会重新配置DMA通道，也就相当于重新打开DMA通道这个概念了，算是自动打开。

这个是打开通道的方法。主要是用在DAM接收方面，感兴趣可以自己F12进去看看，原理都和上面讲的一样。

![重新打开通道](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202312280133741.png)

但手动打开也是差不多，也是在配置DMA通道，配置完通道又打开了。又可以继续接收数据了，这方面要向官方库学习一下，做了很多的配置，可以提前避免或说是预知bug的到来，进而做出对应的处理措施。很值得学习。

对UP主文章感兴趣的可以给个关注，我们一起学习嵌入式的更多知识，不定时更新文章。











