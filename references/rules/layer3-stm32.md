# Layer 3: STM32 专有检查规则

> 共 30 项规则，覆盖 ADC/DAC/DMA/Timer/GPIO/UART/I2C/SPI/Flash/FreeRTOS 等 STM32 HAL 常见配置问题。

---

### STM-001: ADC 时钟使能

**严重度**: Critical
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `HAL_ADC_Init` 或 `HAL_ADC_Start` / `HAL_ADC_Start_DMA` 调用
- 确认在调用之前存在对应的时钟使能：`__HAL_RCC_ADC1_CLK_ENABLE()` 或 `__HAL_RCC_ADC2_CLK_ENABLE()` 等
- 检查 `main.c` 的 `MX_*` 初始化函数或 `SystemClock_Config` 之后是否有 ADC 时钟使能

**通过标准**:
- 每个使用的 ADC 实例都有对应的 `__HAL_RCC_ADCx_CLK_ENABLE()` 调用
- 时钟使能在 ADC 初始化之前执行

**修复建议**:
- 在 `main()` 中 `HAL_Init()` 和 `SystemClock_Config()` 之后、`MX_ADCx_Init()` 之前添加 `__HAL_RCC_ADCx_CLK_ENABLE()`
- 如果使用多个 ADC，确保每个都单独使能

---

### STM-002: ADC GPIO 配置为模拟模式

**严重度**: Critical
**适用平台**: STM32 系列

**检查方法**:
- 搜索 ADC 通道对应的 GPIO 引脚配置（如 `GPIO_PIN_0`, `GPIO_PIN_1` 等）
- 检查 `GPIO_InitStruct.Mode` 是否为 `GPIO_MODE_ANALOG`
- 确认 GPIO 端口时钟已使能（`__HAL_RCC_GPIOx_CLK_ENABLE()`）

**通过标准**:
- 所有 ADC 输入引脚的 GPIO 模式设置为 `GPIO_MODE_ANALOG`
- 对应 GPIO 端口时钟已使能

**修复建议**:
```c
GPIO_InitStruct.Pin = GPIO_PIN_0;
GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;  // 必须是 ANALOG
GPIO_InitStruct.Pull = GPIO_NOPULL;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
```

---

### STM-003: DMA 时钟使能

**严重度**: Critical
**适用平台**: STM32 系列

**检查方法**:
- 搜索 DMA 外设使用（`HAL_DMA_Init`, `__HAL_LINKDMA`, DMA Stream 配置）
- 确认存在 `__HAL_RCC_DMA1_CLK_ENABLE()` 或 `__HAL_RCC_DMA2_CLK_ENABLE()`
- 检查 DMA 时钟使能是否在 DMA 初始化之前

**通过标准**:
- 使用 DMA1 时有 `__HAL_RCC_DMA1_CLK_ENABLE()`
- 使用 DMA2 时有 `__HAL_RCC_DMA2_CLK_ENABLE()`
- 时钟使能在 `HAL_DMA_Init()` 之前

**修复建议**:
- 在 `main()` 的外设初始化区域添加对应的 DMA 时钟使能
- STM32F1/F4: `__HAL_RCC_DMA1_CLK_ENABLE()` / `__HAL_RCC_DMA2_CLK_ENABLE()`
- STM32F0/G0/L0: 只有 DMA1，使用 `__HAL_RCC_DMA1_CLK_ENABLE()`

---

### STM-004: DMA 流/通道配置

**严重度**: Critical
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `DMA_InitTypeDef` 或 `HAL_DMA_Init` 调用
- 检查关键字段：`Channel`, `Direction`, `PeriphInc`, `MemInc`, `PeriphDataAlignment`, `MemDataAlignment`, `Mode`
- 验证 `Direction` 是否匹配使用场景（`DMA_PERIPH_TO_MEMORY` 用于 ADC 采集，`DMA_MEMORY_TO_PERIPH` 用于 DAC 输出）
- 验证 `Mode` 是否为 `DMA_CIRCULAR`（连续采集场景）或 `DMA_NORMAL`

