# GT9147 触摸驱动软件 I2C 实现

## 场景
STM32F407 通过软件 I2C 读取 GT9147 触屏芯片坐标。

## 现象/问题
需确认引脚定义和通信协议。

## 根因分析
硬件 I2C 引脚被占用，使用软件 I2C 模拟时序。

## 解决方案
| 信号 | 引脚 |
|------|------|
| SCL | PE2 |
| RST | PE3 |
| SDA | PE5 |
| INT | PE6 |

```c
// 软件 I2C 读取 GT9147
void GT9147_ReadCoord(uint16_t *x, uint16_t *y) {
    uint8_t buf[4];
    I2C_Start();
    I2C_SendByte(0x81);  // GT9147 地址
    I2C_SendByte(0x81);  // 坐标寄存器
    I2C_Start();
    I2C_SendByte(0x81 | 0x01);  // 读模式
    buf[0] = I2C_ReadByte(1);
    buf[1] = I2C_ReadByte(1);
    buf[2] = I2C_ReadByte(1);
    buf[3] = I2C_ReadByte(0);
    I2C_Stop();
    *x = (buf[1] << 8) | buf[0];
    *y = (buf[3] << 8) | buf[2];
}
```

## 验证方法
触摸屏幕，观察坐标值随触摸位置变化。

## 来源
HelpPort - 附录关键技术点 (2026-07-27)

---

# 触摸驱动与 LVGL 集成

## 场景
将 GT9147 触摸数据接入 LVGL 输入设备框架。

## 现象/问题
LVGL 需要标准 `lv_indev_drv_t` 接口读取触摸数据。

## 根因分析
LVGL 通过 `lv_indev_drv_t.read_cb` 回调获取输入数据。

## 解决方案
```c
// lv_port_indev.c
static void touchpad_read(lv_indev_drv_t *drv, lv_indev_data_t *data) {
    uint16_t x, y;
    if (GT9147_IsPressed()) {
        GT9147_ReadCoord(&x, &y);
        data->point.x = x;
        data->point.y = y;
        data->state = LV_INDEV_STATE_PR;
    } else {
        data->state = LV_INDEV_STATE_REL;
    }
}

void lv_port_indev_init(void) {
    static lv_indev_drv_t indev_drv;
    lv_indev_drv_init(&indev_drv);
    indev_drv.type = LV_INDEV_TYPE_POINTER;
    indev_drv.read_cb = touchpad_read;
    lv_indev_drv_register(&indev_drv);
}
```

**关键**：`read_cb` 中不能包裹 `taskENTER_CRITICAL`，否则 AD7606 ISR 会打断 I2C 时序。

## 验证方法
触摸屏幕，LVGL 按钮能正常响应点击。

## 来源
HelpPort - Caveman mode: ultra (代码重构) (2026-07-04)

---

# 触摸 RST/INT 上电复位时序

## 场景
GT9147 触摸芯片上电复位时序问题导致首次触摸异常。

## 现象/问题
首次上电触屏卡顿，多次上电后流畅。

## 根因分析
触摸芯片 RST/INT 上电复位时序不满足要求，芯片未正确初始化。

## 解决方案
```c
// CTP_Init 时序
void CTP_Init(void) {
    // 1. RST 拉低 10ms
    HAL_GPIO_WritePin(CTP_RST_GPIO_Port, CTP_RST_Pin, GPIO_PIN_RESET);
    HAL_Delay(10);
    
    // 2. RST 拉高，等待 50ms
    HAL_GPIO_WritePin(CTP_RST_GPIO_Port, CTP_RST_Pin, GPIO_PIN_SET);
    HAL_Delay(50);
    
    // 3. 配置 GT9147 寄存器
    GT9147_Config();
}
```

INT 引脚配置为外部中断，下降沿触发，用于通知主控有触摸事件。

## 验证方法
示波器抓 RST 引脚，确认 10ms low + 50ms high 时序。

## 来源
HelpPort - Caveman mode: ultra (代码重构) (2026-07-04)

---

# 手写识别与触摸轨迹采集

## 场景
使用 ATKNCR 手写识别引擎采集触屏轨迹。

## 现象/问题
需将触摸轨迹传递给识别引擎。

## 根因分析
ATKNCR 库需要轨迹点数组作为输入。

## 解决方案
```c
// 轨迹采集 + 识别
uint8_t READ_BUF[200];  // 轨迹点缓冲
uint8_t pcnt = 0;       // 点数计数

// 触摸按下时采集
if (pressed) {
    READ_BUF[pcnt * 2] = x & 0xFF;
    READ_BUF[pcnt * 2 + 1] = y & 0xFF;
    LCD_DrawBlueLine(x, y);  // 画蓝线
    pcnt++;
}

// 触摸释放时识别
if (released && pcnt > 0) {
    char res[7];  // Top-6 候选 + '\0'
    alientek_ncr(READ_BUF, pcnt, 6, mode, res);
    // mode: 0=数字 1=大写 2=小写 3=全识别
}
```

## 验证方法
手写数字/字母，观察识别结果是否正确。

## 来源
HelpPort - 手写识别例程分析 (2026-07-06)
