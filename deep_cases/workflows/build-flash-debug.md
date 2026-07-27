# 编译-烧录-调试标准流程

---

## 流程1: Keil MDK 编译流程

### 触发条件
- 项目使用 Keil MDK (`.uvprojx` 工程文件)
- 需要编译 STM32 固件生成 `.axf` 文件

### 步骤
1. 确认工程文件路径
2. 调用 UV4.exe 命令行编译
3. 检查编译产物是否存在
4. 如有错误，读取 Build.log

### 关键命令
```powershell
# 编译工程
D:\apps\Keil_v5\UV4\UV4.exe -r "D:\reasonix\OV7670_TFT\Project.uvprojx" -j0

# 检查编译产物
Test-Path "D:\reasonix\OV7670_TFT\Objects\OV7670_TFT.axf"
if ($?) { Write-Output "BUILD OK" } else { Write-Output "BUILD FAILED" }

# 读取编译日志
Get-Content "Build.log" -Tail 30
```

### 判断逻辑
- 编译成功 → `.axf` 文件存在，继续烧录
- 编译失败 → 读取 Build.log 定位错误行 → 修复源码 → 重新编译
- UV4.exe 无输出 → 检查进程是否卡死，超时后重试

### 来源案例
- OV7670_TFT - Keil_v5_build.bat 编译
- WSC-16 - UV4.exe 编译 STM32F103VET6 工程
- HelpPort - STM32F407VET6 工程编译

---

## 流程2: ST-Link CLI 烧录流程

### 触发条件
- 已有编译产物 (.hex 或 .bin)
- 需要通过 ST-Link 烧录到 STM32

### 步骤
1. 连接目标芯片
2. 烧录固件
3. 可选：复位运行

### 关键命令
```powershell
# 连接并烧录
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG -P "firmware.hex" 0x08000000 -V -Run

# 烧录 bin 文件
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG -P "firmware.bin" 0x08000000 -V

# 整片擦除
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG -ME

# 烧录后复位运行
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG -Rst
```

### 判断逻辑
- "Flash Programming Successful" → 烧录成功
- "Can not connect to target" → 检查接线/供电/芯片状态
- "Verification failed" → 重新擦除后烧录

### 来源案例
- HelpPort - ST-Link 烧录 LT7580 测试固件
- WSC-16 - ST-Link 烧录 STM32F103VET6

---

## 流程3: OpenOCD 烧录流程

### 触发条件
- 使用 OpenOCD 兼容调试器（ST-Link v2/v3, J-Link, DAP-Link）
- 需要脚本化烧录

### 步骤
1. 启动 OpenOCD 服务
2. 连接 GDB 或 telnet 烧录
3. 验证烧录结果

### 关键命令
```bash
# 启动 OpenOCD 服务
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg -c "program firmware.bin 0x08000000 verify reset exit"

# 仅擦除并烧录
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg \
  -c "init; reset halt; flash write_image erase firmware.bin 0x08000000; reset run; shutdown"
```

### 判断逻辑
- "Verified OK" → 烧录+校验成功
- "Error: open failed" → 检查调试器连接/驱动
- "target not halted" → 先执行 reset halt

### 来源案例
- 通用 STM32 调试流程

---

## 流程4: GDB 在线调试流程

### 触发条件
- 需要断点调试、寄存器读取、单步执行
- 系统异常需崩溃现场分析

### 步骤
1. 启动 GDB Server（OpenOCD 或 J-Link GDB Server）
2. 连接 GDB 客户端
3. 加载符号表
4. 设置断点/查看寄存器

### 关键命令
```bash
# 启动 OpenOCD GDB server
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg

# 连接 GDB
arm-none-eabi-gdb firmware.elf
(gdb) target remote :3333
(gdb) monitor reset halt
(gdb) load

# 常用调试命令
(gdb) break main
(gdb) continue
(gdb) info reg          # 查看所有寄存器
(gdb) x/16xw 0x08000000 # 查看 Flash 内容
(gdb) backtrace         # 查看调用栈
(gdb) print variable    # 打印变量值
```

### 判断逻辑
- GDB 连接成功 → 可正常调试
- "Cannot access memory" → 检查芯片是否 halt
- HardFault → 分析 BFAR/MMAR 寄存器定位错误地址

### 来源案例
- HelpPort - LT7580 初始化时序调试
- WSC-16 - UART5 RS485 通信调试

---

## 流程5: ST-Link 寄存器读取诊断

### 触发条件
- 系统异常重启，需判断复位原因
- 需要读取外设寄存器状态

### 步骤
1. 连接 ST-Link
2. 读取 RCC_CSR 判断复位源
3. 读取相关外设寄存器
4. 清零复位标志

### 关键命令
```powershell
# 连接目标
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG

# 读 RCC_CSR 寄存器判读复位原因 (0x40023874 for F4, 0x40021002 for F1)
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG -r32 0x40023874 1

# 读选项字节确认 BOR 级别
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG -rOB

# 清零复位标志 (写 bit24)
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

### RCC_CSR 寄存器位定义
| Bit | 标志 | 含义 |
|-----|------|------|
| 29 | IWDGRSTF | 看门狗复位 |
| 28 | SFTRSTF | 软件复位 |
| 27 | PORRSTF | 上电复位 |
| 26 | PINRSTF | NRST引脚复位 |
| 25 | BORRSTF | 欠压复位 |

### 判断逻辑
- IWDGRSTF=1 → 看门狗超时，检查喂狗逻辑/死循环
- PORRSTF=1 → 正常上电或电源跌落
- BORRSTF=1 → 欠压，检查电源/BOR级别
- PINRSTF=1 → 外部复位按键

### 来源案例
- HelpPort - 利用看门狗复位实现12秒稳定亮屏 (2026-07-08)
- WSC-16 - 系统异常重启诊断

---

## 流程6: fromelf 符号分析

### 触发条件
- 需要分析 .o/.axf 文件的符号表
- 需要确认函数/变量地址

### 步骤
1. 调用 fromelf 工具
2. 解析符号输出
3. 定位目标符号

### 关键命令
```powershell
# 列出所有符号
& "C:\apps\Keil_v5\ARM\ARMCC\Bin\fromelf.exe" -s "Objects\app.o"

# 反汇编
& "C:\apps\Keil_v5\ARM\ARMCC\Bin\fromelf.exe" -c "Objects\app.o"

# 提取可读字符串
& "C:\apps\Keil_v5\ARM\ARMCC\Bin\fromelf.exe" -z "Objects\app.o"
```

### 判断逻辑
- 符号存在 → 获取地址用于 GDB 断点
- 符号被优化 → 检查编译优化等级 (-O0 保留调试信息)

### 来源案例
- WSC-16 - hc_433.o 符号分析
