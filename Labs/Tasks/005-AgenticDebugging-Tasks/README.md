# Lab 005 - Agentic Debugging Tasks

- Exercises covering hard fault analysis, RTOS race conditions, interrupt latency calculation, and automated debugging reports.
- Each task includes a description, scenario, hint, and collapsible Solution.

#### Table of Contents

- [01. Decode a Hard Fault Dump](#01-decode-a-hard-fault-dump)
- [02. Identify a FreeRTOS Race Condition](#02-identify-a-freertos-race-condition)
- [03. Calculate Interrupt Latency Budget](#03-calculate-interrupt-latency-budget)
- [04. Build an Anomaly Detector Prompt](#04-build-an-anomaly-detector-prompt)
- [05. Generate a Debugging Report](#05-generate-a-debugging-report)

---

#### 01. Decode a Hard Fault Dump

Feed the following hard fault register dump to your agent and ask it to identify the root cause.

```
HardFault at:
  PC  = 0x0800_14AC
  LR  = 0x0800_132F
  PSP = 0x2001_FA00
  CFSR = 0x0000_0200   (MMFSR=0x00, BFSR=0x02, UFSR=0x0000)
  BFAR = 0x4001_1004   (Valid: BFARVALID bit NOT set)

Disassembly fragment:
  0x0800_14A8: LDR  r0, [r1]     ; r1 = 0x4001_1004
  0x0800_14AC: STR  r0, [r2]     ; r2 = 0x2001_FA00
  0x0800_14B0: BX   lr
```

**Prompt to use:**

```
Analyze the following ARM Cortex-M4 hard fault dump. Identify the fault type,
the faulting instruction, the root cause, and recommend a fix.
```

#### Scenario:

◦ A device in the field is crashing with a hard fault. The team has only the register dump from the serial console.
◦ The agent must determine fault type, explain the likely cause, and suggest a specific code fix.

**Hint:** CFSR bit [9] in BFSR (bit 9 of CFSR) = `PRECISERR` — precise bus fault on data access.

<details markdown>
<summary>Solution</summary>

**Fault analysis:**

| Field  | Value         | Meaning                                                         |
| ------ | ------------- | --------------------------------------------------------------- |
| `CFSR` | `0x0000_0200` | `BFSR[1]=1` → `PRECISERR`: precise bus fault during data access |
| `BFAR` | `0x4001_1004` | The faulting bus address (but `BFARVALID` is NOT set!)          |
| `PC`   | `0x0800_14AC` | Faulting instruction: `STR r0, [r2]`                            |

**Root Cause:**
The `STR r0, [r2]` instruction at `0x0800_14AC` attempts to write to `r2 = 0x2001_FA00`. On this particular STM32 configuration, `0x2001_FA00` is **below the bottom of the main stack** (PSP = same address, stack overflow). The processor raises a `PRECISERR` bus fault because the write address is in an unmapped or MPU-protected region.

Note: `BFARVALID=0` means the `BFAR` value is a leftover from a previous fault — it should be ignored.

**Fix:**

1. Increase the stack size in the linker script (e.g., from 0x400 to 0x800 bytes).
2. Enable the MPU to detect stack overflow with a guard region.
3. Add a stack watermark check in FreeRTOS (`uxTaskGetStackHighWaterMark()`).

```c
/* In linker script */
_Min_Stack_Size = 0x800;   /* was 0x400 */
```

</details>

---

#### 02. Identify a FreeRTOS Race Condition

Analyze the following two FreeRTOS tasks. Identify every race condition and shared resource conflict.

```c
// Shared resource accessed by both tasks
static volatile uint32_t g_SensorData[8];
static volatile uint8_t  g_DataReady = 0;

// Task 1: Sensor acquisition (priority 3, runs every 10 ms)
void SensorTask(void *pvParam)
{
    for (;;) {
        // Acquire new data
        for (int i = 0; i < 8; i++) {
            g_SensorData[i] = ADC_GetSample(i);  // takes ~5 µs each
        }
        g_DataReady = 1;
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

// Task 2: Processing task (priority 2, runs every 5 ms)
void ProcessingTask(void *pvParam)
{
    for (;;) {
        if (g_DataReady) {
            uint32_t sum = 0;
            for (int i = 0; i < 8; i++) {
                sum += g_SensorData[i];           // read shared array
            }
            g_DataReady = 0;
            Process(sum);
        }
        vTaskDelay(pdMS_TO_TICKS(5));
    }
}
```

#### Scenario:

◦ The system is producing sporadic incorrect processing results — roughly 1 in 50 iterations.
◦ The agent must identify the exact race condition and recommend a fix.

**Hint:** Consider what happens if FreeRTOS preempts `SensorTask` mid-array-fill and `ProcessingTask` runs immediately after.

<details markdown>
<summary>Solution</summary>

**Race condition identified:**

**Problem 1: Torn read/write on `g_SensorData`**
`SensorTask` writes 8 elements over ~40 µs (8 × 5 µs). FreeRTOS (preemptive) can preempt it after any element. If `ProcessingTask` runs while `SensorTask` is halfway through the array, `ProcessingTask` reads a **partially updated buffer** — some old values, some new values.

**Problem 2: Non-atomic read-modify-write on `g_DataReady`**
Reading `g_DataReady` and clearing it in `ProcessingTask` is not atomic. If both tasks race on this flag, the data could be processed twice or missed.

**Fix — use a FreeRTOS mutex or task notification:**

```c
// Replace shared globals with a mutex-protected pattern
static SemaphoreHandle_t xDataMutex;
static uint32_t g_SensorData[8];
static uint8_t  g_DataReady = 0;

// Sensor task: take mutex, update ALL 8 values atomically, release
void SensorTask(void *pvParam) {
    for (;;) {
        xSemaphoreTake(xDataMutex, portMAX_DELAY);
        for (int i = 0; i < 8; i++) {
            g_SensorData[i] = ADC_GetSample(i);
        }
        g_DataReady = 1;
        xSemaphoreGive(xDataMutex);
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

// Processing task: take mutex, read all 8 values atomically, release
void ProcessingTask(void *pvParam) {
    for (;;) {
        xSemaphoreTake(xDataMutex, portMAX_DELAY);
        if (g_DataReady) {
            uint32_t sum = 0;
            for (int i = 0; i < 8; i++) sum += g_SensorData[i];
            g_DataReady = 0;
            xSemaphoreGive(xDataMutex);
            Process(sum);
        } else {
            xSemaphoreGive(xDataMutex);
        }
        vTaskDelay(pdMS_TO_TICKS(5));
    }
}
```

</details>

---

#### 03. Calculate Interrupt Latency Budget

Calculate the **worst-case interrupt latency** given the following FreeRTOS configuration and hardware parameters.

**System parameters:**

- `configTICK_RATE_HZ = 1000` (1 ms tick)
- `configMAX_SYSCALL_INTERRUPT_PRIORITY = 5` (NVIC priority 5 = highest maskable)
- `taskENTER_CRITICAL()` holds all IRQs at priority ≤ 5 for up to `T_critical`
- Longest critical section in codebase: **T_critical = 12 µs**
- Processor: STM32F4 @ 168 MHz
- NVIC exception entry overhead: 12 CPU cycles

The interrupt under analysis is at NVIC priority 4 (above `SYSCALL` limit, cannot be masked by FreeRTOS).

**Questions:**

1. Is this interrupt maskable by `taskENTER_CRITICAL()`?
2. What is the interrupt entry overhead in µs?
3. What is the worst-case latency from signal asserted to first IRQ handler instruction?

#### Scenario:

◦ A motor control interrupt at priority 4 must respond within 5 µs of a fault signal.
◦ The team needs to verify this is achievable with the current FreeRTOS config.

**Hint:** On Cortex-M, lower numeric priority = higher urgency. `SYSCALL` masks priorities numerically ≥ the limit, not <.

<details markdown>
<summary>Solution</summary>

**Q1: Is priority-4 interrupt maskable by taskENTER_CRITICAL()?**
No. `taskENTER_CRITICAL()` sets `BASEPRI = 5`, which blocks IRQs at priority **5 and above** (numerically ≥ 5). Priority 4 is numerically **lower** (higher urgency) and is **never masked** by FreeRTOS primitives.

**Q2: Interrupt entry overhead:**

```
Cycles = 12 (NVIC exception entry)
Time   = 12 / 168,000,000 Hz = 0.071 µs ≈ 71 ns
```

**Q3: Worst-case latency (priority 4, un-maskable):**
Since the IRQ cannot be masked by FreeRTOS:

```
Worst-case latency = instruction boundary alignment + NVIC entry
                   = 1 instruction @ 168 MHz + 71 ns
                   = ~6 ns + 71 ns ≈ 77 ns
```

This is well within the 5 µs requirement.

**If the IRQ were at priority 6 (maskable), worst case would be:**

```
T_critical = 12 µs + NVIC entry = 12 µs + 0.071 µs ≈ 12.07 µs → MISSED the 5 µs deadline
```

**Conclusion:** Priority 4 is the correct choice for time-critical motor fault detection on this system.

</details>

---

#### 04. Build an Anomaly Detector Prompt

Write a complete system prompt for a firmware debugging agent that:

1. Accepts a FreeRTOS task log (task name, state, stack watermark) as input
2. Identifies tasks with stack watermark < 64 bytes (potential overflow risk)
3. Identifies tasks in the `Blocked` state for more than 200 ms
4. Outputs a structured JSON report with: `task_name`, `issue_type`, `severity` (low/medium/high), `recommendation`

#### Scenario:

◦ A continuous monitoring agent polls the RTOS task list every 5 seconds.
◦ Any anomaly is flagged immediately rather than waiting for a hard fault.

**Hint:** The agent needs to reason about two conditions: stack watermark threshold and blocked-state duration.

<details markdown>
<summary>Solution</summary>

**System prompt:**

```
You are a FreeRTOS anomaly detection agent embedded in a firmware monitoring pipeline.

Input: A JSON array of running FreeRTOS tasks with fields:
  - "task_name": string
  - "state": "Running" | "Ready" | "Blocked" | "Suspended" | "Deleted"
  - "priority": integer (0=idle, higher=more urgent)
  - "stack_watermark_bytes": integer (remaining free stack bytes)
  - "blocked_ms": integer (milliseconds in Blocked state, 0 if not blocked)

Analysis rules (apply ALL of them):
  1. STACK OVERFLOW RISK:
     - HIGH severity:   stack_watermark_bytes < 32
     - MEDIUM severity: stack_watermark_bytes < 64
     - Recommendation: "Increase stack allocation by at least {2× current usage}"

  2. BLOCKED TIMEOUT:
     - HIGH severity:   state == "Blocked" AND blocked_ms > 500
     - MEDIUM severity: state == "Blocked" AND blocked_ms > 200
     - Recommendation: "Investigate blocking resource (mutex, semaphore, queue full)"

  3. IDLE STACK STARVATION:
     - HIGH severity:   task with priority == 0 (Idle) has stack_watermark_bytes < 128
     - Recommendation: "FreeRTOS idle task stack is dangerously low; increase configMINIMAL_STACK_SIZE"

Output: A JSON array like:
[
  {
    "task_name": "SensorTask",
    "issue_type": "STACK_OVERFLOW_RISK",
    "severity": "high",
    "details": "Stack watermark: 28 bytes remaining",
    "recommendation": "Increase SensorTask stack from 512 to 1024 bytes"
  }
]

If no issues are found, output: []
Do not add explanation text outside the JSON array.
```

</details>

---

#### 05. Generate a Debugging Report

Using the agent you built in Lab 005 (or simulating the workflow), run the complete debugging pipeline on the following combined scenario and produce a final Markdown debugging report.

**Input scenario:**

- Hard fault registers: PC=0x08002F14, CFSR=0x00010000 (UNDEFINSTR in UFSR)
- FreeRTOS task log shows: `CommTask` blocked for 800 ms, `DataTask` with 20 bytes stack watermark
- Serial log: `[ERROR] I2C timeout after 100ms on address 0x52`

The report must include sections: **Executive Summary**, **Fault Analysis**, **RTOS State Analysis**, **Peripheral Log Analysis**, **Root Cause**, **Recommended Actions**.

#### Scenario:

◦ The team needs a shareable debugging report to send to the hardware vendor.
◦ The agent aggregates all debug signals into a structured, professional document.

**Hint:** UFSR bit 16 in CFSR → `UNDEFINSTR`: the processor tried to execute an undefined instruction (e.g., Thumb-2 instruction on a Cortex-M0, or corrupted code flash).

<details markdown>
<summary>Solution</summary>

**Prompt to generate the report:**

```
You are a senior firmware debugging agent.
Analyze the following data sources and produce a Markdown debugging report with sections:
Executive Summary, Fault Analysis, RTOS State Analysis, Peripheral Log Analysis,
Root Cause, and Recommended Actions.

Data:
1. Hard fault: PC=0x08002F14, CFSR=0x00010000
2. RTOS task log: CommTask BLOCKED for 800ms; DataTask stack_watermark=20 bytes
3. Serial log: [ERROR] I2C timeout after 100ms on address 0x52
```

**Expected report structure:**

```markdown
# Firmware Debugging Report — [Date]

## Executive Summary

Three critical issues detected: undefined instruction hard fault, I2C communication
timeout, and near-critical stack overflow in DataTask. Probable root cause is I2C
bus lock-up causing CommTask deadlock, which starved DataTask of CPU time,
leading to unchecked stack growth and eventual execution of corrupted code.

## Fault Analysis

- **Fault Type**: UNDEFINSTR (CFSR[16] = 1)
- **Faulting PC**: 0x08002F14
- **Meaning**: The processor attempted to execute an undefined instruction.
  This typically indicates either: (a) execution jumped to an invalid address,
  (b) corrupted code in flash, or (c) Thumb/ARM mode mismatch.

## RTOS State Analysis

| Task     | State         | Stack Watermark | Severity                   |
| -------- | ------------- | --------------- | -------------------------- |
| CommTask | Blocked 800ms | N/A             | HIGH — probable deadlock   |
| DataTask | Running       | 20 bytes        | HIGH — near stack overflow |

## Peripheral Log Analysis

- I2C bus to address 0x52 timed out after 100ms.
- Address 0x52 is likely the external sensor connected via I2C1.
- 100ms timeout with no recovery → bus is locked (SCL or SDA stuck).

## Root Cause

1. I2C peripheral at address 0x52 became unresponsive (power glitch or EMI noise).
2. CommTask waited indefinitely for the I2C mutex → blocked 800ms.
3. With CommTask blocked, DataTask missed scheduled updates, accumulated data in
   its stack, and overflowed into the code region or function pointer table.
4. The corrupted return address caused execution to jump to 0x08002F14, which
   contained an invalid opcode → UNDEFINSTR hard fault.

## Recommended Actions

1. **Immediate**: Implement I2C bus recovery (9 SCL pulses + STOP condition).
2. **Short-term**: Add I2C timeout in CommTask (`HAL_I2C_Master_Transmit` with 50ms timeout).
3. **Short-term**: Increase DataTask stack allocation from current size to at least 512 bytes.
4. **Long-term**: Add FaultHandler that logs PC, LR, and PSP to non-volatile storage before reset.
5. **Long-term**: Enable MPU with a stack guard region to catch overflow before data corruption.
```

</details>

---

> **Back to [Tasks Index](../index.md)** | **Next: [006 Tasks](../006-AgenticFrameworks-Tasks/README.md)**
