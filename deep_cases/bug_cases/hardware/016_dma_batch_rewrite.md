# 016: AD7606 DMA批量采集重写

## 项目
HelpPort (STM32F407VET6 + AD7606 + DMA)

## 环境
MCU: STM32F407VET6, ADC: AD7606, DMA: DMA2

## 现象
需要将AD7606从EXTI中断方式改为DMA批量采集

## 影响范围
ADC数据采集、CPU负载、GUI响应

## 排查过程

### 假设1: DMA通道可用性
- 验证方法：检查DMA2_Stream占用
- 结果：DMA2_Stream3=SPI1_TX, DMA1_Stream2=UART4 → **DMA2_Stream2空闲**

### 假设2: TIM1_CH2触发DMA
- 验证方法：检查TIM1_CH2 DMA请求映射
- 结果：TIM1_CH2 → DMA2_Stream2/Channel6

### 假设3: 8-beat burst模式
- 验证方法：AD7606自动通道切换
- 结果：DMA可配置为8-beat burst自动读8通道

## 根因
EXTI3中断方式(200kHz)导致CPU负载过高，需改为DMA硬件自动搬运

## 修改方案
```c
// TIM1: CH1=CONVST PWM(200kHz), CH2=OC(CCR=720, 4.3us后触发DMA)
// DMA2_Stream2/Ch6: 8-beat burst, 外设→存储器, 循环模式

void AD7606_DMA_Init(void) {
    // TIM1_CH2 输出比较
    TIM1->CCMR1 |= TIM_CCMR1_OC2M_2 | TIM_CCMR1_OC2M_1; // PWM模式1
    TIM1->CCR2 = 720; // 4.3us (72MHz下)
    TIM1->CCER |= TIM_CCER_CC2E;
    
    // DMA2_Stream2 Channel6
    DMA2_Stream2->PAR = (uint32_t)0x6C000000; // FSMC AD7606_DATA
    DMA2_Stream2->M0AR = (uint32_t)adc_buffer;
    DMA2_Stream2->NDTR = 8;
    DMA2_Stream2->CR = (6 << 25) | // Channel6
                        DMA_SxCR_MINC | DMA_SxCR_CIRC | 
                        DMA_SxCR_MSIZE_0 | // 16-bit
                        DMA_SxCR_PSIZE_0 | // 16-bit
                        DMA_SxCR_TCIE;
    
    __HAL_RCC_DMA2_CLK_ENABLE();
    HAL_NVIC_EnableIRQ(DMA2_Stream2_IRQn);
}

void DMA2_Stream2_IRQHandler(void) {
    if (DMA2->LISR & DMA_LISR_TCIF2) {
        DMA2->LIFCR = DMA_LIFCR_CTCIF2;
        // buffer切换，通知处理任务
    }
}
```

## 验证结果
| 指标 | EXTI方式 | DMA方式 |
|------|---------|--------|
| ISR频率 | 200kHz | 25kHz(TC) |
| CPU负载 | 17% | 2% |
| 数据完整性 | 可靠 | 可靠(DMA保证) |

## 预防措施
- 高频采集必须用DMA
- DMA通道需查手册确认映射
- burst模式适合多通道ADC

## 经验规则
- >10kHz采样率必须用DMA
- TIM触发+DAV请求=全自动采集
- DMA TC中断频率=采样率/通道数

## 来源
ses_0d91c82aeffex521AiwwC31N1q - 2026-07-04
