# WSC-16

## 硬件配置

| 参数 | 值 |
|------|-----|
| MCU | STM32F103 (72MHz) |
| 485三色灯 | Modbus RTU, 9600 8N1, 地址0x01 |
| HC-433无线模块 | USART2, 1200波特率 |
| TJC串口屏 | USART3 |
| 灯带控制 | UART5 (PC12 TX, PD2 RX), 9600 8N1 |

### 关键引脚定义

**UART5 - 485三色灯 (PC12/PD2)**:
| 信号 | 引脚 | 模式 |
|------|------|------|
| TX | PC12 | GPIO_MODE_AF_PP |
| RX | PD2 | GPIO_MODE_INPUT |
| 中断 | UART5_IRQn | 优先级5 |

**USART2 - HC-433模块**: 1200波特率

**USART3 - TJC触摸屏**: HMI串口屏

## 软件架构

```
┌────────────────────────────────────┐
│  User/Control.c     ← 业务逻辑     │
│  User/HC_433.c      ← 433协议解析  │
│  User/Tapelight.c   ← 485灯控制    │
│  User/TJC_TFT.c     ← 串口屏驱动   │
│  User/MyUsart/      ← UART驱动层   │
│  User/MyModbus/     ← Modbus协议   │
├────────────────────────────────────┤
│  STM32 HAL + FreeRTOS              │
└────────────────────────────────────┘
```

## 485三色灯 Modbus协议

**参数**: 9600, 8N1, 地址0x01

**寄存器映射**:
| 寄存器 | 功能 | 值 |
|--------|------|-----|
| 0x0064 | 红灯 | 0=关, 1=开 |
| 0x0065 | 绿灯 | 0=关, 1=开 |
| 0x0066 | 黄灯 | 0=关, 1=开 |
| 0x0067 | 蜂鸣器 | 0=关, 1=开 |
| 0x03E8 | 全部开启 | 1=全开 |
| 0x03E9 | 全部关闭 | 1=全关 |

**命令示例**:
| 功能 | 发送 | 返回 |
|------|------|------|
| 开红灯 | `01 06 00 64 00 01 09 D5` | 回显 |
| 关红灯 | `01 06 00 64 00 00 C8 15` | 回显 |
| 开绿灯 | `01 06 00 65 00 01 58 15` | 回显 |
| 开蜂鸣器 | `01 06 00 67 00 01 F9 D5` | 回显 |

## HC-433 GTI协议

**帧格式** (14字节):
```
47 54 49 [type] [time_s(4B)] [reserved(6B)] 01 AA
 │  │  │    │       │              │        │  │
 │  │  │    │       │              │        │  └─ 帧尾
 │  │  │    │       │              │        └─ 固定
 │  │  │    │       │              └─ 保留
 │  │  │    │       └─ 倒计时(秒×10, 大端)
 │  │  │    └─ 0x8F
 │  │  └─ 'I'
 │  └─ 'T'
 └─ 'G'
```

**示例**: `47 54 49 8F B0 04 00 00 00 00 00 00 01 AA`
- 倒计时 = 0x04B0 = 1220 → /10 = 122秒

**协议匹配条件** (`HC_433.c:306-316`):
```c
strstr(USART2_Buffer, "GTI") != NULL       // 帧头
USART2_Buffer[3] == 0x8F                    // 类型
USART2_Buffer[13] == 0xaa                   // 帧尾
```

## 倒计时控制逻辑

**LED_Flag状态机**:
| LED_Flag | 行为 |
|----------|------|
| 0 | 全关 (all_off) |
| 1 | 绿灯常亮 (正常倒计时) |
| 2 | 红灯常亮 + 蜂鸣器5秒 (倒计时结束) |
| 4 | 绿灯闪烁 (最后30秒倒计时) |

**倒计时对照表** (以120秒为例):
| 时间段 | 剩余秒数 | LED_Flag | 行为 |
|--------|---------|----------|------|
| T+0 ~ T+89s | 120~31 | 1 | 绿灯常亮 |
| T+90s | 30 | 4 | 绿灯闪烁160ms |
| T+91~118s | 29~2 | 4 | 绿灯闪烁(渐快到80ms) |
| T+119s | 1 | 2 | 红灯+蜂鸣器 |
| T+120s | 0 | - | 结束 |

