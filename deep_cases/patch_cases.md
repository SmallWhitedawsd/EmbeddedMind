# 二炼知识库 - Patch 案例集

**总计**: 155 个高质量代码修改案例  
**来源**: opencode.db (256 patch events, 321 patch parts)  
**提取时间**: 2026-07-27

---

# PCSE_001: freertos.c - 添加RTOS任务实现并发处理

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/Core/Src/freertos.c

## 修改原因
Now implementing the three fixes. First, priority changes in `freertos.c`:

## 修改前代码
```c
const osThreadAttr_t DAQ_Task_attributes = {
  .name = "DAQ_Task",
  .stack_size = 512 * 2,
  .priority = (osPriority_t) osPriorityHigh,   // �� GUI ͬ�����������ݿ� ISR ���ף����������ܶ��� UI
};
```

## 修改后代码
```c
const osThreadAttr_t DAQ_Task_attributes = {
  .name = "DAQ_Task",
  .stack_size = 512 * 2,
  .priority = (osPriority_t) osPriorityAboveNormal,   // �������� GUI������ UI �}��
};
```

## 验证方式
```
git diff HEAD~1 -- HelpPort/AD7606/bsp_ad7606.c HelpPort/Core/Src/freertos.c HelpPort/LVGL/examples/porting/lv_port_disp.c
```

## 经验规则
添加RTOS任务实现并发处理

---

# PCSE_002: freertos.c - 添加RTOS任务实现并发处理

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/Core/Src/freertos.c

## 修改原因
Now implementing the three fixes. First, priority changes in `freertos.c`:

## 修改前代码
```c
const osThreadAttr_t GUI_Task_attributes = {
  .name = "GUI_Task",
  .stack_size = 1024 * 4,        
  .priority = (osPriority_t) osPriorityHigh,   // GUI ������/����������ʱ����
};
```

## 修改后代码
```c
const osThreadAttr_t GUI_Task_attributes = {
  .name = "GUI_Task",
  .stack_size = 1024 * 4,        
  .priority = (osPriority_t) osPriorityHigh7,  // GUI ������/GUI ���ȼ� > DAQ
};
```

## 验证方式
```
git diff HEAD~1 -- HelpPort/AD7606/bsp_ad7606.c HelpPort/Core/Src/freertos.c HelpPort/LVGL/examples/porting/lv_port_disp.c
```

## 经验规则
添加RTOS任务实现并发处理

---

# PCSE_003: freertos.c - 添加缓冲区处理数据流

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/Core/Src/freertos.c

## 修改原因
Now implementing the three fixes. First, priority changes in `freertos.c`:

## 修改前代码
```c
                ch_sum[3] += g_PingPong_Buf[ready_buf_idx][pt][3];
                ch_sum[4] += g_PingPong_Buf[ready_buf_idx][pt][4];
                ch_sum[5] += g_PingPong_Buf[ready_buf_idx][pt][5];
                ch_sum[6] += g_PingPong_Buf[ready_buf_idx][pt][6];
                ch_sum[7] += g_PingPong_Buf[ready_buf_idx][pt][7];
            }
```

## 修改后代码
```c
                ch_sum[3] += g_PingPong_Buf[ready_buf_idx][pt][3];
                ch_sum[4] += g_PingPong_Buf[ready_buf_idx][pt][4];
                ch_sum[5] += g_PingPong_Buf[ready_buf_idx][pt][5];
                ch_sum[6] += g_PingPong_Buf[ready_buf_idx][pt][6];
                ch_sum[7] += g_PingPong_Buf[ready_buf_idx][pt][7];
                if ((pt & 0xFF) == 0xFF) { osThreadYield(); }
            }
```

## 验证方式
```
git diff HEAD~1 -- HelpPort/AD7606/bsp_ad7606.c HelpPort/Core/Src/freertos.c HelpPort/LVGL/examples/porting/lv_port_disp.c
```

## 经验规则
添加缓冲区处理数据流

---

# PCSE_004: ctp.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/Core/Src/freertos.c

## 修改原因
Now add `osThreadYield()` in DAQ_Task's heavy summing loop. Let me read the current state:

## 修改前代码
```c
void CT_Delay(void)
{
    volatile uint32_t i = 1000;
    while(i--) __NOP();
}
```

## 修改后代码
```c
void CT_Delay(void)
{
    volatile uint32_t i = 300;
    while(i--) __NOP();
}
```

## 验证方式
```
git diff HEAD~1 -- HelpPort/AD7606/bsp_ad7606.c HelpPort/Core/Src/freertos.c HelpPort/LVGL/examples/porting/lv_port_disp.c
```

## 经验规则
添加延时确保时序正确

---

# PCSE_005: lv_port_indev.c - 使用DMA减轻CPU负担

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/LVGL/examples/porting/lv_port_indev.c

## 修改原因
优先级改不动ISR抢占。根因是EXTI3 ISR（优先级5，200kHz）直接打断GPIO bit-bang I2C，与任务优先级无关。需要硬件级解耦：触摸读期间临时屏蔽EXTI3中断。

## 修改前代码
```c
/*Return true is the touchpad is pressed*/
static bool touchpad_is_pressed(void)
{
    // �Ƴ� taskENTER_CRITICAL����ʱ����жϵ��� FreeRTOS Tick ��ʧ��
    Readtouch_CTP();

    if(Touch_State == TOUCH_DOWN)
    {
        return true;
    }
```

## 修改后代码
```c
/*Return true is the touchpad is pressed*/
static bool touchpad_is_pressed(void)
{
    /* Atomically clear EXTI3 pending and mask it so the 200kHz AD7606 ISR
       cannot preempt the GPIO bit-bang I2C during touch read (~300-700us).
       Only EXTI3 is masked; SysTick/DMA/UART continue to run. */
    __disable_irq();
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_3);
    HAL_NVIC_DisableIRQ(EXTI3_IRQn);
    __enable_irq();

    Readtouch_CTP();

    HAL_NVIC_EnableIRQ(EXTI3_IRQn);

    if(Touch_State == TOUCH_DOWN)
    {
        return true;
    }
```

## 验证方式
```
git diff --stat
```

## 经验规则
使用DMA减轻CPU负担

---

# PCSE_006: ctp.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/LVGL/examples/porting/lv_port_indev.c

## 修改原因
优先级和ISR屏蔽没编译进去。直接换成 FreeRTOS `taskENTER_CRITICAL()` 全局屏蔽所有>=5的中断（含EXTI3、DMA、SysTick），比单屏蔽EXTI3更彻底。同时降I2C延迟、降消抖、提轮询率。

## 修改前代码
```c
void CT_Delay(void)
{
    volatile uint32_t i = 300;
    while(i--) __NOP();
}
```

## 修改后代码
```c
void CT_Delay(void)
{
    volatile uint32_t i = 200;
    while(i--) __NOP();
}
```

## 验证方式
```
git diff --stat
```

## 经验规则
添加延时确保时序正确

---

# PCSE_007: lv_port_indev.c - 添加临界区保护共享资源

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/LVGL/examples/porting/lv_port_indev.c

## 修改原因
优先级和ISR屏蔽没编译进去。直接换成 FreeRTOS `taskENTER_CRITICAL()` 全局屏蔽所有>=5的中断（含EXTI3、DMA、SysTick），比单屏蔽EXTI3更彻底。同时降I2C延迟、降消抖、提轮询率。

## 修改前代码
```c
/*Return true is the touchpad is pressed*/
static bool touchpad_is_pressed(void)
{
    /* Atomically clear EXTI3 pending and mask it so the 200kHz AD7606 ISR
       cannot preempt the GPIO bit-bang I2C during touch read (~300-700us).
       Only EXTI3 is masked; SysTick/DMA/UART continue to run. */
    __disable_irq();
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_3);
    HAL_NVIC_DisableIRQ(EXTI3_IRQn);
    __enable_irq();

    Readtouch_CTP();

    HAL_NVIC_EnableIRQ(EXTI3_IRQn);

    if(Touch_State == TOUCH_DOWN)
    {
        return true;
    }
```

## 修改后代码
```c
/*Return true is the touchpad is pressed*/
static bool touchpad_is_pressed(void)
{
    /* Mask all interrupts pri >= 5 (EXTI3 200kHz AD7606 ISR, DMA, etc.)
       during the GPIO bit-bang I2C touch read. FreeRTOS-aware:
       sets BASEPRI, SysTick paused but <1ms dropout is acceptable
       since touch read takes ~300-500us total. */
    taskENTER_CRITICAL();

    Readtouch_CTP();

    taskEXIT_CRITICAL();

    if(Touch_State == TOUCH_DOWN)
    {
        return true;
    }
```

## 验证方式
```
git diff --stat
```

## 经验规则
添加临界区保护共享资源

---

# PCSE_008: lv_port_indev.c - 添加状态机管理复杂逻辑

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/LVGL/examples/porting/lv_port_indev.c

## 修改原因
优先级和ISR屏蔽没编译进去。直接换成 FreeRTOS `taskENTER_CRITICAL()` 全局屏蔽所有>=5的中断（含EXTI3、DMA、SysTick），比单屏蔽EXTI3更彻底。同时降I2C延迟、降消抖、提轮询率。

## 修改前代码
```c
    if (hw_state != reported_state) {
        debounce_cnt++;
        if (debounce_cnt >= 3) {
```

## 修改后代码
```c
    if (hw_state != reported_state) {
        debounce_cnt++;
        if (debounce_cnt >= 1) {
```

## 验证方式
```
git diff --stat
```

## 经验规则
添加状态机管理复杂逻辑

---

# PCSE_009: lv_port_indev.c - 添加定时器实现精确时序控制

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/LVGL/examples/porting/lv_port_indev.c

## 修改原因
Now reduce I2C delay, touch debounce, and polling period:

## 修改前代码
```c
    indev_touchpad = lv_indev_drv_register(&indev_drv);
```

## 修改后代码
```c
    indev_touchpad = lv_indev_drv_register(&indev_drv);
    lv_timer_set_period(lv_indev_get_read_timer(indev_touchpad), 10);  /* poll every 10ms */
```

## 验证方式
```
git diff --stat
```

## 经验规则
添加定时器实现精确时序控制

---

# PCSE_010: lv_port_indev.c - 添加定时器实现精确时序控制

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/LVGL/examples/porting/lv_port_indev.c

## 修改原因
编译错误。`lv_indev_get_read_timer` 接受 `lv_disp_t*` 不是 `lv_indev_t*`。修复：

## 修改前代码
```c
    indev_touchpad = lv_indev_drv_register(&indev_drv);
    lv_timer_set_period(lv_indev_get_read_timer(indev_touchpad), 10);  /* poll every 10ms */
```

## 修改后代码
```c
    indev_touchpad = lv_indev_drv_register(&indev_drv);
    lv_timer_set_period(lv_indev_get_read_timer(lv_disp_get_default()), 10);  /* poll every 10ms */
```

## 验证方式
```
git diff --stat
```

## 经验规则
添加定时器实现精确时序控制

---

# PCSE_011: ui_safehouse.c - 添加定时器实现精确时序控制

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/LVGL/examples/porting/lv_port_indev.c

## 修改原因
编译错误。`lv_indev_get_read_timer` 接受 `lv_disp_t*` 不是 `lv_indev_t*`。修复：

## 修改前代码
```c
    if (kp_timer) lv_timer_set_period(kp_timer, 15);
```

## 修改后代码
```c
    if (kp_timer) lv_timer_set_period(kp_timer, 10);
```

## 验证方式
```
git diff --stat
```

## 经验规则
添加定时器实现精确时序控制

---

# PCSE_012: freertos.c - 添加计数器跟踪操作次数

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/Core/Src/freertos.c

## 修改原因
好思路。核心问题不是ISR抢占，是100ms UI刷新+LVGL渲染开销跟200kHz ISR叠加。降刷新率减渲染即可。

## 修改前代码
```c
		// ========================================================
    // 1. ÿ 100ms ����һ�� UI ˢ�»��� (ͬ�������ֵ�����Ļ)
    // ========================================================
    if(osKernelGetTickCount() - ulLastDispTick >= 100)
    {
        ulLastDispTick = osKernelGetTickCount();
        ui_tick();  // ���������� ui.c �е� tick������������ǰҳ��� tick
				ui_safehouse_tick();
    }
```

## 修改后代码
```c
    // ========================================================
    // 1. 每 UI_DATA_REFRESH_MS 更新一次模拟量显示 (水龙头: 控制开销)
    // ========================================================
    if(osKernelGetTickCount() - ulLastDispTick >= UI_DATA_REFRESH_MS)
    {
        ulLastDispTick = osKernelGetTickCount();
        ui_tick();
        ui_safehouse_tick();
    }
```

## 验证方式
```
git diff --stat
```

## 经验规则
添加计数器跟踪操作次数

---

# PCSE_013: freertos.c - 使用宏定义消除魔术数字

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/Core/Src/freertos.c

## 修改原因
好思路。核心问题不是ISR抢占，是100ms UI刷新+LVGL渲染开销跟200kHz ISR叠加。降刷新率减渲染即可。

## 修改前代码
```c
#define SAMPLE_FREQ 200000  // ���������� (200kHz)����������ϵͳһ��
#define DAQ_INTERNAL_ADC_UPDATE_MS 100U
```

## 修改后代码
```c
#define SAMPLE_FREQ 200000  // ���������� (200kHz)����������ϵͳһ��
#define DAQ_INTERNAL_ADC_UPDATE_MS 100U
#define UI_DATA_REFRESH_MS     500U  /* 水龙头：面板模拟量刷新周期，越大触摸越流畅 */
```

## 验证方式
```
git diff --stat
```

## 经验规则
使用宏定义消除魔术数字

---

# PCSE_014: freertos.c - 修复代码逻辑错误

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/Core/Src/freertos.c

## 修改原因
好思路。核心问题不是ISR抢占，是100ms UI刷新+LVGL渲染开销跟200kHz ISR叠加。降刷新率减渲染即可。

## 修改前代码
```c
  // ��¼ʱ��������� 100ms ˢ��һ������
```

## 修改后代码
```c
  // 记录时间戳，每 UI_DATA_REFRESH_MS 刷新一次数据
```

## 验证方式
```
git diff --stat
```

## 经验规则
修复代码逻辑错误

---

# PCSE_015: ctp.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/2/HelpPort/HelpPort/LCD/ctp.c

## 修改原因
先做阶段三:触屏+AD7606修复。写入修改。

## 修改前代码
```c
void CT_Delay(void)
{
    volatile uint32_t i = 200;
    while(i--) __NOP();
}
```

## 修改后代码
```c
void CT_Delay(void)
{
    /* ~5us per call @168MHz — yields ~100kHz I2C (within GT9147 400kHz max) */
    volatile uint32_t i = 800;
    while(i--) __NOP();
}
```

## 验证方式
```
Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Src\user_ui_app.c"; Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Inc\user_ui_app.h"
```

## 经验规则
添加延时确保时序正确

---

# PCSE_016: lv_port_indev.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/2/HelpPort/HelpPort/LCD/ctp.c

## 修改原因
先做阶段三:触屏+AD7606修复。写入修改。

## 修改前代码
```c
/*Return true is the touchpad is pressed*/
static bool touchpad_is_pressed(void)
{
    /* Mask all interrupts pri >= 5 (EXTI3 200kHz AD7606 ISR, DMA, etc.)
       during the GPIO bit-bang I2C touch read. FreeRTOS-aware:
       sets BASEPRI, SysTick paused but <1ms dropout is acceptable
       since touch read takes ~300-500us total. */
    taskENTER_CRITICAL();

    Readtouch_CTP();

    taskEXIT_CRITICAL();

    if(Touch_State == TOUCH_DOWN)
    {
        return true;
    }

    if(Touch_State == TOUCH_UP)
    {
        Touch_State = TOUCH_NC; 
    }

    return false;
}
```

## 修改后代码
```c
/*
 * Touch I2C read — use BASEPRI mask that keeps AD7606 ISR (NVIC pri 5)
 * enabled during the bit-bang I2C transfer.
 *
 * Previous code used taskENTER_CRITICAL() which masked pri >= 5,
 * blocking the 200kHz AD7606 BUSY# ISR and causing sample dropouts.
 * Now only mask pri >= 6 (DMA, UART, etc.) so AD7606 ISR can fire.
 *
 * The I2C is ~100kHz now (CT_Delay=800 NOPs ≈ 5us per bit).
 * AD7606 ISR latency ~450ns is ~4.5% of bit period — safe for I2C.
 */
static bool touchpad_is_pressed(void)
{
    __set_BASEPRI(0x60);   /* mask NVIC pri >= 6; pri 5 (AD7606) remains unmasked */
    __DSB();
    __ISB();

    Readtouch_CTP();

    __set_BASEPRI(0);      /* unmask all */
    __DSB();
    __ISB();

    if(Touch_State == TOUCH_DOWN)
    {
        return true;
    }

    if(Touch_State == TOUCH_UP)
    {
// ... (truncated)
```

## 验证方式
```
Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Src\user_ui_app.c"; Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Inc\user_ui_app.h"
```

## 经验规则
添加延时确保时序正确

---

# PCSE_017: lv_port_indev.c - 修复代码逻辑错误

## 项目
HelpPort

## 修改文件
- D:/reasonix/2/HelpPort/HelpPort/LCD/ctp.c

## 修改原因
先做阶段三:触屏+AD7606修复。写入修改。

## 修改前代码
```c
#include "lv_port_indev.h"
#include "lvgl.h"
#include "CTP.h" // ��������ͷ�ļ�
#include "gui.h" //����
#include "FreeRTOS.h"
#include "task.h"
```

## 修改后代码
```c
#include "lv_port_indev.h"
#include "lvgl.h"
#include "CTP.h"
#include "gui.h"
#include "main.h"   /* for __set_BASEPRI via CMSIS */
```

## 验证方式
```
Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Src\user_ui_app.c"; Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Inc\user_ui_app.h"
```

## 经验规则
修复代码逻辑错误

---

# PCSE_018: lv_port_indev.c - 添加I2C通信实现传感器接口

## 项目
HelpPort

## 修改文件
- D:/reasonix/2/HelpPort/HelpPort/LCD/ctp.c

## 修改原因
先做阶段三:触屏+AD7606修复。写入修改。

## 修改前代码
```c
    indev_touchpad = lv_indev_drv_register(&indev_drv);
    lv_timer_set_period(lv_indev_get_read_timer(lv_disp_get_default()), 10);  /* poll every 10ms */
```

## 修改后代码
```c
    indev_touchpad = lv_indev_drv_register(&indev_drv);
    lv_timer_set_period(lv_indev_get_read_timer(lv_disp_get_default()), 25);  /* poll every 25ms — reduce I2C traffic */
```

## 验证方式
```
Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Src\user_ui_app.c"; Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Inc\user_ui_app.h"
```

## 经验规则
添加I2C通信实现传感器接口

---

# PCSE_019: freertos.c - 使用宏定义消除魔术数字

## 项目
HelpPort

## 修改文件
- D:/reasonix/2/HelpPort/HelpPort/LVGL/examples/porting/lv_port_indev.c

## 修改原因
触屏轮询周期 10ms→20ms,减少 I2C 调用频率。

## 修改前代码
```c
#define SAMPLE_FREQ 200000  // ���������� (200kHz)����������ϵͳһ��
#define DAQ_INTERNAL_ADC_UPDATE_MS 100U
#define UI_DATA_REFRESH_MS     900U  /* 水龙头：面板模拟量9倍降频，越大触摸越流畅 */
```

## 修改后代码
```c
#define SAMPLE_FREQ 200000  // 采样率 (200kHz)��与原有系统一致
#define DAQ_INTERNAL_ADC_UPDATE_MS 100U
#define UI_DATA_REFRESH_MS     300U  /* UI data refresh: every 300ms (down from 900ms; safe now that render runs @200Hz not 1kHz) */
```

## 验证方式
```
Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Src\user_ui_app.c"; Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Inc\user_ui_app.h"
```

## 经验规则
使用宏定义消除魔术数字

---

# PCSE_020: freertos.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/2/HelpPort/HelpPort/Core/Src/freertos.c

## 修改原因
freertos.c GUI_Task — `lv_task_handler` 从每1ms改为每5ms,减少渲染压力。

## 修改前代码
```c
    for(;;)
    {
		
//// =============����������ʾ================================
//		  if(osKernelGetTickCount() - ulLastDispTick >= 100)
//      {
//          ulLastDispTick = osKernelGetTickCount();
//          GUI_UI_Update(); 
//      }
////========================================================
		
		
    // ========================================================
    // 1. 每 UI_DATA_REFRESH_MS 更新一次模拟量显示 (水龙头: 控制开销)
    // ========================================================
    if(osKernelGetTickCount() - ulLastDispTick >= UI_DATA_REFRESH_MS)
    {
        ulLastDispTick = osKernelGetTickCount();
        ui_tick();
        ui_safehouse_tick();
    }
		
		// ========================================================
    // 2. LVGL ������ת��ʱ������ FreeRTOS ��ʵ Tick��
    // ========================================================
    static uint32_t last_tick = 0;
    uint32_t now = osKernelGetTickCount();
    uint32_t elapsed = (last_tick == 0) ? 5 : (now - last_tick);
    la
```

## 修改后代码
```c
    for(;;)
    {
        // ========================================================
        // 1. Update analog display every UI_DATA_REFRESH_MS
        // ========================================================
        if(osKernelGetTickCount() - ulLastDispTick >= UI_DATA_REFRESH_MS)
        {
            ulLastDispTick = osKernelGetTickCount();
            ui_tick();
            ui_safehouse_tick();
        }

        // ========================================================
        // 2. LVGL tick + render @ ~200Hz (was 1ms=~1000Hz; now 5ms)
        //    Touch I2C uses BASEPRI>=6 mask → AD7606 ISR never blocked
        // ========================================================
        static uint32_t last_tick = 0;
        uint32_t now = osKernelGetTickCount();
        uint32_t elapsed = (last_tick == 0) ? 5 : (now - last_tick);
        last_tick = now;
        lv_tick_inc(elapsed);
        lv_task_handler();

        osDelay(5);
    }
```

## 验证方式
```
Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Src\user_ui_app.c"; Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Inc\user_ui_app.h"
```

## 经验规则
添加延时确保时序正确

---

# PCSE_021: lcd.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/2/HelpPort/HelpPort/Core/Src/freertos.c

## 修改原因
GUI_Task loop — lv_task_handler from 1ms to 5ms.

## 修改前代码
```c
void LCD_Reset(void)
{
   LCD_RST0();
	delay_ms(500);
  LCD_RST1();
	delay_ms(1000);
}
```

## 修改后代码
```c
void LCD_Reset(void)
{
   LCD_RST0();
	delay_ms(10);      /* was 500ms; LT7580 needs ~1ms minimum */
  LCD_RST1();
	delay_ms(50);      /* was 1000ms; PLL lock time ~10ms, margin = 5x */
}
```

## 验证方式
```
Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Src\user_ui_app.c"; Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Inc\user_ui_app.h"
```

## 经验规则
添加延时确保时序正确

---

# PCSE_022: eth_at.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/2/HelpPort/HelpPort/Core/Src/eth_at.c

## 修改原因
ETH AT 查询异步化 — 移出 main.c,放入 RTOS 任务。同时减短 NE2 等待。

## 修改前代码
```c
    HAL_Delay(200);
    HAL_GPIO_WritePin(ETH_UART_RST_GPIO_Port, ETH_UART_RST_Pin, GPIO_PIN_SET);

    // ---------- 2. 等待 NE2 启动完成 ----------
    HAL_Delay(2000);
```

## 修改后代码
```c
    HAL_Delay(100);
    HAL_GPIO_WritePin(ETH_UART_RST_GPIO_Port, ETH_UART_RST_Pin, GPIO_PIN_SET);

    // ---------- 2. Wait for NE2 boot (was 2000ms; NE2 boots ~500ms, 800ms safe) ----------
    HAL_Delay(800);
```

## 验证方式
```
Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Src\user_ui_app.c"; Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Inc\user_ui_app.h"
```

## 经验规则
添加延时确保时序正确

---

# PCSE_023: freertos.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/2/HelpPort/HelpPort/Core/Src/eth_at.c

## 修改原因
ETH AT 查询异步化 — 移出 main.c,放入 RTOS 任务。同时减短 NE2 等待。

## 修改前代码
```c
  lv_task_handler();
  vTaskDelay(pdMS_TO_TICKS(5));    // �� SPI DMA д��
  // Display_ON ���� main.c �е��ã��˴�������������
  ui_safehouse_prerender();   /* prebuild page snapshots while backlight off (hidden ~3.7s) */
  Set_Backlight(100);
```

