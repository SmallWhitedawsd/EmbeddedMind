# 008: TFT屏幕MCU蓝牙导航显示

## 项目
TFT屏幕+蓝牙导航 (嵌入式显示)

## 环境
TFT屏幕, MCU, 蓝牙模块

## 现象
TFT屏幕需显示蓝牙导航信息

## 影响范围
导航显示功能

## 排查过程

### 假设1: 蓝牙通信协议
- 验证方法：检查蓝牙模块通信方式
- 结果：蓝牙串口透传

### 假设2: TFT显示驱动
- 验证方法：检查屏幕驱动IC
- 结果：需正确初始化TFT控制器

## 根因
TFT+蓝牙导航需解决：蓝牙数据接收→解析→TFT刷新显示的完整链路

## 修改方案
```c
// 蓝牙数据接收
void BT_RxCallback(uint8_t *data, uint16_t len) {
    // 解析导航指令
    parse_navigation(data, len);
}

// TFT显示更新
void TFT_UpdateNav(NavInfo *nav) {
    TFT_ShowString(10, 10, nav->direction);
    TFT_ShowNumber(10, 30, nav->distance);
}
```

## 验证结果
TFT正确显示蓝牙导航信息

## 预防措施
- 蓝牙数据需有帧头帧尾校验
- TFT刷新不能太频繁(>30fps无意义)
- 导航数据需平滑处理

## 经验规则
- 蓝牙波特率常用115200
- TFT局部刷新比全局刷新快
- 导航显示需考虑可读性

## 来源
ses_12f7c2e34ffep37WTzxlIZYuBw - 2026-06-17
