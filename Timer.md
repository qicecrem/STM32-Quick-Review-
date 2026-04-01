这份笔记整合了定时器开发中的**所有核心逻辑、最高频面试题、以及最隐蔽的硬件Bug修复**。

无论是应对期末考试、大厂笔试，还是在企业级工程中直接 Copy 代码，这份**《完美定时器（Timer）终极笔记》**都将是你最强大的武器。

---

# 🚀 快速复习掌握：Timer 定时器（完美终极版）

## 📌 0. 核心理论与防坑绝杀（面试/实战必背）

1. **神级公式（刻在脑子里）：**
   * **定时时间 / PWM周期** $T = \frac{(PSC + 1) \times (ARR + 1)}{T_{clk}}$
   * 假设主频 $T_{clk}$ = 72MHz。如果要配 **1ms** 的中断（1kHz）：
     * 让 `PSC = 71` (分频后为 1MHz，即 1us 计数一次)。
     * 让 `ARR = 999` (计数 1000 次，即 1000us = 1ms)。
2. **🚨 第一发立刻中断Bug（标准库必踩坑）：**
   * 调用 `TIM_TimeBaseInit` 时，底层为了装载预分频器，会手动生成一个更新事件（UEV），这会**导致更新中断标志位（UIF）瞬间被置1**。如果不清除，只要一开中断就会立刻误触发一次！
   * **解决**：初始化后必须手动 `TIM_ClearFlag(TIM3, TIM_FLAG_Update);`。
3. **高级定时器 (TIM1/TIM8) 的致命区别：**
   * 必须配置 `RepetitionCounter = 0`（重复计数器）。
   * 输出 PWM 时，除了普通的 `TIM_Cmd`，**必须额外调用主输出使能 `MOE`**，否则波形出不来！
4. **⚠️ SysTick（滴答定时器）死锁警告：**
   * **千万不要在任何中断服务函数里使用 `HAL_Delay()` 或死等 SysTick！** 滴答定时器默认优先级极低，如果在外部中断里用 `HAL_Delay()`，滴答中断会被屏蔽，导致延时函数变成死循环，单片机直接死机。

---

## ⏳ 1. 基础定时 / 毫秒计时 (TimeBase)

> **场景**：TIM3，1ms 触发一次更新中断，实现全局运行时间统计。

### 1.1 标准库写法
```c
#include "stm32f10x.h"

volatile uint32_t currentTick = 0;  // 必须加 volatile，防止编译器优化

void TIM3_TimeBaseInit(void)
{
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM3, ENABLE);
    
    TIM_TimeBaseInitTypeDef TIM_TimeBaseStruct = {0};
    TIM_TimeBaseStruct.TIM_Prescaler = 71;                   // 1MHz
    TIM_TimeBaseStruct.TIM_Period = 999;                     // 1ms
    TIM_TimeBaseStruct.TIM_CounterMode = TIM_CounterMode_Up; 
    TIM_TimeBaseStruct.TIM_ClockDivision = TIM_CKD_DIV1;     
    TIM_TimeBaseInit(TIM3, &TIM_TimeBaseStruct);
    
    // 🚨 完美防坑：清除初始化产生的更新标志位，防止一开中断就进！
    TIM_ClearFlag(TIM3, TIM_FLAG_Update);
    
    TIM_ITConfig(TIM3, TIM_IT_Update, ENABLE);
    
    NVIC_InitTypeDef NVIC_InitStruct = {0};
    NVIC_InitStruct.NVIC_IRQChannel = TIM3_IRQn;
    NVIC_InitStruct.NVIC_IRQChannelPreemptionPriority = 0; 
    NVIC_InitStruct.NVIC_IRQChannelSubPriority = 1;
    NVIC_InitStruct.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStruct);

    TIM_Cmd(TIM3, ENABLE);
}

void TIM3_IRQHandler(void)
{
    if(TIM_GetITStatus(TIM3, TIM_IT_Update) != RESET)
    {
        currentTick++;        
        TIM_ClearITPendingBit(TIM3, TIM_IT_Update); // 必须清标志位！
    }
}
```