## 修改后代码
```c
  lv_task_handler();
  vTaskDelay(pdMS_TO_TICKS(5));    // wait for SPI DMA write
  
  // Lite pre-render: only snapshot the current (AIAO) page; DIDO is rendered
  // lazily on first switch. Backlight on immediately.
  ui_safehouse_prerender_light();
  Set_Backlight(100);
```

## 验证方式
```
Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Src\user_ui_app.c"; Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Inc\user_ui_app.h"
```

## 经验规则
添加延时确保时序正确

---

# PCSE_024: ui_safehouse.c - 添加返回值处理

## 项目
HelpPort

## 修改文件
- D:/reasonix/2/HelpPort/HelpPort/Core/Src/eth_at.c

## 修改原因
ETH AT 查询异步化 — 移出 main.c,放入 RTOS 任务。同时减短 NE2 等待。

## 修改前代码
```c
/* One-time boot pre-render: build both page snapshots so even the first
   switch to DIDO is instant. Call once after the initial AIAO render and
   before the backlight is turned on (hidden behind the black screen). */
void ui_safehouse_prerender(void)
```

## 修改后代码
```c
/* Boot pre-render (light): snapshot only the current AIAO page so backlight
   turns on immediately. DIDO is rendered lazily on first switch (~3.7s). */
void ui_safehouse_prerender_light(void)
{
    if (!objects.aiao) return;

    int i;
    for (i = 0; i < 4; i++) { lv_tick_inc(30); lv_task_handler(); }
    while (spi_dma_is_transmitting);
    LCD_BTE_Copy((unsigned long)Layer0, Layer1, (unsigned short)lcdx, (unsigned short)lcdy);
    s_snap_valid[0] = 1;
    s_cur_page = 0;
}

/* Full dual-page pre-render. Not called at boot to speed up backlight-on time.
   Available for explicit use if needed. */
void ui_safehouse_prerender(void)
```

## 验证方式
```
Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Src\user_ui_app.c"; Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Inc\user_ui_app.h"
```

## 经验规则
添加返回值处理

---

# PCSE_025: ui_safehouse.h - 扩展功能实现

## 项目
HelpPort

## 修改文件
- D:/reasonix/2/HelpPort/HelpPort/Core/Src/freertos.c

## 修改原因
简化预渲染 — 只渲染当前页,亮屏后才后台完成 DIDO。

## 修改前代码
```c
// boot one-shot: prebuild AIAO/DIDO snapshots so the first switch is instant
void ui_safehouse_prerender(void);
```

## 修改后代码
```c
// boot one-shot: snapshot current page only; DIDO rendered lazily on first switch
void ui_safehouse_prerender_light(void);

// full dual-page pre-render (use only when timing allows)
void ui_safehouse_prerender(void);
```

## 验证方式
```
Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Src\user_ui_app.c"; Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Inc\user_ui_app.h"
```

## 经验规则
扩展功能实现

---

# PCSE_026: main.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/2/HelpPort/HelpPort/Core/Inc/ui_safehouse.h

## 修改原因
简化预渲染 — 只渲染当前页,亮屏后才后台完成 DIDO。

## 修改前代码
```c
  /* 6. ��ʾ�봥������ײ��ϵ��ʼ�� */
  LCD_Init();
  Set_Color(color_black, color_white); // Ĭ�Ϻڵװ���
  GUI_CleanScreen();                   // �ϵ�����Ϊ��ɫ
  Display_ON();                        // ��������������ETH��ѯ�ڼ��û��ɼ��ڵ�

  /* 6.5. ��ѯ NE2 ����ģ�飨LCD �������������׵ȣ�*/
  ETH_AT_QueryNetworkInfo();

  CTP_Init();                          // ���ݴ���оƬ��ʼ��
```

## 修改后代码
```c
  /* 6. Display and touch hardware init */
  LCD_Init();
  Set_Color(color_black, color_white);
  GUI_CleanScreen();

  CTP_Init();                          /* touch init → ready before screen on */

  Display_ON();                        /* screen visible */

  /* 6.5. Query NE2 ETH module (reduced delay: 100ms+800ms) */
  ETH_AT_QueryNetworkInfo();
```

## 验证方式
```
Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Src\user_ui_app.c"; Remove-Item -LiteralPath "D:\reasonix\2\HelpPort\HelpPort\Core\Inc\user_ui_app.h"
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_027: freertos.c - 修复代码逻辑错误

## 项目
HelpPort

## 修改文件
- D:/reasonix/2/HelpPort/HelpPort/app/tasks/task_daq.c

## 修改原因
创建 SPI HAL + 项目配置 + 拆分 freertos.c 为独立任务文件。

## 修改前代码
```c
    .priority   = (osPriority_t)osPriorityNormal3,
```

## 修改后代码
```c
    .priority   = (osPriority_t)osPriorityNormal,
```

## 验证方式
```
New-Item -ItemType Directory -Force -Path "D:\reasonix\2\HelpPort\HelpPort\app\hal"; New-Item -ItemType Directory -Force -Path "D:\reasonix\2\HelpPort\HelpPort\app\config"; New-Item -ItemType Director
```

## 经验规则
修复代码逻辑错误

---

# PCSE_028: freertos.c - 添加串口通信实现数据交换

## 项目
HelpPort

## 修改文件
- D:/reasonix/2/HelpPort/HelpPort/app/tasks/task_daq.h

## 修改原因
创建 SPI HAL + 项目配置 + 拆分 freertos.c 为独立任务文件。

## 修改前代码
```c
    .priority   = (osPriority_t)osPriorityNormal,
```

## 修改后代码
```c
    .priority   = (osPriority_t)osPriorityAboveNormal,    /* raised from Normal to prevent UART data loss */
```

## 验证方式
```
New-Item -ItemType Directory -Force -Path "D:\reasonix\2\HelpPort\HelpPort\app\hal"; New-Item -ItemType Directory -Force -Path "D:\reasonix\2\HelpPort\HelpPort\app\config"; New-Item -ItemType Director
```

## 经验规则
添加串口通信实现数据交换

---

# PCSE_029: ctp.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/Core/Src/freertos.c

## 修改原因
远程有 2 个新提交，仅改了 1 个文件。合并后推送：

## 修改前代码
```c
void CT_Delay(void)
{
    volatile uint32_t i = 200;
    while(i--) __NOP();
}
```

## 修改后代码
```c
void CT_Delay(void)
{
    volatile uint32_t i = 30;
    while(i--) __NOP();
}
```

## 验证方式
```
git pull origin main --no-rebase
```

## 经验规则
添加延时确保时序正确

---

# PCSE_030: ctp.c - 修复代码逻辑错误

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/Core/Src/freertos.c

## 修改原因
远程有 2 个新提交，仅改了 1 个文件。合并后推送：

## 修改前代码
```c
        if(ucErrTime++ > 20000)
```

## 修改后代码
```c
        if(ucErrTime++ > 5000)
```

## 验证方式
```
git pull origin main --no-rebase
```

## 经验规则
修复代码逻辑错误

---

# PCSE_031: ctp.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/1/HelpPort/Core/Src/freertos.c

## 修改原因
远程有 2 个新提交，仅改了 1 个文件。合并后推送：

## 修改前代码
```c
        CT_IIC_SCL1;
        receive <<= 1;
        if(CT_READ_SDA) receive++;
        CT_Delay();
```

## 修改后代码
```c
        CT_IIC_SCL1;
        CT_Delay();
        receive <<= 1;
        if(CT_READ_SDA) receive++;
```

## 验证方式
```
git pull origin main --no-rebase
```

## 经验规则
添加延时确保时序正确

---

# PCSE_032: lcd.h - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.h

## 修改原因
Now implementing. First the header prototypes:

## 修改前代码
```c
//Set PLL
void LCD_PLL_Initial(void); 
//Set SDRAM 
void LCD_SDRAM_initail(unsigned char mclk);
```

## 修改后代码
```c
//Set PLL
int LCD_PLL_Initial(void); 
//Set SDRAM 
int LCD_SDRAM_initail(unsigned char mclk);
```

## 验证方式
```
Select-String -Path "D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.h" -Pattern "Check_SDRAM_Ready|LCD_SDRAM_initail|LCD_PLL_Initial|delay_ms|delay_us" | Select-Object LineNumber, Line
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_033: lcd.h - 添加检查确保状态正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.h

## 修改原因
Now implementing. First the header prototypes:

## 修改前代码
```c
void Check_SDRAM_Ready(void);
```

## 修改后代码
```c
int Check_SDRAM_Ready(void);
```

## 验证方式
```
Select-String -Path "D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.h" -Pattern "Check_SDRAM_Ready|LCD_SDRAM_initail|LCD_PLL_Initial|delay_ms|delay_us" | Select-Object LineNumber, Line
```

## 经验规则
添加检查确保状态正确

---

# PCSE_034: lcd.c - 添加返回值处理

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.h

## 修改原因
Now implementing. First the header prototypes:

## 修改前代码
```c
void Check_SDRAM_Ready(void)
{
/*	0: SDRAM is not ready for access
	1: SDRAM is ready for access		*/	
	unsigned char temp; 	
	do
	{	
		temp=LCD_StatusRead();   
	}while( (temp&0x04) == 0x00 );
}
```

## 修改后代码
```c
int Check_SDRAM_Ready(void)
{
/*	0: SDRAM is not ready for access
	1: SDRAM is ready for access		*/	
	unsigned char temp;
	uint32_t to = 0;
	do
	{	
		temp=LCD_StatusRead();
		if ((temp & 0x04) != 0x00)
			return 1;               /* ready */
		delay_us(100);
	}while( ++to < 5000 );          /* ~500ms cap: on cold/residual-charge boot the
	                                   LT7580 SDRAM may never report ready; don't hang
	                                   until the ~30s IWDG timeout, let caller re-reset. */
	return 0;                       /* timeout: SDRAM not ready */
}
```

## 验证方式
```
Select-String -Path "D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.h" -Pattern "Check_SDRAM_Ready|LCD_SDRAM_initail|LCD_PLL_Initial|delay_ms|delay_us" | Select-Object LineNumber, Line
```

## 经验规则
添加返回值处理

---

# PCSE_035: lcd.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Now `Check_SDRAM_Ready` — add timeout and return status:

## 修改前代码
```c
void LCD_SDRAM_initail(unsigned char mclk)
{

uint8_t tempReg = 0;
	unsigned short sdram_itv;
```

## 修改后代码
```c
void LCD_SDRAM_initail(unsigned char mclk)
{

uint8_t tempReg = 0;
	unsigned short sdram_itv;
	int sdram_ready;
```

