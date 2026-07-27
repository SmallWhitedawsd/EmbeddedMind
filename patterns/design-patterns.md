# 设计模式库

## 模式1: 五层架构分层

```
┌──────────────────────────────────────────┐
│  app/tasks/           ← 业务任务           │
│  app/services/        ← 服务层             │
│  app/drivers/         ← 驱动抽象层         │
│  app/hal/             ← HAL适配层          │
│  app/config/          ← 编译期配置         │
├──────────────────────────────────────────┤
│  BSP (bsp_ad7606/lcd/ctp) ← 板级支持      │
├──────────────────────────────────────────┤
│  STM32 HAL + FreeRTOS + LVGL             │
└──────────────────────────────────────────┘
```

**原则**: 上层调用下层, 不跨层访问
**优点**: 硬件变更只需改BSP/HAL层
**缺点**: 无运行时多态, 硬件变更需重新编译

---

## 模式2: 状态机控制

### LED_Flag状态机 (WSC-16)
```
0 → 全关
1 → 绿灯常亮 (正常倒计时)
2 → 红灯+蜂鸣器 (倒计时结束)
4 → 绿灯闪烁 (最后30秒)
```

### 继电器状态机 (SX-MK-010)
```
启动: SET Start_Pin + RESET stop_Pin + 启动PWM
停止: SET stop_Pin + RESET Start_Pin + 反向PWM
```

**原则**: 状态变量集中管理, 状态切换原子化

---

## 模式3: 超时保护

### 初始化超时
```c
// 反例: 死等
do { temp = LCD_StatusRead(); } while ((temp & 0x04) == 0);

// 正例: 超时返回
uint32_t start = HAL_GetTick();
do {
    if (HAL_GetTick() - start > timeout) return -1;
    temp = LCD_StatusRead();
} while ((temp & 0x04) == 0);
```

### 重试+兜底
```c
for (int i = 0; i < 8; i++) {
    if (LCD_Reset() == 0) break;
}
if (ret != 0) HAL_NVIC_SystemReset();  // 兜底整机复位
```

---

## 模式4: 看门狗喂狗策略

1. **初始化全程喂狗**: LCD初始化前每100ms喂狗
2. **阻塞前喂狗**: osDelay前喂狗
3. **循环中喂狗**: while循环内定期喂狗
4. **ISR不喂狗**: 中断服务程序不喂狗

```c
// 上电沉降窗口
for (int i = 0; i < 15; i++) {
    HAL_IWDG_Refresh(&hiwdg);
    HAL_Delay(100);  // 共1.5s
}
```

---

## 模式5: 帧协议设计

### TJC串口屏协议
```
[帧头] [数据] [0xFF] [0xFF] [0xFF]
```

### Modbus RTU
```
[地址] [功能码] [数据] [CRC16低] [CRC16高]
帧间间隔 ≥ 3.5字符
```

### HC-433 GTI协议
```
[GTI头] [类型] [时间4B] [保留6B] [固定] [帧尾]
```

**原则**: 帧头+长度/类型+数据+校验+帧尾

---

## 模式6: 环形缓冲UART接收

```c
// 中断单字节接收
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    ring_buf[write_idx++] = rx_byte;
    write_idx %= BUF_SIZE;
    HAL_UART_Receive_IT(huart, &rx_byte, 1);
}

// 超时判断帧完成
void uart_service(void) {
    if (HAL_GetTick() - last_rx_tick > FRAME_TIMEOUT) {
        // 帧接收完毕, 处理
    }
}
```

---

## 模式7: 放电电阻设计

```
τ = R × C
完全放电 ≈ 5τ

选型:
- 1kΩ / 0805
- 功耗 P = V²/R = 25/1000 = 25mW
- 0805额降后62.5mW ✓
- 放电时间 5τ = 5 × 1k × 470µF = 2.35s
```

---

## 模式8: 抗干扰设计

1. **软件帧校验**: 帧长度/CRC/帧头帧尾检查
2. **命令重发**: 无应答自动重发 (≤3次)
3. **时序延迟**: 强干扰源(刷卡)后延迟100-200ms
4. **硬件滤波**: UART线路加RC滤波
5. **屏蔽**: 敏感信号线屏蔽接地

---

## 模式9: Flash参数存储

```c
// XOR加密存储
void flash_write_encrypted(uint32_t addr, uint8_t *data, uint16_t len) {
    for (int i = 0; i < len; i++) {
        data[i] ^= XOR_KEY;
    }
    HAL_FLASH_Program(FLASH_TYPEPROGRAM_WORD, addr, *(uint32_t*)data);
}
```

---

## 模式10: 任务优先级设计

| 优先级 | 任务 | 说明 |
|--------|------|------|
| 最高 | ISR | 硬件中断 |
| 高 | DAQ_Task | 数据采集 |
| 中 | GUI_Task | 界面刷新 |
| 低 | Background | 后台处理 |

**原则**: 高频任务降频, 避免阻塞低速通信
