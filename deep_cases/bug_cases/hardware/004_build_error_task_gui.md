# 004: HelpPort编译错误 task_gui.h not found

## 项目
HelpPort (STM32F407VET6 + FreeRTOS + LVGL)

## 环境
MCU: STM32F407VET6, 编译器: Keil V5.06, RTOS: FreeRTOS, GUI: LVGL

## 现象
编译错误：`../Core/Src/freertos.c(23): error: #5: cannot open source input file "task_gui.h"`

## 影响范围
整个工程无法编译

## 排查过程

### 假设1: 文件不存在
- 验证方法：搜索task_gui.h
- 结果：文件确实不存在于项目中

### 假设2: 头文件路径未配置
- 验证方法：检查Include Paths
- 结果：路径正确但文件本身不存在

### 假设3: 文件被删除/未提交
- 验证方法：检查git历史
- 结果：task_gui.c/.h从未被创建

## 根因
代码引用了`task_gui.h`但文件不存在。任务实现已内联到freertos.c，但include未清理。

## 修改方案
```c
// 方案1: 创建空头文件
// task_gui.h
#ifndef __TASK_GUI_H
#define __TASK_GUI_H
#include "cmsis_os2.h"
#endif

// 方案2: 删除#include "task_gui.h"，直接声明extern
// freertos.c中删除该行
```

## 验证结果
编译通过

## 预防措施
- 删除源文件时同步删除#include
- 使用IDE的"查找引用"功能检查

## 经验规则
- cannot open source input file = 文件不存在或路径未配置
- 先查文件是否存在，再查路径配置

## 来源
ses_0cb3f538effeb1kVU8F3wDvGwg - 2026-07-06
