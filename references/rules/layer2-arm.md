# Layer 2: ARM Cortex-M 架构检查规则

## ARM-001: NVIC 优先级配置

**严重度**: High
**适用平台**: Cortex-M 系列

**检查方法**:
- 搜索 `HAL_NVIC_SetPriority`、`NVIC_SetPriority`、`HAL_NVIC_EnableIRQ` 调用
- 检查抢占优先级 (Preemption Priority) 和子优先级 (Sub Priority) 是否合理分配
- 确认 `NVIC_PRIORITYGROUP_x` 配置与实际优先级分配一致
- 检查是否存在优先级冲突（多个中断使用相同优先级）

**通过标准**:
- 所有中断的优先级分配符合项目设计文档
- 优先级分组模式与代码中使用的优先级值匹配
- 无优先级冲突，关键中断（如电机控制、通信）优先级高于非关键中断

**修复建议**:
- 统一使用 `HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4)` 实现 16 级抢占优先级
- 为关键外设中断分配较低的优先级数值（更高优先级）
- 在系统初始化阶段集中配置 NVIC，避免分散在各模块中

---

## ARM-002: 中断优先级 <= configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY

**严重度**: Critical
**适用平台**: Cortex-M 系列

**检查方法**:
- 搜索 FreeRTOS 配置中 `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY` 的定义值
- 搜索所有 `HAL_NVIC_SetPriority` 调用，提取优先级数值
- 检查使用 FreeRTOS `FromISR` API 的中断，其优先级是否 <= `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY`
- 确认不会在高于此阈值的中断中调用 FreeRTOS API

**通过标准**:
- 所有调用 `FromISR` API 的中断，其优先级数值 <= `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY`
- 不调用 FreeRTOS API 的中断可以使用任意优先级

**修复建议**:
- 将使用 FreeRTOS API 的中断优先级设置为 `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY` 或更低（数值更大）
- 典型配置：`configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY = 5`，则使用 API 的中断优先级应 >= 5
- 在 `FreeRTOSConfig.h` 中明确文档化优先级分配策略

---

## ARM-003: __DMB() 内存屏障

**严重度**: Critical
**适用平台**: Cortex-M 系列

**检查方法**:
- 搜索 ISR 中对全局变量的写操作（`volatile` 变量赋值）
- 检查对应的读取方（任务或其他上下文）是否在读取前使用 `__DMB()` 或 `__DSB()` 内存屏障
- 特别关注 ISR 写多个相关变量的场景（如标志位 + 数据）
- 检查 `portENTER_CRITICAL()` / `portEXIT_CRITICAL()` 是否已包含屏障

**通过标准**:
- ISR 写多个相关变量时，在写入完成后使用 `__DMB()` 确保写入顺序
- 读取方在读取前使用 `__DMB()` 确保读取到最新值
- 或使用 `portENTER_CRITICAL()` / `portEXIT_CRITICAL()` 替代手动屏障

**修复建议**:
```c
// ISR 中写多变量
volatile uint32_t data;
volatile uint8_t  ready;

void EXTI0_IRQHandler(void) {
    data = read_sensor();
    __DMB();  // 确保 data 写入完成后再写 ready
    ready = 1;
}

// 任务中读取
void Task_Process(void) {
    if (ready) {
        __DMB();  // 确保读取 ready 后再读 data
        process(data);
        ready = 0;
    }
}
```

---

## ARM-004: portYIELD_FROM_ISR()

**严重度**: High
**适用平台**: Cortex-M 系列

**检查方法**:
- 搜索 ISR 中调用 `xQueueSendFromISR`、`xSemaphoreGiveFromISR`、`xTaskNotifyFromISR` 等 API
- 检查调用后是否跟随 `portYIELD_FROM_ISR(xHigherPriorityTaskWoken)` 或 `portEND_SWITCHING_ISR(xHigherPriorityTaskWoken)`
- 确认 `xHigherPriorityTaskWoken` 变量已正确初始化为 `pdFALSE`

**通过标准**:
- 每个调用 `FromISR` API 的 ISR，在 API 返回后调用 `portYIELD_FROM_ISR()`
- `xHigherPriorityTaskWoken` 在使用前初始化为 `pdFALSE`

