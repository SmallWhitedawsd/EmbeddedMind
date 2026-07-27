# STM32教学模块平台

## 硬件配置

| 参数 | 值 |
|------|-----|
| 核心板MCU | STM32F429 (168/180MHz) |
| 底板 | 6排插槽: SPI4/5/6, I2C2/3, UART3/4/6/7, CAN, PWMT, ADC, DAC, GPIO |
| 模块数 | 74个外设模块 |
| 调试器 | ST-Link |

### 底板插槽定义

| 排 | 插槽1 | 插槽2 | 插槽3 | 插槽4 | 插槽5 | 插槽6 |
|----|-------|-------|-------|-------|-------|-------|
| A排 | SPI4 | SPI5 | SPI6 | ADC1-3 | ADC1-3 | ADC11-15 |
| B排 | I2C2 | I2C3 | GPIO1 | ADC9 | ADC9 | DAC1-2 |
| C排 | CAN | UART3 | UART4 | DAC1-2 | DAC1-2 | GPIO3 |
| D排 | PWMT8 | PWMT3/14 | PWMT5 | UART6 | UART7 | - |

## 模块分类

### A组 - 独立可测 (无需LCD)

| 模块 | 功能 | 接口 | 测试方法 |
|------|------|------|---------|
| LED | 4路LED | GPIO | 流水灯, 100ms切换 |
| LED2 | 16路LED | GPIO | 流水灯 |
| shu_ma_guan | 数码管 | GPIO | 显示0-9 |
| MAX7219 | LED点阵 | SPI | 显示图案 |
| bian_ma_qi_EC11 | 编码器 | GPIO | 旋转计数 |
| duo_ji | 舵机 | PWM | 角度转动 |
| H_qiao_dian_ji | H桥电机 | GPIO/PWM | 正反转 |
| wu_shua_dian_ji | 无刷电机 | PWMT8 | 转动 |
| BUZZ_y | 有源蜂鸣器 | GPIO | 响 |
| BUZZ_w | 无源蜂鸣器 | PWM | 响 |
| ji_dian_qi | 继电器 | GPIO | 吸合/释放声 |

### B组 - 传感器 (需LCD12864显示)

| 模块 | 功能 | 接口 | LCD显示 |
|------|------|------|---------|
| DS18B20 | 温度 | GPIO 1-Wire | "DS18B20 XX.XX°C" |
| chao_sheng_bo | 超声波 | GPIO | 距离 |
| ACS712 | 电流 | ADC | 电流值 |
| AD9833 | DDS信号 | SPI | 波形频率 |
| da_qi_ya | BMP280气压 | I2C | 气压/温度 |
| guang_zhao_du | BH1750光照 | I2C | 光照强度 |
| hun_zhuo_du | 浊度 | ADC | 浊度值 |
| huo_yan_cgq | 火焰传感器 | GPIO | 状态 |
| DHT11 | 温湿度 | GPIO | 温湿度 |
| NTC | 热敏电阻 | ADC | 温度 |

### C组 - 通信/无线模块

| 模块 | 功能 | 接口 | 备注 |
|------|------|------|------|
| NRF24L01 | 2.4G无线 | SPI | 需2个模块配对 |
| ESP01WIFI | WiFi透传 | UART | AT指令 |
| ASC32_LoRa | LoRa | UART | 无线串口 |
| CC2530_E18 | ZigBee | UART | 组网 |
| E27_4G&E33_NB | 4G/NB-IoT | UART | 需SIM卡 |
| ETH | 以太网 | SPI | W5500 |
| GPS_HT303B | GPS定位 | UART | NMEA输出 |
| hong_wai_tong_xin | 红外通信 | UART | 短距离 |
| JDY23_BLE | 蓝牙4.0 | UART | BLE |

### D组 - 存储/显示/其他

| 模块 | 功能 | 接口 |
|------|------|------|
| 24C02 | EEPROM | I2C |
| W25Qxx | Flash | SPI |
| SPI_SD | SD卡 | SPI+FATFS |
| LCD12864 | 128×64液晶 | 并口 |
| LCD1602 | 16×2字符液晶 | 并口 |
| RC522 | RFID | SPI |
| re_dian_o | 热电偶 | SPI |
| FMC_LCD | TFT-LCD | FMC |
| OV5640 | 摄像头 | DCMI+FMC |

## 代码模式

| 方面 | 说明 |
|------|------|
| MCU | STM32F429xx |
| 框架 | STM32 HAL库, CubeMX生成 |
| 输出 | 无串口, GPIO直驱+LCD显示 |
| 时钟 | 144MHz或168MHz |
| 项目结构 | Keil MDK, Core/Src/main.c |

## 模块测试通用流程

1. 读模块`安装位置.txt`确认插槽
2. 插到对应底板槽位
3. 编译对应模块固件
4. ST-Link烧录
5. 上电观察现象 (看/听/摸/读LCD)

## usart_lcd模块调试

**现象**: 按开始/停止/清零无反应

**根因**: 上电后需按复位键才能正常工作

**屏幕发送协议**:
| 按键 | 发送数据 |
|------|---------|
| 开始 | `73 74 61 72 74` ("start") |
| 停止 | `73 74 6F 70` ("stop") |
| 清零 | `72 65 73 65 74` ("reset") |
| 按键1 | `6B 65 79 31 0B 00 00 00` |

## 已知问题与解决方案

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 模块上电无反应 | 需按复位键 | 烧录后按复位 |
| OV5640编译失败 | scatter文件路径错误 | 检查.sct文件路径 |
| OV5640 Flash下载失败 | 无Algorithm | 添加STM32F429 Flash算法 |
| 无刷电机不转 | PWMT8引脚/配置 | 检查TIM8 PWM输出 |
| 串口屏按钮无响应 | 上电未复位 | 按复位键 |

## 仓库信息

- 本地路径: `D:\reasonix\单片机设备\STM32\STM32\`
- 原理图: `D:\reasonix\单片机设备\原理图\`
- 模块数: 74个
