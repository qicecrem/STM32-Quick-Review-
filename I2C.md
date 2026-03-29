
#  快速复习掌握：I2C总线篇

##  0. 核心基础理论（面试/复习必背）
*   **为什么 I2C 引脚必须配置为“开漏输出 (Open-Drain)”？**
    *   因为 I2C 允许多个设备挂在同一条总线上。开漏输出配合外部上拉电阻，能实现**“线与 (Wire-AND)”逻辑**。任何一个设备拉低总线，总线就是低电平；所有设备释放总线，总线才会被上拉电阻拉高。这避免了推挽输出时可能导致的电源短路。
*   **7位地址与8位地址的换算（无敌踩坑点）：**
    *   如果是 **7位地址** (如 `0x3C`)：发地址前必须左移1位 (`0x3C << 1` = `0x78`)，然后最低位加上读写控制位（写为0，读为1）。
    *   如果是 **8位地址** (如 `0x78`)：直接使用 `Addr & 0xFE` (写) 或 `Addr | 0x01` (读)。
*   **为什么工程中多用“软件 I2C (模拟 I2C)”？**
    *   STM32F1 系列的硬件 I2C 存在著名的“硅缺陷(Silicon Bug)”，容易在复杂电磁环境下锁死总线。软件 I2C 移植性极强，可以用任意 GPIO 模拟，且极其稳定。

---

## 💻 1. 标准库 (Standard Peripheral Library)

### 1.1 硬件 I2C 写法


