#  快速复习掌握：SPI总线篇

##  0. 核心理论与防坑指南（面试/实战必背）
* **SPI 的 4 种工作模式 (极度重要)**：由 `CPOL` (时钟极性) 和 `CPHA` (时钟相位) 决定。
  * **Mode 0 (`CPOL=0, CPHA=0`)：最常用！(W25Qxx, 各种屏幕默认模式)**。空闲时 SCK 为低电平；**第一个上升沿采样**，下降沿移位出数据。
  * **Mode 3 (`CPOL=1, CPHA=1`)：第二常用**。空闲时 SCK 为高电平；第二个沿（也是上升沿）采样，下降沿移位。
* **为什么大家都用软件 NSS (片选)，不用硬件 NSS？**
  * STM32 的硬件 NSS 逻辑非常死板且复杂（涉及到多主机竞争机制）。在 99% 的工程中，我们都会把 SPI 配置为 `NSS_Soft`，然后随便找一个普通的 GPIO（比如 PA15）手写拉高拉低来控制片选。
* ** PA15 / PB3 / PB4 避坑铁律**：
  * 这三个引脚在 F103 上默认属于 JTAG 调试端口，**无法直接输出高低电平**。如果一定要用作片选 CS，必须**开启 AFIO 时钟，并禁用 JTAG（保留 SWD）**。

---

##  1. 标准库 (Standard Peripheral Library)

### 1.1 硬件 SPI 写法
```c
#include <stm32f10x.h>

#define SPI_CS_GPIO_PORT    GPIOA
#define SPI_CS_GPIO_PIN     GPIO_Pin_15

#define SPI_CS_LOW()        GPIO_WriteBit(SPI_CS_GPIO_PORT, SPI_CS_GPIO_PIN, Bit_RESET)
#define SPI_CS_HIGH()       GPIO_WriteBit(SPI_CS_GPIO_PORT, SPI_CS_GPIO_PIN, Bit_SET)

/**
 * @brief  SPI1 硬件初始化 (PA5=SCK, PA6=MISO, PA7=MOSI, PA15=CS)
 */
void SPI_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    SPI_InitTypeDef SPI_InitStruct={0};

    // 1. 开启时钟：包含 AFIO 时钟（为了重映射 PA15）
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO | RCC_APB2Periph_GPIOA | RCC_APB2Periph_SPI1, ENABLE);

    //  2. 致命大坑：PA15默认是JTAG引脚，必须重映射关闭JTAG保留SWD，才能当普通GPIO！
    GPIO_PinRemapConfig(GPIO_Remap_SWJ_JTAGDisable, ENABLE);

    // 3. 配置 SPI1 引脚
    // PA5(SCK) 和 PA7(MOSI) 必须是复用推挽输出 (AF_PP)
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_5 | GPIO_Pin_7;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStruct);

    // PA6(MISO) 必须是上拉输入或浮空输入 (IPU)
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_6;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOA, &GPIO_InitStruct);

    // 4. 配置 CS 片选引脚 (PA15) 为普通推挽输出
    GPIO_InitStruct.GPIO_Pin = SPI_CS_GPIO_PIN;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(SPI_CS_GPIO_PORT, &GPIO_InitStruct);
    SPI_CS_HIGH(); // CS 默认拉高（取消选中）

    // 5. 配置 SPI1 参数 (Mode 0: CPOL=0, CPHA=0)
    SPI_InitStruct.SPI_Mode = SPI_Mode_Master;                       // 主机模式
    SPI_InitStruct.SPI_Direction = SPI_Direction_2Lines_FullDuplex;  // 双线全双工
    SPI_InitStruct.SPI_DataSize = SPI_DataSize_8b;                   // 8位数据帧
    SPI_InitStruct.SPI_CPOL = SPI_CPOL_Low;                          // 空闲时SCK为低 (Mode 0)
    SPI_InitStruct.SPI_CPHA = SPI_CPHA_1Edge;                        // 第1个边沿采样 (Mode 0)
    SPI_InitStruct.SPI_FirstBit = SPI_FirstBit_MSB;                  // 高位在前
    SPI_InitStruct.SPI_BaudRatePrescaler = SPI_BaudRatePrescaler_8;  // 72M/8 = 9MHz
    SPI_InitStruct.SPI_NSS = SPI_NSS_Soft;                           // 绝对要用软件NSS！
    SPI_InitStruct.SPI_CRCPolynomial = 7;
    SPI_Init(SPI1, &SPI_InitStruct);

    // 使能 SPI 硬件外设
    SPI_Cmd(SPI1, ENABLE);
}

/**
 * @brief  SPI 底层全双工收发核心函数
 * @note   发送一个字节的同时，必然接收到一个字节
 */
uint8_t SPI_SwapByte(SPI_TypeDef *SPIx, uint8_t TxData)
{
    // 1. 等待发送缓冲区空 (TXE)
    while(SPI_I2S_GetFlagStatus(SPIx, SPI_I2S_FLAG_TXE) == RESET);
    // 2. 发送数据
    SPI_I2S_SendData(SPIx, TxData);
    
    // 3. 等待接收缓冲区非空 (RXNE) -> 代表这8个bit已经完全物理交换完毕！
    while(SPI_I2S_GetFlagStatus(SPIx, SPI_I2S_FLAG_RXNE) == RESET);
    // 4. 返回接收到的数据
    return SPI_I2S_ReceiveData(SPIx);
}

/**
 * @brief  SPI 主机发送与接收多字节
 */
void SPI_MasterTransmitReceive(SPI_TypeDef *SPIx, uint8_t *pDataTx, uint8_t *pDataRx, uint16_t Size)
{
    if (SPIx == NULL || Size == 0) return;
     
    SPI_CS_LOW(); // 拉低片选，开始通信
    
    for(uint16_t i=0; i<Size; i++)
    {
        uint8_t tx = pDataTx ? pDataTx[i] : 0xFF; // 如果没有发送数据，默认发0xFF产生时钟
        uint8_t rx = SPI_SwapByte(SPIx, tx);
        if(pDataRx) pDataRx[i] = rx;
    }
    
    SPI_CS_HIGH(); // 拉高片选，结束通信
}
```

