# 001: 首次上电11s亮屏，断电5s再上电需40s

## 项目
HelpPort (STM32F407VET6 + LT7580 LCD控制器 + FreeRTOS)

## 环境
MCU: STM32F407VET6, LCD驱动: LT7580(1024×600 7寸), 看门狗: IWDG, RTOS: FreeRTOS

## 现象
冷启动11秒正常亮屏；断电5秒后重新上电，屏幕需约40秒才能点亮

## 影响范围
用户体验、LCD初始化可靠性

## 排查过程

### 假设1: LCD电源未放尽导致POR失效
- 验证方法：分析LT7580 POR要求
- 结果：断电5s，LT7580电源/内部状态未放尽，不构成有效POR

### 假设2: IWDG超时30s
- 验证方法：计算IWDG超时 Prescaler=256, Reload=3750, LSI~32kHz
- 结果：32000/256=125Hz, 3750/125=**30s** → 30s(冻死)+11s(正常启动)=41s≈观察到的40s

### 假设3: LCD_Init中死循环无超时
- 验证方法：检查Check_SDRAM_Ready/Check_Busy_Draw实现
- 结果：`do{}while((temp&0x04)==0)` 无超时→SDRAM未ready时永远等待

### 假设4: 1s强制复位方案可行性
- 验证方法：分析复位对LT7580的影响
- 结果：不可靠！LT7580需30s才稳定，1s复位时LT7580仍不稳定→可能仍卡

## 根因
**LT7580需完整掉电才能POR**。断电5s时：
- MCU发的LCD_Reset(RST拉低100ms→拉高800ms)对未真正掉电的控制器不构成有效上电复位
- PLL/SDRAM停在坏状态→Check_SDRAM_Ready永远不置位→死循环30s→IWDG复位→第二次启动才成功

## 修改方案
```c
// lcd.c - Check_SDRAM_Ready 加超时
int Check_SDRAM_Ready(void) {
    uint32_t tick = HAL_GetTick();
    uint8_t temp;
    do {
        temp = LCD_StatusRead();
        if (HAL_GetTick() - tick > 500) return 0; // 超时
    } while ((temp & 0x04) == 0);
    return 1;
}

// lcd.c - LCD_Init 重试循环
for (int retry = 0; retry < 5; retry++) {
    LCD_Reset(); // 硬件复位LT7580
    if (LCD_PLL_Initial() && LCD_SDRAM_initail(MCLK))
        break; // 成功
}
```

## 验证结果
坏启动不再等30s IWDG，~2s内自检恢复重试，稳定~12s点亮

## 预防措施
- 所有硬件状态轮询必须加超时
- LCD_Init内部/各轮询循环里喂狗或缩短IWDG超时
- 延长RST低电平时间或确保LT7580电源在断电时能快速泄放

## 经验规则
- 状态轮询`do{}while(!ready)`必须有timeout
- IWDG超时必须覆盖最坏情况初始化时间
- 热启动≠冷启动，控制器内部状态可能残留

## 来源
ses_0bf3076d5ffee5YTz8Dv51QfAk - 2026-07-08
