# UART 环形缓冲区设计

## 场景
STM32 多串口通信中，中断接收数据需缓冲处理。

## 现象/问题
单字节中断接收效率低，高频数据易丢失。

## 根因分析
每字节触发一次 ISR，CPU 频繁上下文切换。

## 解决方案
```c
// my_uart.h - 环形缓冲定义
#define UART_BUF_SIZE 256

typedef struct {
    uint8_t  buffer[UART_BUF_SIZE];
    volatile uint16_t head;
    volatile uint16_t tail;
    volatile uint16_t count;
} UART_RingBuffer;

// my_uart.c - 中断接收写入环形缓冲
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart->Instance == UART5) {
        uint8_t data = UART5->DR;
        if (rx_buf.count < UART_BUF_SIZE) {
            rx_buf.buffer[rx_buf.head] = data;
            rx_buf.head = (rx_buf.head + 1) % UART_BUF_SIZE;
            rx_buf.count++;
        }
    }
}

// 读取数据
uint16_t UART5_Read(uint8_t *buf, uint16_t len) {
    uint16_t read = 0;
    while (rx_buf.count > 0 && read < len) {
        buf[read++] = rx_buf.buffer[rx_buf.tail];
        rx_buf.tail = (rx_buf.tail + 1) % UART_BUF_SIZE;
        rx_buf.count--;
    }
    return read;
}
```

## 验证方法
连续发送大量数据，确认无丢失。

## 来源
WSC-16 - UART5 RS485 control logic (2026-06-23)

---

# UART 帧接收超时检测

## 场景
RS485/Modbus 通信中，需判断一帧数据接收完毕。

## 现象/问题
不知道何时可以处理接收到的数据。

## 根因分析
Modbus RTU 帧间需至少 3.5 个字符时间的静默。

## 解决方案
```c
// USART5_RecService - 超时检测帧接收完毕
#define FRAME_TIMEOUT_MS 10  // 9600bps 下 10ms ≈ 10字节

void USART5_RecService(void) {
    static uint32_t last_rx_time = 0;
    static uint16_t last_count = 0;
    
    if (rx_buf.count != last_count) {
        last_count = rx_buf.count;
        last_rx_time = HAL_GetTick();
    } else if (rx_buf.count > 0 && 
               HAL_GetTick() - last_rx_time > FRAME_TIMEOUT_MS) {
        // 帧接收完毕，处理数据
        ProcessModbusFrame(rx_buf.buffer + rx_buf.tail, rx_buf.count);
        rx_buf.count = 0;
        rx_buf.tail = rx_buf.head;
    }
}
```

## 验证方法
发送完整帧后 10ms，触发帧处理回调。

## 来源
WSC-16 - UART5 RS485 control logic (2026-06-23)

---

# UART DMA 接收 vs 中断接收

## 场景
高频数据接收场景选择接收策略。

## 现象/问题
单字节中断在高速波特率下 CPU 负载过高。

## 根因分析
115200bps 下每 87μs 一次中断，CPU 频繁切换。

## 解决方案
```c
// 方案1：DMA + IDLE 中断（推荐）
void UART_Init_DMA(UART_HandleTypeDef *huart) {
    // 启动 DMA 接收
    HAL_UART_Receive_DMA(huart, dma_buf, DMA_BUF_SIZE);
    // 使能 IDLE 中断
    __HAL_UART_ENABLE_IT(huart, UART_IT_IDLE);
}

void USART_IRQHandler(UART_HandleTypeDef *huart) {
    if (__HAL_UART_GET_FLAG(huart, UART_FLAG_IDLE)) {
        __HAL_UART_CLEAR_IDLEFLAG(huart);
        uint16_t len = DMA_BUF_SIZE - __HAL_DMA_GET_COUNTER(huart->hdmarx);
        ProcessFrame(dma_buf, len);
        // 重启 DMA
        HAL_UART_Receive_DMA(huart, dma_buf, DMA_BUF_SIZE);
    }
}

// 方案2：单字节中断（简单场景）
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    // 处理 1 字节后重新启动接收
    HAL_UART_Receive_IT(huart, &rx_byte, 1);
}
```