### 1.2 软 SPI 写法
```c
#include <stm32f10x.h>

// ===================== 引脚定义 =====================
#define SPI_SCK_PIN    GPIO_Pin_5
#define SPI_SCK_PORT   GPIOA
#define SPI_MISO_PIN   GPIO_Pin_6
#define SPI_MISO_PORT  GPIOA
#define SPI_MOSI_PIN   GPIO_Pin_7
#define SPI_MOSI_PORT  GPIOA
#define SPI_CS_PIN     GPIO_Pin_15
#define SPI_CS_PORT    GPIOA

// ===================== 引脚操作宏 =====================
#define SCK_H()        GPIO_SetBits(SPI_SCK_PORT, SPI_SCK_PIN)
#define SCK_L()        GPIO_ResetBits(SPI_SCK_PORT, SPI_SCK_PIN)
#define MOSI_H()       GPIO_SetBits(SPI_MOSI_PORT, SPI_MOSI_PIN)
#define MOSI_L()       GPIO_ResetBits(SPI_MOSI_PORT, SPI_MOSI_PIN)
#define MISO_READ()    GPIO_ReadInputDataBit(SPI_MISO_PORT, SPI_MISO_PIN)
#define CS_H()         GPIO_SetBits(SPI_CS_PORT, SPI_CS_PIN)
#define CS_L()         GPIO_ResetBits(SPI_CS_PORT, SPI_CS_PIN)

// ===================== 软件微秒延时 =====================
void SPI_Delay_us(uint32_t us)
{
    uint32_t i;
    for(; us>0; us--)
        for(i=0; i<9; i++); // 72MHz主频下约1us
}

// ===================== 初始化 =====================
void SPI_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    
    // 开启 GPIOA 和 AFIO 时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_AFIO, ENABLE);

    //  致命大坑：PA15 禁用JTAG，保留SWD
    GPIO_PinRemapConfig(GPIO_Remap_SWJ_JTAGDisable, ENABLE);

    // SCK + MOSI + CS 配置为推挽输出
    GPIO_InitStruct.GPIO_Pin = SPI_SCK_PIN | SPI_MOSI_PIN | SPI_CS_PIN;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStruct);

    // MISO 配置为上拉输入
    GPIO_InitStruct.GPIO_Pin = SPI_MISO_PIN;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOA, &GPIO_InitStruct);

    // 初始状态 (匹配 Mode 0)
    CS_H();     // 片选不使能
    SCK_L();    // Mode 0: SCK空闲为低
    MOSI_H();   // MOSI 默认高
}

// ===================== 单字节收发 (完美 Mode 0 时序) =====================
uint8_t SPI_SwapByte(uint8_t tx_byte)
{
    uint8_t rx_byte = 0;
    
    // Mode 0: SCL空闲为低，第一个跳变沿（上升沿）采样数据
    for(uint8_t i=0; i<8; i++)
    {
        // 1. 准备数据 (下降沿/起始状态 移位出数据)
        if(tx_byte & 0x80) MOSI_H();
        else MOSI_L();
        tx_byte <<= 1; // 移位，为下一次做准备
        
        SPI_Delay_us(1); 
        
        // 2. 拉高 SCK 产生上升沿
        SCK_H();
        SPI_Delay_us(1); // 给从机足够的时间准备数据
        
        // 3. 在高电平期间采样读取数据
        rx_byte <<= 1;
        if(MISO_READ()) rx_byte |= 0x01;
        
        // 4. 拉低 SCK 产生下降沿 (完成一个周期)
        SCK_L();
        SPI_Delay_us(1);
    }
    return rx_byte;
}

// ===================== 多字节收发 =====================
void SPI_TransmitReceive(uint8_t *pDataTx, uint8_t *pDataRx, uint16_t Size)
{
    if(Size == 0) return;
    
    CS_L();
    SPI_Delay_us(1);
    
    for(uint16_t i=0; i<Size; i++)
    {
        uint8_t tx = pDataTx ? pDataTx[i] : 0xFF;
        uint8_t rx = SPI_SwapByte(tx);
        if(pDataRx) pDataRx[i] = rx;
    }
    
    SPI_Delay_us(1);
    CS_H();
}
```