```c
#include <stm32f10x.h>

/**
 * @brief  I2C初始化 (I2C1, PB6=SCL, PB7=SDA, 400kHz)
 * @note   GPIO=复用开漏 | 必须软件复位 | 400kHz配2:1占空比
 */
void I2C_Init(void)
{
    // 1. GPIOB时钟 + 开漏复用配置
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    GPIO_InitStruct.GPIO_Pin   = GPIO_Pin_6 | GPIO_Pin_7;
    GPIO_InitStruct.GPIO_Mode  = GPIO_Mode_AF_OD;  // I2C硬件引脚必须是复用开漏
    GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOB, &GPIO_InitStruct);

    // 2. I2C时钟 + 软件复位 (防死锁第一招)
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_I2C1, ENABLE);
    RCC_APB1PeriphResetCmd(RCC_APB1Periph_I2C1, ENABLE);
    RCC_APB1PeriphResetCmd(RCC_APB1Periph_I2C1, DISABLE);

    // 3. I2C参数配置
    I2C_InitTypeDef I2C_InitStruct = {0};
    I2C_InitStruct.I2C_ClockSpeed          = 400000;              // 400k高速模式
    I2C_InitStruct.I2C_Mode                = I2C_Mode_I2C;
    I2C_InitStruct.I2C_DutyCycle           = I2C_DutyCycle_2;     // 占空比(高速模式下使用)
    I2C_InitStruct.I2C_OwnAddress1         = 0x00;                // 自身地址(作主机时随便填)
    I2C_InitStruct.I2C_Ack                 = I2C_Ack_Enable;      // 默认开启应答
    I2C_InitStruct.I2C_AcknowledgedAddress = I2C_AcknowledgedAddress_7bit;
    I2C_Init(I2C1, &I2C_InitStruct);

    // 4. 使能I2C外设
    I2C_Cmd(I2C1, ENABLE);
}

/**
 * @brief  I2C 标准写操作（带寄存器地址）
 * @param  SlaveAddr: 8位从机地址 (已左移过)
 * @note   流程：起始→写地址→寄存器→数据→停止
 */
void I2C_Write(I2C_TypeDef *I2Cx, uint8_t SlaveAddr, uint8_t RegAddr, uint8_t *pData, uint16_t Size)
{
    // 等待总线空闲
    while(I2C_GetFlagStatus(I2Cx, I2C_FLAG_BUSY) == SET);

    // ====== 1. 起始信号 ======
    I2C_GenerateSTART(I2Cx, ENABLE);
    while(I2C_GetFlagStatus(I2Cx, I2C_FLAG_SB) == RESET); // SB: 起始位发送完成

    // ====== 2. 发送：从机地址 + 写位(0) ======
    I2C_SendData(I2Cx, SlaveAddr & 0xFE); 
    while(I2C_GetFlagStatus(I2Cx, I2C_FLAG_ADDR) == RESET); // ADDR: 地址发送成功且收到ACK
    // 清除ADDR标志位：必须按顺序读取SR1和SR2
    (void)I2C_ReadRegister(I2Cx, I2C_Register_SR1);
    (void)I2C_ReadRegister(I2Cx, I2C_Register_SR2);

    // ====== 3. 发送：寄存器地址 ======
    I2C_SendData(I2Cx, RegAddr);
    while(I2C_GetFlagStatus(I2Cx, I2C_FLAG_TXE) == RESET);  // TXE: 数据寄存器空

    // ====== 4. 发送数据 ======
    for(uint16_t i=0; i<Size; i++){
        while(I2C_GetFlagStatus(I2Cx, I2C_FLAG_TXE) == RESET);
        I2C_SendData(I2Cx, pData[i]);
    }
    // BTF: 字节传输完成 (移位寄存器和数据寄存器都空了，真正发完)
    while(I2C_GetFlagStatus(I2Cx, I2C_FLAG_BTF) == RESET);

    // ====== 5. 停止信号 ======
    I2C_GenerateSTOP(I2Cx, ENABLE);
}

/**
 * @brief  I2C 标准指定地址读操作
 * @note   【重点纠错】：读取1个字节时，必须在清 ADDR 标志位 前 关闭 ACK！
 */
int I2C_Read(I2C_TypeDef *I2Cx, uint8_t SlaveAddr, uint8_t RegAddr, uint8_t *pBuf, uint16_t Size)
{
    //===================== 第一步：起始 + 写地址 + 写寄存器 =====================
    while(I2C_GetFlagStatus(I2Cx, I2C_FLAG_BUSY) == SET);
    
    I2C_GenerateSTART(I2Cx, ENABLE);
    while(I2C_GetFlagStatus(I2Cx, I2C_FLAG_SB) == RESET);

    I2C_SendData(I2Cx, SlaveAddr & 0xFE);           
    while(I2C_GetFlagStatus(I2Cx, I2C_FLAG_ADDR) == RESET);
    (void)I2C_ReadRegister(I2Cx, I2C_Register_SR1);
    (void)I2C_ReadRegister(I2Cx, I2C_Register_SR2);

    I2C_SendData(I2Cx, RegAddr);                    
    while(I2C_GetFlagStatus(I2Cx, I2C_FLAG_TXE) == RESET);

    //===================== 第二步：重复起始 =====================
    I2C_GenerateSTART(I2Cx, ENABLE);                
    while(I2C_GetFlagStatus(I2Cx, I2C_FLAG_SB) == RESET);

    //===================== 第三步：读地址 + 接收数据 =====================
    I2C_SendData(I2Cx, SlaveAddr | 0x01);           
    while(I2C_GetFlagStatus(I2Cx, I2C_FLAG_ADDR) == RESET);

    // 【核心避坑】单字节读取与多字节读取的清除 ADDR 顺序完全不同！
    if(Size == 1)
    {
        // 1. 提前告诉硬件：下一个字节收完回NACK (必须在清ADDR前配置！)
        I2C_AcknowledgeConfig(I2Cx, DISABLE);       
        
        // 2. 清除ADDR标志位
        (void)I2C_ReadRegister(I2Cx, I2C_Register_SR1);
        (void)I2C_ReadRegister(I2Cx, I2C_Register_SR2);

        // 3. 生成停止信号
        I2C_GenerateSTOP(I2Cx, ENABLE);             
        
        // 4. 等待数据接收并读取
        while(I2C_GetFlagStatus(I2Cx, I2C_FLAG_RXNE) == RESET);
        pBuf[0] = I2C_ReceiveData(I2Cx);
    }
    else
    {
        // 多字节读取：先清ADDR，正常收数据
        (void)I2C_ReadRegister(I2Cx, I2C_Register_SR1);
        (void)I2C_ReadRegister(I2Cx, I2C_Register_SR2);
        I2C_AcknowledgeConfig(I2Cx, ENABLE);
        
        for(uint16_t i=0; i<Size-1; i++){
            while(I2C_GetFlagStatus(I2Cx, I2C_FLAG_RXNE) == RESET);
            pBuf[i] = I2C_ReceiveData(I2Cx);
        }
        
        // 最后一个字节特殊处理
        I2C_AcknowledgeConfig(I2Cx, DISABLE);       
        I2C_GenerateSTOP(I2Cx, ENABLE);             
        while(I2C_GetFlagStatus(I2Cx, I2C_FLAG_RXNE) == RESET);
        pBuf[Size-1] = I2C_ReceiveData(I2Cx);
    }

    // 恢复默认的ACK使能状态，防止影响下一次通讯
    I2C_AcknowledgeConfig(I2Cx, ENABLE);
    return 0;
}
```

