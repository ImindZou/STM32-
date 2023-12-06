# 独立看门狗（IWDG）

[TOC]

![img](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310311621664.gif)

> 每日一句：**克己才是真功夫**
>
> 人须有为己之心，方能克己；能克己，方能成己。
>
> 人需要有为自己着想的心，才能克制约束自己；能够克制约束自己，才能成就自己。
>
> ​																																	————王阳明

> 简介：看门狗（Watchdog）就是MCU上的一种特殊的定时器，用于监视系统的运行，在发生错误（例如程序出现死循环）时，能自动复位。这种机制能够确保系统的稳定性和可靠性，避免由于系统崩溃或死机等问题对整个系统造成的影响。F407上面有一个独立看门狗和一个窗口看门狗，这两个看门狗的作用不一样。

## 独立看门狗的工作原理

独立看门狗（Independent WatchDog，IWDG）是由**内部32kHz低速时钟LSI驱动的自由运行的12位递减计数器。**（如下图所示。）

![image-20231031162348854](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310311623911.png)

![image-20231031163008603](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310311630666.png)

独立看门狗内部还可以对LSI进行再一次分频，分频后的时钟作为计数器的时钟信号。在预分频器寄存器IWDG_PR里，有PR【2:0】用于设置分频系数。

对于分配系数以及溢出时间的计算可以参考下图进行配置，也可以通过计算的手段自己配置出看门狗的重装载值。【注：MCU内部的LSI不是非常准确，例如F407的LSI的范围是17kHz~47kHz，所以在设置刷新看门狗的时间时，要留出一定余量。】

![image-20231031163237831](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310311632886.png)

系统复位时，IWDG的12位递减计数器的值时4095.启动IWDG后，计数器就开始递减计数，但计数器变为0时，系统就发生复位。

独立看门狗还有一个重装载寄存器IWDG_RLR，可以设置一个12位的值，例如4000，将该值重新载入计数器，以避免产生复位（这波操作简称喂狗，没来得及喂狗就发送复位了，被狗咬了。）

独立看门狗还有一个关键字寄存器，IWDG_KR,其KEY【15:0】是一个可以写入的关键字，写入不同的关键字有不同的作用。（如下图。）

![image-20231031165207922](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310311652994.png)

## 独立看门狗的HAL程序

独立看门狗的驱动函数比较少，只有两个常规函数和几个宏函数。独立看门狗没有中断。

```c
//初始化函数
HAL_StatusTypeDef     HAL_IWDG_Init(IWDG_HandleTypeDef *hiwdg);

//外设对象变量
IWDG_HandleTypeDef hiwdg;

//IWDG_HandleTypeDef结构体
typedef struct
{
  IWDG_TypeDef                 *Instance;  /*!< Register base address    */

  IWDG_InitTypeDef             Init;       /*!< IWDG required parameters */
} IWDG_HandleTypeDef;

//IWDG_InitTypeDef配置参数
typedef struct
{
  uint32_t Prescaler;  /*!< Select the prescaler of the IWDG.
                            This parameter can be a value of @ref IWDG_Prescaler */

  uint32_t Reload;     /*!< Specifies the IWDG down-counter reload value.
                            This parameter must be a number between Min_Data = 0 and Max_Data = 0x0FFF */

} IWDG_InitTypeDef;

//刷新看门狗的函数
HAL_StatusTypeDef     HAL_IWDG_Refresh(IWDG_HandleTypeDef *hiwdg);

//几个宏函数
#define __HAL_IWDG_START(__HANDLE__)                WRITE_REG((__HANDLE__)->Instance->KR, IWDG_KEY_ENABLE)

#define __HAL_IWDG_RELOAD_COUNTER(__HANDLE__)       WRITE_REG((__HANDLE__)->Instance->KR, IWDG_KEY_RELOAD)

#define IWDG_ENABLE_WRITE_ACCESS(__HANDLE__)  WRITE_REG((__HANDLE__)->Instance->KR, IWDG_KEY_WRITE_ACCESS_ENABLE)

#define IWDG_DISABLE_WRITE_ACCESS(__HANDLE__) WRITE_REG((__HANDLE__)->Instance->KR, IWDG_KEY_WRITE_ACCESS_DISABLE)

#define IWDG_KEY_RELOAD                 0x0000AAAAu  /*!< IWDG Reload Counter Enable   */
#define IWDG_KEY_ENABLE                 0x0000CCCCu  /*!< IWDG Peripheral Enable       */
#define IWDG_KEY_WRITE_ACCESS_ENABLE    0x00005555u  /*!< IWDG KR Write Access Enable  */
#define IWDG_KEY_WRITE_ACCESS_DISABLE   0x00000000u  /*!< IWDG KR Write Access Disable */
```