**通过标准**:
- Direction 与外设类型匹配（ADC→PERIPH_TO_MEMORY，DAC/UART_TX→MEMORY_TO_PERIPH）
- DataAlignment 与 ADC/DAC 分辨率匹配（12-bit 用 HALFWORD）
- 循环采集模式使用 `DMA_CIRCULAR`

**修复建议**:
- ADC DMA 采集典型配置：
```c
hdma_adc.Init.Direction = DMA_PERIPH_TO_MEMORY;
hdma_adc.Init.PeriphInc = DMA_PINC_DISABLE;
hdma_adc.Init.MemInc = DMA_MINC_ENABLE;
hdma_adc.Init.PeriphDataAlignment = DMA_PDATAALIGN_HALFWORD;
hdma_adc.Init.MemDataAlignment = DMA_MDATAALIGN_HALFWORD;
hdma_adc.Init.Mode = DMA_CIRCULAR;
```

---

### STM-005: __HAL_LINKDMA 链接 DMA 到外设

**严重度**: Critical
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `__HAL_LINKDMA` 调用
- 确认 DMA handle 被正确链接到外设 handle 的 `DMA_Handle` 成员
- 检查链接是否在外设初始化之前或之后正确放置

**通过标准**:
- 每个使用 DMA 的外设都有对应的 `__HAL_LINKDMA` 调用
- 链接在外设 `HAL_xxx_Init()` 之前完成

**修复建议**:
```c
// ADC 示例
__HAL_LINKDMA(&hadc1, DMA_Handle, hdma_adc1);

// DAC 示例
__HAL_LINKDMA(&hdac, DMA_Handle1, hdma_dac1);

// UART 示例
__HAL_LINKDMA(&huart2, hdmatx, hdma_usart2_tx);
```

---

### STM-006: DMA NVIC 使能

**严重度**: High
**适用平台**: STM32 系列

**检查方法**:
- 搜索 DMA 中断相关代码：`HAL_NVIC_SetPriority` + `HAL_NVIC_EnableIRQ` 配对
- 检查 DMA Stream/Channel 对应的 IRQn 是否正确（如 `DMA1_Stream0_IRQn`, `DMA1_Channel1_IRQn`）
- 如果使用 DMA 中断回调（`HAL_ADC_ConvCpltCallback` 等），确认 NVIC 已使能

**通过标准**:
- 使用 DMA 中断时，对应 IRQn 已设置优先级并使能
- 优先级与 FreeRTOS（如使用）的 `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY` 兼容

**修复建议**:
```c
HAL_NVIC_SetPriority(DMA1_Stream0_IRQn, 0, 0);
HAL_NVIC_EnableIRQ(DMA1_Stream0_IRQn);
```
- 如果仅使用 DMA 轮询模式（`HAL_ADC_PollForConversion`），可不使能 NVIC

---

### STM-007: TIM TRGO 配置为 UPDATE 事件

**严重度**: Critical
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `MasterConfig.MasterOutputTrigger` 或 `TIM_TRGO_*` 相关配置
- 确认触发源为 `TIM_TRGO_UPDATE` 而非 `TIM_TRGO_RESET`
- 检查 ADC/DAC 的外部触发源（`ExternalTrigConv`）是否与 TIM TRGO 匹配

**通过标准**:
- `MasterOutputTrigger` 设置为 `TIM_TRGO_UPDATE`
- ADC 的 `ExternalTrigConv` 与定时器 TRGO 事件对应

**修复建议**:
```c
// Timer 主模式配置
TIM_MasterConfigTypeDef sMasterConfig = {0};
sMasterConfig.MasterOutputTrigger = TIM_TRGO_UPDATE;  // 不是 TIM_TRGO_RESET
sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;
HAL_TIMEx_MasterConfigSynchronization(&htimx, &sMasterConfig);
```

---

### STM-008: DMAContinuousRequests 必须使能

**严重度**: Critical
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `ADC_InitTypeDef` 或 `hadc.Init.*` 配置
- 检查 `DMAContinuousRequests` 字段是否为 `ENABLE`
- 如果为 `DISABLE`，DMA 在每次转换后停止，无法连续采集

