# 014: 触摸I2C驱动代码定位与分析

## 项目
HelpPort (STM32F407VET6 + GT9147)

## 环境
MCU: STM32F407VET6, 触屏: GT9147, 接口: I2C

## 现象
需要定位触摸I2C驱动代码以分析问题

## 影响范围
触摸功能调试

## 排查过程

### 假设1: 标准HAL I2C
- 验证方法：搜索HAL_I2C_Mem_Read
- 结果：未使用标准HAL I2C

### 假设2: GPIO模拟I2C
- 验证方法：搜索SCL/SDA GPIO定义
- 结果：找到`bsp_i2c.c`中的GPIO模拟I2C

## 根因
项目使用GPIO模拟I2C而非硬件I2C，驱动在`bsp_i2c.c`中

## 修改方案
```c
// GPIO模拟I2C关键代码
#define CT_IIC_SCL(x)  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_10, x)
#define CT_IIC_SDA(x)  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_11, x)
#define CT_SDA_Read()  HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_11)

void CT_IIC_Init(void) {
    // PB10=SCL, PB11=SDA 开漏输出
}

void CT_IIC_Start(void) {
    CT_SDA_OUT();
    CT_IIC_SCL(1); CT_IIC_SDA(1);
    CT_IIC_Delay();
    CT_IIC_SDA(0); // SCL高时SDA下降沿
    CT_IIC_Delay();
    CT_IIC_SCL(0);
}
```

## 验证结果
定位到驱动代码后可分析时序问题

## 预防措施
- GPIO模拟I2C需注意时序
- 延时函数需根据主频调整
- 中断会影响模拟I2C时序

## 经验规则
- 模拟I2C用NOP延时需考虑主频
- 中断会打乱NOP延时导致时序偏移
- 硬件I2C更可靠但引脚固定

## 来源
ses_0cb20c9e6ffe6XfxbZUx8bPKQC - 2026-07-06
