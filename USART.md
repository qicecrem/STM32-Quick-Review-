
#  STM32复习快速掌握笔记：USART串口篇

##  0. 核心基础知识补充（面试与复习必看）
* **引脚配置规范**：
  * **TX (发送)**：必须配置为 `复用推挽输出 (AF_PP)`，因为引脚电平需要交由 USART 外设来控制。
  * **RX (接收)**：必须配置为 `浮空输入 (Floating)` 或 `上拉输入 (IPU)`。
* **发送标志位的区别 (极其重要)**：
  * `TXE` (Transmit Data Register Empty)：发送数据寄存器为空。代表**可以写下一个数据了**，但当前数据可能还在移位寄存器里往外发。
  * `TC` (Transmission Complete)：发送完成。代表数据寄存器和移位寄存器都空了，**数据真正在物理线上发完了**。
* **HAL库中断机制**：
  * HAL 库的接收中断是**单次**的。触发一次中断后，会自动关闭接收中断。因此在回调函数中**必须重新调用 `HAL_UART_Receive_IT`**。

---

## 💻 1. 标准库写法 (Standard Peripheral Library)

### 1.1 轮询（阻塞）模式写法
```c
#include <stm32f10x.h>

void USART_Init(void)
{
    /* 1. 开启时钟：USART1挂载在APB2总线 */
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1 | RCC_APB2Periph_GPIOA, ENABLE);

    /* 2. GPIO配置 (PA9:TX, PA10:RX) */
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_9;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_AF_PP;    // TX必须是复用推挽
    GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStruct);
    
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_10;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_IPU;      // RX上拉或浮空输入
    GPIO_Init(GPIOA, &GPIO_InitStruct);

    /* 3. USART参数配置 */
    USART_InitTypeDef USART_InitStruct = {0};
    USART_InitStruct.USART_BaudRate = 115200;              
    USART_InitStruct.USART_Mode = USART_Mode_Tx | USART_Mode_Rx;
    USART_InitStruct.USART_WordLength = USART_WordLength_8b;
    USART_InitStruct.USART_StopBits = USART_StopBits_1;
    USART_InitStruct.USART_Parity = USART_Parity_No;
    USART_InitStruct.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
    USART_Init(USART1, &USART_InitStruct);

    /* 4. 使能串口 */
    USART_Cmd(USART1, ENABLE);
}

/* 轮询发送指定长度数据 */
void USART_Send(USART_TypeDef *USARTx, uint8_t *pData, uint16_t Size)
{
    for(uint16_t i=0; i<Size; i++){
        // 等待数据寄存器为空 (说明可以放新数据了)
        while(USART_GetFlagStatus(USARTx, USART_FLAG_TXE) == RESET);
        USART_SendData(USARTx, pData[i]);
    }
    // 等待最后一个字节在物理线路上彻底发送完成
    while(USART_GetFlagStatus(USARTx, USART_FLAG_TC) == RESET);
}

/* 轮询接收指定长度数据 (会死等，通常实际工程很少这样写接收) */
void USART_Receive(USART_TypeDef *USARTx, uint8_t *pData, uint16_t len)
{
    if(USARTx==NULL || pData==NULL || len==0) return;
    for(uint16_t i=0; i<len; i++){
        // 等待接收到数据 (RXNE = Read Data Register Not Empty)
        while(USART_GetFlagStatus(USARTx, USART_FLAG_RXNE) == RESET);
        pData[i] = (uint8_t)USART_ReceiveData(USARTx);
    }
}
```

### 1.2 中断接收模式写法（推荐）


