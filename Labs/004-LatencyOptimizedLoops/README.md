# Lab 004 - Latency-Optimized Loops

!!! hint "Overview"

    - Agentic AI for Embedded & RTOS** - Latency-Optimized Loops module.
    - In this lab, you will use agentic AI to identify and refactor **blocking code** into interrupt-driven or DMA-based architectures.
    - You will learn to prompt agents with timing constraints and have them propose refactoring strategies that respect ISR latency budgets.
    - By the end of this lab, you will be able to take legacy polling-based firmware and produce an interrupt-driven or DMA-backed equivalent using agent assistance.

!!! warning "Hardware Knowledge Required"

    This lab assumes familiarity with interrupt vectors, NVIC priority levels, and DMA stream configuration.
    Having the target MCU's Reference Manual open alongside the exercises is strongly recommended.

## Prerequisites

- Completed [Lab 003 - Automated Register Mapping](../003-AutomatedRegisterMapping/README.md)
- Understanding of interrupt service routines (ISRs) and DMA basics

## What You Will Learn

- How to identify blocking patterns an agent can flag automatically
- How to prompt an agent with timing constraints (e.g., "max 500 µs latency")
- Strategies for refactoring polling loops to interrupt-driven architectures
- Strategies for offloading data movement to DMA with agent-generated configuration code

---

## Background

### The Blocking Code Problem

Blocking loops are one of the most common firmware performance killers:

```c
// BLOCKING - burns CPU while waiting for UART TX complete
while (!(USART2->ISR & USART_ISR_TXE_TXFNF)) {
    /* wait */
}
USART2->TDR = data;
```

On a system where the CPU also manages sensor fusion, motor control, and a real-time OS, every microsecond spent in a busy-wait is a deadline missed somewhere else.

### Agent Refactoring Strategies

An agent given timing context can propose one of three refactoring strategies:

| Strategy            | Use When                                      | CPU Usage |
| ------------------- | --------------------------------------------- | --------- |
| Interrupt-driven TX | Single transfer, low frequency                | Low       |
| DMA circular buffer | Continuous streaming (audio, sensor data)     | Minimal   |
| DMA + half/full ISR | Streaming with real-time processing per chunk | Minimal   |

---

## Lab Steps

### Step 1 - Flag Blocking Patterns

Give the agent your source file and ask it to produce a blocking-code audit:

```
Analyze the following firmware source code.
Identify every location where the CPU is spinning in a busy-wait loop
waiting for a hardware flag. For each occurrence, report:
  1. The file and line number
  2. The hardware peripheral involved
  3. The worst-case blocking duration in CPU cycles if the peripheral
     is running at its slowest valid configuration
  4. A recommended refactoring strategy (interrupt or DMA)

[paste your source code here]
```

### Step 2 - Refactor a UART Polling Loop to Interrupt-Driven

**Input (blocking):**

```c
void UART_SendByte(uint8_t data) {
    while (!(USART2->ISR & USART_ISR_TXE_TXFNF));
    USART2->TDR = data;
}

void UART_SendBuffer(const uint8_t *buf, uint16_t len) {
    for (uint16_t i = 0; i < len; i++) {
        UART_SendByte(buf[i]);
    }
}
```

**Agent Prompt:**

```
Refactor the following UART send functions to be fully interrupt-driven.
Requirements:
- Use a circular TX ring buffer of 256 bytes
- Trigger the ring buffer flush from USART2_IRQHandler (TXE interrupt)
- UART_SendBuffer() must be non-blocking: copy data to ring buffer and return immediately
- The ring buffer must be safe for concurrent access from main context and ISR
- Target MCU: STM32F4, USART2, running at 84 MHz APB1 clock
- Baud rate: 115200

Generate:
1. ring_buffer.h - ring buffer struct and API
2. uart_it.c - interrupt-driven UART implementation
3. uart_it.h - public API
```

### Step 3 - Refactor to DMA

**Agent Prompt:**

```
Replace the interrupt-driven UART TX implementation from Step 2 with a
DMA-based implementation.

Requirements:
- Use DMA1 Stream 6, Channel 4 (USART2_TX on STM32F4)
- Support back-to-back transfers without tearing
- Use DMA Transfer Complete interrupt to signal completion
- Provide a non-blocking API: UART_DMA_Send(buf, len)
- Generate an ASSERT macro that fires if a new transfer is requested
  while a DMA transfer is still in progress

Generate the complete uart_dma.c and uart_dma.h files.
```

### Step 4 - Validate with Timing Assertions

Ask the agent to add compile-time and runtime timing assertions:

```
Add the following safety assertions to the DMA UART driver:
1. A static_assert that verifies the ring buffer size is a power of two
2. A runtime assertion that the DMA transfer length never exceeds 65535 bytes
   (hardware NDTR register limit)
3. A cycle-count measurement macro that logs maximum observed ISR latency
   using DWT->CYCCNT, triggered if latency exceeds 1000 CPU cycles
```

---

## Summary

| Skill                     | What You Practiced                                           |
| ------------------------- | ------------------------------------------------------------ |
| Blocking-code audit       | Prompting agent to flag busy-wait patterns with context      |
| Interrupt-driven refactor | Generating ring-buffer + ISR architecture from blocking code |
| DMA refactor              | DMA stream/channel configuration from natural language spec  |
| Timing assertions         | Adding compile-time and runtime safety checks                |

---

> **Next Lab:** [005 - Agentic JTAG Debugging](../005-AgenticDebugging/README.md)
