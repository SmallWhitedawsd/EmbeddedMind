# STM32F1/F4 系列配置经验

---

# IWDG 超时计算与配置

## 场景
需要配置独立看门狗实现系统死机复位，或调试启动卡死问题。

## 现象
LCD_Init() 中死等状态位导致系统卡死 30s 后被 IWDG 复位。

## 根因
IWDG 时钟源为 LSI (~32kHz)，超时计算公式：
```
超时 = (Prescaler / LSI) × Reload
例: (256 / 32000) × 3750 = 30s
```

## 解决方案
```c
// 启动 IWDG，2.5s 超时
hiwdg.Instance = IWDG;
hiwdg.Init.Prescaler = IWDG_PRESCALER_256;  // /256
hiwdg.Init.Reload = 312;  // 32000/256=125Hz, 312/125≈2.5s
HAL_IWDG_Init(&hiwdg);

// 喂狗
HAL_IWDG_Refresh(&hiwdg);
```

## 验证
- 正常启动每 100ms 喂狗一次
- 故意不喂狗，确认 2.5s 后复位
- 读 RCC_CSR bit29 确认复位源为 IWDG

## 来源
HelpPort - 利用看门狗复位实现12秒稳定亮屏 (2026-07-08)

---

# RCC_CSR 寄存器判读复位原因

## 场景
需要判断 STM32 上次复位原因（上电/看门狗/外部引脚/欠压）。

## 现象
系统异常重启，需定位复位源。

## 根因
RCC_CSR (0x40023874) 记录所有复位标志位。

## 解决方案
```powershell
# ST-LINK_CLI 读 RCC_CSR
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG -r32 0x40023874 1
```

| Bit | 标志 | 含义 |
|-----|------|------|
| 29 | IWDGRSTF | 看门狗复位 |
| 28 | SFTRSTF | 软件复位 |
| 27 | PORRSTF | 上电复位 |
| 26 | PINRSTF | NRST引脚复位 |
| 25 | BORRSTF | 欠压复位 |

清零：`-w32 0x40023874 0x01000000` (写 bit24 清所有标志)

## 验证
读寄存器后对应位为 1 即确认复位源。

## 来源
HelpPort - 利用看门狗复位实现12秒稳定亮屏 (2026-07-08)

---

# STM32F407VET6 系统时钟配置

## 场景
使用 STM32F407VET6 开发，需配置 168MHz 系统时钟。

## 现象
系统时钟决定 CPU、外设、SPI 等速度。

## 根因
F407 最大 168MHz，需通过 PLL 倍频 HSE 8MHz 得到。

## 解决方案
```c
void SystemClock_Config(void) {
    RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};
    
    // HSE 8MHz → PLL → 168MHz
    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState = RCC_HSE_ON;
    RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
    RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
    RCC_OscInitStruct.PLL.PLLM = 8;    // 8MHz/8 = 1MHz
    RCC_OscInitStruct.PLL.PLLN = 336;  // 1MHz*336 = 336MHz
    RCC_OscInitStruct.PLL.PLLP = RCC_PLLP_DIV2;  // 336/2 = 168MHz
    RCC_OscInitStruct.PLL.PLLQ = 7;    // 336/7 = 48MHz (USB)
    HAL_RCC_OscConfig(&RCC_OscInitStruct);
    
    // AHB=168MHz, APB1=42MHz, APB2=84MHz
    RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                                |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
    RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
    RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV4;   // 42MHz
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;   // 84MHz
    HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_5);
}
```

## 验证
- 读 `SystemCoreClock` 变量应为 168000000
- MCO 引脚输出时钟用示波器测量

## 来源
HelpPort - 嵌入式架构深度分析 (2026-07-22)

---

# STM32F429 教学平台时钟配置

## 场景
STM32F429 核心板用于教学实验平台，支持 144MHz 或 168MHz。

## 现象
不同模块代码可能使用不同时钟频率。

## 根因
F429 支持多档时钟，CubeMX 生成时可选。

## 解决方案
- 确认 .ioc 文件中 RCC 配置
- 144MHz: PLLN=360, PLLP=2, AHB 不分频
- 168MHz: PLLN=336, PLLP=2, AHB 不分频
- 修改后需同步调整 Flash 等待周期 (168MHz = WS=5)

## 验证
- 编译后运行 LED 闪烁程序，观察闪烁频率
- 用示波器测 MCO1/MCO2 引脚

## 来源
单片机设备 - 项目模块测试列表整理 (2026-07-20)

---

# JTAG 引脚复用为 GPIO/SPI

## 场景
PA15、PB3、PB4 默认是 JTAG 引脚，需复用为 SPI 片选或 GPIO。

## 现象
SPI 片选不工作或 GPIO 输出异常。

## 根因
JTAG 默认使能，引脚被调试接口占用。

## 解决方案
```c
// 方法1: HAL 初始化时自动禁用 JTAG
__HAL_RCC_AFIO_CLK_ENABLE();
__HAL_AFIO_REMAP_SWJ_NOJTAG();  // 禁用 JTAG，保留 SWD

// 方法2: 完全禁用（失去调试能力，慎用）
// __HAL_AFIO_REMAP_SWJ_DISABLE();
```

## 验证
- PA15/PB3/PB4 可正常输出
- SWD 调试仍可用（用 NOJTAG 模式）

## 来源
HelpPort - 拉取 Ponytail 作为技能使用 (2026-06-25)

---

# 选项字节 BOR 级别配置

## 场景
需要确认或修改欠压复位阈值。

## 现象
电源波动导致意外复位或该复位不复位。

## 根因
BOR (Brown-Out Reset) 级别决定 VDD 跌到何值时触发复位。

## 解决方案
```powershell
# 读选项字节确认 BOR 级别
& "ST-LINK_CLI.exe" -c ID=0 SWD HOTPLUG -rOB
```

| BOR 级别 | 阈值电压 |
|----------|---------|
| Level 1 | ~2.1V |
| Level 2 | ~2.4V |
| Level 3 | ~2.7V |
| Off | 禁用 BOR |

## 验证
- 读选项字节确认当前级别
- 用可调电源缓慢降压，确认复位电压

## 来源
HelpPort - 利用看门狗复位实现12秒稳定亮屏 (2026-07-08)
