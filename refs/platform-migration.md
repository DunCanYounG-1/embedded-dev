# 跨平台迁移指南

> 本文件提供在不同芯片平台之间迁移嵌入式代码的系统性方法，涵盖 STM32/ESP32/Arduino/RISC-V/NXP/TI/国产芯片之间的差异和兼容性处理。

---

## 迁移检查清单

开始迁移前，逐项核对：

| 检查项 | 说明 |
|--------|------|
| [ ] 确认目标芯片架构和资源约束（Flash/RAM/外设数量） |
| [ ] 列出源代码中使用的芯片特定外设和寄存器 |
| [ ] 识别需要重新适配的底层硬件抽象层 |
| [ ] 记录编译器差异（GCC/Clang/Keil/IAR） |
| [ ] 准备目标平台的开发环境（IDE/SDK） |

---

## 平台差异速查表

### 时钟系统对比

| 平台 | 时钟配置方式 | 主频 | 外设时钟 | 注意事项 |
|------|----------|------|---------|---------|
| **STM32** | RCC 寄存器组 | 72MHz/168MHz | APB1/APB2 独立分频 | 不同系列分频规则不同 |
| **ESP32** | rtc_clk/apb_clk | 240MHz | APB/定时器独立 | CPU 时钟可调，影响时序 |
| **Arduino** | 无时钟树（预设） | 16MHz (UNO) | F_CPU 宏定义 | 需手动计算定时器参数 |
| **RISC-V** | 厂商 SDK 时钟配置 | 144MHz/更高 | 总线时钟 | 时钟树结构简化 |
| **NXP** | SystemCoreClockUpdate() | 120MHz/180MHz | PERCLK 分频 | 不同系列外设时钟不同 |
| **TI MSP430** | CSCTL 寄存器 | 16MHz/25MHz | SMCLK 分频 | 超低功耗设计 |

**迁移要点**：
- STM32 → ESP32：需重新计算定时器/波特率（时钟频率从固定变为可调）
- ESP32 → STM32：需固定时钟频率（移除动态调频代码）
- Arduino → 其他平台：需移除 `F_CPU` 宏依赖，使用动态时钟获取函数

---

### 中断系统对比

| 平台 | 中断控制器 | 优先级机制 | 嵌套支持 | 注意事项 |
|------|----------|---------|---------|---------|
| **STM32** | NVIC | 抢占/子优先级 | 支持 | 不同系列分组数不同 |
| **ESP32** | CPU 中断 | Level（0-31） | 支持 | 无子优先级概念 |
| **Arduino** | AVR 中断向量 | 固定优先级 | 有限制 | 向量表固定，需修改 |
| **RISC-V** | PLIC | 优先级分组 | 支持 | 标准 RISC-V 中断 |
| **NXP** | NVIC | 抢占/子优先级 | 支持 | 类 STM32 但寄存器偏移不同 |
| **TI MSP430** | 中断向量表 | 固定优先级 | 有限制 | 超低优先中断数量少 |

**迁移要点**：
- STM32 → ESP32：`NVIC_SetPriority()` → `esp_intr_alloc()`，优先级从分组/子优先变为 0-31 级
- ESP32 → STM32：`esp_intr_alloc()` → `NVIC_SetPriority()`，需要设计优先级到分组/子优先级的映射
- Arduino → 其他平台：直接 ISR 函数 → 需要平台特定的中断注册 API

---

### DMA 系统对比

| 平台 | DMA 控制器 | 通道数量 | 外设支持 | 注意事项 |
|------|----------|---------|---------|---------|
| **STM32** | DMA1/DMA2 | 8-16 通道 | UART/SPI/I2C/ADC/定时器 | 不同系列可用通道不同 |
| **ESP32** | GDMA | 4 个通道 | SPI/I2C/UART | 使用 DMA 描述符 |
| **Arduino** | 无 DMA（UNO） | 无 | 无 | 部分板卡有 DMA（如 Mega） |
| **RISC-V** | 厂商 DMA | 依赖芯片 | UART/SPI/ADC | 实现差异大 |
| **NXP** | DMA | 4-32 通道 | UART/SPI/I2C | 复杂触发配置 |
| **TI MSP430** | DMA | 4-8 通道 | ADC/UART | 传输配置灵活 |

