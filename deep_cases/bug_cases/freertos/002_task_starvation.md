# 002: 按键扫描任务因osDelay(10)周期过长丢触发

## 项目
WSC-16 (STM32F103 + FreeRTOS + 矩阵按键)

## 环境
MCU: STM32F103, RTOS: FreeRTOS, 按键: SW1~SW12矩阵扫描

## 现象
拨钮(自锁)开关快速切换时偶尔无法触发状态变化

## 影响范围
本地按键控制12路SOCKET

## 排查过程

### 假设1: 硬件抖动
- 验证方法：检查消抖逻辑
- 结果：已有20ms消抖（20次×10ms扫描）

### 假设2: 扫描周期过长
- 验证方法：计算完整按键动作所需时间
- 结果：扫描周期10ms，需连续20次确认=200ms有效窗口

### 假设3: 边沿触发vs电平跟随
- 验证方法：分析Process_Single_SW逻辑
- 结果：当前是边沿触发(XOR)，拨钮需电平跟随

## 根因
**点按键(边沿触发) vs 拨钮(电平跟随) 逻辑不兼容**：
- 当前：检测到低电平→XOR翻转bit
- 拨钮ON→持续低→每200ms翻转一次→状态闪烁

## 修改方案
```c
// 方案A: 改为电平跟随（去掉消抖和边沿检测）
static void Process_Single_SW(SW_Data_TypeDef *pSW, 
    GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin, uint8_t bit_offset)
{
    GPIO_PinState currentState = HAL_GPIO_ReadPin(GPIOx, GPIO_Pin);
    if (currentState == GPIO_PIN_RESET)  // 开关ON=低电平
        PowerControlState |= (1 << bit_offset);    // 置1
    else
        PowerControlState &= ~(1 << bit_offset);   // 置0
}

// 方案B: 保留消抖但简化判断
// 保留10~20ms消抖，判断逻辑从"边沿"改为"电平"
```

## 验证结果
拨钮开关状态稳定跟随，不再闪烁

## 预防措施
- 点按键用边沿触发(XOR)
- 拨钮/自锁开关用电平跟随(直接映射)
- 两种机制不能混用同一套逻辑

## 经验规则
- 按键动作≥20ms才能被可靠检测
- 扫描周期10ms时需20次连续确认
- 上位机与本地按键共用变量时注意竞争

## 来源
ses_09779b2f8ffeLgMJSxikVvAq4v - 2026-07-16
