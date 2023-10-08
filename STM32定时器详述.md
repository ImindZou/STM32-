# STM32定时器例程讲解

[TOC]

![img](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310061511341.gif)

> 国庆放假接近尾声，找个事情做一下，借此机会从国庆那种放松的状态拉回平时的工作状态。

## 一、定时器是什么

> 先弄懂概念，这是种事半功倍的做法。本章采取的是通用定时器。

- 通用定时器由一个通过可编程预分频器驱动的16位自动装载计数器构成
- 适用于多种场合，包括测量信号的脉冲长度（输入捕获）或者产生输出波形（输出比较和PWM）。
- 使用定时器预分频器和RCC时钟控制器预分频器，脉冲长度和波形周期可以在几个微秒到几个毫秒之间调整。
- 每个定时器都是完全独立的，没有互相共享如何资源。它们可以一起同步操作。（定时器的级联）。{所有定时器在内部相连，用于定时器同步或链接。当一个定时器处于主模式时，它可以对另一个处于从模式的定时器进行复位、启动、停止或提供时钟等操作。}
    - ![image-20231006152427419](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310061524465.png)

## 二、定时器主要的一些功能

- 定时器可以对输入时钟进行计数、并在计数值达到设定值时触发中断
- 16位计数器、预分频器、自动重装寄存器的时基单元，在72MHz计数时钟下可以实现最大59.65s的定时
- 不仅具备基本的定时中断功能，而且还包含了内外时钟源的选择、输入捕获、输出比较、编码器接口、主从触发模式等多种功能。
- 根据复杂度和应用场景可以分为高级定时器、通用定时器、基本定时器三种类型

## 三、定时器类型

| **类型**   | **编号**               | **挂载总线/接口时钟** | **功能**                                                     | 定时器时钟 |
| ---------- | ---------------------- | --------------------- | ------------------------------------------------------------ | ---------- |
| 高级定时器 | TIM1、TIM8             | APB2/72MHz            | 拥有通用定时器全部功能，并额外具有重复计数器、死区生成、互补输出、刹车输入等功能 | 72MHz      |
| 通用定时器 | TIM2、TIM3、TIM4、TIM5 | APB1/36MHz            | 拥有基本定时器全部功能，并额外具有内外时钟源选择、输入捕获、输出比较、编码器接口、主从触发模式等功能 | 72MHz      |
| 基本定时器 | TIM6、TIM7             | APB1/36MHz            | 拥有定时中断、主模式触发DAC的功能                            | 72MHz      |

> **STM32F103C8T6定时器资源：TIM1、TIM2、TIM3、TIM4**

![image-20231006154539713](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310061545776.png)

![image-20231006154603009](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310061546072.png)

![image-20231006154621936](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310061546991.png)

![image-20231006154651014](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310061546055.png)

## 四、时钟树

![image-20231006155213815](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310061552877.png)

![image-20231006161244849](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310061612928.png)

## 五、定时/计数工作原理

### 1、基本框图

![image-20231006163119227](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310061631291.png)

### 2、三种计数模式

![image-20231006164519628](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310061645685.png)

### 3、定时器中断基本结构

![image-20231006172115642](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310061721717.png)

### 4、定时器时间的计算

- 计数器计数频率：CK_CNT = CK_PSC / (PSC + 1)

    - PSC就是分频系数，（PSC+1）就是频率，至于后面相关（CNT+1）那只不过是个累加值。用来计算定时器移出时间的。时间  ！=  频率！！！

- 定时器移出时间计算：

    - 计数器载CK_CNT的驱动下，计一个数的时间是 CK_CLK的倒数：
        - 即：1/CK_CLK = 1/TIMxCLK/(PSC+1) = (PSC+1)/TIMxCLK = （PSC+1）*（ARR+1）/TIMxCLK
        - ![image-20231006181631138](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310061816304.png)

- 计数器溢出频率：

    - CK_CNT_OV = CK_CNT / (ARR + 1) = CK_PSC / (PSC + 1) / (ARR + 1)


## 五、定时器/计数功能的数据类型和接口函数

### 1、时基单元初始化类型

- 在头文件stm32f1xx_hal_tim.h中针对时基单元设置了如下结构体，它完成了定时器工作参数的配置，定义如下：
    - ![image-20231006200242760](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310062002867.png)

