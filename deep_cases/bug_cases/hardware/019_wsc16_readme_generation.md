# 019: WSC-16项目README.md架构文档编写

## 项目
WSC-16 (STM32F103 + Modbus RTU)

## 环境
MCU: STM32F103, 协议: Modbus RTU, 通信: RS485

## 现象
需要为项目编写完整的架构文档

## 影响范围
项目可维护性、团队协作

## 排查过程

### 假设1: 代码结构不清
- 验证方法：分析目录结构
- 结果：Drivers/Middlewares/MDK-ARM/User四层结构

### 假设2: 核心流程不明确
- 验证方法：追踪main→Task→中断流程
- 结果：找到完整执行链路

## 根因
项目缺乏文档，新成员难以快速理解架构

## 修改方案
```markdown
# WSC-16 智能焊接工作站控制器

## 硬件
- MCU: STM32F103RCT6
- 通信: RS485(Modbus RTU) + HC433无线
- 显示: TJC TFT(UART)
- 输入: 矩阵按键SW1~SW12
- 输出: 12路继电器(SOCKET1~12)

## 软件架构
- RTOS: FreeRTOS
- 协议: Modbus RTU
- 任务: StartTask01(1s), UserTask01(10ms), GUI_Task
```

## 验证结果
README完整描述项目架构

## 预防措施
- 项目初期即编写README
- 文档随代码更新
- 包含硬件接线、软件架构、通信协议

## 经验规则
- README是项目的门面
- 包含: 硬件/软件/协议/任务/中断
- 新人通过README快速上手

## 来源
ses_131ec2effffeoeSICh9lC7izrW - 2026-06-18
