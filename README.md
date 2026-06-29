# STM32 Verifier

STM32 嵌入式项目自动化验证工具。深度融合 codegraph（代码图谱）、verification-methodology（验证方法论）和 keil-workflow（编译/烧录）。

## 功能

### 62 项检查规则

| 层级 | 规则数 | Critical | High | Medium | Low |
|------|--------|----------|------|--------|-----|
| Layer 1: 通用嵌入式 | 20 | 2 | 8 | 8 | 2 |
| Layer 2: ARM Cortex-M | 12 | 3 | 3 | 1 | 5 |
| Layer 3: STM32 专有 | 30 | 10 | 8 | 10 | 2 |
| **总计** | **62** | **15** | **19** | **19** | **9** |

### 4 种验证模式

| 模式 | 触发词 | 执行内容 | 耗时 |
|------|--------|---------|------|
| **quick** | "快速验证" | 代码理解 + 62 项静态检查 | ~30s |
| **full** | "全量验证" | + 编译验证 | ~2min |
| **hardware** | "硬件验证" | + 烧录 + 串口自检 | ~5min |
| **incremental** | "检查改动" | 只检查 git diff 文件 | ~10s |

### 验证引擎（4 步）

```
Step 1: 代码理解（codegraph）
├─ explore → 识别外设函数、ISR、任务
├─ callers → 找共享变量访问点
└─ impact → 分析改动影响范围

Step 2: 规则检查（62 项）
├─ Layer 1: 通用嵌入式（20 项）
├─ Layer 2: ARM Cortex-M（12 项）
└─ Layer 3: STM32 专有（30 项）

Step 3: 编译验证（keil-workflow）
├─ 编译错误/警告
├─ cppcheck 静态分析
└─ .map 文件 Flash/RAM 使用

Step 4: 硬件验证（可选）
├─ 烧录固件
├─ 串口自检命令
└─ 寄存器比对
```

### JSON 输出

```json
{
  "meta": { "tool": "stm32-verifier", "version": "1.0.0" },
  "code_graph": { "peripherals": [...], "isrs": [...], "tasks": [...] },
  "summary": { "total": 62, "pass": 55, "fail": 5, "warn": 2 },
  "checks": [
    {
      "id": "STM-011",
      "title": "启动顺序：先 ADC DMA 后 TIM",
      "severity": "critical",
      "status": "fail",
      "evidence": "...",
      "fix": "..."
    }
  ],
  "compile": { "errors": 0, "warnings": 0 },
  "recommendations": [...]
}
```

## 文件结构

```
stm32-verifier/
├── SKILL.md                    (243 行)
└── references/
    ├── rules/
    │   ├── layer1-general.md   (20 项通用规则)
    │   ├── layer2-arm.md       (12 项 ARM 规则)
    │   └── layer3-stm32.md     (30 项 STM32 规则)
    ├── verification-flow.md    (验证流程详细说明)
    └── output-schema.md        (JSON 输出格式)
```

## 集成能力

| 技能 | 集成方式 |
|------|---------|
| **codegraph** | Step 1 用 explore/callers/impact 分析代码 |
| **keil-workflow** | Step 3/4 调用 compile/analyze/flash |
| **verification-methodology** | 检查规则来自 20 条教训 + 扩展 |

## 安装

```bash
npx skills add pjjuihj/stm32-verifier -g
```

## 使用

```
用户: "快速验证这个项目"
→ 执行 quick 模式，输出 JSON 报告

用户: "全量验证 scope-siggen"
→ 执行 full 模式，包含编译验证

用户: "检查最近的改动"
→ 执行 incremental 模式，只检查 git diff 文件

用户: "验证 ADC+DMA 配置"
→ 执行 quick 模式，重点报告 ADC/DMA 相关检查项
```

## 版本

- v1.0.0 (2026-06-29) - 初始版本

## License

MIT
