# 001: FreeRTOS任务优先级导致GUI被DAQ任务阻塞

## 项目
HelpPort (STM32F407VET6 + FreeRTOS + LVGL + AD7606)

## 环境
MCU: STM32F407VET6, RTOS: FreeRTOS, GUI: LVGL, ADC: AD7606 200kHz

## 现象
AD7606以200kHz采样时，GUI任务完全无响应，光标卡顿

## 影响范围
整个LVGL界面交互

## 排查过程

### 假设1: DAQ_Task优先级过高
- 验证方法：检查DAQ_Task优先级配置
- 结果：DAQ_Task=osPriorityRealtime(24)，GUI_Task=Normal

### 假设2: 中断抢占GUI
- 验证方法：检查EXTI3中断优先级
- 结果：EXTI3优先级=5，会抢占SysTick(15)和GUI任务

## 根因
**DAQ_Task优先级为osPriorityRealtime**，每5ms处理1000×8数据时完全阻塞GUI任务。加上EXTI3每年20万次中断，GUI任务几乎无法运行。

## 修改方案
```c
// freertos.c - 降低DAQ_Task优先级
// 修改前: osPriorityRealtime (24)
// 修改后: osPriorityHigh (16)
xTaskCreate(DAQ_Task, "DAQ", 512, NULL, osPriorityHigh, NULL);

// 配合DMA方案后，DAQ_Task只需在DMA TC中断中做buffer切换
// 不再需要实时优先级
```

## 验证结果
| 指标 | 修改前 | 修改后 |
|------|--------|--------|
| DAQ_Task优先级 | Realtime(24) | High(16) |
| GUI响应 | 完全阻塞 | 正常响应 |
| ADC数据完整性 | 正常 | 正常(DMA保证) |

## 预防措施
- RealTime优先级任务只用于极短时间的临界操作
- 长时间数据处理用High/Normal优先级
- 高频采集用DMA而非中断+任务

## 经验规则
- osPriorityRealtime会阻塞所有低优先级任务
- 数据采集≠需要实时优先级，DMA+High足够
- 中断优先级必须低于configMAX_SYSCALL_INTERRUPT_PRIORITY

## 来源
ses_0ddc710f7ffeQR7XFY9X25oqa1 - 2026-07-03