### 1.2 HAL库写法
```c
#include "stm32f1xx_hal.h"

volatile uint32_t currentTick = 0;
TIM_HandleTypeDef htim3;  

void TIM3_TimeBase_Init(void)
{
    htim3.Instance = TIM3;
    htim3.Init.Prescaler = 71;
    htim3.Init.CounterMode = TIM_COUNTERMODE_UP;
    htim3.Init.Period = 999;
    htim3.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
    htim3.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_DISABLE;

    if (HAL_TIM_Base_Init(&htim3) != HAL_OK) Error_Handler();

    // 启动 TIM3 并开启更新中断 (HAL底层已处理好标志位Bug)
    HAL_TIM_Base_Start_IT(&htim3);
}

void HAL_TIM_Base_MspInit(TIM_HandleTypeDef* tim_baseHandle)
{
    if(tim_baseHandle->Instance == TIM3)
    {
        __HAL_RCC_TIM3_CLK_ENABLE();
        HAL_NVIC_SetPriority(TIM3_IRQn, 0, 1);
        HAL_NVIC_EnableIRQ(TIM3_IRQn);
    }
}

void TIM3_IRQHandler(void) { HAL_TIM_IRQHandler(&htim3); }

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if(htim->Instance == TIM3) currentTick++; 
}
```

---

## 🌊 2. PWM 输出比较 (高级定时器带互补)

> **场景**：高级定时器 TIM1，PA8 (主通道) 与 PB13 (互补通道) 输出 1kHz 占空比可调的 PWM。

### 2.1 标准库写法
```c
#include "stm32f10x.h"

void PWM_Init(void)
{
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_GPIOB | RCC_APB2Periph_TIM1, ENABLE);

    GPIO_InitTypeDef GPIO_InitStruct = {0};  
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_AF_PP; // 必须复用推挽
    GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_8; 
    GPIO_Init(GPIOA, &GPIO_InitStruct);
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_13; 
    GPIO_Init(GPIOB, &GPIO_InitStruct);

    // 时基配置 (1kHz)
    TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStruct = {0};
    TIM_TimeBaseInitStruct.TIM_CounterMode = TIM_CounterMode_Up;        
    TIM_TimeBaseInitStruct.TIM_Period = 999;                  
    TIM_TimeBaseInitStruct.TIM_Prescaler = 71;                
    TIM_TimeBaseInitStruct.TIM_ClockDivision = TIM_CKD_DIV1;            
    TIM_TimeBaseInitStruct.TIM_RepetitionCounter = 0;         
    TIM_TimeBaseInit(TIM1, &TIM_TimeBaseInitStruct);

    TIM_ARRPreloadConfig(TIM1, ENABLE); // 使能 ARR 预装载

    // PWM 模式 1
    TIM_OCInitTypeDef TIM_OCInitStruct = {0};
    TIM_OCInitStruct.TIM_OCMode = TIM_OCMode_PWM1;               
    TIM_OCInitStruct.TIM_OCPolarity = TIM_OCPolarity_High;       
    TIM_OCInitStruct.TIM_OCNPolarity = TIM_OCNPolarity_High;     
    TIM_OCInitStruct.TIM_OutputState = TIM_OutputState_Enable;   
    TIM_OCInitStruct.TIM_OutputNState = TIM_OutputNState_Enable; 
    TIM_OCInitStruct.TIM_Pulse = 0;                              
    TIM_OC1Init(TIM1, &TIM_OCInitStruct);

    TIM_OC1PreloadConfig(TIM1, TIM_OCPreload_Enable); // 使能 CCR 预装载

    // 🚨 高级定时器绝杀：必须使能主输出 (MOE)
    TIM_CtrlPWMOutputs(TIM1, ENABLE);
    TIM_Cmd(TIM1, ENABLE);
}

// 动态修改占空比 (0~999)
void PWM_SetCompare1(uint16_t compare) { TIM_SetCompare1(TIM1, compare); }
// 动态修改频率 (配合ARR预装载)
void PWM_SetFreq(uint16_t arr) { TIM_SetAutoreload(TIM1, arr); }
```