**通过标准**:
- `hadc.Init.DMAContinuousRequests = ENABLE`

**修复建议**:
```c
hadc.Init.DMAContinuousRequests = ENABLE;  // 必须 ENABLE 以支持连续 DMA 采集
```
- 如果设为 `DISABLE`，每次 DMA 传输完成后需要手动重新启动

---

### STM-009: EOCSelection 配置

**严重度**: High
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `EOCSelection` 配置字段
- 检查是否设置为 `ADC_EOC_SINGLE_CONV` 或 `ADC_EOC_SEQ_CONV`
- 单通道连续采集应使用 `ADC_EOC_SINGLE_CONV`

**通过标准**:
- 单通道场景：`ADC_EOC_SINGLE_CONV`
- 多通道扫描场景：`ADC_EOC_SEQ_CONV`
- 不使用默认值（可能导致中断/DMA 标志行为异常）

**修复建议**:
```c
hadc.Init.EOCSelection = ADC_EOC_SINGLE_CONV;  // 单通道连续采集
```

---

### STM-010: ADC 采样时间配置

**严重度**: Medium
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `HAL_ADC_ConfigChannel` 中的 `SamplingTime` 设置
- 检查高阻信号源（如电位器、传感器输出）是否使用了足够长的采样时间
- 高阻源应使用 `ADC_SAMPLETIME_480CYCLES`（F4）或等效的长采样时间

**通过标准**:
- 高阻源（>10kΩ）使用 480 cycles 或更长的采样时间
- 低阻源可使用较短采样时间（如 84 cycles）
- 采样时间不会导致 ADC 转换速率低于应用需求

**修复建议**:
```c
sConfig.SamplingTime = ADC_SAMPLETIME_480CYCLES;  // 高阻源推荐
```
- 如果 ADC 读数不稳定或偏移，优先检查采样时间是否足够

---

### STM-011: ADC/DAC/DMA 启动顺序

**严重度**: Critical
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `HAL_ADC_Start_DMA`, `HAL_DAC_Start`, `HAL_TIM_Base_Start` 调用
- 确认启动顺序：先使能 ADC/DAC 和 DMA，最后启动定时器
- 检查是否有定时器先于 ADC/DMA 启动的情况

**通过标准**:
- 启动顺序为：`HAL_ADC_Start_DMA()` → `HAL_TIM_Base_Start()`（最后启动定时器触发）
- DAC 场景：`HAL_DAC_Start()` → `HAL_DAC_Start_DMA()` → `HAL_TIM_Base_Start()`

**修复建议**:
```c
// 正确顺序
HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adc_buffer, BUFFER_SIZE);  // 1. ADC+DMA
HAL_TIM_Base_Start(&htim2);                                       // 2. 定时器（最后启动）
```

---

### STM-012: DAC 时钟使能

**严重度**: Critical
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `HAL_DAC_Init` 或 `HAL_DAC_Start` 调用
- 确认存在 `__HAL_RCC_DAC_CLK_ENABLE()`
- 检查 DAC 输出引脚的 GPIO 配置为 `GPIO_MODE_ANALOG`

**通过标准**:
- `__HAL_RCC_DAC_CLK_ENABLE()` 在 `HAL_DAC_Init()` 之前调用
- DAC 输出引脚（PA4/PA5）配置为模拟模式

**修复建议**:
```c
__HAL_RCC_DAC_CLK_ENABLE();  // 在 HAL_DAC_Init 之前
```

---

### STM-013: DAC DMA 链接

**严重度**: Critical
**适用平台**: STM32 系列

**检查方法**:
- 搜索 DAC 使用 DMA 的场景（`HAL_DAC_Start_DMA`）
- 确认存在 `__HAL_LINKDMA(&hdac, DMA_Handle1, hdma_dac)` 调用
- 检查 DMA 通道/流是否与 DAC 匹配（参考芯片参考手册）

**通过标准**:
- DAC DMA handle 通过 `__HAL_LINKDMA` 正确链接
- DMA 通道映射到 DAC（如 STM32F4: DMA1 Stream 5 Channel 7）

