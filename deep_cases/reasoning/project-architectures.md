# 项目架构图与调用链

---

## 项目1: HelpPort (STM32F407VET6 + LVGL + FreeRTOS)

### 硬件平台
- MCU: STM32F407VET6 (Cortex-M4, 168MHz)
- LCD: LT7580 控制器, 1024×600 RGB565
- 触摸: GT9147 (软件 I2C)
- 采集: AD7606 8通道 200kHz (FSMC)
- 外设: UART5(RS485), USART3(TJC屏), SPI1(LCD/Flash)

### 五层架构
```
┌──────────────────────────────────────────────┐
│  app/tasks/         ← 业务任务               │
│   - GUI_Task (5ms)                           │
│   - Touch_Task (20ms)                        │
│   - DAQ_Task (50ms)                          │
│   - Comm_Task                                │
├──────────────────────────────────────────────┤
│  app/services/      ← 服务层                 │
│   - Power_Manager                            │
│   - Modbus_Handler                           │
├──────────────────────────────────────────────┤
│  app/drivers/       ← 驱动抽象层             │
│   - LCD Driver (LT7580)                      │
│   - CTP Driver (GT9147)                      │
│   - DAQ Driver (AD7606)                      │
├──────────────────────────────────────────────┤
│  app/hal/           ← HAL 适配层             │
│   - STM32 HAL 配置                           │
│   - 时钟/引脚/DMA 配置                       │
├──────────────────────────────────────────────┤
│  app/config/        ← 编译期配置             │
│   - 功能宏开关                               │
│   - 引脚定义                                 │
├──────────────────────────────────────────────┤
│  BSP                ← 板级支持包             │
│   - bsp_lcd.c                                │
│   - bsp_ad7606.c                             │
│   - bsp_ctp.c                                │
├──────────────────────────────────────────────┤
│  STM32 HAL + FreeRTOS + LVGL v8              │
└──────────────────────────────────────────────┘
```

### 启动时序
```
0.0s  IWDG init (reload=312, ~2.5s)
0.0s  SystemClock_Config (HSE 8MHz → PLL → 168MHz)
0.0s  外设初始化 (GPIO/SPI/FSMC/UART...)
1.5s  LCD 供电稳定窗口 (15×100ms, 每100ms喂狗)
2.4s  LCD_Reset (100ms low + 800ms high)
2.4s  LCD_PLL_Initial (PLL 锁 ~1ms + delay 10ms)
2.4s  LCD_SDRAM_initail → Check_SDRAM_Ready
2.6s  Init_Font (2.1MB SPI Flash 字库加载)
~11s  屏幕点亮
```

### 关键调用链
```
main()
  → HAL_Init()
  → SystemClock_Config()         // 168MHz
  → MX_GPIO_Init()
  → MX_SPI1_Init()               // LCD SPI, /32 = 2.625MHz
  → MX_FSMC_Init()               // AD7606 并行总线
  → MX_UART5_Init()              // RS485 Modbus
  → MX_USART3_Init()             // TJC 串口屏
  → MX_IWDG_Init()               // 2.5s 超时
  → LCD_Init()                   // 上电沉降+重试
      → LCD_Reset()
      → LCD_PLL_Initial()
      → LCD_SDRAM_initail()
      → Check_SDRAM_Ready()      // 500ms 超时
      → Init_Font()              // 2.1MB 字库加载
  → lv_init()
  → lv_port_disp_init()          // 显示驱动注册
  → lv_port_indev_init()         // 触摸驱动注册
  → osKernelStart()
      → GUI_Task()
          → lv_task_handler()    // 每 5ms
          → osDelay(5)
      → Touch_Task()
          → GT9147_ReadCoord()   // 软件 I2C
          → lv_indev_read_timer_cb()
          → osDelay(20)
      → DAQ_Task()
          → AD7606_Process()     // FSMC 读取
          → osDelay(50)
```

### 中断优先级
| 外设 | 优先级 | 说明 |
|------|--------|------|
| AD7606 BUSY | 5 | FSMC 采样完成 |
| USART5 | 6 | RS485 接收 |
| USART3 | 6 | TJC 屏接收 |
| SPI1 TX DMA | 7 | LCD 数据发送 |
| SysTick | 15 | FreeRTOS 时基 |

### 来源
- HelpPort - 嵌入式架构深度分析 (2026-07-22)

---

## 项目2: WSC-16 (STM32F103VET6 + FreeRTOS + Modbus)

### 硬件平台
- MCU: STM32F103VET6 (Cortex-M3, 72MHz)
- 通信: UART5(RS485 Modbus), USART2(433无线), USART3(TJC屏)
- 控制: GPIO 继电器, TIM3 倒计时
- 协议: Modbus RTU (9600/8/N/1)

