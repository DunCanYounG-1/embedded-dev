# 引脚规划指南

> 本文件提供嵌入式开发中的引脚规划方法，包括引脚冲突检测、外设匹配原则、推荐分配方案和常见芯片的引脚约束。

---

## 引脚规划流程

### 阶段 1：收集外设需求

在 RESEARCH 阶段，列出所有需要的外设及其功能需求：

| 外设类型 | 功能 | 需求数量 | 特殊要求 |
|---------|------|---------|---------|
| 串口 | USART1 | 1 | TX/RX |
| PWM | TIM2 | 3 | 互补输出带死区 |
| SPI | SPI1 | 1 | MOSI/MISO/SCK/CS |
| I2C | I2C1 | 1 | SDA/SCL，开漏输出 |
| ADC | ADC1 | 4 | 通道互不冲突 |
| GPIO | LED/按键 | 2 | 上拉/下拉 |

---

### 阶段 2：查询芯片引脚约束

**必须查阅数据手册或使用 Context7 MCP 查询**，记录关键约束：

#### STM32F103C8T6 示例

```
可用外设复用：
- PA0-PA3: ADC_IN0-3
- PA9/PA10: USART1_TX/RX
- PB6/PB7: I2C1_SCL/SDA
- PA13/PA14: SWDIO/SWCLK（调试口，不建议占用）
- PB0-PB1: TIM3_CH3/CH4
- PA6/PA7: TIM3_CH1/CH2

引脚冲突：
- USB: PA11/PA12（不可作为 GPIO）
- OSC: PD0/PD1（不可作为 GPIO）
```

#### ESP32-WROOM-32 示例

```
STRAP 引脚（上电时决定启动模式，使用需谨慎）：
- GPIO0: 启动模式选择，上拉=Flash启动
- GPIO2: 启动模式选择
- GPIO12: 启动模式选择，内部下拉

输入输出模式限制：
- 仅输入：GPIO34-39（无输出驱动能力）
- 仅输出：GPIO25/26（无输入）

默认功能引脚：
- GPIO1/GPIO3: UART0 TX/RX（默认串口）
- GPIO6-11: Flash SPI（不可使用）
```

---

### 阶段 3：生成推荐分配方案

#### 原则 1：优先使用默认引脚

芯片厂商已经验证过的引脚组合，问题最少。

| 外设 | 推荐引脚 | 理由 |
|------|---------|------|
| USART1 | PA9(TX) / PA10(RX) | 默认引脚，无需重映射 |
| I2C1 | PB6(SCL) / PB7(SDA) | 默认引脚，无冲突 |
| SPI1 | PA5(SCK) / PA6(MISO) / PA7(MOSI) | 默认引脚 |
| ADC1_CH0 | PA0 | 默认通道 |

#### 原则 2：避免占用调试口

| 调试口 | 芯片 | 影响 |
|--------|------|------|
| SWDIO/SWCLK | STM32 | 禁止后无法烧录 |
| RX/TX | ESP32 | 禁止后无法串口烧录 |

**替代方案**：如必须占用，先烧录固件再切换引脚功能。

#### 原则 3：按功能区域分组

将相关引脚分配在物理位置相近的区域，便于 PCB 布线。

示例：传感器接口集中在端口 B
```
传感器板连接器:
PB6 - I2C1_SCL
PB7 - I2C1_SDA
PB0 - ADC_IN8（备用）
PB1 - ADC_IN9（备用）
```

---

## 引脚冲突检测矩阵

### STM32F103 系列冲突表

| 引脚 | 复用功能1 | 复用功能2 | 复用功能3 | 冲突提示 |
|------|----------|----------|----------|---------|
| PA0 | ADC_IN0 | TIM2_CH1_ETR | WKUP | 作为 ADC 时不能用作 PWM |
| PA9 | USART1_TX | TIM1_CH2 | - | USART 和 TIM1 冲突 |
| PB3 | TIM2_CH2 | SPI1_SCK | - | 需禁用 SPI1 才能用 TIM2 |
| PB6 | I2C1_SCL | TIM4_CH1 | USART1_TX | 重映射后可用 |