### 1.2 软 I2C 写法 (极度推荐工程应用)
```c
#include <stm32f10x.h>

// ===================== 1. 引脚定义 =====================
#define I2C_SCL_PIN    GPIO_Pin_6
#define I2C_SDA_PIN    GPIO_Pin_7
#define I2C_GPIO_PORT  GPIOB
#define I2C_GPIO_CLK   RCC_APB2Periph_GPIOB

// ===================== 2. 引脚操作宏 =====================
#define I2C_SCL_H      GPIO_SetBits(I2C_GPIO_PORT, I2C_SCL_PIN)   // SCL拉高
#define I2C_SCL_L      GPIO_ResetBits(I2C_GPIO_PORT, I2C_SCL_PIN) // SCL拉低
#define I2C_SDA_H      GPIO_SetBits(I2C_GPIO_PORT, I2C_SDA_PIN)   // SDA拉高
#define I2C_SDA_L      GPIO_ResetBits(I2C_GPIO_PORT, I2C_SDA_PIN) // SDA拉低
// OD模式下直接读取输入寄存器即为引脚真实电平
#define I2C_READ_SDA   GPIO_ReadInputDataBit(I2C_GPIO_PORT, I2C_SDA_PIN) 

// ===================== 3. 微秒延时 =====================
// 400kHz用2us，100kHz用5us
void I2C_Delay_us(uint32_t us)
{
    while(us--)
    {
        uint32_t delay = 10;
        while(delay--);
    }
}

// ===================== 4. GPIO初始化 =====================
void Soft_I2C_Init(void)
{
    RCC_APB2PeriphClockCmd(I2C_GPIO_CLK, ENABLE);
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    GPIO_InitStruct.GPIO_Pin   = I2C_SCL_PIN | I2C_SDA_PIN;
    GPIO_InitStruct.GPIO_Mode  = GPIO_Mode_Out_OD;  // 软I2C核心：普通开漏输出
    GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(I2C_GPIO_PORT, &GPIO_InitStruct);

    // 初始状态：SCL/SDA 都拉高（总线空闲）
    I2C_SCL_H;
    I2C_SDA_H;
}

// ===================== 5. 核心时序函数 =====================
// 1. 起始信号：SCL高时，SDA下降沿
void Soft_I2C_Start(void)
{
    I2C_SDA_H;
    I2C_SCL_H;
    I2C_Delay_us(2);
    I2C_SDA_L;  
    I2C_Delay_us(2);
    I2C_SCL_L;  // 拉低SCL，钳住总线准备传输
}

// 2. 停止信号：SCL高时，SDA上升沿
void Soft_I2C_Stop(void)
{
    I2C_SDA_L;
    I2C_SCL_H;
    I2C_Delay_us(2);
    I2C_SDA_H;  
    I2C_Delay_us(2);
}

// 3. 主机发送 ACK
void Soft_I2C_Send_ACK(void)
{
    I2C_SDA_L;
    I2C_SCL_H;
    I2C_Delay_us(2);
    I2C_SCL_L;
    I2C_SDA_H;  // 释放SDA
}

// 4. 主机发送 NACK
void Soft_I2C_Send_NACK(void)
{
    I2C_SDA_H;
    I2C_SCL_H;
    I2C_Delay_us(2);
    I2C_SCL_L;
}

// 5. 等待从机应答（返回1=无应答失败，0=有应答成功）
uint8_t Soft_I2C_Wait_ACK(void)
{
    uint8_t retry = 0;
    I2C_SDA_H;  // 主机释放SDA，把控制权交给从机
    I2C_SCL_H;
    I2C_Delay_us(2);

    while(I2C_READ_SDA)  // 检测SDA是否被从机拉低
    {
        retry++;
        if(retry > 200)
        {
            Soft_I2C_Stop();
            return 1;
        }
    }
    I2C_SCL_L;
    return 0;
}

// ===================== 6. 字节读写 =====================
// 发送1字节 (高位在前 MSB)
void Soft_I2C_Send_Byte(uint8_t byte)
{
    for(uint8_t i=0; i<8; i++)
    {
        I2C_SCL_L; // SCL拉低期间才允许改变SDA电平
        if(byte & 0x80) I2C_SDA_H;
        else I2C_SDA_L;
        byte <<= 1;
        I2C_Delay_us(2);
        I2C_SCL_H; // SCL拉高期间数据保持稳定供从机读取
        I2C_Delay_us(2);
    }
    I2C_SCL_L;
}

// 读取1字节
uint8_t Soft_I2C_Read_Byte(void)
{
    uint8_t byte = 0;
    I2C_SDA_H;  // 读取前务必释放SDA
    for(uint8_t i=0; i<8; i++)
    {
        I2C_SCL_L;
        I2C_Delay_us(2);
        I2C_SCL_H;  // SCL拉高期间读取SDA电平
        byte <<= 1;
        if(I2C_READ_SDA) byte++;
        I2C_Delay_us(2);
    }
    I2C_SCL_L;
    return byte;
}

// ===================== 7. 完整读写 (假设SlaveAddr是8位地址) =====================
void Soft_I2C_Write(uint8_t SlaveAddr, uint8_t RegAddr, uint8_t *pData, uint16_t Size)
{
    Soft_I2C_Start();
    Soft_I2C_Send_Byte(SlaveAddr & 0xFE); 
    // 实际工程建议判断返回值：if(Soft_I2C_Wait_ACK()) return ERROR;
    Soft_I2C_Wait_ACK();
    
    Soft_I2C_Send_Byte(RegAddr);          
    Soft_I2C_Wait_ACK();

    for(uint16_t i=0; i<Size; i++)
    {
        Soft_I2C_Send_Byte(pData[i]);
        Soft_I2C_Wait_ACK();
    }
    Soft_I2C_Stop();
}

void Soft_I2C_Read(uint8_t SlaveAddr, uint8_t RegAddr, uint8_t *pBuf, uint16_t Size)
{
    Soft_I2C_Start();
    Soft_I2C_Send_Byte(SlaveAddr & 0xFE);
    Soft_I2C_Wait_ACK();
    
    Soft_I2C_Send_Byte(RegAddr);
    Soft_I2C_Wait_ACK();

    Soft_I2C_Start();
    Soft_I2C_Send_Byte(SlaveAddr | 0x01); 
    Soft_I2C_Wait_ACK();

    for(uint16_t i=0; i<Size; i++)
    {
        pBuf[i] = Soft_I2C_Read_Byte();
        if(i == Size-1) 
            Soft_I2C_Send_NACK(); 
        else 
            Soft_I2C_Send_ACK();
    }
    Soft_I2C_Stop();
}
```