**迁移要点**：
- STM32 → ESP32：`DMA_Init()` → `dma_channel_alloc()` + `lldesc`，需重新学习 DMA 描述符
- ESP32 → STM32：`dma_channel_alloc()` → `DMA_Init()`，需手动配置 DMA 寄存器
- 无 DMA 平台 → 有 DMA 平台：性能优化方案（中断轮询 → DMA）

---

### 外设 API 对比

#### GPIO 对比

| 平台 | 初始化 | 设置输出 | 读取输入 | 复用 |
|------|--------|---------|---------|------|
| **STM32** | `GPIO_Init()` | `GPIO_SetBits()` | `GPIO_ReadInputDataBit()` | `GPIO_PinRemapConfig()` |
| **ESP32** | `gpio_config()` | `gpio_set_level()` | `gpio_get_level()` | `gpio_matrix_t` 查表 |
| **Arduino** | `pinMode()` | `digitalWrite()` | `digitalRead()` | 无复用概念 |
| **RISC-V** | 厂商 GPIO API | 厂商 GPIO API | 厂商 GPIO API | 厂商 GPIO API |
| **NXP** | `GPIO_PinInit()` | `GPIO_SetPinLevel()` | `GPIO_ReadPinLevel()` | `GPIO_SetPinMux()` |
| **TI MSP430** | `P1DIR` | `P1OUT` | `P1IN` | 无复用（端口独立） |

**迁移要点**：
- STM32 → ESP32：结构体 `GPIO_InitTypeDef` → `gpio_config_t`，需重构初始化代码
- Arduino → 其他平台：移除 `digitalWrite()` 等高层 API，使用底层寄存器操作

#### UART 对比

| 平台 | 初始化 | 发送 | 接收 | 中断 | 波特率计算 |
|------|--------|------|------|------|---------|
| **STM32** | `USART_Init()` | `USART_SendData()` | `USART_ReceiveData()` | `USART_ITConfig()` | `BRR = fPCLK / (16 * baud)` |
| **ESP32** | `uart_driver_install()` | `uart_write_bytes()` | `uart_read_bytes()` | `uart_enable_intr()` | APB 频率直接配置 |
| **Arduino** | `Serial.begin()` | `Serial.print()` | `Serial.read()` | `attachInterrupt()` | 自动计算 |
| **RISC-V** | 厂商 UART API | 厂商 UART API | 厂商 UART API | 厂商 UART API | 依赖芯片 |
| **NXP** | `UART_Init()` | `UART_WriteData()` | `UART_ReadData()` | `UART_SetIntCmd()` | `BRGVAL = fperiph / (16 * baud)` |
| **TI MSP430** | `UCAxCTL1` | `UCAxTXBUF` | `UCAxRXBUF` | `UCAxIE` | `BR = fBRCLK / baud` |

**迁移要点**：
- 波特率计算方式不同：STM32/NXP 用 BRR 寄存器，TI MSP430 用 BR 寄存器，ESP32 直接配置频率
- 中断模式不同：STM32 用 `USART_ITConfig()`，ESP32 用 `uart_enable_intr()`

#### SPI 对比

| 平台 | 初始化 | 发送 | 接收 | 时钟极性/相位 |
|------|--------|------|------|-------------|
| **STM32** | `SPI_Init()` | `SPI_I2S_SendData()` | `SPI_I2S_ReceiveData()` | `SPI_CPOL_Low/High` `SPI_CPHA_1Edge/2Edge` |
| **ESP32** | `spi_bus_initialize()` | `spi_device_transmit()` | `spi_device_receive()` | `SPI_POLARITY_LOW/HIGH` `SPI_PHASE_1EDGE/2EDGE` |
| **Arduino** | `SPI.begin()` | `SPI.transfer()` | `SPI.transfer()` | `SPI_MODE0/1/2/3` |
| **RISC-V** | 厂商 SPI API | 厂商 SPI API | 厂商 SPI API | 厂商 SPI API | 依赖芯片 |
| **NXP** | `SPI_Init()` | `SPI_WriteData()` | `SPI_ReadData()` | `SPI_CPOL_LOW/HIGH` `SPI_CPHA_1ST/2ND_EDGE` |
| **TI MSP430** | `UxCTL1` | `UxTXBUF` | `UxRXBUF` | `UCCKPH` `UCCKPL` |

**迁移要点**：
- STM32 和 ESP32 的时钟极性/相位配置相似，但枚举值不同
- Arduino 使用 Mode 编码（0-3），需解码为极性/相位参数

#### ADC 对比

