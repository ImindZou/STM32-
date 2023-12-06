# 窗口看门狗（WWDG）详述

[TOC]

![狗 的图像结果](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310312134786.jpeg)

> 每日一句:有些人不喜欢变化，但如果另一个选择是灾难，你需要拥抱变化。
>
> ​																																____Elon Musk

> 简介:窗口看门狗(Window Watchdog, WWDG)是F4上的另外一个看门狗,通常用来监测由外部干扰或不可预见的应用程序软件故障。这种机制能够确保系统的稳定性和可靠性，避免由于系统崩溃或死机等问题对整个系统造成的影响。
>
> 窗口看门狗，之所以称为窗口，是因为其喂狗时间是一个有上下限的范围内，你可以通过设定相关寄存器，设定其上限时间和下限时间：**喂狗的时间不能过早也不能过晚。**

## 独立看门狗的工作原理

如图所示,我们分及部分逐一解释其工作原理：

![image-20231031215300180](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310312153250.png)

### 1、递减计数器

窗口看门狗内部有一个7位的递减计数器，控制寄存器WWDG_CR中的T【6:0】位是计数器的计数值。7位计数器的时钟信号来源于PCLK1，看门狗内部首先对PCLK1进行4096分频，然后再经过可配置的预分频器分频，因此7位递减计数器的时钟频率是：

```
fcnt = fpclk1 / 4096 * DIV
```

fpclk1是时钟xhaoPCLK1的频率，4096是看门狗的固定分配系数，DIV是可设置的分配系数，由寄存器WWDG_CFR的WDGTB【1:0】位决定，DIV可取值为1、2、4、8。

![image-20231031221026607](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310312210669.png)

7位递减计数器在T6位由1变为0时，就会出发系统复位（前提是看门狗必须是激活的，也就是控制寄存器WWDG_CR中的WDGA位是1），也就是计数器由0x40变为0x3F时，产生复位。要避免系统复位，就必须在计数值变为0x3F之前重置计数器，重置计数器的值必须大于0x3F。

![image-20231031221419276](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310312214326.png)

> 注：窗口看门狗的递减计数器是自由运行计数器，即使没有开启看门狗，这个计数器也是在计数的。所以，在启动看门狗之前，应该重置计数器的值，以避免因T6位是0而立刻复位。（这点在我们初始化窗口看门狗的时候已经配置好了，只要设置的窗口阈值合理，就不会出现T6置零的错误）。

### 2、窗口值和比较器

在配置寄存器WWDG_CFR中，有七个窗口值W【6:0】，这个值用来与计数器的当前值T【6:0】进行比较。

**窗口看门狗的工作时序图如下图所示。当T【6:0】> M【6:0】时，比较器输出1，这时不允许重置计数器的值，也就是不允许写WWDG_CR，否则系统复位。只有当T【6:0】<= M【6:0】时,才可以重置计数器的值。如果在T【6:0】变化到0x3F之前没有重置计数器，就会产生系统复位信号。所以，只能在这一一个窗口器件重置看门狗计数器，这也是称为“窗口看门狗”的原因。**👻👻👻

![image-20231031223142935](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310312231003.png)

根据窗口看门狗的工作特点，在初始化设置时，窗口值W【6:0】必须小于或等于计数器的重装载值T【6:0】。窗口看门狗的超时（timeout）就是计数器重置后，计数器变化为0x3F的这段时间长度，也就是上图中不允许刷新跟允许刷新的时间总和。可以根据计数器的时钟信号频率和T【6:0】的重置值计算超时。例如，设置计数器的重置值最大为0x7F，变化到0x3F的计数周期个数就是0x40。

计数器的时钟周期是：Tcnt = 1 / fcnt = 4096DIV / fpclk1

所以，看门狗的超时是：timeout = Tsum = （4096DIV * 4F）/ fpclk1

同样也可以计算出不允许刷新的时间段的长度。

### 3、看门狗的启动

控制寄存器WWDG_CR中的位WDGA用于启动看门狗。系统复位后WDGA被硬件清零，通过向WDGA写1可启动看门狗。此外，启动看门狗后就无法停止，除非系统复位。

