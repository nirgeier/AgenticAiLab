# Lab 008 - Security & Safety for Firmware

!!! hint "Overview"

    - Agentic AI Foundations for R&D (FW Module)** - Security & Safety module.
    - In this lab, you will use agents to **automatically scan firmware code for buffer overflows and power-hungry polling loops**.
    - You will build a two-stage security agent: one that identifies vulnerabilities and one that proposes safe, compilable replacements.
    - By the end of this lab, you will have a reusable security-scan agent that can be integrated into a CI pipeline as an automated pre-commit or pre-release step.

## Prerequisites

- Completed [Lab 007 - Self-Healing FW Workflows](../007-SelfHealingWorkflows/README.md)

## What You Will Learn

- Common firmware security vulnerabilities that agents can detect
- How to prompt agents to scan for buffer overflows, unsafe memory operations, and integer overflows
- How to identify power-hungry polling anti-patterns
- How to generate secure alternatives for flagged code patterns

---

## Background

### Firmware Security vs. Application Security

Firmware security vulnerabilities are different from web application security:

| Firmware Vulnerability          | Risk                                             |
| ------------------------------- | ------------------------------------------------ |
| Buffer overflow (stack)         | Overwrites return address → RCE or system crash  |
| Unsafe `strcpy` / `sprintf`     | No bounds check → stack/heap corruption          |
| Integer overflow in size calc   | `malloc(len + overhead)` wraps → heap overflow   |
| Polling loop on unprotected I/O | Side-channel timing attack or DoS via stuck loop |
| Hardcoded credentials in flash  | Extractable with JTAG or flash readout           |
| Unchecked return values         | Silent failure → undefined behavior              |

### Power-Related Safety Issues

Agents can also flag **power safety** issues that are not traditional security bugs but can cause field failures:

| Pattern                    | Issue                                     |
| -------------------------- | ----------------------------------------- |
| `while(1)` polling on GPIO | Prevents deep sleep → battery drain       |
| Peripheral always-on       | Clock never gated → constant power draw   |
| No watchdog timeout reset  | System hangs without reset if task stalls |

---

## Lab Steps

### Step 1 - Buffer Overflow Scan

Paste the following vulnerable firmware snippet into your agent:

```c
#include <string.h>
#include <stdio.h>
#include <stdint.h>

#define CMD_BUFFER_SIZE 64

typedef struct {
    char command[CMD_BUFFER_SIZE];
    uint8_t checksum;
} UartPacket_t;

void ProcessUartInput(const char *input, size_t input_len) {
    UartPacket_t pkt;

    // Copy input into command buffer
    strcpy(pkt.command, input);  // UNSAFE: no bounds check

    // Parse command
    char response[32];
    sprintf(response, "CMD: %s", pkt.command);  // UNSAFE: overflow if pkt.command > 26 chars

    // Verify checksum
    uint8_t calculated = 0;
    for (int i = 0; i < input_len; i++) {
        calculated += (uint8_t)input[i];
    }
    pkt.checksum = calculated;
}
```

**Agent Prompt:**

```
You are a firmware security audit agent.
Scan the following C code for memory safety vulnerabilities.
For each issue found:
1. Identify the exact line and function
2. Classify the vulnerability (buffer overflow, unsafe format string, etc.)
3. Explain the attack vector and worst-case impact on an embedded system
4. Provide a secure replacement using safe C functions or bounded alternatives

[paste code here]
```

**Expected Findings:**

| Line | Vulnerability        | Fix                                                               |
| ---- | -------------------- | ----------------------------------------------------------------- |
| 12   | `strcpy` - unbounded | `strlcpy(pkt.command, input, sizeof(pkt.command))`                |
| 16   | `sprintf` - overflow | `snprintf(response, sizeof(response), "CMD: %.20s", pkt.command)` |

### Step 2 - Power-Hungry Polling Loop Detection

