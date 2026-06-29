# STM32 Verifier -- Verification Flow

本文档详细定义 stm32-verifier 四种验证模式的执行流程、触发条件和预期输出。

---

## 模式总览

```
                Step 1          Step 2          Step 3          Step 4
              代码理解         规则检查         编译验证         硬件验证
quick         [====]           [====]
full          [====]           [====]           [====]
hardware      [====]           [====]           [====]           [====]
incremental   [==]             [==]
              (仅 diff 文件)   (仅受影响规则)
```

---

## Mode 1: quick

### 触发词

| 触发词 | 优先级 |
|--------|--------|
| "快速验证"、"quick verify"、"quick check" | 高 |
| "验证项目"、"verify project" | 中（无其他模式触发词时） |
| "检查配置"、"check config" | 中 |
| "validate peripheral" | 中 |
| 无明确触发词 | 默认 |

### 执行流程

```
[用户输入] --> 识别项目目录和 MCU 型号
                |
                v
[Step 1] 代码理解
  1.1  扫描项目根目录，识别 .ioc 文件或 Keil .uvprojx 文件
  1.2  从 .ioc 提取 MCU 型号（如 STM32F407VGTx）
  1.3  codegraph explore:
       - 扫描 Core/Src/ 和 Drivers/ 目录下的 .c/.h 文件
       - 识别外设初始化函数（MX_*_Init 模式）
       - 识别中断处理函数（*_IRQHandler 模式）
       - 识别 FreeRTOS 任务函数（xTaskCreate 签名）
  1.4  codegraph callers:
       - 查找全局变量在 ISR / 任务 / main 中的访问点
       - 标记跨上下文共享的变量
  1.5  输出: peripherals[], isrs[], tasks[], shared_vars[]
                |
                v
[Step 2] 规则检查
  2.1  根据平台确定适用规则层:
       - STM32 系列 → layer1 + layer2 + layer3（62 项）
       - 其他 Cortex-M → layer1 + layer2（32 项）
       - 其他嵌入式 → layer1（20 项）
  2.2  逐项执行检查规则:
       - 每条规则读取相关源文件
       - 匹配代码模式，判定 pass/fail/warn/skip
       - 记录证据（evidence）和修复建议（fix）
  2.3  汇总统计: total / pass / fail / warn / skip / by_severity
                |
                v
[输出] JSON 报告 + 人类可读摘要
```

### 预期输出

```json
{
  "meta": {
    "scan_mode": "quick",
    "rules_applied": ["layer1", "layer2", "layer3"]
  },
  "compile": null,
  "hardware": null
}
```

- `compile` 字段为 null（未执行编译）
- `hardware` 字段为 null（未执行硬件验证）
- 人类可读摘要格式:

```
[stm32-verifier] Quick Scan Complete
  Project: scope-siggen  |  Platform: STM32F407VGTx  |  Rules: 62
  PASS: 54  |  FAIL: 3  |  WARN: 4  |  SKIP: 1
  Critical failures: 1  (STM-011: DMA clock not enabled before TIM start)
  Run "full verify" for compile verification.
```

---

## Mode 2: full

### 触发词

| 触发词 | 优先级 |
|--------|--------|
| "全量验证"、"full verify"、"full check" | 高 |
| "准备提交"、"pre-commit verify" | 中 |
| "编译验证"、"compile verify" | 中 |

### 执行流程

```
[Step 1 + Step 2]  同 quick 模式（完整执行）
                |
                v
[Step 3] 编译验证
  3.1  定位 Keil 工程文件:
       - 搜索 *.uvprojx 文件
       - 若未找到，尝试 Makefile / CMakeLists.txt
  3.2  调用 keil-workflow 编译:
       - 执行: workflow.py --steps compile --project <path>
       - 捕获 stdout/stderr
       - 解析编译错误和警告
  3.3  cppcheck 静态分析（如可用）:
       - 执行: cppcheck --enable=all --std=c11 <src_dir>
       - 合并结果到 checks 数组
  3.4  解析 .map 文件:
       - 提取 Flash 使用量（text + data + rodata）
       - 提取 RAM 使用量（data + bss）
       - 计算使用百分比（相对于芯片 Flash/RAM 容量）
  3.5  输出: compile.{errors, warnings, flash_used, ram_used}
                |
                v
[输出] JSON 报告（含 compile 段）+ 人类可读摘要
```

### 预期输出

```json
{
  "meta": {
    "scan_mode": "full",
    "rules_applied": ["layer1", "layer2", "layer3"]
  },
  "compile": {
    "errors": 0,
    "warnings": 2,
    "flash_used": "128KB / 1024KB (12.5%)",
    "ram_used": "48KB / 192KB (25.0%)"
  },
  "hardware": null
}
```

