# STM32串口+中断+DMA

[TOC]



## 对DMA的理解

/MANORMAL//DMECIRCTLR //循环工可以实时采集用口发送的都据，普通煤飞则是固定时间进行采集，不能达到实时的要求/实时跟固定时间两种采集方式从周期的角度来看，实时的延时事件几乎可以忽略，即使加上延时也无济于事，延时时CPO里的功能/实时模式是MA自己的速度，DMB很自我:开启普通模式就是固定时间采集了，也是D自己，但是他会照着CPU的延时来采集数据
/就不是循环模式下那个疯狂采集狂魔了。//所以说DMA，但是循环;

```c
/*while(1)放这里备份，防止被初始化*/
//	HAL_UART_Transmit(&huart1,(uint8_t*)DMA_SendDt,LENGTH,1000);		//严格遵守CPU D_CODE跟I_CODE总线的速率计算，基本举例。
//	HAL_UART_Transmit_DMA(&huart1,(uint8_t*)DMA_SendDt,LENGTH);			//(循环模式)关闭CPU对串口的运算直接卡在延时里，（普通）见uart.c里的注释
//	  HAL_Delay(500);
```

![image-20231013232845298](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310132328462.png)

```c
#include <uart.h>
#pragma  import(__use_no_semihosting)//不使用半主机模式
//避免使用半主机模式
/* USER CODE BEGIN 0 */
void _sys_exit(int x)
{
	x = x;
}
//标准库需要支持的函数
struct __FILE
{
	int handle;
}
//重定向C库函数printf到串口Transmit
int fputc(int ch,FILE *f)
{	
	HAL_UART_Transmit(&huart1,(uint8_t*)&ch,1,1000);
	
	return (ch);
}

//重定向c库函数scanf到串口
int fgetc(FILE *f)
{
	int ch;
	HAL_UART_Receive(&huart1,(uint8_t*)&ch,1,1000);
	return (ch);
}
```

![image-20231013234200904](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310132342980.png)

# 今天对C突然又有灵感

![image-20231013234759096](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310132347152.png)

![image-20231013234808884](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310132348926.png)

**其实有返回值的函数就相当于对变量的声明，其返回值对于接收返回值的变量来说接收方是可变的左值，返回值则是一个右值，被计算好的结果然后传回来。**

举个实例：内存缓冲区出问题，但是思路是对的

![image-20231014003002638](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310140030720.png)

## 挑战任务：不定长数据收发

![image-20231014151058657](https://zdh934.oss-cn-shenzhen.aliyuncs.com/PigGo/202310141510747.png)

从图中可以看到，在发送该数据帧的前后，数据线为高电平，处于IELE状态（空闲）。在该帧有效数据发送过程中，即0xaa字节与0x55字节之间，没有IDLE状态出现，这样利用IDLE状态就可以判断不定长数据的发送结束。

> 没有IDLE的回调函数，需要我们手动写一个，在it.c中写