**修复建议**:
```c
__HAL_LINKDMA(&hdac, DMA_Handle1, hdma_dac_ch1);  // DAC Channel1
__HAL_LINKDMA(&hdac, DMA_Handle2, hdma_dac_ch2);  // DAC Channel2（如使用）
```

---

### STM-014: DAC 触发源配置

**严重度**: High
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `HAL_DAC_ConfigChannel` 中的 `DAC_TRIGGER_*` 配置
- 确认触发源与实际使用的定时器匹配
- 检查 `ExternalTrigConvEdge` 是否正确（通常为 `DAC_TRIGGEREDGE_RISING`）

**通过标准**:
- 触发源与定时器 TRGO 事件匹配
- 使用软件触发时配置为 `DAC_TRIGGER_SOFTWARE`
- 使用硬件触发时配置为 `DAC_TRIGGER_Tx_TRGO`（x 为定时器编号）

**修复建议**:
```c
sConfig.DAC_Trigger = DAC_TRIGGER_T6_TRGO;  // TIM6 TRGO 触发
sConfig.DAC_OutputBuffer = DAC_OUTPUTBUFFER_DISABLE;
HAL_DAC_ConfigChannel(&hdac, &sConfig, DAC_CHANNEL_1);
```

---

### STM-015: TIM5 启动（DAC 触发定时器）

**严重度**: Critical
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `HAL_TIM_Base_Start` 或 `HAL_TIM_Base_Start_IT` 调用
- 如果 DAC 使用 TIM5 作为触发源，确认 TIM5 已启动
- 检查定时器启动是否在 DAC 启动之后（正确顺序：DAC 先，TIM 后）

**通过标准**:
- DAC 触发定时器在 DAC 配置完成后启动
- 启动函数调用成功（返回 `HAL_OK`）

**修复建议**:
```c
// 启动顺序
HAL_DAC_Start(&hdac, DAC_CHANNEL_1);
HAL_TIM_Base_Start(&htim5);  // DAC 已就绪后再启动触发源
```

---

### STM-016: HAL Start/Stop 函数对称

**严重度**: High
**适用平台**: STM32 系列

**检查方法**:
- 搜索所有 `HAL_xxx_Start` 调用（含 `_IT`, `_DMA` 变体）
- 检查是否有对应的 `HAL_xxx_Stop` 调用
- 特别关注运行时可重配置的场景（如改变采样率、切换波形）

**通过标准**:
- 每个 `Start` 在不再需要时有对应的 `Stop`
- 不会对已启动的外设重复调用 `Start`（可能导致状态混乱）
- 修改配置前先 `Stop`，配置完成后再 `Start`

**修复建议**:
```c
// 修改 ADC 采样率前
HAL_ADC_Stop_DMA(&hadc1);
// 修改定时器周期
htim2.Init.Period = new_period;
HAL_TIM_Base_Init(&htim2);
// 重新启动
HAL_ADC_Start_DMA(&hadc1, buffer, size);
HAL_TIM_Base_Start(&htim2);
```

---

### STM-017: 定时器修改前先停止

**严重度**: High
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `HAL_TIM_Base_Init` 调用（用于修改定时器参数）
- 检查在调用 `HAL_TIM_Base_Init` 修改 Period/Prescaler 之前是否先调用了 `HAL_TIM_Base_Stop`
- 对比直接修改 `htim.Init.Period` 而不停止定时器的代码

**通过标准**:
- 修改定时器参数前先 `HAL_TIM_Base_Stop()`
- 修改完成后重新 `HAL_TIM_Base_Start()`

**修复建议**:
```c
HAL_TIM_Base_Stop(&htim2);           // 1. 停止
htim2.Init.Period = new_period;       // 2. 修改
HAL_TIM_Base_Init(&htim2);           // 3. 重新初始化
HAL_TIM_Base_Start(&htim2);          // 4. 重启
```

---

### STM-018: EGR 寄存器触发更新事件