## 验证方式
```
Select-String -Path "D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.h" -Pattern "Check_SDRAM_Ready|LCD_SDRAM_initail|LCD_PLL_Initial|delay_ms|delay_us" | Select-Object LineNumber, Line
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_036: lcd.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Now `LCD_SDRAM_initail` — propagate readiness:

## 修改前代码
```c
void LCD_SDRAM_initail(unsigned char mclk)
{

uint8_t tempReg = 0;
	unsigned short sdram_itv;
	int sdram_ready;
```

## 修改后代码
```c
int LCD_SDRAM_initail(unsigned char mclk)
{

uint8_t tempReg = 0;
	unsigned short sdram_itv;
	int sdram_ready;
```

## 验证方式
```
Select-String -Path "D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.h" -Pattern "Check_SDRAM_Ready|LCD_SDRAM_initail|LCD_PLL_Initial|delay_ms|delay_us" | Select-Object LineNumber, Line
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_037: lcd.c - 添加返回值处理

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Now `LCD_SDRAM_initail` — propagate readiness:

## 修改前代码
```c
	tempReg = LCD_Reg_Read(0xE4);
	LCD_Reg_Write(0xE4, tempReg | 0x01);
Check_SDRAM_Ready();
delay_ms(50);
/*
  Write_Dir(0xe0,0x29);	
	Write_Dir(0xe1,0x03);
	Write_Dir(0xe2,0xe3);
	Write_Dir(0xe3,0x09);
	Write_Dir(0xe4,0x11);

	Check_SDRAM_Ready();	  
 */
delay_ms(100);

}
```

## 修改后代码
```c
	tempReg = LCD_Reg_Read(0xE4);
	LCD_Reg_Write(0xE4, tempReg | 0x01);
sdram_ready = Check_SDRAM_Ready();
delay_ms(50);
/*
  Write_Dir(0xe0,0x29);	
	Write_Dir(0xe1,0x03);
	Write_Dir(0xe2,0xe3);
	Write_Dir(0xe3,0x09);
	Write_Dir(0xe4,0x11);

	Check_SDRAM_Ready();	  
 */
delay_ms(100);

return sdram_ready;
}
```

## 验证方式
```
Select-String -Path "D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.h" -Pattern "Check_SDRAM_Ready|LCD_SDRAM_initail|LCD_PLL_Initial|delay_ms|delay_us" | Select-Object LineNumber, Line
```

## 经验规则
添加返回值处理

---

# PCSE_038: lcd.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Now `LCD_SDRAM_initail` — propagate readiness:

## 修改前代码
```c
 void LCD_PLL_Initial(void) 
{
//FOUT
```

## 修改后代码
```c
 void LCD_PLL_Initial(void) 
{
```

## 验证方式
```
Select-String -Path "D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.h" -Pattern "Check_SDRAM_Ready|LCD_SDRAM_initail|LCD_PLL_Initial|delay_ms|delay_us" | Select-Object LineNumber, Line
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_039: lcd.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Now `LCD_PLL_Initial` — return ready status:

## 修改前代码
```c
 void LCD_PLL_Initial(void) 
{
```

## 修改后代码
```c
 int LCD_PLL_Initial(void) 
{
//FOUT
```

## 验证方式
```
Select-String -Path "D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.c","D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.h" -Pattern "MCLK\s*=|char MCLK|int MCLK|MCLK;" | Select-Object Path,LineNumber,L
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_040: lcd.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Now `LCD_PLL_Initial` — return ready status:

## 修改前代码
```c
	{
		uint32_t lcd_pll_to = 0;
		while ((LCD_Reg_Read(0x00) & 0x80) == 0)
		{
			delay_us(100);
			if (++lcd_pll_to >= 20000) break;   /* ~2s cap: LT7580 PLL can be slow to report ready on cold power-up; don't hang boot (IWDG would reset) */
		}
	}
```

## 修改后代码
```c
	{
		uint32_t lcd_pll_to = 0;
		int pll_ready = 0;
		while ((LCD_Reg_Read(0x00) & 0x80) == 0)
		{
			delay_us(100);
			if (++lcd_pll_to >= 20000) break;   /* ~2s cap: LT7580 PLL can be slow to report ready on cold power-up; don't hang boot (IWDG would reset) */
		}
		if ((LCD_Reg_Read(0x00) & 0x80) != 0) pll_ready = 1;
		g_lcd_pll_ready = pll_ready;
	}
```

## 验证方式
```
Select-String -Path "D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.c","D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.h" -Pattern "MCLK\s*=|char MCLK|int MCLK|MCLK;" | Select-Object Path,LineNumber,L
```

## 经验规则
添加延时确保时序正确

---

# PCSE_041: lcd.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Now `LCD_PLL_Initial` — return ready status:

## 修改前代码
```c
	{
		uint32_t lcd_pll_to = 0;
		int pll_ready = 0;
		while ((LCD_Reg_Read(0x00) & 0x80) == 0)
		{
			delay_us(100);
			if (++lcd_pll_to >= 20000) break;   /* ~2s cap: LT7580 PLL can be slow to report ready on cold power-up; don't hang boot (IWDG would reset) */
		}
		if ((LCD_Reg_Read(0x00) & 0x80) != 0) pll_ready = 1;
		g_lcd_pll_ready = pll_ready;
	}
```

## 修改后代码
```c
	{
		uint32_t lcd_pll_to = 0;
		while ((LCD_Reg_Read(0x00) & 0x80) == 0)
		{
			delay_us(100);
			if (++lcd_pll_to >= 20000) break;   /* ~2s cap: LT7580 PLL can be slow to report ready on cold power-up; don't hang boot (IWDG would reset) */
		}
		if ((LCD_Reg_Read(0x00) & 0x80) != 0) pll_ready = 1;
	}
```

## 验证方式
```
Select-String -Path "D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.c","D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.h" -Pattern "MCLK\s*=|char MCLK|int MCLK|MCLK;" | Select-Object Path,LineNumber,L
```

## 经验规则
添加延时确保时序正确

---

# PCSE_042: lcd.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Now `LCD_PLL_Initial` — return ready status:

## 修改前代码
```c
 int LCD_PLL_Initial(void) 
{
//FOUT = XI *��M �� N���� OD
	unsigned int PCLK_M = 11;
```

## 修改后代码
```c
 int LCD_PLL_Initial(void) 
{
//FOUT = XI *��M �� N���� OD
	int pll_ready = 0;
	unsigned int PCLK_M = 11;
```

## 验证方式
```
Select-String -Path "D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.c","D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.h" -Pattern "MCLK\s*=|char MCLK|int MCLK|MCLK;" | Select-Object Path,LineNumber,L
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_043: lcd.c - 添加返回值处理

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Now `LCD_PLL_Initial` — return ready status:

## 修改前代码
```c
*/
delay_ms(10);

}

int LCD_SDRAM_initail(unsigned char mclk)
```

## 修改后代码
```c
*/
delay_ms(10);

return pll_ready;
}

int LCD_SDRAM_initail(unsigned char mclk)
```

## 验证方式
```
Select-String -Path "D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.c","D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.h" -Pattern "MCLK\s*=|char MCLK|int MCLK|MCLK;" | Select-Object Path,LineNumber,L
```

## 经验规则
添加返回值处理

---

# PCSE_044: lcd.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Now declare `pll_ready` at function top and return it:

## 修改前代码
```c
void LCD_Init(void)
{
LCD_Reset();

LCD_PLL_Initial();
LCD_SDRAM_initail(MCLK);

```

## 修改后代码
```c
void LCD_Init(void)
{
	/* Bring the LT7580 up with bounded retries. On a warm/residual-charge
	   power-up (e.g. off <10s) the controller supply is not fully settled,
	   so its PLL and especially the SDRAM can fail to report ready. Instead
	   of the status polls hanging forever until the ~30s IWDG timeout
	   (giving the ~40s "bad boot"), re-pulse the hardware reset and retry a
	   few times so a bad boot self-recovers in ~1-2s and boot time stays
	   stable. */
	{
		int lcd_try;
		for (lcd_try = 0; lcd_try < 5; lcd_try++)
		{
			LCD_Reset();                 /* hardware reset pulse of LT7580 */
			(void)LCD_PLL_Initial();
			if (LCD_SDRAM_initail(MCLK))  /* SDRAM reported ready */
				break;
			/* SDRAM not ready: LT7580 not stable yet -> reset & retry. */
		}
	}

```

## 验证方式
```
Select-String -Path "D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.c","D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.h" -Pattern "MCLK\s*=|char MCLK|int MCLK|MCLK;" | Select-Object Path,LineNumber,L
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_045: lcd.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
好的，按照你的方法来改

## 修改前代码
```c
void LCD_Init(void)
{
	/* Bring the LT7580 up with bounded retries. On a warm/residual-charge
	   power-up (e.g. off <10s) the controller supply is not fully settled,
	   so its PLL and especially the SDRAM can fail to report ready. Instead
	   of the status polls hanging forever until the ~30s IWDG timeout
	   (giving the ~40s "bad boot"), re-pulse the hardware reset and retry a
	   few times so a bad boot self-recovers in ~1-2s and boot time stays
	   stable. */
	{
		int lcd_try;
		for (lcd_try = 0; lcd_try < 5; lcd_try++)
		{
			LCD_Reset();                 /* hardware reset pulse of LT7580 */
			(void)LCD_PLL_Initial();
			if (LCD_SDRAM_initail(MCLK))  /* SDRAM reported ready */
				break;
			/* SDRAM not ready: LT7580 not stable yet -> reset & retry. */
		}
	}
		Select_LCD_DE_Mode();

```

## 修改后代码
```c
void LCD_Init(void)
{
	/* Cold / warm (residual-charge) power-up robustness for the LT7580.
	   Its own supply rail ramps slower than the MCU, so right after
	   power-on (especially a brief power-off where charge hasn't drained)
	   its PLL/SDRAM may not come up. Two-stage guard:
	     (1) give the LT7580 rail a CONTINUOUS settle window before touching
	         it -- rapid reset re-pulses can otherwise keep interrupting its
	         ramp -- while refreshing the IWDG so the wait never trips it;
	     (2) bring it up with bounded reset+init retries. If every retry
	         fails, force a full system reset instead of continuing with a
	         broken LCD (which, since it no longer hangs, would leave a
	         PERMANENT black screen with no IWDG recovery). Worst case thus
	         degrades to a reboot, never a dead display. */
	{
		int lcd_try;
		int i;

		/* (1) settle window: ~1.5s, refresh IWDG every 100ms */
		for (i = 0; i < 15; i++)
		{
			IWDG->KR = 0x0000AAAAu;          /* 
```

## 验证方式
```
$f="D:\reasonix\HelpPort\HelpPort\HelpPort\Core\Inc\main.h"; if(Test-Path $f){ Select-String -Path $f -Pattern "#include" | Select-Object -ExpandProperty Line } else { "main.h not at Core\Inc" }
```
输出: `#include "stm32f4xx_hal.h"
#include <stdio.h>
`

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_046: main.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
好的，按照你的方法来改

## 修改前代码
```c
  /* 2. ����ϵͳʱ�� (HSE -> PLL 168MHz) */
  /* Start the independent watchdog before clock/peripheral bring-up: any
     boot-phase or task hang auto-resets the MCU so the device always
     recovers to a normal start. LSI ~32kHz /256, reload 3750 => ~30s
     (>=20s even at 47kHz LSI). The long timeout is fine here: every
     boot bottleneck (ETH, prerender) feeds internally, and the runtime
     GUI loop feeds every 5ms. */
  hiwdg.Instance       = IWDG;
  hiwdg.Init.Prescaler = IWDG_PRESCALER_256;
  hiwdg.Init.Reload    = 3750;
  HAL_IWDG_Init(&hiwdg);

```

## 修改后代码
```c
  /* 2. ����ϵͳʱ�� (HSE -> PLL 168MHz) */
  /* Start the independent watchdog before clock/peripheral bring-up. During
     BOOT use a SHORT timeout so any cold/warm power-up hang (LT7580 status
     or DMA-busy polls that wedge before its supply settles) auto-recovers
     fast: a single reset re-inits the LT7580 cleanly, so ~6s recovery gives
     a ~18s worst-case boot instead of the ~44s the old 30s timeout caused.
     LSI ~32kHz /256 => ~125Hz; reload 750 => ~6s (>=4s even at 47kHz LSI).
     The timeout is RAISED back to ~30s just before the scheduler starts
     (see below) so the runtime prerender/GUI loop keeps a wide margin. */
  hiwdg.Instance       = IWDG;
  hiwdg.Init.Prescaler = IWDG_PRESCALER_256;
  hiwdg.Init.Reload    = 750;              /* ~6s boot-phase timeout */
  HAL_IWDG_Init(&hiwdg);

```

## 验证方式
```
$f="D:\reasonix\HelpPort\HelpPort\HelpPort\Core\Inc\main.h"; if(Test-Path $f){ Select-String -Path $f -Pattern "#include" | Select-Object -ExpandProperty Line } else { "main.h not at Core\Inc" }
```
输出: `#include "stm32f4xx_hal.h"
#include <stdio.h>
`

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_047: main.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
好的，按照你的方法来改

## 修改前代码
```c
  /* 8. ���������� (�� freertos.c �ж��岢���� GUI_Task, DAQ_Task, IO_Task) */
  MX_FREERTOS_Init();

  /* 9. ���� RTOS ��������� (�ڴ�֮�󣬿���Ȩ��ȫ�ƽ������񣬲��᷵��) */
  osKernelStart();
```

## 修改后代码
```c
  /* 8. ���������� (�� freertos.c �ж��岢���� GUI_Task, DAQ_Task, IO_Task) */
  MX_FREERTOS_Init();

  /* Boot bring-up done: raise the IWDG timeout back to ~30s for runtime.
     The heavy first-frame prerender runs inside the RTOS tasks and can
     stall the GUI feed longer than the 6s boot timeout, so restore the
     wide margin (reload 3750 => ~30s) now that the LT7580 is up. The GUI
     loop refreshes the watchdog every ~5ms once running. */
  hiwdg.Init.Reload = 3750;               /* ~30s runtime timeout */
  HAL_IWDG_Init(&hiwdg);
  HAL_IWDG_Refresh(&hiwdg);

  /* 9. ���� RTOS ��������� (�ڴ�֮�󣬿���Ȩ��ȫ�ƽ������񣬲��᷵��) */
  osKernelStart();
```

## 验证方式
```
$f="D:\reasonix\HelpPort\HelpPort\HelpPort\Core\Inc\main.h"; if(Test-Path $f){ Select-String -Path $f -Pattern "#include" | Select-Object -ExpandProperty Line } else { "main.h not at Core\Inc" }
```
输出: `#include "stm32f4xx_hal.h"
#include <stdio.h>
`

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_048: lcd.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Src/main.c

## 修改原因
卡死点基本锁定：`LCD_Init` 尾部的 `Init_Font()` 里有 **`Check_Busy_SFI_DMA()`**（`lcd.c:5265`，字库 SPI-Flash DMA 死等，无超时）。冷启动 LT7580 没稳时它会永久卡住 → 30s IWDG → 重启 → 第二次好 → 44s。而且这类无界死等在文件里有十几个（画图/BTE/FIFO），一个个堵是打地鼠。

## 关...

## 修改前代码
```c
void Check_Busy_SFI_DMA(void)
{
	LCD_CmdWrite(0xB6);
	do
	{		
	}while((LCD_DataRead()&0x01)==0x01);
}
```

## 修改后代码
```c
void Check_Busy_SFI_DMA(void)
{
	uint32_t t0 = HAL_GetTick();
	LCD_CmdWrite(0xB6);
	do
	{
		/* Cold-boot guard: the font-flash SFI-DMA busy bit can never clear
		   if the LT7580 was driven before its supply settled. Don't spin
		   forever (that hangs boot until the IWDG); reboot so a fresh,
		   stable-supply init comes up cleanly. Timeout is huge vs. normal
		   DMA time, so runtime never trips it. */
		if ((HAL_GetTick() - t0) > 3000u)
			HAL_NVIC_SystemReset();
	}while((LCD_DataRead()&0x01)==0x01);
}
```

## 验证方式
```
Select-String -Path "D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.c" -Pattern "void Check_Busy_SFI_DMA" | Select-Object LineNumber, Line
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_049: lcd.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Pushed: `3a948ab..5262b38 main -> main` (the red text is just git's stderr progress, not an error).

Now the optimization. Feeding the watchdog inside the long boot waits so the no-feed window drops t...

## 修改前代码
```c
	{
		uint32_t lcd_pll_to = 0;
		while ((LCD_Reg_Read(0x00) & 0x80) == 0)
		{
			delay_us(100);
			if (++lcd_pll_to >= 20000) break;   /* ~2s cap: LT7580 PLL can be slow to report ready on cold power-up; don't hang boot (IWDG would reset) */
		}
		if ((LCD_Reg_Read(0x00) & 0x80) != 0) pll_ready = 1;
	}
```

## 修改后代码
```c
	{
		uint32_t lcd_pll_to = 0;
		while ((LCD_Reg_Read(0x00) & 0x80) == 0)
		{
			if ((lcd_pll_to % 1000u) == 0u) IWDG->KR = 0x0000AAAAu;  /* feed every ~100ms during this legit up-to-2s wait */
			delay_us(100);
			if (++lcd_pll_to >= 20000) break;   /* ~2s cap: LT7580 PLL can be slow to report ready on cold power-up; don't hang boot (IWDG would reset) */
		}
		if ((LCD_Reg_Read(0x00) & 0x80) != 0) pll_ready = 1;
	}
```

## 验证方式
```
$f = Get-ChildItem -Path "D:\reasonix\HelpPort\HelpPort\HelpPort" -Recurse -Filter "CTP.c" | Select-Object -First 1 -ExpandProperty FullName; Write-Output $f; if($f){ Select-String -Path $f -Pattern "
```

## 经验规则
添加延时确保时序正确

---

# PCSE_050: lcd.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Pushed: `3a948ab..5262b38 main -> main` (the red text is just git's stderr progress, not an error).

Now the optimization. Feeding the watchdog inside the long boot waits so the no-feed window drops t...

## 修改前代码
```c
void LCD_Reset(void)
{
   LCD_RST0();
	delay_ms(100);     /* hold reset low: cold power-up needs a real pulse (10ms was too short) */
  LCD_RST1();
	delay_ms(800);     /* LT7580 power/PLL must stabilise on COLD boot; too-short (was 50ms) => later status polls hang forever => IWDG reset (~40s boots) */
}
```

## 修改后代码
```c
void LCD_Reset(void)
{
   int i;
   IWDG->KR = 0x0000AAAAu;
   LCD_RST0();
	delay_ms(100);     /* hold reset low: cold power-up needs a real pulse (10ms was too short) */
  LCD_RST1();
	/* LT7580 power/PLL must stabilise on COLD boot (too-short => later status
	   polls hang). Keep the ~800ms hold but feed the IWDG every 100ms so the
	   short boot-phase watchdog timeout doesn't trip during this legit wait. */
	for (i = 0; i < 8; i++)
	{
		IWDG->KR = 0x0000AAAAu;
		delay_ms(100);
	}
}
```

## 验证方式
```
$f = Get-ChildItem -Path "D:\reasonix\HelpPort\HelpPort\HelpPort" -Recurse -Filter "CTP.c" | Select-Object -First 1 -ExpandProperty FullName; Write-Output $f; if($f){ Select-String -Path $f -Pattern "
```

## 经验规则
添加延时确保时序正确

---

# PCSE_051: lcd.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Pushed: `3a948ab..5262b38 main -> main` (the red text is just git's stderr progress, not an error).

Now the optimization. Feeding the watchdog inside the long boot waits so the no-feed window drops t...

## 修改前代码
```c
	do
	{	
		temp=LCD_StatusRead();
		if ((temp & 0x04) != 0x00)
			return 1;               /* ready */
		delay_us(100);
	}while( ++to < 5000 );          /* ~500ms cap: on cold/residual-charge boot the
```

## 修改后代码
```c
	do
	{	
		temp=LCD_StatusRead();
		if ((temp & 0x04) != 0x00)
			return 1;               /* ready */
		if ((to % 1000u) == 0u) IWDG->KR = 0x0000AAAAu;   /* feed every ~100ms during the bounded wait */
		delay_us(100);
	}while( ++to < 5000 );          /* ~500ms cap: on cold/residual-charge boot the
```

## 验证方式
```
$f = Get-ChildItem -Path "D:\reasonix\HelpPort\HelpPort\HelpPort" -Recurse -Filter "CTP.c" | Select-Object -First 1 -ExpandProperty FullName; Write-Output $f; if($f){ Select-String -Path $f -Pattern "
```

## 经验规则
添加延时确保时序正确

---

# PCSE_052: lcd.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Now the PLL wait loop:

## 修改前代码
```c
{
	uint32_t t0 = HAL_GetTick();
	LCD_CmdWrite(0xB6);
	do
	{
		/* Cold-boot guard: the font-flash SFI-DMA busy bit can never clear
		   if the LT7580 was driven before its supply settled. Don't spin
		   forever (that hangs boot until the IWDG); reboot so a fresh,
		   stable-supply init comes up cleanly. Timeout is huge vs. normal
		   DMA time, so runtime never trips it. */
		if ((HAL_GetTick() - t0) > 3000u)
			HAL_NVIC_SystemReset();
	}while((LCD_DataRead()&0x01)==0x01);
}
```

## 修改后代码
```c
{
	uint32_t t0 = HAL_GetTick();
	LCD_CmdWrite(0xB6);
	do
	{
		/* Feed the IWDG: a legit font-flash DMA transfer can outlast the
		   short boot-phase watchdog timeout. A true wedge (busy bit never
		   clears on unstable supply) is instead bounded by the 3s wall-clock
		   guard below -> reboot for a clean, stable-supply init. The 3s cap
		   is far above the real DMA time, so good boots never trip it. */
		IWDG->KR = 0x0000AAAAu;
		if ((HAL_GetTick() - t0) > 3000u)
			HAL_NVIC_SystemReset();
	}while((LCD_DataRead()&0x01)==0x01);
}
```

## 验证方式
```
$f = Get-ChildItem -Path "D:\reasonix\HelpPort\HelpPort\HelpPort" -Recurse -Filter "CTP.c" | Select-Object -First 1 -ExpandProperty FullName; Write-Output $f; if($f){ Select-String -Path $f -Pattern "
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_053: main.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Src/main.c

## 修改原因
`CTP_Init` is fully bounded (~400ms max, HAL_GetTick-guarded INT wait). Post-`LCD_Init` unfed gap (Set_Color+GUI_CleanScreen+CTP_Init+Display_ON) ≤ ~500ms — and a `GUI_CleanScreen` wedge stays catchab...

## 修改前代码
```c
  /* 2. ����ϵͳʱ�� (HSE -> PLL 168MHz) */
  /* Start the independent watchdog before clock/peripheral bring-up. During
     BOOT use a SHORT timeout so any cold/warm power-up hang (LT7580 status
     or DMA-busy polls that wedge before its supply settles) auto-recovers
     fast: a single reset re-inits the LT7580 cleanly, so ~6s recovery gives
     a ~18s worst-case boot instead of the ~44s the old 30s timeout caused.
     LSI ~32kHz /256 => ~125Hz; reload 750 => ~6s (>=4s even at 47kHz LSI).
     The timeout is RAISED back to ~30s just before the scheduler starts
     (see below) so the runtime prerender/GUI loop keeps a wide margin. */
  hiwdg.Instance       = IWDG;
  hiwdg.Init.Prescaler = IWDG_PRESCALER_256;
  hiwdg.Init.Reload    = 750;              /* ~6s boot-phase timeout */
  HAL_IWDG_Init(&hiwdg);

```

## 修改后代码
```c
  /* 2. ����ϵͳʱ�� (HSE -> PLL 168MHz) */
  /* Start the independent watchdog before clock/peripheral bring-up. During
     BOOT use a SHORT timeout so any cold/warm power-up hang (LT7580 status
     or DMA-busy polls that wedge before its supply settles) auto-recovers
     fast: a single reset re-inits the LT7580 cleanly. All legit long boot
     waits (LCD_Reset hold, LT7580 PLL/SDRAM polls, font SFI-DMA) now feed
     the IWDG internally, so the longest un-fed window is ~0.5s (CTP_Init +
     screen clear); a ~2.5s timeout clears that with wide margin even at the
     fastest LSI, while cutting the warm-boot wedge penalty from ~6s to
     ~2.5s. LSI ~32kHz /256 => ~125Hz; reload 312 => ~2.5s (~1.7s at 47kHz
     LSI, ~4.7s at 17kHz LSI). Raised back to ~30s just before the scheduler
     (see below) so the runtime prerender/GUI loop keeps a wide margin. */
  hiwdg.Instance       = IWDG;
  hiwdg.Init.Prescaler = IWDG_PRESCALER_256;
  hiwdg.Init.Reload    = 312;              /* ~2.5s b
```

## 验证方式
```
Select-String -Path "D:\reasonix\HelpPort\HelpPort\HelpPort\LCD\lcd.c" -Pattern "IWDG->KR = 0x0000AAAAu" | Select-Object LineNumber, Line
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_054: lcd.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Root cause confirmed: unbounded/unfed waits in the prerender/page-switch path (`ui_safehouse.c` SPI-DMA spins + `Check_BTE_Busy`) that run under the 30s runtime IWDG. I'll bound all of them with the s...

## 修改前代码
```c
/*
BTE Function Status
0 : BTE Function is idle.
1 : BTE Function is busy.
*/
	unsigned char temp; 	
	do
	{
		temp=LCD_StatusRead();
	}while(temp&0x08);

}
```

## 修改后代码
```c
/*
BTE Function Status
0 : BTE Function is idle.
1 : BTE Function is busy.
*/
	unsigned char temp;
	uint32_t t0 = HAL_GetTick();
	do
	{
		/* A wedged LT7580 BTE engine (unstable cold/warm-boot supply) would
		   spin here forever with the watchdog un-fed -> ~30s runtime IWDG
		   recovery. Reboot after 3s for a clean, stable-supply init. 3s is
		   far above a real BTE copy time, so normal operation never trips. */
		if ((HAL_GetTick() - t0) > 3000u)
			HAL_NVIC_SystemReset();
		temp=LCD_StatusRead();
	}while(temp&0x08);

}
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Write-Output "=== ui_safehouse extern spi_dma ==="; Select-String -Path "Core\Src\ui_safehouse.c" -Pattern "spi_dma_is_transmitting|#include" | S
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_055: lv_port_disp.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Root cause confirmed: unbounded/unfed waits in the prerender/page-switch path (`ui_safehouse.c` SPI-DMA spins + `Check_BTE_Busy`) that run under the 30s runtime IWDG. I'll bound all of them with the s...

## 修改前代码
```c
static lv_disp_drv_t disp_drv;
volatile uint8_t spi_dma_is_transmitting = 0;
volatile uint8_t g_bte_suppress = 0;          /* set on BTE page-switch to skip the full repaint pass */
```

## 修改后代码
```c
static lv_disp_drv_t disp_drv;
volatile uint8_t spi_dma_is_transmitting = 0;
volatile uint8_t g_bte_suppress = 0;          /* set on BTE page-switch to skip the full repaint pass */

/* Bounded wait for the SPI-DMA flush to finish. A stuck transfer (SPI/DMA
   fault, or an LT7580 wedged on an unstable cold/warm-boot supply) would
   otherwise spin here forever with the watchdog un-fed -> a ~30s runtime
   IWDG recovery (the ~34s boots). Reboot after 3s instead so a fresh,
   stable-supply init comes up fast. 3s is far above a full-frame flush
   time, so normal operation never trips it. */
void SPI_DMA_WaitIdle(void)
{
    uint32_t t0 = HAL_GetTick();
    while (spi_dma_is_transmitting)
    {
        if (HAL_GetTick() - t0 > 3000u)
            HAL_NVIC_SystemReset();
    }
}
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Write-Output "=== ui_safehouse extern spi_dma ==="; Select-String -Path "Core\Src\ui_safehouse.c" -Pattern "spi_dma_is_transmitting|#include" | S
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_056: lv_port_disp.c - 使用DMA减轻CPU负担

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Root cause confirmed: unbounded/unfed waits in the prerender/page-switch path (`ui_safehouse.c` SPI-DMA spins + `Check_BTE_Busy`) that run under the 30s runtime IWDG. I'll bound all of them with the s...

## 修改前代码
```c
    while(spi_dma_is_transmitting) {
        osThreadYield();
    }
```

## 修改后代码
```c
    while(spi_dma_is_transmitting) {
        if (HAL_GetTick() - _flush_t0 > 3000u) HAL_NVIC_SystemReset();
        osThreadYield();
    }
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Write-Output "=== ui_safehouse extern spi_dma ==="; Select-String -Path "Core\Src\ui_safehouse.c" -Pattern "spi_dma_is_transmitting|#include" | S
```

## 经验规则
使用DMA减轻CPU负担

---

# PCSE_057: lv_port_disp.c - 使用DMA减轻CPU负担

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LVGL/examples/porting/lv_port_disp.c

## 修改原因
Edit 2 — add `SPI_DMA_WaitIdle` helper and bound the flush spin in lv_port_disp.c:

## 修改前代码
```c
    while(spi_dma_is_transmitting) {
        if (HAL_GetTick() - _flush_t0 > 3000u) HAL_NVIC_SystemReset();
        osThreadYield();
    }
```

## 修改后代码
```c
    uint32_t _flush_t0 = HAL_GetTick();
    while(spi_dma_is_transmitting) {
        if (HAL_GetTick() - _flush_t0 > 3000u) HAL_NVIC_SystemReset();
        osThreadYield();
    }
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Write-Output "=== ui_safehouse extern spi_dma ==="; Select-String -Path "Core\Src\ui_safehouse.c" -Pattern "spi_dma_is_transmitting|#include" | S
```

## 经验规则
使用DMA减轻CPU负担

---

# PCSE_058: lv_port_disp.h - 使用DMA减轻CPU负担

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LVGL/examples/porting/lv_port_disp.c

## 修改原因
Need to declare `_flush_t0` before that loop:

## 修改前代码
```c
extern volatile uint8_t spi_dma_is_transmitting;
```

## 修改后代码
```c
extern volatile uint8_t spi_dma_is_transmitting;

/* Bounded wait for the SPI-DMA flush to finish (reboots on a >3s stall) */
void SPI_DMA_WaitIdle(void);
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Write-Output "=== remaining plain spins (should be none in ui_safehouse) ==="; Get-ChildItem -Recurse -Include *.c | Select-String -Pattern "whil
```

## 经验规则
使用DMA减轻CPU负担

---

# PCSE_059: ui_safehouse.c - 使用DMA减轻CPU负担

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LVGL/examples/porting/lv_port_disp.h

## 修改原因
Edit 3 — prototype in header:

## 修改前代码
```c
extern volatile uint8_t spi_dma_is_transmitting;  /* defined in lv_port_disp.c */
```

## 修改后代码
```c
extern volatile uint8_t spi_dma_is_transmitting;  /* defined in lv_port_disp.c */
void SPI_DMA_WaitIdle(void);                        /* bounded wait, defined in lv_port_disp.c */
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Write-Output "=== remaining plain spins (should be none in ui_safehouse) ==="; Get-ChildItem -Recurse -Include *.c | Select-String -Pattern "whil
```

## 经验规则
使用DMA减轻CPU负担

---

# PCSE_060: ui_safehouse.c - 使用DMA减轻CPU负担

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Src/ui_safehouse.c

## 修改原因
Edit 4 — `ui_safehouse.c`: add prototype and replace the 5 spins:

## 修改前代码
```c
    while (spi_dma_is_transmitting);
```

## 修改后代码
```c
    SPI_DMA_WaitIdle();
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Write-Output "=== remaining plain spins (should be none in ui_safehouse) ==="; Get-ChildItem -Recurse -Include *.c | Select-String -Pattern "whil
```

## 经验规则
使用DMA减轻CPU负担

---

# PCSE_061: main.c - 添加复位逻辑恢复初始状态

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Inc/boot_trace.h

## 修改原因
All macros exist. Creating the trace facility:

## 修改前代码
```c
#include "gui.h"
#include "eth_at.h"              // NE2 ���� AT ָ���ѯ
```

## 修改后代码
```c
#include "gui.h"
#include "eth_at.h"              // NE2 ���� AT ָ���ѯ
#include "boot_trace.h"          // reset-surviving boot trace in Backup SRAM (ST-Link debug)
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Test-Path "Core\Inc"; Write-Output "--- confirm macros exist in HAL ---"; Select-String -Path "Drivers\STM32F4xx_HAL_Driver\Inc\stm32f4xx_hal_rcc
```
输出: `True
--- confirm macros exist in HAL ---

LineNumber Line                                                            
---------- ----             `

## 经验规则
添加复位逻辑恢复初始状态

---

# PCSE_062: main.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Inc/boot_trace.h

## 修改原因
All macros exist. Creating the trace facility:

## 修改前代码
```c
void SystemClock_Config(void);
void MX_FREERTOS_Init(void);


```

## 修改后代码
```c
void SystemClock_Config(void);
void MX_FREERTOS_Init(void);

/* Reset-surviving boot trace in Backup SRAM (0x40024000). Records boot/reset
   counts and how far each boot got, so a hung boot's wedge point survives the
   IWDG reset and is readable via ST-Link (-r32 0x40024000 8). See boot_trace.h. */
void BootTrace_Init(void)
{
  __HAL_RCC_PWR_CLK_ENABLE();
  __HAL_RCC_BKPSRAM_CLK_ENABLE();
  HAL_PWR_EnableBkUpAccess();

  if (BKP_TRACE[0] != BOOT_TRACE_MAGIC)
  {
    int i;
    for (i = 0; i < 8; i++) BKP_TRACE[i] = 0;
    BKP_TRACE[0] = BOOT_TRACE_MAGIC;      /* first power-on: init region */
  }

  BKP_TRACE[1]++;                          /* boot_count */
  {
    uint32_t csr = RCC->CSR;
    BKP_TRACE[4] = csr;                    /* raw reset-cause flags */
    if (csr & RCC_CSR_IWDGRSTF) BKP_TRACE[2]++;
    if (csr & RCC_CSR_SFTRSTF)  BKP_TRACE[3]++;
  }
  BKP_TRACE[6] = BKP_TRACE[7];             /* previous boot's furthest stage = hang point */
  BKP_TRACE[7] = 0;                 
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Test-Path "Core\Inc"; Write-Output "--- confirm macros exist in HAL ---"; Select-String -Path "Drivers\STM32F4xx_HAL_Driver\Inc\stm32f4xx_hal_rcc
```
输出: `True
--- confirm macros exist in HAL ---

LineNumber Line                                                            
---------- ----             `

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_063: main.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Inc/boot_trace.h

## 修改原因
All macros exist. Creating the trace facility:

## 修改前代码
```c
  /* 1. ��λ�������裬��ʼ��Flash�ӿں�ϵͳ�δ�ʱ�� */
  HAL_Init();

```

## 修改后代码
```c
  /* 1. ��λ�������裬��ʼ��Flash�ӿں�ϵͳ�δ�ʱ�� */
  HAL_Init();

  BootTrace_Init();   /* record reset cause + boot count; expose prev hang stage */

```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Test-Path "Core\Inc"; Write-Output "--- confirm macros exist in HAL ---"; Select-String -Path "Drivers\STM32F4xx_HAL_Driver\Inc\stm32f4xx_hal_rcc
```
输出: `True
--- confirm macros exist in HAL ---

LineNumber Line                                                            
---------- ----             `

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_064: main.c - 添加配置初始化

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Src/main.c

## 修改原因
Now wire it into main.c — include, init function, and stage markers:

## 修改前代码
```c
  SystemClock_Config();
  HAL_IWDG_Refresh(&hiwdg);
  __HAL_RCC_SYSCFG_CLK_ENABLE();

```

## 修改后代码
```c
  SystemClock_Config();
  HAL_IWDG_Refresh(&hiwdg);
  __HAL_RCC_SYSCFG_CLK_ENABLE();
  BOOT_STAGE(ST_CLOCKS_OK);

```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Get-ChildItem -Path "Drivers\STM32F4xx_HAL_Driver\Inc" -Filter *.h | Select-String -Pattern "BKPSRAM_CLK_ENABLE|BKPSRAM_BASE" | Select-Object Pat
```

## 经验规则
添加配置初始化

---

# PCSE_065: main.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Src/main.c

## 修改原因
Now call `BootTrace_Init()` after `HAL_Init()` and add stage markers through the boot path:

## 修改前代码
```c
  DIO_System_Init();            // ����������������ƹܽ���״̬��ʼ��
  DIO_Update_Outputs();         // �������� 74HC595����ֹ AT ��ѯ�ڼ� DO �����

  /* 5. ����ģ�����ͨ�� (DAC �������ʹ��) */
  /* DAC already brought up at 0 right after MX_GPIO_Init (see above) */
  
  /* 6. Display and touch hardware init */
  LCD_Init();
  Set_Color(color_black, color_white);
  GUI_CleanScreen();

  CTP_Init();                          /* touch init → ready before screen on */

  Display_ON();                        /* screen visible */

  /* 6.5. Query NE2 ETH module (reduced delay: 100ms+800ms) */
  HAL_IWDG_Refresh(&hiwdg);
  ETH_AT_QueryNetworkInfo();
  HAL_IWDG_Refresh(&hiwdg);
	
	lv_init();                             // LVGL ��ʼ��
	lv_port_disp_init();                   // ע��LVGL����ʾ����
	lv_port_indev_init();                  // ע��LVGL�Ĵ����������

  /* 7. ��ʼ�� FreeRTOS ϵͳ�ں˶��� */
  HAL_IWDG_Refresh(&hiwdg);
  osKernelInitialize();

  /* 8. ���������� (�� freertos.c �ж��岢���� GUI_Task, DAQ_Task, 
```

## 修改后代码
```c
  DIO_System_Init();            // ����������������ƹܽ���״̬��ʼ��
  DIO_Update_Outputs();         // �������� 74HC595����ֹ AT ��ѯ�ڼ� DO �����
  BOOT_STAGE(ST_PERIPH_OK);

  /* 5. ����ģ�����ͨ�� (DAC �������ʹ��) */
  /* DAC already brought up at 0 right after MX_GPIO_Init (see above) */
  
  /* 6. Display and touch hardware init */
  LCD_Init();
  BOOT_STAGE(ST_LCD_DONE);
  Set_Color(color_black, color_white);
  GUI_CleanScreen();
  BOOT_STAGE(ST_CLEANSCREEN);

  CTP_Init();                          /* touch init → ready before screen on */
  BOOT_STAGE(ST_CTP);

  Display_ON();                        /* screen visible */
  BOOT_STAGE(ST_DISPLAY_ON);

  /* 6.5. Query NE2 ETH module (reduced delay: 100ms+800ms) */
  HAL_IWDG_Refresh(&hiwdg);
  ETH_AT_QueryNetworkInfo();
  HAL_IWDG_Refresh(&hiwdg);
  BOOT_STAGE(ST_ETH);
	
	lv_init();                             // LVGL ��ʼ��
	lv_port_disp_init();                   // ע��LVGL����ʾ����
	lv_port_indev_init();                  // ע��LVGL�Ĵ������
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "LCD\lcd.c" -Pattern "^\s*Init_Font\(\);|screeninit" | Select-Object LineNumber, Line
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_066: lcd.c - 添加复位逻辑恢复初始状态

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Src/main.c

## 修改原因
Now call `BootTrace_Init()` after `HAL_Init()` and add stage markers through the boot path:

## 修改前代码
```c
#include "spi.h"
#include "main.h"
#include "usart.h"
#include "lcd.h"
#include "lvgl.h"
```

## 修改后代码
```c
#include "spi.h"
#include "main.h"
#include "usart.h"
#include "lcd.h"
#include "lvgl.h"
#include "boot_trace.h"   /* reset-surviving boot stage markers (ST-Link debug) */
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "LCD\lcd.c" -Pattern "^\s*Init_Font\(\);|screeninit" | Select-Object LineNumber, Line
```

## 经验规则
添加复位逻辑恢复初始状态

---

# PCSE_067: lcd.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Src/main.c

## 修改原因
Now call `BootTrace_Init()` after `HAL_Init()` and add stage markers through the boot path:

## 修改前代码
```c
	{
		int lcd_try;
		int i;

		/* (1) settle window: ~1.5s, refresh IWDG every 100ms */
		for (i = 0; i < 15; i++)
		{
			IWDG->KR = 0x0000AAAAu;          /* reload IWDG counter */
			delay_ms(100);
		}

		/* (2) bounded bring-up: proceed as soon as SDRAM reports ready */
		for (lcd_try = 0; lcd_try < 8; lcd_try++)
		{
			IWDG->KR = 0x0000AAAAu;
			LCD_Reset();                     /* hardware reset pulse of LT7580 */
			(void)LCD_PLL_Initial();
			if (LCD_SDRAM_initail(MCLK))     /* SDRAM reported ready */
				break;
			/* not ready yet: LT7580 still unstable -> reset & retry */
		}

```

## 修改后代码
```c
	{
		int lcd_try;
		int i;

		BOOT_STAGE(ST_LCD_BEGIN);

		/* (1) settle window: ~1.5s, refresh IWDG every 100ms */
		for (i = 0; i < 15; i++)
		{
			IWDG->KR = 0x0000AAAAu;          /* reload IWDG counter */
			delay_ms(100);
		}
		BOOT_STAGE(ST_LCD_SETTLE);

		/* (2) bounded bring-up: proceed as soon as SDRAM reports ready */
		for (lcd_try = 0; lcd_try < 8; lcd_try++)
		{
			IWDG->KR = 0x0000AAAAu;
			LCD_Reset();                     /* hardware reset pulse of LT7580 */
			(void)LCD_PLL_Initial();
			BOOT_STAGE(ST_LCD_PLL);
			if (LCD_SDRAM_initail(MCLK))     /* SDRAM reported ready */
			{
				BOOT_STAGE(ST_LCD_SDRAM);
				break;
			}
			/* not ready yet: LT7580 still unstable -> reset & retry */
		}

```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "LCD\lcd.c" -Pattern "^\s*Init_Font\(\);|screeninit" | Select-Object LineNumber, Line
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_068: lcd.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Now instrument LCD_Init internals in lcd.c. Adding the include:

## 修改前代码
```c
	//screeninit();
Init_Font();

}
```

## 修改后代码
```c
	//screeninit();
Init_Font();
BOOT_STAGE(ST_LCD_FONT);

}
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "LCD\lcd.c" -Pattern "^\s*Init_Font\(\);|screeninit" | Select-Object LineNumber, Line
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_069: task_gui.c - 操作硬件寄存器实现功能

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Now the `Init_Font()` call at the end of LCD_Init:

## 修改前代码
```c
#include "cmsis_os.h"
#include "main.h"
#include "sys_registers.h"
```

## 修改后代码
```c
#include "cmsis_os.h"
#include "main.h"
#include "sys_registers.h"
#include "boot_trace.h"
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "LCD\lcd.c" -Pattern "^\s*Init_Font\(\);|screeninit" | Select-Object LineNumber, Line
```

## 经验规则
操作硬件寄存器实现功能

---

# PCSE_070: task_gui.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
Now the `Init_Font()` call at the end of LCD_Init:

## 修改前代码
```c
void Task_GUI(void *argument)
{
    HAL_IWDG_Refresh(&hiwdg);
    ui_init();
    ui_safehouse_init();
```

## 修改后代码
```c
void Task_GUI(void *argument)
{
    HAL_IWDG_Refresh(&hiwdg);
    BOOT_STAGE(ST_GUI_ENTRY);
    ui_init();
    ui_safehouse_init();
    BOOT_STAGE(ST_GUI_UIINIT);
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "LCD\lcd.c" -Pattern "^\s*Init_Font\(\);|screeninit" | Select-Object LineNumber, Line
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_071: task_gui.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/app/tasks/task_gui.c

## 修改原因
Now the runtime/prerender path (the 34s region) — `task_gui.c`:

## 修改前代码
```c
    /* Render first frame to SDRAM, then let SPI DMA flush */
    lv_task_handler();
    vTaskDelay(pdMS_TO_TICKS(5));

    /* Pre-render BOTH pages behind the boot black screen so the first
       switch to DIDO is instant (adds ~3.7s black time, then backlight on) */
    ui_safehouse_prerender();
    Set_Backlight(100);
```

## 修改后代码
```c
    /* Render first frame to SDRAM, then let SPI DMA flush */
    lv_task_handler();
    BOOT_STAGE(ST_GUI_FIRSTFRAME);
    vTaskDelay(pdMS_TO_TICKS(5));

    /* Pre-render BOTH pages behind the boot black screen so the first
       switch to DIDO is instant (adds ~3.7s black time, then backlight on) */
    ui_safehouse_prerender();
    Set_Backlight(100);
    BOOT_STAGE(ST_BACKLIGHT);
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "LCD\lcd.c" -Pattern "^\s*Init_Font\(\);|screeninit" | Select-Object LineNumber, Line
```

## 经验规则
添加延时确保时序正确

---

# PCSE_072: task_gui.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/app/tasks/task_gui.c

## 修改原因
Now the runtime/prerender path (the 34s region) — `task_gui.c`:

## 修改前代码
```c
        lv_tick_inc(elapsed);
        lv_task_handler();
        HAL_IWDG_Refresh(&hiwdg);

        osDelay(5);
```

## 修改后代码
```c
        lv_tick_inc(elapsed);
        lv_task_handler();
        HAL_IWDG_Refresh(&hiwdg);
        BOOT_STAGE(ST_RUN_LOOP);

        osDelay(5);
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "LCD\lcd.c" -Pattern "^\s*Init_Font\(\);|screeninit" | Select-Object LineNumber, Line
```

## 经验规则
添加延时确保时序正确

---

# PCSE_073: ui_safehouse.c - 修复代码逻辑错误

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/app/tasks/task_gui.c

## 修改原因
Now the runtime/prerender path (the 34s region) — `task_gui.c`:

## 修改前代码
```c
#include "cmsis_os.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "lcd.h"   /* BTE / Layer macros / LCD_BTE_Copy */
```

## 修改后代码
```c
#include "cmsis_os.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "lcd.h"   /* BTE / Layer macros / LCD_BTE_Copy */
#include "boot_trace.h"
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "LCD\lcd.c" -Pattern "^\s*Init_Font\(\);|screeninit" | Select-Object LineNumber, Line
```

## 经验规则
修复代码逻辑错误

---

# PCSE_074: ui_safehouse.c - 使用DMA减轻CPU负担

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Src/ui_safehouse.c

## 修改原因
Now `ui_safehouse.c` prerender steps:

## 修改前代码
```c
    /* ensure AIAO (boot screen) is fully rendered, then snapshot -> Layer1 */
    for (i = 0; i < 4; i++) { lv_tick_inc(30); lv_task_handler(); HAL_IWDG_Refresh(&hiwdg); }
    SPI_DMA_WaitIdle();
    LCD_BTE_Copy((unsigned long)Layer0, Layer1, (unsigned short)lcdx, (unsigned short)lcdy);
    s_snap_valid[0] = 1;

    /* render DIDO once into Layer0 (the ~3.7s, hidden behind boot black screen) */
    lv_scr_load(objects.dido);
    for (i = 0; i < 15; i++) { lv_tick_inc(30); lv_task_handler(); HAL_IWDG_Refresh(&hiwdg); }
    SPI_DMA_WaitIdle();
    LCD_BTE_Copy((unsigned long)Layer0, Layer2, (unsigned short)lcdx, (unsigned short)lcdy);
    s_snap_valid[1] = 1;

    /* restore AIAO instantly and make it the active screen again */
    SPI_DMA_WaitIdle();
    LCD_BTE_Copy(Layer1, (unsigned long)Layer0, (unsigned short)lcdx, (unsigned short)lcdy);
    lv_scr_load(objects.aiao);
    g_bte_suppress = 1;   /* skip AIAO repaint; snapshot already restored to Layer0 */
    for (i = 0; i < 4; i++)
```

## 修改后代码
```c
    /* ensure AIAO (boot screen) is fully rendered, then snapshot -> Layer1 */
    for (i = 0; i < 4; i++) { lv_tick_inc(30); lv_task_handler(); HAL_IWDG_Refresh(&hiwdg); }
    BOOT_STAGE(ST_PRE_AIAO);
    SPI_DMA_WaitIdle();
    LCD_BTE_Copy((unsigned long)Layer0, Layer1, (unsigned short)lcdx, (unsigned short)lcdy);
    s_snap_valid[0] = 1;
    BOOT_STAGE(ST_PRE_BTE1);

    /* render DIDO once into Layer0 (the ~3.7s, hidden behind boot black screen) */
    lv_scr_load(objects.dido);
    for (i = 0; i < 15; i++) { lv_tick_inc(30); lv_task_handler(); HAL_IWDG_Refresh(&hiwdg); }
    BOOT_STAGE(ST_PRE_DIDO);
    SPI_DMA_WaitIdle();
    LCD_BTE_Copy((unsigned long)Layer0, Layer2, (unsigned short)lcdx, (unsigned short)lcdy);
    s_snap_valid[1] = 1;
    BOOT_STAGE(ST_PRE_BTE2);

    /* restore AIAO instantly and make it the active screen again */
    SPI_DMA_WaitIdle();
    LCD_BTE_Copy(Layer1, (unsigned long)Layer0, (unsigned short)lcdx, (unsigned short)lcdy);
    lv_scr_load(objects.aiao)
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "Drivers\CMSIS\Device\ST\STM32F4xx\Include\stm32f407xx.h" -Pattern "define BKPSRAM_BASE" | Select-Object LineNumber, Line
```

## 经验规则
使用DMA减轻CPU负担

---

# PCSE_075: main.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Inc/boot_trace.h

## 修改原因
Now we have the full picture — and it **rewrites the diagnosis**:

| word | value | meaning |
|---|---|---|
| [1] boot_count | **1** | only ONE boot this power session |
| [2] iwdg_reset_count | **0**...

## 修改前代码
```c
  if (BKP_TRACE[0] != BOOT_TRACE_MAGIC)
  {
    int i;
    for (i = 0; i < 8; i++) BKP_TRACE[i] = 0;
    BKP_TRACE[0] = BOOT_TRACE_MAGIC;      /* first power-on: init region */
  }

  BKP_TRACE[1]++;                          /* boot_count */
  {
    uint32_t csr = RCC->CSR;
    BKP_TRACE[4] = csr;                    /* raw reset-cause flags */
    if (csr & RCC_CSR_IWDGRSTF) BKP_TRACE[2]++;
    if (csr & RCC_CSR_SFTRSTF)  BKP_TRACE[3]++;
  }
  BKP_TRACE[6] = BKP_TRACE[7];             /* previous boot's furthest stage = hang point */
  BKP_TRACE[7] = 0;                        /* reset this-boot max */
  BKP_TRACE[5] = 0;                        /* reset live stage */

```

## 修改后代码
```c
  if (BKP_TRACE[0] != BOOT_TRACE_MAGIC)
  {
    int i;
    for (i = 0; i < 96; i++) BKP_TRACE[i] = 0;   /* header + timestamp table */
    BKP_TRACE[0] = BOOT_TRACE_MAGIC;      /* first power-on: init region */
  }

  BKP_TRACE[1]++;                          /* boot_count */
  {
    uint32_t csr = RCC->CSR;
    BKP_TRACE[4] = csr;                    /* raw reset-cause flags */
    if (csr & RCC_CSR_IWDGRSTF) BKP_TRACE[2]++;
    if (csr & RCC_CSR_SFTRSTF)  BKP_TRACE[3]++;
  }
  BKP_TRACE[6] = BKP_TRACE[7];             /* previous boot's furthest stage = hang point */
  BKP_TRACE[7] = 0;                        /* reset this-boot max */
  BKP_TRACE[5] = 0;                        /* reset live stage */
  {
    int i;
    for (i = 0; i < 32; i++) BKP_TS(i) = 0; /* clear per-stage timestamps */
  }

```

## 验证方式
```
$cli = "C:\Program Files (x86)\STMicroelectronics\STM32 ST-LINK Utility\ST-LINK Utility\ST-LINK_CLI.exe"
foreach($i in 0..7){
  $addr = 0x40024000 + ($i*4)
  $h = "0x{0:X8}" -f $addr
  $out = & $cli -
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_076: boot_trace.h - 使用宏定义消除魔术数字

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Inc/boot_trace.h

## 修改原因
Now we have the full picture — and it **rewrites the diagnosis**:

| word | value | meaning |
|---|---|---|
| [1] boot_count | **1** | only ONE boot this power session |
| [2] iwdg_reset_count | **0**...

## 修改前代码
```c
 *   [5] 0x40024014  cur_stage          (last BOOT_STAGE reached, updated live)
 *   [6] 0x40024018  prev_boot_max_stage(furthest stage the PREVIOUS boot
 *                                       reached before it reset == hang point)
 *   [7] 0x4002401C  this_boot_max_stage(furthest stage reached this boot)
 */
#ifndef BOOT_TRACE_H
#define BOOT_TRACE_H

#include <stdint.h>
#include "main.h"

#define BOOT_TRACE_MAGIC   0xB007C0DEu
#define BKP_TRACE          ((volatile uint32_t *)BKPSRAM_BASE)

/* Call once, as early as possible in main() (after HAL_Init). */
void BootTrace_Init(void);

/* Record progress. cur_stage tracks live position; this-boot max is latched
   so a reset preserves "furthest reached" for the next boot to expose. */
#define BOOT_STAGE(n)                                             \
    do {                                                          \
        uint32_t _s = (uint32_t)(n);                              \
        BKP_TRACE[5] = _s;                          
```

## 修改后代码
```c
 *   [5] 0x40024014  cur_stage          (last BOOT_STAGE reached, updated live)
 *   [6] 0x40024018  prev_boot_max_stage(furthest stage the PREVIOUS boot
 *                                       reached before it reset == hang point)
 *   [7] 0x4002401C  this_boot_max_stage(furthest stage reached this boot)
 *   [8] 0x40024020  sdram_retry_count  (LT7580 SDRAM bring-up retries used)
 *
 * Per-stage arrival timestamps (ms since boot, HAL_GetTick) live at
 *   0x40024000 + 4*(64 + stage_code)  i.e. base 0x40024100 + 4*code.
 * Read the whole timestamp table with ST-Link:
 *   -r32 0x40024100 32
 * The delta between consecutive stage timestamps reveals the slow stage.
 */
#ifndef BOOT_TRACE_H
#define BOOT_TRACE_H

#include <stdint.h>
#include "main.h"

#define BOOT_TRACE_MAGIC   0xB007C0DEu
#define BKP_TRACE          ((volatile uint32_t *)BKPSRAM_BASE)
#define BKP_TS(code)       (BKP_TRACE[64 + (code)])   /* per-stage arrival tick (ms) */

/* Call once, as early as possible in main() (aft
```

## 验证方式
```
$cli = "C:\Program Files (x86)\STMicroelectronics\STM32 ST-LINK Utility\ST-LINK Utility\ST-LINK_CLI.exe"
foreach($i in 0..7){
  $addr = 0x40024000 + ($i*4)
  $h = "0x{0:X8}" -f $addr
  $out = & $cli -
```

## 经验规则
使用宏定义消除魔术数字

---

# PCSE_077: lcd.c - 添加复位逻辑恢复初始状态

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Inc/boot_trace.h

## 修改原因
Now we have the full picture — and it **rewrites the diagnosis**:

| word | value | meaning |
|---|---|---|
| [1] boot_count | **1** | only ONE boot this power session |
| [2] iwdg_reset_count | **0**...

## 修改前代码
```c
			/* not ready yet: LT7580 still unstable -> reset & retry */
		}

		if (lcd_try >= 8)
```

## 修改后代码
```c
			/* not ready yet: LT7580 still unstable -> reset & retry */
		}
		BKP_TRACE[8] = (uint32_t)lcd_try;    /* SDRAM bring-up retries used */

		if (lcd_try >= 8)
```

## 验证方式
```
$cli = "C:\Program Files (x86)\STMicroelectronics\STM32 ST-LINK Utility\ST-LINK Utility\ST-LINK_CLI.exe"
foreach($i in 0..7){
  $addr = 0x40024000 + ($i*4)
  $h = "0x{0:X8}" -f $addr
  $out = & $cli -
```

## 经验规则
添加复位逻辑恢复初始状态

---

# PCSE_078: stm32f4xx_hal_conf.h - 使用DMA减轻CPU负担

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Inc/stm32f4xx_hal_conf.h

## 修改原因
明白，直接动手修，目标稳定 11s。基于已知：拖慢的两大头是①HAL tick 被饿死（确凿 bug）②渲染慢（SPI 只有 2.6MHz + 启动预渲染了两屏）。我一次性上四个修复，并顺带加 **DWT 真实时间戳**好当场验证效果。

**修复 1/5：HAL 心跳提到最高优先级（永不丢 tick）**

## 修改前代码
```c
#define  TICK_INT_PRIORITY            15U   /*!< tick interrupt priority */
```

## 修改后代码
```c
#define  TICK_INT_PRIORITY            0U   /*!< tick interrupt priority (highest: HAL timebase TIM4 must never be starved by the 1ms keypad / SPI-DMA IRQs, else HAL_GetTick loses ticks) */
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "Core\Src\ui_safehouse.c" -Pattern "void ui_fast_switch_to|s_snap_valid|void action_goto_dido" | Select-Object LineNumber, Li
```

## 经验规则
使用DMA减轻CPU负担

---

# PCSE_079: tim.c - 添加中断处理实现异步响应

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Inc/stm32f4xx_hal_conf.h

## 修改原因
明白，直接动手修，目标稳定 11s。基于已知：拖慢的两大头是①HAL tick 被饿死（确凿 bug）②渲染慢（SPI 只有 2.6MHz + 启动预渲染了两屏）。我一次性上四个修复，并顺带加 **DWT 真实时间戳**好当场验证效果。

**修复 1/5：HAL 心跳提到最高优先级（永不丢 tick）**

## 修改前代码
```c
    /* �ж����ã������ȼ����� LVGL ʹ�� */
    HAL_NVIC_SetPriority(TIM2_IRQn, 1, 0);
    HAL_NVIC_EnableIRQ(TIM2_IRQn);
```

## 修改后代码
```c
    /* Keypad scan is not time-critical: keep it BELOW the HAL timebase so it
       never starves the 1ms tick. (Was priority 1 = near-highest, which
       preempted TIM4/HAL_IncTick and made HAL_GetTick lose ~18%+ of ticks.) */
    HAL_NVIC_SetPriority(TIM2_IRQn, 6, 0);
    HAL_NVIC_EnableIRQ(TIM2_IRQn);
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "Core\Src\ui_safehouse.c" -Pattern "void ui_fast_switch_to|s_snap_valid|void action_goto_dido" | Select-Object LineNumber, Li
```

## 经验规则
添加中断处理实现异步响应

---

# PCSE_080: task_gui.c - 修复代码逻辑错误

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Inc/stm32f4xx_hal_conf.h

## 修改原因
明白，直接动手修，目标稳定 11s。基于已知：拖慢的两大头是①HAL tick 被饿死（确凿 bug）②渲染慢（SPI 只有 2.6MHz + 启动预渲染了两屏）。我一次性上四个修复，并顺带加 **DWT 真实时间戳**好当场验证效果。

**修复 1/5：HAL 心跳提到最高优先级（永不丢 tick）**

## 修改前代码
```c
    /* Pre-render BOTH pages behind the boot black screen so the first
       switch to DIDO is instant (adds ~3.7s black time, then backlight on) */
    ui_safehouse_prerender();
    Set_Backlight(100);
    BOOT_STAGE(ST_BACKLIGHT);
```

## 修改后代码
```c
    /* Pre-render ONLY the boot page (AIAO) so the backlight turns on ASAP.
       DIDO is rendered lazily on first switch. Full dual-page prerender added
       ~3.8s (one extra full-screen render + BTE) to every boot. */
    ui_safehouse_prerender_light();
    Set_Backlight(100);
    BOOT_STAGE(ST_BACKLIGHT);
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "Core\Src\ui_safehouse.c" -Pattern "void ui_fast_switch_to|s_snap_valid|void action_goto_dido" | Select-Object LineNumber, Li
```

## 经验规则
修复代码逻辑错误

---

# PCSE_081: ui_safehouse.c - 使用DMA减轻CPU负担

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/app/tasks/task_gui.c

## 修改原因
**修复 3/5：启动只预渲染 AIAO（DIDO 延迟到首次切换），砍掉一屏渲染**

## 修改前代码
```c
    int i;
    for (i = 0; i < 4; i++) { lv_tick_inc(30); lv_task_handler(); HAL_IWDG_Refresh(&hiwdg); }
    SPI_DMA_WaitIdle();
    LCD_BTE_Copy((unsigned long)Layer0, Layer1, (unsigned short)lcdx, (unsigned short)lcdy);
    s_snap_valid[0] = 1;
    s_cur_page = 0;
}
```

## 修改后代码
```c
    int i;
    for (i = 0; i < 4; i++) { lv_tick_inc(30); lv_task_handler(); HAL_IWDG_Refresh(&hiwdg); }
    BOOT_STAGE(ST_PRE_AIAO);
    SPI_DMA_WaitIdle();
    LCD_BTE_Copy((unsigned long)Layer0, Layer1, (unsigned short)lcdx, (unsigned short)lcdy);
    s_snap_valid[0] = 1;
    s_cur_page = 0;
    BOOT_STAGE(ST_PRE_BTE1);
}
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "Core\Src\ui_safehouse.c" -Pattern "void ui_fast_switch_to|s_snap_valid|void action_goto_dido" | Select-Object LineNumber, Li
```

## 经验规则
使用DMA减轻CPU负担

---

# PCSE_082: spi.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Src/ui_safehouse.c

## 修改原因
**修复 3/5：启动只预渲染 AIAO（DIDO 延迟到首次切换），砍掉一屏渲染**

## 修改前代码
```c
  hspi1.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_32; // �Ű����� SPI ֻ���� 2.625MHz
```

## 修改后代码
```c
  hspi1.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_16; // LCD SPI ~5.25MHz (was /32=2.6MHz; halves full-frame flush)
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "Core\Src\ui_safehouse.c" -Pattern "void ui_fast_switch_to|s_snap_valid|void action_goto_dido" | Select-Object LineNumber, Li
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_083: boot_trace.h - 使用宏定义消除魔术数字

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Src/spi.c

## 修改原因
**修复 4/5：SPI1 提速 2.6MHz → 5.25MHz（全屏刷新减半，先保守一档）**

## 修改前代码
```c
 *   [8] 0x40024020  sdram_retry_count  (LT7580 SDRAM bring-up retries used)
 *
 * Per-stage arrival timestamps (ms since boot, HAL_GetTick) live at
 *   0x40024000 + 4*(64 + stage_code)  i.e. base 0x40024100 + 4*code.
 * Read the whole timestamp table with ST-Link:
 *   -r32 0x40024100 32
 * The delta between consecutive stage timestamps reveals the slow stage.
 */
#ifndef BOOT_TRACE_H
#define BOOT_TRACE_H

#include <stdint.h>
#include "main.h"

#define BOOT_TRACE_MAGIC   0xB007C0DEu
#define BKP_TRACE          ((volatile uint32_t *)BKPSRAM_BASE)
#define BKP_TS(code)       (BKP_TRACE[64 + (code)])   /* per-stage arrival tick (ms) */

/* Call once, as early as possible in main() (after HAL_Init). */
void BootTrace_Init(void);

/* Record progress. cur_stage tracks live position; this-boot max is latched
   so a reset preserves "furthest reached" for the next boot to expose. Also
   timestamps the arrival at each stage so per-stage duration is measurable. */
#define BOOT_STAGE(n)         
```

## 修改后代码
```c
 *   [8] 0x40024020  sdram_retry_count  (LT7580 SDRAM bring-up retries used)
 *   [9] 0x40024024  DWT last cycle snapshot (internal)
 *  [10] 0x40024028  accumulated REAL ms (DWT-based, immune to lost ticks)
 *
 * Two per-stage arrival tables (index by stage_code):
 *   HAL-tick ms  at 0x40024100 + 4*code  (may run slow if ticks are lost)
 *   REAL    ms  at 0x40024200 + 4*code  (DWT cycle counter -> ground truth)
 * Read them with ST-Link:
 *   -r32 0x40024100 32     (tick ms)
 *   -r32 0x40024200 32     (REAL ms)   <-- trust these
 * The delta between consecutive REAL timestamps reveals the true slow stage.
 */
#ifndef BOOT_TRACE_H
#define BOOT_TRACE_H

#include <stdint.h>
#include "main.h"

#define BOOT_TRACE_MAGIC   0xB007C0DEu
#define BKP_TRACE          ((volatile uint32_t *)BKPSRAM_BASE)
#define BKP_TS(code)       (BKP_TRACE[64  + (code)])   /* per-stage HAL-tick arrival (ms) */
#define BKP_RT(code)       (BKP_TRACE[128 + (code)])   /* per-stage REAL arrival (ms, DWT) */

/* Call
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "Core\Src\ui_safehouse.c" -Pattern "void ui_fast_switch_to|s_snap_valid|void action_goto_dido" | Select-Object LineNumber, Li
```

## 经验规则
使用宏定义消除魔术数字

---

# PCSE_084: main.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Inc/boot_trace.h

## 修改原因
**修复 5/5：加 DWT 周期计数器（168MHz 硬件真时钟）打真实时间戳**，用来验证。先改 `boot_trace.h`:

## 修改前代码
```c
{
  __HAL_RCC_PWR_CLK_ENABLE();
  __HAL_RCC_BKPSRAM_CLK_ENABLE();
  HAL_PWR_EnableBkUpAccess();

  if (BKP_TRACE[0] != BOOT_TRACE_MAGIC)
  {
    int i;
    for (i = 0; i < 96; i++) BKP_TRACE[i] = 0;   /* header + timestamp table */
    BKP_TRACE[0] = BOOT_TRACE_MAGIC;      /* first power-on: init region */
  }

  BKP_TRACE[1]++;                          /* boot_count */
  {
    uint32_t csr = RCC->CSR;
    BKP_TRACE[4] = csr;                    /* raw reset-cause flags */
    if (csr & RCC_CSR_IWDGRSTF) BKP_TRACE[2]++;
    if (csr & RCC_CSR_SFTRSTF)  BKP_TRACE[3]++;
  }
  BKP_TRACE[6] = BKP_TRACE[7];             /* previous boot's furthest stage = hang point */
  BKP_TRACE[7] = 0;                        /* reset this-boot max */
  BKP_TRACE[5] = 0;                        /* reset live stage */
  {
    int i;
    for (i = 0; i < 32; i++) BKP_TS(i) = 0; /* clear per-stage timestamps */
  }

  __HAL_RCC_CLEAR_RESET_FLAGS();           /* RMVF: next boot's flags ar
```

## 修改后代码
```c
{
  __HAL_RCC_PWR_CLK_ENABLE();
  __HAL_RCC_BKPSRAM_CLK_ENABLE();
  HAL_PWR_EnableBkUpAccess();

  /* Enable the DWT cycle counter (168MHz) as a lost-tick-immune real clock. */
  CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
  DWT->CYCCNT = 0;
  DWT->CTRL  |= DWT_CTRL_CYCCNTENA_Msk;

  if (BKP_TRACE[0] != BOOT_TRACE_MAGIC)
  {
    int i;
    for (i = 0; i < 160; i++) BKP_TRACE[i] = 0;   /* header + tick + real tables */
    BKP_TRACE[0] = BOOT_TRACE_MAGIC;      /* first power-on: init region */
  }

  BKP_TRACE[1]++;                          /* boot_count */
  {
    uint32_t csr = RCC->CSR;
    BKP_TRACE[4] = csr;                    /* raw reset-cause flags */
    if (csr & RCC_CSR_IWDGRSTF) BKP_TRACE[2]++;
    if (csr & RCC_CSR_SFTRSTF)  BKP_TRACE[3]++;
  }
  BKP_TRACE[6] = BKP_TRACE[7];             /* previous boot's furthest stage = hang point */
  BKP_TRACE[7] = 0;                        /* reset this-boot max */
  BKP_TRACE[5] = 0;                      
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"; Select-String -Path "Core\Src\ui_safehouse.c" -Pattern "void ui_fast_switch_to|s_snap_valid|void action_goto_dido" | Select-Object LineNumber, Li
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_085: spi.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Src/spi.c

## 修改原因
SPI 提速搞黑屏了，山寨屏那注释是真的。改回 `/32`：

## 修改前代码
```c
  hspi1.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_16; // LCD SPI ~5.25MHz (was /32=2.6MHz; halves full-frame flush)
```

## 修改后代码
```c
  hspi1.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_32; // LCD SPI 2.625MHz (panel cannot run faster; /16 => black screen)
```

## 验证方式
```
$cli = "C:\Program Files (x86)\STMicroelectronics\STM32 ST-LINK Utility\ST-LINK Utility\ST-LINK_CLI.exe"
& $cli -List 2>&1 | Out-String
Write-Output "=== raw read test ==="
& $cli -c ID=0 SWD HOTPLUG 
```
输出: `STM32 ST-LINK CLI v3.3.0.0
STM32 ST-LINK Command Line Interface

--- Available ST-LINK Probes List ---

ST-LINK Probe 0:
     SN: B55B5A1A000000`

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_086: task_gui.c - 添加RTOS任务实现并发处理

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/app/tasks/task_gui.c

## 修改原因
删掉冗余的那次渲染（第 39-41 行），让 `prerender_light` 只渲染一遍：

## 修改前代码
```c
    /* Render first frame to SDRAM, then let SPI DMA flush */
    lv_task_handler();
    BOOT_STAGE(ST_GUI_FIRSTFRAME);
    vTaskDelay(pdMS_TO_TICKS(5));

    /* Pre-render ONLY the boot page (AIAO) so the backlight turns on ASAP.
       DIDO is rendered lazily on first switch. Full dual-page prerender added
       ~3.8s (one extra full-screen render + BTE) to every boot. */
    ui_safehouse_prerender_light();
    Set_Backlight(100);
```

## 修改后代码
```c
    /* Render the boot page (AIAO) ONCE and snapshot it for instant page
       switching, then turn the backlight on. NOTE: a standalone
       lv_task_handler() here previously rendered AIAO a first time, and
       ui_safehouse_prerender_light() then rendered it AGAIN -> ~3.4s of the
       boot was a redundant full-screen re-render (measured via DWT). */
    ui_safehouse_prerender_light();
    Set_Backlight(100);
```

## 验证方式
```
$cli = "C:\Program Files (x86)\STMicroelectronics\STM32 ST-LINK Utility\ST-LINK Utility\ST-LINK_CLI.exe"
& $cli -List 2>&1 | Out-String
Write-Output "=== raw read test ==="
& $cli -c ID=0 SWD HOTPLUG 
```
输出: `STM32 ST-LINK CLI v3.3.0.0
STM32 ST-LINK Command Line Interface

--- Available ST-LINK Probes List ---

ST-LINK Probe 0:
     SN: B55B5A1A000000`

## 经验规则
添加RTOS任务实现并发处理

---

# PCSE_087: task_gui.c - 添加RTOS任务实现并发处理

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/app/tasks/task_gui.c

## 修改原因
删掉冗余的那次渲染（第 39-41 行），让 `prerender_light` 只渲染一遍：

## 修改前代码
```c
    /* Render the boot page (AIAO) ONCE and snapshot it for instant page
       switching, then turn the backlight on. NOTE: a standalone
       lv_task_handler() here previously rendered AIAO a first time, and
       ui_safehouse_prerender_light() then rendered it AGAIN -> ~3.4s of the
       boot was a redundant full-screen re-render (measured via DWT). */
    ui_safehouse_prerender_light();
    Set_Backlight(100);
```

## 修改后代码
```c
    /* Pre-render BOTH pages (AIAO + DIDO) into their snapshot layers before
       the backlight turns on, so switching to either page is instant (no
       first-visit ~3.5s render freeze). Costs ~3.5s of extra black-screen
       boot time for the DIDO render. The earlier redundant standalone
       lv_task_handler() was removed, so this is one AIAO + one DIDO render. */
    ui_safehouse_prerender();
    Set_Backlight(100);
```

## 验证方式
```
$cli = "C:\Program Files (x86)\STMicroelectronics\STM32 ST-LINK Utility\ST-LINK Utility\ST-LINK_CLI.exe"
& $cli -List 2>&1 | Out-String
Write-Output "=== raw read test ==="
& $cli -c ID=0 SWD HOTPLUG 
```
输出: `STM32 ST-LINK CLI v3.3.0.0
STM32 ST-LINK Command Line Interface

--- Available ST-LINK Probes List ---

ST-LINK Probe 0:
     SN: B55B5A1A000000`

## 经验规则
添加RTOS任务实现并发处理

---

# PCSE_088: task_daq.c - 添加互斥量保护并发访问

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/app/tasks/task_daq.c

## 修改原因
关键数据：`ADC_BLOCK_SIZE=1000`，200kHz → **批完成中断 200 次/秒**，DAQ 任务每批要对 1000×8 求和+平均+写寄存器。显示根本不需要 200Hz 刷新。按你的思路：**采样(输入)保持 200kHz 不动，DAQ 任务处理(输出)降采到 ~20Hz**，CPU 负载降 ~10 倍。

改 `task_daq.c`——每批都计数（输入全保留），但只处理...

## 修改前代码
```c
    uint32_t last_internal_tick = osKernelGetTickCount() - DAQ_INTERNAL_ADC_UPDATE_MS;
    uint16_t ad7606_regs[8];

    for (;;)
    {
        if (osSemaphoreAcquire(DAQ_BatchCpltSemHandle, osWaitForever) == osOK)
        {
            uint8_t ready_buf_idx = (uint8_t)g_Buf_ReadyIdx;
            int32_t ch_sum[8] = {0};
```

## 修改后代码
```c
    uint32_t last_internal_tick = osKernelGetTickCount() - DAQ_INTERNAL_ADC_UPDATE_MS;
    uint16_t ad7606_regs[8];
    uint32_t proc_skip = 0;

    for (;;)
    {
        if (osSemaphoreAcquire(DAQ_BatchCpltSemHandle, osWaitForever) == osOK)
        {
            g_DAQ.sample_cnt += ADC_BLOCK_SIZE;   /* count every sampled batch (input kept at 200kHz) */

            /* Output decimation: only crunch/publish 1 of every N batches so
               the GUI is not starved. Skipped batches are just dropped. */
            if (++proc_skip < DAQ_PROC_DECIMATE) continue;
            proc_skip = 0;

            uint8_t ready_buf_idx = (uint8_t)g_Buf_ReadyIdx;
            int32_t ch_sum[8] = {0};
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"
Write-Output "=== functions in lcd_lt7580.c that contain HAL_Delay ==="
Select-String -Path "UserCode\lcd_lt7580.c" -Pattern "^\s*(void|int|uint\w
```
输出: `=== functions in lcd_lt7580.c that contain HAL_Delay ===

LineNumber Line                                            
---------- ----              `

## 经验规则
添加互斥量保护并发访问

---

# PCSE_089: task_daq.c - 使用DMA减轻CPU负担

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/app/tasks/task_daq.c

## 修改原因
关键数据：`ADC_BLOCK_SIZE=1000`，200kHz → **批完成中断 200 次/秒**，DAQ 任务每批要对 1000×8 求和+平均+写寄存器。显示根本不需要 200Hz 刷新。按你的思路：**采样(输入)保持 200kHz 不动，DAQ 任务处理(输出)降采到 ~20Hz**，CPU 负载降 ~10 倍。

改 `task_daq.c`——每批都计数（输入全保留），但只处理...

## 修改前代码
```c
#define SAMPLE_FREQ                 200000U
#define DAQ_INTERNAL_ADC_UPDATE_MS  100U

extern int16_t g_PingPong_Buf[2][ADC_BLOCK_SIZE][8];
```

## 修改后代码
```c
#define SAMPLE_FREQ                 200000U
#define DAQ_INTERNAL_ADC_UPDATE_MS  100U
/* Keep the 200kHz sampling (DMA input, ~free) but process only 1 of every N
   completed batches. At ADC_BLOCK_SIZE=1000 the batch-complete rate is 200Hz;
   the UI needs only a few Hz, so summing 1000x8 + register writes 200x/s was
   burning CPU and starving the GUI (laggy UI / slow boot renders). Decimating
   the OUTPUT to ~20Hz cuts that DAQ CPU ~10x without touching input rate. */
#define DAQ_PROC_DECIMATE           10U

extern int16_t g_PingPong_Buf[2][ADC_BLOCK_SIZE][8];
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"
Write-Output "=== functions in lcd_lt7580.c that contain HAL_Delay ==="
Select-String -Path "UserCode\lcd_lt7580.c" -Pattern "^\s*(void|int|uint\w
```
输出: `=== functions in lcd_lt7580.c that contain HAL_Delay ===

LineNumber Line                                            
---------- ----              `

## 经验规则
使用DMA减轻CPU负担

---

# PCSE_090: task_daq.c - 添加写入操作输出数据

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/app/tasks/task_daq.c

## 修改原因
关键数据：`ADC_BLOCK_SIZE=1000`，200kHz → **批完成中断 200 次/秒**，DAQ 任务每批要对 1000×8 求和+平均+写寄存器。显示根本不需要 200Hz 刷新。按你的思路：**采样(输入)保持 200kHz 不动，DAQ 任务处理(输出)降采到 ~20Hz**，CPU 负载降 ~10 倍。

改 `task_daq.c`——每批都计数（输入全保留），但只处理...

## 修改前代码
```c
                SysReg_WriteBlock(REG_AI_ADC1_IN0, internal_regs, 2);
            }

            g_DAQ.sample_cnt += ADC_BLOCK_SIZE;
        }
    }
}
```

## 修改后代码
```c
                SysReg_WriteBlock(REG_AI_ADC1_IN0, internal_regs, 2);
            }
        }
    }
}
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"
Write-Output "=== functions in lcd_lt7580.c that contain HAL_Delay ==="
Select-String -Path "UserCode\lcd_lt7580.c" -Pattern "^\s*(void|int|uint\w
```
输出: `=== functions in lcd_lt7580.c that contain HAL_Delay ===

