

#  STM32复习快速掌握笔记：GPIO篇

##  0. 核心基础知识
在写代码之前，必须熟记 **GPIO的8种工作模式**（面试与工程最常考）：
*   **输入模式 (Input)：**
    *   `Analog` (模拟输入)：用于ADC采集。
    *   `Floating` (浮空输入)：电平由外部决定，多用于按键（外部带上下拉）或某些通信线。
    *   `Pull-Up (IPU)` (上拉输入)：内部接上拉电阻，默认高电平（**适合接按键到GND**）。
    *   `Pull-Down (IPD)` (下拉输入)：内部接下拉电阻，默认低电平。
*   **输出模式 (Output)：**
    *   `Push-Pull (PP)` (推挽输出)：能输出强高/低电平，驱动能力强（**最常用**）。
    *   `Open-Drain (OD)` (开漏输出)：只能输出低电平，输出高电平需外接上拉电阻（适合I2C或驱动不同电压的外设）。
*   **复用功能 (Alternate Function)：**
    *   `AF_PP` (复用推挽)：交由外设（如串口TX、SPI）控制的推挽。
    *   `AF_OD` (复用开漏)：交由外设（如I2C SCL/SDA）控制的开漏。

---

## 1. 标准库写法 (Standard Peripheral Library)
> **硬件假设**：PA1接按键（按下接地，所以用上拉输入），PC13接LED（假设外部接VCC，所以用开漏或推挽输出，低电平点亮）。

```c
#include <stm32f10x.h>

int main(void)
{
    /* 1. 开启时钟：GPIO挂载在APB2总线上 */
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    /* 2. 配置GPIOC (PC13 - 控制LED) */
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_13;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_Out_OD;   // 开漏输出 (推挽Out_PP也常用)
    GPIO_InitStruct.GPIO_Speed = GPIO_Speed_2MHz;   // 输出速度
    GPIO_Init(GPIOC, &GPIO_InitStruct);

    /* 3. 配置GPIOA (PA1 - 检测按键) */
    // 注意：标准库不需要重新定义结构体，直接修改参数即可复用
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_1;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_IPU;      // 上拉输入 (按键未按下时为高电平)
    // 输入模式下无需配置Speed参数
    GPIO_Init(GPIOA, &GPIO_InitStruct);

    /* 4. 主循环逻辑 */
    while(1)
    {
        // 读取PA1电平：如果是低电平 (Bit_RESET)，说明按键按下
        if(GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_1) == Bit_RESET)
        {
            // 输出高电平 (熄灭LED，假设PC13低电平亮)
            GPIO_WriteBit(GPIOC, GPIO_Pin_13, Bit_SET);
            
            /* 补充：也可以用这种写法，更常用：
               GPIO_SetBits(GPIOC, GPIO_Pin_13); */
        }
        else
        {
            // 输出低电平 (点亮LED)
            GPIO_WriteBit(GPIOC, GPIO_Pin_13, Bit_RESET);
            
            /* 补充：也可以用这种写法，更常用：
               GPIO_ResetBits(GPIOC, GPIO_Pin_13); */
        }
    }
}
```

---

## 💻 2. HAL库写法 (Hardware Abstraction Layer)

> 1. HAL库中，输入输出相关的参数都在一个结构体里，复用结构体时建议重新清零。


```c
#include "stm32f1xx_hal.h"


void SystemClock_Config(void);
void Error_Handler(void);

int main(void)
{
    /* 1. 初始化HAL库 (重置所有外设，初始化Flash接口和Systick) */
    HAL_Init();

    /* 2. 配置系统时钟 */
    SystemClock_Config();
    
    /* 3. 开启GPIO时钟 */
    __HAL_RCC_GPIOC_CLK_ENABLE();
    __HAL_RCC_GPIOA_CLK_ENABLE();

    GPIO_InitTypeDef GPIO_InitStruct = {0};

    /* 4. 配置GPIOC (PC13 - 控制LED) */
    GPIO_InitStruct.Pin = GPIO_PIN_13;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_OD;   // 开漏输出
    GPIO_InitStruct.Pull = GPIO_NOPULL;           // 输出模式一般不配置内部上下拉
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;  // 低速即可
    HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

    /* 5. 配置GPIOA (PA1 - 检测按键) */
    GPIO_InitStruct.Pin = GPIO_PIN_1;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;       // 输入模式
    GPIO_InitStruct.Pull = GPIO_PULLUP;           // 内部上拉 (硬件按键接地时使用)
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    /* 6. 主循环逻辑 */
    while(1)
    {
        // 读取PA1，判断是否按下 (低电平)
        if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_1) == GPIO_PIN_RESET)
        {
            // 按下按键，输出高电平
            HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);
            
            /* 补充：如果是想做按键翻转状态，可以使用：
               HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13); 
               HAL_Delay(200); // 简单的软件消抖 */
        }
        else
        {
            // 松开按键，输出低电平
            HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET);
        }
    }
}

/* 系统时钟配置 (通常由CubeMX生成) */
void SystemClock_Config(void)
{
    RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

    // 配置外部高速晶振(HSE)和PLL锁相环
    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState = RCC_HSE_ON;
    RCC_OscInitStruct.HSEPredivValue = RCC_HSE_PREDIV_DIV1;
    RCC_OscInitStruct.HSIState = RCC_HSI_ON;
    RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
    RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
    RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL9; // 8MHz * 9 = 72MHz
    
    if(HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK){
        Error_Handler();
    }

    // 配置各种总线时钟的分频系数
    RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK | RCC_CLOCKTYPE_SYSCLK 
                                | RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2;
    RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
    RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;      // AHB 72MHz
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;       // APB1 36MHz (最大36M)
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;       // APB2 72MHz

    if(HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2) != HAL_OK){
        Error_Handler();
    }
}

/* 错误处理函数 (死循环) */
void Error_Handler(void)
{
    while(1){}
}
```

---

## 📑 3. 备忘查表：标准库 vs HAL库 常用函数对比

在复习和笔试/机试时，记住以下映射关系可以极大地提高开发速度：

| 功能需求 | 标准库 (StdPeriph) | HAL库 (STM32Cube HAL) |
| :--- | :--- | :--- |
| **时钟使能** | `RCC_APB2PeriphClockCmd(..., ENABLE)` | `__HAL_RCC_GPIOx_CLK_ENABLE()` |
| **GPIO初始化** | `GPIO_Init(GPIOx, &GPIO_InitStruct)` | `HAL_GPIO_Init(GPIOx, &GPIO_InitStruct)` |
| **读取引脚电平** | `GPIO_ReadInputDataBit(GPIOx, PIN)` | `HAL_GPIO_ReadPin(GPIOx, PIN)` |
| **设置引脚高电平** | `GPIO_SetBits(GPIOx, PIN)` | `HAL_GPIO_WritePin(GPIOx, PIN, GPIO_PIN_SET)` |
| **设置引脚低电平** | `GPIO_ResetBits(GPIOx, PIN)` | `HAL_GPIO_WritePin(GPIOx, PIN, GPIO_PIN_RESET)` |
| **翻转引脚电平** | *(需要自己写异或逻辑 `^= `)* | `HAL_GPIO_TogglePin(GPIOx, PIN)` |
| **位操作写入** | `GPIO_WriteBit(GPIOx, PIN, BitVal)` | `HAL_GPIO_WritePin(GPIOx, PIN, PinState)` |