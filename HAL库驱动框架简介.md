# HAL库驱动框架简介

[TOC]

# HAL库使用主线：外设初始化/外设使用

## 对外设的封装

- xx_HandleTypeDef（xx外设句柄结构体，xx表示任意外设名，如GPIO、UART等）

    - Instance（例子）成员（xx_TypeDef类型）

        - 对对应结构体的实例化，是该结构体的成员（具体的一个外设对象如GPIOA、GPIOB、串口1、串口2、IIC1、IIC2、DMA的一个通道等等。）
        - ![image-20231008233019123](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310082330212.png)

    - Init（初始化）成员（xx_InitTypeDef类型）

        - ![](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310082341877.png)

    - #### 其他资源

        - LOCK锁（HAL_LockTypeDef类型），防止资源竞争，在对外设进行初始化的时候，有些操作是不可重入的，保证操作的完整性。
            - ![image-20231008234540659](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310082345726.png)
        - STATUS状态（HAL_XX_StateTypeDef类型）
            - ![image-20231008234429946](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310082344993.png)

# 外设初始化使用方法

-  HAL_xx_Init，参数一般为xx外设的句柄结构体

    - ![image-20231008234955585](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310082349632.png)
    - 初始化外设函数，通过对xx_HandleTypeDef下的成员配置的参数，将对应的相关寄存器函数配置好。在配置寄存器前，会先调用hal_xx_mspinit函数，将底层驱动的相关资源初始化完成，如时钟、使用到的引脚、中断使能、DMA开启等。
        - ![image-20231008235624657](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310082356717.png)

- HAL_xx_MspInit,参数一般为xx外设的句柄结构体。

    - 将底层驱动的相关资源初始化完成，如时钟、使用到的引脚等。
        - ![image-20231009000115053](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310090001107.png)
        - 如果这么说的话我们平时写的板载驱动移植文件都是跟这个不一样的，但它这边是弱定义说没什么影响，系统主要还是以我们写的板载驱动为主。

- #### 其他方法

    - ![image-20231009000547352](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310090005402.png)
        - HAL库开发人员呕心沥血编写出来的参考文档

# 外设的使用逻辑

![image-20231009001652298](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310090016358.png)

## 阻塞轮询(Polling)

- xx_start

- xx_read\write

- ...等等函数。特诊，传入参数需要一个Timeout参数。**（timeout超时）**


```c
/* IO operation functions  ****************************************************/
/******* Blocking mode: Polling */
HAL_StatusTypeDef HAL_I2C_Master_Transmit(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint8_t *pData, uint16_t Size, uint32_t Timeout);
HAL_StatusTypeDef HAL_I2C_Master_Receive(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint8_t *pData, uint16_t Size, uint32_t Timeout);
HAL_StatusTypeDef HAL_I2C_Slave_Transmit(I2C_HandleTypeDef *hi2c, uint8_t *pData, uint16_t Size, uint32_t Timeout);
HAL_StatusTypeDef HAL_I2C_Slave_Receive(I2C_HandleTypeDef *hi2c, uint8_t *pData, uint16_t Size, uint32_t Timeout);
HAL_StatusTypeDef HAL_I2C_Mem_Write(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint16_t MemAddress, uint16_t MemAddSize, uint8_t *pData, uint16_t Size, uint32_t Timeout);
HAL_StatusTypeDef HAL_I2C_Mem_Read(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint16_t MemAddress, uint16_t MemAddSize, uint8_t *pData, uint16_t Size, uint32_t Timeout);
HAL_StatusTypeDef HAL_I2C_IsDeviceReady(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint32_t Trials, uint32_t Timeout);
```

## 中断(It)

- 使用终端的好处就是不占用太多资源，不像轮询阻塞那样需要占用CPU大量时间。

- xx_start_it

    - HAL_XX_IRQHandler（中断服务函数）
        - xx外设中断处理函数，在中断入口函数调用，该函数传入参数一般为 xx_HandleTypeDef,该函数中，一般会检测外设状态寄存器的标志。根据不同的标志，最终会调用不同的回调函数。
            - 各种HAL_XX_xxCallback（对应的回调函数）
                - ![image-20231009003323989](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310090033051.png)