**严重度**: Medium
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `__HAL_TIM_SET_PRESCALER` 或直接修改 `htim.Instance->PSC` / `htim.Instance->ARR` 的代码
- 检查修改后是否触发了更新事件使影子寄存器生效
- 查找 `htim.Instance->EGR = TIM_EGR_UG` 或等效操作

**通过标准**:
- 直接修改 PSC/ARR 后，通过 `EGR = TIM_EGR_UG` 触发更新事件
- 或者使用 HAL 库函数（`HAL_TIM_Base_Init`）自动处理

**修复建议**:
```c
htim2.Instance->ARR = new_period;
htim2.Instance->EGR = TIM_EGR_UG;  // 触发更新事件，影子寄存器立即生效
```

---

### STM-019: GPIO 模式与外设匹配

**严重度**: High
**适用平台**: STM32 系列

**检查方法**:
- 搜索所有 `HAL_GPIO_Init` 调用
- 检查 GPIO 模式是否与连接的外设匹配：
  - ADC 输入: `GPIO_MODE_ANALOG`
  - UART TX/RX: `GPIO_MODE_AF_PP`
  - SPI SCK/MOSI/MISO: `GPIO_MODE_AF_PP`
  - I2C SDA/SCL: `GPIO_MODE_AF_OD`
  - 按键输入: `GPIO_MODE_INPUT` + `GPIO_PULLUP/PULLDOWN`

**通过标准**:
- ADC 引脚 → `GPIO_MODE_ANALOG`
- UART 引脚 → `GPIO_MODE_AF_PP`
- I2C 引脚 → `GPIO_MODE_AF_OD`（开漏）
- 无遗漏或错误模式配置

**修复建议**:
- 参考芯片数据手册确认每个引脚的外设功能
- CubeMX 生成的配置通常正确，手动修改时需格外注意

---

### STM-020: UART 复用功能配置

**严重度**: Medium
**适用平台**: STM32 系列

**检查方法**:
- 搜索 UART 引脚的 GPIO 配置
- 检查 `GPIO_InitStruct.Alternate` 是否设置了正确的 AF 编号
- 验证 AF 编号与芯片参考手册一致（如 USART1 可能是 `GPIO_AF7_USART1`）

**通过标准**:
- `Alternate` 字段设置为正确的 `GPIO_AFx_USARTy` 值
- TX 引脚和 RX 引脚使用相同的 AF 编号

**修复建议**:
```c
GPIO_InitStruct.Pin = GPIO_PIN_9 | GPIO_PIN_10;
GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
GPIO_InitStruct.Pull = GPIO_PULLUP;
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
GPIO_InitStruct.Alternate = GPIO_AF7_USART1;  // 参考手册确认
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
```

---

### STM-021: I2C 时钟速度配置

**严重度**: Medium
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `I2C_InitTypeDef` 或 `hi2c.Init.ClockSpeed` 配置
- 检查时钟速度是否在合理范围内（标准模式 100kHz，快速模式 400kHz）
- 验证 I2C 时钟源频率是否足够（通常需要 >= 4x I2C 速度）

**通过标准**:
- `ClockSpeed` 在 100000（100kHz）到 400000（400kHz）之间
- I2C 外设时钟源（APB1）频率足够

**修复建议**:
```c
hi2c.Init.ClockSpeed = 100000;  // 标准模式
// 或
hi2c.Init.ClockSpeed = 400000;  // 快速模式
```

---

### STM-022: SPI 时钟极性和相位

**严重度**: Medium
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `SPI_InitTypeDef` 配置
- 检查 `CLKPolarity` 和 `CLKPhase` 是否与从设备要求匹配
- 常见模式：Mode 0 (CPOL=0, CPHA=0), Mode 3 (CPOL=1, CPHA=1)

**通过标准**:
- 时钟极性和相位与目标设备数据手册一致
- 数据位数（`DataSize`）与协议匹配（通常 8-bit 或 16-bit）

