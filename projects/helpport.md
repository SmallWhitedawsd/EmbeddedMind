# HelpPort

## 硬件配置

| 参数 | 值 |
|------|-----|
| MCU | STM32F407VET6 (168MHz, 512KB Flash, 112KB SRAM) |
| 显示屏 | LT7580 外置SDRAM LCD控制器, 1024×600 RGB565 |
| 触屏 | GT9147 电容触摸 (软件I2C) |
| AD采集 | AD7606 8通道, FSMC 200kHz采样 + DMA |
| DI | 16路光耦输入 (SPI 74HC165) |
| DO | 16路继电器输出 (SPI 74HC595) |
| AI | 2路内部ADC |
| AO | 2路DAC |
| 以太网 | NE2模块 (UART4 AT指令) |
| 手写识别 | ATKNCR闭源库 |

### 关键引脚定义

**LT7580 SPI接口 (SPI1 + DMA2)**:
| 信号 | 引脚 | 备注 |
|------|------|------|
| SCS | PA15 | 需禁用JTAG |
| SCL | PB3 | 需禁用JTAG |
| SDO | PA6 | MISO |
| SDI | PA7 | MOSI |
| RST | PC8 | 复位 |

**GT9147触摸 (软件I2C)**:
| 信号 | 引脚 |
|------|------|
| SCL | PE2 |
| RST | PE3 |
| SDA | PE5 |
| INT | PE6 |

**SPI配置**: CPOL=High, CPHA=2Edge (Mode 3), /32=2.625MHz (杜邦线极限)

## 软件架构

```
┌──────────────────────────────────────────┐
│  app/tasks/           ← 业务任务           │
│  app/services/        ← 服务层             │
│  app/drivers/         ← 驱动抽象层         │
│  app/hal/             ← HAL适配层          │
│  app/config/          ← 编译期配置         │
├──────────────────────────────────────────┤
│  BSP (bsp_ad7606/lcd/ctp) ← 板级支持      │
├──────────────────────────────────────────┤
│  STM32 HAL + FreeRTOS + LVGL v8.4        │
└──────────────────────────────────────────┘
```

**FreeRTOS任务**:
| 任务 | 周期 | 说明 |
|------|------|------|
| GUI_Task | 5ms | LVGL handler (原1ms, 降频减少总线竞争) |
| DAQ_Task | 50ms | 数据采集 (原200Hz→20Hz) |

**RTOS**: FreeRTOS, LVGL v8.4

## 已知问题与解决方案

| 问题 | 原因 | 解决方案 | 验证 |
|------|------|---------|------|
| 首次上电11s亮屏, 断电5s再上电需40s | LT7580 SDRAM残压→Check_SDRAM_Ready死等→IWDG 30s超时复位 | 加放电电阻1kΩ到LT7580 VDD对地 | RCC_CSR bit29=1确认IWDG复位消除 |
| 12s/12s/29s不稳定亮屏 | Check_SDRAM_Ready 500ms超时太紧, 冷态未就绪 | 超时改3s + 8次重试 + 兜底HAL_NVIC_SystemReset() | 连续10次上电均在12s±1s亮屏 |
| SPI /16黑屏 | 20cm FPC软排线信号完整性 | GPIO SCK/MOSI速度降LOW, 或保持/32 | 回退/32后正常 |
| 触屏卡顿 | AD7606 FSMC 200kHz+DMA总线竞争 + taskENTER_CRITICAL阻塞I2C | 移除taskENTER_CRITICAL, GUI_Task 1ms→5ms, DAQ降频 | 触屏流畅, 无卡顿 |
| 上电前沿高频纹波 | LT7580 VCC上升沿振铃 | 1kΩ放电电阻 + 1.5s上电沉降窗口 | 示波器确认纹波消除 |

## 关键代码位置

