# Lab 014 - Constraint-Based Code Generation

!!! hint "Overview"

    - Applied Agents for Hardware-Software R&D** - Constraint-Based Generation module.
    - In this lab, you will learn to prompt agents with **strict hardware limits** and have them generate code that provably respects those constraints.
    - Constraints include: SRAM budget, flash footprint, stack depth, interrupt latency budget, and power envelope.
    - By the end of this lab, you will be able to specify a complete set of hardware constraints and receive agent-generated code with a verification report showing each constraint is satisfied.

## Prerequisites

- Completed [Lab 013 - Agentic Documentation](../013-AgenticDocumentation/README.md)

## What You Will Learn

- How to express hardware constraints formally in agent prompts
- How to request constraint compliance reports alongside generated code
- How to use `__attribute__((section))` and linker map analysis to verify memory footprint
- How to build a constraint-checking pipeline that rejects non-compliant generated code

---

## Background

### Why Constraints Matter for Firmware

In application software, performance is measured in milliseconds. In firmware, it is measured in microseconds - and the boundaries are hard:

| Constraint             | Consequence of Violation                          |
| ---------------------- | ------------------------------------------------- |
| SRAM overflow          | Stack collision, silent data corruption           |
| Flash overflow         | Linker error - firmware won't fit on MCU          |
| Stack overflow         | Hard fault, undefined behavior                    |
| ISR latency exceeded   | Real-time deadline missed, motor/actuator failure |
| Power budget exceeded  | Battery lifetime collapsed, certification failed  |
| DMA alignment violated | DMA transfer silent failure or corruption         |

An agent that generates code without these constraints can produce functionally correct but physically undeployable firmware.

### The Constraint Contract

For every code generation task, you should provide:

```
CONSTRAINTS:
- Flash budget:         MAX 8 KB for this module (currently 512 KB total, 498 KB used)
- SRAM budget:          MAX 512 bytes static, MAX 256 bytes stack (in any call path)
- ISR latency:          ISR must complete in < 200 CPU cycles at 168 MHz
- DMA buffer alignment: Must be 4-byte aligned
- No dynamic allocation: malloc/calloc/realloc are FORBIDDEN
- No floating point:    FPU not available on this target (Cortex-M0)
```

---

## Lab Steps

### Step 1 - Constrained UART Driver Generation

```
You are a firmware code generation agent with strict hardware constraints.

Task: Generate a complete UART1 driver for a Cortex-M0+ microcontroller.

HARD CONSTRAINTS - the generated code MUST satisfy ALL of these:
1. Flash footprint: the entire driver must compile to < 1 KB of .text section
2. SRAM: no static buffers larger than 128 bytes total; no malloc/calloc
3. Stack depth: no call chain deeper than 3 functions (main → driver_fn → helper only)
4. No floating-point operations (Cortex-M0+ has no FPU)
5. ISR must complete in < 500 cycles at 48 MHz (≈ 10 µs)
6. Re-entrant safe: the driver must be safe to call from both task and ISR context

After generating the code, produce a CONSTRAINT COMPLIANCE REPORT:
For each constraint above:
- State: PASS / FAIL / CANNOT VERIFY (with reason)
- Evidence: specific code line or measurement that demonstrates compliance
```

### Step 2 - Memory Section Placement Constraints

```
Generate the following for an STM32F4 firmware module that must run entirely from SRAM
during firmware update (XIP from SRAM, not flash):

Constraints:
- All functions in this module must be decorated with __attribute__((section(".sram_code")))
- All data must be in __attribute__((section(".sram_data")))
- The entire module (code + data) must fit in 4 KB
- No calls to functions outside this module are allowed (no external dependencies)
- The module entry point is: int FWUpdate_Flash(uint32_t dest_addr, const uint8_t *src, size_t len)

Generate:
1. fw_update.h - module API
2. fw_update.c - implementation with correct section attributes
3. A fragment of the linker script that defines .sram_code and .sram_data regions
4. A build command to verify the module size with arm-none-eabi-size
```

### Step 3 - Stack Depth Analysis

Ask the agent to generate code with explicit stack depth guarantees:

```
The following firmware module will run on a system with a 512-byte ISR stack shared
across 4 interrupt handlers. The highest-priority ISR may call into this module.

Generate a CRC calculation module with these constraints:
1. Maximum stack usage of this module: 64 bytes maximum (to leave 448 bytes for other ISRs)
2. No recursion of any kind
3. No local arrays - use register variables only for temporaries
4. The function signature must be: uint32_t CRC32_Compute(const uint8_t *data, size_t len)

After generating the code, prove the 64-byte stack bound by providing:
- A manual worst-case stack frame size calculation
- The GCC command to measure actual stack usage:
  arm-none-eabi-gcc -fstack-usage -fdump-rtl-expand ...
```

### Step 4 - Build a Constraint-Checking Pipeline