**修复建议**:
```c
hspi.Init.CLKPolarity = SPI_POLARITY_LOW;   // CPOL = 0
hspi.Init.CLKPhase = SPI_PHASE_1EDGE;       // CPHA = 0 (Mode 0)
// 或
hspi.Init.CLKPolarity = SPI_POLARITY_HIGH;  // CPOL = 1
hspi.Init.CLKPhase = SPI_PHASE_2EDGE;       // CPHA = 1 (Mode 3)
```

---

### STM-023: PWM 死区时间配置

**严重度**: Medium
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `HAL_TIMEx_ConfigBreakDeadTime` 调用
- 检查 `DeadTime` 值是否合理（太小可能导致上下桥臂直通，太大影响效率）
- 验证 `OffStateRunMode` 和 `OffStateIDLEMode` 配置

**通过标准**:
- 死区时间根据功率器件开关特性合理设置
- `BreakState` 根据安全需求配置
- 互补输出通道的死区时间已正确配置

**修复建议**:
```c
sBreakDeadTimeConfig.OffStateRunMode = TIM_OSSR_DISABLE;
sBreakDeadTimeConfig.OffStateIDLEMode = TIM_OSSI_DISABLE;
sBreakDeadTimeConfig.LockLevel = TIM_LOCKLEVEL_OFF;
sBreakDeadTimeConfig.DeadTime = 100;  // 根据功率管特性调整
sBreakDeadTimeConfig.BreakState = TIM_BREAK_DISABLE;
HAL_TIMEx_ConfigBreakDeadTime(&htim1, &sBreakDeadTimeConfig);
```

---

### STM-024: 编码器模式配置

**严重度**: Low
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `HAL_TIM_Encoder_Start` 或 `TIM_ENCODERMODE_*` 配置
- 检查编码器模式是否正确（`TI1`, `TI2`, `TI12`）
- 验证输入捕获通道的极性配置

**通过标准**:
- 编码器模式与实际编码器信号匹配
- 输入滤波器设置适当（抗抖动）
- 计数方向与机械方向一致

**修复建议**:
```c
sConfig.EncoderMode = TIM_ENCODERMODE_TI12;  // 使用 TI1 和 TI2
sConfig.IC1Polarity = TIM_ICPOLARITY_RISING;
sConfig.IC2Polarity = TIM_ICPOLARITY_RISING;
HAL_TIM_Encoder_Init(&htim3, &sConfig);
```

---

### STM-025: ADC 校准

**严重度**: Low
**适用平台**: STM32 系列（F0, F3, G0, L0, L4 等）

**检查方法**:
- 搜索 `HAL_ADCEx_Calibration_Start` 调用
- 确认校准在 ADC 初始化之后、启动转换之前执行
- F1/F4 系列无需软件校准（硬件自动校准）

**通过标准**:
- 支持软件校准的系列（F0/F3/G0/L0/L4）在启动前调用了校准函数
- 校准返回值为 `HAL_OK`

**修复建议**:
```c
HAL_ADCEx_Calibration_Start(&hadc1);  // ADC 初始化之后，Start 之前
HAL_ADC_Start_DMA(&hadc1, buffer, size);
```

---

### STM-026: Flash 解锁/锁定

**严重度**: High
**适用平台**: STM32 系列

**检查方法**:
- 搜索 `HAL_FLASH_Unlock` 和 `HAL_FLASH_Lock` 调用
- 确认每次 Flash 写入/擦除操作前后都有 Unlock/Lock 配对
- 检查是否有异常路径导致 Lock 被跳过

**通过标准**:
- `HAL_FLASH_Unlock()` 在写入/擦除之前
- `HAL_FLASH_Lock()` 在操作完成之后（包括错误路径）
- Unlock/Lock 严格配对

**修复建议**:
```c
HAL_FLASH_Unlock();
// ... Flash 操作 ...
HAL_FLASH_Lock();  // 确保所有路径都执行 Lock
```
- 建议使用 `__try/__finally` 或 goto 统一出口确保 Lock

---

### STM-027: Flash 扇区擦除

**严重度**: Medium
**适用平台**: STM32 系列（F4 等）

**检查方法**:
- 搜索 `HAL_FLASHEx_Erase` 或 `FLASH_EraseSector` 调用
- 检查擦除的扇区编号是否正确（不要擦除代码区）
- 验证电压范围（`VoltageRange`）与实际 VDD 匹配

