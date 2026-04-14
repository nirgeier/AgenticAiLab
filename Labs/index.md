---
hide:
  - toc
---

# Agentic AI for Firmware - Hands-On Labs

---

- Welcome to the **Agentic AI for Firmware** 1-day intensive training!
- This course is designed for R&D and firmware engineering teams who want to move beyond simple coding assistants and harness **Autonomous Agents** that understand registers, timing constraints, and power states.
- Each lab is independent and can be completed in any order.

---

## Course Modules

| #   | Lab                                                                         | Description                                                                |
| --- | --------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| 001 | [Introduction to Agentic AI for FW](001-Introduction/README.md)             | Agents vs. coding assistants; why firmware needs hardware-aware AI.        |
| 002 | [Hardware-Aware Agent Architecture](002-HardwareAwareAgents/README.md)      | How agents process datasheets, hardware manuals, and constraint files.     |
| 003 | [Automated Register Mapping](003-AutomatedRegisterMapping/README.md)        | Agents that parse PDF datasheets and generate C header files.              |
| 004 | [Latency-Optimized Loops](004-LatencyOptimizedLoops/README.md)              | Refactoring blocking code into interrupt-driven and DMA architectures.     |
| 005 | [Agentic JTAG Debugging](005-AgenticDebugging/README.md)                    | Integrating agents with Serial/JTAG logs to fix race conditions.           |
| 006 | [Agentic Frameworks - LangGraph](006-AgenticFrameworks/README.md)           | Building stateful agents that reason before writing firmware code.         |
| 007 | [Self-Healing FW Workflows](007-SelfHealingWorkflows/README.md)             | Agents that write, compile, analyze errors, and iterate autonomously.      |
| 008 | [Security & Safety for Firmware](008-SecuritySafety/README.md)              | Scanning for buffer overflows and power-hungry polling loops.              |
| 009 | [Vibe Coding for Hardware](009-VibeCodingForHW/README.md)                   | Claude Code / Copilot Workspace for complex multi-file FW refactoring.     |
| 010 | [Power-Sensitive Refactoring](010-PowerSensitiveRefactoring/README.md)      | Identifying high-power patterns and rewriting for low-power sleep modes.   |
| 011 | [Tool-Use & Function Calling](011-ToolUseFunctionCalling/README.md)         | Teaching agents to trigger QEMU and verify firmware logic autonomously.    |
| 012 | [ReAct Patterns for HW-SW Debugging](012-ReActPatterns/README.md)           | Reason+Act patterns for hardware-software synchronization debugging.       |
| 013 | [Agentic Documentation](013-AgenticDocumentation/README.md)                 | Agents that keep FW code in sync with technical design documents.          |
| 014 | [Constraint-Based Code Generation](014-ConstraintBasedGeneration/README.md) | Prompting agents with strict hardware limits (SRAM, flash, power budgets). |