- `hardware` 字段为 null（未执行硬件验证）
- 人类可读摘要格式:

```
[stm32-verifier] Full Scan Complete
  Project: scope-siggen  |  Platform: STM32F407VGTx  |  Rules: 62
  PASS: 54  |  FAIL: 3  |  WARN: 4  |  SKIP: 1
  Critical failures: 1  (STM-011: DMA clock not enabled before TIM start)
  Compile: 0 errors, 2 warnings
  Flash: 128KB / 1024KB (12.5%)  |  RAM: 48KB / 192KB (25.0%)
```

---

## Mode 3: hardware

### 触发词

| 触发词 | 优先级 |
|--------|--------|
| "硬件验证"、"hardware verify"、"hardware check" | 高 |
| "烧录验证"、"flash verify" | 中 |
| "寄存器检查"、"register check" | 中 |
| "版本发布前验证"、"release verify" | 中 |

### 执行流程

```
[Step 1 + Step 2 + Step 3]  同 full 模式（完整执行）
                |
                v
[Step 4] 硬件验证
  4.1  烧录固件:
       - 调用 keil-workflow: workflow.py --steps flash
       - 等待烧录完成，检查返回码
       - 烧录失败则终止，输出错误信息
  4.2  串口自检:
       - 通过配置的串口（默认 /dev/ttyUSB0 或 COM 端口）发送自检命令
       - 自检命令格式: {"cmd": "selftest", "id": "<timestamp>"}
       - 等待响应，超时 5 秒
  4.3  寄存器比对:
       - 读取关键外设寄存器值（通过串口自检响应）
       - 比对列表:
         | 外设 | 寄存器 | 期望值来源 |
         |------|--------|-----------|
         | RCC  | AHB1ENR | .ioc 中启用的外设 |
         | ADC  | CR1/CR2 | MX_ADC_Init 参数 |
         | TIM  | PSC/ARR | MX_TIM_Init 参数 |
         | DMA  | SxCR   | MX_DMA_Init 参数 |
         | GPIO | MODER   | .ioc GPIO 配置 |
       - 每个寄存器: match / mismatch / not_readable
  4.4  输出: hardware.{flash_ok, serial_ok, registers[]}
                |
                v
[输出] JSON 报告（含 compile + hardware 段）+ 人类可读摘要
```

### 预期输出

```json
{
  "meta": {
    "scan_mode": "hardware",
    "rules_applied": ["layer1", "layer2", "layer3"]
  },
  "compile": {
    "errors": 0,
    "warnings": 2,
    "flash_used": "128KB / 1024KB (12.5%)",
    "ram_used": "48KB / 192KB (25.0%)"
  },
  "hardware": {
    "flash_ok": true,
    "serial_ok": true,
    "registers": [
      {
        "peripheral": "RCC",
        "register": "AHB1ENR",
        "expected": "0x00100007",
        "actual": "0x00100007",
        "status": "match"
      },
      {
        "peripheral": "ADC1",
        "register": "CR2",
        "expected": "0x00000001",
        "actual": "0x00000000",
        "status": "mismatch"
      }
    ]
  }
}
```

- 人类可读摘要格式:

```
[stm32-verifier] Hardware Scan Complete
  Project: scope-siggen  |  Platform: STM32F407VGTx  |  Rules: 62
  PASS: 54  |  FAIL: 3  |  WARN: 4  |  SKIP: 1
  Critical failures: 1  (STM-011: DMA clock not enabled before TIM start)
  Compile: 0 errors, 2 warnings
  Flash: 128KB / 1024KB (12.5%)  |  RAM: 48KB / 192KB (25.0%)
  Hardware: flash OK, serial OK, registers 5/6 match (1 mismatch: ADC1.CR2)
```

---

## Mode 4: incremental

### 触发词

| 触发词 | 优先级 |
|--------|--------|
| "检查改动"、"check diff"、"check changes" | 高 |
| "增量验证"、"incremental verify" | 高 |
| "最近改动"、"recent changes" | 中 |
| "diff 检查"、"diff check" | 中 |

### 执行流程

```
[用户输入] --> 获取 git diff
                |
                v
[Git Diff 解析]
  1.1  执行: git diff --name-only HEAD
       - 获取自上次提交以来改动的文件列表
  1.2  过滤: 仅保留 .c / .h 文件
  1.3  若无改动文件，直接输出空报告并退出
  1.4  执行: git diff --unified=3 HEAD -- <files>
       - 获取每个文件的具体改动行
                |
                v
[Step 1] 代码理解（受限范围）
  1.1  codegraph explore: 仅扫描改动文件
  1.2  codegraph callers: 仅查找改动文件中变量的访问点
  1.3  输出: 受影响的 peripherals[], isrs[], tasks[], shared_vars[]
                |
                v
[Step 2] 规则检查（仅受影响规则）
  2.1  根据改动文件类型选择受影响的规则子集:
       - 改动涉及 ISR → 检查并发安全规则（Layer 1/2）
       - 改动涉及外设初始化 → 检查初始化链规则（Layer 3）
       - 改动涉及全局变量 → 检查 volatile / 互斥规则
       - 改动涉及 FreeRTOS API → 检查 RTOS 规则
  2.2  仅执行受影响的规则项（非全部 62 项）
  2.3  输出: 受影响规则的 pass/fail/warn 结果
                |
                v
[输出] 增量 JSON 报告（含 diff 上下文）
```

