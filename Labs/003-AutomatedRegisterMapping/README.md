# Lab 003 - Automated Register Mapping

!!! hint "Overview"

    - Agentic AI for Embedded & RTOS**.
    - In this lab, you will build an agent that reads a hardware datasheet and **automatically generates C header files and initialization code**.
    - You will simulate the process a junior firmware engineer spends hours on - manually transcribing register tables into `#define` macros - and replace it with an agentic pipeline.
    - By the end of this lab, you will have a working register-mapping agent that takes raw register descriptions as input and outputs production-quality C headers.

## Prerequisites

- Completed [Lab 002 - Hardware-Aware Agent Architecture](../002-HardwareAwareAgents/README.md)
- Basic understanding of C `#define`, `struct`, and bitfield syntax

## What You Will Learn

- How to feed datasheet register tables to an LLM agent
- How to prompt agents to produce structured C header output
- How to validate generated register maps against reference documentation
- How to chain a register-map agent into an initialization-code agent

---

## Background

### The Register Mapping Problem

Every new peripheral integration starts with the same tedious task: open the Reference Manual, locate the register description table, and hand-transcribe dozens of bit-field definitions into C:

```c
// Manually written - error-prone and time-consuming
#define USART_CR1_UE        (1U << 0)   /*!< USART Enable */
#define USART_CR1_UESM      (1U << 1)   /*!< USART Enable in Stop Mode */
#define USART_CR1_RE        (1U << 2)   /*!< Receiver Enable */
#define USART_CR1_TE        (1U << 3)   /*!< Transmitter Enable */
// ... 20 more fields
```

An agent given the raw register description table can generate this in seconds - with **zero transcription errors**.

---

## Lab Steps

### Step 1 - Provide a Register Table to the Agent

Copy the following simulated datasheet excerpt and use it as agent input:

```
Register: UART_CR1 - Control Register 1
Base Address Offset: 0x00
Reset Value: 0x0000 0000

Bit  | Field    | R/W | Description
-----|----------|-----|--------------------------------------------
31:29| Reserved | -   | Must be kept at reset value
28   | RXFFIE   | R/W | Receive FIFO Full Interrupt Enable
27   | TXFEIE   | R/W | Transmit FIFO Empty Interrupt Enable
26   | FIFOEN   | R/W | FIFO Mode Enable
25   | OVER8    | R/W | Oversampling Mode (0=16x, 1=8x)
 5   | RTOIE    | R/W | Receiver Timeout Interrupt Enable
 4   | IDLEIE   | R/W | IDLE Interrupt Enable
 3   | TE       | R/W | Transmitter Enable
 2   | RE       | R/W | Receiver Enable
 1   | UESM     | R/W | USART Enable in Stop Mode
 0   | UE       | R/W | USART Enable
```

**Agent Prompt:**

```
You are a firmware code generation agent. Given the following UART_CR1 register
description, generate a complete C header file section that includes:
1. A #define for the register base offset
2. Individual #define macros for each named bit field (bit position and mask)
3. A C bitfield struct for the register
4. A short Doxygen comment for each field

Use STM32 HAL naming conventions.

[paste register table here]
```

### Step 2 - Validate the Output

Compare the agent's generated header against the following expected output:

```c
/** @defgroup UART_CR1 Control Register 1 */

#define UART_CR1_OFFSET     0x00U

/* Bit position defines */
#define UART_CR1_UE_POS     0U
#define UART_CR1_UE_MSK     (1U << UART_CR1_UE_POS)    /*!< USART Enable */

#define UART_CR1_UESM_POS   1U
#define UART_CR1_UESM_MSK   (1U << UART_CR1_UESM_POS)  /*!< USART Enable in Stop Mode */

#define UART_CR1_RE_POS     2U
#define UART_CR1_RE_MSK     (1U << UART_CR1_RE_POS)    /*!< Receiver Enable */

#define UART_CR1_TE_POS     3U
#define UART_CR1_TE_MSK     (1U << UART_CR1_TE_POS)    /*!< Transmitter Enable */

#define UART_CR1_FIFOEN_POS 26U
#define UART_CR1_FIFOEN_MSK (1U << UART_CR1_FIFOEN_POS) /*!< FIFO Mode Enable */

/* Bitfield struct */
typedef union {
    struct {
        uint32_t UE     : 1;  /*!< USART Enable */
        uint32_t UESM   : 1;  /*!< USART Enable in Stop Mode */
        uint32_t RE     : 1;  /*!< Receiver Enable */
        uint32_t TE     : 1;  /*!< Transmitter Enable */
        uint32_t IDLEIE : 1;  /*!< IDLE Interrupt Enable */
        uint32_t RTOIE  : 1;  /*!< Receiver Timeout Interrupt Enable */
        uint32_t        : 20; /*!< Reserved */
        uint32_t FIFOEN : 1;  /*!< FIFO Mode Enable */
        uint32_t OVER8  : 1;  /*!< Oversampling Mode */
        uint32_t TXFEIE : 1;  /*!< TXFIFO Empty Interrupt Enable */
        uint32_t RXFFIE : 1;  /*!< RXFIFO Full Interrupt Enable */
        uint32_t        : 3;  /*!< Reserved */
    } b;
    uint32_t w;
} UART_CR1_TypeDef;
```

### Step 3 - Chain to Initialization Code

Extend the agent with a second prompt that takes the generated header and produces an `UART_Init()` function:

```
Given the UART_CR1 header you just generated, write a C function:
  void UART_Init(UART_TypeDef *huart, uint32_t baud, uint32_t clockHz)

The function must:
1. Enable the USART (UE bit)
2. Enable TX and RX (TE + RE bits)
3. Enable FIFO mode (FIFOEN bit)
4. Calculate and set the BRR register for the requested baud rate
5. Include inline comments explaining the math for BRR calculation
```

### Step 4 - Automate with a Script

Write a Python script that:

1. Accepts a register description as a text file
2. Sends it to an LLM API (OpenAI-compatible or local Ollama)
3. Writes the generated header to `output/<register_name>.h`

```python
import openai
import sys
import pathlib

def generate_header(register_text: str, register_name: str) -> str:
    client = openai.OpenAI()
    prompt = f"""
You are a firmware code generation agent.
Generate a complete C header for the following register description.
Follow STM32 HAL naming conventions.
Include bit-position defines, bit-mask defines, and a bitfield union struct.

Register description:
{register_text}
"""
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content

if __name__ == "__main__":
    input_file = pathlib.Path(sys.argv[1])
    register_name = input_file.stem
    text = input_file.read_text()
    header = generate_header(text, register_name)

    output_dir = pathlib.Path("output")
    output_dir.mkdir(exist_ok=True)
    (output_dir / f"{register_name}.h").write_text(header)
    print(f"Header written to output/{register_name}.h")
```

---

## Summary

| Skill                    | What You Practiced                                     |
| ------------------------ | ------------------------------------------------------ |
| Register table ingestion | Feeding raw datasheet text to an agent                 |
| Structured C generation  | Prompting for `#define`, bitfield structs, and Doxygen |
| Output validation        | Comparing agent output against a known-good reference  |
| Agent chaining           | Using header output as context for initialization code |
| Automation script        | Wrapping the agent call in a reusable Python pipeline  |

---

> **Next Lab:** [004 - Latency-Optimized Loops](../004-LatencyOptimizedLoops/README.md)