### 2.2 HAL库写法
```c
#include "stm32f1xx_hal.h"

TIM_HandleTypeDef htim1;

void PWM_Init(void)
{
    htim1.Instance = TIM1;
    htim1.Init.Prescaler = 71;                        
    htim1.Init.CounterMode = TIM_COUNTERMODE_UP;      
    htim1.Init.Period = 999;                          
    htim1.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
    htim1.Init.RepetitionCounter = 0;                 
    htim1.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_ENABLE; 

    if (HAL_TIM_PWM_Init(&htim1) != HAL_OK) Error_Handler();

    TIM_OC_InitTypeDef sConfigOC = {0};
    sConfigOC.OCMode = TIM_OCMODE_PWM1;               
    sConfigOC.Pulse = 0;                              
    sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;       
    sConfigOC.OCNPolarity = TIM_OCNPOLARITY_HIGH;     
    HAL_TIM_PWM_ConfigChannel(&htim1, &sConfigOC, TIM_CHANNEL_1);

    // 🚨 高级定时器必备：配置死区并使能主输出 (MOE)
    TIM_BreakDeadTimeConfigTypeDef sBreakDeadTimeConfig = {0};
    HAL_TIMEx_ConfigBreakDeadTime(&htim1, &sBreakDeadTimeConfig);

    // 启动主通道 PWM 和 互补通道 PWM
    HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
    HAL_TIMEx_PWMN_Start(&htim1, TIM_CHANNEL_1);
}

// 动态修改占空比和频率神级宏
void PWM_SetCompare1(uint16_t compare) { __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, compare); }
void PWM_SetFreq(uint16_t arr) { __HAL_TIM_SET_AUTORELOAD(&htim1, arr); }
```

---

## 🎯 3. 高级输入捕获 (从模式硬件复位法测脉宽)

> **原理**：双通道抓同一个引脚。CH1上升沿触发硬件复位计数器(清零)，CH2下降沿直接捕获数值，读出来的**绝对是完美的高电平时间**，无需任何减法运算！

### 3.1 标准库写法
```c
#include "stm32f10x.h"

uint16_t HighLevel_Us = 0; // 高电平时间(us)

void TIM1_ICapture_Init(void)
{
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_TIM1 | RCC_APB2Periph_GPIOA, ENABLE);

    // 时基配置 (1MHz)
    TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStruct = {0}; 
    TIM_TimeBaseInitStruct.TIM_CounterMode = TIM_CounterMode_Up;
    TIM_TimeBaseInitStruct.TIM_Period = 65535;                   
    TIM_TimeBaseInitStruct.TIM_Prescaler = 71;                   
    TIM_TimeBaseInit(TIM1, &TIM_TimeBaseInitStruct);

    GPIO_InitTypeDef GPIO_InitStruct = {0};
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_8;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_IN_FLOATING;  
    GPIO_Init(GPIOA, &GPIO_InitStruct);

    TIM_ICInitTypeDef TIM_ICInitStruct = {0};
    // CH1：直连，上升沿
    TIM_ICInitStruct.TIM_Channel = TIM_Channel_1;
    TIM_ICInitStruct.TIM_ICPolarity = TIM_ICPolarity_Rising;
    TIM_ICInitStruct.TIM_ICSelection = TIM_ICSelection_DirectTI; 
    TIM_ICInitStruct.TIM_ICPrescaler = TIM_ICPSC_DIV1;
    TIM_ICInit(TIM1, &TIM_ICInitStruct);

    // CH2：间接连，下降沿
    TIM_ICInitStruct.TIM_Channel = TIM_Channel_2;
    TIM_ICInitStruct.TIM_ICPolarity = TIM_ICPolarity_Falling;
    TIM_ICInitStruct.TIM_ICSelection = TIM_ICSelection_IndirectTI; 
    TIM_ICInit(TIM1, &TIM_ICInitStruct);

    // 🚨 核心魔法：上升沿自动复位计数器
    TIM_SelectInputTrigger(TIM1, TIM_TS_TI1FP1);    
    TIM_SelectSlaveMode(TIM1, TIM_SlaveMode_Reset); 

    TIM_ITConfig(TIM1, TIM_IT_CC2, ENABLE); // 只需下降沿中断即可
    // NVIC配置略...
    TIM_Cmd(TIM1, ENABLE);  
}

void TIM1_CC_IRQHandler(void)
{
    if(TIM_GetITStatus(TIM1, TIM_IT_CC2) != RESET)
    {
        HighLevel_Us = TIM_GetCapture2(TIM1); // 直接读出脉宽！
        TIM_ClearITPendingBit(TIM1, TIM_IT_CC2);
    }
}
```

