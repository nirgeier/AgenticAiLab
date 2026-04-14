# Lab 005 - Agentic JTAG Debugging

!!! hint "Overview"

    - Agentic AI for Embedded & RTOS** - Agentic Debugging module.
    - In this lab, you will integrate an AI agent with Serial and JTAG log outputs to diagnose firmware failures in real time.
    - You will learn to feed crash dumps, assert logs, and register state snapshots to an agent and receive actionable fix suggestions.
    - By the end of this lab, you will know how to construct an agent-in-the-loop debugging workflow that accelerates root-cause analysis of race conditions and hard faults.

## Prerequisites

- Completed [Lab 004 - Latency-Optimized Loops](../004-LatencyOptimizedLoops/README.md)
- Basic familiarity with ARM Cortex-M fault registers (CFSR, HFSR, MMAR, BFAR)

## What You Will Learn

- How to capture and format JTAG / SWD register dumps for agent consumption
- How to prompt an agent to diagnose ARM Cortex-M hard faults
- How to use agents to identify race conditions from concurrent RTOS task logs
- How to build an automated debugging loop: agent reads log → suggests fix → engineer applies → repeat

---

## Background

### The Firmware Debugging Bottleneck

Hard faults and race conditions in firmware are notoriously difficult to debug:

- Hard faults produce a fault status register dump that must be decoded manually
- Race conditions may only manifest under specific timing conditions and are rarely reproducible
- JTAG session data is raw and verbose - extracting root cause requires experience

An agent fed with structured fault data can correlate register values, stack frames, and source code to suggest a root cause in seconds.

---

## Lab Steps

### Step 1 - Decode a Hard Fault Dump

Paste the following simulated JTAG output into your agent:

```
=== ARM Cortex-M7 Hard Fault ===
PC  = 0x08012A48
LR  = 0x08012A05 (EXC_RETURN = 0xFFFFFFF9 → Thread mode, MSP)
SP  = 0x20001F80

Fault Status Registers:
  HFSR  = 0x40000000  (FORCED - escalated from configurable fault)
  CFSR  = 0x00008200
    BFSR = 0x82
      PRECISERR = 1   ← Precise bus error
      BFARVALID = 1   ← BFAR contains a valid address
  BFAR  = 0x40013804  ← Faulting address

Stack Frame (stacked by hardware):
  R0  = 0x00000000
  R1  = 0x00000001
  R2  = 0x40013800
  R3  = 0x00000000
  R12 = 0x20000A10
  LR  = 0x08012A05
  PC  = 0x08012A48
  xPSR= 0x01000000
```

**Agent Prompt:**

```
You are a firmware debugging agent for ARM Cortex-M microcontrollers.
Analyze the following hard fault dump and provide:
1. The fault type and affected memory region
2. The most likely root cause (e.g., NULL pointer, unaligned access, peripheral not clocked)
3. The source code patterns most likely to produce this fault
4. Concrete next debugging steps

[paste fault dump here]
```

### Step 2 - Race Condition Analysis from FreeRTOS Logs

Paste the following RTOS trace log into your agent:

```
[T=1024ms] [TASK:SensorRead]  Acquired mutex SENSOR_MUTEX
[T=1024ms] [TASK:SensorRead]  Reading ADC channel 3... value=2048
[T=1025ms] [TASK:CommTask]    Attempting to acquire SENSOR_MUTEX (timeout=10ms)
[T=1035ms] [TASK:CommTask]    MUTEX TIMEOUT - proceeding without lock
[T=1035ms] [TASK:CommTask]    Reading sensor_buffer[2] = 0x0000 (stale/zero)
[T=1035ms] [TASK:CommTask]    Sending sensor data via UART...
[T=1036ms] [TASK:SensorRead]  Released mutex SENSOR_MUTEX
[T=1036ms] [TASK:SensorRead]  sensor_buffer updated: [2048, 1984, 3010]
```

**Agent Prompt:**

```
Analyze the following FreeRTOS task trace log.
Identify any race conditions or data consistency violations.
For each issue found:
1. Describe the race window (which tasks and which shared resource)
2. Explain the worst-case consequence (e.g., stale data sent over UART)
3. Propose the correct RTOS primitive to fix it (mutex, semaphore, queue, or event group)
4. Show a corrected code snippet for the CommTask mutex acquisition

[paste log here]
```

### Step 3 - Build a Log-Monitoring Agent Loop

Create a Python script that watches a serial port and automatically forwards anomalies to the agent:

```python
import serial
import openai
import re
import time

FAULT_PATTERNS = [
    r"Hard Fault",
    r"CFSR\s*=\s*0x[0-9A-Fa-f]+",
    r"ASSERT FAILED",
    r"MUTEX TIMEOUT",
    r"Stack overflow",
]

def is_anomaly(line: str) -> bool:
    return any(re.search(pattern, line, re.IGNORECASE) for pattern in FAULT_PATTERNS)

def ask_agent(log_snippet: str) -> str:
    client = openai.OpenAI()
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{
            "role": "user",
            "content": (
                "You are a firmware debugging agent. "
                "Analyze the following firmware log snippet and suggest the root cause and fix:\n\n"
                + log_snippet
            )
        }]
    )
    return response.choices[0].message.content

def monitor_serial(port: str, baud: int = 115200):
    buffer = []
    with serial.Serial(port, baud, timeout=1) as ser:
        print(f"Monitoring {port} at {baud} baud...")
        while True:
            line = ser.readline().decode("utf-8", errors="replace").strip()
            if line:
                print(f"[UART] {line}")
                buffer.append(line)
                if is_anomaly(line):
                    context = "\n".join(buffer[-20:])  # last 20 lines as context
                    print("\n[AGENT] Anomaly detected - querying agent...")
                    suggestion = ask_agent(context)
                    print(f"[AGENT SUGGESTION]\n{suggestion}\n")
                    buffer.clear()

if __name__ == "__main__":
    import sys
    monitor_serial(sys.argv[1] if len(sys.argv) > 1 else "/dev/ttyUSB0")
```

### Step 4 - Document Findings

After running the agent on a real or simulated log, ask it to produce a debugging report:

```
Based on the fault analysis sessions above, generate a structured debugging report with:
1. Executive Summary (2–3 sentences)
2. Root Cause Table (fault type, affected module, contributing factor)
3. Recommended Code Changes (with before/after snippets)
4. Preventive Measures (static analysis rules, RTOS configuration changes)
```

---

## Summary

| Skill                   | What You Practiced                                             |
| ----------------------- | -------------------------------------------------------------- |
| Hard fault decoding     | Feeding Cortex-M fault register dumps to an agent              |
| Race condition analysis | Using RTOS trace logs as agent input                           |
| Automated monitoring    | Python serial monitor that triggers agent on anomaly detection |
| Debugging report        | Agent-generated structured report from session context         |

---

> **Next Lab:** [006 - Agentic Frameworks - LangGraph](../006-AgenticFrameworks/README.md)
