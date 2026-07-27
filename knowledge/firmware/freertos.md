# FreeRTOS 使用经验

---

# FreeRTOS 任务划分原则

## 场景
嵌入式系统需要多任务并行（GUI采集、通信、控制）。

## 现象
任务耦合严重，一个任务阻塞导致整个系统卡死。

## 根因
未按职责划分任务，或任务间同步不当。

## 解决方案
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

任务优先级建议：
- 最高：硬件中断服务（ADC/DMA 完成）
- 高：通信处理（Modbus 响应）
- 中：GUI 渲染（LVGL task handler 5ms）
- 低：数据采集、日志

## 验证
- 各任务独立运行，一个任务死循环不影响其他
- 系统总运行时间 > 72h 无异常

## 来源
HelpPort - 嵌入式架构深度分析 (2026-07-22)

---

# osDelay 阻塞导致竞态条件

## 场景
倒计时任务中用 `osDelay(1000)` 等待，导致状态切换时序错乱。

## 现象
倒计时结束后红灯+蜂鸣器不工作。

## 根因
```c
// HC_433.c:569 — 错误代码
else if (*start_tick - *end_tick == 1) {
    osDelay(1000);            // ← BUG! 阻塞1秒
    HC433.LED_Flag = 2;      // 红灯+蜂鸣器
    HC433.Counter_Start = 0;
}
// 问题: osDelay 期间 Counter_Run 继续把 countdown_time 从 1 减到 0
// competition() 仍拿着 LED_Flag=4 闪绿灯，醒来后时序错乱
```

## 解决方案
```c
// 修复: 先设标志再延迟
else if (*start_tick - *end_tick == 1) {
    HC433.LED_Flag = 2;      // 先设标志，competition() 立刻响应
    osDelay(1000);            // 红灯+蜂鸣器持续1秒
    HC433.Counter_Start = 0;  // 停止计时
}
```

## 验证
- 倒计时结束 → 红灯立即亮起 + 蜂鸣器响
- 无时序错乱

## 来源
WSC-16 - UART5 RS485 control logic (2026-06-23)

---

# FreeRTOS 临界区与中断优先级

## 场景
`taskENTER_CRITICAL` 导致高优先级 ISR 被阻塞。

## 现象
AD7606 ISR 打断 I2C 通信导致触摸异常。

## 根因
`taskENTER_CRITICAL` 屏蔽所有中断（包括 ISR 优先级 5），AD7606 FSMC 采样 ISR 被阻塞。

## 解决方案
```c
// 错误: 全局临界区
taskENTER_CRITICAL();
i2c_read(...);  // 期间 AD7606 ISR 无法响应
taskEXIT_CRITICAL();

// 正确: 仅用互斥锁保护共享资源
xSemaphoreTake(i2c_mutex, portMAX_DELAY);
i2c_read(...);
xSemaphoreGive(i2c_mutex);

// 或: 使用 BASEPRI 仅屏蔽特定优先级以下的中断
uint32_t pri = taskENTER_CRITICAL_FROM_ISR();
// ... 临界区 ...
taskEXIT_CRITICAL_FROM_ISR(pri);
```

## 验证
- AD7606 采样不丢点
- I2C 触摸通信正常

## 来源
HelpPort - Caveman mode: ultra (代码重构) (2026-07-04)

---

# LVGL 任务周期优化

## 场景
LVGL GUI 渲染占用过多 CPU。

## 现象
GUI 渲染导致其他任务（如 DAQ 输出）频率下降。

## 根因
`lv_task_handler()` 每 1ms 调用过于频繁。

## 解决方案
```c
// GUI_Task 周期从 1ms 改为 5ms
void GUI_Task(void *argument) {
    for (;;) {
        lv_task_handler();
        osDelay(5);  // 原来是 osDelay(1)
    }
}

// 触屏轮询从 10ms 改为 20ms
void Touch_Task(void *argument) {
    for (;;) {
        lv_indev_read_timer_cb(NULL);
        osDelay(20);
    }
}
```

## 验证
- GUI 刷新流畅（>20fps）
- DAQ 输出频率恢复到预期

## 来源
HelpPort - Caveman mode: ultra (代码重构) (2026-07-04)

---

# FreeRTOS 任务间通信

## 场景
多个任务需要共享数据（如倒计时时间、LED 状态）。

## 现象
数据竞争导致状态不一致。

## 根因
无保护的全局变量被多任务同时读写。

## 解决方案
```c
// 方法1: 消息队列
QueueHandle_t xLedQueue;
xLedQueue = xQueueCreate(10, sizeof(uint8_t));

// 发送
uint8_t flag = 2;
xQueueSend(xLedQueue, &flag, portMAX_DELAY);

// 接收
uint8_t received;
xQueueReceive(xLedQueue, &received, portMAX_DELAY);

// 方法2: 任务通知（轻量）
xTaskNotifyGive(xCompetitionTask);
ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

// 方法3: 互斥锁保护共享变量
SemaphoreHandle_t xControlMutex;
xSemaphoreTake(xControlMutex, portMAX_DELAY);
HC433.LED_Flag = new_value;
xSemaphoreGive(xControlMutex);
```

## 验证
- 多任务并发访问无数据损坏
- 无优先级反转问题

## 来源
WSC-16 - UART5 RS485 control logic (2026-06-23)

---

# FreeRTOS 栈水位检测

## 场景
任务运行一段时间后 HardFault，疑似栈溢出。

## 现象
系统随机崩溃，PC 指向非法地址。

## 根因
任务栈空间不足，局部变量或函数调用链过深。

## 解决方案
```c
// 开启栈溢出检测 (FreeRTOSConfig.h)
#define configCHECK_FOR_STACK_OVERFLOW  2

// 溢出回调
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
    // 记录溢出任务名
    printf("Stack overflow in task: %s\n", pcTaskName);
    __disable_irq();
    while (1);
}

// 运行时查栈水位
UBaseType_t watermark = uxTaskGetStackHighWaterMark(xTaskHandle);
printf("Task %s free stack: %u words\n", pcTaskName, watermark);
// watermark < 100 → 增加栈大小
```

## 验证
- 所有任务 watermark > 100 words
- 长时间运行无 HardFault

## 来源
HelpPort - 嵌入式架构深度分析 (2026-07-22)
