# SX-MK-010

## 硬件配置

| 参数 | 值 |
|------|-----|
| MCU | STM32F103 |
| 屏幕 | 3.5寸电阻屏 (单相) / 4.3寸电容屏 (三相) |
| 电能计量 | RN7302 SPI接口 |
| RFID | MFRC522 13.56MHz |
| 继电器 | 启动PB15, 停止PC6 |
| 触摸屏通信 | USART3, TJC协议 |

### 两个版本
| 版本 | 项目名 | 屏幕 | 稳定性 |
|------|--------|------|--------|
| 单相 | SX-MK-010-0140 | 3.5寸电阻屏 | 问题较多 |
| 三相 | SX-803G-B-06 | 4.3寸电容屏 | 相对稳定 |

### 关键引脚定义

**继电器控制**:
| 引脚 | 功能 | 状态 |
|------|------|------|
| PB15 (Start_Pin) | 继电器吸合 | SET=吸合, RESET=释放 |
| PC6 (stop_Pin) | 继电器释放 | SET=释放 |

**RN7302 SPI**: 电能计量芯片

**USART3**: TJC串口屏, 帧头+数据+3×0xFF帧尾

## 软件架构

```
┌──────────────────────────────────────────────┐
│  User/Control.c           ← 核心业务逻辑      │
│  User/mfrc522.c           ← RFID读卡器        │
│  User/TJC_TFT.c           ← 串口屏驱动        │
│  User/RN7302.c            ← 电能计量          │
│  User/Power_Supply_Testing.c ← 继电器电源     │
│  User/Runtime_Manager.c   ← 倒计时管理        │
│  User/MyModbus/           ← Modbus协议        │
│  User/MyFlash/            ← Flash参数存储     │
│  User/MyUsart/            ← UART环形缓冲      │
├──────────────────────────────────────────────┤
│  STM32 HAL + FreeRTOS                        │
└──────────────────────────────────────────────┘
```

**加密状态**: 大部分User/源文件加密 (文件头`18 1B 03 1A`), 仅Control.c等可读

## Modbus寄存器映射

| 地址 | 名称 | 说明 |
|------|------|------|
| 10 | Uc/Ic (单相) / Usum/Isum (三相) | 电压/电流 |
| 14 | SRW | 写权限控制 (1=解锁写入) |
| 26 | set_start_step | 启动/停止 (1=启动, 0=停止) |
| 44 | card_status | 卡状态 (0=已充值) |

## 继电器控制逻辑

**启动** (屏幕命令0x01):
```c
HAL_GPIO_WritePin(Start_GPIO_Port, Start_Pin, GPIO_PIN_SET);   // 吸合
HAL_GPIO_WritePin(stop_GPIO_Port, stop_Pin, GPIO_PIN_RESET);   // 释放stop
HAL_TIM_Base_Start_IT(&htim2);                                  // 启动定时器
HAL_TIM_PWM_Start(&htim4, TIM_CHANNEL_2);                       // 启动PWM
HAL_TIM_PWM_Stop(&htim4, TIM_CHANNEL_3);                        // 停止反向PWM
```

**停止** (屏幕命令0x00):
```c
HAL_GPIO_WritePin(stop_GPIO_Port, stop_Pin, GPIO_PIN_SET);     // SET stop
HAL_GPIO_WritePin(Start_GPIO_Port, Start_Pin, GPIO_PIN_RESET); // RESET start
HAL_TIM_Base_Stop_IT(&htim2);                                   // 停止定时器
HAL_TIM_PWM_Stop(&htim4, TIM_CHANNEL_2);                        // 停止PWM
HAL_TIM_PWM_Start(&htim4, TIM_CHANNEL_3);                       // 反向PWM释放
Power_Manager(0);                                               // 关闭电源
```

**Modbus停止** (reg26=0):
- `Power_Manager(0)` → 继电器断电
- 停止后 `experiment_duration=0` 阻止RFID重启动

## RFID刷卡逻辑