**修复建议**:
```c
void USART1_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    uint8_t ch = USART1->DR;
    xQueueSendFromISR(uartQueue, &ch, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);  // 必须调用
}
```

---

## ARM-005: FromISR API 使用

**严重度**: Critical
**适用平台**: Cortex-M 系列

**检查方法**:
- 搜索所有 ISR 函数（`*_IRQHandler`、`HAL_*_Callback`）
- 检查 ISR 中是否使用了非 `FromISR` 版本的 FreeRTOS API
- 禁止的 API：`xQueueSend`、`xSemaphoreGive`、`xQueueReceive`、`vTaskDelay`、`vTaskSuspend` 等
- 必须使用的 API：`xQueueSendFromISR`、`xSemaphoreGiveFromISR`、`xQueueReceiveFromISR` 等

**通过标准**:
- ISR 中所有 FreeRTOS API 调用均使用 `FromISR` 后缀版本
- ISR 中无阻塞调用（`vTaskDelay`、`xSemaphoreTake` 等）

**修复建议**:
- 将 ISR 中的 `xQueueSend` 替换为 `xQueueSendFromISR`
- 将 `xSemaphoreGive` 替换为 `xSemaphoreGiveFromISR`
- 移除 ISR 中的阻塞调用，改为通过队列/信号量通知任务处理

---

## ARM-006: HardFault 处理

**严重度**: High
**适用平台**: Cortex-M 系列

**检查方法**:
- 搜索 `HardFault_Handler` 函数定义
- 检查是否实现了有效的故障诊断（读取 `SCB->HFSR`、`SCB->CFSR`、`SCB->MMFAR`、`SCB->BFAR`）
- 检查是否有断点或 LED 指示用于调试
- 确认生产版本中有适当的故障恢复机制

**通过标准**:
- `HardFault_Handler` 已实现且包含故障寄存器读取
- 调试版本中有断点或串口输出故障信息
- 生产版本中有安全状态恢复或系统复位

**修复建议**:
```c
void HardFault_Handler(void) {
    // 读取故障状态寄存器
    uint32_t hfsr = SCB->HFSR;
    uint32_t cfsr = SCB->CFSR;

    // 调试模式下进入死循环
    #ifdef DEBUG
    __BKPT(0);
    #else
    // 生产模式下安全复位
    NVIC_SystemReset();
    #endif
}
```

---

## ARM-007: 栈水位检查

**严重度**: Low
**适用平台**: Cortex-M 系列

**检查方法**:
- 搜索 `uxTaskGetStackHighWaterMark` 调用
- 检查是否在关键任务中定期检查栈使用情况
- 检查任务创建时的栈大小是否合理

**通过标准**:
- 关键任务有栈水位监控（可选，非强制）
- 任务栈大小有合理余量（高水位标记 < 栈大小的 80%）

**修复建议**:
```c
void Task_Monitor(void *pvParameters) {
    for (;;) {
        UBaseType_t watermark = uxTaskGetStackHighWaterMark(NULL);
        if (watermark < 64) {  // 剩余不足 64 字
            // 记录警告或增加栈大小
        }
        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}
```

---

## ARM-008: 位带操作原子性

**严重度**: Medium
**适用平台**: Cortex-M 系列

**检查方法**:
- 搜索使用 `ODR` 寄存器进行位操作的代码（如 `GPIOA->ODR |= GPIO_PIN_5`）
- 检查是否使用 `BSRR` 寄存器进行原子位操作
- 检查是否存在非原子的读-修改-写操作在中断上下文中

**通过标准**:
- GPIO 位操作使用 `BSRR` 寄存器而非 `ODR` 读-修改-写
- 或在临界区内进行 `ODR` 操作

**修复建议**:
```c
// 错误：非原子操作
GPIOA->ODR |= GPIO_PIN_5;   // 读-修改-写，可能被中断打断

// 正确：原子操作
GPIOA->BSRR = GPIO_PIN_5;   // 原子置位
GPIOA->BSRR = GPIO_PIN_5 << 16;  // 原子清零

// 或使用 HAL 库
HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);
```

