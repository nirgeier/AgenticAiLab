# Lab 012 - ReAct Patterns Tasks

- Exercises covering ReAct step formatting, action parsing, hard fault debugging, RTOS issue diagnosis, and multi-agent coordination.
- Each task includes a description, scenario, hint, and collapsible Solution.

#### Table of Contents

- [01. Format a ReAct Trace by Hand](#01-format-a-react-trace-by-hand)
- [02. Parse a ReAct Action Line](#02-parse-a-react-action-line)
- [03. Debug a Hard Fault with ReAct](#03-debug-a-hard-fault-with-react)
- [04. Solve an RTOS Sync Bug with ReAct](#04-solve-an-rtos-sync-bug-with-react)
- [05. Coordinate Two ReAct Agents](#05-coordinate-two-react-agents)

---

#### 01. Format a ReAct Trace by Hand

Write a complete, well-formed **ReAct trace** (Thought → Action → Observation → Thought → …) for the following task:

**Task:** _"Find out what baud rate USART2 is currently configured for, given that the system clock is 84 MHz and APB1 prescaler is 2."_

**Available actions (tools):**

- `ReadRegister(peripheral, register)` → returns hex value
- `CalculateBaudRate(APB_clock_Hz, BRR_value)` → returns baud rate in bps

Your trace must contain at least 3 Thought-Action-Observation steps and end with a `Final Answer:` line.

#### Scenario:

◦ Creating ReAct traces by hand is the best way to understand the format before implementing it in code.
◦ A well-formed trace has `Thought:` (reasoning), `Action:` (tool call and argument), `Observation:` (tool result), cycling until the Final Answer.

**Hint:** APB1 clock = 84 MHz / 2 = 42 MHz. USART_BRR formula: baud = f_apb / BRR_value.

<details markdown>
<summary>Solution</summary>

```
Task: Find out what baud rate USART2 is currently configured for,
      given system clock = 84 MHz and APB1 prescaler = 2.

Thought: To find the baud rate, I need the BRR register value and the APB1 clock.
         The APB1 clock = 84 MHz / 2 = 42 MHz.
         I'll start by reading the USART2 BRR register.

Action: ReadRegister("USART2", "BRR")

Observation: 0x016D

Thought: BRR = 0x016D = 365 decimal.
         Now I need to calculate the baud rate from APB1 clock and BRR.
         Formula: baud = f_APB1 / BRR_value

Action: CalculateBaudRate(42000000, 365)

Observation: 115068 bps (≈ 115200 bps with ±1% tolerance from integer rounding)

Thought: The calculated baud rate is ~115068 bps, which rounds to the standard
         115200 baud rate. The ±1% error is within UART tolerance.
         I now have enough information to give the final answer.

Action: ReadRegister("USART2", "CR1")

Observation: 0x200C

Thought: CR1 = 0x200C → bits: TE=1, RE=1, UE=1. USART2 is enabled, transmitting and receiving.
         M bit (bit 12) = 0 → 8-bit word length. PCE bit = 0 → no parity.
         This confirms 8N1 configuration at ~115200 baud.

Final Answer: USART2 is configured for 8N1 at approximately 115200 baud.
              APB1 clock = 42 MHz, BRR = 0x016D (365), calculated baud = 115068 bps.
```

</details>

---

#### 02. Parse a ReAct Action Line

Write a Python function `parse_react_action(line: str)` that extracts the **tool name** and **arguments** from a ReAct action line.

The function must handle all these formats:

```
Action: ReadRegister("USART2", "BRR")
Action: CalculateBaudRate(42000000, 365)
Action: compile_and_check(source_code="int main(){}", target="cortex-m4")
Action: Search("STM32F4 USART BRR formula")
```

Return a tuple `(tool_name: str, args: list, kwargs: dict)`.

#### Scenario:

◦ The ReAct executor needs to parse the LLM's `Action:` line and dispatch the correct tool.
◦ A robust parser handles both positional and keyword arguments.

**Hint:** Use `ast.literal_eval` for safe argument parsing. Extract the tool name with a simple regex before the `(`.

<details markdown>
<summary>Solution</summary>

```python
import re
import ast
from typing import Tuple, List, Dict, Any

def parse_react_action(line: str) -> Tuple[str, List[Any], Dict[str, Any]]:
    """
    Parse a ReAct action line like:
      "Action: ToolName(arg1, arg2, key=val)"
    Returns: (tool_name, positional_args, keyword_args)
    """
    # Strip the "Action: " prefix (case-insensitive)
    line = re.sub(r'^[Aa]ction:\s*', '', line.strip())

    # Extract tool name (everything before the first '(')
    match = re.match(r'^(\w+)\((.*)\)$', line, re.DOTALL)
    if not match:
        raise ValueError(f"Cannot parse action: {line!r}")

    tool_name = match.group(1)
    args_str  = match.group(2).strip()

    if not args_str:
        return tool_name, [], {}

    # Wrap in a tuple so ast.literal_eval can parse comma-separated args
    try:
        parsed = ast.literal_eval(f"({args_str},)")
    except Exception:
        # Fallback: treat entire args_str as a single string argument
        parsed = (args_str,)

    positional = []
    keyword    = {}
    # ast.literal_eval loses keyword argument names - use a more robust parse
    # for keyword args, try a manual key=value split:
    for part in re.split(r',\s*(?=\w+=)', args_str):
        if '=' in part.split('(')[0]:  # keyword arg
            k, _, v = part.partition('=')
            keyword[k.strip()] = ast.literal_eval(v.strip())
        else:
            try:
                positional.append(ast.literal_eval(part.strip()))
            except Exception:
                positional.append(part.strip())

    return tool_name, positional, keyword

# Tests
cases = [
    'Action: ReadRegister("USART2", "BRR")',
    'Action: CalculateBaudRate(42000000, 365)',
    'Action: compile_and_check(source_code="int main(){}", target="cortex-m4")',
    'Action: Search("STM32F4 USART BRR formula")',
]
for c in cases:
    name, args, kwargs = parse_react_action(c)
    print(f"{name}  args={args}  kwargs={kwargs}")
```

</details>

---

#### 03. Debug a Hard Fault with ReAct

Write a complete ReAct system prompt for a hard fault debugging agent, then run it against the following scenario.

**System prompt requirements:**

1. Defines available tools: `ReadRegister`, `DisassembleAt(address)`, `LookupSymbol(address)`
2. Instructs the agent to reason step by step using Thought/Action/Observation
3. Instructs the agent to stop after finding root cause and providing a fix

**Scenario to run:**

```
Task: Debug this hard fault.
PC  = 0x08004A22
LR  = 0x08004810
CFSR = 0x00020000   (BFSR[1] = PRECISERR)
BFAR = 0x40005400
```

Include the simulated tool responses and the full expected ReAct trace.

#### Scenario:

◦ Hard fault debugging is a domain where ReAct shines: the agent alternates between reading memory, looking up symbols, and reasoning about what went wrong.
◦ A straight completion would guess; ReAct forces tool-verified analysis.

**Hint:** BFAR = 0x40005400 is in the APB1 peripheral address range (if valid). Check if it's a known STM32F4 peripheral. `PRECISERR` means the faulting address is known and is in BFAR.

<details markdown>
<summary>Solution</summary>

**System prompt:**

```
You are a firmware hard fault debugging agent.
You have the following tools available:
  ReadRegister(peripheral, register) - reads a register value as hex
  DisassembleAt(address) - returns the assembly instruction at a given address
  LookupSymbol(address) - returns the C symbol name at or near an address

Use Thought/Action/Observation steps to analyze the hard fault.
On each Thought, reason about what the data means and what to check next.
When you have identified the root cause and a concrete fix, write "Final Answer:" and stop.
```

**Expected ReAct trace:**

```
Task: Debug hard fault. PC=0x08004A22, CFSR=0x00020000 (PRECISERR), BFAR=0x40005400

Thought: CFSR bit 17 (BFSR[1]) is PRECISERR - a precise bus fault. The faulting address
         is in BFAR = 0x40005400. This is a valid, known address in BFAR.
         0x40005400 is in the STM32F4 APB1 range (0x40000000-0x40007FFF).
         Let me look up what peripheral is at 0x40005400.

Action: LookupSymbol(0x40005400)

Observation: Address 0x40005400 = I2C1_BASE (I2C1 peripheral registers)

Thought: The fault occurred during a write to the I2C1 base address.
         Reading from/writing to a disabled peripheral's registers can cause a bus fault
         if the peripheral clock is not enabled. Let me check the PC to see what code ran.

Action: DisassembleAt(0x08004A22)

Observation: 0x08004A22: STR r1, [r0, #0]   ; r0 = 0x40005400

Thought: The faulting instruction is STR to 0x40005400 (I2C1->CR1 or first register).
         This is a write to I2C1 before its clock is enabled.
         Let me verify by reading the I2C1 clock enable bit in RCC_APB1ENR.

Action: ReadRegister("RCC", "APB1ENR")

Observation: 0x00000000

Thought: RCC_APB1ENR = 0x00000000. Bit 21 (I2C1EN) is 0 - I2C1 clock is NOT enabled.
         Writing to a gated peripheral register causes a PRECISERR bus fault on STM32F4.
         The code at 0x08004A22 attempts to write to I2C1->CR1 before enabling the clock.
         Let me confirm the calling function.

Action: LookupSymbol(0x08004810)

Observation: 0x08004810 = I2C1_Init() in i2c.c line 45

Final Answer:
Root cause: I2C1_Init() at address 0x08004810 writes to I2C1->CR1 (0x40005400)
before enabling the I2C1 peripheral clock (RCC_APB1ENR bit 21 = 0).
Writing to a gated peripheral register causes a PRECISERR hard fault on STM32F4.

Fix: Add RCC->APB1ENR |= (1U << 21) before the first I2C1 register access in I2C1_Init().
```

</details>

---

#### 04. Solve an RTOS Sync Bug with ReAct

The following FreeRTOS scenario produces a deadlock. Write a ReAct trace (without running any tools - reason through the logic step by step) that identifies the deadlock, explains the exact sequence of events, and proposes a fix.

```c
SemaphoreHandle_t mutex_A, mutex_B;

void TaskHigh(void *p)          // priority 3
{
    for (;;) {
        xSemaphoreTake(mutex_A, portMAX_DELAY);  // Step 1
        xSemaphoreTake(mutex_B, portMAX_DELAY);  // Step 3
        // ... critical section ...
        xSemaphoreGive(mutex_B);
        xSemaphoreGive(mutex_A);
        vTaskDelay(10);
    }
}

void TaskLow(void *p)           // priority 2
{
    for (;;) {
        xSemaphoreTake(mutex_B, portMAX_DELAY);  // Step 2
        xSemaphoreTake(mutex_A, portMAX_DELAY);  // Step 4
        // ... critical section ...
        xSemaphoreGive(mutex_A);
        xSemaphoreGive(mutex_B);
        vTaskDelay(10);
    }
}
```

#### Scenario:

◦ Classic mutex inversion / circular wait deadlock - one of the most common RTOS bugs.
◦ The ReAct reasoning trace demonstrates how the agent walks through the interleaved execution steps to find the cycle.

**Hint:** Number the lock acquisition steps and draw the resource allocation graph mentally. Look for a cycle.

<details markdown>
<summary>Solution</summary>

```
Task: Identify the deadlock in the two-task mutex scenario.

Thought: I need to trace the interleaved execution to find a circular wait.
         TaskHigh takes mutex_A first, then mutex_B.
         TaskLow takes mutex_B first, then mutex_A.
         This is a classic "opposite lock order" pattern - high deadlock risk.

Action: AnalyzeExecution(trace=[
  "TaskHigh starts: takes mutex_A (succeeds)",
  "Preempted (or RR tick)",
  "TaskLow starts: takes mutex_B (succeeds)",
  "TaskLow tries mutex_A → BLOCKED (held by TaskHigh)",
  "TaskHigh resumes: tries mutex_B → BLOCKED (held by TaskLow)"
])

Observation: (Reasoning step - no external tool needed)
  After the sequence:
  1. TaskHigh holds mutex_A, waiting for mutex_B
  2. TaskLow holds mutex_B, waiting for mutex_A
  → Circular wait: TaskHigh → mutex_B → TaskLow → mutex_A → TaskHigh

Thought: The resource allocation graph has a cycle:
  TaskHigh → waits_for → mutex_B → held_by → TaskLow → waits_for → mutex_A → held_by → TaskHigh
  This is the definition of a deadlock. Both tasks will wait forever.

Action: CheckFreeRTOSPriority(TaskHigh=3, TaskLow=2)

Observation: TaskHigh has higher priority (3 > 2). FreeRTOS uses preemptive scheduling.
             Priority inversion is also possible here, but the deadlock occurs regardless of priority.

Thought: The root cause is lock-order inconsistency:
  TaskHigh acquires (A, B); TaskLow acquires (B, A).
  The fix is to enforce a global lock ordering - both tasks must acquire mutexes
  in the SAME order: always A before B.

Final Answer:
Deadlock caused by inconsistent mutex acquisition order.
  - TaskHigh: takes A, then B
  - TaskLow: takes B, then A
  → Circular dependency when both tasks interleave at the correct moment.

Fix: Enforce a canonical lock order (always acquire mutex_A before mutex_B in ALL tasks):

  void TaskLow(void *p) {
      for (;;) {
          xSemaphoreTake(mutex_A, portMAX_DELAY);  // Same order as TaskHigh
          xSemaphoreTake(mutex_B, portMAX_DELAY);  // ← fixed
          // ...
          xSemaphoreGive(mutex_B);
          xSemaphoreGive(mutex_A);
          vTaskDelay(10);
      }
  }
```

</details>

---

#### 05. Coordinate Two ReAct Agents

Design a **multi-agent system** where two ReAct agents collaborate to solve a firmware task:

- **Agent A (Hardware Inspector)**: Has tools `ReadRegister`, `DisassembleAt` - no code generation
- **Agent B (Code Fixer)**: Has tools `compile_and_check`, `write_firmware_file` - no register access

Task: _"USART2 is configured but not transmitting. Find the misconfigured register and generate a corrected initialization function."_

Specify:

1. The message passing protocol (what Agent A sends to Agent B)
2. The complete ReAct trace for each agent
3. How Agent B consumes Agent A's findings

#### Scenario:

◦ Separating concerns between agents mirrors production multi-engineer workflows: one reads the hardware state, another fixes the code.
◦ This design prevents Agent B from reading registers directly (which it shouldn't need to do) and keeps Agent A from generating code (which it might do incorrectly).

**Hint:** Agent A outputs a structured "Findings Report" that Agent B receives as its task context.

<details markdown>
<summary>Solution</summary>

**Message Passing Protocol:**

Agent A produces a `FindingsReport` JSON at the end of its ReAct trace:

```json
{
  "peripheral": "USART2",
  "issue": "UE bit (bit 13) in CR1 is NOT set - USART is disabled",
  "current_register_values": {
    "CR1": "0x000C",
    "BRR": "0x016D"
  },
  "correct_cr1_value": "0x200C",
  "required_changes": ["Set CR1 bit 13 (UE) to enable USART2"]
}
```

**Agent A ReAct Trace:**

```
Task: USART2 is configured but not transmitting. Find the misconfigured register.

Thought: I'll read the key USART2 registers to diagnose the issue.

Action: ReadRegister("USART2", "CR1")
Observation: 0x000C

Thought: CR1=0x000C. Bit 3 (TE)=1 (transmitter enabled), bit 2 (RE)=1 (receiver enabled).
         But bit 13 (UE = USART Enable) = 0. The USART is NOT enabled regardless of TE/RE.

Action: ReadRegister("USART2", "BRR")
Observation: 0x016D

Thought: BRR=0x016D → correct for 115200 baud at 42 MHz APB1.
         The only problem is UE=0 in CR1. Setting bit 13 will enable the USART.

Final Answer (Findings Report):
{
  "peripheral": "USART2",
  "issue": "UE bit (bit 13) in CR1 is NOT set - USART is disabled",
  "current_register_values": {"CR1": "0x000C", "BRR": "0x016D"},
  "correct_cr1_value": "0x200C",
  "required_changes": ["Set USART2 CR1 bit 13 (UE = USART Enable)"]
}
```

**Agent B ReAct Trace:**

```
Task: Fix USART2 initialization based on findings report:
  - Issue: UE bit (bit 13) not set in CR1
  - Required change: Set CR1 = 0x200C to enable USART2

Thought: I need to generate a corrected USART2_Init() that sets CR1 with UE=1.
         Current code likely writes 0x000C to CR1. I need to add bit 13 → 0x200C.

Action: compile_and_check(source_code="
  #include <stdint.h>
  #define USART2_CR1  (*(volatile uint32_t*)0x4000440C)
  #define USART2_BRR  (*(volatile uint32_t*)0x40004408)
  void USART2_Init(void) {
      USART2_BRR = 0x016D;
      USART2_CR1 = (1U<<13)|(1U<<3)|(1U<<2);  /* UE|TE|RE */
  }", target="cortex-m4")

Observation: {"success": true, "errors": [], "warnings": []}

Thought: Code compiles cleanly. All three bits (UE, TE, RE) are set correctly.
         Now I'll save the corrected file.

Action: write_firmware_file("usart2_init.c", <corrected_content>)
Observation: "File written: usart2_init.c"

Final Answer: Generated corrected USART2_Init() with CR1=0x200C (UE+TE+RE).
              Root cause was missing UE bit - USART was configured but not enabled.
```

</details>

---

> **Back to [Tasks Index](../index.md)** | **Next: [013 Tasks](../013-AgenticDocumentation-Tasks/README.md)**
