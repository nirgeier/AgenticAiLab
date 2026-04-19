# Lab 008 - Security & Safety Tasks

- Exercises covering buffer overflow detection, unsafe function audits, power anti-pattern identification, and automated security scanning.
- Each task includes a description, scenario, hint, and collapsible Solution.

#### Table of Contents

- [01. Identify Buffer Overflow Vulnerabilities](#01-identify-buffer-overflow-vulnerabilities)
- [02. Audit Unsafe String Functions](#02-audit-unsafe-string-functions)
- [03. Detect Power-Wasting Anti-Patterns](#03-detect-power-wasting-anti-patterns)
- [04. Calculate Power Impact of a Bug](#04-calculate-power-impact-of-a-bug)
- [05. Build an Automated Security Scanner](#05-build-an-automated-security-scanner)

---

#### 01. Identify Buffer Overflow Vulnerabilities

Analyze the following firmware code and identify every buffer overflow vulnerability. For each vulnerability, state: the vulnerable line, the attack vector, and the recommended fix.

```c
#define CMD_BUF_SIZE  64

static char cmd_buffer[CMD_BUF_SIZE];
static uint8_t rx_data[8];

// Receives data from UART and accumulates a command
void UART_ProcessRx(uint8_t byte, uint16_t index)
{
    cmd_buffer[index] = byte;              // Line 9
}

// Parses a received command string
void Parse_Command(const char *input)
{
    char local_cmd[32];
    strcpy(local_cmd, input);              // Line 15
    // process local_cmd...
}

// Copies device name from OTA packet into config
void OTA_SetDeviceName(const char *name)
{
    static char device_name[16];
    memcpy(device_name, name, strlen(name)); // Line 22
}

// Formats a log entry
void Log_Write(uint32_t code, const char *msg)
{
    char log_buf[128];
    sprintf(log_buf, "[%08X] %s\n", code, msg);  // Line 28
}
```

#### Scenario:

◦ These patterns represent the most common security vulnerabilities in embedded firmware (OWASP Embedded Top 10).
◦ An agent auditing firmware for production release must catch all four before signing off.

**Hint:** Look for: unchecked index writes, unbounded copies, length passed as `strlen(src)` instead of `sizeof(dst)`, and `sprintf` without size limits.

<details markdown>
<summary>Solution</summary>

| Line   | Vulnerability                                | Attack Vector                                                                                                                                               | Fix                                                                           |
| ------ | -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| **9**  | **Out-of-bounds write**                      | `index` is never bounds-checked. An attacker sending 65+ bytes over UART writes past `cmd_buffer[63]` into adjacent stack/BSS memory.                       | `if (index < CMD_BUF_SIZE - 1) cmd_buffer[index] = byte;`                     |
| **15** | **Stack buffer overflow via `strcpy`**       | `strcpy` copies until `\0`. If `input` is longer than 31 bytes + null, it overflows `local_cmd[32]` onto the call stack frame, enabling ROP/code injection. | `strncpy(local_cmd, input, sizeof(local_cmd) - 1); local_cmd[31] = '\0';`     |
| **22** | **Heap/static buffer overflow via `memcpy`** | `strlen(name)` is the **source** length, not the **destination** size. Any `name` longer than 15 bytes overflows `device_name[16]`.                         | `memcpy(device_name, name, sizeof(device_name) - 1); device_name[15] = '\0';` |
| **28** | **Stack overflow via `sprintf`**             | `msg` is unbounded. A long `msg` string produces a formatted string longer than 128 bytes, overflowing `log_buf`.                                           | `snprintf(log_buf, sizeof(log_buf), "[%08X] %s\n", code, msg);`               |

</details>

---

#### 02. Audit Unsafe String Functions

Write a prompt for a security auditing agent that:

1. Scans a C source file for the following banned functions: `strcpy`, `strcat`, `sprintf`, `gets`, `scanf`, `memcpy` (when used without size bounds)
2. For each occurrence, reports: function name, line number, reason it is dangerous, safe replacement
3. Outputs results as a Markdown table

Also write the Python code that calls this prompt with the content of a given `.c` file.

#### Scenario:

◦ Code review of 10,000 lines of legacy firmware code before production sign-off.
◦ An agent can scan the entire codebase in seconds; a human would take days.

**Hint:** Include the full list of banned functions and their safe replacements in the system prompt. Pass the file content directly in the user message.

<details markdown>
<summary>Solution</summary>

**System prompt:**

```
You are a firmware security auditing agent.

Scan the provided C source code for the following BANNED functions:
  - strcpy  → use strncpy or strlcpy
  - strcat  → use strncat or strlcat
  - sprintf → use snprintf
  - gets    → use fgets
  - scanf   → use fgets + sscanf with length limit
  - memcpy  → flag only if no explicit destination size bound is provided

For each occurrence found, output a row in this Markdown table:
| Line | Function | Risk | Safe Replacement |
|------|----------|------|-----------------|

If no unsafe functions are found, output: "No unsafe functions detected."
Do NOT explain or add text outside the table.
```

**Python caller:**

````python
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage
import pathlib

AUDIT_SYSTEM = """... (system prompt above) ..."""

llm = ChatOpenAI(model="gpt-4o", temperature=0)

def audit_c_file(filepath: str) -> str:
    source = pathlib.Path(filepath).read_text()
    messages = [
        SystemMessage(content=AUDIT_SYSTEM),
        HumanMessage(content=f"Audit this C source file:\n\n```c\n{source}\n```"),
    ]
    response = llm.invoke(messages)
    return response.content

if __name__ == "__main__":
    import sys
    report = audit_c_file(sys.argv[1])
    print(report)
````

</details>

---

#### 03. Detect Power-Wasting Anti-Patterns

Analyze the following BLE peripheral firmware and identify every power-wasting anti-pattern. For each, state: the pattern name, the impact in milliwatts or milliamp-hours, and the recommended fix.

```c
// Main loop - BLE peripheral running on CR2032 battery
while (1) {
    // 1. Poll BLE event register every 1ms
    if (BLE_Read_Events() != 0) {
        BLE_Process_Events();
    }
    HAL_Delay(1);

    // 2. ADC reads continuously even when no change expected
    uint16_t val = ADC_Read_Channel(3);
    if (val != last_val) {
        BLE_Notify(val);
        last_val = val;
    }

    // 3. RF transmit power set to maximum
    BLE_Set_TxPower(+8);   // dBm

    // 4. CPU clock at 64 MHz at all times
    SystemClock_Set(64000000);
}
```

#### Scenario:

◦ A battery-powered sensor is depleting its CR2032 (240 mAh) in 3 days instead of the expected 6 months.
◦ The agent audits the code to find why power consumption is ~100× too high.

**Hint:** BLE best practice: use connection events + sleep between events. ADC: use threshold interrupt. TX power: minimum that achieves required range.

<details markdown>
<summary>Solution</summary>

| #   | Anti-Pattern                         | Power Impact                                                                                                                      | Fix                                                                                                                                                    |
| --- | ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | **BLE event polling + HAL_Delay(1)** | CPU active 100% of the time (16-25 mA typical on Cortex-M0+). CR2032 drains in < 10 hours.                                        | Use BLE connection events: enter `WFE/WFI` or `__WFI()` between events. CPU active only during BLE connection interval (e.g., 100 ms → 1% duty cycle). |
| 2   | **Continuous ADC polling**           | ADC in continuous mode: ~0.3-1 mA. For slow-changing sensors, ADC should sample at 1 Hz or use a comparator to wake on threshold. | Configure ADC window watchdog: wake only when signal crosses a threshold. Average ADC current drops from 1 mA to ~10 µA.                               |
| 3   | **TX power = +8 dBm always**         | Every 3 dBm reduction halves transmit current. +8 dBm → ~18 mA active TX; -4 dBm → ~4.5 mA (75% reduction).                       | Use the minimum TX power that achieves the required RSSI margin. Start at 0 dBm and reduce until margin < 10 dB is violated.                           |
| 4   | **CPU at 64 MHz continuously**       | At 64 MHz: ~10-15 mA. At 4 MHz (necessary during BLE events only): ~1-2 mA.                                                       | Use dynamic frequency scaling: run at 4 MHz in sleep mode, boost to 64 MHz only during BLE event processing window (~1 ms).                            |

**Combined fix savings:** Estimated current drops from ~50 mA average to ~0.05 mA → CR2032 lifetime increases from 5 hours to ~200 days.

</details>

---

#### 04. Calculate Power Impact of a Bug

Given the following parameters, calculate the extra energy consumed per day by the power-wasting main loop from Task 03 compared to the optimized implementation.

**Parameters:**

| Metric          | Inefficient | Optimized |
| --------------- | ----------- | --------- |
| Average current | 50 mA       | 0.05 mA   |
| Supply voltage  | 3.0 V       | 3.0 V     |
| CR2032 capacity | 240 mAh     | 240 mAh   |

**Questions:**

1. What is the daily energy consumption (mWh) of each implementation?
2. What is the battery lifetime (hours) of each?
3. By what factor does the optimization extend battery life?

#### Scenario:

◦ Before presenting the audit results to management, the agent calculates the business impact in concrete numbers.
◦ "Battery life extended 1000×" is far more persuasive than "we removed a polling loop".

**Hint:** Energy = Power × time = V × I × t. Lifetime = Capacity / Average_Current.

<details markdown>
<summary>Solution</summary>

**Daily energy consumption:**

```
Inefficient: P = V × I = 3.0 V × 0.050 A = 150 mW
             E/day = 150 mW × 24 h = 3,600 mWh = 3.6 Wh/day

Optimized:   P = 3.0 V × 0.00005 A = 0.15 mW
             E/day = 0.15 mW × 24 h = 3.6 mWh/day
```

**Battery lifetime:**

```
Inefficient: 240 mAh / 50 mA = 4.8 hours
Optimized:   240 mAh / 0.05 mA = 4,800 hours ≈ 200 days
```

**Improvement factor:**

```
4,800 h / 4.8 h = 1,000×
```

**Summary:**

- The power-wasting implementation drains the CR2032 in **< 5 hours**.
- The optimized implementation achieves **200 days** of battery life on the same cell.
- The optimization represents a **1,000× improvement** in battery lifetime.

</details>

---

#### 05. Build an Automated Security Scanner

Build a Python security scanner that:

1. Accepts a directory path as input
2. Recursively finds all `.c` and `.h` files
3. Runs two agents in parallel: the **buffer overflow scanner** (Task 01 prompt) and the **unsafe function scanner** (Task 02 prompt)
4. Aggregates results into a single Markdown security report
5. Exits with code `1` if any HIGH severity issues are found, `0` if only LOW/MEDIUM or none

#### Scenario:

◦ This scanner is the pre-commit hook / CI gate for a production firmware repository.
◦ Any HIGH severity finding blocks the merge until resolved.

**Hint:** Use `asyncio.gather()` or `concurrent.futures.ThreadPoolExecutor` to call both LLM agents concurrently per file.

<details markdown>
<summary>Solution</summary>

````python
import pathlib
import sys
import concurrent.futures
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage

llm = ChatOpenAI(model="gpt-4o", temperature=0)

OVERFLOW_SYSTEM = """
Scan the following C code for buffer overflow vulnerabilities.
Output: Markdown table with columns: Line | Vulnerability | Severity (high/medium/low) | Fix
If none found, output: "No buffer overflows detected."
No explanation outside the table.
"""

UNSAFE_FN_SYSTEM = """
Scan the following C code for unsafe string/memory functions:
strcpy, strcat, sprintf, gets, scanf, unbounded memcpy.
Output: Markdown table with columns: Line | Function | Risk | Safe Replacement
If none found, output: "No unsafe functions detected."
No explanation outside the table.
"""

def scan_file(filepath: pathlib.Path) -> dict:
    source = filepath.read_text(errors="replace")
    with concurrent.futures.ThreadPoolExecutor(max_workers=2) as ex:
        overflow_future = ex.submit(
            llm.invoke,
            [SystemMessage(content=OVERFLOW_SYSTEM),
             HumanMessage(content=f"```c\n{source}\n```")]
        )
        unsafe_future = ex.submit(
            llm.invoke,
            [SystemMessage(content=UNSAFE_FN_SYSTEM),
             HumanMessage(content=f"```c\n{source}\n```")]
        )
    return {
        "file":     str(filepath),
        "overflow": overflow_future.result().content,
        "unsafe":   unsafe_future.result().content,
    }

def main(directory: str) -> int:
    files = list(pathlib.Path(directory).rglob("*.c")) + \
            list(pathlib.Path(directory).rglob("*.h"))

    if not files:
        print("No C/H files found.")
        return 0

    report_lines = ["# Security Scan Report\n"]
    has_high = False

    for fpath in files:
        print(f"Scanning {fpath}...")
        result = scan_file(fpath)
        report_lines.append(f"\n## {result['file']}\n")
        report_lines.append("### Buffer Overflow Scan\n" + result["overflow"])
        report_lines.append("\n### Unsafe Function Scan\n" + result["unsafe"])

        # Simple HIGH detection (check if "high" appears in results)
        combined = result["overflow"].lower() + result["unsafe"].lower()
        if "high" in combined:
            has_high = True

    report = "\n".join(report_lines)
    pathlib.Path("security_report.md").write_text(report)
    print("\nReport written to security_report.md")

    if has_high:
        print("FAILED: HIGH severity issues found.")
        return 1
    print("PASSED: No HIGH severity issues.")
    return 0

if __name__ == "__main__":
    sys.exit(main(sys.argv[1] if len(sys.argv) > 1 else "."))
````

</details>

---

> **Back to [Tasks Index](../index.md)** | **Next: [009 Tasks](../009-VibeCodingForHW-Tasks/README.md)**
