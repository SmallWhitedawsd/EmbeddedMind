# 硬件调试方法论

---

# ST-Link CLI 调试技巧

## 场景
需要通过 ST-Link 命令行读取寄存器、判断复位原因、清零标志。

## 现象
系统异常重启，需快速定位原因。

## 根因
手动读寄存器效率低，需自动化脚本。

## 解决方案
```powershell
# 连接目标
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG

# 读 RCC_CSR 寄存器判读复位原因
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG -r32 0x40023874 1

# 读选项字节确认 BOR 级别
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG -rOB

# 清零复位标志
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG -w32 0x40023874 0x01000000

# 3分钟监听脚本: 每6秒读一次 RCC_CSR
$cli = "C:\Program Files (x86)\STMicroelectronics\STM32 ST-LINK Utility\ST-LINK Utility\ST-LINK_CLI.exe"
$end = (Get-Date).AddMinutes(3)
while ((Get-Date) -lt $end) {
    $out = & $cli -c ID=0 SWD HOTPLUG -r32 0x40023874 1 2>&1 | Out-String
    $m = [regex]::Match($out, '0x40023874\s*:\s*([0-9A-Fa-f]{8})')
    if ($m.Success) {
        $val = [Convert]::ToUInt32($m.Groups[1].Value, 16)
        $iwdg = ($val -shr 29) -band 1
        $por  = ($val -shr 27) -band 1
        Write-Host "$(Get-Date -Format 'HH:mm:ss') IWDG=$iwdg POR=$por"
    }
    Start-Sleep -Seconds 6
}
```

## 验证
- 能正确读取寄存器值
- 监听脚本能捕获复位事件

## 来源
HelpPort - 利用看门狗复位实现12秒稳定亮屏 (2026-07-08)

---

# ST-Link 对启动时序的影响

## 场景
接 ST-Link 调试时系统正常，断开后异常。

## 现象
ST-Link 的 VCC/GND 接住芯片初始电平，掩盖启动问题。

## 根因
ST-Link 的 VTref + SWDIO/SWCLK 接住芯片初始电平，提供额外上拉/电容。

## 解决方案
```c
// 1. 断开 ST-Link 独立上电测试
// 2. 如果断开后出问题，检查:
//    - 电源上升时间是否足够快
//    - NRST 引脚上拉/电容是否合适
//    - 各外设供电时序

// 3. 用示波器同时抓 VDD、NRST、SWDIO 上电波形
```

## 验证
- 断开 ST-Link 后系统正常启动
- 电源上升时间 < 1ms

## 来源
HelpPort - 利用看门狗复位实现12秒稳定亮屏 (2026-07-08)

---

# Modbus RTU 通信调试

## 场景
Modbus 设备无响应，需排查协议/硬件问题。

## 现象
发送 `01 06 00 64 00 01 09 D5` 后无应答。

## 根因
可能原因：波特率不匹配、地址/寄存器错误、485 收发切换问题、接线错误。

## 解决方案
```powershell
# 1. 验证 CRC
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

# 2. 先读再写（确认设备在线）
# 读: 01 03 00 64 00 01 CRC
# 写: 01 06 00 64 00 01 CRC

# 3. 排查顺序:
#    a. 确认供电 (12V/24V)
#    b. 确认接线 (A→A, B→B)
#    c. 确认 COM 端口号
#    d. 确认波特率 (9600/8/N/1)
#    e. 确认 485 收发切换 (RTS 控制)
```

## 验证
- 读命令返回正确数据
- 写命令后设备状态改变 + 回显

## 来源
WSC-16 - Serial protocol debug no response (2026-06-23)

---

# 电阻屏与电容屏抗干扰差异

## 场景
刷卡（RFID）后电阻屏停止按钮偶发失效，电容屏无此问题。

## 现象
刷卡瞬间 MFRC522 产生强电磁场（13.56MHz），干扰电阻屏 ADC 采样。

## 根因
- 电阻屏通过 ADC 检测触摸，易受电磁干扰
- 刷卡瞬间干扰串口通信（USART3 单字节中断接收）
- 电容屏本身抗干扰能力更强

