# 串口调试标准流程

---

## 流程1: Modbus RTU 通信调试

### 触发条件
- RS485/Modbus 设备无响应
- 需要验证 CRC、接线、波特率

### 步骤
1. 确认供电 (12V/24V)
2. 确认接线 (A→A, B→B)
3. 确认 COM 端口号
4. 确认波特率 (9600/8/N/1)
5. 确认 485 收发切换 (RTS 控制)
6. 先读再写确认设备在线

### 关键命令
```powershell
# PowerShell Modbus 调试脚本
function Get-ModbusCRC {
    param([byte[]]$data)
    $crc = 0xFFFF
    foreach ($b in $data) {
        $crc = $crc -bxor $b
        for ($i = 0; $i -lt 8; $i++) {
            if ($crc -band 0x0001) { $crc = ($crc -shr 1) -bxor 0xA001 }
            else { $crc = $crc -shr 1 }
        }
    }
    return $crc
}

# 打开串口
$sp = New-Object System.IO.Ports.SerialPort("COM4", 9600, None, 8, One)
$sp.ReadTimeout = 2000
$sp.Open()

# RTS 控制 485 方向
$sp.RtsEnable = $true   # 发送模式
Start-Sleep -Milliseconds 20

# 发送 Modbus 写命令: 01 06 00 64 00 01 09 D5
$cmd = [byte[]]@(0x01, 0x06, 0x00, 0x64, 0x00, 0x01, 0x09, 0xD5)
$sp.Write($cmd, 0, $cmd.Length)

# 切回接收
$sp.RtsEnable = $false
Start-Sleep -Milliseconds 500

# 读取响应
if ($sp.BytesToRead -gt 0) {
    $recv = [byte[]]::new($sp.BytesToRead)
    $sp.Read($recv, 0, $recv.Length)
    $hex = ($recv | ForEach-Object { "{0:X2}" -f $_ }) -join "-"
    Write-Output "Response: $hex"
}
$sp.Close()
```

### Modbus RTU 命令参考
```
开红灯:   01 06 00 64 00 01 09 D5
关红灯:   01 06 00 64 00 00 C8 15
开绿灯:   01 06 00 65 00 01 58 15
关绿灯:   01 06 00 65 00 00 99 D5
开黄灯:   01 06 00 66 00 01 A8 15
关黄灯:   01 06 00 66 00 00 69 D5
开蜂鸣器: 01 06 00 67 00 01 F9 D5
关蜂鸣器: 01 06 00 67 00 00 38 15
全部开启: 01 06 03 E8 00 01 C8 7A
全部关闭: 01 06 03 E9 00 01 99 BA
```

### 判断逻辑
- 收到回显相同帧 → 设备正常响应
- 无响应 → 检查 RTS 时序/接线/波特率
- 乱码 → 波特率不匹配或接地问题
- CRC 错误 → 重新计算 CRC

### 来源案例
- WSC-16 - Serial protocol debug no response (2026-06-23)
- SX-MK-010 - Modbus 寄存器读写调试

---

## 流程2: 串口监听与日志抓取

### 触发条件
- 需要实时查看串口输出
- 需要抓取设备通信日志

### 步骤
1. 识别正确 COM 端口
2. 配置串口参数
3. 打开串口监听
4. 分析日志内容

### 关键命令
```powershell
# 列出可用串口
Get-PnpDevice -Class Ports | Where-Object { $_.Status -eq "OK" } | Select-Object FriendlyName

# 实时监听串口输出
$sp = New-Object System.IO.Ports.SerialPort("COM6", 9600, None, 8, One)
$sp.Open()
try {
    while ($true) {
        if ($sp.BytesToRead -gt 0) {
            $line = $sp.ReadLine()
            Write-Host "$(Get-Date -Format 'HH:mm:ss') $line"
        }
        Start-Sleep -Milliseconds 100
    }
} finally {
    $sp.Close()
}

# 一次性读取所有可用数据
$sp = New-Object System.IO.Ports.SerialPort("COM4", 9600)
$sp.ReadTimeout = 1000
$sp.Open()
Start-Sleep -Milliseconds 500
$output = $sp.ReadExisting()
$sp.Close()
$output
```

### 判断逻辑
- 有数据输出 → 设备正在工作，分析协议内容
- 无数据 → 检查 TX/RX 是否交叉、GND 是否共地
- 乱码 → 波特率/数据位/校验位不匹配

### 来源案例
- WSC-16 - RS485 三色灯通信监听
- HelpPort - 串口日志分析

---

## 流程3: RS485 半双工收发切换调试

### 触发条件
- RS485 设备发送后无响应
- 方向切换时序问题

### 步骤
1. 确认方向控制方式 (RTS/GPIO)
2. 测试发送后切接收的延迟
3. 验证设备响应