```c
#include <stm32f10x.h>

#define RX_BUF_SIZE 100         
uint8_t USART1_RX_BUF[RX_BUF_SIZE]; 
uint16_t USART1_RX_CNT = 0;   

void USART1_Init(void)
{
    /* 0. 中断优先级分组 (整个工程只需调用一次，一般放main最前面) */
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);

    /* 1. 开启时钟 */
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1 | RCC_APB2Periph_GPIOA, ENABLE);

    /* 2. GPIO配置 */
    GPIO_InitTypeDef GPIO_InitStruct={0};
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_9;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStruct);
    
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_10;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOA, &GPIO_InitStruct);

    /* 3. USART配置 */
    USART_InitTypeDef USART_InitStruct={0};
    USART_InitStruct.USART_BaudRate = 115200;
    USART_InitStruct.USART_Mode = USART_Mode_Tx | USART_Mode_Rx;
    USART_InitStruct.USART_WordLength = USART_WordLength_8b;
    USART_InitStruct.USART_StopBits = USART_StopBits_1;
    USART_InitStruct.USART_Parity = USART_Parity_No;
    USART_InitStruct.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
    USART_Init(USART1, &USART_InitStruct);

    /* 4. NVIC中断控制器配置 */
    NVIC_InitTypeDef NVIC_InitStruct = {0};
    NVIC_InitStruct.NVIC_IRQChannel = USART1_IRQn;    
    NVIC_InitStruct.NVIC_IRQChannelPreemptionPriority = 1; 
    NVIC_InitStruct.NVIC_IRQChannelSubPriority = 1;   
    NVIC_InitStruct.NVIC_IRQChannelCmd = ENABLE;      
    NVIC_Init(&NVIC_InitStruct);

    /* 5. 开启USART接收中断 (RXNE) */
    USART_ITConfig(USART1, USART_IT_RXNE, ENABLE);

    /* 6. 使能串口 */
    USART_Cmd(USART1, ENABLE);
}

/* USART1 中断服务函数 (函数名不可更改) */
void USART1_IRQHandler(void)
{
    // 判断是否是接收中断
    if(USART_GetITStatus(USART1, USART_IT_RXNE) != RESET)
    {
        // 读取数据 (注意：读取操作会自动清除 RXNE 标志位，无需手动清除)
        uint8_t rec_data = USART_ReceiveData(USART1);
        
        // 存入缓冲区
        if(USART1_RX_CNT < RX_BUF_SIZE)
        {
            USART1_RX_BUF[USART1_RX_CNT++] = rec_data;
        }
    }
}
```

---

## 💻 2. HAL库写法 (Hardware Abstraction Layer)

### 2.1 轮询模式写法
HAL 库高度封装，轮询收发只需调用 `HAL_UART_Transmit` 和 `HAL_UART_Receive`。

```c
#include "stm32f1xx_hal.h"

UART_HandleTypeDef huart1;

void MX_USART1_UART_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    __HAL_RCC_USART1_CLK_ENABLE();
    __HAL_RCC_GPIOA_CLK_ENABLE();

    /* 配置TX */
    GPIO_InitStruct.Pin = GPIO_PIN_9;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    /* 配置RX */
    GPIO_InitStruct.Pin = GPIO_PIN_10;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    /* 配置UART */
    huart1.Instance = USART1;
    huart1.Init.BaudRate = 115200;
    huart1.Init.WordLength = UART_WORDLENGTH_8B;
    huart1.Init.StopBits = UART_STOPBITS_1;
    huart1.Init.Parity = UART_PARITY_NONE;
    huart1.Init.Mode = UART_MODE_TX_RX;
    huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
    huart1.Init.OverSampling = UART_OVERSAMPLING_16;
    HAL_UART_Init(&huart1);
}

/* 轮询发送 (HAL_MAX_DELAY表示死等，直到发完) */
void USART_Send(USART_TypeDef *USARTx, uint8_t *pData, uint16_t Size)
{
    HAL_UART_Transmit(&huart1, pData, Size, HAL_MAX_DELAY);
}

/* 轮询接收 */
void USART_Receive(USART_TypeDef *USARTx, uint8_t *pData, uint16_t len)
{
    if(USARTx==NULL || pData==NULL || len==0) return;
    HAL_UART_Receive(&huart1, pData, len, HAL_MAX_DELAY);
}
```

### 2.2 中断接收模式写法（推荐）
> **核心考点**：HAL库的中断机制中，`HAL_UART_Receive_IT` 既是开启中断的函数，也是指定接收缓存的函数。**一旦接收完成，中断会自动关闭**。必须在回调函数里重新调用它。

```c
#include "stm32f1xx_hal.h"

#define RX_BUF_SIZE 100         
uint8_t USART1_RX_BUF[RX_BUF_SIZE]; 
uint16_t USART1_RX_CNT = 0;   
UART_HandleTypeDef huart1;

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    USART1_Init();

    /* 【关键步骤 1】: main中第一次开启接收中断，请求接收1个字节 */
    HAL_UART_Receive_IT(&huart1, &USART1_RX_BUF[USART1_RX_CNT], 1);

    while(1) {}
}

void USART1_Init(void)
{
    // ... [同上的 GPIO 与 UART 参数配置代码，省略重复部分] ...
    HAL_UART_Init(&huart1);

    /* 配置 NVIC 中断优先级并使能 */
    HAL_NVIC_SetPriority(USART1_IRQn, 1, 1);
    HAL_NVIC_EnableIRQ(USART1_IRQn);
}

/* 硬件中断入口函数 (定死名字) */
void USART1_IRQHandler(void)
{
    // 交给HAL库的统筹处理函数，它会自动判断标志位，并最终调用 RxCpltCallback
    // 也可以自己写中断处理逻辑
    HAL_UART_IRQHandler(&huart1);
}

/* 【关键步骤 2】: 接收完成回调函数 (弱函数重写) */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if(huart->Instance == USART1)  
    {
        // 1. 处理接收到的数据 (此处仅作索引递增操作)
        if(USART1_RX_CNT < RX_BUF_SIZE - 1)
        {
            USART1_RX_CNT++;
        }

        // 2. 【核心坑点】重新使能接收中断！如果不写这句，只能进一次中断！
        // 让下一个数据存入 buf 的下一个位置
        HAL_UART_Receive_IT(&huart1, &USART1_RX_BUF[USART1_RX_CNT], 1);
    }
}
```