## 验证方法
高速连续发送数据，对比 CPU 占用率。

## 来源
WSC-16 - UART5 RS485 control logic (2026-06-23)

---

# TJC 串口屏通信协议

## 场景
STM32 通过 USART3 与 TJC 触摸屏通信。

## 现象/问题
需了解帧格式和命令结构。

## 根因分析
TJC 协议使用帧头 + 数据 + 帧尾（3 个 0xFF）。

## 解决方案
```c
// TJC 协议帧格式
// 帧头: 无（直接发送数据）
// 数据: 命令字符串或 Modbus 帧
// 帧尾: 0xFF 0xFF 0xFF

void Uart3_SendBuff(char *str) {
    HAL_UART_Transmit(&huart3, (uint8_t *)str, strlen(str), 100);
    uint8_t end[3] = {0xFF, 0xFF, 0xFF};
    HAL_UART_Transmit(&huart3, end, 3, 100);
}

// 示例：同步按钮状态
Uart3_SendBuff("home.bt222.val=1");

// 示例：更新功率显示
char buf[64];
sprintf(buf, "home.POWER.txt=\"%.2f kW·h\"", power);
Uart3_SendBuff(buf);
```

**注意**：屏幕文件为 GB2312 编码，编辑时需字节安全。

## 验证方法
发送命令后，屏幕显示相应更新。

## 来源
SX-MK-010 - 触摸屏停止按钮继电器失效问题 (2026-07-10)

---

# UART 多协议复用

## 场景
同一 UART 外设需支持多种协议（如 Modbus + 自定义协议）。

## 现象/问题
不同设备协议不同，需动态切换解析器。

## 根因分析
UART5 复用灯带 Modbus 和 433 无线数据两条业务线。

## 解决方案
```c
// 协议分发器
void UART5_RxHandler(uint8_t *data, uint16_t len) {
    if (data[0] == 0x01 && (data[1] == 0x03 || data[1] == 0x06)) {
        // Modbus RTU
        Modbus_Process(data, len);
    } else if (strncmp((char *)data, "GTI", 3) == 0) {
        // GTI 协议（433 无线）
        GTI_Process(data, len);
    }
}
```

## 验证方法
分别发送 Modbus 和 GTI 帧，确认正确分发。

## 来源
WSC-16 - UART5 RS485 control logic (2026-06-23)

---

# UART 发送阻塞与 DMA 选择

## 场景
发送大量数据时选择阻塞或非阻塞方式。

## 现象/问题
`HAL_UART_Transmit` 阻塞等待发送完成，影响实时性。

## 根因分析
阻塞发送在 9600bps 下每字节耗时 ~1ms，长帧阻塞时间长。

## 解决方案
```c
// 阻塞发送（简单，适合短帧）
void Uart5_SendBuff_Block(uint8_t *data, uint16_t len) {
    HAL_UART_Transmit(&huart5, data, len, 100);
}

// DMA 发送（非阻塞，适合长帧）
void Uart5_SendBuff_DMA(uint8_t *data, uint16_t len) {
    HAL_UART_Transmit_DMA(&huart5, data, len);
    // 发送完成后触发 HAL_UART_TxCpltCallback
}

// 中断发送（平衡方案）
void Uart5_SendBuff_IT(uint8_t *data, uint16_t len) {
    HAL_UART_Transmit_IT(&huart5, data, len);
}
```

**选择原则**：
- < 10 字节：阻塞发送
- 10-100 字节：中断发送
- > 100 字节：DMA 发送

## 验证方法
对比三种方式的发送延迟和 CPU 占用。

## 来源
WSC-16 - UART5 RS485 control logic (2026-06-23)