---

##  2. HAL库 (Hardware Abstraction Layer)

### 2.1 硬件 SPI 写法
```c
#include "stm32f1xx_hal.h"

#define SPI_CS_GPIO_PORT GPIOA
#define SPI_CS_GPIO_PIN  GPIO_PIN_15

#define SPI_CS_LOW()     HAL_GPIO_WritePin(SPI_CS_GPIO_PORT, SPI_CS_GPIO_PIN, GPIO_PIN_RESET)
#define SPI_CS_HIGH()    HAL_GPIO_WritePin(SPI_CS_GPIO_PORT, SPI_CS_GPIO_PIN, GPIO_PIN_SET)

SPI_HandleTypeDef hspi1;

/**
 * @brief SPI外设参数配置
 */
void SPI_Init(void)
{
    hspi1.Instance = SPI1;
    hspi1.Init.Mode = SPI_MODE_MASTER;
    hspi1.Init.Direction = SPI_DIRECTION_2LINES;
    hspi1.Init.DataSize = SPI_DATASIZE_8BIT;
    hspi1.Init.CLKPolarity = SPI_POLARITY_LOW;          // Mode 0: CPOL=0
    hspi1.Init.CLKPhase = SPI_PHASE_1EDGE;              // Mode 0: CPHA=0
    hspi1.Init.NSS = SPI_NSS_SOFT;                      // 软件 NSS
    hspi1.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_8; 
    hspi1.Init.FirstBit = SPI_FIRSTBIT_MSB;
    hspi1.Init.TIMode = SPI_TIMODE_DISABLE;
    hspi1.Init.CRCCalculation = SPI_CRCCALCULATION_DISABLE;
    hspi1.Init.CRCPolynomial = 10;
    
    // 初始化SPI (内部会自动调用 HAL_SPI_MspInit)
    HAL_SPI_Init(&hspi1); 
}

/**
 * @brief HAL库内部回调，用于初始化引脚和时钟
 */
void HAL_SPI_MspInit(SPI_HandleTypeDef* spiHandle)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    if(spiHandle->Instance == SPI1)
    {
        // 1. 开启时钟
        __HAL_RCC_SPI1_CLK_ENABLE();
        __HAL_RCC_GPIOA_CLK_ENABLE();
        __HAL_RCC_AFIO_CLK_ENABLE(); // 为了重映射PA15

        //  2. 致命大坑：禁用PA15的JTAG功能，保留SWD
        __HAL_AFIO_REMAP_SWJ_NOJTAG();

        // 3. 配置引脚 SCK(PA5), MOSI(PA7) -> 复用推挽
        GPIO_InitStruct.Pin = GPIO_PIN_5 | GPIO_PIN_7;
        GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
        GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
        HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

        // 4. 配置引脚 MISO(PA6) -> 上拉输入
        GPIO_InitStruct.Pin = GPIO_PIN_6;
        GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
        GPIO_InitStruct.Pull = GPIO_PULLUP;
        HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

        // 5. 配置引脚 CS(PA15) -> 软件推挽输出
        GPIO_InitStruct.Pin = SPI_CS_GPIO_PIN;
        GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
        GPIO_InitStruct.Pull = GPIO_NOPULL;
        GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
        HAL_GPIO_Init(SPI_CS_GPIO_PORT, &GPIO_InitStruct);
        
        SPI_CS_HIGH(); // 默认拉高片选
    }
}

/**
 * @brief  SPI 全双工收发
 */
HAL_StatusTypeDef SPI_TransmitReceive(uint8_t *pTxData, uint8_t *pRxData, uint16_t Size)
{
    HAL_StatusTypeDef status;
    SPI_CS_LOW();                    
    // Timeout设置1000ms
    status = HAL_SPI_TransmitReceive(&hspi1, pTxData, pRxData, Size, 1000);
    SPI_CS_HIGH();                   
    return status;
}

/**
 * @brief  SPI 只发送 (常用于刷屏幕)
 */
HAL_StatusTypeDef SPI_Transmit(uint8_t *pData, uint16_t Size)
{
    HAL_StatusTypeDef status;
    SPI_CS_LOW();
    status = HAL_SPI_Transmit(&hspi1, pData, Size, 1000);
    SPI_CS_HIGH();
    return status;
}

/**
 * @brief  SPI 只接收 (常用于读取传感器)
 */
HAL_StatusTypeDef SPI_Receive(uint8_t *pData, uint16_t Size)
{
    HAL_StatusTypeDef status;
    SPI_CS_LOW();
    status = HAL_SPI_Receive(&hspi1, pData, Size, 1000);
    SPI_CS_HIGH();
    return status;
}
```