根据窗口看门狗的特点，用户可以使用软件使系统马上复位。具体的操作是将WDGA位置1（启动看门狗），并将T6清零（使看门狗立刻产生复位），也就是设置一个小于0x3F的重置值即可。

### 4、提前唤醒中断

窗口看门狗有一个提前唤醒中断（Early Wakeup Interrupt，EWI）事件，如果已开启此中断事件源，且启动了看门狗，在递减计数器的值变为0x40时，就会触发此中断。

用户可以在此中断服务程序里执行复位之前的一些关键操作，但是执行事件有限，只有一个计数器时钟周期。当然，用户也可以在此中断服务程序里重置计数器的值，避免复位，但是这样就违背了使用看门狗的初衷。

```c
/**
  * @brief  Refresh the WWDG.
  * @param  hwwdg  pointer to a WWDG_HandleTypeDef structure that contains
  *                the configuration information for the specified WWDG module.
  * @retval HAL status
  */
HAL_StatusTypeDef HAL_WWDG_Refresh(WWDG_HandleTypeDef *hwwdg)
{
  /* Write to WWDG CR the WWDG Counter value to refresh with */
  WRITE_REG(hwwdg->Instance->CR, (hwwdg->Init.Counter));

  /* Return function status */
  return HAL_OK;
}
```

## 窗口看门狗的HAL驱动程序

```c
HAL_StatusTypeDef     HAL_WWDG_Init(WWDG_HandleTypeDef *hwwdg);

WWDG_HandleTypeDef hwwdg;

typedef struct
{
  uint32_t Prescaler;     /*!< Specifies the prescaler value of the WWDG.
                               This parameter can be a value of @ref WWDG_Prescaler */

  uint32_t Window;        /*!< Specifies the WWDG window value to be compared to the downcounter.
                               This parameter must be a number Min_Data = 0x40 and Max_Data = 0x7F */

  uint32_t Counter;       /*!< Specifies the WWDG free-running downcounter  value.
                               This parameter must be a number between Min_Data = 0x40 and Max_Data = 0x7F */

  uint32_t EWIMode ;      /*!< Specifies if WWDG Early Wakeup Interrupt is enable or not.
                               This parameter can be a value of @ref WWDG_EWI_Mode */

} WWDG_InitTypeDef;

typedef struct
#endif /* USE_HAL_WWDG_REGISTER_CALLBACKS */
{
  WWDG_TypeDef      *Instance;  /*!< Register base address */

  WWDG_InitTypeDef  Init;       /*!< WWDG required parameters */

#if (USE_HAL_WWDG_REGISTER_CALLBACKS == 1)
  void (* EwiCallback)(struct __WWDG_HandleTypeDef *hwwdg);                  /*!< WWDG Early WakeUp Interrupt callback */

  void (* MspInitCallback)(struct __WWDG_HandleTypeDef *hwwdg);              /*!< WWDG Msp Init callback */
#endif /* USE_HAL_WWDG_REGISTER_CALLBACKS */
} WWDG_HandleTypeDef;

//刷新
HAL_StatusTypeDef     HAL_WWDG_Refresh(WWDG_HandleTypeDef *hwwdg);

//中断WWDG仅有的一个中断回调函数
__weak void HAL_WWDG_EarlyWakeupCallback(WWDG_HandleTypeDef *hwwdg)
{
  /* Prevent unused argument(s) compilation warning */
  UNUSED(hwwdg);

  /* NOTE: This function should not be modified, when the callback is needed,
           the HAL_WWDG_EarlyWakeupCallback could be implemented in the user file
   */
}

#define WWDG_IT_EWI                         WWDG_CFR_EWI  /*!< Early wakeup interrupt */

//有一个宏函数可用于开启EWI中断事件，即
#define __HAL_WWDG_ENABLE_IT(__HANDLE__, __INTERRUPT__)       SET_BIT((__HANDLE__)->Instance->CFR, (__INTERRUPT__))

*
*
*
*
*
*
//下面省略
```