### 2、定时器相关接口函数

![image-20231006203358213](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310062033284.png)

1. ![image-20231006204040032](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310062040083.png)

    - ```c
        /**
          * @brief  Initializes the TIM Time base Unit according to the specified
          *         parameters in the TIM_HandleTypeDef and initialize the associated handle.
          * @note   Switching from Center Aligned counter mode to Edge counter mode (or reverse)
          *         requires a timer reset to avoid unexpected direction
          *         due to DIR bit readonly in center aligned mode.
          *         Ex: call @ref HAL_TIM_Base_DeInit() before HAL_TIM_Base_Init()
          * @param  htim TIM Base handle
          * @retval HAL status
          */
        HAL_StatusTypeDef HAL_TIM_Base_Init(TIM_HandleTypeDef *htim)
        {
          /* Check the TIM handle allocation */
          if (htim == NULL)
          {
            return HAL_ERROR;
          }
        
          /* Check the parameters */
          assert_param(IS_TIM_INSTANCE(htim->Instance));
          assert_param(IS_TIM_COUNTER_MODE(htim->Init.CounterMode));
          assert_param(IS_TIM_CLOCKDIVISION_DIV(htim->Init.ClockDivision));
          assert_param(IS_TIM_PERIOD(htim->Init.Period));
          assert_param(IS_TIM_AUTORELOAD_PRELOAD(htim->Init.AutoReloadPreload));
        
          if (htim->State == HAL_TIM_STATE_RESET)
          {
            /* Allocate lock resource and initialize it */
            htim->Lock = HAL_UNLOCKED;
        
        #if (USE_HAL_TIM_REGISTER_CALLBACKS == 1)
            /* Reset interrupt callbacks to legacy weak callbacks */
            TIM_ResetCallback(htim);
        
            if (htim->Base_MspInitCallback == NULL)
            {
              htim->Base_MspInitCallback = HAL_TIM_Base_MspInit;
            }
            /* Init the low level hardware : GPIO, CLOCK, NVIC */
            htim->Base_MspInitCallback(htim);
        #else
            /* Init the low level hardware : GPIO, CLOCK, NVIC */
            HAL_TIM_Base_MspInit(htim);
        #endif /* USE_HAL_TIM_REGISTER_CALLBACKS */
          }
        
          /* Set the TIM state */
          htim->State = HAL_TIM_STATE_BUSY;
        
          /* Set the Time Base configuration */
          TIM_Base_SetConfig(htim->Instance, &htim->Init);
        
          /* Initialize the DMA burst operation state */
          htim->DMABurstState = HAL_DMA_BURST_STATE_READY;
        
          /* Initialize the TIM channels state */
          TIM_CHANNEL_STATE_SET_ALL(htim, HAL_TIM_CHANNEL_STATE_READY);
          TIM_CHANNEL_N_STATE_SET_ALL(htim, HAL_TIM_CHANNEL_STATE_READY);
        
          /* Initialize the TIM state*/
          htim->State = HAL_TIM_STATE_READY;
        
          return HAL_OK;
        }
        ```

2. ![image-20231006210155363](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310062101405.png)

    - ```C
        /**
          * @brief  Starts the TIM Base generation.
          * @param  htim TIM Base handle
          * @retval HAL status
          */
        HAL_StatusTypeDef HAL_TIM_Base_Start(TIM_HandleTypeDef *htim)
        {
          uint32_t tmpsmcr;
        
          /* Check the parameters */
          assert_param(IS_TIM_INSTANCE(htim->Instance));
        
          /* Check the TIM state */
          if (htim->State != HAL_TIM_STATE_READY)
          {
            return HAL_ERROR;
          }
        
          /* Set the TIM state */
          htim->State = HAL_TIM_STATE_BUSY;
        
          /* Enable the Peripheral, except in trigger mode where enable is automatically done with trigger */
          if (IS_TIM_SLAVE_INSTANCE(htim->Instance))
          {
            tmpsmcr = htim->Instance->SMCR & TIM_SMCR_SMS;
            if (!IS_TIM_SLAVEMODE_TRIGGER_ENABLED(tmpsmcr))
            {
              __HAL_TIM_ENABLE(htim);
            }
          }
          else
          {
            __HAL_TIM_ENABLE(htim);
          }
        
          /* Return function status */
          return HAL_OK;
        }
        ```

