# Spec: EmbeddedMind — OpenCode 历史聊天记录知识炼丹

## Objective

将 285 个 OpenCode 历史工程会话（8578 条消息、111k 事件、248 次代码修改）提炼为高质量嵌入式开发 AI Skill 知识库。

**最终产物**：`EmbeddedMind` 项目，可被 OpenCode / AI Coding Agent 加载为嵌入式领域专家知识层。

**成功标准**：
- 覆盖 4 个核心项目（HelpPort/WSC-16/SX-MK-010/STM32单片机）
- 提炼 ≥50 个 Bug 解决案例
- 提炼 ≥30 个代码模式/架构模式
- 生成可被 Agent 加载的 SKILL.md 入口
- 知识去重率 ≥80%（不重复无意义内容）

## Tech Stack

- SQLite3 数据提取
- Markdown 知识表示
- OpenCode Skill 格式（SKILL.md）
- 向量检索（可选，后期）

## Commands

```bash
# 数据提取
sqlite3 opencode.db "SELECT ..."

# 项目结构生成
python extract_knowledge.py

# 验证
python validate_knowledge.py
```

## Project Structure

```
EmbeddedMind/
├── SKILL.md                    # Agent 加载入口
├── SPEC.md                     # 本文档（炼丹规格）
├── README.md                   # 项目说明
├── knowledge/                  # 领域知识库
│   ├── mcu/                    # MCU 开发经验
│   │   ├── stm32-f1.md         # STM32F1 系列
│   │   ├── stm32-f4.md         # STM32F4 系列
│   │   ├── peripherals.md      # 外设配置（UART/DMA/I2C/SPI/ADC）
│   │   └── debug-tips.md       # 调试技巧
│   ├── firmware/                # 固件工程
│   │   ├── freertos.md         # FreeRTOS 经验
│   │   ├── boot-sequence.md    # 启动流程
│   │   ├── watchdog.md         # 看门狗经验
│   │   └── state-machine.md    # 状态机设计
│   ├── lvgl/                   # LVGL GUI
│   │   ├── architecture.md     # 架构模式
│   │   ├── display-driver.md   # 显示驱动
│   │   ├── touch-driver.md     # 触摸驱动
│   │   └── performance.md      # 性能优化
│   ├── protocols/              # 通信协议
│   │   ├── modbus.md           # Modbus RTU/TCP
│   │   ├── rs485.md            # RS485 硬件/软件
│   │   └── uart-fifo.md        # UART FIFO 设计
│   ├── hardware-debug/         # 硬件调试
│   │   ├── ov7670.md           # 摄像头调试
│   │   ├── power-supply.md     # 电源问题
│   │   └── signal-integrity.md # 信号完整性
│   └── reverse-engineering/    # 逆向工程
│       ├── android-apk.md      # APK 逆向
│       ├── native-so.md        # .so 分析
│       └── protocol-reverse.md # 协议逆向
├── patterns/                   # 设计/调试模式
│   ├── debugging-patterns.md   # 调试方法论
│   ├── design-patterns.md      # 代码设计模式
│   └── troubleshooting.md      # 故障排查决策树
├── projects/                   # 项目专属知识
│   ├── helpport.md             # HelpPort 项目
│   ├── wsc-16.md               # WSC-16 三色灯
│   ├── sx-mk-010.md            # SX-MK-010 电源
│   └── stm32-modules.md        # STM32 模块库
└── examples/                   # 代码示例
    ├── uart-fifo-example.md
    ├── ad7606-dma-example.md
    ├── lvgl-init-example.md
    └── modbus-master-example.md
```

## Code Style

知识条目格式：

```markdown
# [问题/模式名称]

## 场景
何时遇到此问题。

## 现象
观察到的具体表现。

## 根因
深层原因分析。

## 解决方案
具体修改步骤（含代码）。

## 验证方法
如何确认修复成功。

## 关联项目
HelpPort / WSC-16 / SX-MK-010
```

## Testing Strategy

- 每个知识条目必须来自 ≥1 个真实会话
- 代码示例必须经过验证（编译通过或硬件测试通过）
- 不收录未经验证的理论描述
- 同一问题多个案例取最完整的一个

## Boundaries

- Always: 标注知识来源（session_id / 日期）
- Always: 保留原始代码修改上下文
- Ask first: 涉及未公开代码时脱敏
- Never: 复制粘贴聊天原文
- Never: 收录闲聊/问候/无关内容
- Never: 收录错误方案（只保留最终有效方案）

## Success Criteria

- [ ] SKILL.md 可被 OpenCode 正确加载
- [ ] 知识库覆盖 4 个核心项目的 ≥90% 技术主题
- [ ] 每个 knowledge/ 文件有 ≥3 个真实案例支撑
- [ ] 无重复条目（相似度 <70%）
- [ ] 总文件大小 <5MB（不含 snapshot）

## Open Questions

1. 是否需要向量检索层？（后期扩展）
2. 代码示例是否需要脱敏？
3. 是否收录网络配置/系统优化类内容？