## 配置

本例用到两个LED，冗余小一些，时钟PCLK1配置2MHz的一个比较低频的信号，用于看门狗递减计数器，便于观察程序运行效果。

![image-20231031232938502](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310312329571.png)

根据上面公式，可以计算出看门狗的超时为1049ms，计数器不允许刷新时间长度为442ms

![image-20231031233134500](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310312331568.png)

理解这两个时间对看门狗的意义。看门狗在启动或上次刷新后，在442ms之内不能再刷新，在443ms至1049ms之内可以刷新看门狗，如果超过1049没有刷新看门狗，看门狗就会触发系统复位。

中断采用默认优先级0即可，保证可以正常进入中断。

## Demo

```c
/* USER CODE BEGIN Header */
/**
  ******************************************************************************
  * @file           : main.c
  * @brief          : Main program body
  ******************************************************************************
  * @attention
  *
  * Copyright (c) 2023 STMicroelectronics.
  * All rights reserved.
  *
  * This software is licensed under terms that can be found in the LICENSE file
  * in the root directory of this software component.
  * If no LICENSE file comes with this software, it is provided AS-IS.
  *
  ******************************************************************************
  */
/* USER CODE END Header */
/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include "wwdg.h"
#include "gpio.h"

/* Private includes ----------------------------------------------------------*/
/* USER CODE BEGIN Includes */
#include "keyled.h"
/* USER CODE END Includes */

/* Private typedef -----------------------------------------------------------*/
/* USER CODE BEGIN PTD */

/* USER CODE END PTD */

/* Private define ------------------------------------------------------------*/
/* USER CODE BEGIN PD */

/* USER CODE END PD */

/* Private macro -------------------------------------------------------------*/
/* USER CODE BEGIN PM */

/* USER CODE END PM */

/* Private variables ---------------------------------------------------------*/

/* USER CODE BEGIN PV */

/* USER CODE END PV */

/* Private function prototypes -----------------------------------------------*/
void SystemClock_Config(void);
/* USER CODE BEGIN PFP */

/* USER CODE END PFP */

/* Private user code ---------------------------------------------------------*/
/* USER CODE BEGIN 0 */

/* USER CODE END 0 */

/**
  * @brief  The application entry point.
  * @retval int
  */
int main(void)
{
  /* USER CODE BEGIN 1 */

  /* USER CODE END 1 */

  /* MCU Configuration--------------------------------------------------------*/

  /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
  HAL_Init();

  /* USER CODE BEGIN Init */

  /* USER CODE END Init */

  /* Configure the system clock */
  SystemClock_Config();

  /* USER CODE BEGIN SysInit */

  /* USER CODE END SysInit */

  /* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_WWDG_Init();
  /* USER CODE BEGIN 2 */
	/******************Test1*****************/
//	LED1_ON();
//	LED2_ON();
/********************Test2******************/
		LED1_ON();
		LED2_OFF();
  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
    /* USER CODE END WHILE */
		/****************  Test1  *******************/
//		HAL_Delay(800);			//在允许的时间内不会触发WWDG复位，LED正常运行
////		HAL_Delay(1200);		//超时，刷新，LED1无翻转
//		HAL_WWDG_Refresh(&hwwdg);		//刷新看门狗，的CR——【T6:T0】也就是刷新CNT，！=WWDG复位
//		LED1_Toggle();
		//将延迟分开处理也是可以滴，灵活运用嘛
/****************  Test2  *******************/		
//		HAL_Delay(200);		//执行
//		LED1_Toggle();		//执行		//这些现象都是复位的初始化现象，而不是喂狗现象
//		LED2_Toggle();		//执行
//		HAL_Delay(200);		//执行
//		/*******************这段时间属于不允许刷新的时间窗口************************/
//		HAL_WWDG_Refresh(&hwwdg);		//在不允许刷新的时间窗口刷新了，会导致系统复位
//		LED1_Toggle();		//不执行	上面违法操作导致系统复位了，下面操作都死了
//		HAL_Delay(500);			//这段不会被执行
//		/************************** 刷新时间窗口******************************/
/****************  Test3  Interrupt  *******************/		
		HAL_Delay(1200);		//直接上超时		---》超时的那段时间里在复位前已经触发中断做一些加急处理，如果说引入中断喂狗的话，那么看门狗就是去了看门狗的意义了，可以这么做，但何必呢？
		LED1_Toggle();
		HAL_WWDG_Refresh(&hwwdg);		//刷新但不执行
    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
}

/**
  * @brief System Clock Configuration
  * @retval None
  */
void SystemClock_Config(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  /** Configure the main internal regulator output voltage
  */
  __HAL_RCC_PWR_CLK_ENABLE();
  __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE1);

  /** Initializes the RCC Oscillators according to the specified parameters
  * in the RCC_OscInitTypeDef structure.
  */
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;
  RCC_OscInitStruct.HSIState = RCC_HSI_ON;
  RCC_OscInitStruct.HSICalibrationValue = RCC_HSICALIBRATION_DEFAULT;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSI;
  RCC_OscInitStruct.PLL.PLLM = 8;
  RCC_OscInitStruct.PLL.PLLN = 64;
  RCC_OscInitStruct.PLL.PLLP = RCC_PLLP_DIV2;
  RCC_OscInitStruct.PLL.PLLQ = 4;
  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  /** Initializes the CPU, AHB and APB buses clocks
  */
  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                              |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV2;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV16;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_1) != HAL_OK)
  {
    Error_Handler();
  }
}

/* USER CODE BEGIN 4 */
void HAL_WWDG_EarlyWakeupCallback(WWDG_HandleTypeDef *hwwdg)
{
	LED2_ON();		//模拟复位前的加急处理
}
/* USER CODE END 4 */

/**
  * @brief  This function is executed in case of error occurrence.
  * @retval None
  */
void Error_Handler(void)
{
  /* USER CODE BEGIN Error_Handler_Debug */
  /* User can add his own implementation to report the HAL error return state */
  __disable_irq();
  while (1)
  {
  }
  /* USER CODE END Error_Handler_Debug */
}

#ifdef  USE_FULL_ASSERT
/**
  * @brief  Reports the name of the source file and the source line number
  *         where the assert_param error has occurred.
  * @param  file: pointer to the source file name
  * @param  line: assert_param error line source number
  * @retval None
  */
void assert_failed(uint8_t *file, uint32_t line)
{
  /* USER CODE BEGIN 6 */
  /* User can add his own implementation to report the file name and line number,
     ex: printf("Wrong parameters value: file %s on line %d\r\n", file, line) */
  /* USER CODE END 6 */
}
#endif /* USE_FULL_ASSERT */

```