### ESP32-WROOM-32 冲突表

| 引脚 | 默认功能 | STRAP影响 | 冲突提示 |
|------|---------|----------|---------|
| GPIO0 | UART0_TX | 启动模式选择 | 1=Flash烧录，0=UART下载 |
| GPIO12 | JTAG_MTCK | 内部下拉+上电影响 | 上电时影响电压 |
| GPIO34 | ADC1_CH6 | 仅输入 | 不可作为输出 |
| GPIO16 | PSRAM_CS（WROVER 模组占用） | 双向 IO | WROVER 模组不可用，WROOM 可用 |

---

## 引脚规划检查清单

在 PLAN 阶段，逐项检查：

- [ ] **调试口保留**：未占用 SWDIO/SWCLK 或烧录关键引脚
- [ ] **STRAP 引脚确认**：已查阅启动模式引脚，避免冲突
- [ ] **外设互斥检查**：同一引脚的复用功能不会同时启用
- [ ] **电源域匹配**：引脚的电源域（如 1.8V vs 3.3V）符合要求
- [ ] **电气特性匹配**：
  - 推挽输出 vs 开漏输出
  - 上拉/下拉电阻需求
  - 驱动强度
- [ ] **中断映射**：使用了外部中断的引脚是否支持 EXTI
- [ ] **DMA 映射**：所选外设实例及其功能（TX/RX 等）是否有可用 DMA request，且不与其他外设 DMA request 冲突
- [ ] **物理布局**：相同模块的引脚是否物理相邻，便于 PCB 布线

---

## 常见芯片引脚约束速查

### STM32F1 系列（F103/C8T6）

**调试口**：PA13(SWDIO), PA14(SWCLK)

**不可用引脚**：PA11/PA12（USB，无 USB 外设时可用）

**ADC 通道映射**：
```
ADC1_IN0: PA0
ADC1_IN1: PA1
ADC1_IN2: PA2
ADC1_IN3: PA3
ADC1_IN4: PA4
ADC1_IN5: PA5
ADC1_IN6: PA6
ADC1_IN7: PA7
ADC1_IN8: PB0
ADC1_IN9: PB1
```

### STM32F4 系列（F407/Discovery）

**调试口**：PA13(SWDIO), PA14(SWCLK)

**唤醒引脚**：PA0(WKUP)（注意：PA0 不是调试口，但低功耗模式下有特殊功能）

**ADC 通道映射**（12位ADC）：
```
ADC1_IN0: PA0
ADC1_IN1: PA1
...
ADC3_IN10: PF3
ADC3_IN11: PF4
ADC3_IN12: PF5
```

**DAC 输出**：PA4(DAC1_OUT1), PA5(DAC1_OUT2)

### ESP32-WROOM-32

**STRAP 引脚**：
```
GPIO0: 上拉=Boot Flash, 下拉=UART Download
GPIO2: 上拉=正常启动, 下拉=进入Boot
GPIO12: 无上电影响=正常启动, 上电高=影响Flash电压
```

**仅输入引脚**：GPIO34, GPIO35, GPIO36, GPIO39

**DAC 输出引脚**：GPIO25(DAC1), GPIO26(DAC2)（可同时作为普通 GPIO 输入输出）

**默认 UART0**：GPIO1(TX), GPIO3(RX)

### CH32V307VCT6（逐飞）

**调试口**：PA13(SWDIO), PA14(SWCLK)

**不可用引脚**：OSC_IN(PD0), OSC_OUT(PD1)

**ADC 通道**：PA0-PA7, PB0-PB1

---

## 引脚规划输出模板

生成后更新到 `硬件资源表.md` 的"引脚分配表"部分。

### 分配方案 A（默认引脚）

