---
name: stm32-verifier
description: |
  STM32 嵌入式项目验证技能。触发词：验证项目、快速验证、全量验证、硬件验证、检查改动、
  verify project、check config、validate peripheral、review init chain。
  融合 codegraph（代码图谱）、verification-methodology（验证方法论）、keil-workflow（编译烧录），
  自动执行 62 项检查规则，输出结构化 JSON 验证报告。
---

# STM32 Verifier

自动化嵌入式项目验证技能。融合代码理解、规则检查、编译验证和硬件自检四步流程，覆盖通用嵌入式 / ARM Cortex-M / STM32 专有三层共 62 项检查规则。

## 适用范围

| 平台 | 支持程度 |
|------|---------|
| STM32F0/F1/F2/F3/F4/F7 | 完全支持 |
| STM32G0/G4/L0/L4/H7 | 完全支持 |
| ESP32 | 部分支持（Layer 1 通用规则） |
| 其他 Cortex-M | 部分支持（Layer 1+2） |

## 验证模式

根据用户意图选择对应模式。触发词匹配时自动选择；无明确触发词时默认 `quick`。

| 模式 | 触发词 | 执行步骤 | 耗时 | 适用场景 |
|------|--------|---------|------|---------|
| **quick** | "快速验证"、"verify"、"check" | Step 1 + 2 | ~30s | 开发中频繁检查 |
| **full** | "全量验证"、"full verify" | Step 1 + 2 + 3 | ~2min | 提交前验证 |
| **hardware** | "硬件验证"、"hardware verify" | Step 1 + 2 + 3 + 4 | ~5min | 版本发布前 |
| **incremental** | "检查改动"、"check diff" | 仅 git diff 文件 | ~10s | 频繁提交快检 |

## 验证引擎（4 步流程）

### Step 1: 代码理解（内嵌 codegraph 能力）

使用 codegraph 工具分析项目代码结构：

1. **explore** — 扫描项目，识别外设初始化函数、ISR、FreeRTOS 任务
2. **callers** — 查找共享变量的访问点，标记跨任务/ISR 共享的变量
3. **impact** — 分析改动影响范围，构建"外设依赖图"
4. 输出：外设列表、ISR 列表、任务列表、共享变量列表

### Step 2: 规则检查（62 项三层规则）

基于 Step 1 的依赖图，逐项检查。规则分三层，技能根据项目类型自动选择适用层：

- **Layer 1: 通用嵌入式规则（20 项）** — volatile、ISR 长度、互斥锁、看门狗、栈溢出、I2C/SPI/UART 防御等
- **Layer 2: ARM Cortex-M 规则（12 项）** — NVIC 优先级、内存屏障、FromISR API、HardFault 等
- **Layer 3: STM32 专有规则（30 项）** — ADC/DAC/TIM/DMA 时钟使能、GPIO 模式、HAL 初始化链顺序、FreeRTOS 配置等

每项检查输出：ID、严重度（critical/high/medium/low）、状态（pass/fail/warn/skip）、证据、修复建议。

规则详情参见 `references/rules/` 目录下的分层文件。

### Step 3: 编译验证（内嵌 keil-workflow 能力）

仅 `full`、`hardware` 模式执行：

1. 调用 keil-workflow 编译项目，捕获 errors/warnings
2. 运行 cppcheck 静态分析（如可用）
3. 解析 .map 文件，提取 Flash/RAM 使用量

### Step 4: 硬件验证（可选）

仅 `hardware` 模式执行：

1. 通过 keil-workflow 烧录固件到目标板
2. 通过串口发送自检命令，读取关键寄存器值
3. 比对寄存器值与预期配置

## 输出格式

输出为 JSON 格式验证报告，结构如下：

```json
{
  "meta": {
    "tool": "stm32-verifier",
    "version": "1.0.0",
    "timestamp": "<ISO8601>",
    "project": "<项目名>",
    "platform": "<MCU型号>",
    "scan_mode": "<quick|full|hardware|incremental>",
    "rules_applied": ["layer1", "layer2", "layer3"]
  },
  "code_graph": {
    "files_scanned": 0,
    "symbols_found": 0,
    "peripherals": [],
    "isrs": [],
    "tasks": [],
    "shared_vars": []
  },
  "summary": {
    "total": 62,
    "pass": 0,
    "fail": 0,
    "warn": 0,
    "skip": 0,
    "by_severity": {
      "critical": {"total": 15, "fail": 0},
      "high": {"total": 19, "fail": 0},
      "medium": {"total": 19, "fail": 0},
      "low": {"total": 9, "fail": 0}
    }
  },
  "checks": [
    {
      "id": "STM-011",
      "dimension": "init_chain",
      "layer": "layer3",
      "title": "启动顺序：先 ADC DMA 后 TIM",
      "severity": "critical",
      "status": "fail",
      "file": "oscilloscope.c",
      "line": 183,
      "evidence": "代码片段或分析结果",
      "fix": "修复建议",
      "reference": "参考手册章节"
    }
  ],
  "compile": {
    "errors": 0,
    "warnings": 0,
    "flash_used": "",
    "ram_used": ""
  },
  "recommendations": []
}
```