这些宏函数都是直接操纵寄存器的，个人认为效率还是比较高的。

## Demo

> 条件：
>
> - 配置独立看门狗的超时时间位8192ms
> - 使用RTC周期唤醒，周期为1s，使用一个全局变量Seconds进行计数
> - 按下如何一个按键时，可以看到看门狗刷新，Second置零
> - 超过8s无按键按下，系统自动复位

![image-20231031172856852](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310311728894.png)

![image-20231031173008765](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310311730814.png)

这样设置各种不论是递减计数器还是看门狗复位的窗口时间都如上图所示，单位ms。

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
#include "i2c.h"
#include "iwdg.h"
#include "rtc.h"
#include "gpio.h"

/* Private includes ----------------------------------------------------------*/
/* USER CODE BEGIN Includes */
#include "oled.h"
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
uint8_t Second =0;		//秒计数器
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
  MX_I2C2_Init();
  MX_IWDG_Init();
  MX_RTC_Init();
  /* USER CODE BEGIN 2 */
	OLED_Init();
	OLED_Clear();
	OLED_ShowString(0,0,"Hello World!",16);
	HAL_Delay(1000);
	OLED_Clear();
	OLED_ShowString(0,0,"Seconds:",16);
  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
		LED1_Toggle();
		LED2_Toggle();

    /* USER CODE END WHILE */
		KEYS curKey=ScanPressedKey(KEY_WAIT_ALWAYS);
		switch(curKey)
		{
			case KEY1:
				HAL_IWDG_Refresh(&hiwdg);		//刷新看门狗
			OLED_ShowString(0,2,"IWD was refreshed IN HAL",16);
			HAL_Delay(200);
			LED1_OFF();
			OLED_ShowString(0,2,"                        ",16);
			break;
			case KEY2:
				__HAL_IWDG_RELOAD_COUNTER(&hiwdg);	//刷新看门狗
			LED2_OFF();
			OLED_ShowString(0,2,"IWD was refreshed IN __HAL",16);
			HAL_Delay(200);
			OLED_ShowString(0,2,"                          ",16);
			break;
		}
		Second=0;


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
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSI|RCC_OSCILLATORTYPE_LSI
                              |RCC_OSCILLATORTYPE_LSE;
  RCC_OscInitStruct.LSEState = RCC_LSE_ON;
  RCC_OscInitStruct.HSIState = RCC_HSI_ON;
  RCC_OscInitStruct.HSICalibrationValue = RCC_HSICALIBRATION_DEFAULT;
  RCC_OscInitStruct.LSIState = RCC_LSI_ON;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSI;
  RCC_OscInitStruct.PLL.PLLM = 8;
  RCC_OscInitStruct.PLL.PLLN = 100;
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
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV4;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_3) != HAL_OK)
  {
    Error_Handler();
  }
}

