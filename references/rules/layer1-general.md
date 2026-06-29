# Layer 1: General Embedded Rules

通用嵌入式检查规则，适用于所有 MCU 平台。

---

### GEN-001: 共享变量 volatile

**严重度**: Critical
**适用平台**: 所有 MCU

**检查方法**:
- 搜索所有 ISR（中断服务函数）中访问的全局变量
- 检查这些变量是否声明为 `volatile`
- 搜索模式：在 `void *_Handler(void)`、`void *_IRQHandler(void)` 函数体内赋值或读取的全局变量
- 反向验证：搜索未加 `volatile` 的全局变量，检查是否在 ISR 中被引用

**通过标准**:
- 所有在 ISR 中读写的全局变量均声明为 `volatile`
- DMA 缓冲区变量声明为 `volatile`

**修复建议**:
- 对 ISR 共享变量添加 `volatile` 关键字：`volatile uint32_t flag = 0;`
- DMA 缓冲区同样需要 `volatile`：`volatile uint8_t dma_buf[256];`

---

### GEN-002: ISR 保持简短

**严重度**: High
**适用平台**: 所有 MCU

**检查方法**:
- 在所有 `*_IRQHandler` 函数中检查以下阻塞操作：
  - `HAL_Delay`、`delay_ms`、`delay_us` 等延时函数
  - `HAL_UART_Transmit`（非 DMA/IT 模式）等阻塞式外设调用
  - `printf`、`sprintf` 等格式化输出
  - `for`/`while` 忙等循环（非超时保护）
  - 浮点运算（无 FPU 的 MCU）

**通过标准**:
- ISR 中无阻塞式延时
- ISR 中无阻塞式外设传输
- ISR 中无浮点运算（无 FPU 时）
- 长操作通过标志位延迟到主循环处理

**修复建议**:
- 将耗时操作移到主循环，ISR 仅设置标志位
- 使用 DMA/IT 模式替代阻塞式传输
- 使用 `xQueueSendFromISR()` 等 RTOS 安全 API 传递数据

---

### GEN-003: 互斥锁保护共享资源

**严重度**: Critical
**适用平台**: 所有 MCU

**检查方法**:
- 搜索 `HAL_I2C_*`、`HAL_SPI_*`、`HAL_UART_*` 等外设 API 调用
- 检查同一外设句柄是否被多个任务/线程使用
- 如果多任务共享同一外设句柄，检查是否有 `osMutexWait`/`xSemaphoreTake` 等互斥保护
- 检查模式：`osMutexId` 或 `SemaphoreHandle_t` 声明 → `Wait/Take` → 外设操作 → `Release/Give`

**通过标准**:
- 多任务共享的外设有互斥锁保护
- 互斥锁覆盖完整的 "获取 → 操作 → 释放" 流程

**修复建议**:
- 为共享外设创建互斥锁：`osMutexId(i2c_mutex)`
- 调用前获取锁，调用后释放锁
- 单任务独占的外设可不加互斥锁

---

### GEN-004: 互斥锁内无耗时操作

**严重度**: High
**适用平台**: 所有 MCU

**检查方法**:
- 在 `osMutexWait`/`xSemaphoreTake` 和 `osMutexRelease`/`xSemaphoreGive` 之间检查：
  - 浮点运算（`float`/`double` 运算、`sqrt`、`sin` 等数学函数）
  - `HAL_UART_Transmit`（阻塞模式，尤其是 printf 重定向）
  - `HAL_Delay` 延时
  - 大块 `memcpy`/`memset`（超过 256 字节）

**通过标准**:
- 互斥锁持有时间尽量短
- 互斥锁内无浮点运算
- 互斥锁内无阻塞式 UART 传输

**修复建议**:
- 浮点计算在锁外完成后，仅在锁内赋值结果
- UART 输出移到锁外
- 减小临界区范围，只保护必要的外设操作

---

### GEN-005: 看门狗配置

**严重度**: High
**适用平台**: 所有 MCU

**检查方法**:
- 搜索 `MX_IWDG_Init` 或 `IWDG_HandleTypeDef` 是否存在
- 如果使用 IWDG，检查预分频器和重载值是否配置合理
- 检查 CubeMX `.ioc` 文件中 IWDG 是否启用
- 验证超时时间 = `(Prescaler × Reload) / LSI_Freq`，建议 1~5 秒

**通过标准**:
- IWDG 已初始化且超时时间在 1~5 秒范围内
- 或项目明确说明不使用看门狗（有注释说明原因）

