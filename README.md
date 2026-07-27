# EmbeddedMind

> 从 285 个真实工程会话中提炼的嵌入式开发 AI 知识库。
> 
> 两轮炼化：142 知识点 + 155 Patch 案例 + 32 Bug 闭环 + 26 流程模式

## 数据来源

- **285** 个 OpenCode 工程会话
- **8,578** 条消息
- **111,482** 个事件
- **248** 次代码修改
- **58,052** 次命令执行
- **1,482 万 tokens** 投入

## 覆盖项目

| 项目 | MCU | 关键技术 |
|------|-----|----------|
| HelpPort | STM32F407VET6 | LT7580 + LVGL + FreeRTOS + AD7606 |
| WSC-16 | STM32F103 | RS485 三色灯 + HC-433 + Modbus RTU |
| SX-MK-010 | STM32F103 | 继电器控制 + RFID + RN7302 + Modbus |
| STM32 教学平台 | STM32F1/F4 | 74 个外设模块 |

## 知识统计

### 一轮：领域知识地图
| 分类 | 文件数 | 知识点 |
|------|--------|--------|
| MCU 开发 | 2 | 13 |
| 固件工程 | 2 | 10 |
| LVGL GUI | 4 | 19 |
| 通信协议 | 3 | 25 |
| 硬件调试 | 1 | 8 |
| 项目知识 | 4 | 40+ |
| 调试模式 | 3 | 27 |
| **小计** | **19** | **~142** |

### 二轮：深度经验库
| 分类 | 文件数 | 案例数 |
|------|--------|--------|
| Patch 案例 | 1 | 155 |
| Bug 闭环 | 6 目录 | 32 |
| 工作流程 | 3 | 18 |
| 推理方法 | 2 | 17 |
| **小计** | **12** | **222** |

**总知识点: 364+**

## 知识颗粒度

| 层级 | 描述 | 数量 | 示例 |
|------|------|------|------|
| L1 原子经验 | 单条踩坑/技巧 | 142+ | "断电再上电需加放电电阻" |
| L2 模式级 | 调试/设计模式 | 27 | "Modbus无应答五步排查法" |
| L3 案例级 | 完整Bug/Patch闭环 | 187 | "LT7580启动41秒完整推理链" |
| L4 方法论 | 工作流/推理框架 | 17 | "硬件异常四步排查顺序" |
| L5 项目级 | 项目完整知识 | 4 | HelpPort引脚+架构+已知问题 |

## 使用方式

### 作为 OpenCode Skill

```bash
cp SKILL.md ~/.config/opencode/skills/embedded-engineer/SKILL.md
cp -r knowledge/ ~/.config/opencode/skills/embedded-engineer/
cp -r deep_cases/ ~/.config/opencode/skills/embedded-engineer/
cp -r patterns/ ~/.config/opencode/skills/embedded-engineer/
cp -r projects/ ~/.config/opencode/skills/embedded-engineer/
```

### 作为 Agent 上下文

在 AGENTS.md 中添加：

```markdown
## Embedded Development
- 加载 EmbeddedMind/SKILL.md
- 遇到 Bug 先查 deep_cases/bug_cases/
- 需要修改先查 deep_cases/patch_cases.md
- 调试时参考 deep_cases/reasoning/debug-methodology-v2.md
```

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

## 项目结构

```
EmbeddedMind/
├── SKILL.md              ← Agent 加载入口
├── README.md             ← 本文件
├── LICENSE               ← MIT
├── SPEC.md               ← 炼丹规格
├── knowledge/            ← 一轮：领域知识 (12文件)
├── patterns/             ← 一轮：模式库 (3文件)
├── projects/             ← 一轮：项目知识 (4文件)
└── deep_cases/           ← 二轮：深度经验库
    ├── patch_cases.md    ← 155个代码修改案例
    ├── bug_cases/        ← 32个Bug闭环 (6目录)
    ├── workflows/        ← 18个标准流程
    └── reasoning/        ← 17个推理方法+架构
```

## License

MIT
