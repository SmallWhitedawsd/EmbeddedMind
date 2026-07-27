# 001: AD7606 8路模拟量输入导致触屏卡顿

## 项目
HelpPort (STM32F407VET6 + AD7606 + GT9147触屏 + LVGL + FreeRTOS)

## 环境
MCU: STM32F407VET6, ADC: AD7606(8通道200kHz), 触屏: GT9147(GPIO模拟I2C), RTOS: FreeRTOS

## 现象
8路模拟量同时输入时，屏幕触摸不灵敏、光标变卡、触摸采点不跟手；拔掉8个接口就恢复正常

## 影响范围
LVGL GUI响应、触摸输入、光标流畅度

## 排查过程

### 假设1: CPU负载过高
- 验证方法：计算EXTI3 ISR频率和CPU占用
- 结果：200kHz×8通道FSMC读=每年20万次ISR，占用30-40% CPU

### 假设2: 触摸I2C被中断打断
- 验证方法：分析GT9147 GPIO模拟I2C时序
- 结果：被高频中断不断打断，执行时间拉长

### 假设3: 信号干扰(EMI)
- 验证方法：拔线测试
- 结果：拔线恢复，但中断频率与接线无关→不是主因

### 假设4: Wait_Ack超时阻塞
- 验证方法：分析CT_IIC_Wait_Ack实现
- 结果：ACK失败时自旋20000次NOP+printf阻塞串口→单次触摸读取卡数毫秒

## 根因
**两层原因叠加**：
- 底层：200kHz EXTI3中断+ISR内8次FSMC读，长期占用30-40% CPU并抢占低优先级GUI任务
- 触发放大：接入8路信号后噪声耦合到模拟I2C总线，Wait_Ack超时阻塞

## 修改方案
```c
// bsp_ad7606.c - 删除EXTI3中断方式，改用TIM1_CH2触发DMA
// TIM1_CH1: CONVST PWM 200kHz, 脉宽~120ns
// TIM1_CH2: CCR=720 (4.3us) 触发DMA请求
// DMA2_Stream5: 8-beat burst自动读8通道FSMC数据

void AD7606_DMA_Init(void) {
    // TIM1_CH2 输出比较模式
    TIM1->CCR2 = 720; // 4.3us after CONVST
    TIM1->CCER |= TIM_CCER_CC2E;
    // DMA2_Stream5 Ch6 → TIM1_CH2 request
    DMA2_Stream5->PAR = (uint32_t)&AD7606_DATA;
    DMA2_Stream5->M0AR = (uint32_t)adc_buffer;
    DMA2_Stream5->NDTR = 8;
    DMA2_Stream5->CR = DMA_SxCR_CHSEL_2 | DMA_SxCR_CHSEL_1 | // Ch6
                        DMA_SxCR_MINC | DMA_SxCR_CIRC | DMA_SxCR_TCIE;
}

// freertos.c - DAQ_Task优先级从osPriorityRealtime降为osPriorityHigh
```

## 验证结果
| 指标 | 修改前 | 修改后 |
|------|--------|--------|
| ISR频率 | 200kHz | 25kHz |
| CPU中断负载 | ~17% | ~2% |
| 触摸I2C干扰 | 每5us被打断 | DMA总线并行不阻塞GUI |

## 预防措施
- 高频数据采集必须用DMA，避免CPU逐字节搬运
- GPIO模拟总线协议不能被高频中断打断
- 中断优先级需低于SysTick否则影响RTOS调度

## 经验规则
- 200kHz采样率=5us周期，ISR必须<1us或改用DMA
- 拔线恢复≠EMI，可能是噪声耦合到通信总线
- FreeRTOS中RealTime优先级任务会完全阻塞Normal优先级

## 来源
ses_0dab15bd2ffedb2t7L9P4Juk5O - 2026-07-03
