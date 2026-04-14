# Lab 009 - Vibe Coding for Hardware

!!! hint "Overview"

    - Future of AI: Masterclass in Agentic Engineering**.
    - "Vibe Coding" is the practice of using AI agents to handle **complex, multi-file firmware refactoring** from a high-level description.
    - In this lab, you will use Claude Code or GitHub Copilot Workspace to refactor a multi-file firmware project from a top-down task description.
    - By the end of this lab, you will be able to delegate large refactoring tasks to an agent and effectively guide it with architectural intent rather than line-by-line instructions.

## Prerequisites

- Completed [Lab 008 - Security & Safety for Firmware](../008-SecuritySafety/README.md)
- GitHub Copilot Agent Mode (VS Code) **or** Claude Code CLI installed

```bash
# Claude Code CLI
npm install -g @anthropic-ai/claude-code

# Verify
claude --version
```

## What You Will Learn

- The Vibe Coding mindset: describe _intent_, not steps
- How to structure multi-file refactoring tasks for agents
- How to use `@workspace` context in GitHub Copilot Agent Mode
- Practical patterns for reviewing and accepting large agent-generated changesets

---

## Background

### What Is Vibe Coding for Firmware?

Traditional firmware development is instruction-oriented: you tell the compiler (and yourself) exactly what to do at each step.

Vibe coding with agents is **intent-oriented**: you describe _what the system should do_ and the agent reasons over your entire codebase to make it happen.

| Traditional Approach       | Vibe Coding Approach                                    |
| -------------------------- | ------------------------------------------------------- |
| Edit `uart.c` line 42      | "Convert all blocking UART operations to async"         |
| Add `#include "dma.h"`     | "Add DMA support across all peripheral drivers"         |
| Update `SPI_Init()` params | "Standardize all init functions to use a config struct" |

The agent reads **all relevant files**, proposes a coherent change plan, and applies edits.

---

## Lab Steps

### Step 1 - Describe an Architectural Refactoring Goal

Using GitHub Copilot Agent Mode (`@workspace`) or Claude Code, run the following task:

```
@workspace

I need to refactor this firmware project's peripheral driver layer.
The current code uses direct register writes scattered across main.c and multiple
driver files. I want to:

1. Create a unified driver_config.h with typed config structs for UART, SPI, and I2C
2. Move all peripheral init code from main.c into dedicated uart.c, spi.c, and i2c.c files
3. Replace all magic numbers (register bit positions) with named constants from the new headers
4. Ensure every driver file includes only its own header - no cross-includes
5. Keep the public API unchanged: UART_Init(), SPI_Init(), I2C_Init() must work as before

Do not change any business logic. Only restructure the driver layer.
Produce a summary of every file you plan to create, modify, or delete before making changes.
```

### Step 2 - Review the Agent's Change Plan

Before accepting changes, ask the agent to explain its plan:

```
Before applying any edits, list:
1. Every file that will be CREATED with a one-line description of its purpose
2. Every file that will be MODIFIED with a summary of what changes
3. Every file that will be DELETED and why it is safe to remove
4. Any risks or assumptions in the refactoring (e.g., "assumes I2C is only used in main.c")
```

Review this plan before proceeding. Ask follow-up questions such as:

```
Why are you moving the DMA configuration to spi.c instead of keeping it in dma.c?
```

### Step 3 - Multi-File Rename and Pattern Standardization

Give the agent a cross-cutting naming convention task:

```
@workspace

Rename all firmware functions across the project to follow this convention:
- Format: <Module>_<Action>_<Object>
- Examples:
    HAL_UART_SendByte   → UART_Send_Byte
    HAL_SPI_TransmitDMA → SPI_Transmit_DMA
    I2C_MasterReceive   → I2C_Receive_Master

Rules:
1. Update both declarations in .h files and definitions in .c files
2. Update all call sites
3. Do NOT rename third-party HAL functions (prefixed with HAL_ in vendor headers)
4. Generate a rename mapping table (old_name → new_name) as a comment at the top of CHANGES.md
```

### Step 4 - Validate With a Build Check

After the agent applies changes, trigger a build check:

```
@workspace

Run the ARM cross-compiler on the refactored project and show me:
1. All compiler warnings (even if the build succeeds)
2. Any newly introduced warnings compared to before the refactoring
3. The final binary size (text + data + bss sections) from the linker map
```

If the agent cannot execute the build directly, ask it to produce the build command and validate output yourself.

### Step 5 - Incremental Vibe: Add a Feature Across Files

Add a new cross-cutting feature using a single intent prompt:

```
@workspace

Add timestamping to all debug log calls in the firmware.
Requirements:
- Use the existing SysTick millisecond counter (Sys_GetTick())
- Every call to DEBUG_LOG(), TRACE(), and ERROR_LOG() must prepend [T=<ms>]
- Update the macro definitions in debug.h
- Do not change call sites - only the macro definitions should change
- Add a unit test in tests/test_debug.c that verifies the timestamp prefix format
```

---

## Summary

| Skill                     | What You Practiced                                           |
| ------------------------- | ------------------------------------------------------------ |
| Intent-based prompting    | Describing architectural goals instead of line-level edits   |
| Change plan review        | Asking agent for file-level plan before accepting changes    |
| Cross-cutting refactoring | Renaming conventions applied across all files simultaneously |
| Build validation          | Verifying agent changes don't break compilation              |
| Feature addition          | Adding a cross-cutting concern without touching call sites   |

---

> **Next Lab:** [010 - Power-Sensitive Refactoring](../010-PowerSensitiveRefactoring/README.md)