| 平台 | 初始化 | 启动转换 | 读取结果 | 分辨率 |
|------|--------|---------|---------|--------|
| **STM32** | `ADC_Init()` | `ADC_SoftwareStartConvCmd()` | `ADC_GetConversionValue()` | 12位 |
| **ESP32** | `adc1_config_width()` | `adc1_start()` | `adc1_get_raw()` | 12位可配置 |
| **Arduino** | `analogReference()` | `analogRead()` | 内部启动 | 10位（UNO） |
| **RISC-V** | 厂商 ADC API | 厂商 ADC API | 厂商 ADC API | 厂商 ADC API | 依赖芯片 |
| **NXP** | `ADC_Init()` | `ADC_StartCmd()` | `ADC_GetDataReg()` | 10-12位 |
| **TI MSP430** | `ADC12CTL0` | 自动连续采样 | `ADC12MEM0` | 12位 |

**迁移要点**：
- ADC 启动方式不同：STM32 软件触发，ESP32 可配置连续/单次，TI MSP430 自动采样
- 分辨率处理：10位→12位需调整量程和精度计算

#### 定时器对比

| 平台 | 初始化 | 启动 | PWM 配置 | 互补输出 |
|------|--------|------|---------|---------|
| **STM32** | `TIM_TimeBaseInit()` | `TIM_Cmd()` | `TIM_OCInit()` | `TIM_CCxN`（高级定时器） |
| **ESP32** | `timer_init()` | `timer_start()` | `ledc_set_duty()` | `ledc_channel_config_t` |
| **Arduino** | `TimerX.initialize()` | `TimerX.start()` | `TimerX.setPwmDuty()` | 有限制 |
| **RISC-V** | 厂商 Timer API | 厂商 Timer API | 厂商 Timer API | 厂商 Timer API | 依赖芯片 |
| **NXP** | `TIM_Init()` | `TIM_Cmd()` | `TIM_MatchUpdate()` | 有限制 |
| **TI MSP430** | `TAxCCR0` | `TAxCTL` | `TAxCCR0` | 无互补输出 |

**迁移要点**：
- 互补输出：STM32 高级定时器支持，ESP32 用 LEDC 模拟，需重新设计死区插入逻辑
- PWM 占空比计算：不同平台的自动重装载值计算方式不同

---

## 编译器差异处理

### 宏定义差异

| 宏 | STM32 | ESP32 | Arduino | RISC-V | 处理方式 |
|----|-------|-------|---------|--------|---------|
| 位操作 | `__set_bit()` / `__get_bit()` | 无 | 无 | 无 | 统一使用 `(1 << n)` 移位 |
| 中断声明 | `__attribute__((interrupt))` | `IRAM_ATTR` | `ISR()` | `__attribute__((interrupt))` | 平台特定宏封装 |
| 对齐 | `__attribute__((aligned(4)))` | 无 | 无 | 无 | 条件编译或统一用标准属性 |
| 优化 | `__attribute__((optimize("O3")))` | `__attribute__((optimize("O2")))` | `-O2` 编译选项 | 厂商编译选项 | 检查编译器文档 |

### 关键字差异

| 类型 | STM32/标准 C | ESP32 | Arduino | 处理方式 |
|------|-------------|-------|---------|---------|
| 布尔 | `uint8_t` (0/1) | `bool` | `bool` | 条件定义 |
| 中断向量 | `typedef void (*pFunc)(void)` | `void (*fn)(void*)` | `void (*fn)()` | 宏统一类型 |
| volatile | `volatile` | `volatile` | `volatile` | 统一使用 |

---

## 常见迁移场景

### STM32 → ESP32

**时钟迁移**：
```c
// STM32
RCC_APB2PeriphClockCmd(RCC_APB2Periph_TIM1, ENABLE);
TIM_TimeBaseInit(TIM1, &TIM_TimeBaseStructure);

// ESP32
periph_module_enable(PERIPH_TIMG0_MODULE);  // 使用厂商 SDK 函数
timer_init(TIMER_GROUP_0, TIMER_0, &timer_config);
```

**GPIO 迁移**：
```c
// STM32
GPIO_InitTypeDef GPIO_InitStructure;
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_5;
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
GPIO_Init(GPIOA, &GPIO_InitStructure);

// ESP32
gpio_config_t io_conf = {
    .pin_bit_mask = (1ULL << 5),
    .mode = GPIO_MODE_OUTPUT,
};
gpio_config(&io_conf);
gpio_set_level(5, 1);
```