### 架构
```
┌──────────────────────────────────────────────┐
│  应用层                                      │
│   - HC433_Task (433无线接收+倒计时控制)       │
│   - Modbus_Task (RS485 三色灯控制)            │
│   - Countdown_Task (TIM3 秒级倒计时)          │
├──────────────────────────────────────────────┤
│  协议层                                      │
│   - Modbus RTU (读/写寄存器)                  │
│   - GTI 协议 (433 无线)                       │
│   - TJC 协议 (串口屏)                         │
├──────────────────────────────────────────────┤
│  驱动层                                      │
│   - UART5 (RS485 + RTS 方向控制)              │
│   - USART2 (433 模块 1200bps)                 │
│   - USART3 (TJC 屏)                           │
│   - TIM3 (1Hz 倒计时中断)                     │
│   - GPIO (继电器控制)                          │
├──────────────────────────────────────────────┤
│  STM32 HAL + FreeRTOS                        │
└──────────────────────────────────────────────┘
```

### 关键调用链
```
main()
  → SystemClock_Config()         // 72MHz
  → MX_UART5_Init()              // RS485 9600
  → MX_USART2_Init()             // 433 模块 1200bps
  → MX_USART3_Init()             // TJC 屏 9600
  → MX_TIM3_Init()               // 1Hz 倒计时
  → osKernelStart()
      → HC433_Task()
          → 解析 GTI 协议帧
          → competition(LED_Flag) // 控制三色灯
          → osDelay(100)
      → Countdown_Task()
          → 等待 TIM3 通知
          → 倒计时减一
          → 结束 → 红灯+蜂鸣器

// 433 协议转发
USART2_RxIRQ()
  → 解析 GTI 帧
  → 提取倒计时时间
  → 转发到 RS485 (UART5)
  → competition(LED_Flag)

// Modbus 控制三色灯
RS485_Send(cmd)
  → RTS = TX
  → HAL_UART_Transmit(&huart5, cmd, len, 100)
  → RTS = RX
```

### Modbus 寄存器映射
| 地址 | 名称 | 说明 |
|------|------|------|
| 100 | 红灯 | 0=关, 1=开 |
| 101 | 绿灯 | 0=关, 1=开 |
| 102 | 黄灯 | 0=关, 1=开 |
| 103 | 蜂鸣器 | 0=关, 1=开 |
| 1000 | 全部开启 | 1=全开 |
| 1001 | 全部关闭 | 1=全关 |

### 来源
- WSC-16 - UART5 RS485 control logic (2026-06-23)

---

## 项目3: SX-MK-010 (STM32F103 + FreeRTOS + Modbus + TJC屏)

### 硬件平台
- MCU: STM32F103 (Cortex-M3)
- 显示: TJC 串口屏 (USART3)
- 通信: RS485 Modbus (继电器控制)
- 刷卡: MFRC522 RFID (13.56MHz)
- 电能计量: RN7302 (SPI)

### 架构
```
┌──────────────────────────────────────────────┐
│  应用层                                      │
│   - RFID_Task (刷卡识别)                      │
│   - Control_Task (继电器控制)                  │
│   - Modbus_Task (寄存器读写)                   │
├──────────────────────────────────────────────┤
│  协议层                                      │
│   - Modbus RTU (继电器/参数)                   │
│   - TJC 协议 (屏幕交互)                        │
│   - RFID 协议 (MFRC522)                        │
├──────────────────────────────────────────────┤
│  驱动层                                      │
│   - USART3 (TJC 屏)                           │
│   - UART4/5 (RS485)                           │
│   - SPI (RN7302 电能计量)                      │
│   - GPIO (继电器)                              │
├──────────────────────────────────────────────┤
│  STM32 HAL + FreeRTOS                        │
└──────────────────────────────────────────────┘
```

### 关键调用链
```
// 刷卡流程
RFID_Process()
  → MFRC522_ReadCard()
  → 检查卡状态 (card_status)
  → 启动/继续倒计时
  → 继电器吸合
  → Uart3_SendBuff("home.bt222.val=0")  // 更新屏幕

// 倒计时流程
TIM6_Callback()
  → countdown_time--
  → if countdown_time == 0:
      → competition(2)  // 红灯+蜂鸣器
      → osDelay(1000)
      → 继电器关闭
      → experiment_duration = 0  // 阻止重启动

// Modbus 停止
Modbus_WriteReg(26, 0)  // set_start_step=0
  → Power_Manager(0)
  → 继电器关闭
  → Uart3_SendBuff("home.bt222.val=1")  // 同步屏幕
  → experiment_duration = 0
```

### 来源
- SX-MK-010 - 触摸屏停止按钮继电器失效问题 (2026-07-10)

---

## 项目4: OV7670_TFT (STM32 + Keil MDK)

### 硬件平台
- MCU: STM32 (具体型号待确认)
- 摄像头: OV7670
- 显示: TFT LCD
- 开发环境: Keil MD5/MDK

### 构建流程
```powershell
# Keil 命令行编译
D:\apps\Keil_v5\UV4\UV4.exe -r "Project.uvprojx" -j0

# 检查编译产物
Test-Path "Objects\OV7670_TFT.axf"
```

