# 007: 嵌入式架构深度分析：BSP/HAL/LVGL/RTOS与内存管理

## 项目
HelpPort (STM32F407VET6 + FreeRTOS + LVGL + BSP)

## 环境
MCU: STM32F407VET6, RTOS: FreeRTOS, GUI: LVGL, 架构: BSP/HAL分层

## 现象
需要理解整体架构以定位复杂Bug

## 影响范围
系统整体稳定性和可维护性

## 排查过程

### 假设1: BSP层与HAL层边界不清
- 验证方法：分析代码分层
- 结果：BSP直接调用HAL，部分驱动绕过BSP

### 假设2: LVGL与RTOS集成问题
- 验证方法：检查lv_task_handler调用位置
- 结果：在GUI_Task(Normal优先级)中调用

### 假设3: 内存管理不当
- 验证方法：检查heap使用和动态分配
- 结果：LVGL使用静态buffer，FreeRTOS使用heap_4

## 根因
架构分层：
```
应用层: GUI_Task, DAQ_Task, Modbus_Task
中间件: LVGL, FreeRTOS, FatFS
BSP层: bsp_ad7606, bsp_lcd, bsp_touch
HAL层: STM32F4xx HAL Driver
硬件: STM32F407VET6外设
```

## 修改方案
```c
// 保持清晰分层:
// 1. 应用层不直接调用HAL
// 2. BSP提供统一接口
// 3. 中断/DMA在BSP层管理
// 4. RTOS任务通过消息队列通信
```

## 验证结果
架构清晰，Bug定位效率提升

## 预防措施
- 严格遵循BSP/HAL分层
- 应用层不直接操作寄存器
- 跨层通信通过消息队列/事件标志

## 经验规则
- BSP层封装硬件差异
- HAL层提供标准API
- 应用层只调用BSP接口
- 中断优先级统一管理

## 来源
ses_0780d2d53ffetbCnShun1SvFGL - 2026-07-22
