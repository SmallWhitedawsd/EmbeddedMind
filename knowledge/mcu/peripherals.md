# 外设驱动经验（UART/DMA/I2C/SPI/ADC/Timer/GPIO/中断）

---

# UART + DMA 接收不定长数据

## 场景
USART 接收不定长帧数据（如 Modbus RTU、屏幕命令）。

## 现象
单字节中断接收在噪声环境下易丢帧或收错。

## 根因
单字节 `HAL_UART_Receive_IT` 每字节进一次中断，高负载时响应不及时。

## 解决方案
```c
// 方法1: IDLE 中断 + DMA 接收
// 1. 开启 IDLE 中断
__HAL_UART_ENABLE_IT(&huart3, UART_IT_IDLE);
// 2. DMA 循环接收
HAL_UART_Receive_DMA(&huart3, rx_buf, RX_BUF_SIZE);
// 3. 中断回调
void USART3_IRQHandler(void) {
    if (__HAL_UART_GET_FLAG(&huart3, UART_FLAG_IDLE)) {
        __HAL_UART_CLEAR_IDLEFLAG(&huart3);
        uint16_t len = RX_BUF_SIZE - __HAL_DMA_GET_COUNTER(huart3.hdmarx);
        // 处理 rx_buf[0..len-1]
    }
}

// 方法2: 超时帧检测（WSC-16 方案）
// USART5_RecService() 通过超时判断帧接收完毕
```

## 验证
- 发送已知长度数据，确认收到完整帧
- 高波特率 (115200+) 长时间运行不丢包

## 来源
WSC-16 - UART5 RS485 control logic (2026-06-23)

---

# RS485 半双工收发切换

## 场景
USB-RS485 适配器通信无响应。

## 现象
发送 Modbus 命令后设备无应答，但设备本身正常。

## 根因
RS485 半双工需切换收发方向。部分适配器用 RTS 控制方向。

## 解决方案
```powershell
# PowerShell 测试脚本
$sp = New-Object System.IO.Ports.SerialPort("COM4", 9600, None, 8, One)
$sp.ReadTimeout = 2000
$sp.Open()

# RTS high → 发送模式
$sp.RtsEnable = $true
Start-Sleep -Milliseconds 20
$sp.Write([byte[]]@(0x01,0x06,0x00,0x64,0x00,0x01,0x09,0xD5), 0, 8)

# RTS low → 接收模式
$sp.RtsEnable = $false
Start-Sleep -Milliseconds 500

if ($sp.BytesToRead -gt 0) {
    $recv = [byte[]]::new($sp.BytesToRead)
    $sp.Read($recv, 0, $recv.Length)
}
$sp.Close()
```

## 验证
- 发送后收到回显（Modbus 写命令回显相同数据）
- 示波器看 RTS 在 TX 期间为高，TX 后变低

## 来源
WSC-16 - Serial protocol debug no response (2026-06-23)

---

# SPI 信号完整性与波特率

## 场景
SPI 驱动 LCD 屏，提高波特率后黑屏。

## 现象
SPI1 从 /32 (2.625MHz) 改到 /16 (5.25MHz) 后屏幕黑屏。

## 根因
20cm FPC 软排线 + 高速信号 → 信号反射/振铃 → 通信失败。

## 解决方案
```c
// 方案1: 降低 GPIO 压摆率再试高速
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;  // 降低压摆率减少反射
// 然后 BaudRatePrescaler = SPI_BAUDRATEPRESCALER_16

// 方案2: 杜邦线极限约 2.6MHz，FPC 排线可更高
// 杜邦线: /32 = 2.625MHz
// FPC 20cm: 可尝试 /16 但需降 GPIO speed
```

## 验证
- 屏幕正常显示
- 示波器看 SCK 波形无明显过冲/振铃

## 来源
HelpPort - 拉取 Ponytail 作为技能使用 (2026-06-25)

---

# 软件 I2C 驱动触摸芯片

## 场景
GT9147 触摸芯片使用软件 I2C（硬件 I2C 引脚被占用）。

## 现象
首次上电触屏卡顿，多次上电后正常。

## 根因
- AD7606 FSMC 200kHz 采样 + DMA → 总线竞争
- 软件 I2C 在 `taskENTER_CRITICAL` 下被 AD7606 ISR 打断
- 首次上电 SPI DMA/FSMC 总线竞争 → 触摸异常

## 解决方案
```c
// 1. 移除 taskENTER_CRITICAL（AD7606 ISR 优先级5可打断 I2C）
// lv_port_indev.c 中:
// 删除 taskENTER_CRITICAL / taskEXIT_CRITICAL

// 2. 触屏轮询周期从 10ms 改为 20ms
// lv_task_handler 从每1ms改为每5ms
```

## 验证
- 连续上电 10 次触摸均正常
- AD7606 采样不影响触摸响应

## 来源
HelpPort - Caveman mode: ultra (代码重构) (2026-07-04)

---

# FSMC 驱动 AD7606 并行采集

## 场景
8 通道 AD7606 并行数据采集，200kHz 采样率。

## 现象
FSMC 总线与 SPI/DMA 竞争导致其他外设异常。

## 根因
FSMC 并行接口持续占用总线带宽。