### 2.2 软 SPI 写法 (HAL 库环境)
```c
#include "stm32f1xx_hal.h"

// ===================== 引脚定义 =====================
#define SPI_SCK_PORT     GPIOA
#define SPI_SCK_PIN      GPIO_PIN_5

#define SPI_MISO_PORT    GPIOA
#define SPI_MISO_PIN     GPIO_PIN_6 

#define SPI_MOSI_PORT    GPIOA
#define SPI_MOSI_PIN     GPIO_PIN_7    

#define SPI_CS_PORT      GPIOA
#define SPI_CS_PIN       GPIO_PIN_15  

// ===================== 引脚电平操作宏 =====================
#define SCK_H()          HAL_GPIO_WritePin(SPI_SCK_PORT, SPI_SCK_PIN, GPIO_PIN_SET)
#define SCK_L()          HAL_GPIO_WritePin(SPI_SCK_PORT, SPI_SCK_PIN, GPIO_PIN_RESET)

#define MOSI_H()         HAL_GPIO_WritePin(SPI_MOSI_PORT, SPI_MOSI_PIN, GPIO_PIN_SET)
#define MOSI_L()         HAL_GPIO_WritePin(SPI_MOSI_PORT, SPI_MOSI_PIN, GPIO_PIN_RESET)

#define MISO_READ()      HAL_GPIO_ReadPin(SPI_MISO_PORT, SPI_MISO_PIN)

#define CS_H()           HAL_GPIO_WritePin(SPI_CS_PORT, SPI_CS_PIN, GPIO_PIN_SET)
#define CS_L()           HAL_GPIO_WritePin(SPI_CS_PORT, SPI_CS_PIN, GPIO_PIN_RESET)

// ===================== 微秒延时 =====================
void SoftSPI_Delay_us(uint32_t us)
{
    uint8_t i;
    while(us--)
    {
        for(i=0; i<72; i++); // 适配72MHz主频
    }
}

// ===================== 软SPI初始化 =====================
void SPI_Init(void)
{
    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_AFIO_CLK_ENABLE();

    //  致命大坑：PA15 禁用 JTAG 功能
    __HAL_AFIO_REMAP_SWJ_NOJTAG();

    GPIO_InitTypeDef GPIO_InitStruct = {0};

    // SCK + MOSI + CS 配置为推挽输出
    GPIO_InitStruct.Pin = SPI_SCK_PIN | SPI_MOSI_PIN | SPI_CS_PIN;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    // MISO 配置为上拉输入
    GPIO_InitStruct.Pin = SPI_MISO_PIN;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    // 初始状态 (Mode 0: SCK空闲为低)
    CS_H();
    SCK_L();
}

// ===================== 单字节收发 (完美 Mode 0 时序) =====================
uint8_t SPI_SendRead_Byte(uint8_t tx_byte)
{
    uint8_t rx_byte = 0;

    for(uint8_t i=0; i<8; i++)
    {
        // 1. 移位出数据
        if(tx_byte & 0x80) MOSI_H();
        else MOSI_L();
        tx_byte <<= 1;

        SoftSPI_Delay_us(1);

        // 2. 产生上升沿
        SCK_H();
        SoftSPI_Delay_us(1);

        // 3. 上升沿期间读取数据
        rx_byte <<= 1;
        if(MISO_READ() == GPIO_PIN_SET) rx_byte |= 0x01;

        // 4. 产生下降沿
        SCK_L();
        SoftSPI_Delay_us(1);
    }
    return rx_byte;
}

// ===================== 多字节收发 =====================
void SPI_TransmitReceive(uint8_t *pTxData, uint8_t *pRxData, uint16_t Size)
{
    if (Size == 0) return;

    CS_L(); // 选中芯片
    SoftSPI_Delay_us(1);

    for(uint16_t i=0; i<Size; i++)
    {
        uint8_t tx = pTxData ? pTxData[i] : 0xFF;
        uint8_t rx = SPI_SendRead_Byte(tx);
        if(pRxData) pRxData[i] = rx;
    }

    SoftSPI_Delay_us(1);
    CS_H(); // 取消选中
}
```