LineNumber Line                                            
---------- ----              `

## 经验规则
添加写入操作输出数据

---

# PCSE_091: main.h - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Inc/main.h

## 修改原因
方向定了。先做核心修复（`delay_ms` 智能化，一改通吃），再顺手让 SPI 等待也让出 CPU。不做大重构（风险高），把确定的根因先干掉。

**1. `main.h` — delay_ms 智能化：**

## 修改前代码
```c
#include <stdio.h>
#define delay_ms(x) HAL_Delay(x)
extern void delay_us(uint32_t us); // ��������֮ǰд��΢����ʱ
```

## 修改后代码
```c
#include <stdio.h>
/* Scheduler-aware delay: yields the CPU via vTaskDelay once FreeRTOS is
   running (so a delay in the render/LCD path no longer busy-spins in
   HAL_Delay and starves other tasks); falls back to HAL_Delay during boot
   (before the scheduler) and in ISR context. */
extern void delay_ms_rtos(uint32_t ms);
#define delay_ms(x) delay_ms_rtos(x)
extern void delay_us(uint32_t us); // ��������֮ǰд��΢����ʱ
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"
Write-Output "=== functions in lcd_lt7580.c that contain HAL_Delay ==="
Select-String -Path "UserCode\lcd_lt7580.c" -Pattern "^\s*(void|int|uint\w
```
输出: `=== functions in lcd_lt7580.c that contain HAL_Delay ===