| 引脚 | 功能 | 外设 | 方向 | 优先级 | 备注 |
|------|------|------|------|--------|------|
| PA0 | ADC_IN0 | ADC1 | 输入 | 高 | 电压采样 |
| PA9 | USART1_TX | USART1 | AF推挽 | 高 | 串口调试 |
| PA10 | USART1_RX | USART1 | 浮空输入 | 高 | 串口调试 |
| PB6 | I2C1_SCL | I2C1 | 开漏输出 | 中 | 传感器I2C |
| PB7 | I2C1_SDA | I2C1 | 开漏输出 | 中 | 传感器I2C |
| PC13 | LED | GPIO | 推挽输出 | 低 | 板载LED |

### 冲突警告

```
⚠️ 警告：
- PA9 同时复用了 TIM1_CH2，如需使用 PWM 需选择其他引脚
- PB6/PB7 支持 TIM4_CH1/CH2，若需要 I2C 和 TIM4 同时工作需重映射
```

---

## 辅助工具使用

### grok-search + Document Skills 查询引脚约束

```python
# 示例：查询 STM32F103 的引脚约束
mcp__context7__query-docs(
    libraryId="/stm/stm32f1",
    query="GPIO alternate functions pin mapping PA0 PA9"
)
```

### 厂商配置工具（STM32CubeMX）

1. 选择芯片型号
2. 在 Pinout 视图中分配外设
3. 系统自动检测冲突
4. 导出引脚分配报告

---

## 引脚变更管理

### 变更流程

1. **查阅影响**：检查新引脚是否与其他外设冲突
2. **更新硬件资源表**：在 `硬件资源表.md` 的"引脚分配表"中更新
3. **更新代码**：修改引脚配置代码
4. **测试验证**：确认新引脚功能正常
5. **记录变更**：在硬件资源表底部的"变更记录"中追加

### 变更检查项

- [ ] 调试功能是否受影响
- [ ] STRAP 引脚是否正确配置
- [ ] PCB 是否支持新引脚位置（如有硬件）
- [ ] 中断优先级是否需要调整
- [ ] DMA 通道是否需要重新分配

---

## 实战案例：平衡车引脚规划

### 需求清单

- 2 路电机 PWM（TIM1 互补输出，带死区）
- 1 路 I2C（MPU6050 陀螺仪）
- 1 路 ADC（电池电压）
- 2 路 GPIO（LED 指示灯）
- 1 路串口（调试输出）

### 推荐方案（STM32F103C8T6）

| 引脚 | 功能 | 外设 | 理由 |
|------|------|------|------|
| PA8 | TIM1_CH1 | TIM1 | 电机1 PWM 高侧 |
| PB13 | TIM1_CH1N | TIM1 | 电机1 PWM 低侧 |
| PA9 | TIM1_CH2 | TIM1 | 电机2 PWM 高侧 |
| PB14 | TIM1_CH2N | TIM1 | 电机2 PWM 低侧 |
| PB6 | I2C1_SCL | I2C1 | MPU6050 时钟 |
| PB7 | I2C1_SDA | I2C1 | MPU6050 数据 |
| PA1 | ADC_IN1 | ADC1 | 电池电压 |
| PC13 | LED1 | GPIO | 运行指示 |
| PC14 | LED2 | GPIO | 错误指示 |
| PA2 | USART2_TX | USART2 | 调试输出（避免与 TIM1_CH2 PA9 冲突） |
| PA3 | USART2_RX | USART2 | 调试输入 |

### 验证结果

✅ TIM1 互补输出支持死区插入
✅ I2C 默认引脚，无冲突
✅ 调试口 PA13/PA14 保留
✅ 无 STRAP 引脚问题
✅ 物理分布合理：TIM1 在端口 A/B，I2C 在端口 B

---

## 总结

引脚规划是嵌入式开发的关键环节，规划不当会导致：

- 调试口被占用，无法烧录
- STRAP 引脚配置错误，无法正常启动
- 外设功能冲突，无法同时工作
- PCB 布线困难，成本增加

**原则**：
1. 优先使用默认引脚
2. 保留调试口和烧录口
3. 查阅数据手册，了解引脚约束
4. 生成方案后进行冲突检测
5. 记录到硬件资源表，统一管理