## 解决方案
```c
// FSMC 配置为 NOR/SRAM 模式
// AD7606 数据就绪 (BUSY) → 触发中断 → DMA 读取
// 或使用定时器触发转换 + DMA 读取

// 降低采样率减少总线占用
// 200Hz → 20Hz (task_daq.c 中修改)
```

## 验证
- 8 通道数据正确读取
- 不影响 SPI LCD 和 I2C 触摸

## 来源
HelpPort - Caveman mode: ultra (代码重构) (2026-07-04)

---

# TIM 定时器中断用于倒计时

## 场景
需要精确秒级倒计时（如 120s 倒计时控制三色灯）。

## 现象
倒计时精度受 FreeRTOS 任务调度影响。

## 根因
`osDelay(1000)` 受任务切换影响有偏差。

## 解决方案
```c
// TIM3 配置为 1Hz 中断
htim3.Instance = TIM3;
htim3.Init.Prescaler = 8400 - 1;  // 84MHz/8400 = 10kHz
htim3.Init.Period = 10000 - 1;    // 10kHz/10000 = 1Hz
HAL_TIM_Base_Start_IT(&htim3);

// 中断回调
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (htim->Instance == TIM3) {
        if (HC433.Counter_Start && Wsc16packet.countdown_time > 0) {
            Wsc16packet.countdown_time--;
        }
    }
}
```

## 验证
- 用秒表对比，120s 误差 < 1s
- 倒计时结束触发回调正确

## 来源
WSC-16 - UART5 RS485 control logic (2026-06-23)

---

# GPIO 控制继电器

## 场景
通过 GPIO 控制继电器吸合/释放。

## 现象
继电器启动后无法停止。

## 根因
启动和停止共用一个引脚，或停止逻辑未正确执行。

## 解决方案
```c
// 定义
#define Start_Pin GPIO_PIN_15
#define Start_GPIO_Port GPIOB
#define stop_Pin GPIO_PIN_6
#define stop_GPIO_Port GPIOC

// 启动
HAL_GPIO_WritePin(Start_GPIO_Port, Start_Pin, GPIO_PIN_SET);    // 吸合
HAL_GPIO_WritePin(stop_GPIO_Port, stop_Pin, GPIO_PIN_RESET);

// 停止
HAL_GPIO_WritePin(stop_GPIO_Port, stop_Pin, GPIO_PIN_SET);      // 释放
HAL_GPIO_WritePin(Start_GPIO_Port, Start_Pin, GPIO_PIN_RESET);
HAL_TIM_PWM_Start(&htim4, TIM_CHANNEL_3);  // 反向PWM释放继电器
```

## 验证
- 启动命令 → 继电器吸合（听到咔嗒声）
- 停止命令 → 继电器释放

## 来源
SX-MK-010 - 查找单相继电器停止bug (2026-07-10)

---

# ADC 内部通道采集

## 场景
STM32F407 内部 ADC 采集模拟信号（如 2AI）。

## 现象
ADC 读数不准或跳变大。

## 根因
ADC 时钟过快、采样时间不足、电源噪声。

## 解决方案
```c
// ADC 配置
hadc1.Instance = ADC1;
hadc1.Init.ClockPrescaler = ADC_CLOCK_SYNC_PCLK_DIV4;  // 21MHz (APB2=84MHz)
hadc1.Init.Resolution = ADC_RESOLUTION_12B;
hadc1.Init.ScanConvMode = DISABLE;
hadc1.Init.ContinuousConvMode = ENABLE;
hadc1.Init.SamplingTime = ADC_SAMPLETIME_480CYCLES;  // 长采样时间提高精度

// 软件滤波
uint16_t adc_avg(uint8_t times) {
    uint32_t sum = 0;
    for (int i = 0; i < times; i++) {
        HAL_ADC_Start(&hadc1);
        HAL_ADC_PollForConversion(&hadc1, 10);
        sum += HAL_ADC_GetValue(&hadc1);
    }
    return sum / times;
}
```

## 验证
- 输入已知电压，读数误差 < 1%
- 多次采样取平均后跳变 < 2 LSB

## 来源
HelpPort - 拉取 Ponytail 作为技能使用 (2026-06-25)

---

# SPI 驱动 RN7302 电能计量芯片

## 场景
RN7302 电能计量芯片通过 SPI 读取电压/电流/功率。

## 现象
电流读数与实际不符。

## 根因
校准系数 (Kic/Kct) 未正确设置或三相/单相系数混用。

## 解决方案
```c
// 单相: Isum = Isum_REG * Kic * Kct / 1000
// 三相: Isum = Isum_REG * Kisum * Kct / 1000

// RN7302 初始化
void RN7302_Init(void) {
    // SPI 配置: CPOL=Low, CPHA=1Edge (Mode 0)
    // 写校准时系数
    RN7302_WriteReg(0x10, Kic);   // 电流校准
    RN7302_WriteReg(0x11, Kct);   // CT 变比
}

// 读取电流
float RN7302_ReadCurrent(void) {
    uint32_t reg = RN7302_ReadReg(0x22);  // 电流有效值寄存器
    return reg * Kic * Kct / 1000.0f;
}
```

## 验证
- 输入已知电流，读数误差 < 2%
- 三相版本使用 Kisum 而非 Kic

## 来源
SX-MK-010 - 克隆并对比NEW和yuan仓库 (2026-07-14)
