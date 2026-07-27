# 012: WSC-16单相与三相版本差异分析

## 项目
SX-MK-010-0140 (STM32F103 + RN7302计量芯片)

## 环境
MCU: STM32F103, 计量: RN7302, 单相/三相版本

## 现象
需要理解单相和三相版本的代码差异

## 影响范围
版本维护、Bug修复

## 排查过程

### 假设1: 仅RN7302驱动不同
- 验证方法：diff两个版本
- 结果：Control.c、RN7302.c/.h、my_flashData.h不同

### 假设2: 三相版本增加的功能
- 验证方法：分析三相Control.c
- 结果：三相增加MAX/MAX3宏、Usum/Isum聚合计算

## 根因
**单相vs三相差异**：
1. 三相增加`Usum=(Ua+Ub+Uc)/3`, `Isum=Ia+Ib+Ic`聚合
2. 三相TJC屏幕多推两个字段(Uavg, Isum)
3. Param_Default()中三相有注释掉的Kusum/Kisum

## 修改方案
```c
// 单相: Modbus reg 10/11 = Uc/Ic (原始值)
// 三相: Modbus reg 10/11 = Usum/Isum (聚合值)

// 三相Control.c
#define MAX(a,b) ((a)>(b)?(a):(b))
#define MAX3(a,b,c) MAX(MAX(a,b),c)
float Usum = (Ua + Ub + Uc) / 3.0f;
float Isum = Ia + Ib + Ic;
```

## 验证结果
理解差异后可分别维护两个版本

## 预防措施
- 版本差异文档化
- 公共逻辑提取到共享文件
- RN7302驱动差异单独维护

## 经验规则
- 三相电压取平均，电流取总和
- 版本差异主要在计量聚合层
- 控制逻辑(倒计时/Modbus/Flash)相同

## 来源
ses_0b4c429ceffeBuCyAo6VBOj1Nb - 2026-07-10