### 3.2 HAL库写法
```c
#include "stm32f1xx_hal.h"

// ===================== 全局变量 =====================
uint16_t HighLevel_Us = 0;   // 存储读取到的高电平时间 (单位: us)
TIM_HandleTypeDef htim1;     // TIM1 定时器句柄

// ===================== 初始化函数 =====================
/**
  * @brief  TIM1 高级输入捕获初始化 (从模式复位法测量高电平脉宽)
  * @note   PA8 (TIM1_CH1) 接入待测脉冲
  */
void TIM1_ICapture_Init(void)
{
    // 1. TIM1 时基配置 (72MHz主频 -> 1MHz计数频率 -> 1us精度)
    htim1.Instance = TIM1;
    htim1.Init.Prescaler = 71;                        // 预分频 71+1=72 -> 1us计数一次
    htim1.Init.CounterMode = TIM_COUNTERMODE_UP;      // 向上计数
    htim1.Init.Period = 65535;                        // 最大重装载值 (最大能测65.5ms的脉冲)
    htim1.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
    htim1.Init.RepetitionCounter = 0;                 // 高级定时器固定为0
    htim1.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_DISABLE;

    if (HAL_TIM_IC_Init(&htim1) != HAL_OK) 
    {
        Error_Handler();
    }

    // 2. 输入捕获通道配置
    TIM_IC_InitTypeDef sConfigIC = {0};

    // === 配置 CH1：直连引脚，抓取上升沿 ===
    sConfigIC.ICPolarity  = TIM_ICPOLARITY_RISING;       // 上升沿触发
    sConfigIC.ICSelection = TIM_ICSELECTION_DIRECTTI;    // 直接映射到 TI1(PA8)
    sConfigIC.ICPrescaler = TIM_ICPSC_DIV1;              // 捕获不分频
    sConfigIC.ICFilter    = 0;                           // 无滤波
    if (HAL_TIM_IC_ConfigChannel(&htim1, &sConfigIC, TIM_CHANNEL_1) != HAL_OK) 
    {
        Error_Handler();
    }

    // === 配置 CH2：间接连接到CH1引脚，抓取下降沿 ===
    sConfigIC.ICPolarity  = TIM_ICPOLARITY_FALLING;      // 下降沿触发
    sConfigIC.ICSelection = TIM_ICSELECTION_INDIRECTTI;  // 间接映射到 TI1(PA8)
    // Prescaler 和 Filter 保持不变
    if (HAL_TIM_IC_ConfigChannel(&htim1, &sConfigIC, TIM_CHANNEL_2) != HAL_OK) 
    {
        Error_Handler();
    }

    // 3. 🚨 核心魔法：从模式配置 (上升沿自动复位计数器)
    TIM_SlaveConfigTypeDef sSlaveConfig = {0};
    sSlaveConfig.SlaveMode     = TIM_SLAVEMODE_RESET;    // 动作：复位 CNT 计数器
    sSlaveConfig.InputTrigger  = TIM_TS_TI1FP1;          // 触发源：TI1FP1 (CH1的上升沿)
    if (HAL_TIM_SlaveConfigSynchronization(&htim1, &sSlaveConfig) != HAL_OK) 
    {
        Error_Handler();
    }

    // 4. 启动捕获通道
    // 🚨 避坑：CH1 必须启动，否则无法产生复位触发信号！但不需要开中断
    HAL_TIM_IC_Start(&htim1, TIM_CHANNEL_1);
    
    // 开 CH2(下降沿) 的中断，下降沿到来时触发中断读取脉宽
    HAL_TIM_IC_Start_IT(&htim1, TIM_CHANNEL_2);
}

// ===================== MSP 底层初始化 =====================
/**
  * @brief  HAL库自动回调，用于配置时钟、GPIO和NVIC
  */
void HAL_TIM_IC_MspInit(TIM_HandleTypeDef* tim_icHandle)
{
    if(tim_icHandle->Instance == TIM1)
    {
        // 1. 开启时钟
        __HAL_RCC_TIM1_CLK_ENABLE();
        __HAL_RCC_GPIOA_CLK_ENABLE();

        // 2. 配置 PA8 浮空输入
        GPIO_InitTypeDef GPIO_InitStruct = {0};
        GPIO_InitStruct.Pin = GPIO_PIN_8;
        GPIO_InitStruct.Mode = GPIO_MODE_INPUT;          // 输入模式
        GPIO_InitStruct.Pull = GPIO_NOPULL;              // 浮空
        HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

        // 3. 配置 NVIC 中断优先级
        HAL_NVIC_SetPriority(TIM1_CC_IRQn, 1, 1);
        HAL_NVIC_EnableIRQ(TIM1_CC_IRQn);
    }
}

// ===================== 中断服务函数 =====================
/**
  * @brief  TIM1 捕获比较硬件中断入口
  */
void TIM1_CC_IRQHandler(void)
{
    // 交给 HAL 库统筹处理（自动清标志位、判断通道并调用回调）
    HAL_TIM_IRQHandler(&htim1);  
}

// ===================== 业务逻辑回调 =====================
/**
  * @brief  输入捕获完成回调函数 (弱函数重写)
  */
void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim)
{
    // 判断是否是 CH2 (下降沿) 触发了捕获
    if(htim->Channel == HAL_TIM_ACTIVE_CHANNEL_2)
    {
        // 直接读取 CH2 的捕获值，由于上升沿已经把计数器清零了，
        // 此时的计数值就是绝对完美的脉冲宽度 (单位: us)！
        HighLevel_Us = HAL_TIM_ReadCapturedValue(htim, TIM_CHANNEL_2);
        
        /* 如果你想在这个中断里接着测下一个波形，什么都不用管，
           硬件已经自动在准备下一次上升沿复位了！真正的全自动！ */
    }
}

// ===================== 错误处理 =====================
void Error_Handler(void)
{
    // 发生初始化错误时进入死循环
    __disable_irq();
    while(1)
    {
    }
}
```

