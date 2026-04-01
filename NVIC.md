#  快速复习掌握：中断（EXTI与NVIC）篇

##  0. 核心理论与防坑指南（面试/实战必背）

1. **什么是 NVIC (嵌套向量中断控制器)？**
   * **抢占优先级 (Preemption Priority)**：决定能不能“打断”别人。抢占优先级高的中断，可以打断正在执行的低抢占优先级中断（中断嵌套）。
   * **响应优先级 (Sub Priority / 子优先级)**：当抢占优先级相同时，谁也不能打断谁。如果两个中断同时发生，响应优先级高的先执行。
   * ** 避坑铁律**：`NVIC_PriorityGroupConfig`（中断分组配置）**在一个工程中只能调用一次**！通常放在 `main` 函数的最开头。千万不要在各个模块里重复分组，否则会导致中断逻辑彻底混乱！
2. **外部中断线 (EXTI Line) 的映射规则 (面试必考)：**
   * STM32 有 16 根外部中断线 (`EXTI0` ~ `EXTI15`) 对应 GPIO 的 16 个引脚。
   * **PA0、PB0、PC0... 统统共享 `EXTI0`！** 
   * **结论：你不能同时把 PA0 和 PB0 都配置为外部中断！** 同一时间，一个中断线只能映射到一个端口。
3. **中断服务函数名 (IRQ Handler) 的合并规则：**
   * `EXTI0` 到 `EXTI4`：每个人都有独立的中断服务函数（如 `EXTI0_IRQHandler`）。
   * `EXTI5` 到 `EXTI9`：**共享一个**中断服务函数 `EXTI9_5_IRQHandler`。
   * `EXTI10` 到 `EXTI15`：**共享一个**中断服务函数 `EXTI15_10_IRQHandler`。
4. ** 标准库最容易忘的两个命门：**
   * **开启 AFIO 时钟**：只要用到外部中断，必须开启 AFIO 时钟，否则引脚映射无效！
   * **清除中断标志位**：在中断服务函数末尾，**必须**调用清除标志位函数。如果忘了写，单片机会无限卡死在中断里出不来！

---

##  1. 标准库 (Standard Peripheral Library)

> **场景假设**：
> PA0 接按键（按下接地），配置为**下降沿**触发外部中断 (`EXTI0`)。
> PC13 接按键（按下接地），配置为**下降沿**触发外部中断 (`EXTI15_10`)。