**修复建议**:
- 在 CubeMX 中启用 IWDG，配置合理超时
- 典型配置：Prescaler=64, Reload=625 → 超时约 2 秒（LSI=40kHz）

---

### GEN-006: 看门狗定期刷新

**严重度**: High
**适用平台**: 所有 MCU

**检查方法**:
- 搜索 `HAL_IWDG_Refresh` 或 `IWDG->KR = 0xAAAA` 调用
- 确认刷新调用在主循环中定期执行（而非仅在初始化时）
- 检查刷新频率是否小于看门狗超时时间
- 在 FreeRTOS 中，检查是否在所有关键任务中都有刷新点

**通过标准**:
- 主循环或至少一个周期性任务中包含 `HAL_IWDG_Refresh` 调用
- 刷新间隔 < 看门狗超时时间

**修复建议**:
- 在主循环 `while(1)` 中添加 `HAL_IWDG_Refresh(&hiwdg);`
- FreeRTOS 中可在最高优先级任务中刷新

---

### GEN-007: 栈溢出检测

**严重度**: High
**适用平台**: 所有 MCU（使用 RTOS 时）

**检查方法**:
- 搜索 `vApplicationStackOverflowHook` 函数是否实现
- 检查 `FreeRTOSConfig.h` 中 `configCHECK_FOR_STACK_OVERFLOW` 是否设置为 2
- 检查各任务栈大小是否合理（不应小于 128 字，即 512 字节）

**通过标准**:
- `vApplicationStackOverflowHook` 已实现且有错误处理（如 LED 报警、复位）
- `configCHECK_FOR_STACK_OVERFLOW >= 2`
- 各任务栈大小合理

**修复建议**:
- 实现 `vApplicationStackOverflowHook`，进入死循环或触发复位
- 设置 `configCHECK_FOR_STACK_OVERFLOW = 2` 启用完整检测
- 使用 `uxTaskGetStackHighWaterMark` 评估实际栈使用

---

### GEN-008: 堆分配失败处理

**严重度**: High
**适用平台**: 所有 MCU（使用 RTOS 时）

**检查方法**:
- 搜索 `vApplicationMallocFailedHook` 函数是否实现
- 检查 `FreeRTOSConfig.h` 中 `configUSE_MALLOC_FAILED_HOOK` 是否为 1
- 检查 `configTOTAL_HEAP_SIZE` 是否合理

**通过标准**:
- `vApplicationMallocFailedHook` 已实现且有错误处理
- `configUSE_MALLOC_FAILED_HOOK = 1`

**修复建议**:
- 实现 `vApplicationMallocFailedHook`，进入死循环或触发复位
- 使用 `xPortGetFreeHeapSize()` 监控堆使用情况
- 合理配置 `configTOTAL_HEAP_SIZE`

---

### GEN-009: 错误处理函数

**严重度**: Medium
**适用平台**: 所有 MCU

**检查方法**:
- 搜索 `Error_Handler` 函数实现
- 检查函数体是否为空或仅有死循环
- 检查是否有诊断信息输出（LED 闪烁、UART 输出、复位等）

**通过标准**:
- `Error_Handler` 函数存在且非空
- 有至少一种错误指示方式（LED、UART、复位）

**修复建议**:
- 在 `Error_Handler` 中关闭中断、闪烁 LED 指示错误
- 可选：输出错误信息到 UART 或触发软复位
- 示例：
  ```c
  void Error_Handler(void) {
      __disable_irq();
      while (1) {
          HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
          HAL_Delay(100);
      }
  }
  ```

---

### GEN-010: 复位原因检查

**严重度**: Low
**适用平台**: 所有 MCU

**检查方法**:
- 搜索 `RCC->CSR` 寄存器读取
- 检查 `__HAL_RCC_GET_FLAG(RCC_FLAG_*)` 调用
- 检查复位原因是否被记录或处理

**通过标准**:
- 程序启动时读取并记录复位原因
- 异常复位（看门狗、低功耗）有特殊处理逻辑

**修复建议**:
- 启动时读取 `RCC->CSR` 判断复位源
- 记录到日志或变量中，便于调试
- 示例：
  ```c
  uint32_t reset_cause = RCC->CSR;
  if (reset_cause & RCC_CSR_IWDGRSTF) { /* IWDG 复位处理 */ }
  __HAL_RCC_CLEAR_RESET_FLAGS();
  ```

---