- xx_read\write_it

- xx_xx_it...等等中断启动函数，特指，函数名以IT结尾。


```c
/******* Non-Blocking mode: Interrupt */
HAL_StatusTypeDef HAL_I2C_Master_Transmit_IT(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_I2C_Master_Receive_IT(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_I2C_Slave_Transmit_IT(I2C_HandleTypeDef *hi2c, uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_I2C_Slave_Receive_IT(I2C_HandleTypeDef *hi2c, uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_I2C_Mem_Write_IT(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint16_t MemAddress, uint16_t MemAddSize, uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_I2C_Mem_Read_IT(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint16_t MemAddress, uint16_t MemAddSize, uint8_t *pData, uint16_t Size);

HAL_StatusTypeDef HAL_I2C_Master_Seq_Transmit_IT(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint8_t *pData, uint16_t Size, uint32_t XferOptions);
HAL_StatusTypeDef HAL_I2C_Master_Seq_Receive_IT(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint8_t *pData, uint16_t Size, uint32_t XferOptions);
HAL_StatusTypeDef HAL_I2C_Slave_Seq_Transmit_IT(I2C_HandleTypeDef *hi2c, uint8_t *pData, uint16_t Size, uint32_t XferOptions);
HAL_StatusTypeDef HAL_I2C_Slave_Seq_Receive_IT(I2C_HandleTypeDef *hi2c, uint8_t *pData, uint16_t Size, uint32_t XferOptions);
HAL_StatusTypeDef HAL_I2C_EnableListen_IT(I2C_HandleTypeDef *hi2c);
HAL_StatusTypeDef HAL_I2C_DisableListen_IT(I2C_HandleTypeDef *hi2c);
HAL_StatusTypeDef HAL_I2C_Master_Abort_IT(I2C_HandleTypeDef *hi2c, uint16_t DevAddress);
```

## DMA

- xx_start_dma

    - DMA功能

- xx_read\write_dma

- xx_xx_dma...等等DMA启动函数。特征，函数名以DMA结尾。


```c
/******* Non-Blocking mode: DMA */
HAL_StatusTypeDef HAL_I2C_Master_Transmit_DMA(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_I2C_Master_Receive_DMA(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_I2C_Slave_Transmit_DMA(I2C_HandleTypeDef *hi2c, uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_I2C_Slave_Receive_DMA(I2C_HandleTypeDef *hi2c, uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_I2C_Mem_Write_DMA(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint16_t MemAddress, uint16_t MemAddSize, uint8_t *pData, uint16_t Size);
HAL_StatusTypeDef HAL_I2C_Mem_Read_DMA(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint16_t MemAddress, uint16_t MemAddSize, uint8_t *pData, uint16_t Size);

HAL_StatusTypeDef HAL_I2C_Master_Seq_Transmit_DMA(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint8_t *pData, uint16_t Size, uint32_t XferOptions);
HAL_StatusTypeDef HAL_I2C_Master_Seq_Receive_DMA(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint8_t *pData, uint16_t Size, uint32_t XferOptions);
HAL_StatusTypeDef HAL_I2C_Slave_Seq_Transmit_DMA(I2C_HandleTypeDef *hi2c, uint8_t *pData, uint16_t Size, uint32_t XferOptions);
HAL_StatusTypeDef HAL_I2C_Slave_Seq_Receive_DMA(I2C_HandleTypeDef *hi2c, uint8_t *pData, uint16_t Size, uint32_t XferOptions);
```

## 其他功能

- 标志查询\清除、中断功能使\失能、时钟使\失能
    - __HAL_xx_ENABLE_IT
    - __HAL_xx_GET_FLAG
    - ...等等

## 对HAL库驱动全面了解：

查看：

```ABAP
==============================================================================
                        ##### How to use this driver #####
  ==============================================================================
```