### 1.1 中断初始化
```c
#include <stm32f10x.h>

/**
 * @brief  外部中断初始化 (PA0 和 PC13)
 */
void EXTI_Key_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    EXTI_InitTypeDef EXTI_InitStruct = {0};
    NVIC_InitTypeDef NVIC_InitStruct = {0};

    // 1. 开启时钟：GPIOA、GPIOC 以及 极其重要的 AFIO 时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_GPIOC | RCC_APB2Periph_AFIO, ENABLE);

    // 2. 配置 GPIO 模式为上拉输入 (按键未按下时为高电平)
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_0;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOA, &GPIO_InitStruct);

    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_13;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOC, &GPIO_InitStruct);

    // 3. 配置 AFIO 引脚映射 (告诉单片机 EXTI 线连接到哪个 GPIO)
    GPIO_EXTILineConfig(GPIO_PortSourceGPIOA, GPIO_PinSource0);  // EXTI0 连 PA0
    GPIO_EXTILineConfig(GPIO_PortSourceGPIOC, GPIO_PinSource13); // EXTI13 连 PC13

    // 4. 配置 EXTI 外部中断线
    // 配置 EXTI0 (对应 PA0)
    EXTI_InitStruct.EXTI_Line = EXTI_Line0;
    EXTI_InitStruct.EXTI_Mode = EXTI_Mode_Interrupt;    // 中断模式 (非事件模式)
    EXTI_InitStruct.EXTI_Trigger = EXTI_Trigger_Falling;// 下降沿触发
    EXTI_InitStruct.EXTI_LineCmd = ENABLE;
    EXTI_Init(&EXTI_InitStruct);

    // 配置 EXTI13 (对应 PC13)
    EXTI_InitStruct.EXTI_Line = EXTI_Line13;
    EXTI_InitStruct.EXTI_Mode = EXTI_Mode_Interrupt;
    EXTI_InitStruct.EXTI_Trigger = EXTI_Trigger_Falling;
    EXTI_InitStruct.EXTI_LineCmd = ENABLE;
    EXTI_Init(&EXTI_InitStruct);

    // 5. 配置 NVIC 中断优先级控制器
    // 注意：NVIC_PriorityGroupConfig 应该在 main 函数最开头调一次即可，这里写出来是为了完整性
    // NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2); 

    // 使能 EXTI0 中断通道
    NVIC_InitStruct.NVIC_IRQChannel = EXTI0_IRQn;
    NVIC_InitStruct.NVIC_IRQChannelPreemptionPriority = 1; // 抢占优先级 1
    NVIC_InitStruct.NVIC_IRQChannelSubPriority = 1;        // 响应优先级 1
    NVIC_InitStruct.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStruct);

    // 使能 EXTI15_10 中断通道 (EXTI10~15共用这个通道)
    NVIC_InitStruct.NVIC_IRQChannel = EXTI15_10_IRQn;
    NVIC_InitStruct.NVIC_IRQChannelPreemptionPriority = 2; // 抢占优先级 2 (比PA0低)
    NVIC_InitStruct.NVIC_IRQChannelSubPriority = 1;
    NVIC_InitStruct.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStruct);
}
```

### 1.2 中断服务函数 (通常放在同一个.c文件或 `stm32f10x_it.c` 中)
```c
/**
 * @brief  PA0 的中断服务函数 (拥有独立函数名)
 * @note   函数名在 startup_stm32f10x_xx.s 中已定死，绝不能拼错！
 */
void EXTI0_IRQHandler(void)
{
    // 1. 判断是不是 EXTI0 触发的中断
    if(EXTI_GetITStatus(EXTI_Line0) != RESET)
    {
        // ... 在这里写 PA0 触发后的业务逻辑 ...
        
        // 🚨 2. 致命操作：清除中断标志位！否则无限进中断！
        EXTI_ClearITPendingBit(EXTI_Line0);
    }
}

/**
 * @brief  PC13 的中断服务函数 (线10到15共享此函数)
 */
void EXTI15_10_IRQHandler(void)
{
    // 1. 既然是共享的，就必须判断到底是哪根线触发的 (判断是否是Line13)
    if(EXTI_GetITStatus(EXTI_Line13) != RESET)
    {
        // ... 在这里写 PC13 触发后的业务逻辑 ...
        
        // 🚨 2. 清除中断标志位！
        EXTI_ClearITPendingBit(EXTI_Line13);
    }
    
    // 如果你还配置了Line14，就继续写：
    // if(EXTI_GetITStatus(EXTI_Line14) != RESET) { ... EXTI_ClearITPendingBit(EXTI_Line14); }
}
```

---

## 💻 2. HAL库 (Hardware Abstraction Layer)

> **HAL库的爽点所在**：
> 1. HAL 库把 AFIO 时钟开启、引脚映射、触发边沿配置，**全部封装进了 `HAL_GPIO_Init` 中**！代码量骤减。
> 2. HAL 库自带统筹处理函数，**自动帮你清除中断标志位**，再统一回调 `HAL_GPIO_EXTI_Callback`。

