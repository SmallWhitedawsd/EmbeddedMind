# 003: EXTI中断优先级抢占SysTick导致RTOS调度异常

## 项目
HelpPort (STM32F407VET6 + FreeRTOS)

## 环境
MCU: STM32F407VET6, RTOS: FreeRTOS, 中断: EXTI3(AD7606 BUSY)

## 现象
系统运行异常，任务调度不均匀，GUI卡顿

## 影响范围
RTOS任务调度、GUI响应

## 排查过程

### 假设1: SysTick被抢占
- 验证方法：检查所有中断优先级配置
- 结果：EXTI3优先级=5，SysTick优先级=15(FreeRTOS默认)

### 假设2: 中断嵌套导致栈溢出
- 验证方法：检查中断嵌套层数
- 结果：EXTI3每年20万次，抢占SysTick

## 根因
**EXTI3中断优先级(5)高于SysTick(15)**，在EXTI3执行期间SysTick无法触发，导致RTOS时间片调度延迟。

## 修改方案
```c
// FreeRTOSConfig.h中configMAX_SYSCALL_INTERRUPT_PRIORITY设置
// 中断优先级必须≥configMAX_SYSCALL_INTERRUPT_PRIORITY才能调用RTOS API

// 方案1: 降低EXTI3优先级到≥15
HAL_NVIC_SetPriority(EXTI3_IRQn, 15, 0); // 改为与SysTick同级或更高

// 方案2: 改用DMA方案消除中断(推荐)
// DMA不产生中断(仅TC中断25kHz)，彻底解决优先级问题
```

## 验证结果
DMA方案彻底消除EXTI3中断，SysTick不再被抢占

## 预防措施
- 中断优先级必须考虑RTOS内核优先级
- configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY以上优先级不能调用RTOS API
- 高频外设优先用DMA

## 经验规则
- 数值越小优先级越高(ARM NVIC)
- SysTick和PendSV通常是最低优先级
- 高频中断(>10kHz)应避免

## 来源
ses_0ddd631c4ffeWM2vQOYliQ8zGA - 2026-07-03