---

## 💻 2. HAL库 (Hardware Abstraction Layer)

### 2.1 硬件 I2C 写法
> **避坑指南**：HAL库的 `Mem_Write / Mem_Read` 极大简化了流程，但请注意，它要求的**是从机的 7位地址左移1位后的值**！

```c
#include "stm32f1xx_hal.h"

I2C_HandleTypeDef hi2c1;

/**
  * @brief  I2C1 初始化 (400kHz, PB6/PB7)
  */
void MX_I2C1_Init(void)
{
  hi2c1.Instance = I2C1;
  hi2c1.Init.ClockSpeed = 400000;                // 400kHz 快速模式
  hi2c1.Init.DutyCycle = I2C_DUTYCYCLE_2;        // 2:1 占空比
  hi2c1.Init.OwnAddress1 = 0;                    // 填0代表不使用从机模式
  hi2c1.Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT; 
  hi2c1.Init.DualAddressMode = I2C_DUALADDRESS_DISABLE;
  hi2c1.Init.OwnAddress2 = 0;
  hi2c1.Init.GeneralCallMode = I2C_GENERALCALL_DISABLE;
  hi2c1.Init.NoStretchMode = I2C_NOSTRETCH_DISABLE; // 允许从机拉伸时钟(重要)
  HAL_I2C_Init(&hi2c1);
}

/**
  * @brief  底层硬件初始化 (HAL_I2C_Init 会自动调用它)
  */
void HAL_I2C_MspInit(I2C_HandleTypeDef* i2cHandle)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  if(i2cHandle->Instance==I2C1)
  {
    __HAL_RCC_GPIOB_CLK_ENABLE();
    __HAL_RCC_I2C1_CLK_ENABLE();

    // I2C硬件引脚必须 = 复用开漏输出(AF_OD)
    GPIO_InitStruct.Pin = GPIO_PIN_6|GPIO_PIN_7;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_OD;
    GPIO_InitStruct.Pull = GPIO_PULLUP;        // 内部上拉 (外部也必须接上拉电阻)
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
  }
}

/**
 * @brief  HAL库 I2C 写数据 (封装了起始+地址+寄存器+数据+停止)
 * @param  SlaveAddr: 7位从机地址 (0x00~0x7F)
 */
HAL_StatusTypeDef I2C_Write(I2C_HandleTypeDef *hi2c, uint8_t SlaveAddr, uint8_t RegAddr, uint8_t *pData, uint16_t Size)
{
  // 注意：SlaveAddr << 1，HAL库内部会自动将最低位清0代表写
  return HAL_I2C_Mem_Write(hi2c, SlaveAddr << 1, RegAddr, I2C_MEMADD_SIZE_8BIT, pData, Size, 1000);
}

/**
 * @brief  HAL库 I2C 读数据 (封装了重复起始和ACK管理)
 * @param  SlaveAddr: 7位从机地址 (0x00~0x7F)
 */
HAL_StatusTypeDef I2C_Read(I2C_HandleTypeDef *hi2c, uint8_t SlaveAddr, uint8_t RegAddr, uint8_t *pBuf, uint16_t Size)
{
  // 注意：SlaveAddr << 1，HAL库内部会自动将最低位置1代表读
  return HAL_I2C_Mem_Read(hi2c, SlaveAddr << 1, RegAddr, I2C_MEMADD_SIZE_8BIT, pBuf, Size, 1000);
}
```

