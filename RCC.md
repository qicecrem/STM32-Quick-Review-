# 快速复习掌握：RCC 系统时钟篇

##  0. 核心基础理论（面试 / 排错必背）
1. **STM32F1 的 5 大时钟源：**
   * **HSE** (高速外部)：通常为 8MHz 晶振（最常用，最精准）。
   * **HSI** (高速内部)：RC 振荡器 8MHz（精度差，没有外部晶振时的备胎）。
   * **LSE** (低速外部)：32.768kHz 晶振（专给 RTC 实时时钟用）。
   * **LSI** (低速内部)：40kHz RC 振荡器（专给独立看门狗 IWDG 用）。
   * **PLL** (锁相环)：用来倍频。通常把 HSE 的 8MHz 放大 9 倍，得到 **72MHz**。
2. **总线最高时钟限制 (死记硬背)：**
   * **SYSCLK (系统时钟)**：最高 **72MHz**。
   * **AHB (高级高性能总线)**：最高 **72MHz**。
   * **APB2 (高速外设总线)**：最高 **72MHz** (挂载 GPIO、USART1、SPI1、ADC 等)。
   * **APB1 (低速外设总线)**：最高 **36MHz** (挂载 TIM2~7、USART2/3、I2C 等)。**因此 APB1 必须 2 分频！**
3. **为什么要有 FLASH_Latency (Flash 等待周期)？**
   * CPU 跑 72MHz 太快了，而内部 Flash 闪存读取速度跟不上。如果不让 CPU 等待，读取代码就会出错直接死机。
   * **规则**：`0~24MHz` (0周期)；`24~48MHz` (1周期)；`48~72MHz` (2周期)。所以跑 72M 必须配 `FLASH_Latency_2`！
4. **隐藏知识**：在使用标准库时，启动文件 `startup_stm32f10x_hd.s` 在跳转到 `main()` 之前，默认会先调用 `SystemInit()`，这个函数内部其实已经把时钟配成 72MHz 了。自己手写是为了深度定制或应对考试。

---

## 💻 1. 标准库写法 (Standard Peripheral Library)

> **优化说明**：


```c
#include <stm32f10x.h>

/**
  * @brief  纯手工配置系统时钟为 72MHz (HSE 8MHz * 9)
  */
void RCC_Configuration(void)
{
    // 步骤1：复位 RCC 寄存器到默认状态
    RCC_DeInit();
    
    // 步骤2：开启外部高速时钟 (HSE)
    RCC_HSEConfig(RCC_HSE_ON);

    // 步骤3：等待 HSE 启动稳定
    if (RCC_WaitForHSEStartUp() == SUCCESS)
    {
        // 步骤4：配置 FLASH 预取指缓存和等待周期 (跑72M必须设为2，且必须在提速前设置！)
        FLASH_PrefetchBufferCmd(FLASH_PrefetchBuffer_Enable);
        FLASH_SetLatency(FLASH_Latency_2);
        
        // 步骤5：配置三大总线分频系数
        RCC_HCLKConfig(RCC_SYSCLK_Div1);   // AHB = SYSCLK = 72MHz
        RCC_PCLK2Config(RCC_HCLK_Div1);    // APB2 = AHB = 72MHz
        RCC_PCLK1Config(RCC_HCLK_Div2);    // APB1 = AHB / 2 = 36MHz (绝对不能超频过36M)
        
        // 步骤6：配置 PLL 锁相环 (来源=HSE不分频, 倍频=9 -> 8*9=72MHz)
        RCC_PLLConfig(RCC_PLLSource_HSE_Div1, RCC_PLLMul_9);
        
        // 步骤7：使能 PLL 并等待其锁定稳定
        RCC_PLLCmd(ENABLE);
        while (RCC_GetFlagStatus(RCC_FLAG_PLLRDY) == RESET); 
        
        // 步骤8：将系统时钟源 (SYSCLK) 切换为 PLL
        RCC_SYSCLKConfig(RCC_SYSCLKSource_PLLCLK);
        
        // 步骤9：检查是否真正切换成功 (0x00=HSI, 0x04=HSE, 0x08=PLL)
        while (RCC_GetSYSCLKSource() != 0x08); 
    }
    else
    {
        // 如果外部晶振坏了或没焊，启用 HSI (内部8MHz RC) 作为紧急备胎
        RCC_HSICmd(ENABLE);
        while (RCC_GetFlagStatus(RCC_FLAG_HSIRDY) == RESET);
        RCC_SYSCLKConfig(RCC_SYSCLKSource_HSICLK);
        
        // 注意：此时系统跑在 8MHz
    }
}
```

