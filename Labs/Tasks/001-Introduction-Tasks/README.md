# Lab 001 - Introduction Tasks

- Hands-on exercises to solidify the difference between AI coding assistants and autonomous agents.
- Each task includes a description, a scenario, a hint, and a collapsible Solution.

#### Table of Contents

- [01. Copilot vs Agent Classification](#01-copilot-vs-agent-classification)
- [02. Firmware Task Mapping](#02-firmware-task-mapping)
- [03. Missing Context Identification](#03-missing-context-identification)
- [04. Agent Scope Definition](#04-agent-scope-definition)

---

#### 01. Copilot vs Agent Classification

Read the following five AI tool descriptions and classify each as **Copilot** or **Agent**. Justify each answer with one sentence referencing the definition from Lab 001.

1. A tool that suggests the next line of code based on cursor position.
2. A system that accepts the task "initialize SPI1 in master mode", reads the reference manual PDF, writes code, runs the compiler, reads the error, and retries until the build passes.
3. A chat window where you ask "what does this function do?" and receive an explanation.
4. A pipeline that monitors serial port output, detects hard faults, queries the LLM for a diagnosis, and posts the result to a Slack channel.
5. A VS Code extension that offers "Accept", "Reject", or "Next" for a suggested code block.

#### Scenario:

◦ A team is evaluating AI tooling for their embedded firmware project.
◦ They need to distinguish which tools require human-in-the-loop oversight at every step vs. which can operate autonomously.

**Hint:** Review the Copilot vs. Agent comparison table in Lab 001.

<details markdown>
<summary>Solution</summary>

| #   | Classification | Justification                                                                                 |
| --- | -------------- | --------------------------------------------------------------------------------------------- |
| 1   | **Copilot**    | Triggered by cursor position, single-shot, no external tools used.                            |
| 2   | **Agent**      | Accepts a high-level task, uses multiple tools (PDF, compiler), iterates until goal achieved. |
| 3   | **Copilot**    | Single-shot question→answer with no external action or iteration.                             |
| 4   | **Agent**      | Multi-step autonomous loop: monitors → detects → queries LLM → acts (posts to Slack).         |
| 5   | **Copilot**    | Reactive UI component; the human decides at every step.                                       |

</details>

---

#### 02. Firmware Task Mapping

Map each of the following firmware tasks to the **agent dimension** it primarily exercises (registers, timing, power states, or tool use). Then state which agent action is required.

1. "Set the baud rate for USART2 to 115200 at 84 MHz APB1 clock."
2. "Refactor the ADC polling loop to run entirely in DMA mode so the CPU can enter Stop mode between conversions."
3. "Detect when the I2C bus hangs and automatically reset the peripheral."
4. "Run the ARM cross-compiler on the generated C file and return any errors."

#### Scenario:

◦ Before designing an agent, a firmware architect needs to understand which hardware dimension drives each task.
◦ Misidentifying the dimension leads to agents that lack the right context or tools.

**Hint:** The four dimensions are: registers, timing, power states, tool use.

<details markdown>
<summary>Solution</summary>

| Task | Primary Dimension | Required Agent Action                                                                          |
| ---- | ----------------- | ---------------------------------------------------------------------------------------------- |
| 1    | **Registers**     | Compute BRR value from clock/baud math; write to USART2->BRR register.                         |
| 2    | **Power States**  | Identify DMA channel config; refactor code; verify STOP-mode wake-up source.                   |
| 3    | **Timing**        | Detect timeout condition (SCL stuck); issue I2C bus reset within SDA stretch window.           |
| 4    | **Tool Use**      | Call `arm-none-eabi-gcc` as an external tool; capture stdout/stderr; return structured result. |

</details>

---

#### 03. Missing Context Identification

Run the following prompt exactly as written (no extra context added) in your agent of choice, then record every piece of hardware context the agent identifies as missing.

```
You are a firmware agent.
Initialize the ADC peripheral to sample channel 3 at 1 kHz.
Before generating any code, list every hardware constraint you need that was NOT provided in this prompt.
```

After running the prompt, answer:

1. How many missing constraints did the agent identify?
2. Which constraint do you consider the most critical for generating correct code?
3. What does the agent's self-awareness about missing constraints tell you about its design quality?

#### Scenario:

◦ A junior engineer submits an underspecified task to an agent.
◦ A well-designed agent should pause and enumerate missing information rather than guessing.

**Hint:** Expected missing info includes: MCU family, APB bus clock, ADC resolution, trigger source, DMA or interrupt vs. polling.

<details markdown>
<summary>Solution</summary>

**Expected agent output (representative sample):**

```
Missing hardware constraints:
1. MCU family / ADC peripheral name (STM32, PSoC, NXP, etc.)
2. APB2 clock frequency (determines ADCCLK prescaler for 1 kHz sample rate)
3. ADC resolution (8, 10, 12, or 16-bit)
4. Conversion trigger source (software trigger, timer TRGOx, external pin)
5. Transfer mode: polling, interrupt (EOC), or DMA circular
6. Reference voltage (VREF+ source and value)
7. Sampling time per channel (ADC_SMP register value)
8. Single-channel or scan mode
```

**Answers:**

1. A well-behaved agent should identify **6-10** missing constraints.
2. The most critical is the **APB2 clock frequency** - without it, the agent cannot compute the prescaler needed to achieve exactly 1 kHz.
3. An agent that lists missing constraints before generating code has a **planning/reasoning loop** (not a pure token predictor), which is a hallmark of a well-structured agent.

</details>

---

#### 04. Agent Scope Definition

Write a task description for a firmware agent that satisfies all five criteria of a well-scoped agent task:

1. Has a **specific, measurable goal**
2. Provides **minimum necessary hardware context**
3. Names the **tools** the agent is allowed to use
4. Defines **success criteria** (how do we know the agent is done?)
5. Defines **failure criteria** (when should the agent stop and ask for help?)

The task should be: _"Initialize the FreeRTOS watchdog task that resets the MCU if the system becomes unresponsive."_

#### Scenario:

◦ Poorly scoped agent tasks lead to agents that generate plausible but incorrect code, or that loop indefinitely.
◦ A well-scoped task gives the agent everything it needs to succeed - and to know when to stop.

**Hint:** Reference the four agent dimensions and consider what "unresponsive" means in an RTOS context.

<details markdown>
<summary>Solution</summary>

```
Task: Implement IWDG watchdog integration for FreeRTOS on STM32F4.

Goal (measurable):
  Generate a FreeRTOS task WatchdogTask() that:
  - Initializes the IWDG with a 2-second timeout (LSI ~32 kHz, IWDG prescaler /256)
  - Runs at priority 5 (highest) every 500 ms and refreshes the IWDG via HAL_IWDG_Refresh()
  - A missing refresh within 2 seconds triggers an MCU reset

Hardware context provided:
  - MCU: STM32F407, LSI clock ~32 kHz, HAL library available
  - FreeRTOS version: 10.4.x, configTICK_RATE_HZ = 1000
  - IWDG register base: 0x40003000, prescaler options: /4, /8, /16, /32, /64, /128, /256

Allowed tools:
  compile_firmware(src, target="cortex-m4")
  read_peripheral_register(peripheral="IWDG", register="PR/RLR/SR")

Success criteria:
  - Code compiles for cortex-m4 with zero errors and zero warnings
  - IWDG_PR and IWDG_RLR values are mathematically correct for a 2-second timeout
  - WatchdogTask() calls HAL_IWDG_Refresh() at least once per 500 ms according to code inspection

Failure criteria (ask for help if):
  - Compilation fails after 3 iterations
  - The IWDG timeout math cannot be resolved with given LSI frequency
  - HAL_IWDG_Init() is not available in the provided header context
```

</details>

---

> **Back to [Tasks Index](../index.md)** | **Next: [002 Tasks](../002-HardwareAwareAgents-Tasks/README.md)**
