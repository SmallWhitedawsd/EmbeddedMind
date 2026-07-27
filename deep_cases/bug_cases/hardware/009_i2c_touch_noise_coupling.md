# 009: GPIO模拟I2C触摸噪声耦合导致Wait_Ack死循环

## 项目
HelpPort (STM32F407VET6 + GT9147触屏)

## 环境
MCU: STM32F407VET6, 触屏: GT9147(GPIO模拟I2C), 8路模拟量输入

## 现象
接入8路模拟量后触摸严重卡顿，拔线恢复

## 影响范围
LVGL触摸输入功能

## 排查过程

### 假设1: CPU负载过高
- 验证方法：计算中断频率
- 结果：200kHz中断占30-40% CPU但不是直接原因

### 假设2: 信号干扰(EMI)
- 验证方法：拔8路模拟量线
- 结果：拔线恢复→确认是接入信号后的问题

### 假设3: Wait_Ack超时死循环
- 验证方法：分析CT_IIC_Wait_Ack实现
- 结果：**ACK失败时自旋20000次NOP+printf阻塞串口**

## 根因
**噪声耦合到模拟I2C总线**：
- 8路模拟量线接入后噪声耦合到GT9147模拟I2C总线(PE口，开漏+上拉，抗干扰差)
- ACK失败→CT_IIC_Wait_Ack自旋20000次NOP(约2ms)→printf阻塞串口
- 每次触摸读取叠加多次这种阻塞→GUI任务一卡数毫秒→触摸严重不跟手

## 修改方案
```c
// ctp.c - 修复Wait_Ack超时
uint8_t CT_IIC_Wait_Ack(void) {
    uint16_t timeout = 100; // 有限超时而非20000
    CT_SDA_IN();
    CT_IIC_Delay();
    CT_IIC_SCL = 1;
    CT_IIC_Delay();
    while (CT_SDA_Read) {
        if (--timeout == 0) {
            CT_IIC_Stop();
            return 1; // 超时返回错误
        }
    }
    CT_IIC_SCL = 0;
    return 0;
}

// 删除printf阻塞
// 改用硬件I2C或加屏蔽线
```

## 验证结果
触摸恢复跟手，光标流畅

## 预防措施
- GPIO模拟I2C必须加有限超时
- 不要在I2C通信中加printf
- 模拟I2C走线远离高频/高噪声信号
- 优先用硬件I2C

## 经验规则
- Wait_Ack必须有超时退出
- 阻塞式I2C读取会拖累整个GUI任务
- 开漏+上拉的模拟I2C极易受EMI影响

## 来源
ses_0dab15bd2ffedb2t7L9P4Juk5O - 2026-07-03
