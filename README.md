<a href="https://stackoverflow.com/users/1755598"><img src="https://stackexchange.com/users/flair/1951642.png" width="208" height="58" alt="profile for CodeWizard on Stack Exchange" title="profile for CodeWizard on Stack Exchange"></a>

![Visitor Badge](https://visitor-badge.laobi.icu/badge?page_id=nirgeier.AgenticAI-FW-Course)
[![Linkedin Badge](https://img.shields.io/badge/-nirgeier-blue?style=flat-square&logo=Linkedin&logoColor=white&link=https://www.linkedin.com/in/nirgeier/)](https://www.linkedin.com/in/nirgeier/)
[![Gmail Badge](https://img.shields.io/badge/-nirgeier@gmail.com-fcc624?style=flat-square&logo=Gmail&logoColor=red&link=mailto:nirgeier@gmail.com)](mailto:nirgeier@gmail.com)
[![Outlook Badge](https://img.shields.io/badge/-nirg@codewizard.co.il-fcc624?style=flat-square&logo=microsoftoutlook&logoColor=blue&link=mailto:nirg@codewizard.co.il)](mailto:nirg@codewizard.co.il)

---

# Agentic AI for Firmware - 1-Day Intensive Training

> A hands-on, hardware-aware course for R&D and firmware engineering teams.
> Covering **Autonomous Agents** that understand registers, timing constraints, and power states.

- Each lab is a **standalone** module and does not require completing previous labs.

## Pre-Requirements

- Basic knowledge of C/C++ firmware development
- Familiarity with memory-mapped I/O concepts
- VS Code (latest version) with GitHub Copilot or access to Claude Code
- Python 3.10+ (for LangGraph exercises)

---

## [🚀 Interactive Labs & Documentation](https://nirgeier.github.io/AgenticAI-FW-Course/)

---

## Course Modules

| Module | Lab                                                                     | Description                                                                |
| :----- | :---------------------------------------------------------------------- | :------------------------------------------------------------------------- |
| 001    | [Introduction to Agentic AI for FW](Labs/001-Introduction/)             | Agents vs. coding assistants; why firmware needs hardware-aware AI.        |
| 002    | [Hardware-Aware Agent Architecture](Labs/002-HardwareAwareAgents/)      | How agents process datasheets, hardware manuals, and constraint files.     |
| 003    | [Automated Register Mapping](Labs/003-AutomatedRegisterMapping/)        | Agents that parse PDF datasheets and generate C header files.              |
| 004    | [Latency-Optimized Loops](Labs/004-LatencyOptimizedLoops/)              | Refactoring blocking code into interrupt-driven and DMA architectures.     |
| 005    | [Agentic JTAG Debugging](Labs/005-AgenticDebugging/)                    | Integrating agents with Serial/JTAG logs to fix race conditions.           |
| 006    | [Agentic Frameworks - LangGraph](Labs/006-AgenticFrameworks/)           | Building stateful agents that reason before writing firmware code.         |
| 007    | [Self-Healing FW Workflows](Labs/007-SelfHealingWorkflows/)             | Agents that write, compile, analyze errors, and iterate autonomously.      |
| 008    | [Security & Safety for Firmware](Labs/008-SecuritySafety/)              | Scanning for buffer overflows and power-hungry polling loops.              |
| 009    | [Vibe Coding for Hardware](Labs/009-VibeCodingForHW/)                   | Claude Code / Copilot Workspace for complex multi-file FW refactoring.     |
| 010    | [Power-Sensitive Refactoring](Labs/010-PowerSensitiveRefactoring/)      | Identifying high-power patterns and rewriting for low-power sleep modes.   |
| 011    | [Tool-Use & Function Calling](Labs/011-ToolUseFunctionCalling/)         | Teaching agents to trigger QEMU and verify firmware logic autonomously.    |
| 012    | [ReAct Patterns for HW-SW Debugging](Labs/012-ReActPatterns/)           | Reason+Act patterns for hardware-software synchronization debugging.       |
| 013    | [Agentic Documentation](Labs/013-AgenticDocumentation/)                 | Agents that keep FW code in sync with technical design documents.          |
| 014    | [Constraint-Based Code Generation](Labs/014-ConstraintBasedGeneration/) | Prompting agents with strict hardware limits (SRAM, flash, power budgets). |
