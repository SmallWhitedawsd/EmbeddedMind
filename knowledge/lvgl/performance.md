# LVGL 任务调度周期优化

## 场景
FreeRTOS 中 LVGL 任务频率过高导致系统负载过大。

## 现象/问题
GUI 卡顿，其他任务响应延迟。

## 根因分析
`lv_task_handler()` 每 1ms 调用一次，占用大量 CPU 周期。

## 解决方案
```c
// freertos.c - GUI_Task 周期调整
void GUI_Task(void *argument) {
    for (;;) {
        lv_task_handler();
        osDelay(5);  // 从 1ms 改为 5ms
    }
}
```

**权衡**：
- 1ms：响应快但 CPU 占用高
- 5ms：平衡点，GUI 刷新率仍达 200Hz
- 10ms：可能感知到轻微延迟

## 验证方法
触摸滑动物体，观察是否跟手、无拖影。

## 来源
HelpPort - Caveman mode: ultra (代码重构) (2026-07-04)

---

# AD7606 采样频率与 GUI 性能平衡

## 场景
8 通道 AD7606 以 200kHz 采样，与 LVGL 渲染竞争总线。

## 现象/问题
GUI 渲染期间触摸响应变慢。

## 根因分析
AD7606 FSMC 200kHz 采样 + DMA 持续占用总线，与 LVGL 的 SPI 通信冲突。

## 解决方案
```c
// task_daq.c - DAQ 输出降频
void DAQ_Task(void *argument) {
    for (;;) {
        // 从 200Hz 降为 20Hz，减少 CPU 占用
        DAQ_ProcessData();
        osDelay(50);  // 原为 5ms(200Hz)，改为 50ms(20Hz)
    }
}
```

**影响**：
- DAQ 数据更新率降低，但对显示刷新无影响
- 总线竞争减少，触摸响应改善

## 验证方法
示波器观察 SPI 总线，确认无持续占用。

## 来源
HelpPort - 拉取 HelpPort_NEW 最新代码 (2026-07-09)

---

# 双缓冲/三缓冲策略

## 场景
LVGL 帧缓冲管理，避免撕裂（tearing）。

## 现象/问题
屏幕刷新时出现画面撕裂。

## 根因分析
单缓冲下，写入帧数据的同时正在显示，导致一帧内显示两个不同帧的内容。

## 解决方案
LT7580 硬件支持双缓冲：
```c
// LT7580 硬件双缓冲配置
void LCD_ConfigDoubleBuffer(void) {
    // 设置显示缓冲地址
    LCD_WriteReg(0x04, FRAME_BUFFER_A);  // 显示缓冲
    // 绘制在另一个缓冲
    LCD_WriteReg(0x05, FRAME_BUFFER_B);  // 绘制缓冲
    
    // BTE 切换（硬件级，无撕裂）
    LCD_BTE_Switch(FRAME_BUFFER_A, FRAME_BUFFER_B);
}
```

**LT7580 优势**：BTE 页面切换是硬件操作，无需 CPU 参与。

## 验证方法
快速滑动界面，观察无撕裂现象。

## 来源
HelpPort - 嵌入式架构深度分析 (2026-07-22)

---

# LVGL 刷新流程

## 场景
LVGL 从绘制到显示的完整流程。

## 现象/问题
需理解 LVGL 如何与硬件显示驱动交互。

## 根因分析
LVGL 通过 `disp_drv.flush_cb` 将绘制完成的像素缓冲发送到显示设备。

## 解决方案
```c
// lv_port_dsp.c - 显示刷新回调
static void disp_flush(lv_disp_drv_t *drv, const lv_area_t *area, lv_color_t *color_p) {
    // 通过 SPI 发送像素数据到 LT7580
    LCD_SetWindow(area->x1, area->y1, area->x2, area->y2);
    HAL_SPI_Transmit_DMA(&hspi1, (uint8_t *)color_p,
        (area->x2 - area->x1 + 1) * (area->y2 - area->y1 + 1) * 2);
    
    // 通知 LVGL 刷新完成
    lv_disp_flush_ready(drv);
}
```

**DMA 优势**：数据传输不占用 CPU，可并行处理下一帧绘制。

## 验证方法
观察屏幕刷新是否流畅，无闪烁。

## 来源
HelpPort - 嵌入式架构深度分析 (2026-07-22)

---

# IWDG 与 LCD 初始化时序

## 场景
LCD 初始化耗时 11s，IWDG 超时 2.5s，需防止复位。

## 现象/问题
LCD 初始化期间系统复位。

## 根因分析
IWDG reload=312, 预分频 256, LSI≈32kHz → 超时 2.5s。LCD 初始化全程无喂狗。

## 解决方案
```c
// lcd.c - LCD_Init 中插入喂狗点
int LCD_Init(void) {
    for (int i = 0; i < 15; i++) {
        HAL_IWDG_Refresh(&hiwdg);  // 上电沉降期间喂狗
        HAL_Delay(100);
    }
    
    LCD_Reset();
    HAL_IWDG_Refresh(&hiwdg);  // 复位后喂狗
    
    if (LCD_PLL_Initial() != 0) return -1;
    HAL_IWDG_Refresh(&hiwdg);  // PLL 初始化后喂狗
    
    if (LCD_SDRAM_initail() != 0) return -1;
    HAL_IWDG_Refresh(&hiwdg);  // SDRAM 初始化后喂狗
    
    return 0;
}
```

## 验证方法
断电 5s 后上电，系统应在 12s 内稳定点亮。

## 来源
HelpPort - 利用看门狗复位实现12秒稳定亮屏 (2026-07-08)

---

# 字库加载优化

## 场景
2.1MB SPI Flash 字库加载耗时 ~8s（2.4s ~ 11s 区间）。

## 现象/问题
启动时间过长。

## 根因分析
字库从 SPI Flash 读取到 SDRAM，数据量大，SPI 速率有限。

## 解决方案
**方案1：压缩字库**
- 使用 LVGL 内置字体子集化工具
- 只包含用到的字符

**方案2：异步加载**
```c
// 先显示界面，字库后台加载
void GUI_Task(void *argument) {
    lv_init();
    lv_port_disp_init();
    lv_port_indev_init();
    
    // 启动字库加载任务（低优先级）
    xTaskCreate(Font_LoadTask, "Font", 512, NULL, 1, NULL);
    
    for (;;) {
        lv_task_handler();
        osDelay(5);
    }
}
```

**方案3：预烧录**
- 将字库烧录到外部 SPI Flash 固定地址
- 启动时直接映射（XIP 模式，如果支持）

## 验证方法
对比优化前后启动时间。

## 来源
HelpPort - 拉取 HelpPort_NEW 最新代码 (2026-07-09)