3. ![image-20231006205709047](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310062057090.png)

    - ```c
        /**
          * @brief  Starts the TIM Base generation in interrupt mode.
          * @param  htim TIM Base handle
          * @retval HAL status
          */
        HAL_StatusTypeDef HAL_TIM_Base_Start_IT(TIM_HandleTypeDef *htim)
        {
          uint32_t tmpsmcr;
        
          /* Check the parameters */
          assert_param(IS_TIM_INSTANCE(htim->Instance));
        
          /* Check the TIM state */
          if (htim->State != HAL_TIM_STATE_READY)
          {
            return HAL_ERROR;
          }
        
          /* Set the TIM state */
          htim->State = HAL_TIM_STATE_BUSY;
        
          /* Enable the TIM Update interrupt */
          __HAL_TIM_ENABLE_IT(htim, TIM_IT_UPDATE);
        
          /* Enable the Peripheral, except in trigger mode where enable is automatically done with trigger */
          if (IS_TIM_SLAVE_INSTANCE(htim->Instance))
          {
            tmpsmcr = htim->Instance->SMCR & TIM_SMCR_SMS;
            if (!IS_TIM_SLAVEMODE_TRIGGER_ENABLED(tmpsmcr))
            {
              __HAL_TIM_ENABLE(htim);
            }
          }
          else
          {
            __HAL_TIM_ENABLE(htim);
          }
        
          /* Return function status */
          return HAL_OK;
        }
        ```

4. ![image-20231006210123078](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310062101120.png)

    - ```c
        /**
          * @brief  Stops the TIM Base generation.
          * @param  htim TIM Base handle
          * @retval HAL status
          */
        HAL_StatusTypeDef HAL_TIM_Base_Stop(TIM_HandleTypeDef *htim)
        {
          /* Check the parameters */
          assert_param(IS_TIM_INSTANCE(htim->Instance));
        
          /* Disable the Peripheral */
          __HAL_TIM_DISABLE(htim);
        
          /* Set the TIM state */
          htim->State = HAL_TIM_STATE_READY;
        
          /* Return function status */
          return HAL_OK;
        }
        ```

5. ![image-20231006210224683](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310062102724.png)

    - ```c
        HAL_StatusTypeDef HAL_TIM_Base_Stop_IT(TIM_HandleTypeDef *htim)
        {
          /* Check the parameters */
          assert_param(IS_TIM_INSTANCE(htim->Instance));
        
          /* Disable the TIM Update interrupt */
          __HAL_TIM_DISABLE_IT(htim, TIM_IT_UPDATE);
        
          /* Disable the Peripheral */
          __HAL_TIM_DISABLE(htim);
        
          /* Set the TIM state */
          htim->State = HAL_TIM_STATE_READY;
        
          /* Return function status */
          return HAL_OK;
        }
        ```