---

## 💻 2. HAL库写法 (Hardware Abstraction Layer)



```c
#include "stm32f1xx_hal.h"

// 函数声明
void SystemClock_Config(void);
void Error_Handler(void);

int main(void)
{
    /** 🚨 1. 初始化 HAL 库底层 (必须放在 main 的第一行！) */
    HAL_Init();

    /** 2. 配置系统时钟为 72MHz */
    SystemClock_Config();
    
    while(1)
    {
        // 主循环
    }
}

/**
  * @brief  系统时钟配置 (标准实现：HSE = 8MHz, PLL = ×9, SYSCLK = 72MHz)
  */
void SystemClock_Config(void)
{
    RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

    /** 1. 配置振荡器：HSE + PLL 倍频 */
    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState = RCC_HSE_ON;
    RCC_OscInitStruct.HSEPredivValue = RCC_HSE_PREDIV_DIV1;  // HSE 不分频 (8MHz)
    RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;             // 开启 PLL
    RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;     // PLL 源选择 HSE
    RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL9;             // PLL 9 倍频 → 8×9=72MHz

    // 应用振荡器配置并检查是否成功
    if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
    {
        Error_Handler();
    }

    /** 2. 配置时钟总线分频 + 切换系统时钟 */
    // 选择要配置的时钟类型：SYSCLK / HCLK(AHB) / PCLK1(APB1) / PCLK2(APB2)
    RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK   | RCC_CLOCKTYPE_SYSCLK
                                | RCC_CLOCKTYPE_PCLK1  | RCC_CLOCKTYPE_PCLK2;
                                
    RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;  // 系统时钟 = PLL (72MHz)
    RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;         // AHB 不分频 (72MHz)
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;          // APB1 2分频 (36MHz，F1强制要求！)
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;          // APB2 不分频 (72MHz)

    // 应用时钟配置，并设置 FLASH 延时为 2 个周期 (跑72M必须是 FLASH_LATENCY_2)
    if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2) != HAL_OK)
    {
        Error_Handler();
    }
}

/**
  * @brief  错误处理函数
  */
void Error_Handler(void)
{
    // 发生致命错误（如外部晶振不起振），进入死循环保护
    // 实际工程中，这里可以点亮故障指示灯或触发看门狗复位
    __disable_irq(); // 关闭所有中断
    while(1)
    {
    }
}
```

---

## 📑 3. 备忘查表：时钟总线外设挂载情况 (极度高频考点)

在配置或者复习外设时，你必须知道谁挂在哪个总线上，才能正确开启时钟 `RCC_APB1PeriphClockCmd` 或 `RCC_APB2PeriphClockCmd`。

| 总线名称 | 最高频率 | 主要挂载的外设 (STM32F103系列) |
| :--- | :--- | :--- |
| **AHB** | **72 MHz** | CPU、DMA、SRAM、Flash接口、SDIO、FSMC |
| **APB2** | **72 MHz** | **所有的 GPIO (PA~PG)**、**AFIO (复用功能)**、**EXTI**、**USART1**、SPI1、TIM1、TIM8、ADC1/2/3 |
| **APB1** | **36 MHz** | TIM2/3/4/5/6/7、**USART2/3/4/5**、SPI2/3、**I2C1/2**、CAN、USB、BKP、WWDG |

*(巧记方法：除了**USART1**、**SPI1**、**TIM1/8** 和所有的 **GPIO外**，绝大多数普通外设都挂载在低速的 APB1 总线上。)*