### GEN-011: I2C 总线恢复

**严重度**: High
**适用平台**: 所有 MCU

**检查方法**:
- 搜索 `HAL_I2C_GetError` 返回 `HAL_I2C_ERROR_BUSY` 的处理
- 检查是否有 I2C 总线恢复函数（手动时钟脉冲释放 SDA）
- 检查 `HAL_I2C_Init` 调用前是否有总线状态检查
- 搜索 SCL/SDA 引脚的手动 GPIO 翻转逻辑

**通过标准**:
- I2C 通信失败时有总线恢复机制
- 或有 I2C 外设重新初始化的逻辑

**修复建议**:
- 实现 I2C Bus Recovery：手动驱动 SCL 产生最多 9 个时钟脉冲直到 SDA 释放
- 检测到 BUSY 错误时执行恢复序列后重新初始化 I2C
- 示例：
  ```c
  void I2C_BusRecovery(void) {
      // 切换 SCL 为 GPIO 输出，手动产生时钟脉冲
      for (int i = 0; i < 9; i++) {
          HAL_GPIO_WritePin(SCL_GPIO, SCL_PIN, GPIO_PIN_SET);
          delay_us(5);
          HAL_GPIO_WritePin(SCL_GPIO, SCL_PIN, GPIO_PIN_RESET);
          delay_us(5);
      }
      // 重新初始化 I2C 外设
      HAL_I2C_Init(&hi2c1);
  }
  ```

---

### GEN-012: I2C 超时处理

**严重度**: Medium
**适用平台**: 所有 MCU

**检查方法**:
- 搜索所有 `HAL_I2C_Master_Transmit`、`HAL_I2C_Mem_Read` 等 HAL_I2C 调用
- 检查超时参数是否合理（不应使用 `HAL_MAX_DELAY`）
- 检查返回值是否被检查（`!= HAL_OK` 的处理）

**通过标准**:
- HAL_I2C 调用的超时参数在合理范围内（如 100~1000ms）
- 返回值被检查且有错误处理

**修复建议**:
- 使用合理超时值：`HAL_I2C_Master_Transmit(&hi2c, addr, buf, len, 500);`
- 检查返回值并执行恢复或重试逻辑
- 避免使用 `HAL_MAX_DELAY` 导致无限等待

---

### GEN-013: SPI 片选管理

**严重度**: Medium
**适用平台**: 所有 MCU

**检查方法**:
- 搜索 SPI 通信中 CS（片选）引脚的管理方式
- 检查是否使用 `HAL_GPIO_WritePin` 手动控制 CS
- 确认 CS 在传输前拉低、传输后拉高
- 检查是否存在 CS 未释放就进行下一次通信的情况

**通过标准**:
- SPI 通信中 CS 引脚有正确的拉低/拉高时序
- 使用 HAL 管理 CS 时 NSS 配置正确

**修复建议**:
- 传输前拉低 CS：`HAL_GPIO_WritePin(CS_GPIO, CS_PIN, GPIO_PIN_RESET);`
- 传输完成后拉高 CS：`HAL_GPIO_WritePin(CS_GPIO, CS_PIN, GPIO_PIN_SET);`
- 确保即使传输失败也会释放 CS（放在 finally 或检查返回值后释放）

---

### GEN-014: UART 接收缓冲区初始化

**严重度**: Medium
**适用平台**: 所有 MCU

**检查方法**:
- 搜索 UART 接收缓冲区声明
- 检查是否在使用前通过 `memset` 或 `= {0}` 初始化
- 检查 DMA 接收缓冲区是否在启动接收前清零

**通过标准**:
- UART 接收缓冲区在首次使用前已清零
- DMA 接收缓冲区已初始化

**修复建议**:
- 声明时初始化：`uint8_t rx_buf[256] = {0};`
- 或在启动接收前：`memset(rx_buf, 0, sizeof(rx_buf));`

---

### GEN-015: UART 命令解析防御

**严重度**: Medium
**适用平台**: 所有 MCU

**检查方法**:
- 搜索 UART 命令解析函数
- 检查是否处理以下边界情况：
  - 空命令（长度为 0 或仅包含换行符）
  - 超长命令（超过缓冲区长度）
  - 无效字符
  - 命令字符串未正确终止（缺少 `\0`）

**通过标准**:
- 空命令被忽略或返回错误
- 超长命令被截断并有错误提示
- 缓冲区溢出已防御