### 2.2 软 I2C 写法 (HAL库环境)
> **说明**：这里将原先笔记里 `SlaveAddr` 左移的操作统一保留，即外部传入的一律当作 **7位原地址** 处理。

```c
#include "stm32f1xx_hal.h"

/************************* 1. 软I2C引脚定义 *************************/
#define SCL_PIN GPIO_PIN_6
#define SDA_PIN GPIO_PIN_7
#define I2C_GPIO_PORT GPIOB

/************************* 2. 引脚电平操作宏 *************************/
// 注意：如果想要追求极速，可以用 GPIOB->BSRR 寄存器直接替换 HAL 函数
#define I2C_SCL_HIGH() HAL_GPIO_WritePin(I2C_GPIO_PORT, SCL_PIN, GPIO_PIN_SET)
#define I2C_SCL_LOW()  HAL_GPIO_WritePin(I2C_GPIO_PORT, SCL_PIN, GPIO_PIN_RESET)
#define I2C_SDA_HIGH() HAL_GPIO_WritePin(I2C_GPIO_PORT, SDA_PIN, GPIO_PIN_SET)
#define I2C_SDA_LOW()  HAL_GPIO_WritePin(I2C_GPIO_PORT, SDA_PIN, GPIO_PIN_RESET)
#define I2C_READ_SDA() HAL_GPIO_ReadPin(I2C_GPIO_PORT, SDA_PIN)

/************************* 3. 微秒延时 *************************/
void I2C_Delay_us(uint32_t us)
{
    uint8_t i;
    while(us--)
    {
        for(i=0; i<72; i++); // 适配 STM32F103 72M 主频的粗略延时
    }
}

/************************* 4. 软I2C初始化 *************************/
void Soft_I2C_Init(void)
{
    __HAL_RCC_GPIOB_CLK_ENABLE();
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    // GPIO配置：普通开漏输出 (OD)
    GPIO_InitStruct.Pin = SCL_PIN | SDA_PIN;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_OD;  
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(I2C_GPIO_PORT, &GPIO_InitStruct);

    I2C_SCL_HIGH();
    I2C_SDA_HIGH();
}

/************************* 5. I2C核心时序函数 *************************/
void Soft_I2C_Start(void)
{
    I2C_SDA_HIGH();
    I2C_SCL_HIGH();
    I2C_Delay_us(2);
    I2C_SDA_LOW();   
    I2C_Delay_us(2);
    I2C_SCL_LOW();   
}

void Soft_I2C_Stop(void)
{
    I2C_SDA_LOW();
    I2C_SCL_HIGH();
    I2C_Delay_us(2);
    I2C_SDA_HIGH();   
    I2C_Delay_us(2);
}

void Soft_I2C_Send_ACK(void)
{
    I2C_SDA_LOW();
    I2C_SCL_HIGH();
    I2C_Delay_us(2);
    I2C_SCL_LOW();
    I2C_SDA_HIGH();
}

void Soft_I2C_Send_NACK(void)
{
    I2C_SDA_HIGH();
    I2C_SCL_HIGH();
    I2C_Delay_us(2);
    I2C_SCL_LOW();
}

uint8_t Soft_I2C_Wait_ACK(void)
{
    uint8_t retry = 0;
    I2C_SDA_HIGH();  
    I2C_SCL_HIGH();
    I2C_Delay_us(2);

    while(I2C_READ_SDA())
    {
        retry++;
        if(retry > 200)
        {
            Soft_I2C_Stop();
            return 1;
        }
    }
    I2C_SCL_LOW();
    return 0;
}

/************************* 6. 单字节读写 *************************/
void Soft_I2C_Send_Byte(uint8_t byte)
{
    for(uint8_t i=0; i<8; i++)
    {
        I2C_SCL_LOW();
        if(byte & 0x80) I2C_SDA_HIGH();
        else I2C_SDA_LOW();
        byte <<= 1;
        I2C_Delay_us(2);
        I2C_SCL_HIGH();
        I2C_Delay_us(2);
    }
    I2C_SCL_LOW();
}

uint8_t Soft_I2C_Read_Byte(void)
{
    uint8_t byte = 0;
    I2C_SDA_HIGH();
    for(uint8_t i=0; i<8; i++)
    {
        I2C_SCL_LOW();
        I2C_Delay_us(2);
        I2C_SCL_HIGH();
        byte <<= 1;
        if(I2C_READ_SDA()) byte++;
        I2C_Delay_us(2);
    }
    I2C_SCL_LOW();
    return byte;
}

/************************* 7. 标准读写 (参数 SlaveAddr 视为7位地址) *************************/
void Soft_I2C_Write(uint8_t SlaveAddr, uint8_t RegAddr, uint8_t *pData, uint16_t Size)
{
    Soft_I2C_Start();
    // 传入7位地址，左移1位，最低位默认是0(写)
    Soft_I2C_Send_Byte(SlaveAddr << 1);  
    Soft_I2C_Wait_ACK();

    Soft_I2C_Send_Byte(RegAddr);         
    Soft_I2C_Wait_ACK();

    for(uint16_t i=0; i<Size; i++)
    {
        Soft_I2C_Send_Byte(pData[i]);
        Soft_I2C_Wait_ACK();
    }
    Soft_I2C_Stop();
}

void Soft_I2C_Read(uint8_t SlaveAddr, uint8_t RegAddr, uint8_t *pBuf, uint16_t Size)
{
    Soft_I2C_Start();
    // 写地址
    Soft_I2C_Send_Byte(SlaveAddr << 1);
    Soft_I2C_Wait_ACK();
    
    // 写寄存器
    Soft_I2C_Send_Byte(RegAddr);
    Soft_I2C_Wait_ACK();

    // 重复起始
    Soft_I2C_Start();
    // 读地址：左移1位后，与 0x01 作或运算，把最低位置 1
    Soft_I2C_Send_Byte((SlaveAddr << 1) | 0x01);  
    Soft_I2C_Wait_ACK();

    // 读取数据
    for(uint16_t i=0; i<Size; i++)
    {
        pBuf[i] = Soft_I2C_Read_Byte();
        if(i == Size-1) 
            Soft_I2C_Send_NACK();
        else 
            Soft_I2C_Send_ACK();
    }
    Soft_I2C_Stop();
}
```