### 来源
- OV7670_TFT - Keil 编译工程

---

## 项目5: WIFI_CAM (Android APK 逆向分析)

### 分析架构
```
APK (base.apk)
  ├── classes*.dex    → Java 代码 (jadx 反编译)
  ├── libCamera.so    → Native 代码 (ARM64)
  └── resources       → 资源文件

通信协议:
  - UDP 8080: 视频流控制
  - UDP 8090: 摄像头控制命令
  - TCP 8080: 握手/状态
```

### 协议逆向流程
```
1. 解压 APK → 提取 DEX/SO
2. jadx 反编译 DEX → 分析 Java 层逻辑
3. strings libCamera.so → 提取关键字符串
4. 搜索 Socket/UDP/TCP 相关函数
5. 分析 native 函数 → 确定命令格式
6. Python 脚本验证命令
```

### 关键命令格式
```
握手: A0 + 34字节 (36字节总)
拍照: AA 80 80 00 80 55
状态查询: 42 76
```

### 来源
- WIFI_CAM - 摄像头协议逆向分析

---

## 项目6: 通用 STM32 + LVGL 显示系统架构

### 显示数据流
```
LVGL 绘制
  → lv_task_handler() (5ms)
  → lv_disp_drv_t.flush_cb()
  → LCD_SetWindow(x1,y1,x2,y2)
  → HAL_SPI_Transmit_DMA(hspi1, color_p, size)
  → DMA 传输完成中断
  → lv_disp_flush_ready(drv)
```

### 触摸数据流
```
GT9147 触摸中断 (INT 引脚)
  → Touch_Task 唤醒
  → GT9147_ReadCoord() (软件 I2C)
  → lv_indev_data_t.point.x/y
  → LVGL 输入处理
  → 按钮/滑块事件
```

### 中断与任务协作
```
AD7606 200kHz 采样
  → FSMC BUSY 中断 (优先级 5)
  → DMA 读取 8 通道数据
  → 通知 DAQ_Task

GUI_Task (优先级 Normal)
  → lv_task_handler()
  → SPI DMA 发送
  → 与 AD7606 总线竞争
  → 降低 DAQ 频率到 20Hz 缓解
```

### 来源
- HelpPort - Caveman mode: ultra (代码重构) (2026-07-04)

---

## 项目7: 网络应用 (GameViewer) 路由控制

### 网络拓扑
```
┌─────────────────────────────────────────────┐
│  Windows PC                                  │
│  ┌─────────┐    ┌──────────────┐            │
│  │ GameViewer│    │ Clash Proxy  │            │
│  │ App      │    │ 127.0.0.1:7890│           │
│  └────┬─────┘    └──────────────┘            │
│       │                                      │
│  ┌────┴─────────────────────────────────┐   │
│  │ 路由选择                              │   │
│  │ WiFi (metric=5) vs Ethernet (metric=999)│  │
│  └────┬──────────────────────┬───────────┘   │
└───────┼──────────────────────┼───────────────┘
        │                      │
   ┌────┴────┐           ┌────┴────┐
   │ WiFi GW │           │ Eth GW  │
   │10.169.76│           │192.168.1│
   └────┬────┘           └────┬────┘
        │                      │
   ┌────┴──────────────────────┴────┐
   │ UU Server (117.147.201.71)     │
   └────────────────────────────────┘
```

### 路由控制方法
1. **接口 metric**: WiFi=5, Ethernet=999 → WiFi 优先
2. **防火墙规则**: 阻止 GameViewer 走 192.168.0.0/16
3. **ForceBindIP**: 强制绑定 WiFi IP 启动
4. **持久路由清理**: 删除注册表冲突路由

### 来源
- GameViewer - 网络优化与路由控制

---

## 通用调用链模板

### STM32 启动到 main()
```
Reset_Handler
  → SystemInit()                // 设置 VTOR/中断向量
  → __libc_init_array()         // C++ 全局构造
  → main()
      → HAL_Init()
          → HAL_InitTick()       // SysTick 配置
          → HAL_NVIC_SetPriorityGrouping()
      → SystemClock_Config()     // 系统时钟
      → MX_xxx_Init()            // 外设初始化
      → osKernelStart()          // 启动 FreeRTOS
          → StartScheduler()
          → 最高优先级任务运行
```

### FreeRTOS 任务切换
```
SysTick 中断
  → xPortPendSVHandler()
  → 保存当前任务上下文
  → 选择最高优先级就绪任务
  → 恢复新任务上下文
  → 新任务运行
```

### HAL 中断流程
```
外设中断
  → IRQHandler()
  → HAL_xxx_IRQHandler()
  → 清除中断标志
  → HAL_xxx_Callback()          // 用户回调
  → 释放信号量/通知任务
```

### 来源
- 通用 STM32 + FreeRTOS 架构知识
