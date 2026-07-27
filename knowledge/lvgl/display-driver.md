# LT7580 显示驱动初始化时序

## 场景
STM32F407 通过 SPI 驱动 LT7580 LCD 控制器，上电后需稳定点亮屏幕。

## 现象/问题
首次上电 11s 亮屏，断电 5s 再上电需 40s 才亮屏。

## 根因分析
- IWDG 超时 ~30s（Prescaler=256, Reload=3750, LSI≈32kHz）
- `Check_SDRAM_Ready()` 无超时死等：`do{ temp=LCD_StatusRead(); }while((temp&0x04)==0);`
- 残压导致 SDRAM 不 ready → 死等 → IWDG 复位 → 第二次才点亮

## 解决方案
**v3 最终方案**：
```c
// lcd.c - LCD_Init 重试循环
int LCD_Init(void) {
    // 1. 上电沉降：等 1.5s，每 100ms 喂狗
    for (int i = 0; i < 15; i++) { HAL_IWDG_Refresh(&hiwdg); HAL_Delay(100); }
    
    // 2. 自适应重试：最多 8 次硬件复位 + 重初始化
    for (int retry = 0; retry < 8; retry++) {
        LCD_Reset();
        if (LCD_PLL_Initial() == 0 && LCD_SDRAM_initail() == 0)
            return 0;  // 成功
    }
    
    // 3. 兜底：整机复位
    HAL_NVIC_SystemReset();
    return -1;
}
```

**Check_SDRAM_Ready 加超时**：
```c
int Check_SDRAM_Ready(void) {
    uint32_t start = HAL_GetTick();
    uint8_t temp;
    do {
        temp = LCD_StatusRead();
        if (HAL_GetTick() - start > 3000) return -1;  // 3s 超时
    } while ((temp & 0x04) == 0);
    return 0;
}
```

## 验证方法
3 分钟监听脚本，每 6s 读 RCC_CSR，确认 IWDGRSTF(bit29) 不触发。

## 来源
HelpPort - 利用看门狗复位实现12秒稳定亮屏 (2026-07-08)

---

# LT7580 正常启动时序

## 场景
分析 LCD 从上电到点亮的时间分布。

## 现象/问题
需确认各阶段耗时，优化启动速度。

## 根因分析
```
0.0s  IWDG init (reload=312, ~2.5s)
0.0s  SystemClock_Config
0.0s  外设初始化 (GPIO/SPI/FSMC/UART...)
1.5s  LCD 供电稳定窗口 (15×100ms, 每100ms喂狗)
2.4s  LCD_Reset (100ms low + 800ms high)
2.4s  LCD_PLL_Initial (PLL 锁 ~1ms + delay 10ms)
2.4s  LCD_SDRAM_initail → Check_SDRAM_Ready 第1次即就绪
2.6s  Init_Font (2.1MB SPI Flash 字库加载)
~11s  屏幕点亮
```

## 解决方案
- 上电沉降 1.5s 不可省略（LT7580 供电稳定需要）
- 字库加载 2.1MB 是主要耗时点，可考虑压缩或异步加载

## 验证方法
示波器抓 LCD RST 引脚，确认 100ms low + 800ms high 时序。

## 来源
HelpPort - 拉取 HelpPort_NEW 最新代码 (2026-07-09)

---

# SPI 波特率与信号完整性

## 场景
LT7580 通过 SPI 接口连接 STM32，杜邦线布线。

## 现象/问题
SPI 波特率从 /32 改到 /16 就黑屏（20cm FPC 软排线）。

## 根因分析
- /32 = 2.625MHz（杜邦线极限）
- /16 = 5.25MHz → 20cm FPC 软排线 + 高速信号 → 信号完整性问题
- 反射/振铃导致 LT7580 通信失败

## 解决方案
```c
// spi.c - 保持 /32
hspi1.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_32;

// 零硬件成本优化：降低 GPIO 压摆率
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;  // SCK/MOSI
```

降低 GPIO 压摆率 → 减少信号反射/振铃 → 可能在高频下稳定。

## 验证方法
用示波器观察 SCK 波形，确认无明显过冲/振铃。

## 来源
HelpPort - 拉取 Ponytail 作为技能使用 (2026-06-25)

---

# 显示接口引脚配置

## 场景
STM32F407 连接 LT7580 的 SPI 接口。

## 现象/问题
需确认引脚定义和复用功能。

## 根因分析
PA15/PB3 默认是 JTAG 引脚，需禁用 JTAG 才能用作 SPI。

## 解决方案
| 信号 | 引脚 | 备注 |
|------|------|------|
| SCS (片选) | PA15 | 需禁用 JTAG |
| SCL (时钟) | PB3 | 需禁用 JTAG |
| SDO (数据出) | PA6 | |
| SDI (数据入) | PA7 | |
| RST (复位) | PC8 | |

SPI 模式：CPOL=High, CPHA=2 Edge (Mode 3)

## 验证方法
初始化后读 LT7580 寄存器 0x00 bit7（PLL 锁定状态），确认通信正常。

## 来源
HelpPort - 附录关键技术点 (2026-07-27)

---

# 放电电阻选型

## 场景
LT7580 VDD 残压导致 POR 异常，需加放电电阻。

## 现象/问题
断电后 VDD 残留电荷未放净，SDRAM 状态不确定。

## 根因分析
τ = R × C，完全放电 ≈ 5τ。C=470µF，目标 τ=0.2s → R=430Ω。

## 解决方案
```c
// 选型：1kΩ / 0805
// τ = 1k × 470µF = 470ms
// 5τ = 2.35s 放完
// 功耗：P = V²/R = 25/1000 = 25mW
// 0805 额定功率 1/8W，50%降额 ≤62.5mW ✓
```

**加在 LT7580 VDD 对地**（不是 RST 引脚）。

## 验证方法
示波器观察 VDD 掉电波形，确认 2s 内跌落到 0V。

## 来源
HelpPort - 利用看门狗复位实现12秒稳定亮屏 (2026-07-08)