## 已知问题与解决方案

| 问题 | 原因 | 解决方案 | 验证 |
|------|------|---------|------|
| 倒计时结束红灯+蜂鸣器不工作 | osDelay(1000)阻塞导致时序错乱, Counter_Run把countdown_time从1减到0 | LED_Flag=2移到osDelay之前, 让competition()立刻响应 | 倒计时结束红灯亮+蜂鸣器响 |
| all_off指令锁死后续写入 | `01 06 03 E9 00 01 99 BA` (全部关闭) 可能锁住单个寄存器写入 | case 2去掉all_off, 只发red_on+buzzer_on | 红灯正常亮起 |
| 485无应答 | RS485半双工收发切换延迟 / RTS未控制 | 发送后RtsEnable=false, 等待≥500ms收应答 | 收到正确回显 |
| COM端口不匹配 | 设备管理器显示COM号与实际不一致 | 检查设备管理器确认USB-485适配器实际端口 | 端口打开成功 |

## 关键代码位置

| 功能 | 文件路径 | 函数名 |
|------|---------|--------|
| 433协议解析 | `User/HC_433.c` | `HC433_Call_Led_Process` |
| 倒计时运行 | `User/HC_433.c` | `HC433_Counter_Run` |
| 灯控制状态机 | `User/Tapelight.c` | `competition` |
| UART5发送 | `User/MyUsart/my_uart.c` | `Uart5_SendBuff` |
| UART5接收 | `User/MyUsart/my_uart.c` | `HAL_UART_RxCpltCallback` |
| 屏幕控制 | `User/TJC_TFT.c` | `Uart3_TJC_Manage` |
| 主任务 | `Core/Src/freertos.c` | `UserTask_01` |
| 倒计时定时器 | `Core/Src/stm32f1xx_it.c` | `TIM3_IRQHandler` |

## Modbus CRC16计算

```powershell
function Get-ModbusCRC {
    param([byte[]]$data)
    $crc = 0xFFFF
    foreach ($b in $data) {
        $crc = $crc -bxor $b
        for ($i = 0; $i -lt 8; $i++) {
            if ($crc -band 0x0001) {
                $crc = ($crc -shr 1) -bxor 0xA001
            } else {
                $crc = $crc -shr 1
            }
        }
    }
    return $crc
}
```

## RS485通信调试

```powershell
# 打开COM4, 9600 8N1, 控制RTS切换收发
$sp = New-Object System.IO.Ports.SerialPort("COM4", 9600, None, 8, One)
$sp.ReadTimeout = 2000
$sp.Open()

# 发送: RTS高→发数据→RTS低→等应答
$sp.RtsEnable = $true
Start-Sleep -Milliseconds 20
$buf = [byte[]]@(0x01, 0x06, 0x00, 0x64, 0x00, 0x01, 0x09, 0xD5)
$sp.Write($buf, 0, $buf.Length)
$sp.RtsEnable = $false

Start-Sleep -Milliseconds 500
if ($sp.BytesToRead -gt 0) {
    $recv = [byte[]]::new($sp.BytesToRead)
    $sp.Read($recv, 0, $recv.Length)
    ($recv | ForEach-Object { "{0:X2}" -f $_ }) -join " "
}
$sp.Close()
```

## 调试 checklist

- [ ] 485三色灯供电12V/24V正常
- [ ] A/B线未接反
- [ ] 波特率9600 8N1
- [ ] RS485适配器RTS收发切换时序正确
- [ ] 发送后等待≥500ms收应答
- [ ] 433模块波特率1200
- [ ] GTI协议14字节完整
- [ ] HC_433.c中LED_Flag状态机时序正确
- [ ] osDelay不阻塞关键状态切换

## 仓库信息

- 仓库: `https://github.com/SmallWhitedawsd/WSC-16.git`
- 本地路径: `D:\reasonix\WSC-16改造\程序\WSC16 - 2025-8-18\`
