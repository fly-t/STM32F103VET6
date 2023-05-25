# HAL_UART_Receive_IT
stm32 使用中断模式接收不定长数据, 但是数据量比较大时, 中断量依旧比较大, 对mcu的负载也比较大, 所以还推荐使用DMA模式.



# 试验硬件

- STM32F103VET6 (野火指南者)
- STLINK



# 试验设计

使用串口助手发送数据给`stm32`开发板, 然后`stm32`接收数据原路返回给串口助手.


## 0. 设置将要使用到的变量
``` c
/* 每个中断接收一个数据 */
uint8_t  rx1_data  = 0;
/* 接收数据计数 */
uint16_t rx1_count = 0;
/* 预留设置超时发送的计数 */
uint16_t rx1_timeout = 0;
/* 中断接收数据的buffer, 默认接收数据16个 */
uint8_t  rx1_buffer[16];

```


## 1. 启用中断接收

先使能中断接收模式.
``` c
	HAL_UART_Receive_IT(&huart1,&rx1_data,1);

    while (1)
    {
        
    }
```

## 2. 重写`HAL_UART_RxCpltCallback`函数
在文件`stm32f1xx_hal_uart.c`中找到以下发送回调函数, 进行重写该函数

``` c
__weak void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
  /* Prevent unused argument(s) compilation warning */
  UNUSED(huart);
  /* NOTE: This function should not be modified, when the callback is needed,
           the HAL_UART_RxCpltCallback could be implemented in the user file
   */
}
```

在`main.c`中重写需要去掉`__weak`标记, 接收的数据量最好不要超过buffer的大小, 在中断函数中接收串口助手发送过来的数据, 并拼接到buffer中.


``` c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
  UNUSED(huart);
 
   /* 判断是那个串口的中断 */
	if(huart ==&huart1){
        /* 如果是`huart1`将接收到的数据放到接收`buffer`中 */
		rx1_buffer[rx1_count++] = rx1_data;
		
		/* 如果接收到的数据量大于接收的buffer */
		if(rx1_count>=sizeof(rx1_buffer)){
			// 将超出的长度的大小归位, 最大为buffer大小.
			rx1_count = sizeof(rx1_buffer)-1;
		}
		/* 开始计时 */
		rx1_timeout =1;
		/* 重新开启中断接收 */
        HAL_UART_Receive_IT(&huart1,&rx1_data,1);

	}
}
```

## 3. 处理接收的数据

这里的操作是将接收到的数据再原路发送回去, 通常这里跟上数据的处理.


``` c
 while (1)
  {
        /* 判断是否接收到数据, 如果接收到数据, 开始计时 */
		if(rx1_timeout >0){
            /* 如果时间超过 5ms, 各个数据复位, 然后数据发送出去. */
			if(++rx1_timeout >= 5){
				rx1_timeout = 0;
				rx1_count = 0;
                HAL_UART_Transmit_IT(&huart1,rx1_buffer,rx1_count);
			}
		}
        /* 计时函数 */
		HAL_Delay(1);
    /* USER CODE BEGIN 3 */
  }
```









