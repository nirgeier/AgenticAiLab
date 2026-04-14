# Agentic AI for Firmware - Tasks

- Welcome to the **AgenticAI-FW-Course Tasks** section.
- Each folder below contains hands-on exercises that you can complete independently to practice specific agentic AI skills for firmware development.
- Every task includes a description, a scenario, a hint, and a collapsible **Solution** block.

## Task Index

### Foundations

|                                                                              |                                                                                                            |
| ---------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| [001 - Introduction Tasks](001-Introduction-Tasks/README.md)                 | Distinguish copilots from agents, map firmware tasks to agent capabilities, evaluate context completeness. |
| [002 - Hardware-Aware Agents Tasks](002-HardwareAwareAgents-Tasks/README.md) | Identify agent pillars, select context injection methods, design minimal hardware-aware agents.            |

### Register Mapping & Optimization

|                                                                                  |                                                                                                           |
| -------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| [003 - Register Mapping Tasks](003-AutomatedRegisterMapping-Tasks/README.md)     | Generate C headers from register tables, validate bitfield structs, chain to initialization code.         |
| [004 - Latency-Optimized Loops Tasks](004-LatencyOptimizedLoops-Tasks/README.md) | Flag blocking patterns, calculate worst-case latency, refactor to interrupt-driven and DMA architectures. |
| [005 - Agentic Debugging Tasks](005-AgenticDebugging-Tasks/README.md)            | Decode Cortex-M hard faults, analyze FreeRTOS race conditions, build automated serial log monitoring.     |

### Agentic Frameworks

|                                                                                |                                                                                                                 |
| ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| [006 - Agentic Frameworks Tasks](006-AgenticFrameworks-Tasks/README.md)        | Define LangGraph state, build plan/generate/review nodes, wire conditional edges, add human-in-the-loop checks. |
| [007 - Self-Healing Workflows Tasks](007-SelfHealingWorkflows-Tasks/README.md) | Wrap arm-gcc as a tool, parse GCC errors, build iteration loop with max-retry guards.                           |
| [008 - Security & Safety Tasks](008-SecuritySafety-Tasks/README.md)            | Identify buffer overflows, detect polling anti-patterns, build an automated security scanner.                   |

### Advanced Patterns

|                                                                                          |                                                                                                  |
| ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| [009 - Vibe Coding Tasks](009-VibeCodingForHW-Tasks/README.md)                           | Write architectural refactoring intents, get file-level plans, perform cross-cutting renames.    |
| [010 - Power-Sensitive Refactoring Tasks](010-PowerSensitiveRefactoring-Tasks/README.md) | Identify high-power patterns, calculate power budgets, refactor to low-power sleep mode.         |
| [011 - Tool-Use & Function Calling Tasks](011-ToolUseFunctionCalling-Tasks/README.md)    | Define tool schemas, implement handlers, parse tool responses, build the autonomous loop.        |
| [012 - ReAct Patterns Tasks](012-ReActPatterns-Tasks/README.md)                          | Implement Thought/Action/Observation cycles, run ReAct on real hardware bugs.                    |
| [013 - Agentic Documentation Tasks](013-AgenticDocumentation-Tasks/README.md)            | Detect documentation drift, extract public API, run git-diff-aware scans, auto-generate Doxygen. |
| [014 - Constraint-Based Generation Tasks](014-ConstraintBasedGeneration-Tasks/README.md) | Express constraints formally, generate constrained drivers, measure with arm-none-eabi-size.     |

### Final Project

|                                                          |                                                                                                                                     |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| [Firmware Intelligence Pipeline](FinalProject/README.md) | End-to-end project: register mapping → self-healing compilation → security scan → constraint check → living documentation pipeline. |

---

Happy learning and building with Agentic AI for Firmware!
