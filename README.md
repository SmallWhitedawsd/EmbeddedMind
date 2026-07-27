# EmbeddedMind

> 从 285 个真实工程会话中提炼的嵌入式开发 AI 知识库。

## 数据来源

- **285** 个 OpenCode 工程会话
- **8,578** 条消息
- **111,482** 个事件
- **248** 次代码修改
- **1,482 万 tokens** 投入

## 覆盖项目

| 项目 | MCU | 关键技术 |
|------|-----|----------|
| HelpPort | STM32F407VET6 | LT7580 + LVGL + FreeRTOS + AD7606 |
| WSC-16 | STM32F103 | RS485 三色灯 + HC-433 + Modbus RTU |
| SX-MK-010 | STM32F103 | 继电器控制 + RFID + RN7302 + Modbus |
| STM32 教学平台 | STM32F1/F4 | 74 个外设模块 |

## 知识统计

| 分类 | 文件数 | 知识点 |
|------|--------|--------|
| MCU 开发 | 2 | 13 |
| 固件工程 | 2 | 10 |
| LVGL GUI | 4 | 19 |
| 通信协议 | 3 | 25 |
| 硬件调试 | 1 | 8 |
| 项目知识 | 4 | 40+ |
| 调试模式 | 3 | 27 |
| **总计** | **19** | **142+** |

## 使用方式

### 作为 OpenCode Skill

将 `SKILL.md` 复制到 `~/.config/opencode/skills/embedded-engineer/SKILL.md`：

```bash
cp SKILL.md ~/.config/opencode/skills/embedded-engineer/SKILL.md
cp -r knowledge/ ~/.config/opencode/skills/embedded-engineer/knowledge/
cp -r patterns/ ~/.config/opencode/skills/embedded-engineer/patterns/
cp -r projects/ ~/.config/opencode/skills/embedded-engineer/projects/
```

### 作为 Agent 上下文

在 AGENTS.md 中添加：

```markdown
## Embedded Development
- 加载 EmbeddedMind/SKILL.md 获取嵌入式开发知识
- 遇到 STM32/LVGL/Modbus 问题时查阅 knowledge/ 目录
- 调试时参考 patterns/debugging-patterns.md
```

## 知识格式

每条知识包含：
- **场景** — 何时遇到
- **现象** — 观察到的表现
- **根因** — 深层原因
- **解决方案** — 具体方法（含代码）
- **验证** — 如何确认成功
- **来源** — 原始会话引用

## 关键经验

### 启动时序
- IWDG 30s 超时 = Prescaler=256, Reload=3750, LSI=32kHz
- 断电再上电需加放电电阻 (1kΩ/0805)
- LCD 初始化需 1.5s 上电沉降窗口

### 通信协议
- Modbus CRC16: 多项式 0xA001
- RS485 半双工切换延迟 ≥500ms
- SPI /32 = 2.625MHz (杜邦线极限)

### 调试方法论
- 先读 RCC_CSR 判断复位原因
- 先排除硬件再排查软件
- 关键信号用示波器确认
- 修改后连续 ≥10 次验证稳定性

## License

内部工程经验总结，仅限团队使用。
