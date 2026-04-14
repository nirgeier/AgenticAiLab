# Lab 014 - Constraint-Based Generation Tasks

- Exercises covering constraint prompt blocks, constrained driver generation, linker section attributes, code-size measurement, and CI pipeline assembly.
- Each task includes a description, scenario, hint, and collapsible Solution.

#### Table of Contents

- [01. Write a Constraint Prompt Block](#01-write-a-constraint-prompt-block)
- [02. Generate a Constrained UART Driver](#02-generate-a-constrained-uart-driver)
- [03. Add Linker Section Attributes](#03-add-linker-section-attributes)
- [04. Measure and Enforce Code Size](#04-measure-and-enforce-code-size)
- [05. Assemble the Full Constraint Pipeline](#05-assemble-the-full-constraint-pipeline)

---

#### 01. Write a Constraint Prompt Block

Translate the following hardware engineering requirements into a structured **constraint prompt block** that an agent can enforce during code generation.

**Engineering requirements (informal):**

> "The ISR handler for TIM2 must complete in under 500 nanoseconds at 168 MHz. It can't use any FreeRTOS calls, can't allocate memory, and must be in ITCM RAM to minimize flash fetch latency. The output compare register must be updated before the function returns."

Your constraint block must use the format from Lab 014:

```
**Constraints:**
- TIMING: ...
- MEMORY: ...
- PLACEMENT: ...
- CORRECTNESS: ...
```

#### Scenario:

◦ The firmware team has strict real-time requirements for a motor control ISR.
◦ Informal requirements passed to an agent without structure produce code that meets some constraints but silently violates others.

**Hint:** Express timing constraints as cycle counts (500 ns × 168 MHz = 84 cycles max). Express placement as a specific section name.

<details markdown>
<summary>Solution</summary>

```
**Task:** Implement TIM2_IRQHandler for a motor control output compare interrupt.

**Constraints:**

- TIMING:
  - Total ISR execution time MUST NOT EXCEED 84 CPU cycles at 168 MHz (= 500 ns).
  - The output compare register (TIM2->CCR1) MUST be updated on every ISR entry before the function returns.
  - No loops with variable iteration count.

- MEMORY:
  - NO dynamic memory allocation (no malloc, no new, no stack-allocated arrays > 8 bytes).
  - NO FreeRTOS API calls of any kind (no xQueueSendFromISR, no portYIELD_FROM_ISR, etc.).
  - All state must be in module-level static variables, not the stack.

- PLACEMENT:
  - The function MUST be placed in ITCM RAM using __attribute__((section(".itcm_text"))).
  - All lookup tables accessed in the ISR must also be in ITCM or CCM RAM.

- CORRECTNESS:
  - The interrupt pending flag (TIM2->SR & TIM_SR_CC1IF) MUST be cleared at the START of the handler.
  - The next compare value written to TIM2->CCR1 must be computed from the current counter value (TIM2->CNT) plus a fixed period offset.
  - The function must NOT call any function that is not also in ITCM RAM (risk of instruction cache miss).
```

</details>

---

#### 02. Generate a Constrained UART Driver

Using the constraint prompt block format from Task 01, write a **complete constrained prompt** for a UART TX driver and verify the agent output satisfies all constraints.

**Required constraints:**

- **ROM budget**: Total compiled size of the UART driver module ≤ 512 bytes of Flash
- **Stack budget**: No function in the driver may use more than 32 bytes of stack
- **No dynamic memory** anywhere
- **Interrupt-driven TX** only (no polling, no DMA for this version)
- **Circular buffer size**: Exactly 128 bytes, `static` allocation
- **Re-entrancy**: `UART_Send()` must be safe to call from multiple FreeRTOS tasks using a mutex
- **Baud rate**: Fixed at compile time via a `UART_BAUD_RATE` macro; no runtime configurability

Show the generated driver and the verification checklist.

#### Scenario:

◦ A safety-critical embedded system requires every driver module to meet ROM + stack budgets.
◦ Without explicit constraints in the prompt, agents often generate feature-rich but over-large implementations.

**Hint:** Verify ROM budget with `arm-none-eabi-size -A uart_driver.o`. Verify stack depth with `arm-none-eabi-gcc -fstack-usage`.

<details markdown>
<summary>Solution</summary>

**Constraint prompt:**

```
Task: Implement a UART1 TX driver for STM32F4 (APB2=84 MHz).

Constraints:
- ROM BUDGET: Entire module (uart_driver.c compiled to uart_driver.o) must be ≤ 512 bytes of .text section.
- STACK BUDGET: No function may allocate more than 32 bytes of stack (verify with -fstack-usage).
- MEMORY: No dynamic allocation (no malloc/calloc/new). TX buffer: exactly 128 bytes, static uint8_t array.
- INTERRUPT: TX must use USART1_TXE interrupt only. No polling loops. No DMA.
- THREAD SAFETY: UART_Send() must acquire a FreeRTOS mutex before writing to the ring buffer.
- CONFIGURATION: Baud rate set at compile time via UART_BAUD_RATE macro. No runtime baud change function.
- NO PRINTF/SPRINTF anywhere in the module.
```

**Generated driver (representative):**

```c
#include "stm32f4xx.h"
#include "FreeRTOS.h"
#include "semphr.h"
#include <stdint.h>
#include <string.h>

#ifndef UART_BAUD_RATE
#define UART_BAUD_RATE 115200UL
#endif
#define APB2_CLOCK_HZ  84000000UL
#define BUF_SIZE       128U

static uint8_t  tx_buf[BUF_SIZE];
static uint32_t tx_head = 0;
static uint32_t tx_tail = 0;
static SemaphoreHandle_t s_mutex;

void UART1_Init(void)
{
    s_mutex = xSemaphoreCreateMutex();
    configASSERT(s_mutex != NULL);

    RCC->APB2ENR  |= RCC_APB2ENR_USART1EN;
    USART1->BRR    = (uint16_t)(APB2_CLOCK_HZ / UART_BAUD_RATE);
    USART1->CR1    = USART_CR1_TE | USART_CR1_UE;
    NVIC_SetPriority(USART1_IRQn, 6);
    NVIC_EnableIRQ(USART1_IRQn);
}

void UART_Send(const uint8_t *data, uint16_t len)
{
    xSemaphoreTake(s_mutex, portMAX_DELAY);
    for (uint16_t i = 0; i < len; i++) {
        uint32_t next = (tx_head + 1U) % BUF_SIZE;
        while (next == tx_tail) { taskYIELD(); }
        tx_buf[tx_head] = data[i];
        tx_head = next;
    }
    USART1->CR1 |= USART_CR1_TXEIE;
    xSemaphoreGive(s_mutex);
}

void USART1_IRQHandler(void)
{
    if (USART1->SR & USART_SR_TXE) {
        if (tx_tail != tx_head) {
            USART1->DR = tx_buf[tx_tail];
            tx_tail = (tx_tail + 1U) % BUF_SIZE;
        } else {
            USART1->CR1 &= ~USART_CR1_TXEIE;
        }
    }
}
```

**Verification checklist:**

```bash
# 1. Compile
arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -Wall -Os -fstack-usage \
  -c uart_driver.c -o uart_driver.o

# 2. Check ROM budget (≤512 bytes .text)
arm-none-eabi-size -A uart_driver.o | grep '\.text'
# Expected: .text < 512 bytes with -Os optimization

# 3. Check stack usage (≤32 bytes per function)
cat uart_driver.su
# Expected: each function ≤ 32 bytes

# 4. Verify no malloc/printf
grep -n 'malloc\|calloc\|printf\|sprintf' uart_driver.c
# Expected: no output
```

</details>

---

#### 03. Add Linker Section Attributes

Add appropriate GCC `__attribute__((section(...)))` attributes to place the following functions in the correct linker sections. Also write the linker script fragment that defines the sections.

| Function                                                         | Requirement                                                        |
| ---------------------------------------------------------------- | ------------------------------------------------------------------ |
| `void TIM2_IRQHandler(void)`                                     | Must run from ITCM RAM (fast fetch)                                |
| `const uint16_t sin_table[256]`                                  | Read-only LUT, must be in DTCMRAM                                  |
| `void SpinDelay_Cycles(uint32_t n)`                              | Calibrated busy-wait, must not be optimized or inlined             |
| `void OTA_WriteFlash(uint32_t addr, uint8_t *buf, uint32_t len)` | Must be in a special `.ota_flash` section for easy firmware update |

#### Scenario:

◦ Fine-grained placement control is a production firmware requirement for MCUs with separated TCM and SRAM bus matrices.
◦ Generating these attributes correctly requires knowledge of both GCC and the target's memory layout.

**Hint:** ITCM mapped at `0x00000000`; DTCM at `0x20000000` on STM32F7/H7. Use `__attribute__((section(".itcm_text")))` + linker SECTIONS block.

<details markdown>
<summary>Solution</summary>

**C source attributes:**

```c
/* TIM2 ISR in ITCM RAM */
__attribute__((section(".itcm_text")))
void TIM2_IRQHandler(void)
{
    TIM2->SR  = ~TIM_SR_CC1IF;
    TIM2->CCR1 = TIM2->CNT + PERIOD_TICKS;
}

/* Sine LUT in DTCM RAM (fast read by DSP/FPU) */
__attribute__((section(".dtcm_data")))
const uint16_t sin_table[256] = { /* 256 entries */ };

/* Calibrated spin delay — no optimization, no inlining */
__attribute__((noinline, optimize("O0"), section(".text")))
void SpinDelay_Cycles(uint32_t n)
{
    while (n--) { __asm volatile("nop"); }
}

/* OTA flash writer in dedicated section */
__attribute__((section(".ota_flash")))
void OTA_WriteFlash(uint32_t addr, uint8_t *buf, uint32_t len)
{
    HAL_FLASH_Unlock();
    for (uint32_t i = 0; i < len; i++) {
        HAL_FLASH_Program(FLASH_TYPEPROGRAM_BYTE, addr + i, buf[i]);
    }
    HAL_FLASH_Lock();
}
```

**Linker script fragment (STM32F7 example):**

```ld
MEMORY
{
  FLASH    (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K
  ITCM     (rwx) : ORIGIN = 0x00000000, LENGTH = 16K
  DTCM     (rw)  : ORIGIN = 0x20000000, LENGTH = 128K
  SRAM     (rw)  : ORIGIN = 0x20020000, LENGTH = 384K
}

SECTIONS
{
  .itcm_text :
  {
    *(.itcm_text)
    *(.itcm_text.*)
  } > ITCM AT > FLASH          /* LMA in Flash, VMA in ITCM */

  .dtcm_data :
  {
    *(.dtcm_data)
    *(.dtcm_data.*)
  } > DTCM AT > FLASH

  .ota_flash :
  {
    . = ALIGN(4);
    _ota_start = .;
    *(.ota_flash)
    _ota_end = .;
  } > FLASH
}
```

</details>

---

#### 04. Measure and Enforce Code Size

Write a Python CI script `check_code_size.py` that:

1. Compiles a set of `.c` source files with `arm-none-eabi-gcc -Os`
2. Runs `arm-none-eabi-size -A` to extract the size of each section (`.text`, `.data`, `.bss`) per object file
3. Checks each file against a budget file `size_budget.json`
4. Fails (exit code 1) and prints a table of over-budget files if any budget is exceeded
5. Passes (exit code 0) and prints a summary table if all are within budget

**Example `size_budget.json`:**

```json
{
  "uart_driver.o": { ".text": 512, ".data": 0, ".bss": 132 },
  "spi_driver.o": { ".text": 768, ".data": 0, ".bss": 8 },
  "i2c_driver.o": { ".text": 640, ".data": 0, ".bss": 16 }
}
```

#### Scenario:

◦ ROM budget enforcement is a hard requirement in safety-critical firmware.
◦ The CI gate runs this script on every PR; if any new code exceeds its budget, the build fails.

**Hint:** `arm-none-eabi-size -A` outputs one section per line. Parse with regex: `^(\.\w+)\s+(\d+)`.

<details markdown>
<summary>Solution</summary>

```python
#!/usr/bin/env python3
import json
import pathlib
import re
import subprocess
import sys
import tempfile

SIZE_PATTERN = re.compile(r'^(\.\w+)\s+(\d+)', re.MULTILINE)

def compile_file(src: pathlib.Path, obj_dir: pathlib.Path) -> pathlib.Path:
    obj = obj_dir / (src.stem + ".o")
    result = subprocess.run(
        ["arm-none-eabi-gcc", "-mcpu=cortex-m4", "-mthumb", "-Os", "-c",
         str(src), "-o", str(obj)],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        raise RuntimeError(f"Compile failed for {src.name}:\n{result.stderr}")
    return obj

def measure_size(obj: pathlib.Path) -> dict:
    result = subprocess.run(
        ["arm-none-eabi-size", "-A", str(obj)],
        capture_output=True, text=True
    )
    sizes = {}
    for match in SIZE_PATTERN.finditer(result.stdout):
        sizes[match.group(1)] = int(match.group(2))
    return sizes

def check_code_size(src_dir: str, budget_file: str) -> int:
    budget = json.loads(pathlib.Path(budget_file).read_text())
    sources = list(pathlib.Path(src_dir).glob("*.c"))

    violations = []
    summary    = []

    with tempfile.TemporaryDirectory() as tmp:
        obj_dir = pathlib.Path(tmp)
        for src in sources:
            try:
                obj   = compile_file(src, obj_dir)
                sizes = measure_size(obj)
                obj_name = obj.name

                if obj_name in budget:
                    for section, limit in budget[obj_name].items():
                        actual = sizes.get(section, 0)
                        status = "PASS" if actual <= limit else "FAIL"
                        summary.append((obj_name, section, actual, limit, status))
                        if status == "FAIL":
                            violations.append((obj_name, section, actual, limit))
            except RuntimeError as e:
                print(str(e))
                return 1

    print(f"\n{'File':<20} {'Section':<10} {'Actual':>8} {'Budget':>8} {'Status'}")
    print("-" * 60)
    for row in summary:
        print(f"{row[0]:<20} {row[1]:<10} {row[2]:>8} {row[3]:>8} {row[4]}")

    if violations:
        print(f"\nFAILED: {len(violations)} section(s) over budget.")
        return 1

    print(f"\nPASSED: All {len(summary)} sections within budget.")
    return 0

if __name__ == "__main__":
    src_dir     = sys.argv[1] if len(sys.argv) > 1 else "src/"
    budget_file = sys.argv[2] if len(sys.argv) > 2 else "size_budget.json"
    sys.exit(check_code_size(src_dir, budget_file))
```

</details>

---

#### 05. Assemble the Full Constraint Pipeline

Combine all elements from Tasks 01–04 into a complete **constraint-based generation pipeline** that:

1. Accepts a hardware requirement description (plain text)
2. Generates a structured constraint prompt block (Task 01 pattern)
3. Generates the firmware code satisfying the constraints (Task 02 pattern)
4. Checks linker section attributes are correct (Task 03 pattern — verify `__attribute__((section` is present)
5. Compiles and measures code size (Task 04 pattern)
6. Reports: constraint violations, code size vs. budget, overall PASS/FAIL

Test the full pipeline with the requirement: _"Implement a DMA-based SPI receive driver for STM32F4 with a 256-byte RX buffer, ROM budget of 640 bytes, no FreeRTOS dependencies, and all ISR code in ITCM RAM."_

#### Scenario:

◦ This end-to-end pipeline is the production-ready version of constraint-based generation.
◦ It moves from informal human intent → formal constraints → verified code in a single automated workflow.

**Hint:** Structure as a LangGraph graph: `parse_requirements` → `generate_constraints` → `generate_code` → `verify_sections` → `check_size` → `report`.

<details markdown>
<summary>Solution</summary>

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict
from typing import Optional, List
import re, json, tempfile, subprocess, pathlib, os

llm = ChatOpenAI(model="gpt-4o", temperature=0)

class PipelineState(TypedDict):
    requirement:  str
    constraints:  Optional[str]
    code:         Optional[str]
    section_ok:   Optional[bool]
    size_result:  Optional[dict]
    violations:   List[str]
    status:       str

# Node 1: Parse requirements → constraint block
def generate_constraints(state: PipelineState) -> PipelineState:
    resp = llm.invoke([
        SystemMessage(content="Turn this hardware requirement into a structured constraint block with sections: TIMING, MEMORY, PLACEMENT, CORRECTNESS, ROM_BUDGET."),
        HumanMessage(content=state["requirement"])
    ])
    return {**state, "constraints": resp.content}

# Node 2: Generate code from constraints
def generate_code(state: PipelineState) -> PipelineState:
    resp = llm.invoke([
        SystemMessage(content="Generate C firmware code satisfying ALL listed constraints. Return only C code, no markdown."),
        HumanMessage(content=state["constraints"])
    ])
    return {**state, "code": resp.content}

# Node 3: Verify section attributes
def verify_sections(state: PipelineState) -> PipelineState:
    code = state["code"] or ""
    # Check that ISR has ITCM section attribute (per constraint)
    isr_ok = bool(re.search(r'__attribute__\s*\(\s*\(section\s*\(\s*"\.itcm_text"\s*\)\s*\)\s*\)', code))
    violations = list(state["violations"])
    if not isr_ok:
        violations.append("ISR missing __attribute__((section(\".itcm_text\")))")
    return {**state, "section_ok": isr_ok, "violations": violations}

# Node 4: Compile and check size
def check_size(state: PipelineState) -> PipelineState:
    code = state["code"] or ""
    violations = list(state["violations"])
    with tempfile.TemporaryDirectory() as tmp:
        src = pathlib.Path(tmp) / "driver.c"
        obj = pathlib.Path(tmp) / "driver.o"
        src.write_text(code)

        compile_result = subprocess.run(
            ["arm-none-eabi-gcc", "-mcpu=cortex-m4", "-mthumb", "-Os", "-c", str(src), "-o", str(obj)],
            capture_output=True, text=True
        )
        if compile_result.returncode != 0:
            violations.append(f"Compile failed: {compile_result.stderr[:200]}")
            return {**state, "violations": violations, "status": "FAIL"}

        size_result = subprocess.run(
            ["arm-none-eabi-size", "-A", str(obj)],
            capture_output=True, text=True
        ).stdout

        text_match = re.search(r'^\.text\s+(\d+)', size_result, re.MULTILINE)
        text_size  = int(text_match.group(1)) if text_match else 0

        # Budget from requirement: 640 bytes
        if text_size > 640:
            violations.append(f".text section {text_size} bytes exceeds 640-byte ROM budget")

        return {**state,
                "size_result": {"text": text_size},
                "violations": violations}

# Node 5: Report
def report(state: PipelineState) -> PipelineState:
    ok = len(state["violations"]) == 0
    print("\n=== CONSTRAINT PIPELINE REPORT ===")
    print(f"Sections valid: {state.get('section_ok')}")
    print(f"ROM (.text):    {state.get('size_result', {}).get('text', 'N/A')} bytes (budget: 640)")
    if state["violations"]:
        print("\nVIOLATIONS:")
        for v in state["violations"]: print(f"  - {v}")
    print(f"\nOverall: {'PASS' if ok else 'FAIL'}")
    return {**state, "status": "PASS" if ok else "FAIL"}

# Wire graph
builder = StateGraph(PipelineState)
builder.add_node("generate_constraints", generate_constraints)
builder.add_node("generate_code",        generate_code)
builder.add_node("verify_sections",      verify_sections)
builder.add_node("check_size",           check_size)
builder.add_node("report",               report)

builder.add_edge(START,                "generate_constraints")
builder.add_edge("generate_constraints","generate_code")
builder.add_edge("generate_code",       "verify_sections")
builder.add_edge("verify_sections",     "check_size")
builder.add_edge("check_size",          "report")
builder.add_edge("report",              END)

graph = builder.compile()

if __name__ == "__main__":
    result = graph.invoke(PipelineState(
        requirement=(
            "Implement a DMA-based SPI receive driver for STM32F4 with a 256-byte RX buffer, "
            "ROM budget of 640 bytes, no FreeRTOS dependencies, and all ISR code in ITCM RAM."
        ),
        constraints=None, code=None, section_ok=None,
        size_result=None, violations=[], status="running"
    ))
    print("\nFinal status:", result["status"])
```

</details>

---

> **Back to [Tasks Index](../index.md)** | **Next: [Final Project](../FinalProject/README.md)**