```c
// Anti-pattern: polling-based button debounce
void WaitForButtonPress(void) {
    while (GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET) {
        // Busy-wait - CPU at 100% doing nothing useful
    }
    // Debounce
    for (volatile int i = 0; i < 100000; i++);
}

// Anti-pattern: always-on ADC polling
uint16_t ReadADC_Blocking(void) {
    ADC1->CR2 |= ADC_CR2_SWSTART;
    while (!(ADC1->SR & ADC_SR_EOC)) {
        // Busy-wait for conversion
    }
    return (uint16_t)(ADC1->DR & 0x0FFF);
}
```

**Agent Prompt:**

```
Analyze the following firmware code for power efficiency issues.
For each polling loop found:
1. Estimate the worst-case CPU utilization (assuming the condition may never become true)
2. Calculate the approximate power impact on a 3.3V Cortex-M4 running at 168 MHz
   (assume 100 mA active current vs 5 µA deep sleep)
3. Propose an interrupt-driven or low-power replacement
4. Show the replacement code with required NVIC and peripheral configuration
```

### Step 3 - Build an Automated Security Scanner

Create a Python script that scans all `.c` files in a project directory:

```python
import pathlib
import openai
import json

SECURITY_PROMPT = """
You are a firmware security scanning agent.
Analyze the following C firmware source file for:
1. Buffer overflow vulnerabilities (strcpy, sprintf, gets, scanf without bounds)
2. Integer overflow in memory size calculations
3. Unchecked return values from memory allocation or peripheral init functions
4. Hardcoded secrets or credentials in string literals
5. Busy-wait polling loops that block CPU

Return a JSON array of findings. Each finding must have:
  - "line": approximate line number (integer)
  - "severity": "HIGH" | "MEDIUM" | "LOW"
  - "type": vulnerability type string
  - "description": one-sentence description
  - "recommendation": one-sentence fix

If no issues found, return an empty array [].
"""

def scan_file(file_path: pathlib.Path) -> list[dict]:
    client = openai.OpenAI()
    source = file_path.read_text()

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": SECURITY_PROMPT},
            {"role": "user", "content": f"File: {file_path.name}\n\n{source}"}
        ],
        response_format={"type": "json_object"}
    )

    content = response.choices[0].message.content
    data = json.loads(content)
    findings = data if isinstance(data, list) else data.get("findings", [])
    return findings

def scan_project(project_dir: str) -> None:
    root = pathlib.Path(project_dir)
    c_files = list(root.rglob("*.c"))
    print(f"Scanning {len(c_files)} source files in {project_dir}...\n")

    total_findings = 0
    for f in c_files:
        findings = scan_file(f)
        if findings:
            print(f"[{f.relative_to(root)}] - {len(findings)} finding(s):")
            for item in findings:
                sev = item.get("severity", "?")
                print(f"  [{sev}] Line {item.get('line','?')}: {item.get('type')} - {item.get('description')}")
                print(f"         Fix: {item.get('recommendation')}")
            print()
            total_findings += len(findings)

    print(f"Scan complete. Total findings: {total_findings}")

if __name__ == "__main__":
    import sys
    scan_project(sys.argv[1] if len(sys.argv) > 1 else ".")
```

### Step 4 - Generate a Secure Replacement File

Ask the agent to produce a hardened version of the scanned file:

```
Given the security audit findings above, rewrite ProcessUartInput() to:
1. Replace strcpy with a bounds-checked copy (max CMD_BUFFER_SIZE - 1 bytes + null terminator)
2. Replace sprintf with snprintf using explicit buffer size
3. Add an input validation check that rejects input_len > CMD_BUFFER_SIZE
4. Return an error code (int) instead of void so the caller can handle failures
5. Add a brief comment explaining the security fix for each change
```

---

## Summary

| Skill                     | What You Practiced                                           |
| ------------------------- | ------------------------------------------------------------ |
| Buffer overflow detection | Identifying strcpy/sprintf misuse with agent-guided audit    |
| Power analysis            | Quantifying busy-wait cost in CPU cycles and battery impact  |
| Automated scanner         | Python pipeline that scans all .c files and reports findings |
| Hardened code generation  | Using audit findings as context for secure code replacement  |

---

> **Next Lab:** [009 - Vibe Coding for Hardware](../009-VibeCodingForHW/README.md)