LineNumber Line                                            
---------- ----              `

## 经验规则
添加延时确保时序正确

---

# PCSE_092: main.c - 添加初始化调用确保外设就绪

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Inc/main.h

## 修改原因
方向定了。先做核心修复（`delay_ms` 智能化，一改通吃），再顺手让 SPI 等待也让出 CPU。不做大重构（风险高），把确定的根因先干掉。

**1. `main.h` — delay_ms 智能化：**

## 修改前代码
```c
void SystemClock_Config(void);
void MX_FREERTOS_Init(void);


```

## 修改后代码
```c
void SystemClock_Config(void);
void MX_FREERTOS_Init(void);

/* Scheduler-aware millisecond delay. Before the RTOS starts (boot) or inside
   an ISR it busy-waits via HAL_Delay (unchanged boot timing). Once the
   scheduler is running it uses osDelay, which BLOCKS the calling task and
   yields the CPU instead of spinning in HAL_Delay -> other tasks (and the
   GUI) run during the wait. Every delay_ms() in the codebase routes here. */
void delay_ms_rtos(uint32_t ms)
{
  if (__get_IPSR() == 0U && osKernelGetState() == osKernelRunning)
  {
    osDelay(ms ? ms : 1U);
  }
  else
  {
    HAL_Delay(ms);
  }
}


```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"
Write-Output "=== functions in lcd_lt7580.c that contain HAL_Delay ==="
Select-String -Path "UserCode\lcd_lt7580.c" -Pattern "^\s*(void|int|uint\w
```
输出: `=== functions in lcd_lt7580.c that contain HAL_Delay ===

LineNumber Line                                            
---------- ----              `

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_093: lv_port_disp.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Inc/main.h

## 修改原因
方向定了。先做核心修复（`delay_ms` 智能化，一改通吃），再顺手让 SPI 等待也让出 CPU。不做大重构（风险高），把确定的根因先干掉。

**1. `main.h` — delay_ms 智能化：**

## 修改前代码
```c
    uint32_t t0 = HAL_GetTick();
    while (spi_dma_is_transmitting)
    {
        if (HAL_GetTick() - t0 > 3000u)
            HAL_NVIC_SystemReset();
    }
}
```

## 修改后代码
```c
    uint32_t t0 = HAL_GetTick();
    while (spi_dma_is_transmitting)
    {
        if (HAL_GetTick() - t0 > 3000u)
            HAL_NVIC_SystemReset();
        /* yield the CPU while the DMA runs so other tasks aren't starved */
        if (__get_IPSR() == 0U && osKernelGetState() == osKernelRunning)
            osDelay(1);
    }
}
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"
Write-Output "=== functions in lcd_lt7580.c that contain HAL_Delay ==="
Select-String -Path "UserCode\lcd_lt7580.c" -Pattern "^\s*(void|int|uint\w
```
输出: `=== functions in lcd_lt7580.c that contain HAL_Delay ===

LineNumber Line                                            
---------- ----              `

## 经验规则
添加延时确保时序正确

---

# PCSE_094: lv_port_disp.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LVGL/examples/porting/lv_port_disp.c

## 修改原因
**3. `SPI_DMA_WaitIdle` — 等待时让出 CPU（不再纯忙等）:**

## 修改前代码
```c
    uint32_t _flush_t0 = HAL_GetTick();
    while(spi_dma_is_transmitting) {
        if (HAL_GetTick() - _flush_t0 > 3000u) HAL_NVIC_SystemReset();
        osThreadYield();
    }
```

## 修改后代码
```c
    uint32_t _flush_t0 = HAL_GetTick();
    while(spi_dma_is_transmitting) {
        if (HAL_GetTick() - _flush_t0 > 3000u) HAL_NVIC_SystemReset();
        /* sleep (not busy-spin) while the slow ~2.6MHz SPI DMA drains, so the
           GUI task frees the CPU for DAQ/IO/Modbus during the transfer */
        osDelay(1);
    }
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"
Write-Output "=== functions in lcd_lt7580.c that contain HAL_Delay ==="
Select-String -Path "UserCode\lcd_lt7580.c" -Pattern "^\s*(void|int|uint\w
```
输出: `=== functions in lcd_lt7580.c that contain HAL_Delay ===

LineNumber Line                                            
---------- ----              `

## 经验规则
添加延时确保时序正确

---

# PCSE_095: lv_port_disp.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LVGL/examples/porting/lv_port_disp.c

## 修改原因
5 处复位守卫。全部改成"超时就跳出继续"而非复位（避免复位循环，也避免死循环，用更大的超时兜底，IWDG 做最终后备）。先看 SPI_DMA_WaitIdle 上下文：

## 修改前代码
```c
void SPI_DMA_WaitIdle(void)
{
    uint32_t t0 = HAL_GetTick();
    while (spi_dma_is_transmitting)
    {
        if (HAL_GetTick() - t0 > 3000u)
            HAL_NVIC_SystemReset();
        /* yield the CPU while the DMA runs so other tasks aren't starved */
        if (__get_IPSR() == 0U && osKernelGetState() == osKernelRunning)
            osDelay(1);
    }
```

## 修改后代码
```c
void SPI_DMA_WaitIdle(void)
{
    uint32_t t0 = HAL_GetTick();
    while (spi_dma_is_transmitting)
    {
        /* Safety bound only: if a transfer is truly stuck, give up waiting and
           proceed (do NOT reboot -- an over-eager reset here caused boot loops
           when a legit slow render exceeded the threshold). IWDG is the final
           backstop for a real hang. */
        if (HAL_GetTick() - t0 > 8000u)
            break;
        /* yield the CPU while the DMA runs so other tasks aren't starved */
        if (__get_IPSR() == 0U && osKernelGetState() == osKernelRunning)
            osDelay(1);
    }
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"
Select-String -Path "LVGL\examples\porting\lv_port_disp.c","LCD\lcd.c" -Pattern "HAL_NVIC_SystemReset" | Select-Object Path, LineNumber, Line | Fo
```

## 经验规则
添加延时确保时序正确

---

# PCSE_096: lv_port_disp.c - 添加延时确保时序正确

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LVGL/examples/porting/lv_port_disp.c

## 修改原因
5 处复位守卫。全部改成"超时就跳出继续"而非复位（避免复位循环，也避免死循环，用更大的超时兜底，IWDG 做最终后备）。先看 SPI_DMA_WaitIdle 上下文：

## 修改前代码
```c
    uint32_t _flush_t0 = HAL_GetTick();
    while(spi_dma_is_transmitting) {
        if (HAL_GetTick() - _flush_t0 > 3000u) HAL_NVIC_SystemReset();
        /* sleep (not busy-spin) while the slow ~2.6MHz SPI DMA drains, so the
           GUI task frees the CPU for DAQ/IO/Modbus during the transfer */
        osDelay(1);
    }
```

## 修改后代码
```c
    uint32_t _flush_t0 = HAL_GetTick();
    while(spi_dma_is_transmitting) {
        if (HAL_GetTick() - _flush_t0 > 8000u) break;
        /* sleep (not busy-spin) while the slow ~2.6MHz SPI DMA drains, so the
           GUI task frees the CPU for DAQ/IO/Modbus during the transfer */
        osDelay(1);
    }
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"
Select-String -Path "LVGL\examples\porting\lv_port_disp.c","LCD\lcd.c" -Pattern "HAL_NVIC_SystemReset" | Select-Object Path, LineNumber, Line | Fo
```

## 经验规则
添加延时确保时序正确

---

# PCSE_097: lcd.c - 添加复位逻辑恢复初始状态

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LVGL/examples/porting/lv_port_disp.c

## 修改原因
5 处复位守卫。全部改成"超时就跳出继续"而非复位（避免复位循环，也避免死循环，用更大的超时兜底，IWDG 做最终后备）。先看 SPI_DMA_WaitIdle 上下文：

## 修改前代码
```c
		/* A wedged LT7580 BTE engine (unstable cold/warm-boot supply) would
		   spin here forever with the watchdog un-fed -> ~30s runtime IWDG
		   recovery. Reboot after 3s for a clean, stable-supply init. 3s is
		   far above a real BTE copy time, so normal operation never trips. */
		if ((HAL_GetTick() - t0) > 3000u)
			HAL_NVIC_SystemReset();
```