/* USER CODE BEGIN 4 */
void HAL_RTCEx_WakeUpTimerEventCallback(RTC_HandleTypeDef *hrtc)
{
	Second++;
	OLED_ShowNum(70,0,Second,1,16);
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

现象:

![tutieshi_640x1137_17s](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310311741106.gif)

> 在我刚学习的时候就有一点疑惑，看门狗的周期那么小，那么它是如何满足在工控环境下那种需要持续工作的场合的，难道一直复位吗？

在查阅了一通后，算是迷迷糊糊知道一点吧。

有一个回答是这样的，在满足工控这种持续工作的需求方面，独立看门狗并不是一直复位系统。相反，它会在系统正常运行期间处于休眠状态，不会干扰系统的正常运行。当系统出现异常或故障时，独立看门狗会检测到并强制系统复位，以便重新启动系统并恢复正常的运行状态。

在工控应用中，独立看门狗通常具有较大的窗口周期，以确保系统的稳定运行。窗口周期的大小取决于具体的应用需求和系统配置。一些工控应用需要较长的窗口周期，以确保系统的稳定性和可靠性。在这种情况下，独立看门狗会在系统正常运行期间处于休眠状态，只有在检测到系统异常或故障时才会强制系统复位。

总之，独立看门狗在满足工控这种持续工作的需求方面，通过在系统正常运行期间处于休眠状态，只在检测到系统异常或故障时强制系统复位，确保系统的稳定性和可靠性。

仅供参考：

```c
/***********************************Step1**************************************/
//初始化独立看门狗：在系统启动时，需要初始化独立看门狗，设置窗口周期和复位向量。窗口周期是看门狗计时器的最大值，复位向量是系统在检测到故障时需要执行的代码。
// 初始化独立看门狗  
void watchdog_init() {  
    // 设置窗口周期（以毫秒为单位）  
    watchdog_set_window(5000);  
    // 设置复位向量  
    watchdog_set_reset_vector((uint32_t)reset_handler);  
    // 启动看门狗计时器  
    watchdog_start();  
}
/***********************************Step2**************************************/
//启动和停止独立看门狗：在系统正常运行期间，需要启动独立看门狗，以监测系统是否出现故障。如果系统出现故障，独立看门狗会自动复位系统。在系统不需要持续监测时，可以停止独立看门狗。
// 启动独立看门狗  
void watchdog_start() {  
    // 启动看门狗计时器  
    watchdog_timer_start();  
}  
  
// 停止独立看门狗  
void watchdog_stop() {  
    // 停止看门狗计时器  
    watchdog_timer_stop();  
}
/***********************************Step3**************************************/
//设置独立看门狗的窗口周期：窗口周期是独立看门狗计时器的最大值，需要根据具体的应用需求和系统配置进行设置。
// 设置窗口周期（以毫秒为单位）  
void watchdog_set_window(uint32_t window) {  
    // 设置看门狗计时器的窗口周期  
    watchdog_timer_set_window(window);  
}
/***********************************Step4**************************************/
//重置独立看门狗计时器：在系统正常运行期间，可以通过重置独立看门狗计时器来延长窗口周期。
// 重置独立看门狗计时器  
void watchdog_reset() {  
    // 重置看门狗计时器  
    watchdog_timer_reset();  
}
/***********************************Step5**************************************/
//看门狗超时处理：当独立看门狗检测到系统异常或故障时，会自动复位系统。此时需要实现一个复位向量（reset handler），用于处理看门狗超时事件。
// 看门狗超时处理函数（复位向量）  
void reset_handler(void) {  
    // 处理看门狗超时事件（可以在这里进行系统重启等操作）  
    // ...  
}
```

有一个缺点就是独立看门狗所在的程序出现问题后，因为独立看门狗没有中断向量表，所以只能等待看门狗重装载值（阈值）溢出后触发系统复位操作，为了解决这一方法，于是诞生了窗口看门狗。

> 先预热一下知识：

## 既然独立看门狗跟窗口看门狗都可以触发系统复位，那么它们之间又有什么区别呢？

1. 独立看门狗（IWDG）是一个独立的定时器，它不受CPU的控制，可以独立运行。它的时钟源是片内的40k RC振荡器。即使在CPU进入休眠模式时，它也可以继续工作。当定时器达到设定的阈值时，会触发系统复位。这种复位功能可以用于超低功耗的系统设计，比如在深度休眠模式下，定时器可以作为CPU的唤醒闹钟。
2. 窗口看门狗（WWDG）则是与CPU紧密相连的看门狗，它受CPU的控制。它的时钟源是由CPU的时钟分频得到的。当CPU进入休眠模式时，WWDG也可以进入休眠模式。当WWDG的定时器达到设定的阈值时，会触发中断或系统复位。这种复位功能可以用于检测和解决由软件错误引起的故障。
3. 总之，两种看门狗都有复位系统的功能，但是使用时需要根据具体的需求和应用场景选择适合的看门狗。

> 下期窗口看门狗+独立看门狗

























