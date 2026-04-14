# Lab 010 - Power-Sensitive Refactoring

!!! hint "Overview"

    - Future of AI: Masterclass in Agentic Engineering** - Power-Sensitive Refactoring module.
    - In this lab, you will use agents to **identify high-power consumption patterns** in firmware and rewrite them for low-power sleep modes.
    - You will learn to embed power budgets as constraints in your prompts so the agent can reason about energy tradeoffs rather than just correctness.
    - By the end of this lab, you will be able to use an agent to systematically maximize sleep time and minimize active consumption across a firmware codebase.

## Prerequisites

- Completed [Lab 009 - Vibe Coding for Hardware](../009-VibeCodingForHW/README.md)
- Basic understanding of MCU sleep modes (Sleep, Stop, Standby) and peripheral clock gating

## What You Will Learn

- How to include power budgets as explicit agent constraints
- Common high-power firmware anti-patterns an agent can flag
- How to prompt for sleep-mode-compatible refactoring
- How to evaluate power impact before and after using static current estimates

---

## Background

### Power Consumption Hierarchy (STM32H7 Example)

| Mode          | CPU    | Peripherals | Typical Current |
| ------------- | ------ | ----------- | --------------- |
| Run (480 MHz) | Active | Active      | ~300 mA         |
| Sleep         | Halted | Active      | ~100 mA         |
| Stop 0        | Halted | Gated       | ~200 µA         |
| Stop 1        | Halted | Gated       | ~100 µA         |
| Standby       | Halted | Off         | ~2 µA           |

A firmware that spends 80% of its time in a busy-wait loop instead of Stop 1 uses ~1000x more power than necessary. For a battery-powered IoT node, this is the difference between months and hours of runtime.

### Agent Power Reasoning

To reason about power, an agent needs three pieces of context:

1. The current firmware's wakeup triggers (timers, external GPIOs, UART RX)
2. The MCU's sleep mode hierarchy and constraints
3. The minimum acceptable response latency (determines which sleep mode is safe)

---

## Lab Steps

### Step 1 - Power Anti-Pattern Audit

Paste the following firmware snippet:

```c
void AppRun(void) {
    while (1) {
        // Read sensor every 100ms
        uint32_t start = HAL_GetTick();
        SensorData_t data = Sensor_Read();
        ProcessData(&data);
        TransmitBLE(&data);

        // Wait for 100ms interval
        while ((HAL_GetTick() - start) < 100) {
            // busy-wait - CPU running at full clock
        }
    }
}
```

**Agent Prompt:**

```
You are a firmware power optimization agent targeting an STM32H7 IoT sensor node
running at 480 MHz on a 3.7V LiPo battery.

Power budget:
- Active periods: 300 mA maximum
- Idle periods: 50 µA maximum (Stop 1 mode target)
- Battery capacity: 1000 mAh
- Target battery life: 1 year minimum

Analyze the following firmware loop:
1. Identify every location where the CPU is active but doing no useful work
2. Calculate the estimated current draw of the current implementation over a 24-hour period
3. Propose a refactoring that uses HAL_PWR_EnterSTOPMode() during the 100ms idle interval
4. Estimate the new 24-hour current consumption after refactoring
5. Calculate the projected battery life improvement

[paste code here]
```

### Step 2 - Peripheral Clock Gating

Ask the agent to add clock gating to all unused peripherals:

```c
void SystemInit_App(void) {
    // All peripherals enabled by HAL_Init() and never disabled
    MX_USART2_UART_Init();
    MX_SPI1_Init();
    MX_I2C1_Init();
    MX_ADC1_Init();
    MX_TIM6_Init();
    // Application only uses I2C1 and TIM6 in normal operation.
    // USART2, SPI1, ADC1 are only used at startup for configuration.
}
```

**Agent Prompt:**

```
Analyze the following peripheral initialization code.
The comment states which peripherals are only used at startup vs. continuously.

Generate code to:
1. Disable the clock to USART2, SPI1, and ADC1 after their startup configuration is complete
   (use __HAL_RCC_USART2_CLK_DISABLE() etc.)
2. Add a PowerMode_EnterLowPower() function that also gates the unused peripherals
3. Add a PowerMode_ExitLowPower() function that re-enables clocks before next use
4. Add a static assertion that fails at compile time if a peripheral is used after
   its clock has been disabled (using a compile-time flag FW_POWER_LOW_POWER_MODE)
```

### Step 3 - Wakeup Source Configuration

```
Generate the complete configuration for Stop 1 mode on STM32H7 with:
- Wakeup source 1: LPTIM1 running on LSE (32.768 kHz) - fires every 100ms
- Wakeup source 2: EXTI line 0 (PA0, GPIO wakeup, falling edge) for button press
- On wakeup from LPTIM1: call ProcessData() only
- On wakeup from EXTI0: call UserInterrupt_Handler() which re-enters run mode

Generate:
1. LPTIM1_Stop1_Config() - LPTIM initialization for 100ms period
2. EXTI0_WakeupConfig() - GPIO EXTI line configuration
3. PowerManager_EnterStop1() - enters Stop 1, handles wakeup source dispatch
4. Include the PWR->WKUPSRC register check in the wakeup handler to distinguish sources
```

### Step 4 - Before/After Current Budget

Ask the agent to generate a power budget comparison document:

```
Based on the refactoring analysis above, generate a power budget comparison table in Markdown format:

Columns: Firmware State | Before (mA or µA) | After (mA or µA) | Time per 24h (%) | Energy per 24h (mWh)

Rows:
- Sensor reading + processing (active)
- BLE transmission (active)
- 100ms idle interval
- Total 24h energy
- Estimated battery life (1000 mAh at 3.7V = 3700 mWh)
```

---

## Summary

| Skill                     | What You Practiced                                        |
| ------------------------- | --------------------------------------------------------- |
| Power budget constraints  | Embedding mA/µA limits as explicit agent context          |
| Anti-pattern detection    | Agent identifying busy-waits that prevent sleep           |
| Clock gating              | Disabling peripheral clocks for unused hardware           |
| Sleep mode configuration  | LPTIM + EXTI dual-wakeup Stop 1 mode setup                |
| Energy budget calculation | Before/after current estimate with projected battery life |

---

> **Next Lab:** [011 - Tool-Use & Function Calling](../011-ToolUseFunctionCalling/README.md)