## 解决方案
```c
// 软件方案:
// 1. 帧长度校验，丢弃异常长度帧
void Uart3_TJC_Manage(void) {
    if (rx_len < MIN_FRAME_LEN || rx_len > MAX_FRAME_LEN) {
        rx_len = 0;  // 丢弃
        return;
    }
}

// 2. 刷卡后延迟 100-200ms 再处理屏幕命令
void RFID_Process(void) {
    MFRC522_ReadCard();
    osDelay(150);  // 避开 RFID 干扰窗口
}

// 3. 命令重发机制
// 屏幕端发停止命令后等待 MCU 应答，无应答则重发

// 硬件方案:
// - 电阻屏 UART 线路加滤波/屏蔽
// - 降低波特率
```

## 验证
- 连续刷卡 50 次，停止按钮 100% 响应
- 串口无乱码帧

## 来源
SX-MK-010 - 触摸屏停止按钮继电器失效问题 (2026-07-10)

---

# 竞态条件调试方法

## 场景
停止后重刷卡，继电器意外吸合。

## 现象
TFT 停止 → 继电器 OFF → 用户再刷卡 → 继电器又吸合。

## 根因
```c
// 竞态流程:
// 1. TFT停止 → 继电器OFF + experiment_duration保存剩余时间
// 2. 用户再刷卡 → RFID_RUN2 检查 experiment_duration>0
//    → 重启倒计时+继电器ON → 覆盖停止
```

## 解决方案
```c
// 在所有停止路径添加:
sx_mk_Control.experiment_duration = 0;

// 影响路径:
// - 不同卡接管时，若有剩余时间且 set_start_step==0 → 不重启
// - 同卡重刷时，若有剩余时间且 set_start_step==0 → 不重启
// - 新卡激活和倒计时运行中接力不受影响
```

## 验证
- 停止后刷卡 → 继电器不吸合
- 倒计时运行中刷卡接力 → 正常

## 来源
SX-MK-010 - 拉取代码+功能开发+BUG修复 (2026-07-10)

---

# 文件编码安全编辑

## 场景
修改 GB2312 编码的中文注释文件后编译报错。

## 现象
编辑后中文乱码或编译失败。

## 根因
GB2312 编码的中文占 2 字节，直接按 ASCII 编辑会破坏编码。

## 解决方案
```bash
# 1. 确认文件编码
file -i Control.c  # 显示 charset=iso-8859-1 或 gb2312

# 2. 只改 ASCII 部分（如 K→k），保留中文不变
# 3. 使用支持 GB2312 的编辑器
# 4. 编辑后验证中文显示正常
```

## 验证
- 编译无警告
- 屏幕中文显示正常

## 来源
SX-MK-010 - 阅读单相和三相版本BUG代码 (2026-07-10)

---

# 放电电阻选型计算

## 场景
需要快速放电外设 VDD 轨残留电荷。

## 现象
断电后短时间内重新上电，外设因残压状态不确定。

## 根因
大电容缓放电路导致 VDD 下降缓慢，未触发 POR。

## 解决方案
```
计算公式:
τ = R × C
完全放电 ≈ 5τ

选型步骤:
1. 确定目标放电时间 t_target
2. τ = t_target / 5
3. R = τ / C
4. 功率 P = V² / R，需 < 电阻额定功率 × 50%降额

实例:
C = 470µF, 目标 1s 放完
τ = 1/5 = 0.2s
R = 0.2 / 0.00047 ≈ 430Ω → 取 1kΩ（保守）
P = 5² / 1000 = 25mW < 62.5mW (0805) ✓
```

## 验证
- 示波器测量 VDD 在目标时间内放电到 0
- 电阻温升正常

## 来源
HelpPort - 利用看门狗复位实现12秒稳定亮屏 (2026-07-08)

---

# 嵌入式系统分层架构设计

## 场景
项目需要支持多硬件平台、多人协作。

## 现象
代码耦合严重，换硬件需大量修改。

## 根因
未分层抽象，业务代码直接调用 HAL。

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

// BSP 抽象方式: 编译期宏开关 + 初始化函数暴露
#ifdef USE_AD7606
    bsp_InitAD7606();
#endif
#ifdef USE_LT7580
    LCD_Init();
#endif
```

## 验证
- 换 MCU 平台只需改 app/hal/ 和 BSP
- 业务代码零修改

## 来源
HelpPort - 嵌入式架构深度分析 (2026-07-22)
