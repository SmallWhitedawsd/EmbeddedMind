# LVGL 架构分层设计

## 场景
STM32F407 + FreeRTOS + LVGL v8 + LT7580 LCD 控制器项目重构时，需要解耦五层架构。

## 现象/问题
业务代码与硬件驱动耦合严重，换 MCU 需大量修改。

## 根因分析
原始代码无分层，LCD/触摸/业务逻辑混在同一个文件。

## 解决方案
五层架构：

```
┌──────────────────────────────────────────┐
│  app/tasks/          ← 业务任务           │
│  app/services/       ← 服务层             │
│  app/drivers/        ← 驱动抽象层         │
│  app/hal/            ← HAL 适配层         │
│  app/config/         ← 编译期配置         │
├──────────────────────────────────────────┤
│  BSP (bsp_ad7606/lcd/ctp) ← 板级支持      │
├──────────────────────────────────────────┤
│  STM32 HAL + FreeRTOS + LVGL             │
└──────────────────────────────────────────┘
```

BSP 抽象方式：编译期宏开关 + 初始化函数暴露（`bsp_InitAD7606()`, `LCD_Init()`, `CTP_Init()`）。

缺点：无运行时多态，硬件变更需重新编译。

## 验证方法
换 MCU 时只需修改 `app/hal/` 和 `BSP/` 层，业务代码零改动。

## 来源
HelpPort - 嵌入式架构深度分析 (2026-07-22)

---

# LVGL 与 LT7580 职责边界

## 场景
评估是否使用 LT7580 GPU 加速 LVGL 渲染。

## 现象/问题
LVGL 绘制效率低，CPU 占用高。

## 根因分析
LVGL v8.4 渲染模型与 LT7580 硬件能力根本性错配。

## 解决方案
**LT7580 硬件完成**：帧缓冲管理、BTE 页面切换、图层合成、SPI 通信。

**CPU 软件完成**：所有绘制（矩形填充、圆弧、图像解码、文本渲染）。

**未接入 GPU 原因**：LT7580 的 BTE/图层操作无法对接 LVGL 的 flush 模型。

## 验证方法
1024×600 RGB565 = 1.2MB 帧缓冲，STM32F407VET6 仅 112KB SRAM，必须外置 SDRAM。

## 来源
HelpPort - 嵌入式架构深度分析 (2026-07-22)

---

# GUI 任务与 RTOS 调度

## 场景
FreeRTOS 多任务系统中集成 LVGL。

## 现象/问题
触屏卡顿，GUI 响应延迟。

## 根因分析
`lv_task_handler()` 每 1ms 调用过于频繁，且与 AD7606 ISR 总线竞争。

## 解决方案
```c
// freertos.c - GUI_Task
void GUI_Task(void *argument) {
    for (;;) {
        lv_task_handler();
        osDelay(5);  // 从 1ms 改为 5ms
    }
}
```

触屏轮询周期从 10ms 改为 20ms。

## 验证方法
触摸响应流畅，无掉帧。

## 来源
HelpPort - Caveman mode: ultra (代码重构) (2026-07-04)

---

# 中断与 GUI 任务协作

## 场景
AD7606 FSMC 200kHz 采样 + DMA 与 LVGL 触摸 I2C 通信冲突。

## 现象/问题
首次上电触屏卡顿，多次上电后流畅。

## 根因分析
- AD7606 ISR 优先级 5 可打断软件 I2C（GT9147）
- `taskENTER_CRITICAL` 保护下仍被 ISR 打断
- SPI DMA/FSMC 总线竞争导致触摸异常

## 解决方案
```c
// lv_port_indev.c - 移除 taskENTER_CRITICAL
void touchpad_read(lv_indev_drv_t *drv, lv_indev_data_t *data) {
    // 不再包裹 taskENTER_CRITICAL
    GT9147_ReadCoord(&x, &y);
    data->point.x = x;
    data->point.y = y;
    data->state = pressed ? LV_INDEV_STATE_PR : LV_INDEV_STATE_REL;
}
```

## 验证方法
连续上电 10 次，触摸均正常响应。

## 来源
HelpPort - Caveman mode: ultra (代码重构) (2026-07-04)