## 修改后代码
```c
		/* Safety bound only: if the BTE engine is truly wedged, give up and
		   proceed (do NOT reboot -- an over-eager reset here caused boot
		   loops during legit-slow renders). IWDG is the final backstop. */
		if ((HAL_GetTick() - t0) > 8000u)
			break;
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"
Select-String -Path "LVGL\examples\porting\lv_port_disp.c","LCD\lcd.c" -Pattern "HAL_NVIC_SystemReset" | Select-Object Path, LineNumber, Line | Fo
```

## 经验规则
添加复位逻辑恢复初始状态

---

# PCSE_098: lcd.c - 添加break提前退出循环

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LVGL/examples/porting/lv_port_disp.c

## 修改原因
Now `disp_flush` (~line 103):

## 修改前代码
```c
		IWDG->KR = 0x0000AAAAu;
		if ((HAL_GetTick() - t0) > 3000u)
			HAL_NVIC_SystemReset();
	}while((LCD_DataRead()&0x01)==0x01);
```

## 修改后代码
```c
		IWDG->KR = 0x0000AAAAu;
		if ((HAL_GetTick() - t0) > 8000u)
			break;
	}while((LCD_DataRead()&0x01)==0x01);
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"
Select-String -Path "LVGL\examples\porting\lv_port_disp.c","LCD\lcd.c" -Pattern "HAL_NVIC_SystemReset" | Select-Object Path, LineNumber, Line | Fo
```

## 经验规则
添加break提前退出循环

---

# PCSE_099: ui_safehouse.c - 添加NULL指针检查防止段错误

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/Core/Src/ui_safehouse.c

## 修改原因
`LCD_Init` 的 8 次重试耗尽复位（line 499）保留——它只在 SDRAM 真的 8 次起不来时才触发（本次 retry=0 没触发），是最后手段。渲染路径那 4 个误杀守卫已全部改为"超时跳出不复位"。

现在修渲染慢：DIDO 预渲染 13.9s，其中 `lv_scr_load(dido)` 触发了淡入动画导致多遍重绘。像 AIAO 那样杀掉动画：

## 修改前代码
```c
    /* render DIDO once into Layer0 (the ~3.7s, hidden behind boot black screen) */
    lv_scr_load(objects.dido);
    for (i = 0; i < 15; i++) { lv_tick_inc(30); lv_task_handler(); HAL_IWDG_Refresh(&hiwdg); }
```

## 修改后代码
```c
    /* render DIDO once into Layer0 (the ~3.7s, hidden behind boot black screen) */
    lv_scr_load(objects.dido);
    /* kill the EEZ screen-load fade so DIDO renders in ONE pass, not several
       (the animation was re-rendering the full screen every frame -> ~14s) */
    lv_anim_del(objects.dido, NULL);
    lv_obj_remove_local_style_prop(objects.dido, LV_STYLE_OPA, 0);
    for (i = 0; i < 15; i++) { lv_tick_inc(30); lv_task_handler(); HAL_IWDG_Refresh(&hiwdg); }
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"
Select-String -Path "LVGL\examples\porting\lv_port_disp.c","LCD\lcd.c" -Pattern "HAL_NVIC_SystemReset" | Select-Object Path, LineNumber, Line | Fo
```

## 经验规则
添加NULL指针检查防止段错误

---

# PCSE_100: lcd.c - 添加复位逻辑恢复初始状态

## 项目
HelpPort

## 修改文件
- D:/reasonix/HelpPort/HelpPort/HelpPort/LCD/lcd.c

## 修改原因
**关键发现**：`boot_count=27, soft=26` 且在涨 → **重启循环还在！** 而且渲染时间戳和之前**一模一样**。这说明：**你烧录的固件并不包含我最新的守卫删除改动**（`.axf` 是 11:52 的、早于我删守卫的编辑）——你烧的是旧构建。

两件事：

**1. 我把最后一个复位源也去掉**（`LCD_Init` 8 次重试耗尽也不再复位，改为继续+IWDG兜底...

## 修改前代码
```c
		BKP_TRACE[8] = (uint32_t)lcd_try;    /* SDRAM bring-up retries used */

		if (lcd_try >= 8)
		{
			/* never came up: reboot rather than run a dead display */
			HAL_NVIC_SystemReset();
		}
	}
		Select_LCD_DE_Mode();
```

## 修改后代码
```c
		BKP_TRACE[8] = (uint32_t)lcd_try;    /* SDRAM bring-up retries used */

		/* If SDRAM never reported ready after all retries, PROCEED anyway
		   (the settle + retries have given the LT7580 many seconds; it is
		   usually up by now). Do NOT reboot -- an over-eager reset here
		   caused warm-boot reboot loops. IWDG is the final backstop. */
	}
		Select_LCD_DE_Mode();
```

## 验证方式
```
Set-Location "D:\reasonix\HelpPort\HelpPort\HelpPort"
Get-ChildItem -Recurse -Include *.c | Select-String -Pattern "HAL_NVIC_SystemReset|NVIC_SystemReset" | Where-Object { $_.Path -notmatch "HAL_Drive
```

## 经验规则
添加复位逻辑恢复初始状态

---

# PCSE_101: Control_Relay.c - 使用HAL库函数实现硬件抽象

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control_Relay.c

## 修改原因
可以改了，改完之后记得从宏观和微观的角度进行review

## 修改前代码
```c
    Process_Single_SW(&SW1, SW1_GPIO_Port, SW1_Pin, SW_BIT_MAPPING[0]);
    Process_Single_SW(&SW2, SW2_GPIO_Port, SW2_Pin, SW_BIT_MAPPING[1]);
    Process_Single_SW(&SW3, SW3_GPIO_Port, SW3_Pin, SW_BIT_MAPPING[2]);
```

## 修改后代码
```c
    // SW1/SW2 改为船型开关：边沿检测，翻转对应照明 bit
    uint8_t sw1_now = HAL_GPIO_ReadPin(SW1_GPIO_Port, SW1_Pin);
    if (sw1_now != SW1_LastLevel) {
        SW1_LastLevel = sw1_now;
        PowerControlState ^= (1 << 4);  // bit4: 桌面照明
    }
    uint8_t sw2_now = HAL_GPIO_ReadPin(SW2_GPIO_Port, SW2_Pin);
    if (sw2_now != SW2_LastLevel) {
        SW2_LastLevel = sw2_now;
        PowerControlState ^= (1 << 5);  // bit5: 灯棚照明
    }

    Process_Single_SW(&SW3, SW3_GPIO_Port, SW3_Pin, SW_BIT_MAPPING[2]);
```

## 验证方式
```
chcp 65001 >nul && type "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\更新内容.txt"
```

## 经验规则
使用HAL库函数实现硬件抽象

---

# PCSE_102: Control_Relay.c - 修复代码逻辑错误

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control_Relay.c

## 修改原因
可以改了，改完之后记得从宏观和微观的角度进行review

## 修改前代码
```c
// 新增按键通道位映射表（放在文件开头全局变量区域）
static const uint8_t SW_BIT_MAPPING[] = {
    0,  // SW1 -> bit0
    1,  // SW2 -> bit1
    2,  // SW3 -> bit2
    3,  // SW4 -> bit3
    4,  // SW5 -> bit4
    5,  // SW6 -> bit5
    6,  // SW7 -> bit6
    7,  // SW8 -> bit7
    8,  // SW9 -> bit8
    9,  // SW10-> bit9
    10, // SW11-> bit10
    11  // SW12-> bit11
};
```

## 修改后代码
```c
// 新增按键通道位映射表（放在文件开头全局变量区域）
static const uint8_t SW_BIT_MAPPING[] = {
    0,  // SW1 -> bit0 (已改为船型开关，不再使用)
    1,  // SW2 -> bit1 (已改为船型开关，不再使用)
    2,  // SW3 -> bit2
    3,  // SW4 -> bit3
    4,  // SW5 -> bit4
    5,  // SW6 -> bit5
    6,  // SW7 -> bit6
    7,  // SW8 -> bit7
    8,  // SW9 -> bit8
    9,  // SW10-> bit9
    10, // SW11-> bit10
    11  // SW12-> bit11
};

// 船型开关边沿检测状态保存
static uint8_t SW1_LastLevel = 0;
static uint8_t SW2_LastLevel = 0;
```

## 验证方式
```
chcp 65001 >nul && type "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\更新内容.txt"
```

## 经验规则
修复代码逻辑错误

---

# PCSE_103: Control_Relay.c - 添加初始化调用确保外设就绪

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control_Relay.c

## 修改原因
可以改了，改完之后记得从宏观和微观的角度进行review

## 修改前代码
```c
// 船型开关边沿检测状态保存
static uint8_t SW1_LastLevel = 0;
static uint8_t SW2_LastLevel = 0;
```

## 修改后代码
```c
// 船型开关边沿检测状态保存
static uint8_t SW1_LastLevel = 0;
static uint8_t SW2_LastLevel = 0;
static uint8_t SW1_Debounce = 0;
static uint8_t SW2_Debounce = 0;
static uint8_t SW1_Initialized = 0;
static uint8_t SW2_Initialized = 0;
```

## 验证方式
```
chcp 65001 >nul && type "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\更新内容.txt"
```

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_104: Control_Relay.c - 精简/优化代码逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control_Relay.c

## 修改原因
可以添加防抖，和之前的防抖一样

## 修改前代码
```c
// 船型开关边沿检测状态保存
static uint8_t SW1_LastLevel = 0;
static uint8_t SW2_LastLevel = 0;
static uint8_t SW1_Debounce = 0;
static uint8_t SW2_Debounce = 0;
static uint8_t SW1_Initialized = 0;
static uint8_t SW2_Initialized = 0;
```

## 修改后代码
```c
// 船型开关边沿检测状态保存
static uint8_t SW1_LastLevel = 0;
static uint8_t SW2_LastLevel = 0;
static uint16_t SW1_DebounceTime = 0;
static uint16_t SW2_DebounceTime = 0;
```

## 验证方式
```
chcp 65001 >nul && type "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\更新内容.txt"
```

## 经验规则
精简/优化代码逻辑

---

# PCSE_105: Control_Relay.c - 添加计数/索引递增

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control_Relay.c

## 修改原因
可以添加防抖，和之前的防抖一样

## 修改前代码
```c
    // SW1/SW2 改为船型开关：边沿检测，翻转对应照明 bit
    uint8_t sw1_now = HAL_GPIO_ReadPin(SW1_GPIO_Port, SW1_Pin);
    if (sw1_now != SW1_LastLevel) {
        SW1_LastLevel = sw1_now;
        PowerControlState ^= (1 << 4);  // bit4: 桌面照明
    }
    uint8_t sw2_now = HAL_GPIO_ReadPin(SW2_GPIO_Port, SW2_Pin);
    if (sw2_now != SW2_LastLevel) {
        SW2_LastLevel = sw2_now;
        PowerControlState ^= (1 << 5);  // bit5: 灯棚照明
    }
```

## 修改后代码
```c
    // SW1/SW2 改为船型开关：边沿检测 + 防抖，翻转对应照明 bit
    uint8_t sw1_now = HAL_GPIO_ReadPin(SW1_GPIO_Port, SW1_Pin);
    if (sw1_now != SW1_LastLevel) {
        SW1_DebounceTime++;
        if (SW1_DebounceTime >= 20) {
            SW1_LastLevel = sw1_now;
            SW1_DebounceTime = 0;
            PowerControlState ^= (1 << 4);  // bit4: 桌面照明
        }
    } else {
        SW1_DebounceTime = 0;
    }

    uint8_t sw2_now = HAL_GPIO_ReadPin(SW2_GPIO_Port, SW2_Pin);
    if (sw2_now != SW2_LastLevel) {
        SW2_DebounceTime++;
        if (SW2_DebounceTime >= 20) {
            SW2_LastLevel = sw2_now;
            SW2_DebounceTime = 0;
            PowerControlState ^= (1 << 5);  // bit5: 灯棚照明
        }
    } else {
        SW2_DebounceTime = 0;
    }
```

## 验证方式
```
chcp 65001 >nul && type "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\更新内容.txt"
```

## 经验规则
添加计数/索引递增

---

# PCSE_106: Power.h - 添加extern声明实现跨文件访问

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.h

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
#ifndef __CTRLPOWER_H
#define __CTRLPOWER_H
#include "main.h"


uint16_t Power_control(uint16_t* power_ctrl);

#endif /* __POWER_H */
```

## 修改后代码
```c
#ifndef __CTRLPOWER_H
#define __CTRLPOWER_H
#include "main.h"

extern uint16_t lighting_state;  // bit0=桌面照明(PD14), bit1=灯棚照明(PD15)

uint16_t Power_control(uint16_t* power_ctrl);

#endif /* __POWER_H */
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加extern声明实现跨文件访问

---

# PCSE_107: Power.h - 使用宏定义消除魔术数字

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.h

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
#ifndef __CTRLPOWER_H
#define __CTRLPOWER_H
#include "main.h"

extern uint16_t lighting_state;  // bit0=桌面照明(PD14), bit1=灯棚照明(PD15)

uint16_t Power_control(uint16_t* power_ctrl);

```

## 修改后代码
```c
#ifndef __CTRLPOWER_H
#define __CTRLPOWER_H
#include "main.h"
#include "MMU.h"

extern uint16_t lighting_state;  // bit0=桌面照明(PD14), bit1=灯棚照明(PD15)

uint16_t Power_control(uint16_t* power_ctrl);

```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
使用宏定义消除魔术数字

---

# PCSE_108: Power.h - 添加extern声明实现跨文件访问

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.h

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
#ifndef __CTRLPOWER_H
#define __CTRLPOWER_H
#include "main.h"


uint16_t Power_control(uint16_t* power_ctrl);

#endif /* __POWER_H */
```

## 修改后代码
```c
#ifndef __CTRLPOWER_H
#define __CTRLPOWER_H
#include "main.h"
#include "MMU.h"

extern uint16_t lighting_state;  // bit0=桌面照明(PD14), bit1=灯棚照明(PD15)

uint16_t Power_control(uint16_t* power_ctrl);

#endif /* __POWER_H */
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加extern声明实现跨文件访问

---

# PCSE_109: Power.h - 添加状态机管理复杂逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.h

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
extern uint16_t lighting_state;  // bit0=桌面照明(PD14), bit1=灯棚照明(PD15)
```

## 修改后代码
```c
extern uint16_t lighting_state;  // bit4=桌面照明(PD14), bit5=灯棚照明(PD15)，与 Holding_Reg[46] bit4/bit5 直接对应
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加状态机管理复杂逻辑

---

# PCSE_110: Power.c - 使用移位操作实现位操作

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
#include "gpio.h"
#include "Power.h"
#include "Tapelight.h"
#include "Control_Relay.h"

uint16_t PowerControlState = 0;  // 全局状态变量

uint16_t Power_control(uint16_t* power_ctrl) // 电源控制
{
    static uint16_t lastState = 0xFFFF;
    
    if (!power_ctrl) {
        return (PowerControlState & 0x0FFF) | Get_GPIO_States(); // 合并软件控制位和硬件状态
    }

    // 新逻辑：上位机控制字直接设置低12位，高4位保持硬件状态
    uint16_t hardware_states = Get_GPIO_States() & 0xF000;
    uint16_t newState = (*power_ctrl & 0x0FFF) | hardware_states;
    
    if (newState != lastState) {
        PowerControlState = newState;
        lastState = newState;
        Wsc16packet.power_ctrl = newState;  // 同步完整状态
        Check_CT();  // 立即同步硬件状态
    }
    
    return PowerControlState;
}
```

## 修改后代码
```c
#include "gpio.h"
#include "Power.h"
#include "Tapelight.h"
#include "Control_Relay.h"

uint16_t PowerControlState = 0;  // 全局状态变量（bit0~bit11 控制插座/设备，不含灯控）
uint16_t lighting_state = 0;     // 灯控独立状态：bit0=桌面照明(PD14), bit1=灯棚照明(PD15)

uint16_t Power_control(uint16_t* power_ctrl) // 电源控制
{
    static uint16_t lastState = 0xFFFF;
    static uint16_t lastLighting = 0xFFFF;

    if (!power_ctrl) {
        // 回读时：power_ctrl 包含设备位(bit0~bit11) + 灯控位(lighting_state映射到bit4/bit5)
        uint16_t device_bits = PowerControlState & 0x0FFF;
        uint16_t light_bits = ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
        return device_bits | light_bits | (Get_GPIO_States() & 0xF000);
    }

    // 分离设备控制位和灯控位
    uint16_t device_cmd = *power_ctrl & 0x0FF0;  // bit0~bit3, bit6~bit11（不含bit4/bit5）
    uint16_t light_cmd = (*power_ctrl >> 4) & 0x03;  // bit4->bit0, bit5->bit1

    // 更新灯控状态（上位机写灯控位）
    if (light_cmd != lastLighting) {
        lighting_state = light
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
使用移位操作实现位操作

---

# PCSE_111: Power.c - 添加状态机管理复杂逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
    uint16_t newState = device_cmd | (Get_GPIO_States() & 0xF000);

    if (newState != lastState) {
        PowerControlState = newState;
        lastState = newState;
        Wsc16packet.power_ctrl = newState | ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
        Check_CT();  // 立即同步硬件状态
    }

    return PowerControlState;
```

## 修改后代码
```c
    uint16_t newState = device_cmd | (Get_GPIO_States() & 0xF000);

    if (newState != lastState) {
        PowerControlState = newState;
        lastState = newState;
    }

    // 同步 power_ctrl（含灯状态）供 Holding_Reg[46] 回读
    Wsc16packet.power_ctrl = (PowerControlState & 0x0FF0) | ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);

    return PowerControlState;
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加状态机管理复杂逻辑

---

# PCSE_112: Power.c - 添加extern声明实现跨文件访问

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
#include "main.h"
#include "gpio.h"
#include "Power.h"
#include "Tapelight.h"
#include "Control_Relay.h"

uint16_t PowerControlState = 0;  // 全局状态变量（bit0~bit11 控制插座/设备，不含灯控）
uint16_t lighting_state = 0;     // 灯控独立状态：bit0=桌面照明(PD14), bit1=灯棚照明(PD15)
```

## 修改后代码
```c
#include "main.h"
#include "gpio.h"
#include "Power.h"
#include "Tapelight.h"
#include "Control_Relay.h"
#include "MMU.h"

extern volatile ushort *Holding_Reg;

uint16_t PowerControlState = 0;  // 全局状态变量（bit0~bit11 控制插座/设备，不含灯控）
uint16_t lighting_state = 0;     // 灯控独立状态：bit0=桌面照明(PD14), bit1=灯棚照明(PD15)
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加extern声明实现跨文件访问

---

# PCSE_113: Power.c - 添加状态机管理复杂逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
    // 同步 power_ctrl（含灯状态）供 Holding_Reg[46] 回读
    Wsc16packet.power_ctrl = (PowerControlState & 0x0FF0) | ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);

    return PowerControlState;
```

## 修改后代码
```c
    // 同步 power_ctrl（含灯状态）供 Holding_Reg[46] 回读
    Wsc16packet.power_ctrl = (PowerControlState & 0x0FF0) | ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
    // 更新 Holding_Reg[46]，确保上位机读到的是实际灯状态
    Holding_Reg[46] = Wsc16packet.power_ctrl;

    return PowerControlState;
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加状态机管理复杂逻辑

---

# PCSE_114: Power.c - 使用位或操作设置标志位

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
    // 分离设备控制位和灯控位
    uint16_t device_cmd = *power_ctrl & 0x0FF0;  // bit0~bit3, bit6~bit11（不含bit4/bit5）
    uint16_t light_cmd = (*power_ctrl >> 4) & 0x03;  // bit4->bit0, bit5->bit1
```

## 修改后代码
```c
    // 分离设备控制位和灯控位
    // bit4/bit5 是灯控位，不属于 device_cmd
    uint16_t device_cmd = *power_ctrl & 0xF00F;  // bit0~bit3, bit12~bit15（不含bit4~bit11中的灯控位）
    // 但实际需要保留 bit6~bit11（插座7~12），所以用更精确的掩码
    device_cmd = *power_ctrl & 0xF00F;  // 只保留 bit0~bit3 和 bit12~bit15
    // bit6~bit11 也需要保留（插座7~12控制）
    device_cmd |= *power_ctrl & 0x0FC0;  // bit6~bit11
    uint16_t light_cmd = (power_ctrl[0] >> 4) & 0x03;  // bit4->bit0(桌面照明), bit5->bit1(灯棚照明)
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
使用位或操作设置标志位

---

# PCSE_115: Power.c - 精简/优化代码逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
    // 分离设备控制位和灯控位
    // bit4/bit5 是灯控位，不属于 device_cmd
    uint16_t device_cmd = *power_ctrl & 0xF00F;  // bit0~bit3, bit12~bit15（不含bit4~bit11中的灯控位）
    // 但实际需要保留 bit6~bit11（插座7~12），所以用更精确的掩码
    device_cmd = *power_ctrl & 0xF00F;  // 只保留 bit0~bit3 和 bit12~bit15
    // bit6~bit11 也需要保留（插座7~12控制）
    device_cmd |= *power_ctrl & 0x0FC0;  // bit6~bit11
    uint16_t light_cmd = (power_ctrl[0] >> 4) & 0x03;  // bit4->bit0(桌面照明), bit5->bit1(灯棚照明)
```

## 修改后代码
```c
    // 分离设备控制位和灯控位
    // bit4=桌面照明, bit5=灯棚照明（独立控制）
    // bit0~bit3=插座1~4, bit6~bit11=插座7~12
    uint16_t device_cmd = *power_ctrl & 0xFFCF;  // 保留 bit0~3, bit6~15（排除bit4/bit5）
    uint16_t light_cmd = (*power_ctrl >> 4) & 0x03;  // bit4->bit0(桌面照明), bit5->bit1(灯棚照明)
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
精简/优化代码逻辑

---

# PCSE_116: Power.c - 修复代码逻辑错误

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
    // 分离设备控制位和灯控位
    // bit4=桌面照明, bit5=灯棚照明（独立控制）
    // bit0~bit3=插座1~4, bit6~bit11=插座7~12
    uint16_t device_cmd = *power_ctrl & 0xFFCF;  // 保留 bit0~3, bit6~15（排除bit4/bit5）
    uint16_t light_cmd = (*power_ctrl >> 4) & 0x03;  // bit4->bit0(桌面照明), bit5->bit1(灯棚照明)
```

## 修改后代码
```c
    // 分离设备控制位和灯控位
    // bit4=桌面照明, bit5=灯棚照明（独立控制）
    // bit0~bit3=插座1~4, bit6~bit11=插座7~12
    uint16_t device_cmd = *power_ctrl & 0x0FCF;  // 保留 bit0~3, bit6~11（排除bit4/bit5灯控位）
    uint16_t light_cmd = (*power_ctrl >> 4) & 0x03;  // bit4->bit0(桌面照明), bit5->bit1(灯棚照明)
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
修复代码逻辑错误

---

# PCSE_117: Power.c - 添加状态机管理复杂逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
    // 同步 power_ctrl（含灯状态）供 Holding_Reg[46] 回读
    Wsc16packet.power_ctrl = (PowerControlState & 0x0FF0) | ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
```

## 修改后代码
```c
    // 同步 power_ctrl（含灯状态）供 Holding_Reg[46] 回读
    // PowerControlState 的 bit0~3=插座1~4, bit6~11=插座7~12（bit4/5始终=0）
    // lighting_state bit0->bit4(桌面照明), bit1->bit5(灯棚照明)
    Wsc16packet.power_ctrl = (PowerControlState & 0x0FCF) | ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加状态机管理复杂逻辑

---

# PCSE_118: Power.c - 添加状态机管理复杂逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
    if (!power_ctrl) {
        // 回读时：power_ctrl 包含设备位(bit0~bit11) + 灯控位(lighting_state映射到bit4/bit5)
        uint16_t device_bits = PowerControlState & 0x0FFF;
        uint16_t light_bits = ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
        return device_bits | light_bits | (Get_GPIO_States() & 0xF000);
    }
```

## 修改后代码
```c
    if (!power_ctrl) {
        // 回读时：power_ctrl 包含设备位 + 灯控位(lighting_state映射到bit4/bit5)
        uint16_t device_bits = PowerControlState & 0x0FCF;  // bit0~3, bit6~11
        uint16_t light_bits = ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
        return device_bits | light_bits | (Get_GPIO_States() & 0xF000);
    }
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加状态机管理复杂逻辑

---

# PCSE_119: Power.c - 添加状态机管理复杂逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
#include "MMU.h"

extern volatile ushort *Holding_Reg;

uint16_t PowerControlState = 0;
```

