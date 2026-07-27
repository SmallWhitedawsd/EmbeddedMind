# 看门狗/启动时序/复位诊断

---

# 上电残压导致 POR 异常

## 场景
断电后短时间内重新上电，系统启动异常（40s 才亮屏）。

## 现象
首次上电 11s 亮屏，断电 5s 再上电需 40s 才亮屏。

## 根因
- VDD/LT7580 残留电荷未放净 → POR 未触发 → 内部状态不确定
- SDRAM 始终不 ready → `Check_SDRAM_Ready()` 死等 → IWDG 30s 复位 → 第二次才点亮
- 41s = ~30s IWDG 超时 + 11s 正常启动

## 解决方案
```c
// 1. 加放电电阻: LT7580 VDD 对地 1kΩ/0805
//    τ = R×C = 1k×470µF = 470ms, 5τ ≈ 2.35s 放完
//    P = V²/R = 25/1000 = 25mW < 62.5mW (0805 50%降额)

// 2. Check_SDRAM_Ready 加超时返回
int Check_SDRAM_Ready(void) {
    uint32_t start = HAL_GetTick();
    uint16_t temp;
    do {
        temp = LCD_StatusRead();
        if (HAL_GetTick() - start > 500) return -1;  // 500ms 超时
    } while ((temp & 0x04) == 0);
    return 0;
}

// 3. LCD_Init 重试 + 上电沉降
int LCD_Init(void) {
    // 上电沉降: 等 1.5s，每 100ms 喂狗
    for (int i = 0; i < 15; i++) { HAL_Delay(100); HAL_IWDG_Refresh(&hiwdg); }
    
    // 自适应重试: 最多 8 次
    for (int retry = 0; retry < 8; retry++) {
        LCD_Reset();
        if (LCD_PLL_Initial() == 0 && LCD_SDRAM_initail() == 0) return 0;
    }
    // 兜底: 整机复位
    HAL_NVIC_SystemReset();
}
```

## 验证
- 连续断电-上电 20 次，每次均在 12s 内亮屏
- 3 分钟监听脚本: 每 6s 读 RCC_CSR，IWDGRSTF(bit29) 始终为 0

## 来源
HelpPort - 利用看门狗复位实现12秒稳定亮屏 (2026-07-08)

---

# 启动时序优化

## 场景
系统上电后需要等待外设稳定再初始化。

## 现象
LCD 初始化失败，因为 SDRAM/PLL 未稳定。

## 根因
外设上电后需要稳定时间，立即初始化会失败。

## 解决方案
```
正常启动时序:
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

## 验证
- 逻辑分析仪/示波器抓取各信号时序
- 确认 PLL 锁定后 (reg 0x00 bit7=1) 才进行 SDRAM 初始化

## 来源
HelpPort - 拉取 HelpPort_NEW 最新代码 (2026-07-09)

---

# 复位引脚未正确释放导致启动失败

## 场景
上电后按复位键才能正常工作。

## 现象
上电直接运行异常，手动复位后正常。

## 根因
NRST 引脚电容过大或上拉电阻不合适，导致复位释放时电源未稳定。

## 解决方案
```c
// 硬件: NRST 引脚 10kΩ 上拉 + 100nF 对地
// 软件: 启动后加延时等待电源稳定
int main(void) {
    HAL_Init();
    SystemClock_Config();
    
    // 等待电源稳定
    HAL_Delay(100);
    
    // 清复位标志
    __HAL_RCC_CLEAR_RESET_FLAGS();
    
    // 判断复位源
    if (__HAL_RCC_GET_FLAG(RCC_FLAG_IWDGRST)) {
        // 看门狗复位处理
    }
}
```

## 验证
- 连续上电 10 次均正常启动
- 无需手动复位

## 来源
单片机设备 - 项目模块测试列表整理 (2026-07-20)

---

# 看门狗喂狗策略

## 场景
IWDG 超时时间内需要多个初始化步骤完成。

## 现象
初始化过程中因未喂狗导致复位。

## 根因
`LCD_Init()` 全程无喂狗，卡住后 30s 后 IWDG 硬复位。

## 解决方案
```c
// 策略1: 长超时 + 关键节点喂狗
hiwdg.Init.Reload = 3750;  // 30s 超时，给初始化足够时间

// 策略2: 初始化各阶段喂狗
void LCD_Init_FeedDog(void) {
    for (int i = 0; i < 15; i++) {
        HAL_Delay(100);
        HAL_IWDG_Refresh(&hiwdg);  // 每 100ms 喂狗
    }
}

// 策略3: 状态轮询加超时+喂狗
int Check_2D_Busy(void) {
    uint32_t start = HAL_GetTick();
    while (LCD_StatusRead() & 0x02) {
        if (HAL_GetTick() - start > 3000) return -1;
        HAL_IWDG_Refresh(&hiwdg);  // 轮询中也喂狗
    }
    return 0;
}
```

## 验证
- 初始化过程中不复位
- 真正卡死时能超时复位

## 来源
HelpPort - 首次上电11s亮屏断电5s再上电需40s分析 (2026-07-08)

---

# 上电前沿高频纹波诊断

## 场景
加放电电阻后仍有启动问题。

## 现象
示波器发现上电前沿有高频纹波。

## 根因
电源开关瞬间产生高频噪声，影响 MCU 和外设初始化。

## 解决方案
```c
// 硬件:
// 1. 电源输入加 TVS 管 + 电解电容
// 2. MCU VDD 引脚加 100nF 陶瓷电容就近放置
// 3. 放电电阻加在 LT7580 VDD 对地（不是 RST 引脚）

// 软件:
// 上电后等待电源稳定再开始初始化
HAL_Delay(50);  // 等待纹波衰减
```

## 验证
- 示波器看 VDD 上电前沿无明显高频纹波
- 连续上电测试 50 次均正常

## 来源
HelpPort - 拉取 HelpPort_NEW 最新代码 (2026-07-09)