**修复建议**:
- 检查接收长度 > 0 再解析
- 确保字符串以 `\0` 终止：`rx_buf[rx_len] = '\0';`
- 限制最大命令长度并检查输入

---

### GEN-016: Flash 写入保护

**严重度**: High
**适用平台**: 所有 MCU

**检查方法**:
- 搜索 `HAL_FLASH_Unlock` 和 `HAL_FLASH_Lock` 调用
- 确认每个 `Unlock` 都有对应的 `Lock`
- 检查 Flash 写入操作是否在 Unlock/Lock 对之间
- 检查写入失败时是否执行 Lock

**通过标准**:
- 每次 `HAL_FLASH_Unlock` 都有配对的 `HAL_FLASH_Lock`
- 写入失败时也执行 Lock
- Flash 操作期间中断已被适当处理

**修复建议**:
- 使用 RAII 模式或确保所有路径都有 Lock
- 示例：
  ```c
  HAL_FLASH_Unlock();
  // ... Flash 操作 ...
  HAL_FLASH_Lock();  // 确保所有分支都能执行
  ```

---

### GEN-017: Flash 写入对齐

**严重度**: Medium
**适用平台**: 所有 MCU

**检查方法**:
- 搜索 `HAL_FLASH_Program` 调用
- 检查目标地址是否按 Flash 字长对齐
  - STM32F1: 半字（2 字节）对齐
  - STM32F4/H7: 字（4 字节）或双字（8 字节）对齐
- 检查 `FLASH_TYPEPROGRAM` 类型与地址对齐是否匹配

**通过标准**:
- Flash 写入地址按目标 MCU 的 Flash 字长正确对齐
- 编程类型与对齐要求匹配

**修复建议**:
- 确保地址对齐：`addr & 0x03 == 0`（字对齐）或 `addr & 0x07 == 0`（双字对齐）
- 使用 `__ALIGN` 宏或手动对齐检查

---

### GEN-018: 时钟频率不要硬编码

**严重度**: Medium
**适用平台**: 所有 MCU

**检查方法**:
- 搜索波特率、定时器周期等计算中直接使用数字常量的地方
  - 例：`huart1.Init.BaudRate = 115200;` 本身没问题
  - 例：`htim.Init.Prescaler = 7200 - 1;` 中 `7200` 硬编码了时钟频率
- 检查是否使用 `HAL_RCC_GetPCLK1Freq()`、`HAL_RCC_GetPCLK2Freq()` 获取实际时钟
- 检查 `SystemClock_Config` 中的时钟树配置注释

**通过标准**:
- 波特率计算使用 `HAL_RCC_GetPCLKxFreq()` 而非硬编码值
- 定时器周期计算基于实际时钟频率

**修复建议**:
- 使用 HAL 函数获取时钟：`uint32_t pclk = HAL_RCC_GetPCLK1Freq();`
- 定时器计算：`Prescaler = (pclk / desired_freq) - 1;`

---

### GEN-019: 除零防御

**严重度**: Medium
**适用平台**: 所有 MCU

**检查方法**:
- 搜索频率、采样率、波特率等计算中的除法运算
- 检查除数是否可能为 0（变量作为除数时特别关注）
- 关注以下场景：
  - 用户输入的频率/采样率值
  - 运行时计算的分频系数
  - 数组索引除法

**通过标准**:
- 所有除法运算的除数在运行时不可能为 0
- 用户输入的值在除法前有范围检查

**修复建议**:
- 除法前检查：`if (divisor == 0) return ERROR;`
- 限制用户输入范围：`if (freq == 0 || freq > MAX_FREQ) return ERROR;`
- 示例：
  ```c
  uint32_t prescaler = SystemCoreClock / target_freq;
  if (prescaler == 0) prescaler = 1;  // 防御
  ```

---

### GEN-020: 资源使用记录

**严重度**: Low
**适用平台**: 所有 MCU

**检查方法**:
- 检查是否有 `.map` 文件或链接器脚本中的资源使用记录
- 搜索 `__attribute__((section` 或自定义链接脚本
- 检查是否有 Flash/RAM 使用率的注释或文档

**通过标准**:
- 项目中有 Flash/RAM 使用量的记录（文档、注释或构建输出）
- Flash 使用率 < 90%，RAM 使用率 < 80%（留有余量）

**修复建议**:
- 记录构建输出中的 `text`（Flash）和 `data`/`bss`（RAM）大小
- 在 README 或构建脚本中记录资源使用情况
- 监控使用率，避免接近上限