## 修改后代码
```c
#include "MMU.h"

uint16_t PowerControlState = 0;
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加状态机管理复杂逻辑

---

# PCSE_120: Power.c - 添加状态机管理复杂逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
uint16_t PowerControlState = 0;  // 全局状态变量（bit0~bit11 控制插座/设备，不含灯控）
uint16_t lighting_state = 0;     // 灯控独立状态：bit0=桌面照明(PD14), bit1=灯棚照明(PD15)

uint16_t Power_control(uint16_t* power_ctrl) // 电源控制
{
    static uint16_t lastState = 0xFFFF;
    static uint16_t lastLighting = 0xFFFF;

    if (!power_ctrl) {
        // 回读时：power_ctrl 包含设备位 + 灯控位(lighting_state映射到bit4/bit5)
        uint16_t device_bits = PowerControlState & 0x0FCF;  // bit0~3, bit6~11
        uint16_t light_bits = ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
        return device_bits | light_bits | (Get_GPIO_States() & 0xF000);
    }

    // 分离设备控制位和灯控位
    // bit4=桌面照明, bit5=灯棚照明（独立控制）
    // bit0~bit3=插座1~4, bit6~bit11=插座7~12
    uint16_t device_cmd = *power_ctrl & 0x0FCF;  // 保留 bit0~3, bit6~11（排除bit4/bit5灯控位）
    uint16_t light_cmd = (*power_ctrl >> 4) & 0x03;  // bit4->bit0(桌面照明), bit5->bit1(灯棚照明)

    // 更新灯控状态（上位机写灯控位）
    if (light_cmd != lastLighting) {
        lighting_state = light_cmd;
    
```

## 修改后代码
```c
uint16_t PowerControlState = 0;  // 全局状态变量（bit0~bit11 控制插座/设备，不含灯控）
uint16_t lighting_state = 0;     // 灯控独立状态：bit0=桌面照明(PD14), bit1=灯棚照明(PD15)

uint16_t Power_control(uint16_t* power_ctrl) // 电源控制（仅处理插座，不处理灯控）
{
    static uint16_t lastState = 0xFFFF;

    if (!power_ctrl) {
        // 回读时：只返回插座控制位（不含灯控）
        return (PowerControlState & 0x0FCF) | (Get_GPIO_States() & 0xF000);
    }

    // 只提取插座控制位（bit0~3, bit6~11），排除bit4/bit5灯控位
    uint16_t device_cmd = *power_ctrl & 0x0FCF;

    // 更新插座控制位
    uint16_t newState = device_cmd | (Get_GPIO_States() & 0xF000);

    if (newState != lastState) {
        PowerControlState = newState;
        lastState = newState;
    }

    return PowerControlState;
}
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加状态机管理复杂逻辑

---

# PCSE_121: Power.c - 添加状态机管理复杂逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
#include "gpio.h"
#include "Power.h"
#include "Tapelight.h"
#include "Control_Relay.h"

uint16_t PowerControlState = 0;  // 全局状态变量

uint16_t Power_control(uint16_t* power_ctrl) // 电源控制
{
    static uint16_t lastState = 0xFFFF;
    
    if (!power_ctrl) {
        return (PowerControlState & 0x0FFF) | Get_GPIO_States(); // 合并软件控制位和硬件状态
    }

    // 新逻辑：上位机控制字直接设置低12位，高4位保持硬件状态
    uint16_t hardware_states = Get_GPIO_States() & 0xF000;
    uint16_t newState = (*power_ctrl & 0x0FFF) | hardware_states;
    
    if (newState != lastState) {
        PowerControlState = newState;
        lastState = newState;
        Wsc16packet.power_ctrl = newState;  // 同步完整状态
        Check_CT();  // 立即同步硬件状态
    }
    
    return PowerControlState;
}
```

## 修改后代码
```c
#include "gpio.h"
#include "Power.h"
#include "Tapelight.h"
#include "Control_Relay.h"

uint16_t PowerControlState = 0;  // 全局状态变量（不含灯控）
uint16_t lighting_state = 0;     // 灯控独立状态：bit0=桌面照明(PD14), bit1=灯棚照明(PD15)

uint16_t Power_control(uint16_t* power_ctrl) // 电源控制（仅处理插座/设备位，不处理灯控）
{
    static uint16_t lastState = 0xFFFF;

    if (!power_ctrl) {
        return (PowerControlState & 0x0FFF) | Get_GPIO_States();
    }

    // 只提取设备控制位（bit0~3, bit6~11），排除bit4/bit5灯控位
    uint16_t device_cmd = *power_ctrl & 0x0FCF;
    uint16_t hardware_states = Get_GPIO_States() & 0xF000;
    uint16_t newState = device_cmd | hardware_states;

    if (newState != lastState) {
        PowerControlState = newState;
        lastState = newState;
    }

    return PowerControlState;
}
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加状态机管理复杂逻辑

---

# PCSE_122: Power.c - 添加状态机管理复杂逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Power.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
uint16_t lighting_state = 0;     // 灯控独立状态：bit0=桌面照明(PD14), bit1=灯棚照明(PD15)
```

## 修改后代码
```c
uint16_t lighting_state = 0;     // 灯控独立状态：bit4=桌面照明(PD14), bit5=灯棚照明(PD15)，与 Holding_Reg[46] bit4/bit5 直接对应
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加状态机管理复杂逻辑

---

# PCSE_123: Control_Relay.c - 添加extern声明实现跨文件访问

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control_Relay.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
#include "Control_Relay.h"
#include "main.h"
#include "gpio.h"
#include "power.h"
#include "485_Send.h"
```

## 修改后代码
```c
#include "Control_Relay.h"
#include "main.h"
#include "gpio.h"
#include "power.h"
#include "485_Send.h"

extern uint16_t lighting_state;  // 灯控独立状态
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加extern声明实现跨文件访问

---

# PCSE_124: Control_Relay.c - 使用三元运算符简化条件赋值

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control_Relay.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
    // SW1/SW2 改为船型开关：边沿检测 + 防抖，翻转对应照明 bit
    uint8_t sw1_now = HAL_GPIO_ReadPin(SW1_GPIO_Port, SW1_Pin);
    if (sw1_now != SW1_LastLevel) {
        SW1_DebounceTime++;
        if (SW1_DebounceTime >= 20) {
            SW1_LastLevel = sw1_now;
            SW1_DebounceTime = 0;
            PowerControlState ^= (1 << 4);  // bit4: 桌面照明
        }
    } else {
        SW1_DebounceTime = 0;
    }

    uint8_t sw2_now = HAL_GPIO_ReadPin(SW2_GPIO_Port, SW2_Pin);
    if (sw2_now != SW2_LastLevel) {
        SW2_DebounceTime++;
        if (SW2_DebounceTime >= 20) {
            SW2_LastLevel = sw2_now;
            SW2_DebounceTime = 0;
            PowerControlState ^= (1 << 5);  // bit5: 灯棚照明
        }
    } else {
        SW2_DebounceTime = 0;
    }
```

## 修改后代码
```c
    // SW1/SW2 船型开关：只翻转 lighting_state，不影响 PowerControlState
    uint8_t sw1_now = HAL_GPIO_ReadPin(SW1_GPIO_Port, SW1_Pin);
    if (sw1_now != SW1_LastLevel) {
        SW1_DebounceTime++;
        if (SW1_DebounceTime >= 20) {
            SW1_LastLevel = sw1_now;
            SW1_DebounceTime = 0;
            lighting_state ^= 0x01;  // bit0: 桌面照明（独立变量，不干扰 power_ctrl）
            // 直接输出到硬件
            Socket_Channel_Select(SOCKET_CH5, (lighting_state & 0x01) ? 1 : 0);
        }
    } else {
        SW1_DebounceTime = 0;
    }

    uint8_t sw2_now = HAL_GPIO_ReadPin(SW2_GPIO_Port, SW2_Pin);
    if (sw2_now != SW2_LastLevel) {
        SW2_DebounceTime++;
        if (SW2_DebounceTime >= 20) {
            SW2_LastLevel = sw2_now;
            SW2_DebounceTime = 0;
            lighting_state ^= 0x02;  // bit1: 灯棚照明（独立变量，不干扰 power_ctrl）
            // 直接输出到硬件
            Socket_Channel_Select(SOCKET_CH6, (lighting_state & 0x02) ? 1 : 0);
        }
    } else {
        SW2_DebounceTime = 0;
  
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
使用三元运算符简化条件赋值

---

# PCSE_125: Control_Relay.c - 添加状态机管理复杂逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control_Relay.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
    Socket_Channel_Select(SOCKET_CH5, (PowerControlState & 0x0010) ? 1 : 0);  // 修改为布尔判断 灯棚
    Socket_Channel_Select(SOCKET_CH6, (PowerControlState & 0x0020) ? 1 : 0);  // 修改为布尔判断
```

## 修改后代码
```c
    Socket_Channel_Select(SOCKET_CH5, (lighting_state & 0x01) ? 1 : 0);  // 桌面照明（独立状态）
    Socket_Channel_Select(SOCKET_CH6, (lighting_state & 0x02) ? 1 : 0);  // 灯棚照明（独立状态）
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加状态机管理复杂逻辑

---

# PCSE_126: Control_Relay.c - 使用移位操作实现位操作

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control_Relay.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
    PowerControlState = (PowerControlState & 0x0FFF) | (Get_GPIO_States() & 0xF000);
    Wsc16packet.power_ctrl = PowerControlState;  // 最终状态回传
```

## 修改后代码
```c
    PowerControlState = (PowerControlState & 0x0FFF) | (Get_GPIO_States() & 0xF000);
    // power_ctrl 回读：设备位 + 灯控位（lighting_state 映射回 bit4/bit5）
    Wsc16packet.power_ctrl = (PowerControlState & 0x0FF0) | ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
使用移位操作实现位操作

---

# PCSE_127: Control_Relay.c - 添加extern声明实现跨文件访问

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control_Relay.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
    // power_ctrl 回读：设备位 + 灯控位（lighting_state 映射回 bit4/bit5）
    Wsc16packet.power_ctrl = (PowerControlState & 0x0FF0) | ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
}
```

## 修改后代码
```c
    // power_ctrl 回读：设备位 + 灯控位（lighting_state 映射回 bit4/bit5）
    Wsc16packet.power_ctrl = (PowerControlState & 0x0FF0) | ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
    // 同步 Holding_Reg[46]，确保船型开关翻转后上位机读到最新灯状态
    extern volatile ushort *Holding_Reg;
    Holding_Reg[46] = Wsc16packet.power_ctrl;
}
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加extern声明实现跨文件访问

---

# PCSE_128: Control_Relay.c - 精简/优化代码逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control_Relay.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
    // 同步 Holding_Reg[46]，确保船型开关翻转后上位机读到最新灯状态
    extern volatile ushort *Holding_Reg;
    Holding_Reg[46] = Wsc16packet.power_ctrl;
```

## 修改后代码
```c
    // 同步 Holding_Reg[46]，确保船型开关翻转后上位机读到最新灯状态
    Holding_Reg[46] = Wsc16packet.power_ctrl;
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
精简/优化代码逻辑

---

# PCSE_129: Control_Relay.c - 添加状态机管理复杂逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control_Relay.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
    // power_ctrl 回读：设备位 + 灯控位（lighting_state 映射回 bit4/bit5）
    Wsc16packet.power_ctrl = (PowerControlState & 0x0FF0) | ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
```

## 修改后代码
```c
    // power_ctrl 回读：设备位 + 灯控位（lighting_state 映射回 bit4/bit5）
    Wsc16packet.power_ctrl = (PowerControlState & 0x0FCF) | ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加状态机管理复杂逻辑

---

# PCSE_130: Control_Relay.c - 添加static限定符减少符号暴露

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control_Relay.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
void Check_CT(void) 
{
    Process_Single_SW(&SW1, SW1_GPIO_Port, SW1_Pin, SW_BIT_MAPPING[0]);
    Process_Single_SW(&SW2, SW2_GPIO_Port, SW2_Pin, SW_BIT_MAPPING[1]);
    Process_Single_SW(&SW3, SW3_GPIO_Port, SW3_Pin, SW_BIT_MAPPING[2]);
    Process_Single_SW(&SW4, SW4_GPIO_Port, SW4_Pin, SW_BIT_MAPPING[3]);
    Process_Single_SW(&SW5, SW5_GPIO_Port, SW5_Pin, SW_BIT_MAPPING[4]);  // 新增位偏移参数
    Process_Single_SW(&SW6, SW6_GPIO_Port, SW6_Pin, SW_BIT_MAPPING[5]);  // 新增位偏移参数
    Process_Single_SW(&SW7, SW7_GPIO_Port, SW7_Pin, SW_BIT_MAPPING[6]);  // 新增位偏移参数
    Process_Single_SW(&SW8, SW8_GPIO_Port, SW8_Pin, SW_BIT_MAPPING[7]);  // 新增位偏移参数
    Process_Single_SW(&SW9, SW9_GPIO_Port, SW9_Pin, SW_BIT_MAPPING[8]);  // 新增位偏移参数
    Process_Single_SW(&SW10, SW10_GPIO_Port, SW10_Pin, SW_BIT_MAPPING[9]); // 新增位偏移参数
    Process_Single_SW(&SW11, SW11_GPIO_Port, SW11_Pin, SW_BIT_MAPPING[10]);// 新增位偏移参数
    Process_Single_SW(&SW12, SW12_GPIO_Port, SW12_Pin, SW_BIT_MAPPING[11]);// 新增位偏移参数
    // 保持未启
```

## 修改后代码
```c
// 船型开关边沿检测状态保存
static uint8_t SW1_LastLevel = 0;
static uint8_t SW2_LastLevel = 0;
static uint16_t SW1_DebounceTime = 0;
static uint16_t SW2_DebounceTime = 0;

void Check_CT(void)
{
    // SW1(PD9)/SW2(PD8) 船型开关：翻转 lighting_state，不影响 PowerControlState
    uint8_t sw1_now = HAL_GPIO_ReadPin(SW1_GPIO_Port, SW1_Pin);
    if (sw1_now != SW1_LastLevel) {
        SW1_DebounceTime++;
        if (SW1_DebounceTime >= 20) {
            SW1_LastLevel = sw1_now;
            SW1_DebounceTime = 0;
            lighting_state ^= 0x01;  // bit0: 桌面照明
            Socket_Channel_Select(SOCKET_CH5, (lighting_state & 0x01) ? 1 : 0);
        }
    } else {
        SW1_DebounceTime = 0;
    }

    uint8_t sw2_now = HAL_GPIO_ReadPin(SW2_GPIO_Port, SW2_Pin);
    if (sw2_now != SW2_LastLevel) {
        SW2_DebounceTime++;
        if (SW2_DebounceTime >= 20) {
            SW2_LastLevel = sw2_now;
            SW2_DebounceTime = 0;
            lighting_state ^= 0x02;  // bit1: 灯棚照明
            Socket_Channel_Sele
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加static限定符减少符号暴露

---

# PCSE_131: Control_Relay.c - 添加读取操作获取数据

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control_Relay.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
    // SW1(PD9)/SW2(PD8) 船型开关：翻转 lighting_state，不影响 PowerControlState
    uint8_t sw1_now = HAL_GPIO_ReadPin(SW1_GPIO_Port, SW1_Pin);
    if (sw1_now != SW1_LastLevel) {
        SW1_DebounceTime++;
        if (SW1_DebounceTime >= 20) {
            SW1_LastLevel = sw1_now;
            SW1_DebounceTime = 0;
            lighting_state ^= 0x01;  // bit0: 桌面照明
            Socket_Channel_Select(SOCKET_CH5, (lighting_state & 0x01) ? 1 : 0);
        }
    } else {
        SW1_DebounceTime = 0;
    }

    uint8_t sw2_now = HAL_GPIO_ReadPin(SW2_GPIO_Port, SW2_Pin);
    if (sw2_now != SW2_LastLevel) {
        SW2_DebounceTime++;
        if (SW2_DebounceTime >= 20) {
            SW2_LastLevel = sw2_now;
            SW2_DebounceTime = 0;
            lighting_state ^= 0x02;  // bit1: 灯棚照明
            Socket_Channel_Select(SOCKET_CH6, (lighting_state & 0x02) ? 1 : 0);
        }
    } else {
        SW2_DebounceTime = 0;
    }
```

## 修改后代码
```c
    // SW1(PD9)/SW2(PD8) 船型开关：翻转 lighting_state bit4/bit5
    uint8_t sw1_now = HAL_GPIO_ReadPin(SW1_GPIO_Port, SW1_Pin);
    if (sw1_now != SW1_LastLevel) {
        SW1_DebounceTime++;
        if (SW1_DebounceTime >= 20) {
            SW1_LastLevel = sw1_now;
            SW1_DebounceTime = 0;
            lighting_state ^= 0x0010;  // bit4: 桌面照明
            Socket_Channel_Select(SOCKET_CH5, (lighting_state & 0x0010) ? 1 : 0);
        }
    } else {
        SW1_DebounceTime = 0;
    }

    uint8_t sw2_now = HAL_GPIO_ReadPin(SW2_GPIO_Port, SW2_Pin);
    if (sw2_now != SW2_LastLevel) {
        SW2_DebounceTime++;
        if (SW2_DebounceTime >= 20) {
            SW2_LastLevel = sw2_now;
            SW2_DebounceTime = 0;
            lighting_state ^= 0x0020;  // bit5: 灯棚照明
            Socket_Channel_Select(SOCKET_CH6, (lighting_state & 0x0020) ? 1 : 0);
        }
    } else {
        SW2_DebounceTime = 0;
    }
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加读取操作获取数据

---

# PCSE_132: Control_Relay.c - 添加状态机管理复杂逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control_Relay.c

## 修改原因
好，现在开始实现修改：

## 修改前代码
```c
    // 同步 Holding_Reg[46]：设备位 + 灯控位（lighting_state 映射回 bit4/bit5）
    PowerControlState = (PowerControlState & 0x0FFF) | (Get_GPIO_States() & 0xF000);
    Wsc16packet.power_ctrl = (PowerControlState & 0x0FCF)
                           | ((lighting_state & 0x01) << 4)
                           | ((lighting_state & 0x02) << 4);
    Holding_Reg[46] = Wsc16packet.power_ctrl;
```

## 修改后代码
```c
    // 同步 Holding_Reg[46]：设备位 + 灯控位（lighting_state 的 bit4/bit5 直接对应）
    PowerControlState = (PowerControlState & 0x0FFF) | (Get_GPIO_States() & 0xF000);
    Wsc16packet.power_ctrl = (PowerControlState & 0x0FCF) | (lighting_state & 0x0030);
    Holding_Reg[46] = Wsc16packet.power_ctrl;
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\*.c" -Pattern "Holding_Reg|ModbusREG" | Select-Object -First 20
```

## 经验规则
添加状态机管理复杂逻辑

---

# PCSE_133: Control.c - 添加extern声明实现跨文件访问

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control.c

## 修改原因
需要修改的逻辑：在 `ModbusREG_update()` 中，处理完 power_ctrl 后，把 `lighting_state` 合并回 `Wsc16packet.power_ctrl`，然后写回 `Holding_Reg[46]`。

## 修改前代码
```c
 	// 33.��Դ����
	if (FCR_UserUpdate(RW, 46, 1, &power_ctrl))
	{
		Wsc16packet.power_ctrl = power_ctrl;
		SF += 1;
	}
```

## 修改后代码
```c
 	// 33.��Դ����
	if (FCR_UserUpdate(RW, 46, 1, &power_ctrl))
	{
		Wsc16packet.power_ctrl = power_ctrl;
		SF += 1;
	}
	// ͬ��ʵ��״̬�� Holding_Reg[46]����λ4/5 = lighting_state ʵ��ֵ
	{
		extern uint16_t lighting_state;
		uint16_t real_power_ctrl = (Wsc16packet.power_ctrl & 0xFF0) | ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
		Holding_Reg[46] = real_power_ctrl;
		Wsc16packet.power_ctrl = real_power_ctrl;
	}
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\MyModbus\MMU.c" -Pattern "Holding_Reg|FCR_UserUpdate|power_ctrl" | Select-Object -First 20
```

## 经验规则
添加extern声明实现跨文件访问

---

# PCSE_134: Control.c - 添加状态机管理复杂逻辑

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control.c

## 修改原因
需要修改的逻辑：在 `ModbusREG_update()` 中，处理完 power_ctrl 后，把 `lighting_state` 合并回 `Wsc16packet.power_ctrl`，然后写回 `Holding_Reg[46]`。

## 修改前代码
```c
	// ͬ��ʵ��״̬�� Holding_Reg[46]����λ4/5 = lighting_state ʵ��ֵ
	{
		extern uint16_t lighting_state;
		uint16_t real_power_ctrl = (Wsc16packet.power_ctrl & 0xFF0) | ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
		Holding_Reg[46] = real_power_ctrl;
		Wsc16packet.power_ctrl = real_power_ctrl;
	}
```

## 修改后代码
```c
	// ͬ��ʵ��״̬�� Holding_Reg[46]����λ4/5 = lighting_state ʵ��ֵ
	{
		extern uint16_t lighting_state;
		uint16_t real_power_ctrl = (Wsc16packet.power_ctrl & 0x0FF0) | ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
		Holding_Reg[46] = real_power_ctrl;
		Wsc16packet.power_ctrl = real_power_ctrl;
	}
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\MyModbus\MMU.c" -Pattern "Holding_Reg|FCR_UserUpdate|power_ctrl" | Select-Object -First 20
```

## 经验规则
添加状态机管理复杂逻辑

---

# PCSE_135: Control.c - 添加更新逻辑同步状态

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control.c

## 修改原因
需要修改的逻辑：在 `ModbusREG_update()` 中，处理完 power_ctrl 后，把 `lighting_state` 合并回 `Wsc16packet.power_ctrl`，然后写回 `Holding_Reg[46]`。

## 修改前代码
```c
 	// 33.��Դ����
	if (FCR_UserUpdate(RW, 46, 1, &power_ctrl))
	{
		Wsc16packet.power_ctrl = power_ctrl;
		SF += 1;
	}
	// ͬ��ʵ��״̬�� Holding_Reg[46]����λ4/5 = lighting_state ʵ��ֵ
	{
		extern uint16_t lighting_state;
		uint16_t real_power_ctrl = (Wsc16packet.power_ctrl & 0x0FF0) | ((lighting_state & 0x01) << 4) | ((lighting_state & 0x02) << 4);
		Holding_Reg[46] = real_power_ctrl;
		Wsc16packet.power_ctrl = real_power_ctrl;
	}
```

## 修改后代码
```c
 	// 33.��Դ����
	if (FCR_UserUpdate(RW, 46, 1, &power_ctrl))
	{
		Wsc16packet.power_ctrl = power_ctrl;
		SF += 1;
	}
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\MyModbus\MMU.c" -Pattern "Holding_Reg|FCR_UserUpdate|power_ctrl" | Select-Object -First 20
```

## 经验规则
添加更新逻辑同步状态

---

# PCSE_136: Control.c - 添加extern声明实现跨文件访问

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control.c

## 修改原因
需要修改的逻辑：在 `ModbusREG_update()` 中，处理完 power_ctrl 后，把 `lighting_state` 合并回 `Wsc16packet.power_ctrl`，然后写回 `Holding_Reg[46]`。

## 修改前代码
```c
 	// 33.��Դ����
	if (FCR_UserUpdate(RW, 46, 1, &power_ctrl))
	{
		Wsc16packet.power_ctrl = power_ctrl;
		SF += 1;
	}