---

##  附加：超常用的 printf 重定向
无论是标准库还是HAL库，调试时最常需要的就是 `printf`。在任何一个 `.c` 文件（通常放在 `main.c` 或 `usart.c`）加入以下代码：

```c
#include <stdio.h>

/* 标准库重定向 */
int fputc(int ch, FILE *f)
{
    while(USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
    USART_SendData(USART1, (uint8_t)ch);
    return ch;
}

/* HAL库重定向 (选其一即可) */
int fputc(int ch, FILE *f)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
```
*注意：使用 `printf` 必须在工程选项（Keil中为魔术棒 -> Target）勾选 **Use MicroLIB**。*

---


这份笔记为你提炼了 GCC 环境下（STM32CubeIDE、VSCode 等）`printf` 重定向的核心干货，方便随时查阅：

---

# GCC (Newlib) 环境下 `printf` 重定向

### 1. 核心原理
与 Keil (ARMCC) 重写 `fputc` 不同，GCC (Newlib) 最终调用的是系统调用 **`_write`**。我们通过重写该函数将数据流向 UART。

### 2. 代码实现 (以 HAL 库为例)
在 `main.c` 或 `usart.c` 中添加以下代码（注意包含 `stdio.h`）：

```c
#include <stdio.h>

extern UART_HandleTypeDef huart1; // 声明你的串口句柄

// 重写 _write 函数
int _write(int file, char *ptr, int len) {
    // 直接利用 HAL 库的超时发送函数，效率比单字节发送高
    HAL_UART_Transmit(&huart1, (uint8_t *)ptr, len, HAL_MAX_DELAY);
    return len;
}

/* 如果是 STM32CubeIDE，也可以只重写 __io_putchar (底层被 syscalls.c 调用) */
int __io_putchar(int ch) {
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
```

---

### 3. 避坑指南（必看）

| 常见问题 | 现象 | 解决方法 |
| :--- | :--- | :--- |
| **无法打印浮点数** | `%f` 打印出空白或报错 | **增加链接参数**：`-u _printf_float` <br>*(IDE 中在 MCU Settings 勾选 "Use float with printf")* |
| **不加 `\n` 不输出** | 串口助手死活没反应 | **原因**：行缓冲机制。<br>**方法 A**：`printf` 末尾必加 `\n`。<br>**方法 B**：在 `main` 初始化时关掉缓冲：<br>`setvbuf(stdout, NULL, _IONBF, 0);` |

配置浮点数支持示例:
在你的 CMakeLists.txt 中，找到 add_executable(...) 之后的位置，添加以下链接选项：
```CMake
# 启用 printf 打印浮点数支持
target_link_options(${PROJECT_NAME} PRIVATE "-u _printf_float")

# (可选) 如果你还需要 scanf 支持输入浮点数，请取消下面这行的注释
# target_link_options(${PROJECT_NAME} PRIVATE "-u _scanf_float")
```

---



---

## 📑 3. 备忘查表：标准库 vs HAL库 串口常用函数对比

| 功能需求 | 标准库 (StdPeriph) | HAL库 (STM32Cube HAL) |
| :--- | :--- | :--- |
| **串口初始化** | `USART_Init(USARTx, &InitStruct)` | `HAL_UART_Init(&huart)` |
| **发送一个字节** | `USART_SendData(USARTx, Data)` | `HAL_UART_Transmit(&huart, &pData, size, timeout)` |
| **接收一个字节** | `USART_ReceiveData(USARTx)` | `HAL_UART_Receive(&huart, &pData, size, timeout)` |
| **读取状态标志** | `USART_GetFlagStatus(USARTx, FLAG)` | *(HAL封装在收发函数内，外界少用)* |
| **开启接收中断** | `USART_ITConfig(USARTx, IT_RXNE, ENABLE)`| `HAL_UART_Receive_IT(&huart, pData, Size)` *(兼具接收与开中断)*|
| **获取中断状态** | `USART_GetITStatus(USARTx, IT_RXNE)` | 由 `HAL_UART_IRQHandler` 自动完成 |
| **中断回调入口** | 在 `USARTx_IRQHandler` 里手写判断逻辑 | 重写 `HAL_UART_RxCpltCallback` |