- test1.1
    - ![test1.1](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310312349091.gif)
        - 在不超时的情况下，程序可以正常运行，看门狗递减计数器正常刷新，LED1正常翻转。
- test1.2
    - ![test1.2](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310312349454.gif)
        - 超出设定窗口阈值，程序卡死跑飞，到了规定阈值1049看门狗触发复位，LED还是原来初始化的模样，一点没变。
- test2
    - ![test2](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310312350800.gif)
        - LED1跟LED2在不能刷新的时间内正常运行，但是却在非法时间内刷新了看门狗的计数值，导致触发复位，后面的程序坏死，而看门狗因此违法操作一直重复复位，导致LED不停翻转
- test3,中间LED超时翻转
    - ![test3](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310312350445.gif)
        - 最后一个直接超时，但是我们设置了提前唤醒中断函数，该函数可在计数器来到0x40时，触发中断，用于保留一些日志消息，这里我们用LED灯的翻转模拟了一下，值得一讲的使，这个中断函数只有一个计数值的周期，也就是要尽量简短，否则超时了，什么都没了。

## 最后对比一下独立看门狗跟窗口看门狗的区别

### 使用条件对比

![image-20231101001132491](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202311010011646.png)

## 特点对比

![image-20231101003004876](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202311010030943.png)











