6. ![image-20231006210604669](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310062106725.png)

    - ```c
        /**
          * @brief  This function handles TIM interrupts requests.
          * @param  htim TIM  handle
          * @retval None
          */
        void HAL_TIM_IRQHandler(TIM_HandleTypeDef *htim)
        {
          /* Capture compare 1 event */
          if (__HAL_TIM_GET_FLAG(htim, TIM_FLAG_CC1) != RESET)
          {
            if (__HAL_TIM_GET_IT_SOURCE(htim, TIM_IT_CC1) != RESET)
            {
              {
                __HAL_TIM_CLEAR_IT(htim, TIM_IT_CC1);
                htim->Channel = HAL_TIM_ACTIVE_CHANNEL_1;
        
                /* Input capture event */
                if ((htim->Instance->CCMR1 & TIM_CCMR1_CC1S) != 0x00U)
                {
        #if (USE_HAL_TIM_REGISTER_CALLBACKS == 1)
                  htim->IC_CaptureCallback(htim);
        #else
                  HAL_TIM_IC_CaptureCallback(htim);
        #endif /* USE_HAL_TIM_REGISTER_CALLBACKS */
                }
                /* Output compare event */
                else
                {
        #if (USE_HAL_TIM_REGISTER_CALLBACKS == 1)
                  htim->OC_DelayElapsedCallback(htim);
                  htim->PWM_PulseFinishedCallback(htim);
        #else
                  HAL_TIM_OC_DelayElapsedCallback(htim);
                  HAL_TIM_PWM_PulseFinishedCallback(htim);
        #endif /* USE_HAL_TIM_REGISTER_CALLBACKS */
                }
                htim->Channel = HAL_TIM_ACTIVE_CHANNEL_CLEARED;
              }
            }
          }
          /* Capture compare 2 event */
          if (__HAL_TIM_GET_FLAG(htim, TIM_FLAG_CC2) != RESET)
          {
            if (__HAL_TIM_GET_IT_SOURCE(htim, TIM_IT_CC2) != RESET)
            {
              __HAL_TIM_CLEAR_IT(htim, TIM_IT_CC2);
              htim->Channel = HAL_TIM_ACTIVE_CHANNEL_2;
              /* Input capture event */
              if ((htim->Instance->CCMR1 & TIM_CCMR1_CC2S) != 0x00U)
              {
        #if (USE_HAL_TIM_REGISTER_CALLBACKS == 1)
                htim->IC_CaptureCallback(htim);
        #else
                HAL_TIM_IC_CaptureCallback(htim);
        #endif /* USE_HAL_TIM_REGISTER_CALLBACKS */
              }
              /* Output compare event */
              else
              {
        #if (USE_HAL_TIM_REGISTER_CALLBACKS == 1)
                htim->OC_DelayElapsedCallback(htim);
                htim->PWM_PulseFinishedCallback(htim);
        #else
                HAL_TIM_OC_DelayElapsedCallback(htim);
                HAL_TIM_PWM_PulseFinishedCallback(htim);
        #endif /* USE_HAL_TIM_REGISTER_CALLBACKS */
              }
              htim->Channel = HAL_TIM_ACTIVE_CHANNEL_CLEARED;
            }
          }
          /* Capture compare 3 event */
          if (__HAL_TIM_GET_FLAG(htim, TIM_FLAG_CC3) != RESET)
          {
            if (__HAL_TIM_GET_IT_SOURCE(htim, TIM_IT_CC3) != RESET)
            {
              __HAL_TIM_CLEAR_IT(htim, TIM_IT_CC3);
              htim->Channel = HAL_TIM_ACTIVE_CHANNEL_3;
              /* Input capture event */
              if ((htim->Instance->CCMR2 & TIM_CCMR2_CC3S) != 0x00U)
              {
        #if (USE_HAL_TIM_REGISTER_CALLBACKS == 1)
                htim->IC_CaptureCallback(htim);
        #else
                HAL_TIM_IC_CaptureCallback(htim);
        #endif /* USE_HAL_TIM_REGISTER_CALLBACKS */
              }
              /* Output compare event */
              else
              {
        #if (USE_HAL_TIM_REGISTER_CALLBACKS == 1)
                htim->OC_DelayElapsedCallback(htim);
                htim->PWM_PulseFinishedCallback(htim);
        #else
                HAL_TIM_OC_DelayElapsedCallback(htim);
                HAL_TIM_PWM_PulseFinishedCallback(htim);
        #endif /* USE_HAL_TIM_REGISTER_CALLBACKS */
              }
              htim->Channel = HAL_TIM_ACTIVE_CHANNEL_CLEARED;
            }
          }
          /* Capture compare 4 event */
          if (__HAL_TIM_GET_FLAG(htim, TIM_FLAG_CC4) != RESET)
          {
            if (__HAL_TIM_GET_IT_SOURCE(htim, TIM_IT_CC4) != RESET)
            {
              __HAL_TIM_CLEAR_IT(htim, TIM_IT_CC4);
              htim->Channel = HAL_TIM_ACTIVE_CHANNEL_4;
              /* Input capture event */
              if ((htim->Instance->CCMR2 & TIM_CCMR2_CC4S) != 0x00U)
              {
        #if (USE_HAL_TIM_REGISTER_CALLBACKS == 1)
                htim->IC_CaptureCallback(htim);
        #else
                HAL_TIM_IC_CaptureCallback(htim);
        #endif /* USE_HAL_TIM_REGISTER_CALLBACKS */
              }
              /* Output compare event */
              else
              {
        #if (USE_HAL_TIM_REGISTER_CALLBACKS == 1)
                htim->OC_DelayElapsedCallback(htim);
                htim->PWM_PulseFinishedCallback(htim);
        #else
                HAL_TIM_OC_DelayElapsedCallback(htim);
                HAL_TIM_PWM_PulseFinishedCallback(htim);
        #endif /* USE_HAL_TIM_REGISTER_CALLBACKS */
              }
              htim->Channel = HAL_TIM_ACTIVE_CHANNEL_CLEARED;
            }
          }
          /* TIM Update event */
          if (__HAL_TIM_GET_FLAG(htim, TIM_FLAG_UPDATE) != RESET)
          {
            if (__HAL_TIM_GET_IT_SOURCE(htim, TIM_IT_UPDATE) != RESET)
            {
              __HAL_TIM_CLEAR_IT(htim, TIM_IT_UPDATE);
        #if (USE_HAL_TIM_REGISTER_CALLBACKS == 1)
              htim->PeriodElapsedCallback(htim);
        #else
              HAL_TIM_PeriodElapsedCallback(htim);
        #endif /* USE_HAL_TIM_REGISTER_CALLBACKS */
            }
          }
          /* TIM Break input event */
          if (__HAL_TIM_GET_FLAG(htim, TIM_FLAG_BREAK) != RESET)
          {
            if (__HAL_TIM_GET_IT_SOURCE(htim, TIM_IT_BREAK) != RESET)
            {
              __HAL_TIM_CLEAR_IT(htim, TIM_IT_BREAK);
        #if (USE_HAL_TIM_REGISTER_CALLBACKS == 1)
              htim->BreakCallback(htim);
        #else
              HAL_TIMEx_BreakCallback(htim);
        #endif /* USE_HAL_TIM_REGISTER_CALLBACKS */
            }
          }
          /* TIM Trigger detection event */
          if (__HAL_TIM_GET_FLAG(htim, TIM_FLAG_TRIGGER) != RESET)
          {
            if (__HAL_TIM_GET_IT_SOURCE(htim, TIM_IT_TRIGGER) != RESET)
            {
              __HAL_TIM_CLEAR_IT(htim, TIM_IT_TRIGGER);
        #if (USE_HAL_TIM_REGISTER_CALLBACKS == 1)
              htim->TriggerCallback(htim);
        #else
              HAL_TIM_TriggerCallback(htim);
        #endif /* USE_HAL_TIM_REGISTER_CALLBACKS */
            }
          }
          /* TIM commutation event */
          if (__HAL_TIM_GET_FLAG(htim, TIM_FLAG_COM) != RESET)
          {
            if (__HAL_TIM_GET_IT_SOURCE(htim, TIM_IT_COM) != RESET)
            {
              __HAL_TIM_CLEAR_IT(htim, TIM_FLAG_COM);
        #if (USE_HAL_TIM_REGISTER_CALLBACKS == 1)
              htim->CommutationCallback(htim);
        #else
              HAL_TIMEx_CommutCallback(htim);
        #endif /* USE_HAL_TIM_REGISTER_CALLBACKS */
            }
          }
        }
        ```

7. ![image-20231006211140422](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310062111471.png)

    - ```c
        /**
          * @brief  Period elapsed callback in non-blocking mode
          * @param  htim TIM handle
          * @retval None
          */
        __weak void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
        {
          /* Prevent unused argument(s) compilation warning */
          UNUSED(htim);
        
          /* NOTE : This function should not be modified, when the callback is needed,
                    the HAL_TIM_PeriodElapsedCallback could be implemented in the user file
           */
        }
        ```

![img](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310062114574.gif)

# 参考例程

> 任务：使用定时器定时一秒：
>
> 根据前面所学知识，定时器计算公式应为：。。。
>
> T = ((PSC + 1) * (ARR + 1))/TIMxCLK
>
> 通用定时器TIM2，递增计数，若：
>
> PSC = 7200 - 1，ARR = 10000 - 1，T = 1s

![image-20231006212947452](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310062129517.png)

## Step：

![image-20231006213908344](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310062139401.png)

![image-20231006213958696](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310062139761.png)

![image-20231006214119365](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310062141441.png)

![image-20231006214144324](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310062141381.png)
