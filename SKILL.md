# EmbeddedMind

## Description
AI embedded engineering knowledge for STM32/LVGL/Modbus/FreeRTOS development. Covers HelpPort, WSC-16, SX-MK-010, and 74 STM32 teaching modules.

## When to use
触发条件: STM32、LVGL、Modbus、RS485、FreeRTOS、嵌入式调试、IWDG、FSMC、LT7580、TJC串口屏、RN7302、MFRC522、485三色灯、HC-433

## Knowledge Base

### 项目知识 (projects/)
| 文件 | 项目 | 内容 |
|------|------|------|
| `projects/helpport.md` | HelpPort | STM32F407+LT7580+LVGL, 上电时序/IWDG/LCD初始化/放电电阻 |
| `projects/wsc-16.md` | WSC-16 | STM32F103+485三色灯+HC-433, Modbus RTU/GTI协议/状态机 |
| `projects/sx-mk-010.md` | SX-MK-010 | STM32F103智能电源, 继电器控制/RFID/RN7302/Modbus |
| `projects/stm32-modules.md` | STM32教学平台 | 74个外设模块, 插槽定义/测试方法/代码模式 |

### 调试模式 (patterns/)
| 文件 | 内容 |
|------|------|
| `patterns/debugging-patterns.md` | 10种调试模式: 启动异常/通信速率/触屏卡顿/Modbus无应答/状态机/EMI/看门狗/竞态/编译 |
| `patterns/design-patterns.md` | 10种设计模式: 五层架构/状态机/超时保护/喂狗策略/帧协议/环形缓冲/放电电阻/抗干扰/Flash存储/任务优先级 |
| `patterns/troubleshooting.md` | 决策树: 启动/屏幕/通信/触摸/继电器/倒计时/编译/复位/性能 |

## Workflow

1. **判断项目类型**
   - HelpPort → LT7580/LVGL/STM32F407
   - WSC-16 → 485三色灯/HC-433/STM32F103
   - SX-MK-010 → 继电器/RFID/RN7302
   - 模块平台 → 74个外设模块

2. **加载对应项目知识**
   - 引脚定义、寄存器地址、关键数值

3. **搜索相关调试模式**
   - 现象→定位→根因→解决→验证

4. **执行修改**
   - 遵循代码风格、编码格式 (GB2312/UTF-8)
   - 注意字节安全编辑

5. **验证**
   - ST-Link读寄存器确认
   - 示波器/逻辑分析仪波形确认
   - 连续多次测试稳定性

## Conventions

### 代码风格
- 不添加注释 (除非用户要求)
- 保持原有命名风格
- GB2312编码文件需字节安全编辑

### 提交规则
- 提交前检查 `git status` / `git diff`
- 提交信息: `type(scope): description`
- 不提交临时文件

### 调试规范
- 先读RCC_CSR判断复位原因
- 先排除硬件再排查软件
- 关键信号用示波器确认
- 修改后连续≥10次验证稳定性

### 关键数值
- IWDG 30s超时: Prescaler=256, Reload=3750, LSI=32kHz
- SPI /32 = 2.625MHz (杜邦线极限)
- 放电电阻: 1kΩ/0805, 25mW
- Modbus CRC16: 多项式0xA001
- GTI协议: 14字节, 1200波特率