```python
import subprocess
import re
import openai
import pathlib
import tempfile

client = openai.OpenAI()

class HardwareConstraints:
    def __init__(self, max_flash_bytes: int, max_sram_bytes: int,
                 max_stack_bytes: int, no_malloc: bool = True, no_fpu: bool = False):
        self.max_flash_bytes = max_flash_bytes
        self.max_sram_bytes = max_sram_bytes
        self.max_stack_bytes = max_stack_bytes
        self.no_malloc = no_malloc
        self.no_fpu = no_fpu

    def to_prompt_block(self) -> str:
        lines = [
            "HARD CONSTRAINTS:",
            f"- Flash budget: < {self.max_flash_bytes} bytes (.text section)",
            f"- SRAM budget: < {self.max_sram_bytes} bytes (.data + .bss)",
            f"- Stack depth: < {self.max_stack_bytes} bytes per call chain",
        ]
        if self.no_malloc:
            lines.append("- No dynamic memory allocation (malloc/calloc/realloc FORBIDDEN)")
        if self.no_fpu:
            lines.append("- No floating-point operations (no FPU on target)")
        return "\n".join(lines)

def compile_and_measure(c_source: str, mcu: str = "cortex-m0plus") -> dict:
    """Compile and measure binary size."""
    with tempfile.TemporaryDirectory() as tmpdir:
        src = pathlib.Path(tmpdir) / "module.c"
        obj = pathlib.Path(tmpdir) / "module.o"
        src.write_text(c_source)

        compile_result = subprocess.run(
            ["arm-none-eabi-gcc", f"-mcpu={mcu}", "-mthumb",
             "-Os", "-nostdlib", "-fstack-usage",
             "-c", "-o", str(obj), str(src)],
            capture_output=True, text=True
        )

        if compile_result.returncode != 0:
            return {"success": False, "error": compile_result.stderr}

        size_result = subprocess.run(
            ["arm-none-eabi-size", str(obj)],
            capture_output=True, text=True
        )

        size_match = re.search(r'(\d+)\s+(\d+)\s+(\d+)', size_result.stdout)
        text, data, bss = (int(size_match.group(i)) for i in (1, 2, 3)) if size_match else (0, 0, 0)

        return {
            "success": True,
            "flash_bytes": text,
            "sram_bytes": data + bss,
            "size_output": size_result.stdout
        }

def check_constraints(code: str, constraints: HardwareConstraints) -> list[dict]:
    """Check generated code against hardware constraints."""
    results = []
    measurements = compile_and_measure(code)

    if not measurements["success"]:
        return [{"constraint": "Compilation", "pass": False, "detail": measurements["error"]}]

    results.append({
        "constraint": f"Flash < {constraints.max_flash_bytes}B",
        "pass": measurements["flash_bytes"] < constraints.max_flash_bytes,
        "detail": f"Actual: {measurements['flash_bytes']} bytes"
    })

    results.append({
        "constraint": f"SRAM < {constraints.max_sram_bytes}B",
        "pass": measurements["sram_bytes"] < constraints.max_sram_bytes,
        "detail": f"Actual: {measurements['sram_bytes']} bytes"
    })

    if constraints.no_malloc:
        uses_malloc = any(kw in code for kw in ["malloc", "calloc", "realloc", "free"])
        results.append({
            "constraint": "No dynamic allocation",
            "pass": not uses_malloc,
            "detail": "malloc/calloc/realloc found in code" if uses_malloc else "Clean"
        })

    return results

def constrained_generation(task: str, constraints: HardwareConstraints) -> str:
    """Generate firmware code and verify it meets constraints."""
    prompt = f"{task}\n\n{constraints.to_prompt_block()}"

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "You are a firmware code generation agent. Return only raw C source code, no markdown."},
            {"role": "user", "content": prompt}
        ]
    )
    code = response.choices[0].message.content

    results = check_constraints(code, constraints)
    all_pass = all(r["pass"] for r in results)

    print(f"\n=== CONSTRAINT CHECK {'PASSED' if all_pass else 'FAILED'} ===")
    for r in results:
        status = "PASS" if r["pass"] else "FAIL"
        print(f"  [{status}] {r['constraint']}: {r['detail']}")

    return code

# Example usage
constraints = HardwareConstraints(
    max_flash_bytes=1024,
    max_sram_bytes=128,
    max_stack_bytes=64,
    no_malloc=True,
    no_fpu=True,
)

constrained_generation(
    task="Generate a CRC-16/CCITT checksum function for an 8-bit data buffer.",
    constraints=constraints
)
```

---

## Summary

| Skill                      | What You Practiced                                              |
| -------------------------- | --------------------------------------------------------------- |
| Constraint specification   | Expressing SRAM, flash, stack, and power limits as prompt rules |
| Compliance reporting       | Requesting and parsing agent-generated evidence for each limit  |
| Linker section placement   | `__attribute__((section(...)))` for SRAM execution              |
| Automated constraint check | Python pipeline that compiles, measures, and validates output   |

---

## Course Complete

Congratulations - you have completed all 14 modules of **Agentic AI for Firmware**!

| Modules | Skills Built                                         |
| ------- | ---------------------------------------------------- |
| 001-002 | Agent architecture, hardware context, tool concepts  |
| 003-005 | Register mapping, latency, JTAG debugging            |
| 006-008 | LangGraph, self-healing workflows, security scanning |
| 009-011 | Vibe coding, power refactoring, QEMU tool-use        |
| 012-014 | ReAct, living docs, constraint-based generation      |

---

> **Back to Start:** [001 - Introduction](../001-Introduction/README.md)
