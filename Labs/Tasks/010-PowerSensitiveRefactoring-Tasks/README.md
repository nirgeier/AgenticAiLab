# Lab 010 - Power-Sensitive Refactoring Tasks

- Exercises covering power pattern identification, energy math, sleep-mode refactoring, and power verification.
- Each task includes a description, scenario, hint, and collapsible Solution.

#### Table of Contents

- [01. Identify High-Power Patterns](#01-identify-high-power-patterns)
- [02. Calculate the Active vs Sleep Power Budget](#02-calculate-the-active-vs-sleep-power-budget)
- [03. Refactor to Low-Power Sleep Mode](#03-refactor-to-low-power-sleep-mode)
- [04. Verify the Refactoring with a Power Profiling Prompt](#04-verify-the-refactoring-with-a-power-profiling-prompt)

---

#### 01. Identify High-Power Patterns

Analyze the following STM32L4-series firmware main loop. List every high-power anti-pattern and classify it as **Critical**, **Major**, or **Minor**.

```c
// Product: IoT sensor node, battery-powered (AA × 2 = 3.0V, 2700 mAh target)
// Expected battery life: 2 years. Actual: 3 weeks.

int main(void)
{
    HAL_Init();
    SystemClock_Config_80MHz();        // Line 5: Run mode, 80 MHz, MSI + PLL

    // Peripheral initialization
    MX_USART2_UART_Init();             // Always-on UART at 115200
    MX_SPI1_Init();                    // SPI to flash - always on
    MX_ADC1_Init();                    // ADC to temperature sensor

    while (1)
    {
        // 1. Check sensor every 100ms
        uint16_t temp = ADC_Read_Blocking();  // Line 15: polling loop
        if (temp != last_temp) {
            BLE_Send_Notification(temp);      // Line 17: full BLE TX at max power
        }
        last_temp = temp;

        // 2. Log to flash every second
        if (++tick_counter % 10 == 0) {
            Flash_Write_Log(temp);            // Line 22: SPI flash write
        }

        HAL_Delay(100);                       // Line 25: busy wait, CPU running
    }
}
```

#### Scenario:

◦ Battery lifetime is 3 weeks instead of 2 years - a 35× gap.
◦ The power audit identifies which patterns are responsible, prioritized by impact.

**Hint:** On STM32L4: Run 80 MHz ≈ 15-20 mA; Stop2 mode ≈ 2 µA. Reducing active time from 100% to 1% with Stop2 is the single largest win.

<details markdown>
<summary>Solution</summary>

| #   | Pattern                                         | Location          | Classification | Estimated Impact                                                                                                             |
| --- | ----------------------------------------------- | ----------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| 1   | **CPU at 80 MHz in full Run mode continuously** | Line 5 + while(1) | **Critical**   | 15-20 mA constant. With Stop2 at 2 µA and 1% duty cycle, average drops to ~202 µA.                                           |
| 2   | **HAL_Delay(100) busy-wait**                    | Line 25           | **Critical**   | CPU active 100% of the time; this is the reason Run mode never exits. Replace with Stop2 + RTC wakeup.                       |
| 3   | **ADC polling loop (blocking)**                 | Line 15           | **Major**      | ADC takes up to 20 µs; acceptable if burst-mode, not polling full-time. Use ADC single conversion triggered by wakeup event. |
| 4   | **BLE TX at maximum power on every change**     | Line 17           | **Major**      | TX current peak can be 10-20 mA for 1-5 ms. Batch notifications and reduce TX power.                                         |
| 5   | **SPI flash write every 1 second**              | Line 22           | **Major**      | Flash write: 5-10 mW for 10-100 ms. Reduce to once per 10 minutes or on threshold events.                                    |
| 6   | **Always-on UART at 115200**                    | Line 7            | **Minor**      | UART peripheral clock always enabled. Disable between transactions (HAL_UART_DeInit / re-init).                              |
| 7   | **SPI peripheral always powered**               | Line 8            | **Minor**      | SPI clock always running. Disable SPI clock via RCC when flash is idle.                                                      |

</details>

---

#### 02. Calculate the Active vs Sleep Power Budget

Using the following power parameters for STM32L4 running at 3.0V, calculate the **average current** and **battery lifetime** for both the inefficient and optimized duty cycles.

**Power parameters:**
| Mode | Current |
|------|---------|
| Run mode 80 MHz | 18 mA |
| Stop 2 mode | 2 µA |
| BLE TX burst (5 ms every 10 s) | 15 mA during burst |
| ADC single conversion (1 ms) | 2.5 mA during conversion |

**Duty cycles:**

- **Inefficient**: CPU Run mode 100% of the time
- **Optimized**: CPU in Stop2 99% of the time; active 1% (10 ms wake every 1 s)
  - During 10 ms wake: ADC (1 ms @ 2.5 mA) + CPU processing (9 ms @ 18 mA)
  - BLE TX burst: 5 ms @ 15 mA every 10 s

**Battery capacity:** 2700 mAh @ 3.0 V

#### Scenario:

◦ Management needs a slide with concrete numbers to justify the re-spin cost.
◦ The agent calculates and formats the comparison table.

**Hint:** Average current = Σ(Iₙ × duty_cycle_n). Lifetime (h) = Capacity (mAh) / Average_current (mA).

<details markdown>
<summary>Solution</summary>

**Inefficient average current:**

```
I_avg = 18 mA × 100% = 18 mA
Lifetime = 2700 mAh / 18 mA = 150 h ≈ 6.25 days
```

**Optimized average current:**

```
Per 10-second window:
  ADC:        1 ms @ 2.5 mA  → 2.5 mA × (1/10000)    = 0.00025 mA
  CPU active: 9 ms @ 18 mA   → 18 mA × (9/10000)      = 0.0162  mA
  BLE TX:     5 ms @ 15 mA   → 15 mA × (5/10000)      = 0.0075  mA
  Stop2:      9985 ms @ 2µA  → 0.002 mA × (9985/10000)= 0.001997 mA
              ─────────────────────────────────────────────────────
  I_avg ≈ 0.00025 + 0.0162 + 0.0075 + 0.001997 ≈ 0.0259 mA

Lifetime = 2700 mAh / 0.0259 mA ≈ 104,247 h ≈ 4,344 days ≈ 11.9 years
```

**Comparison:**

| Metric               | Inefficient | Optimized       | Improvement     |
| -------------------- | ----------- | --------------- | --------------- |
| Average current      | 18 mA       | 0.026 mA        | 692× reduction  |
| Battery lifetime     | 6.25 days   | ~11.9 years     | **695× longer** |
| Meets 2-year target? | No          | Yes (6× margin) | ✓               |

</details>

---

#### 03. Refactor to Low-Power Sleep Mode

Refactor the main loop from Task 01 to use STM32L4 **Stop 2 mode** with an **RTC wakeup timer** instead of `HAL_Delay(100)`.

Requirements:

1. Configure RTC wakeup for every 1 second (replaces the 100 ms HAL_Delay loop - we reduce sampling to 1 Hz)
2. Enter Stop2 mode with `HAL_PWREx_EnterSTOP2Mode(PWR_STOPENTRY_WFI)`
3. On wakeup: re-initialize system clock, take ADC reading, send BLE if changed
4. Disable SPI and UART clocks while in Stop2; re-enable on wakeup

#### Scenario:

◦ The single most impactful refactoring is replacing `HAL_Delay` with Stop2 + RTC wakeup.
◦ This moves the device from 18 mA constant to < 30 µA average.

**Hint:** After exiting Stop2, the CPU clock returns to MSI 4 MHz. Call `SystemClock_Config_80MHz()` again before any peripheral use.

<details markdown>
<summary>Solution</summary>

```c
#include "stm32l4xx_hal.h"

static RTC_HandleTypeDef hrtc;

void RTC_Wakeup_Init(uint32_t seconds)
{
    __HAL_RCC_RTC_ENABLE();
    hrtc.Instance = RTC;
    hrtc.Init.HourFormat     = RTC_HOURFORMAT_24;
    hrtc.Init.AsynchPrediv   = 127;
    hrtc.Init.SynchPrediv    = 255;
    HAL_RTC_Init(&hrtc);

    /* Configure RTC wakeup: RTCCLK/16 = 2048 Hz for LSI ~32 kHz */
    HAL_RTCEx_SetWakeUpTimer_IT(
        &hrtc,
        (uint32_t)(seconds * 2048U),
        RTC_WAKEUPCLOCK_RTCCLK_DIV16
    );
    HAL_NVIC_SetPriority(RTC_WKUP_IRQn, 0, 0);
    HAL_NVIC_EnableIRQ(RTC_WKUP_IRQn);
}

static void DisablePeripheralClocks(void)
{
    __HAL_RCC_SPI1_CLK_DISABLE();
    __HAL_RCC_USART2_CLK_DISABLE();
}

static void EnablePeripheralClocks(void)
{
    __HAL_RCC_SPI1_CLK_ENABLE();
    __HAL_RCC_USART2_CLK_ENABLE();
}

int main(void)
{
    HAL_Init();
    SystemClock_Config_80MHz();
    MX_USART2_UART_Init();
    MX_SPI1_Init();
    MX_ADC1_Init();
    RTC_Wakeup_Init(1);  /* Wakeup every 1 second */

    while (1)
    {
        /* Active window: ADC + BLE */
        uint16_t temp = ADC_Read_Single();  /* Single conversion, not blocking loop */
        if (temp != last_temp) {
            BLE_Send_Notification(temp);
            last_temp = temp;
        }

        /* Flash log every 10 wakeups (~10 seconds) */
        if (++wakeup_count % 10 == 0) {
            Flash_Write_Log(temp);
        }

        /* Prepare for Stop2 */
        DisablePeripheralClocks();
        HAL_SuspendTick();               /* Stop systick to avoid spurious wakeups */

        /* Enter Stop2 mode (wake on RTC interrupt) */
        HAL_PWREx_EnterSTOP2Mode(PWR_STOPENTRY_WFI);

        /* --- Execution resumes here after RTC wakeup --- */
        HAL_ResumeTick();
        SystemClock_Config_80MHz();      /* Restore 80 MHz after Stop2 */
        EnablePeripheralClocks();
    }
}

void RTC_WKUP_IRQHandler(void)
{
    HAL_RTCEx_WakeUpTimerIRQHandler(&hrtc);
}
```

</details>

---

#### 04. Verify the Refactoring with a Power Profiling Prompt

Write a system prompt for an agent that:

1. Accepts the refactored firmware source code from Task 03
2. Analyzes every `HAL_Delay()` call and flags any that remain (should be zero after refactoring)
3. Identifies any peripheral that is still clocked during Stop2 (look for missing `__HAL_RCC_x_CLK_DISABLE()`)
4. Checks that `SystemClock_Config_80MHz()` is called after every `HAL_PWREx_EnterSTOP2Mode()` exit
5. Outputs a pass/fail report with specific line references

#### Scenario:

◦ After a refactoring, the team runs this verification prompt as a CI step (similar to a static analyzer).
◦ It catches regressions before the code is flashed to hardware.

**Hint:** The agent's analysis is structural (pattern matching in source), not dynamic. Ask it to output structured JSON so it can be parsed by CI.

<details markdown>
<summary>Solution</summary>

**Power verification system prompt:**

```
You are a firmware power consumption auditor.

Analyze the provided C source code and check for these power management violations:

RULE 1 - NO HAL_Delay IN RUN MODE:
  Find all calls to HAL_Delay(). Each one indicates CPU busy-waiting in Run mode.
  Flag every occurrence with line number and duration.

RULE 2 - PERIPHERAL CLOCK DISABLE BEFORE STOP2:
  Find all calls to HAL_PWREx_EnterSTOP2Mode().
  For each call, check that the preceding code (within 20 lines) contains
  __HAL_RCC_*_CLK_DISABLE() for USART, SPI, and ADC peripherals.
  Flag any missing clock-disable calls.

RULE 3 - CLOCK RESTORE AFTER STOP2:
  Find all calls to HAL_PWREx_EnterSTOP2Mode().
  For each call, check that SystemClock_Config() or SystemClock_Config_XXMHz()
  is called in the code following the Stop2 call (within 20 lines after the next
  executable statement).
  Flag any missing clock-restore calls.

RULE 4 - RTC WAKEUP CONFIGURED:
  Verify that HAL_RTCEx_SetWakeUpTimer_IT() is called during initialization.
  If not found, flag as missing.

Output format - JSON:
{
  "summary": {
    "hal_delay_violations": <count>,
    "clock_disable_violations": <count>,
    "clock_restore_violations": <count>,
    "rtc_wakeup_configured": <true|false>,
    "overall_status": "PASS" | "FAIL"
  },
  "violations": [
    {
      "rule": "RULE 1",
      "line": <number>,
      "description": "<explanation>"
    }
  ]
}

overall_status is PASS only if all violation counts are 0 and rtc_wakeup_configured is true.
```

**Python CI integration:**

````python
import json, sys
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage
import pathlib

POWER_AUDIT_SYSTEM = """... (prompt above) ..."""

llm = ChatOpenAI(model="gpt-4o", temperature=0)

def run_power_audit(source_file: str) -> int:
    source = pathlib.Path(source_file).read_text()
    result = llm.invoke([
        SystemMessage(content=POWER_AUDIT_SYSTEM),
        HumanMessage(content=f"```c\n{source}\n```")
    ]).content

    try:
        report = json.loads(result)
    except json.JSONDecodeError:
        print("AUDIT ERROR: Agent did not return valid JSON.")
        return 1

    print(json.dumps(report, indent=2))
    return 0 if report["summary"]["overall_status"] == "PASS" else 1

if __name__ == "__main__":
    sys.exit(run_power_audit(sys.argv[1]))
````

</details>

---

> **Back to [Tasks Index](../index.md)** | **Next: [011 Tasks](../011-ToolUseFunctionCalling-Tasks/README.md)**
