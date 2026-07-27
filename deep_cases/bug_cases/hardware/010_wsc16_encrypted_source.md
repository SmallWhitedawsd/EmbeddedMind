# 010: WSC-16加密源码无法调试

## 项目
SX-MK-010-0140 (STM32F103 + 加密源码)

## 环境
MCU: STM32F103, 编译器: Keil/eIDE, 部分源码加密

## 现象
大量核心源文件被加密(头部`18 1B 03 1A 15 10 19 7C`)，无法读取和调试

## 影响范围
Bug定位、功能修改

## 排查过程

### 假设1: 文件损坏
- 验证方法：检查文件头
- 结果：统一头部`18 1B 03 1A...`→故意加密

### 假设2: 编码问题
- 验证方法：尝试不同编码
- 结果：非文本文件，是加密/混淆

### 假设3: 哪些文件可读
- 验证方法：逐个检查
- 结果：仅Control.c、NTC、Dht11等少数文件可读

## 根因
项目使用eIDE的代码保护功能对核心源码加密(RN7302、modbusrtu、Power_Supply_Testing等)

## 修改方案
```
可读文件: Control.c, NTC, Dht11, DS1302, MMU, CRC16, my_Encryption, WIFI.c
加密文件: RN7302.c/.h, Power_Supply_Testing, modbusrtu, my_uart, my_flashData, 
          mfrc522, LED, TJC_TFT, Bluetooth.c
```

## 验证结果
只能分析Control.c中的顶层逻辑，无法深入RN7302计量内部

## 预防措施
- 核心算法加密需保留解密密钥
- 交付客户代码前确认调试需求
- 加密影响Bug修复效率

## 经验规则
- 加密文件头部有固定magic bytes
- 可读文件足够分析顶层控制逻辑
- 修改加密文件需eIDE解密密钥

## 来源
ses_0b4c429ceffeBuCyAo6VBOj1Nb - 2026-07-10
