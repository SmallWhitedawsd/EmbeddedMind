# EmbeddedMind

## Description
AI embedded engineering knowledge distilled from 285 real-world sessions (8,578 messages, 111K events, 248 code patches). Covers STM32/LVGL/Modbus/FreeRTOS development with complete bug-reasoning-verification loops.

## When to use
触发条件: STM32、LVGL、Modbus、RS485、FreeRTOS、嵌入式调试、IWDG、FSMC、LT7580、TJC串口屏、RN7302、MFRC522、485三色灯、HC-433、OV7670、ESP01、AD7606

## Knowledge Base

### 一轮：领域知识 (knowledge/)
| 文件 | 内容 |
|------|------|
| `mcu/stm32-general.md` | STM32F1/F4 配置、IWDG、时钟、BOR |
| `mcu/peripherals.md` | UART+DMA、RS485、SPI、I2C、FSMC、TIM、ADC |
| `firmware/freertos.md` | 任务设计、osDelay竞态、临界区、栈水位 |
| `firmware/boot-watchdog.md` | 启动时序、复位诊断、放电电阻、喂狗策略 |
| `lvgl/architecture.md` | 五层分层、LT7580职责、GUI任务调度 |
| `lvgl/display-driver.md` | LT7580初始化、SPI信号完整性、引脚配置 |
| `lvgl/touch-driver.md` | GT9147软件I2C、RST/INT时序 |
| `lvgl/performance.md` | 任务周期、AD7606降频、双缓冲 |
| `protocols/modbus.md` | RTU格式、CRC16、寄存器映射、BUG案例 |
| `protocols/rs485.md` | 半双工切换、终端电阻、收发延迟 |
| `protocols/uart-fifo.md` | 环形缓冲、超时检测、DMA vs 中断 |
| `hardware-debug/debug-methodology.md` | ST-Link CLI、Modbus调试、放电计算 |

### 二轮：深度案例 (deep_cases/)
| 文件 | 内容 | 数量 |
|------|------|------|
| `patch_cases.md` | 代码修改完整上下文 | 155个 |
| `bug_cases/` | Bug闭环（现象→假设→验证→根因→修改→验证） | 32个 |
| `workflows/` | 编译/烧录/调试/串口/网络标准流程 | 18个 |
| `reasoning/` | 推理方法论 + 项目架构图 | 17个 |

### 项目知识 (projects/)
| 文件 | 项目 | 内容 |
|------|------|------|
| `helpport.md` | HelpPort | STM32F407+LT7580+LVGL, 上电时序/IWDG/LCD初始化 |
| `wsc-16.md` | WSC-16 | STM32F103+485三色灯+HC-433, Modbus/GTI协议 |
| `sx-mk-010.md` | SX-MK-010 | STM32F103智能电源, 继电器/RFID/RN7302 |
| `stm32-modules.md` | STM32教学平台 | 74个外设模块, 插槽定义 |

### 模式库 (patterns/)
| 文件 | 内容 |
|------|------|
| `debugging-patterns.md` | 10种调试模式 (启动/通信/触屏/Modbus/状态机/EMI/看门狗) |
| `design-patterns.md` | 10种设计模式 (五层架构/状态机/超时保护/喂狗策略) |
| `troubleshooting.md` | 7类决策树 (启动/屏幕/通信/触摸/继电器/编译/复位) |

## Workflow

### 1. 判断项目类型
- HelpPort → LT7580/LVGL/STM32F407
- WSC-16 → 485三色灯/HC-433/STM32F103
- SX-MK-010 → 继电器/RFID/RN7302
- 模块平台 → 74个外设模块

### 2. 加载对应项目知识
- 引脚定义、寄存器地址、关键数值
- 查阅 `projects/[项目].md`

### 3. 搜索历史案例
- 遇到 Bug → 查 `bug_cases/` 是否有相似案例
- 需要修改 → 查 `patch_cases.md` 是否有先例
- 参考 `reasoning/debug-methodology-v2.md` 推理方法

### 4. 执行修改
- 遵循代码风格、编码格式 (GB2312/UTF-8)
- 注意字节安全编辑
- 参考 `workflows/build-flash-debug.md` 验证

### 5. 验证
- ST-Link读寄存器确认
- 示波器/逻辑分析仪波形确认
- 连续多次测试稳定性

## Debug Reasoning (from deep_cases/reasoning/)

### 硬件异常排查顺序
1. 电源 → 电压/纹波/上电时序
2. 时钟 → 晶振/PLL/时钟树
3. 复位 → RCC_CSR寄存器判读
4. 通信 → 波特率/接线/信号完整性

### 软件异常排查顺序
1. 任务 → 优先级/栈水位/死锁
2. 中断 → 优先级/嵌套/ISR耗时
3. 状态机 → 状态转换/时序竞争
4. 资源竞争 → DMA总线/临界区

### 时间分析法
- 异常时间 ≈ 固定周期 + 看门狗超时 → 检查IWDG
- 异常时间 ≈ 上电时间 × N → 检查重试/复位
- 随机时间 → 检查EMI/竞态/中断

## Conventions

### 代码风格
- 不添加注释 (除非用户要求)
- 保持原有命名风格
- GB2312编码文件需字节安全编辑

### 关键数值
- IWDG 30s超时: Prescaler=256, Reload=3750, LSI=32kHz
- SPI /32 = 2.625MHz (杜邦线极限)
- 放电电阻: 1kΩ/0805, 25mW
- Modbus CRC16: 多项式0xA001
- GTI协议: 14字节, 1200波特率

### 调试规范
- 先读RCC_CSR判断复位原因
- 先排除硬件再排查软件
- 关键信号用示波器确认
- 修改后连续≥10次验证稳定性