### 关键命令
```powershell
# 方法1: RTS 硬件自动控制
$sp = New-Object System.IO.Ports.SerialPort("COM4", 9600, None, 8, One)
$sp.RtsEnable = $true
$sp.Open()

# 发送
$sp.RtsEnable = $true          # TX mode
Start-Sleep -Milliseconds 20   # 方向切换稳定
$sp.Write($buf, 0, $buf.Length)
$sp.Flush()                    # 等待发送完成

# 接收
$sp.RtsEnable = $false         # RX mode
Start-Sleep -Milliseconds 500  # 等待设备响应
if ($sp.BytesToRead -gt 0) {
    $resp = [byte[]]::new($sp.BytesToRead)
    $sp.Read($resp, 0, $resp.Length)
}
$sp.Close()

# 方法2: 手动 GPIO 控制 (STM32 固件端)
# HAL_GPIO_WritePin(RS485_DE_GPIO_Port, RS485_DE_Pin, GPIO_PIN_SET);  // TX
# HAL_UART_Transmit(&huart5, data, len, 100);
# HAL_GPIO_WritePin(RS485_DE_GPIO_Port, RS485_DE_Pin, GPIO_PIN_RESET); // RX
```

### 关键延迟参数
- 方向切换后: ~20ms 稳定时间
- 发送完成后: 等待最后一个字节移位完毕 (9600bps 下 1字节≈1ms)
- 设备响应: 100-500ms

### 判断逻辑
- 发送后 500ms 内有响应 → 正常
- 无响应 → 增加方向切换延迟/检查 A-B 接线
- 收到错误数据 → 检查波特率匹配

### 来源案例
- WSC-16 - UART5 RS485 control logic (2026-06-23)

---

## 流程4: TJC 串口屏通信调试

### 触发条件
- 需要与 TJC 串口屏交互
- 屏幕显示异常或按钮无响应

### 步骤
1. 确认串口参数 (9600/8/N/1)
2. 发送命令 + 帧尾 (3个 0xFF)
3. 验证屏幕响应

### 关键命令
```powershell
# TJC 协议: 命令字符串 + 0xFF 0xFF 0xFF
$sp = New-Object System.IO.Ports.SerialPort("COM3", 9600)
$sp.Open()

# 同步按钮状态
$cmd = "home.bt222.val=1"
$sp.Write($cmd)
$sp.Write([byte[]]@(0xFF, 0xFF, 0xFF), 0, 3)

# 更新显示值
$power = 12.34
$cmd = "home.POWER.txt=`"$power kW·h`""
$sp.Write($cmd)
$sp.Write([byte[]]@(0xFF, 0xFF, 0xFF), 0, 3)

$sp.Close()
```

### 判断逻辑
- 屏幕显示更新 → 通信正常
- 无变化 → 检查波特率/帧尾/地址
- 乱码 → 文件编码问题 (GB2312)

### 来源案例
- SX-MK-010 - 触摸屏停止按钮继电器失效问题 (2026-07-10)

---

## 流程5: 串口协议逆向分析

### 触发条件
- 未知协议的设备需要分析
- 需要抓包解析通信格式

### 步骤
1. 抓取原始数据 (逻辑分析仪/串口监听)
2. 分析帧格式 (地址+功能码+数据+CRC)
3. 验证 CRC 校验
4. 构造测试命令

### 关键命令
```powershell
# 抓取并解析未知协议
$sp = New-Object System.IO.Ports.SerialPort("COM4", 9600)
$sp.Open()
Start-Sleep -Milliseconds 1000
$raw = [byte[]]::new($sp.BytesToRead)
$sp.Read($raw, 0, $raw.Length)
$sp.Close()

# 转十六进制显示
$hex = ($raw | ForEach-Object { "{0:X2}" -f $_ }) -join " "
Write-Output $hex

# 分析帧结构
# 地址: 第1字节
# 功能码: 第2字节
# 数据: 第3~N-2字节
# CRC: 最后2字节 (低字节在前)
```

### 判断逻辑
- CRC 校验通过 → 帧格式正确
- 固定地址字节 → 设备地址
- 功能码 0x03=读, 0x06=写 → Modbus RTU 协议

### 来源案例
- WIFI_CAM - 摄像头 UDP/TCP 协议逆向
- WSC-16 - 433 无线模块协议分析

---

## 流程6: UART 环形缓冲区调试

### 触发条件
- 串口数据丢失
- 高频数据接收异常

### 步骤
1. 检查缓冲区大小是否足够
2. 验证中断/DMA 配置
3. 检查帧超时检测逻辑
4. 测试高负载下数据完整性

### 关键命令
```c
// 环形缓冲区实现
#define UART_BUF_SIZE 256
typedef struct {
    uint8_t  buffer[UART_BUF_SIZE];
    volatile uint16_t head;
    volatile uint16_t tail;
    volatile uint16_t count;
} UART_RingBuffer;

// 帧超时检测 (10ms 无新数据视为帧结束)
#define FRAME_TIMEOUT_MS 10
void USART5_RecService(void) {
    static uint32_t last_rx_time = 0;
    static uint16_t last_count = 0;
    if (rx_buf.count != last_count) {
        last_count = rx_buf.count;
        last_rx_time = HAL_GetTick();
    } else if (rx_buf.count > 0 &&
               HAL_GetTick() - last_rx_time > FRAME_TIMEOUT_MS) {
        ProcessModbusFrame(rx_buf.buffer + rx_buf.tail, rx_buf.count);
        rx_buf.count = 0;
        rx_buf.tail = rx_buf.head;
    }
}
```

### 判断逻辑
- 数据完整 → 缓冲区大小/超时配置正确
- 数据丢失 → 增大缓冲区/缩短中断处理时间
- 帧错位 → 检查超时阈值

### 来源案例
- WSC-16 - UART5 RS485 control logic (2026-06-23)