**通过标准**:
- 扇区编号在安全范围内（不擦除程序代码所在扇区）
- `VoltageRange` 正确设置（通常 `FLASH_VOLTAGE_RANGE_3` 对应 2.7-3.6V）
- 擦除前检查目标地址是否为空（可选优化）

**修复建议**:
```c
FLASH_EraseInitTypeDef erase;
erase.TypeErase = FLASH_TYPEERASE_SECTORS;
erase.Sector = FLASH_SECTOR_7;      // 确认是数据存储区
erase.NbSectors = 1;
erase.VoltageRange = FLASH_VOLTAGE_RANGE_3;
uint32_t error;
HAL_FLASHEx_Erase(&erase, &error);
```

---

### STM-028: FreeRTOSConfig.h 关键配置

**严重度**: Medium
**适用平台**: STM32 + FreeRTOS

**检查方法**:
- 搜索 `FreeRTOSConfig.h` 文件
- 检查关键配置项：
  - `configCPU_CLOCK_HZ` 是否与实际系统时钟匹配
  - `configTICK_RATE_HZ` 是否合理（通常 1000）
  - `configMAX_PRIORITIES` 是否足够
  - `configTOTAL_HEAP_SIZE` 是否在 RAM 范围内
  - `configMAX_SYSCALL_INTERRUPT_PRIORITY` 是否正确

**通过标准**:
- `configCPU_CLOCK_HZ` 与 `SystemClock_Config` 中配置的频率一致
- `configTOTAL_HEAP_SIZE` 不超过芯片 RAM 大小（留出静态变量空间）
- `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY` 正确设置（影响中断安全 API）

**修复建议**:
```c
#define configCPU_CLOCK_HZ    ((uint32_t)168000000)  // 匹配 SystemClock
#define configTOTAL_HEAP_SIZE ((size_t)(32 * 1024))   // 根据 RAM 调整
```

---

### STM-029: 任务栈大小

**严重度**: Medium
**适用平台**: STM32 + FreeRTOS

**检查方法**:
- 搜索 `osThreadNew` 或 `xTaskCreate` 调用
- 检查栈大小参数（单位：word，不是 byte）
- 评估每个任务的栈使用：局部变量、函数调用深度、中断嵌套

**通过标准**:
- 栈大小足够容纳最深调用路径 + 局部变量 + 中断上下文
- 使用 `uxTaskGetStackHighWaterMark` 验证实际使用量（调试阶段）
- 无栈溢出风险（建议保留 20% 以上余量）

**修复建议**:
- 最小栈大小 128 words（512 bytes）适用于简单任务
- 涉及 printf、数学运算的任务建议 256+ words
- 使用 `configCHECK_FOR_STACK_OVERFLOW` = 2 进行运行时检测

---

### STM-030: RTOS 对象创建时机

**严重度**: Critical
**适用平台**: STM32 + FreeRTOS / CMSIS-RTOS

**检查方法**:
- 搜索 `osThreadNew`, `osTimerNew`, `osMutexNew`, `osSemaphoreNew` 等 RTOS 对象创建调用
- 确认这些调用在 `osKernelStart()` 之前（在 `main()` 的初始化阶段）
- 检查是否有在中断或任务运行时动态创建 RTOS 对象的情况

**通过标准**:
- 所有 RTOS 对象在 `osKernelStart()` 之前创建
- 中断中不创建 RTOS 对象
- 如果需要延迟创建，在任务入口函数的开头创建（而非运行过程中反复创建）

**修复建议**:
```c
int main(void) {
    HAL_Init();
    SystemClock_Config();
    // 外设初始化...

    // RTOS 对象创建（osKernelStart 之前）
    tid_thread1 = osThreadNew(Thread1, NULL, &attr_thread1);
    osTimerId = osTimerNew(TimerCallback, osTimerPeriodic, NULL, NULL);

    osKernelStart();  // 最后启动调度器
    while (1) {}      // 不应到达这里
}
```
