# 003: ESP01 WiFi模块通信调试

## 项目
单片机设备/STM32/ESP01WIFI (STM32F103 + ESP01)

## 环境
MCU: STM32F103, WiFi模块: ESP01(ESP8266), 通信: UART→AT指令

## 现象
ESP01与电脑同热点透传测试

## 影响范围
WiFi透传功能

## 排查过程

### 假设1: AT指令正确性
- 验证方法：发送AT测试指令
- 结果：AT指令格式正确

### 假设2: WiFi连接配置
- 验证方法：检查gzs2.4热点配置
- 结果：热点SSID/密码正确

### 假设3: 透传模式配置
- 验证方法：检查CIPMODE/CIPSTART配置
- 结果：需配置为透传模式

## 根因
ESP01需正确配置为STA模式+连接热点+建立TCP连接后才能透传

## 修改方案
```c
// ESP01初始化序列
"AT+RST\r\n"                    // 复位
"AT+CWMODE=1\r\n"              // STA模式
"AT+CWJAP=\"SSID\",\"PWD\"\r\n" // 连接热点
"AT+CIPSTART=\"TCP\",\"IP\",port\r\n" // 建立TCP
"AT+CIPMODE=1\r\n"             // 透传模式
"AT+CIPSEND\r\n"               // 开始透传
```

## 验证结果
WiFi透传正常

## 预防措施
- ESP01需3.3V供电，电流>250mA
- AT指令需等待"OK"响应后再发下一条
- 透传模式下+++退出

## 经验规则
- ESP01默认波特率115200
- AT指令以\r\n结尾
- 连接WiFi需等待几秒

## 来源
ses_06ce2977bffee3dArshb9w9MJ2 - 2026-07-24
