# 005: 启动摄像头控制bat编码错误

## 项目
WIFI_CAM (Python + bat脚本 + 摄像头控制)

## 环境
Windows CMD, Python, 摄像头: WIFL_8K Series (192.168.1.1)

## 现象
双击bat文件提示"不是内部或外部命令"，无法启动

## 影响范围
摄像头控制工具无法启动

## 排查过程

### 假设1: bat文件编码问题
- 验证方法：检查文件编码
- 结果：bat是UTF-8编码，Windows CMD默认用GBK(ANSI)解析→中文路径乱码

### 假设2: Python路径问题
- 验证方法：检查python是否在PATH中
- 结果：python可执行但bat中的中文路径无法解析

## 根因
**bat文件UTF-8编码 + Windows CMD默认GBK编码** → 中文路径/命令被错误解析

## 修改方案
```bat
:: 方案1: 将bat保存为ANSI编码(GBK)
:: 方案2: 在bat开头加chcp 65001并保存为UTF-8
:: 方案3: 去掉所有中文，用英文路径
@echo off
chcp 65001
python camera_control.py
```

## 验证结果
bat正常启动摄像头控制程序

## 预防措施
- Windows bat文件保存为ANSI编码
- 避免在bat中使用中文路径
- 或用chcp 65001切换代码页

## 经验规则
- Windows CMD默认GBK(CP936)
- bat文件中文路径需ANSI编码
- PowerShell默认UTF-8无此问题

## 来源
ses_14fb4a455ffe60601C6DDb09l2 - 2026-06-17