---

## ⚙️ 4. 编码器模式 (电机驱动/旋钮核心)

> **原理**：无需外部中断，利用定时器硬件的 `Encoder Mode`，自动判断 AB 相电平逻辑，硬件四倍频计数，正转累加，反转递减，极其稳定！

### 4.1 标准库写法
```c
#include "stm32f10x.h"

void Encoder_Init(void)
{
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    GPIO_InitTypeDef GPIO_InitStruct = {0};
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_0 | GPIO_Pin_1;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_IN_FLOATING; 
    GPIO_Init(GPIOA, &GPIO_InitStruct);

    TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStruct = {0};
    TIM_TimeBaseInitStruct.TIM_CounterMode = TIM_CounterMode_Up;
    TIM_TimeBaseInitStruct.TIM_Period = 65535;       
    TIM_TimeBaseInitStruct.TIM_Prescaler = 0;        // 编码器绝对不分频
    TIM_TimeBaseInit(TIM2, &TIM_TimeBaseInitStruct);

    // 🚨 编码器核心：TI1 和 TI2 双边沿计数（4倍频，精度最高）
    TIM_EncoderInterfaceConfig(TIM2, TIM_EncoderMode_TI12, 
                               TIM_ICPolarity_Rising, TIM_ICPolarity_Rising);

    TIM_SetCounter(TIM2, 0); 
    TIM_Cmd(TIM2, ENABLE);   
}

// 每次调用此函数，返回上一次到现在的旋转步数 (正代表正转，负代表反转)
int16_t Encoder_GetCount(void)
{
    // 利用 int16_t 强转，利用溢出特性完美解决 0 - 1 = -1 的越界问题！
    int16_t count = (int16_t)TIM_GetCounter(TIM2); 
    TIM_SetCounter(TIM2, 0); 
    return count;
}
```

### 4.2 HAL库写法
```c
#include "stm32f1xx_hal.h"

TIM_HandleTypeDef htim2;

void Encoder_Init(void)
{
    htim2.Instance = TIM2;
    htim2.Init.Prescaler = 0;
    htim2.Init.CounterMode = TIM_COUNTERMODE_UP;
    htim2.Init.Period = 65535;

    TIM_Encoder_InitTypeDef sConfig = {0};
    sConfig.EncoderMode = TIM_ENCODERMODE_TI12;       
    
    sConfig.IC1Polarity = TIM_ICPOLARITY_RISING;
    sConfig.IC1Selection = TIM_ICSELECTION_DIRECTTI;
    sConfig.IC1Prescaler = TIM_ICPSC_DIV1;
    sConfig.IC1Filter = 10; // 硬件滤波，防毛刺
    
    sConfig.IC2Polarity = TIM_ICPOLARITY_RISING;
    sConfig.IC2Selection = TIM_ICSELECTION_DIRECTTI;
    sConfig.IC2Prescaler = TIM_ICPSC_DIV1;
    sConfig.IC2Filter = 10;

    if (HAL_TIM_Encoder_Init(&htim2, &sConfig) != HAL_OK) Error_Handler();

    // 启动双通道编码器模式
    HAL_TIM_Encoder_Start(&htim2, TIM_CHANNEL_ALL);
}

int16_t Encoder_GetCount(void)
{
    int16_t count = (int16_t)__HAL_TIM_GET_COUNTER(&htim2);
    __HAL_TIM_SET_COUNTER(&htim2, 0);
    return count;
}
```