| 功能 | 文件路径 | 函数名 |
|------|---------|--------|
| LCD初始化 | `HelpPort/LCD/lcd.c` | `LCD_Init` |
| SDRAM就绪检测 | `HelpPort/LCD/lcd.c` | `Check_SDRAM_Ready` |
| PLL初始化 | `HelpPort/LCD/lcd.c` | `LCD_PLL_Initial` |
| SDRAM初始化 | `HelpPort/LCD/lcd.c` | `LCD_SDRAM_initail` |
| LCD复位 | `HelpPort/LCD/lcd.c` | `LCD_Reset` |
| 字库加载 | `HelpPort/LCD/lcd.c` | `Init_Font` |
| GUI任务 | `HelpPort/Core/Src/freertos.c` | `GUI_Task` |
| 触屏驱动 | `HelpPort/Core/Src/lv_port_indev.c` | `touchpad_read` |
| 手写识别 | `HelpPort/User/ATKNCR/` | `alientek_ncr` |
| DAQ任务 | `HelpPort/app/tasks/task_daq.c` | `DAQ_Task` |

## IWDG配置

```
Prescaler = 256
Reload = 3750
LSI ≈ 32kHz
超时 = 32000/256/3750 = 30s
```

**RCC_CSR 0x40023874 寄存器位**:
| Bit | 标志 | 含义 |
|-----|------|------|
| 29 | IWDGRSTF | 看门狗复位 |
| 28 | SFTRSTF | 软件复位 |
| 27 | PORRSTF | 上电复位 |
| 26 | PINRSTF | NRST引脚复位 |
| 25 | BORRSTF | 欠压复位 |

## 正常启动时序

```
0.0s  IWDG init (reload=312, ~2.5s)
0.0s  SystemClock_Config
0.0s  外设初始化 (GPIO/SPI/FSMC/UART...)
1.5s  LCD供电稳定窗口 (15×100ms, 每100ms喂狗)
2.4s  LCD_Reset (100ms low + 800ms high)
2.4s  LCD_PLL_Initial (PLL锁~1ms + delay 10ms)
2.4s  LCD_SDRAM_initail → Check_SDRAM_Ready第1次即就绪
2.6s  Init_Font (2.1MB SPI Flash字库加载)
~11s  屏幕点亮
```

## 调试 checklist

- [ ] 上电前确认LT7580 VDD无残压
- [ ] ST-Link连接时注意VTref会掩盖启动问题
- [ ] 首次上电测量VCC上升沿是否有纹波
- [ ] 断电后再上电间隔≥5τ (1kΩ×470µF≈0.5s)
- [ ] 调试时LCD_Init全程有IWDG喂狗
- [ ] SPI波特率不超过/32 (杜邦线)
- [ ] 触屏I2C不在taskENTER_CRITICAL中操作
- [ ] DAQ频率≤20Hz避免总线竞争

## ST-Link调试命令

```powershell
# 连接目标
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG

# 读RCC_CSR判读复位原因
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG -r32 0x40023874 1

# 读选项字节确认BOR级别
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG -rOB

# 清零复位标志
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG -w32 0x40023874 0x01000000
```

## 放电电阻选型

```
τ = R × C
完全放电 ≈ 5τ
R = τ / C = 0.2s / 470µF ≈ 430Ω → 取1kΩ (保守)
功耗: P = V²/R = 25/1000 = 25mW (0805额降后62.5mW ✓)
```

## LT7580硬件寄存器

| 状态 | 寄存器/命令 | 位 |
|------|------------|-----|
| PLL锁定 | reg 0x00 | bit 7 |
| SDRAM就绪 | StatusRead() | bit 2 |

## 仓库信息

- 主仓库: `https://github.com/SmallWhitedawsd/HelpPort.git`
- 新版仓库: `https://github.com/SmallWhitedawsd/HelpPort_NEW.git`
- 本地路径: `D:\reasonix\HelpPort\HelpPort`
- 已知commit: `1329e83bb636773c53236701fa6f366fcf8e4045`
