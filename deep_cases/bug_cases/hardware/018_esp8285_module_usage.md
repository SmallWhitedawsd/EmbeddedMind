# 018: ESP8285模块用途与玩法

## 项目
ESP_8285 (WiFi模块开发)

## 环境
ESP8285(ESP8266变种), 2.4GHz WiFi

## 现象
探索ESP8285模块的用途和玩法

## 影响范围
IoT项目开发

## 排查过程

### 假设1: AT指令固件
- 验证方法：发送AT指令测试
- 结果：默认AT固件可工作

### 假设2: SDK开发
- 验证方法：检查SDK支持
- 结果：支持Arduino/ESP-IDF/NodeMCU

## 根因
ESP8285=ESP8266+内置Flash，支持AT固件和SDK开发两种模式

## 修改方案
```c
// 模式1: AT指令(串口透传)
"AT+CWMODE=1\r\n"      // STA模式
"AT+CWJAP=\"SSID\",\"PWD\"\r\n"
"AT+CIPSTART=\"TCP\",\"IP\",port\r\n"
"AT+CIPSEND\r\n"

// 模式2: Arduino SDK
#include <ESP8266WiFi.h>
WiFi.begin("SSID", "PWD");
WiFiClient client;
client.connect(IP, port);

// 模式3: MicroPython
import network
wlan = network.WLAN(network.STA_IF)
wlan.connect('SSID', 'PWD')
```

## 验证结果
三种模式均可正常工作

## 预防措施
- ESP8285=ESP8266+1MB Flash
- AT固件简单但功能受限
- SDK开发灵活但需编译环境

## 经验规则
- ESP8285默认波特率115200
- 3.3V供电，TX/RX也是3.3V
- 透传模式+++退出

## 来源
ses_12fbfe558ffex5I85oKLo6k7OL - 2026-06-17