**串口迁移**：
```c
// STM32
USART_Init(USART1, &USART_InitStructure);
USART_Cmd(USART1, ENABLE);

// ESP32
uart_config_t uart_config = {
    .baud_rate = 115200,
    .data_bits = UART_DATA_8_BITS,
};
uart_param_config(UART_NUM_1, &uart_config);
uart_driver_install(UART_NUM_1, &uart_config);
```

### Arduino → STM32

**GPIO 迁移**：
```c
// Arduino
pinMode(13, OUTPUT);
digitalWrite(13, HIGH);

// STM32
GPIO_InitTypeDef GPIO_InitStructure;
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
GPIO_Init(GPIOC, &GPIO_InitStructure);
GPIO_SetBits(GPIOC, GPIO_Pin_13);
```

**定时器迁移**：
```c
// Arduino
Timer1.initialize(1000000);  // 1MHz
Timer1.setPwmDuty(500);      // 50% 占空比

// STM32 (假设 72MHz 时钟)
TIM_TimeBaseInitStructure.TIM_Prescaler = 71;  // 72MHz / 72 = 1MHz
TIM_TimeBaseInitStructure.TIM_Period = 999;  // 1000 计数
TIM_OCInitStructure.TIM_Pulse = 499;     // 50% 占空比 (499+1)/1000
```

### RISC-V → STM32

**GPIO 迁移**：
```c
// RISC-V (厂商 SDK，如 GD32)
gpio_init_type gpio_init_struct;
gpio_init_struct.pin = GPIO_PIN_5;
gpio_init_struct.mode = GPIO_MODE_OUTPUT;
gpio_init(&gpio_init_struct);

// STM32
GPIO_InitTypeDef GPIO_InitStructure;
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_5;
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
GPIO_Init(GPIOA, &GPIO_InitStructure);
```

**中断迁移**：
```c
// RISC-V
plic_set_interrupt_priority(IRQ_TMR0_BRK, 1);
plic_interrupt_enable(IRQ_TMR0_BRK);

// STM32
NVIC_Init(&NVIC_InitStructure);
NVIC_InitStructure.NVIC_IRQChannel = TIM1_BRK_IRQn;
NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1;
NVIC_Init(&NVIC_InitStructure);
TIM_ITConfig(TIM1, TIM_IT_Break, ENABLE);
```

---

## 迁移后验证清单

完成迁移后，逐项验证：

| 验证项 | 验证方法 | 通过标准 |
|--------|---------|---------|
| 编译无错误 | 编译输出 clean | 0 errors, 0 warnings |
| 链接成功 | map 文件检查 | 所有符号正确解析 |
| 代码大小 | 查看 size 输出 | Flash/RAM 在限制内 |
| 外设功能 | 逻辑分析仪/示波器 | 波形符合预期 |
| 中断响应 | 断点测试 | 进入中断函数 |
| DMA 传输 | 内存检查 | 数据正确传输 |

---

## 迁移优化建议

**性能优化**：
- 优先使用平台优化的外设 API（如 STM32 HAL 的 DMA 优化版本）
- 避免平台无关的抽象层（直接使用寄存器操作）
- 利用平台特定特性（如 ESP32 的双核、STM32 的硬件浮点）

**可维护性**：
- 使用条件编译隔离平台特定代码
```c
#ifdef STM32F10X
    GPIO_Init(...);
#elif defined(ESP_PLATFORM)
    gpio_config(...);
#endif
```
- 创建硬件抽象层（HAL）统一接口，底层由具体平台实现
- 文档化平台差异，添加 TODO 标记需要适配的地方

**调试友好**：
- 保留源代码的结构，便于对比问题
- 添加平台特定的断言和检查
- 使用相同的命名约定，减少搜索成本

---

## 联系其他参考文档

- **引脚规划**：见 `refs/pin-planning.md` 跨平台引脚约束速查
- **代码规范**：见 `refs/coding-standards.md` 模块化编程和命名规范
- **故障排查**：见 `refs/troubleshooting.md` 平台特定问题排查

---

## 总结

跨平台迁移的关键成功因素：
1. **充分理解源平台**的架构、API 设计和假设
2. **系统化映射**目标平台 API，而非逐行翻译
3. **优先使用平台 SDK**提供的优化和抽象
4. **验证每个外设**的功能，而非一次性整体测试
5. **文档化迁移决策**，记录为什么不直接翻译而采用不同实现