### 2.1 中断初始化
```c
#include "stm32f1xx_hal.h"

/**
 * @brief  HAL库 外部中断初始化 (PA0 和 PC13)
 */
void MX_EXTI_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    // 1. 开启 GPIO 时钟 (AFIO 时钟 HAL 库会自动处理)
    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOC_CLK_ENABLE();

    // 2. 配置 PA0 为下降沿触发中断，并开启内部上拉
    GPIO_InitStruct.Pin = GPIO_PIN_0;
    // HAL库神级宏：IT代表Interrupt，FALLING代表下降沿
    GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING; 
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    // 3. 配置 PC13 为下降沿触发中断，并开启内部上拉
    GPIO_InitStruct.Pin = GPIO_PIN_13;
    GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

    // 4. 配置 NVIC 中断优先级并使能
    // 配置 EXTI0 (PA0)
    HAL_NVIC_SetPriority(EXTI0_IRQn, 1, 1);
    HAL_NVIC_EnableIRQ(EXTI0_IRQn);

    // 配置 EXTI15_10 (PC13)
    HAL_NVIC_SetPriority(EXTI15_10_IRQn, 2, 1);
    HAL_NVIC_EnableIRQ(EXTI15_10_IRQn);
}
```

### 2.2 中断响应机制 (HAL库核心套路)
在 HAL 库中，硬件中断触发后，依然会进入 `xxx_IRQHandler`。但是我们**不需要在里面写业务逻辑，也不需要手动清标志位**。我们将控制权交给 HAL 的统筹函数：

```c
// 硬件中断入口函数 (通常存放在 stm32f1xx_it.c 中，也可以放在 main.c)

void EXTI0_IRQHandler(void)
{
    // 交给 HAL 库的统筹函数处理
    // 它内部会自动检查标志位，并自动清除标志位，然后调用回调函数
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_0);
}

void EXTI15_10_IRQHandler(void)
{
    // 交给 HAL 库的统筹函数处理
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_13);
    
    // 如果同一组还有其他的引脚，比如 PIN_14，也一并放进来即可
    // HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_14);
}
```

### 2.3 中断回调函数 (我们写业务逻辑的地方)
HAL 库处理完标志位后，会统一跳转到一个**弱函数 (Weak Function)**：`HAL_GPIO_EXTI_Callback`。我们只需要在任意的 `.c` 文件中重写这个函数即可。

```c
/**
 * @brief  外部中断统一回调函数 (所有的外部中断最终都会来到这里)
 * @param  GPIO_Pin: 是哪个引脚触发了中断
 */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    // 通过判断传入的 Pin 号，来区分是谁触发的
    if(GPIO_Pin == GPIO_PIN_0)
    {
        // ... 执行 PA0 按下后的业务逻辑 ...
        
        // 建议增加简单的软件消抖
        // HAL_Delay(20);
        // if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET) { ... }
    }
    else if(GPIO_Pin == GPIO_PIN_13)
    {
        // ... 执行 PC13 按下后的业务逻辑 ...
    }
}
```

---

## 📑 3. 备忘查表：标准库 vs HAL库 差异对比

| 操作项 | 标准库 (StdPeriph) | HAL库 (STM32Cube HAL) |
| :--- | :--- | :--- |
| **开启 AFIO 时钟** | **必须手动开启** `RCC_APB2Periph_AFIO` | HAL 库自动管理，无需关心 |
| **配置触发模式** | 需配置 `EXTI_InitTypeDef` (上升沿/下降沿) | 封装在 `GPIO_Mode` 中 (如 `GPIO_MODE_IT_FALLING`) |
| **引脚映射** | `GPIO_EXTILineConfig(Port, Pin)` | `HAL_GPIO_Init` 内部自动映射 |
| **中断服务函数位置** | 在 `EXTI_IRQHandler` 中手写逻辑 | 只在 `IRQHandler` 中调用 `HAL_GPIO_EXTI_IRQHandler` |
| **清除标志位** | **必须手动清除** `EXTI_ClearITPendingBit` | HAL库自动清除 |
| **写业务逻辑的位置**| 在硬件入口 `EXTI_IRQHandler` 中 | 在重写的 `HAL_GPIO_EXTI_Callback` 中 |