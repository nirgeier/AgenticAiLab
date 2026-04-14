# Lab 001 - Introduction to Agentic AI for Firmware

!!! hint "Overview"

    - In this lab, you will understand the fundamental shift from traditional AI coding assistants to **Autonomous Agents** purpose-built for firmware development.
    - You will learn why standard LLM completions fall short when dealing with registers, IRQ priorities, DMA channels, and power states.
    - You will survey the landscape of available agentic tools in 2026 and map them to firmware engineering workflows.
    - By the end of this lab, you will be able to articulate the difference between a copilot and an agent, and identify where agents add the most value in your FW development cycle.

## Prerequisites

- Basic familiarity with C/C++ microcontroller development
- A general understanding of what an LLM / AI coding assistant is

## What You Will Learn

- The definition of an **Agentic AI** system and how it differs from a code completion tool
- Why firmware engineering has unique requirements that generic AI assistants cannot address
- The four dimensions of hardware-aware agents: **registers**, **timing**, **power states**, and **tool use**
- An overview of the current agentic AI landscape and which tools are best for different firmware tasks
- How to evaluate whether an agent is truly "hardware-aware" or just a fancy autocomplete

## Background

### Copilots vs. Agents

A **coding copilot** (e.g., GitHub Copilot inline completions) responds to your current cursor context and suggests the next token or block of code.
It is reactive - you type, it completes.

An **autonomous agent** operates differently:

| Aspect    | Coding Copilot           | Autonomous Agent                               |
| --------- | ------------------------ | ---------------------------------------------- |
| Trigger   | Cursor position / prompt | High-level task description                    |
| Scope     | Current file / selection | Multiple files, external tools, build systems  |
| Memory    | Context window only      | Stateful graph (can remember previous steps)   |
| Actions   | Suggests text            | Writes files, runs compiler, reads JTAG output |
| Iteration | Single shot              | Self-corrects until goal is achieved           |

For firmware, the agent must also understand:

- **Memory-mapped registers** - address offsets, bit-field semantics, read/write-clear flags
- **Timing constraints** - interrupt latency budgets, DMA transfer windows, watchdog deadlines
- **Power states** - sleep modes, peripheral clock gating, wake-up sources
- **Toolchain integration** - cross-compilers, linker scripts, hardware simulators

---

## Lab Steps

### Step 1 - Identify Agent Use Cases in Your FW Workflow

Review the following list and mark the tasks where an agent would save the most time in your team's day-to-day work:

- [ ] Parsing a new peripheral datasheet and creating a C driver skeleton
- [ ] Refactoring a busy-wait loop into an interrupt service routine
- [ ] Running the cross-compiler and iterating on build errors autonomously
- [ ] Scanning the codebase for unsafe `memcpy` / buffer overflow patterns
- [ ] Keeping the technical design document in sync with header file changes
- [ ] Verifying firmware logic in QEMU before flashing to hardware

### Step 2 - Compare Agent Outputs

Using your preferred AI tool (GitHub Copilot Agent Mode or Claude Code), run the following two prompts and compare the depth of responses:

**Prompt A (generic):**

```
Write a function to initialize UART.
```

**Prompt B (hardware-aware):**

```
Generate a C function to initialize UART2 on an STM32H7 running at 480 MHz.
Baud rate: 115200. FIFO enabled. DMA TX on stream 3, channel 4.
Use the STM32H7 HAL library. Include register-level comments explaining
each bit field written to CR1, CR2, and CR3.
```

Observe how specificity about hardware constraints dramatically improves agent output quality.

### Step 3 - Map the Course to Your Needs

Review the course table of contents and identify which 3 labs are most relevant to your current project:

```
001 - Introduction (this lab)
002 - Hardware-Aware Agent Architecture
003 - Automated Register Mapping
004 - Latency-Optimized Loops
005 - Agentic JTAG Debugging
006 - Agentic Frameworks / LangGraph
007 - Self-Healing FW Workflows
008 - Security & Safety
009 - Vibe Coding for Hardware
010 - Power-Sensitive Refactoring
011 - Tool-Use & Function Calling
012 - ReAct Patterns for HW-SW
013 - Agentic Documentation
014 - Constraint-Based Generation
```

---

## Summary

| Key Takeaway                   | Detail                                                                            |
| ------------------------------ | --------------------------------------------------------------------------------- |
| Agents are stateful            | They maintain a task graph, not just a context window                             |
| Hardware context is critical   | Registers, timing, and power states must be in the prompt                         |
| Tool use is the differentiator | Agents that can run the compiler and read JTAG output close the loop autonomously |

---

> **Next Lab:** [002 - Hardware-Aware Agent Architecture](../002-HardwareAwareAgents/README.md)