### 预期输出

```json
{
  "meta": {
    "scan_mode": "incremental",
    "rules_applied": ["layer1", "layer2", "layer3"]
  },
  "code_graph": {
    "files_scanned": 3,
    "changed_files": [
      "Core/Src/oscilloscope.c",
      "Core/Src/stm32f4xx_it.c",
      "Core/Inc/oscilloscope.h"
    ],
    "diff_summary": {
      "insertions": 42,
      "deletions": 18
    }
  },
  "summary": {
    "total": 8,
    "pass": 6,
    "fail": 1,
    "warn": 1,
    "skip": 0
  },
  "checks": [
    {
      "id": "STM-011",
      "dimension": "init_chain",
      "layer": "layer3",
      "title": "启动顺序：先 ADC DMA 后 TIM",
      "severity": "critical",
      "status": "fail",
      "file": "Core/Src/oscilloscope.c",
      "line": 183,
      "diff_context": "@@ -180,6 +180,10 @@\n+  HAL_TIM_Base_Start(&htim8);\n+  HAL_ADC_Start_DMA(&hadc1, ...);",
      "evidence": "TIM8 在 ADC DMA 之前启动",
      "fix": "交换 HAL_ADC_Start_DMA 和 HAL_TIM_Base_Start 的调用顺序",
      "reference": "RM0090 Section 13.3.15"
    }
  ],
  "compile": null,
  "hardware": null
}
```

- `compile` 和 `hardware` 字段为 null（incremental 模式不执行编译和硬件验证）
- `checks` 仅包含受影响的规则项（远少于 62 项）
- `diff_context` 字段为增量模式特有，展示相关代码改动

### 与 full 模式的区别

| 维度 | incremental | full |
|------|------------|------|
| 扫描范围 | 仅 git diff 文件 | 全部项目文件 |
| 规则范围 | 仅受影响规则子集 | 全部 62 项 |
| 耗时 | ~10s | ~2min |
| 编译验证 | 否 | 是 |
| 硬件验证 | 否 | 否 |
| 输出特有字段 | diff_context, changed_files | compile |

---

## 模式选择逻辑

```python
def select_mode(user_input: str) -> str:
    """根据用户输入选择验证模式。"""
    input_lower = user_input.lower()

    # 优先级 1: 明确的模式触发词
    if any(kw in input_lower for kw in ["硬件验证", "hardware verify", "hardware check",
                                         "烧录验证", "flash verify", "寄存器检查"]):
        return "hardware"

    if any(kw in input_lower for kw in ["全量验证", "full verify", "full check",
                                         "准备提交", "pre-commit"]):
        return "full"

    if any(kw in input_lower for kw in ["检查改动", "check diff", "check changes",
                                         "增量验证", "incremental", "最近改动"]):
        return "incremental"

    if any(kw in input_lower for kw in ["快速验证", "quick verify", "quick check"]):
        return "quick"

    # 优先级 2: 通用触发词 → quick
    if any(kw in input_lower for kw in ["验证", "verify", "check", "validate"]):
        return "quick"

    # 默认: quick
    return "quick"
```

---

## 错误处理

### Step 1 失败

- codegraph explore 无法识别项目结构 → 输出错误，建议手动指定项目路径
- .ioc / .uvprojx 文件缺失 → 跳过平台特定规则，仅执行 Layer 1

### Step 2 失败

- 单条规则执行异常 → 该规则标记为 `skip`，继续执行后续规则
- 源文件不可读 → 该文件相关规则标记为 `skip`

### Step 3 失败

- Keil 未安装或工程文件缺失 → 跳过编译验证，`compile` 字段设为 null
- 编译失败（errors > 0）→ 正常记录，不中断流程

### Step 4 失败

- 目标板未连接 → 输出错误，建议检查 USB/ST-Link 连接
- 串口无响应 → `serial_ok: false`，跳过寄存器比对
- 烧录失败 → 输出错误码，终止 Step 4

---

## 输出目标

所有模式均输出到两个目标:

1. **JSON 报告** — 结构化数据，供程序消费（完整 schema 见 `output-schema.md`）
2. **人类可读摘要** — 文本格式，直接显示在终端，包含关键指标和失败项高亮