---

## ARM-009: SysTick 配置

**严重度**: Low
**适用平台**: Cortex-M 系列

**检查方法**:
- 搜索 `HAL_SYSTICK_Config`、`SysTick_Config`、`HAL_SYSTICK_CLKSourceConfig` 调用
- 检查 SysTick 中断优先级是否设置为最低（`SysTick_IRQn` 优先级）
- 确认 `HAL_SYSTICK_Config(SystemCoreClock / 1000)` 是否与系统时钟匹配

**通过标准**:
- SysTick 配置为 1ms 中断周期
- SysTick 中断优先级为最低（FreeRTOS 要求）
- `HAL_Init()` 中已正确配置 SysTick

**修复建议**:
- 确保 `HAL_Init()` 在时钟配置前调用
- SysTick 优先级由 HAL 库自动设置为最低，通常无需手动配置
- 如使用自定义 SysTick，确保不与 FreeRTOS 冲突

---

## ARM-010: 低功耗模式配置

**严重度**: Low
**适用平台**: Cortex-M 系列

**检查方法**:
- 搜索 `HAL_PWR_EnterSLEEPMode`、`HAL_PWR_EnterSTOPMode`、`HAL_PWR_EnterSTANDBYMode` 调用
- 检查 FreeRTOS 的 `configUSE_IDLE_HOOK` 和 `configPRE_SLEEP_PROCESSING` / `configPOST_SLEEP_PROCESSING` 配置
- 确认唤醒源配置是否完整

**通过标准**:
- 如使用低功耗模式，唤醒源已正确配置
- FreeRTOS 空闲钩子中的低功耗逻辑正确
- 或项目明确不使用低功耗模式

**修复建议**:
- 如不需要低功耗，可忽略此规则
- 如需低功耗，实现 `vApplicationIdleHook` 并使用 `__WFI()` 指令
- 配置 `configPRE_SLEEP_PROCESSING` 和 `configPOST_SLEEP_PROCESSING` 进行睡眠前后处理

---

## ARM-011: 电源监控 PVD

**严重度**: Low
**适用平台**: Cortex-M 系列

**检查方法**:
- 搜索 `HAL_PWR_ConfigPVD`、`HAL_PWR_EnablePVD` 调用
- 检查 PVD 中断处理 `PVD_IRQHandler` 或 `PVD_VDDIO2_IRQHandler`
- 确认 PVD 阈值设置是否合理

**通过标准**:
- 如使用 PVD，阈值设置合理且中断处理正确
- 或项目明确不需要电源监控

**修复建议**:
- 如不需要电源监控，可忽略此规则
- 如需 PVD，配置合理的阈值并实现中断处理：
```c
HAL_PVD_ConfigTypeDef pvdConfig = {
    .PVDLevel = PVD_PLEVEL_2V2,  // 根据实际需求选择
    .Mode = PVD_MODE_IT_RISING_FALLING
};
HAL_PWR_ConfigPVD(&pvdConfig);
HAL_PWR_EnablePVD();
```

---

## ARM-012: 温度传感器读取

**严重度**: Low
**适用平台**: Cortex-M 系列

**检查方法**:
- 搜索内部温度传感器相关代码（`ADC_CHANNEL_TEMPSENSOR`、`TEMPSENSOR_CAL1_ADDR` 等）
- 检查校准值读取是否正确（从 Flash 读取出厂校准数据）
- 检查温度计算公式是否正确

**通过标准**:
- 如使用内部温度传感器，校准值读取和温度计算正确
- 或项目明确不使用内部温度传感器

**修复建议**:
```c
// 读取校准值
uint16_t cal30 = *TEMPSENSOR_CAL1_ADDR;  // 30度时的 ADC 值
uint16_t cal110 = *TEMPSENSOR_CAL2_ADDR; // 110度时的 ADC 值

// 计算温度
float temperature = ((float)(adc_value - cal30)) / ((float)(cal110 - cal30)) * (110.0f - 30.0f) + 30.0f;
```
- 确保 ADC 采样时间足够长（>= 10us）以获得准确读数