每个检查项字段说明：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 检查项 ID，如 "STM-011"、"GEN-001" |
| `dimension` | string | 所属维度（init_chain/concurrency/peripheral/...） |
| `layer` | string | 所属层级（layer1/layer2/layer3） |
| `title` | string | 检查项标题 |
| `severity` | string | 严重度：critical / high / medium / low |
| `status` | string | 状态：pass / fail / warn / skip |
| `file` | string | 相关源文件路径 |
| `line` | number | 相关行号 |
| `evidence` | string | 证据：代码片段或分析结论 |
| `fix` | string | 修复建议 |
| `reference` | string | 参考文档（手册章节、规范条款） |

## 执行流程

### 用户说"快速验证"或类似触发词

```
1. 识别项目目录和 MCU 型号（从 .ioc 或 Keil 工程文件提取）
2. Step 1: codegraph explore → 输出外设/ISR/任务/共享变量列表
3. Step 2: 逐层逐项检查 → 收集 pass/fail/warn 结果
4. 输出 JSON 报告 + 人类可读摘要
```

### 用户说"全量验证"

```
1-3. 同 quick 模式
4. Step 3: keil-workflow compile → 编译结果 + .map 分析
5. 输出 JSON 报告（含 compile 段）
```

### 用户说"硬件验证"

```
1-4. 同 full 模式
5. Step 4: flash 烧录 → 串口自检 → 寄存器比对
6. 输出 JSON 报告（含 compile + hardware 段）
```

### 用户说"检查改动"

```
1. git diff --name-only → 获取改动文件列表
2. codegraph explore → 只分析改动文件及其依赖
3. 只检查受影响的规则项
4. 输出增量 JSON 报告
```

## 与现有技能的集成

| 技能 | 集成方式 |
|------|---------|
| **codegraph** | Step 1 调用 `explore` 识别符号，`callers` 找访问点，`impact` 分析影响范围 |
| **keil-workflow** | Step 3 调用 `workflow.py --steps compile,analyze` 编译并分析 .map 文件 |
| **verification-methodology** | 检查规则来源于 methodology 的 20 条教训 + 扩展至 62 项 |
| **stm32-hal-development** | HAL 初始化链顺序、外设配置正确性检查规则 |

数据流：

```
codegraph explore → 识别外设函数、ISR、任务
         |
codegraph callers → 找共享变量访问点
         |
verification rules → 逐项检查（62 项）
         |
keil-workflow compile → 编译验证（full/hardware 模式）
         |
JSON 报告输出
```

## 检查维度

62 项规则覆盖以下维度：

- **初始化链完整性** — 时钟使能顺序、GPIO 模式、HAL_LINKDMA、启动顺序
- **并发安全** — volatile 标记、互斥锁、FromISR API、内存屏障
- **外设配置** — ADC/DAC/TIM/DMA/SPI/I2C/UART 参数正确性
- **错误处理** — 看门狗、栈溢出检测、I2C 总线恢复、Flash 保护
- **资源管理** — Flash/RAM 使用量、栈大小、堆分配
- **RTOS 集成** — 任务栈大小、优先级配置、对象创建时机

## 严重度定义

| 严重度 | 含义 | 示例 |
|--------|------|------|
| **Critical** | 必然导致功能故障或硬件异常 | DMA 时钟未使能、共享变量无 volatile |
| **High** | 大概率导致间歇性故障 | ISR 优先级冲突、缺少超时处理 |
| **Medium** | 特定条件下可能出问题 | SPI 时钟极性错误、GPIO 模式不匹配 |
| **Low** | 建议改进，不影响基本功能 | 时钟频率硬编码、缺少复位原因检查 |

## 使用示例

```
用户: 快速验证这个项目
→ 执行 quick 模式，输出 62 项静态检查结果

用户: 全量验证，我要准备提交了
→ 执行 full 模式，输出检查结果 + 编译结果

用户: 检查最近的改动有没有问题
→ 执行 incremental 模式，只检查 git diff 涉及的文件

用户: 验证 ADC+DMA 配置是否正确
→ 执行 quick 模式，重点报告 ADC/DMA 相关检查项
```