**MFRC522 13.56MHz**, `RFID_RUN2()` 处理流程:

1. **老师充电**: card_status写为0 (addr44)
2. **学生刷卡**: bench_id匹配 → 卡片激活(status→1) → 用卡内时间启动倒计时
3. **学生加时间** (card_status==0): Modbus地址44为0且实验台ID匹配时, 把卡内实验时间作为新倒计时

**关键修复**: `mfrc522.c` 第1890行, 移除 `&& experiment_duration == 0` 条件限制

## RN7302电能计量

**换算公式**:
```
单相: Isum = Isum_REG * Kic * Kct / 1000
三相: Isum = Isum_REG * Kisum * Kct / 1000
```

**三相额外参数**:
- Kusum_Default = 1.24887802e-05
- Kisum_Default = 1.36180427E-06

## 已知问题与解决方案

| 问题 | 原因 | 解决方案 | 验证 |
|------|------|---------|------|
| reg26=0停止时屏幕按键不同步 | 停止路径缺失屏幕同步指令 | 插入`Uart3_SendBuff("home.bt222.val=1")` | 停止后屏幕按钮状态正确 |
| 停止后重刷继电器又吸合 | 竞态: experiment_duration>0, 刷卡重启覆盖停止 | 停止路径加`experiment_duration=0` | 停止后刷卡不重启 |
| 电阻屏停止键有概率失灵 | MFRC522 13.56MHz电磁干扰电阻屏ADC | 帧校验/命令重发/硬件滤波/刷卡后延迟100ms | 停止键响应可靠 |
| 单位显示KW.h错误 | 应为kW.h | Control.c改`K`→`k` (GB2312字节安全编辑) | 屏幕显示kW.h |
| 新卡只能刷一次 | 修改破坏了readBuf[5]==1已激活卡重启逻辑 | 加`set_start_step!=0`检查 | 同卡可重复刷 |

## 关键代码位置

| 功能 | 文件路径 | 函数名 |
|------|---------|--------|
| 核心控制逻辑 | `User/Control.c` | `main_loop`, `Param_Default` |
| RFID处理 | `User/mfrc522.c` | `RFID_RUN2` |
| 屏幕通信 | `User/TJC_TFT.c` | `Uart3_TJC_Manage`, `Uart3_SendEnd` |
| 继电器控制 | `User/TJC_TFT.c` | case 0x01(启动), case 0x00(停止) |
| 电能计量 | `User/RN7302.c` | RN7302初始化/采样 |
| Modbus映射 | `User/MyModbus/MMU.c` | 寄存器读写 |
| Flash存储 | `User/MyFlash/my_flashData.c` | 参数保存/恢复 |
| 倒计时管理 | `User/Runtime_Manager.c` | CountDown_Start/Stop/Get |

## 文件编码

- 源代码: GB2312编码 (中文注释)
- LF行尾
- 编辑需字节安全, 避免破坏编码

## TJC协议帧格式

```
[帧头] [数据] [0xFF] [0xFF] [0xFF]
         └── 3字节帧尾
```

**按键状态同步**: `home.bt222.val=1`

## 调试 checklist

- [ ] 继电器PB15/PC6初始状态RESET(低电平)
- [ ] 刷卡后延时100ms再处理屏幕命令
- [ ] USART3帧尾3×0xFF正确
- [ ] Modbus写reg14=1解锁写权限
- [ ] 停止后experiment_duration=0
- [ ] RN7302 SPI时序正确
- [ ] Flash参数保存/恢复一致
- [ ] 电阻屏UART线路滤波

## 仓库信息

- BUG修复: `https://github.com/SmallWhitedawsd/2026-07-10-BUG.git`
- 原始代码: `https://github.com/SmallWhitedawsd/dy_yuan.git`
- 新版代码: `https://github.com/SmallWhitedawsd/dy_NEW.git`
- 本地路径: `D:\reasonix\SX-MK-010-0140\`
- 已知commit: `319c4aa`, `ad25b8e`, `8ce9385`
