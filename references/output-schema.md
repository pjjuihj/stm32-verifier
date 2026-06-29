# STM32 Verifier -- Output Schema

本文档定义 stm32-verifier JSON 输出格式的完整 schema，用于程序化消费验证报告。

---

## 顶层结构

```json
{
  "meta": { ... },
  "code_graph": { ... },
  "summary": { ... },
  "checks": [ ... ],
  "compile": { ... } | null,
  "hardware": { ... } | null,
  "recommendations": [ ... ]
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `meta` | object | 是 | 元信息：工具版本、时间戳、项目、扫描模式 |
| `code_graph` | object | 是 | 代码图谱扫描结果 |
| `summary` | object | 是 | 检查结果汇总统计 |
| `checks` | array | 是 | 逐项检查结果列表 |
| `compile` | object \| null | 否 | 编译验证结果（full/hardware 模式） |
| `hardware` | object \| null | 否 | 硬件验证结果（hardware 模式） |
| `recommendations` | array | 是 | 优先修复建议列表 |

---

## meta

元信息字段，标识本次验证的基本上下文。

```json
{
  "meta": {
    "tool": "stm32-verifier",
    "version": "1.0.0",
    "timestamp": "2026-06-29T14:30:00+08:00",
    "project": "scope-siggen",
    "platform": "STM32F407VGTx",
    "scan_mode": "quick",
    "rules_applied": ["layer1", "layer2", "layer3"]
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `tool` | string | 是 | 固定值 `"stm32-verifier"` |
| `version` | string | 是 | 技能版本号，语义化版本 |
| `timestamp` | string | 是 | ISO 8601 格式的验证时间戳 |
| `project` | string | 是 | 项目名称（从目录名或 .ioc 文件提取） |
| `platform` | string | 是 | MCU 型号（从 .ioc 或 Keil 工程提取） |
| `scan_mode` | string | 是 | 扫描模式，枚举值: `quick` / `full` / `hardware` / `incremental` |
| `rules_applied` | string[] | 是 | 实际应用的规则层列表 |

### rules_applied 枚举值

| 值 | 含义 | 规则数量 |
|----|------|---------|
| `"layer1"` | 通用嵌入式规则 | 20 |
| `"layer2"` | ARM Cortex-M 规则 | 12 |
| `"layer3"` | STM32 专有规则 | 30 |

---

## code_graph

代码图谱扫描结果，由 Step 1（代码理解）产出。

```json
{
  "code_graph": {
    "files_scanned": 42,
    "symbols_found": 186,
    "peripherals": [
      {
        "name": "ADC1",
        "init_func": "MX_ADC1_Init",
        "file": "Core/Src/adc.c",
        "line": 32
      },
      {
        "name": "TIM8",
        "init_func": "MX_TIM8_Init",
        "file": "Core/Src/tim.c",
        "line": 78
      }
    ],
    "isrs": [
      {
        "name": "DMA2_Stream0_IRQHandler",
        "file": "Core/Src/stm32f4xx_it.c",
        "line": 120,
        "peripheral": "DMA2"
      }
    ],
    "tasks": [
      {
        "name": "OscilloscopeTask",
        "file": "Core/Src/oscilloscope.c",
        "line": 45,
        "stack_size": 512,
        "priority": "osPriorityNormal"
      }
    ],
    "shared_vars": [
      {
        "name": "adc_buffer",
        "type": "uint16_t[]",
        "file": "Core/Src/oscilloscope.c",
        "line": 18,
        "has_volatile": false,
        "accessed_by": ["DMA2_Stream0_IRQHandler", "OscilloscopeTask"]
      }
    ]
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `files_scanned` | number | 是 | 扫描的源文件数量 |
| `symbols_found` | number | 是 | 识别的符号总数 |
| `peripherals` | array | 是 | 外设初始化函数列表 |
| `isrs` | array | 是 | 中断处理函数列表 |
| `tasks` | array | 是 | FreeRTOS 任务列表 |
| `shared_vars` | array | 是 | 跨上下文共享变量列表 |

### peripherals[] 元素

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 外设名称（如 ADC1、TIM8、SPI1） |
| `init_func` | string | 是 | 初始化函数名（MX_*_Init 模式） |
| `file` | string | 是 | 初始化函数所在文件 |
| `line` | number | 是 | 初始化函数起始行号 |

### isrs[] 元素

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | ISR 函数名 |
| `file` | string | 是 | ISR 所在文件 |
| `line` | number | 是 | ISR 起始行号 |
| `peripheral` | string | 否 | 关联的外设名称 |

### tasks[] 元素

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 任务函数名 |
| `file` | string | 是 | 任务函数所在文件 |
| `line` | number | 是 | 任务函数起始行号 |
| `stack_size` | number | 否 | 栈大小（字节），从 xTaskCreate 参数提取 |
| `priority` | string | 否 | 优先级，从 xTaskCreate 参数提取 |

### shared_vars[] 元素

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 变量名 |
| `type` | string | 是 | 变量类型 |
| `file` | string | 是 | 变量定义所在文件 |
| `line` | number | 是 | 变量定义行号 |
| `has_volatile` | boolean | 是 | 是否已标记 volatile |
| `accessed_by` | string[] | 是 | 访问该变量的 ISR/任务/函数列表 |

---

## summary

检查结果汇总统计。

```json
{
  "summary": {
    "total": 62,
    "pass": 54,
    "fail": 3,
    "warn": 4,
    "skip": 1,
    "by_severity": {
      "critical": {
        "total": 15,
        "pass": 13,
        "fail": 1,
        "warn": 1,
        "skip": 0
      },
      "high": {
        "total": 19,
        "pass": 17,
        "fail": 1,
        "warn": 1,
        "skip": 0
      },
      "medium": {
        "total": 19,
        "pass": 16,
        "fail": 1,
        "warn": 2,
        "skip": 0
      },
      "low": {
        "total": 9,
        "pass": 8,
        "fail": 0,
        "warn": 0,
        "skip": 1
      }
    }
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `total` | number | 是 | 检查项总数（quick/full: 62, incremental: 受影响子集） |
| `pass` | number | 是 | 通过的检查项数量 |
| `fail` | number | 是 | 失败的检查项数量 |
| `warn` | number | 是 | 警告的检查项数量 |
| `skip` | number | 是 | 跳过的检查项数量（不适用或无法执行） |
| `by_severity` | object | 是 | 按严重度分类的统计 |

### by_severity 子结构

每个严重度级别（critical / high / medium / low）包含:

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `total` | number | 是 | 该严重度的检查项总数 |
| `pass` | number | 是 | 通过数 |
| `fail` | number | 是 | 失败数 |
| `warn` | number | 是 | 警告数 |
| `skip` | number | 是 | 跳过数 |

---

## checks

逐项检查结果列表。每个元素代表一条检查规则的执行结果。

```json
{
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
      "evidence": "HAL_TIM_Base_Start(&htim8) 在第 183 行调用，HAL_ADC_Start_DMA(&hadc1, ...) 在第 190 行调用。TIM 在 ADC DMA 之前启动。",
      "fix": "交换调用顺序：先调用 HAL_ADC_Start_DMA，再调用 HAL_TIM_Base_Start。参考 main.c 中 MX 的生成顺序。",
      "reference": "RM0090 Section 13.3.15: ADC must be configured and DMA started before TIM triggers conversion."
    },
    {
      "id": "GEN-003",
      "dimension": "concurrency",
      "layer": "layer1",
      "title": "ISR/任务共享变量需标记 volatile",
      "severity": "critical",
      "status": "fail",
      "file": "Core/Src/oscilloscope.c",
      "line": 18,
      "evidence": "变量 'adc_buffer' 在 DMA2_Stream0_IRQHandler 和 OscilloscopeTask 中访问，但未标记 volatile。",
      "fix": "声明改为 'volatile uint16_t adc_buffer[BUFFER_SIZE];'",
      "reference": "ARM Cortex-M Programming Guide: Memory access ordering"
    },
    {
      "id": "ARM-005",
      "dimension": "nvic",
      "layer": "layer2",
      "title": "FreeRTOS 管理的中断优先级需在有效范围内",
      "severity": "high",
      "status": "pass",
      "file": "Core/Src/stm32f4xx_it.c",
      "line": 120,
      "evidence": "DMA2_Stream0_IRQHandler 优先级为 5，在 configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY (5) 范围内。",
      "fix": "",
      "reference": "FreeRTOS Reference Manual: Interrupt priority configuration"
    },
    {
      "id": "GEN-010",
      "dimension": "watchdog",
      "layer": "layer1",
      "title": "看门狗应在 main 循环中定期喂狗",
      "severity": "medium",
      "status": "warn",
      "file": "Core/Src/main.c",
      "line": 95,
      "evidence": "IWDG 已初始化但未在 while(1) 中发现 HAL_IWDG_Refresh 调用。可能存在但位于其他任务中。",
      "fix": "在主循环或看门狗任务中添加 HAL_IWDG_Refresh(&hiwdg)。",
      "reference": "RM0090 Section 22.3: IWDG key register"
    }
  ]
}
```

### checks[] 元素字段定义

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 检查项 ID，格式为 `<前缀>-<数字>` |
| `dimension` | string | 是 | 所属维度（见维度枚举表） |
| `layer` | string | 是 | 所属规则层: `layer1` / `layer2` / `layer3` |
| `title` | string | 是 | 检查项标题（人类可读的简短描述） |
| `severity` | string | 是 | 严重度: `critical` / `high` / `medium` / `low` |
| `status` | string | 是 | 状态: `pass` / `fail` / `warn` / `skip` |
| `file` | string | 是 | 相关源文件路径（相对于项目根目录） |
| `line` | number | 是 | 相关行号 |
| `evidence` | string | 是 | 证据：代码片段或分析结论 |
| `fix` | string | 否 | 修复建议（pass 时可为空字符串） |
| `reference` | string | 否 | 参考文档（手册章节、规范条款） |

### ID 前缀规则

| 前缀 | 对应层 | 规则来源 | 示例 |
|------|--------|---------|------|
| `GEN-` | layer1 | 通用嵌入式规则 | GEN-001 ~ GEN-020 |
| `ARM-` | layer2 | ARM Cortex-M 规则 | ARM-001 ~ ARM-012 |
| `STM-` | layer3 | STM32 专有规则 | STM-001 ~ STM-030 |

### dimension 枚举

| 维度值 | 含义 | 典型检查项 |
|--------|------|-----------|
| `init_chain` | 初始化链完整性 | 时钟使能顺序、GPIO 模式、HAL_LINKDMA、启动顺序 |
| `concurrency` | 并发安全 | volatile、互斥锁、FromISR API、内存屏障 |
| `peripheral` | 外设配置 | ADC/DAC/TIM/DMA/SPI/I2C/UART 参数正确性 |
| `error_handling` | 错误处理 | 看门狗、栈溢出检测、I2C 总线恢复 |
| `resource` | 资源管理 | Flash/RAM 使用量、栈大小、堆分配 |
| `rtos` | RTOS 集成 | 任务栈大小、优先级配置、对象创建时机 |
| `nvic` | 中断管理 | NVIC 优先级、中断嵌套、HardFault 处理 |
| `gpio` | GPIO 配置 | 模式匹配、复用功能、上拉/下拉 |
| `clock` | 时钟配置 | PLL 参数、总线分频、外设时钟使能 |
| `dma` | DMA 配置 | 传输方向、循环模式、中断配置 |

### status 枚举

| 状态值 | 含义 | 计入 |
|--------|------|------|
| `pass` | 检查通过 | summary.pass |
| `fail` | 检查失败，存在确定性问题 | summary.fail |
| `warn` | 检查警告，可能存在但不确定 | summary.warn |
| `skip` | 检查跳过（不适用或无法执行） | summary.skip |

### severity 枚举

| 严重度 | 含义 | 示例 |
|--------|------|------|
| `critical` | 必然导致功能故障或硬件异常 | DMA 时钟未使能、共享变量无 volatile |
| `high` | 大概率导致间歇性故障 | ISR 优先级冲突、缺少超时处理 |
| `medium` | 特定条件下可能出问题 | SPI 时钟极性错误、GPIO 模式不匹配 |
| `low` | 建议改进，不影响基本功能 | 时钟频率硬编码、缺少复位原因检查 |

---

## compile

编译验证结果。仅 `full` 和 `hardware` 模式产出。`quick` 和 `incremental` 模式为 `null`。

```json
{
  "compile": {
    "errors": 0,
    "warnings": 2,
    "flash_used": "128KB / 1024KB (12.5%)",
    "ram_used": "48KB / 192KB (25.0%)",
    "warnings_detail": [
      {
        "file": "Core/Src/oscilloscope.c",
        "line": 45,
        "message": "unused variable 'debug_counter'"
      }
    ]
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `errors` | number | 是 | 编译错误数量 |
| `warnings` | number | 是 | 编译警告数量 |
| `flash_used` | string | 是 | Flash 使用量（格式: `"已用 / 总量 (百分比)"`） |
| `ram_used` | string | 是 | RAM 使用量（格式: `"已用 / 总量 (百分比)"`） |
| `warnings_detail` | array | 否 | 警告详情列表（仅当 warnings > 0 时存在） |

### warnings_detail[] 元素

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file` | string | 是 | 警告所在文件 |
| `line` | number | 是 | 警告所在行号 |
| `message` | string | 是 | 警告信息 |

### Flash/RAM 容量来源

从芯片型号查表获取:

| 芯片系列 | Flash | RAM |
|----------|-------|-----|
| STM32F407VG | 1024KB | 192KB |
| STM32F407VE | 512KB | 192KB |
| STM32F103C8 | 64KB | 20KB |
| STM32F103RCT6 | 256KB | 48KB |
| STM32H743VI | 2048KB | 1024KB |

---

## hardware

硬件验证结果。仅 `hardware` 模式产出。其他模式为 `null`。

```json
{
  "hardware": {
    "flash_ok": true,
    "serial_ok": true,
    "selftest_duration_ms": 1250,
    "registers": [
      {
        "peripheral": "RCC",
        "register": "AHB1ENR",
        "address": "0x40023830",
        "expected": "0x00100007",
        "actual": "0x00100007",
        "status": "match",
        "bits_decoded": "DMA2EN=1, GPIOAEN=1, GPIOBEN=1, GPIOCEN=1"
      },
      {
        "peripheral": "ADC1",
        "register": "CR2",
        "address": "0x40012008",
        "expected": "0x00000001",
        "actual": "0x00000000",
        "status": "mismatch",
        "bits_decoded": "ADON=0 (expected 1)"
      }
    ]
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `flash_ok` | boolean | 是 | 固件烧录是否成功 |
| `serial_ok` | boolean | 是 | 串口通信是否正常 |
| `selftest_duration_ms` | number | 否 | 自检耗时（毫秒） |
| `registers` | array | 是 | 寄存器比对结果列表 |

### registers[] 元素

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `peripheral` | string | 是 | 外设名称 |
| `register` | string | 是 | 寄存器名称 |
| `address` | string | 是 | 寄存器地址（十六进制） |
| `expected` | string | 是 | 期望值（十六进制） |
| `actual` | string | 是 | 实际读取值（十六进制） |
| `status` | string | 是 | 比对状态: `match` / `mismatch` / `not_readable` |
| `bits_decoded` | string | 否 | 关键位的解码说明 |

---

## recommendations

优先修复建议列表。根据失败项的严重度和影响范围排序。

```json
{
  "recommendations": [
    {
      "priority": 1,
      "check_ids": ["STM-011", "GEN-003"],
      "summary": "修复 2 个 critical 级别问题：DMA 启动顺序和共享变量 volatile 标记",
      "files_affected": ["Core/Src/oscilloscope.c"],
      "estimated_effort": "15min"
    },
    {
      "priority": 2,
      "check_ids": ["ARM-002", "STM-015"],
      "summary": "修复 2 个 high 级别问题：NVIC 优先级配置和 TIM 时钟使能",
      "files_affected": ["Core/Src/stm32f4xx_it.c", "Core/Src/tim.c"],
      "estimated_effort": "10min"
    }
  ]
}
```

### recommendations[] 元素

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `priority` | number | 是 | 修复优先级（1 = 最高） |
| `check_ids` | string[] | 是 | 关联的检查项 ID 列表 |
| `summary` | string | 是 | 修复建议摘要 |
| `files_affected` | string[] | 是 | 需要修改的文件列表 |
| `estimated_effort` | string | 否 | 预估工作量（如 "15min"、"1h"） |

### 排序规则

1. critical fail 项排在最前
2. 同一严重度下，按影响文件数量降序
3. 涉及相同文件的 fail 项合并为一条建议

---

## 增量模式特有字段

当 `scan_mode` 为 `incremental` 时，部分字段有额外扩展:

### meta 扩展

无额外字段。

### code_graph 扩展

```json
{
  "code_graph": {
    "files_scanned": 3,
    "symbols_found": 28,
    "changed_files": [
      "Core/Src/oscilloscope.c",
      "Core/Src/stm32f4xx_it.c",
      "Core/Inc/oscilloscope.h"
    ],
    "diff_summary": {
      "insertions": 42,
      "deletions": 18
    },
    "peripherals": [ ... ],
    "isrs": [ ... ],
    "tasks": [ ... ],
    "shared_vars": [ ... ]
  }
}
```

| 增量字段 | 类型 | 说明 |
|---------|------|------|
| `changed_files` | string[] | git diff 涉及的文件列表 |
| `diff_summary.insertions` | number | 总插入行数 |
| `diff_summary.deletions` | number | 总删除行数 |

### checks[] 扩展

```json
{
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
      "diff_context": "@@ -180,6 +180,10 @@\n   /* USER CODE BEGIN */\n+  HAL_TIM_Base_Start(&htim8);\n+  HAL_ADC_Start_DMA(&hadc1, adc_buffer, BUFFER_SIZE);\n   /* USER CODE END */",
      "evidence": "TIM8 在 ADC DMA 之前启动",
      "fix": "交换调用顺序",
      "reference": "RM0090 Section 13.3.15"
    }
  ]
}
```

| 增量字段 | 类型 | 说明 |
|---------|------|------|
| `diff_context` | string | 相关代码的 git diff 片段（unified 格式） |

---

## 人类可读摘要格式

除 JSON 外，验证器同时输出人类可读的文本摘要。

### quick 模式

```
[stm32-verifier] Quick Scan Complete
  Project: scope-siggen  |  Platform: STM32F407VGTx  |  Rules: 62
  PASS: 54  |  FAIL: 3  |  WARN: 4  |  SKIP: 1
  Critical failures: 1  (STM-011: DMA clock not enabled before TIM start)
  Run "full verify" for compile verification.
```

### full 模式

```
[stm32-verifier] Full Scan Complete
  Project: scope-siggen  |  Platform: STM32F407VGTx  |  Rules: 62
  PASS: 54  |  FAIL: 3  |  WARN: 4  |  SKIP: 1
  Critical failures: 1  (STM-011: DMA clock not enabled before TIM start)
  Compile: 0 errors, 2 warnings
  Flash: 128KB / 1024KB (12.5%)  |  RAM: 48KB / 192KB (25.0%)
```

### hardware 模式

```
[stm32-verifier] Hardware Scan Complete
  Project: scope-siggen  |  Platform: STM32F407VGTx  |  Rules: 62
  PASS: 54  |  FAIL: 3  |  WARN: 4  |  SKIP: 1
  Critical failures: 1  (STM-011: DMA clock not enabled before TIM start)
  Compile: 0 errors, 2 warnings
  Flash: 128KB / 1024KB (12.5%)  |  RAM: 48KB / 192KB (25.0%)
  Hardware: flash OK, serial OK, registers 5/6 match (1 mismatch: ADC1.CR2)
```

### incremental 模式

```
[stm32-verifier] Incremental Scan Complete
  Project: scope-siggen  |  Changed files: 3  |  Rules checked: 8/62
  PASS: 6  |  FAIL: 1  |  WARN: 1  |  SKIP: 0
  Critical failures: 1  (STM-011: DMA clock not enabled before TIM start)
  Run "full verify" for complete validation.
```

---

## JSON Schema (JSON Schema Draft 2020-12)

以下为机器可读的 JSON Schema 定义，可用于验证输出格式。

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "STM32 Verifier Output",
  "type": "object",
  "required": ["meta", "code_graph", "summary", "checks", "recommendations"],
  "properties": {
    "meta": {
      "type": "object",
      "required": ["tool", "version", "timestamp", "project", "platform", "scan_mode", "rules_applied"],
      "properties": {
        "tool": { "const": "stm32-verifier" },
        "version": { "type": "string", "pattern": "^\\d+\\.\\d+\\.\\d+$" },
        "timestamp": { "type": "string", "format": "date-time" },
        "project": { "type": "string" },
        "platform": { "type": "string" },
        "scan_mode": { "enum": ["quick", "full", "hardware", "incremental"] },
        "rules_applied": {
          "type": "array",
          "items": { "enum": ["layer1", "layer2", "layer3"] }
        }
      }
    },
    "code_graph": {
      "type": "object",
      "required": ["files_scanned", "symbols_found", "peripherals", "isrs", "tasks", "shared_vars"],
      "properties": {
        "files_scanned": { "type": "number" },
        "symbols_found": { "type": "number" },
        "peripherals": { "type": "array" },
        "isrs": { "type": "array" },
        "tasks": { "type": "array" },
        "shared_vars": { "type": "array" }
      }
    },
    "summary": {
      "type": "object",
      "required": ["total", "pass", "fail", "warn", "skip", "by_severity"],
      "properties": {
        "total": { "type": "number" },
        "pass": { "type": "number" },
        "fail": { "type": "number" },
        "warn": { "type": "number" },
        "skip": { "type": "number" },
        "by_severity": { "type": "object" }
      }
    },
    "checks": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "dimension", "layer", "title", "severity", "status", "file", "line", "evidence"],
        "properties": {
          "id": { "type": "string", "pattern": "^(GEN|ARM|STM)-\\d{3}$" },
          "dimension": { "type": "string" },
          "layer": { "enum": ["layer1", "layer2", "layer3"] },
          "title": { "type": "string" },
          "severity": { "enum": ["critical", "high", "medium", "low"] },
          "status": { "enum": ["pass", "fail", "warn", "skip"] },
          "file": { "type": "string" },
          "line": { "type": "number" },
          "evidence": { "type": "string" },
          "fix": { "type": "string" },
          "reference": { "type": "string" }
        }
      }
    },
    "compile": {
      "oneOf": [
        { "type": "null" },
        {
          "type": "object",
          "required": ["errors", "warnings", "flash_used", "ram_used"],
          "properties": {
            "errors": { "type": "number" },
            "warnings": { "type": "number" },
            "flash_used": { "type": "string" },
            "ram_used": { "type": "string" }
          }
        }
      ]
    },
    "hardware": {
      "oneOf": [
        { "type": "null" },
        {
          "type": "object",
          "required": ["flash_ok", "serial_ok", "registers"],
          "properties": {
            "flash_ok": { "type": "boolean" },
            "serial_ok": { "type": "boolean" },
            "registers": { "type": "array" }
          }
        }
      ]
    },
    "recommendations": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["priority", "check_ids", "summary", "files_affected"],
        "properties": {
          "priority": { "type": "number" },
          "check_ids": { "type": "array", "items": { "type": "string" } },
          "summary": { "type": "string" },
          "files_affected": { "type": "array", "items": { "type": "string" } },
          "estimated_effort": { "type": "string" }
        }
      }
    }
  }
}
```