```

## 修改后代码
```c
 	// 33.��Դ����
	if (FCR_UserUpdate(RW, 46, 1, &power_ctrl))
	{
		Wsc16packet.power_ctrl = power_ctrl;
		SF += 1;
		// 处理上位机写灯控位（bit4=桌面照明, bit5=灯棚照明）
		extern uint16_t lighting_state;
		uint16_t new_lighting = (power_ctrl >> 4) & 0x03;
		if (new_lighting != lighting_state) {
			lighting_state = new_lighting;
			// 直接输出到硬件
			Socket_Channel_Select(SOCKET_CH5, (lighting_state & 0x01) ? 1 : 0);
			Socket_Channel_Select(SOCKET_CH6, (lighting_state & 0x02) ? 1 : 0);
		}
	}
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\MyModbus\MMU.c" -Pattern "Holding_Reg|FCR_UserUpdate|power_ctrl" | Select-Object -First 20
```

## 经验规则
添加extern声明实现跨文件访问

---

# PCSE_137: Control.c - 添加发送操作输出数据

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control.c

## 修改原因
需要修改的逻辑：在 `ModbusREG_update()` 中，处理完 power_ctrl 后，把 `lighting_state` 合并回 `Wsc16packet.power_ctrl`，然后写回 `Holding_Reg[46]`。

## 修改前代码
```c
#include "HC_433.h"
#include "485_Send.h"
```

## 修改后代码
```c
#include "HC_433.h"
#include "485_Send.h"
#include "Control_Relay.h"
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\MyModbus\MMU.c" -Pattern "Holding_Reg|FCR_UserUpdate|power_ctrl" | Select-Object -First 20
```

## 经验规则
添加发送操作输出数据

---

# PCSE_138: Control.c - 添加检查确保状态正确

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control.c

## 修改原因
需要修改的逻辑：在 `ModbusREG_update()` 中，处理完 power_ctrl 后，把 `lighting_state` 合并回 `Wsc16packet.power_ctrl`，然后写回 `Holding_Reg[46]`。

## 修改前代码
```c
 	// 33.��Դ����
	if (FCR_UserUpdate(RW, 46, 1, &power_ctrl))
	{
		Wsc16packet.power_ctrl = power_ctrl;
		SF += 1;
		// 处理上位机写灯控位（bit4=桌面照明, bit5=灯棚照明）
		extern uint16_t lighting_state;
		uint16_t new_lighting = (power_ctrl >> 4) & 0x03;
		if (new_lighting != lighting_state) {
			lighting_state = new_lighting;
			// 直接输出到硬件
			Socket_Channel_Select(SOCKET_CH5, (lighting_state & 0x01) ? 1 : 0);
			Socket_Channel_Select(SOCKET_CH6, (lighting_state & 0x02) ? 1 : 0);
		}
	}
```

## 修改后代码
```c
 	// 33.��Դ����
	if (FCR_UserUpdate(RW, 46, 1, &power_ctrl))
	{
		// 只提取插座控制位（bit0~3, bit6~11），不处理灯控位（bit4/bit5）
		// 灯控位由 Check_CT() 根据 lighting_state 恢复
		Wsc16packet.power_ctrl = (power_ctrl & 0x0FCF) | (Wsc16packet.power_ctrl & 0x0030);
		SF += 1;
	}
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\MyModbus\MMU.c" -Pattern "Holding_Reg|FCR_UserUpdate|power_ctrl" | Select-Object -First 20
```

## 经验规则
添加检查确保状态正确

---

# PCSE_139: Control.c - 添加extern声明实现跨文件访问

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control.c

## 修改原因
需要修改的逻辑：在 `ModbusREG_update()` 中，处理完 power_ctrl 后，把 `lighting_state` 合并回 `Wsc16packet.power_ctrl`，然后写回 `Holding_Reg[46]`。

## 修改前代码
```c
 	// 33.��Դ����
	if (FCR_UserUpdate(RW, 46, 1, &power_ctrl))
	{
		// 只提取插座控制位（bit0~3, bit6~11），不处理灯控位（bit4/bit5）
		// 灯控位由 Check_CT() 根据 lighting_state 恢复
		Wsc16packet.power_ctrl = (power_ctrl & 0x0FCF) | (Wsc16packet.power_ctrl & 0x0030);
		SF += 1;
	}
```

## 修改后代码
```c
 	// 33.��Դ����
	if (FCR_UserUpdate(RW, 46, 1, &power_ctrl))
	{
		Wsc16packet.power_ctrl = power_ctrl;
		SF += 1;
		// 处理上位机写灯控位（bit4=桌面照明, bit5=灯棚照明）
		extern uint16_t lighting_state;
		uint16_t new_lighting = (power_ctrl >> 4) & 0x03;
		if (new_lighting != lighting_state) {
			lighting_state = new_lighting;
			// 直接输出到硬件
			Socket_Channel_Select(SOCKET_CH5, (lighting_state & 0x01) ? 1 : 0);
			Socket_Channel_Select(SOCKET_CH6, (lighting_state & 0x02) ? 1 : 0);
		}
	}
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\MyModbus\MMU.c" -Pattern "Holding_Reg|FCR_UserUpdate|power_ctrl" | Select-Object -First 20
```

## 经验规则
添加extern声明实现跨文件访问

---

# PCSE_140: Control.c - 添加extern声明实现跨文件访问

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control.c

## 修改原因
需要修改的逻辑：在 `ModbusREG_update()` 中，处理完 power_ctrl 后，把 `lighting_state` 合并回 `Wsc16packet.power_ctrl`，然后写回 `Holding_Reg[46]`。

## 修改前代码
```c
	if (FCR_UserUpdate(RW, 46, 1, &power_ctrl))
	{
		Wsc16packet.power_ctrl = power_ctrl;
		SF += 1;
	}
```

## 修改后代码
```c
	if (FCR_UserUpdate(RW, 46, 1, &power_ctrl))
	{
		Wsc16packet.power_ctrl = power_ctrl;
		SF += 1;
		// 处理上位机写灯控位（bit4=桌面照明, bit5=灯棚照明）
		extern uint16_t lighting_state;
		uint16_t new_lighting = (power_ctrl >> 4) & 0x03;
		if (new_lighting != lighting_state) {
			lighting_state = new_lighting;
			Socket_Channel_Select(SOCKET_CH5, (lighting_state & 0x01) ? 1 : 0);
			Socket_Channel_Select(SOCKET_CH6, (lighting_state & 0x02) ? 1 : 0);
		}
	}
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\MyModbus\MMU.c" -Pattern "Holding_Reg|FCR_UserUpdate|power_ctrl" | Select-Object -First 20
```

## 经验规则
添加extern声明实现跨文件访问

---

# PCSE_141: Control.c - 添加更新逻辑同步状态

## 项目
WSC16 - 2025-8-18

## 修改文件
- D:/reasonix/WSC-16改造/程序/CSQY-WS16-TP 2026-07-xx/CSQY-WS16-TP/CSQY-WS16-TP/User/Control.c

## 修改原因
需要修改的逻辑：在 `ModbusREG_update()` 中，处理完 power_ctrl 后，把 `lighting_state` 合并回 `Wsc16packet.power_ctrl`，然后写回 `Holding_Reg[46]`。

## 修改前代码
```c
	if (FCR_UserUpdate(RW, 46, 1, &power_ctrl))
	{
		Wsc16packet.power_ctrl = power_ctrl;
		SF += 1;
		// 处理上位机写灯控位（bit4=桌面照明, bit5=灯棚照明）
		extern uint16_t lighting_state;
		uint16_t new_lighting = (power_ctrl >> 4) & 0x03;
		if (new_lighting != lighting_state) {
			lighting_state = new_lighting;
			Socket_Channel_Select(SOCKET_CH5, (lighting_state & 0x01) ? 1 : 0);
			Socket_Channel_Select(SOCKET_CH6, (lighting_state & 0x02) ? 1 : 0);
		}
	}
```

## 修改后代码
```c
	if (FCR_UserUpdate(RW, 46, 1, &power_ctrl))
	{
		Wsc16packet.power_ctrl = power_ctrl;
		SF += 1;
		// 处理上位机写灯控位（lighting_state 的 bit4/bit5 直接对应 Holding_Reg[46] 的 bit4/bit5）
		extern uint16_t lighting_state;
		uint16_t new_lighting = power_ctrl & 0x0030;  // 提取 bit4/bit5
		if (new_lighting != lighting_state) {
			lighting_state = new_lighting;
			Socket_Channel_Select(SOCKET_CH5, (lighting_state & 0x0010) ? 1 : 0);
			Socket_Channel_Select(SOCKET_CH6, (lighting_state & 0x0020) ? 1 : 0);
		}
	}
```

## 验证方式
```
Select-String -Path "D:\reasonix\WSC-16改造\程序\CSQY-WS16-TP 2026-07-xx\CSQY-WS16-TP\CSQY-WS16-TP\User\MyModbus\MMU.c" -Pattern "Holding_Reg|FCR_UserUpdate|power_ctrl" | Select-Object -First 20
```

## 经验规则
添加更新逻辑同步状态

---

# PCSE_142: main.c - 修复代码逻辑错误

## 项目
单片机设备

## 修改文件
- D:/reasonix/单片机设备/STM32/STM32/chao_sheng_bo/MDK-ARM/RTE/_chao_sheng_bo/RTE_Components.h

## 修改原因
它把图存在了 `D:\reasonix\单片机设备\.code-review-graph\` 而非项目根目录。让我确认：

## 修改前代码
```c
u8 remote_ip[4]={192,168,10,104};
```

## 修改后代码
```c
u8 remote_ip[4]={192,168,10,2};
```

## 验证方式
```
Get-ChildItem -Path D:\reasonix\单片机设备 -Filter ".code-review-graph" -Recurse -Directory 2>$null | Select-Object FullName
```

## 经验规则
修复代码逻辑错误

---

# PCSE_143: main.c - 修复代码逻辑错误

## 项目
单片机设备

## 修改文件
- D:/reasonix/单片机设备/STM32/STM32/chao_sheng_bo/MDK-ARM/RTE/_chao_sheng_bo/RTE_Components.h

## 修改原因
它把图存在了 `D:\reasonix\单片机设备\.code-review-graph\` 而非项目根目录。让我确认：

## 修改前代码
```c
u8 remote_ip[4]={192,168,10,2};
```

## 修改后代码
```c
u8 remote_ip[4]={192,168,10,104};
```

## 验证方式
```
Get-ChildItem -Path D:\reasonix\单片机设备 -Filter ".code-review-graph" -Recurse -Directory 2>$null | Select-Object FullName
```

## 经验规则
修复代码逻辑错误

---

# PCSE_144: main.c - 修复代码逻辑错误

## 项目
单片机设备

## 修改文件
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/E27_4G&E33_NB/Core/Src/main.c

## 修改原因
这就是安信可透传云的 TCP 服务器。你的 NB-IoT 代码里写的 IP 是 `36.137.226.30:34935`，但透传云分配的端口是 **36755**。

需要改代码里的端口：

```c
u16 remote_com=34935;  // 改为 36755
```

同时确认 IP 地址是否还是 `36.137.226.30`（透传云显示的 IP 一致）。

### 测试步骤

1...

## 修改前代码
```c
u16 remote_com=34935;
```

## 修改后代码
```c
u16 remote_com=36755;
```

## 验证方式
```
Get-ChildItem -Recurse "D:\reasonix\单片机设备\STM32\STM32\STM32_extracted\STM32\E27_4G&E33_NB" -Include *.c | Where-Object { $_.Name -ne "main.c" } | Select-Object FullName
Write-Host "---"
Get-ChildItem 
```

## 经验规则
修复代码逻辑错误

---

# PCSE_145: main.c - 修复代码逻辑错误

## 项目
单片机设备

## 修改文件
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/E27_4G&E33_NB/Core/Src/main.c

## 修改原因
这就是安信可透传云的 TCP 服务器。你的 NB-IoT 代码里写的 IP 是 `36.137.226.30:34935`，但透传云分配的端口是 **36755**。

需要改代码里的端口：

```c
u16 remote_com=34935;  // 改为 36755
```

同时确认 IP 地址是否还是 `36.137.226.30`（透传云显示的 IP 一致）。

### 测试步骤

1...

## 修改前代码
```c
u16 remote_com=36755;
```

## 修改后代码
```c
u16 remote_com=42606;
```

## 验证方式
```
Get-ChildItem -Recurse "D:\reasonix\单片机设备\STM32\STM32\STM32_extracted\STM32\E27_4G&E33_NB" -Include *.c | Where-Object { $_.Name -ne "main.c" } | Select-Object FullName
Write-Host "---"
Get-ChildItem 
```

## 经验规则
修复代码逻辑错误

---

# PCSE_146: main.c - 修复代码逻辑错误

## 项目
单片机设备

## 修改文件
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/ETH/Core/Src/main.c

## 修改原因
你帮我把代码改为502吧

## 修改前代码
```c
	ESP01WIFI_wifi_sock_set("gzs2.4","1234567899",10000,2);
```

## 修改后代码
```c
	ESP01WIFI_wifi_sock_set("隐藏的网络","123456789",10000,2);
```

## 验证方式
```
Get-Content -Path "D:\reasonix\单片机设备\STM32\STM32\STM32_extracted\STM32\ETH\Core\Src\main.c" -TotalCount 70 | Select-Object -Last 20
```

## 经验规则
修复代码逻辑错误

---

# PCSE_147: main.c - 修复代码逻辑错误

## 项目
单片机设备

## 修改文件
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/ETH/Core/Src/main.c

## 修改原因
你帮我把代码改为502吧

## 修改前代码
```c
	ESP01WIFI_wifi_sock_set("隐藏的网络","123456789",10000,2);
```

## 修改后代码
```c
	ESP01WIFI_wifi_sock_set("ycdwl","123456789",10000,2);
```

## 验证方式
```
Get-Content -Path "D:\reasonix\单片机设备\STM32\STM32\STM32_extracted\STM32\ETH\Core\Src\main.c" -TotalCount 70 | Select-Object -Last 20
```

## 经验规则
修复代码逻辑错误

---

# PCSE_148: main.c - 修复代码逻辑错误

## 项目
单片机设备

## 修改文件
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/ETH/Core/Src/main.c

## 修改原因
你帮我把代码改为502吧

## 修改前代码
```c
	u8 ip[4]={192,168,10,104};
```

## 修改后代码
```c
	u8 ip[4]={192,168,3,2};
```

## 验证方式
```
Get-Content -Path "D:\reasonix\单片机设备\STM32\STM32\STM32_extracted\STM32\ETH\Core\Src\main.c" -TotalCount 70 | Select-Object -Last 20
```

## 经验规则
修复代码逻辑错误

---

# PCSE_149: main.c - 添加初始化调用确保外设就绪

## 项目
单片机设备

## 修改文件
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/ESP01WIFI/Core/Src/main.c

## 修改原因
esp01和你用网卡连接了同一个wifi

## 修改前代码
```c
	while(i!=0)
	{
		i--;
		E27_uart_init();
		E27_get_at(500,5);
		E27_sock_set(1,"TCP",remote_ip,remote_com,1000,3);
		E27_save_set(500);
		E27_rst(500);
		HAL_Delay(20000);
		{
			state=1;
			break;
		}
	}
```

## 修改后代码
```c
	while(i!=0)
	{
		i--;
		E27_uart_init();
		E27_get_at(500,5);
		E27_cmd_up("AT+CGDCONT=1,\"IP\",\"CMNET\"\r\n",500);
		E27_cmd_up("AT+CGACT=1,1\r\n",5000);
		E27_sock_set(1,"TCP",remote_ip,remote_com,1000,3);
		E27_save_set(500);
		E27_rst(500);
		HAL_Delay(30000);
		{
			state=1;
			break;
		}
	}
```

## 验证方式
```
python "D:\reasonix\单片机设备\STM32\STM32\STM32_extracted\STM32\ESP01WIFI\test\tcp_server_test.py"
```
输出: `(no output)

<shell_metadata>
shell tool terminated command after exceeding timeout 300000 ms. If this command is expected to take longer and is not w`

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_150: usart.c - 使用DMA减轻CPU负担

## 项目
单片机设备

## 修改文件
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/ESP01WIFI/Core/Src/main.c

## 修改原因
esp01和你用网卡连接了同一个wifi

## 修改前代码
```c
UART_HandleTypeDef huart7;
DMA_HandleTypeDef hdma_uart7_rx;
DMA_HandleTypeDef hdma_uart7_tx;
```

## 修改后代码
```c
UART_HandleTypeDef huart7;
UART_HandleTypeDef huart1;
DMA_HandleTypeDef hdma_uart7_rx;
DMA_HandleTypeDef hdma_uart7_tx;
```

## 验证方式
```
python "D:\reasonix\单片机设备\STM32\STM32\STM32_extracted\STM32\ESP01WIFI\test\tcp_server_test.py"
```
输出: `(no output)

<shell_metadata>
shell tool terminated command after exceeding timeout 300000 ms. If this command is expected to take longer and is not w`

## 经验规则
使用DMA减轻CPU负担

---

# PCSE_151: usart.c - 添加初始化调用确保外设就绪

## 项目
单片机设备

## 修改文件
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/ESP01WIFI/Core/Src/main.c

## 修改原因
esp01和你用网卡连接了同一个wifi

## 修改前代码
```c
void HAL_UART_MspInit(UART_HandleTypeDef* uartHandle)
{

  GPIO_InitTypeDef GPIO_InitStruct = {0};
  if(uartHandle->Instance==UART7)
  {
```

## 修改后代码
```c
void MX_USART1_UART_Init(void)
{
  huart1.Instance = USART1;
  huart1.Init.BaudRate = 9600;
  huart1.Init.WordLength = UART_WORDLENGTH_8B;
  huart1.Init.StopBits = UART_STOPBITS_1;
  huart1.Init.Parity = UART_PARITY_NONE;
  huart1.Init.Mode = UART_MODE_TX_RX;
  huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
  huart1.Init.OverSampling = UART_OVERSAMPLING_16;
  HAL_UART_Init(&huart1);
}

void HAL_UART_MspInit(UART_HandleTypeDef* uartHandle)
{

  GPIO_InitTypeDef GPIO_InitStruct = {0};
  if(uartHandle->Instance==USART1)
  {
    __HAL_RCC_USART1_CLK_ENABLE();
    __HAL_RCC_GPIOA_CLK_ENABLE();
    GPIO_InitStruct.Pin = GPIO_PIN_9|GPIO_PIN_10;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
    GPIO_InitStruct.Alternate = GPIO_AF7_USART1;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
  }
  else if(uartHandle->Instance==UART7)
  {
```

## 验证方式
```
python "D:\reasonix\单片机设备\STM32\STM32\STM32_extracted\STM32\ESP01WIFI\test\tcp_server_test.py"
```
输出: `(no output)

<shell_metadata>
shell tool terminated command after exceeding timeout 300000 ms. If this command is expected to take longer and is not w`

## 经验规则
添加初始化调用确保外设就绪

---

# PCSE_152: uart_fifo.c - 添加NULL指针检查防止段错误

## 项目
单片机设备

## 修改文件
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/ZigBee_EWM181/Core/Src/uart_fifo.c

## 修改原因
好，没文档。那用逻辑分析仪或串口调试先确认模块实际响应。

但代码里还有**确定性的软件 bug** 可以先修。让我先修 `HAL_UART_TxCpltCallback` 的 NULL 指针问题，然后给你一个测试方案来确认 AT 指令。

先修确定的 bug：

## 修改前代码
```c
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
	u8 i;
	for(i=0;i<uart_max;i++)
	{
		if(uart_data_[i]->uart==huart)
		{
			uart_data_[i]->uart_t_idle=0;
			break;
		}
	}
```

## 修改后代码
```c
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
	u8 i;
	for(i=0;i<uart_max;i++)
	{
		if(uart_data_[i]!=NULL && uart_data_[i]->uart==huart)
		{
			uart_data_[i]->uart_t_idle=0;
			break;
		}
	}
```

## 验证方式
```
python C:\Users\Lenovo\.agents\skills\debug-jlink\scripts\jlink_debugger.py --detect
```
输出: `Traceback (most recent call last):
  File "C:\Users\Lenovo\.agents\skills\debug-jlink\scripts\jlink_debugger.py", line 44, in <module>
    from tool`

## 经验规则
添加NULL指针检查防止段错误

---

# PCSE_153: uart_fifo.c - 添加NULL指针检查防止段错误

## 项目
单片机设备

## 修改文件
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/ZigBee_EWM181/Core/Src/uart_fifo.c

## 修改原因
好，没文档。那用逻辑分析仪或串口调试先确认模块实际响应。

但代码里还有**确定性的软件 bug** 可以先修。让我先修 `HAL_UART_TxCpltCallback` 的 NULL 指针问题，然后给你一个测试方案来确认 AT 指令。

先修确定的 bug：

## 修改前代码
```c
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t Size)
{
	u8 i;
	for(i=0;i<uart_max;i++)
	{
		if(uart_data_[i]->uart==huart)
		{
```

## 修改后代码
```c
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t Size)
{
	u8 i;
	for(i=0;i<uart_max;i++)
	{
		if(uart_data_[i]!=NULL && uart_data_[i]->uart==huart)
		{
```

## 验证方式
```
python C:\Users\Lenovo\.agents\skills\debug-jlink\scripts\jlink_debugger.py --detect
```
输出: `Traceback (most recent call last):
  File "C:\Users\Lenovo\.agents\skills\debug-jlink\scripts\jlink_debugger.py", line 44, in <module>
    from tool`

## 经验规则
添加NULL指针检查防止段错误

---

# PCSE_154: ZigBee_EWM181.c - 使用正确的长度计算避免缓冲区溢出

## 项目
单片机设备

## 修改文件
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/ZigBee_EWM181/Core/Src/uart_fifo.c

## 修改原因
好，没文档。那用逻辑分析仪或串口调试先确认模块实际响应。

但代码里还有**确定性的软件 bug** 可以先修。让我先修 `HAL_UART_TxCpltCallback` 的 NULL 指针问题，然后给你一个测试方案来确认 AT 指令。

先修确定的 bug：

## 修改前代码
```c
u8 ZigBee_EWM181_cmd_up(_ZigBee_EWM181_data *EWM181,char *data,char* ack,u16 duration)//命令发送
{
	strcpy((char*)EWM181->cmd_data,data);
	uart_fifo_t(&EWM181->uart, EWM181->cmd_data, strlen((char*)EWM181->cmd_data), 100);
	if(duration==0)
	{
		return 1;
	}
	return ZigBee_EWM181_ack_wait(EWM181,ack,duration);
```

## 修改后代码
```c
u8 ZigBee_EWM181_cmd_up(_ZigBee_EWM181_data *EWM181,char *data,char* ack,u16 duration)//命令发送
{
	strcpy((char*)EWM181->cmd_data,data);
	u16 len=strlen((char*)EWM181->cmd_data);
	if(len>0 && EWM181->cmd_data[len-1]!='\n')
	{
		EWM181->cmd_data[len]='\r';
		EWM181->cmd_data[len+1]='\n';
		EWM181->cmd_data[len+2]='\0';
		len+=2;
	}
	uart_fifo_t(&EWM181->uart, EWM181->cmd_data, len, 100);
	if(duration==0)
	{
		return 1;
	}
	return ZigBee_EWM181_ack_wait(EWM181,ack,duration);
```

## 验证方式
```
python C:\Users\Lenovo\.agents\skills\debug-jlink\scripts\jlink_debugger.py --detect
```
输出: `Traceback (most recent call last):
  File "C:\Users\Lenovo\.agents\skills\debug-jlink\scripts\jlink_debugger.py", line 44, in <module>
    from tool`

## 经验规则
使用正确的长度计算避免缓冲区溢出

---

# PCSE_155: ZigBee_EWM181.c - 添加初始化调用确保外设就绪

## 项目
单片机设备

## 修改文件
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/ZigBee_EWM181_TEST/Core/Inc/ZigBee_EWM181.h
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/ZigBee_EWM181_TEST/Core/Inc/adc.h
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/ZigBee_EWM181_TEST/Core/Inc/dma.h
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/ZigBee_EWM181_TEST/Core/Inc/fifo.h
- D:/reasonix/单片机设备/STM32/STM32/STM32_extracted/STM32/ZigBee_EWM181_TEST/Core/Inc/gpio.h
- ... 共109个文件

## 修改原因
你复制一份当前代码用来验证你的想法，不要在原本的代码上直接修改，你复制一份再改

## 修改前代码
```c
u8 ZigBee_EWM181_get_at(_ZigBee_EWM181_data *EWM181,u16 duration,u8 retry)//进入AT模式
{
	while(retry!=0)
	{
		if(ZigBee_EWM181_cmd_up(EWM181,"+AT","AT_MODE\r\n",duration)==1)
		{
			HAL_Delay(10);
			return 1;
		}
		retry--;
```

## 修改后代码
```c
u8 ZigBee_EWM181_get_at(_ZigBee_EWM181_data *EWM181,u16 duration,u8 retry)//进入AT模式
{
	while(retry!=0)
	{
		HAL_Delay(1000);
		if(ZigBee_EWM181_cmd_up(EWM181,"+++","AT_MODE\r\n",duration)==1)
		{
			HAL_Delay(10);
			return 1;
		}
		retry--;
```

## 验证方式
```
Copy-Item -Path "D:\reasonix\单片机设备\STM32\STM32\STM32_extracted\STM32\ZigBee_EWM181" -Destination "D:\reasonix\单片机设备\STM32\STM32\STM32_extracted\STM32\ZigBee_EWM181_TEST" -Recurse -Force; Test-Path "D:
```
输出: `True
`

## 经验规则
添加初始化调用确保外设就绪

---
