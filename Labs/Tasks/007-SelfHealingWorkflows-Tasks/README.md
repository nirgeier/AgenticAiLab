# Lab 007 - Self-Healing Workflows Tasks

- Exercises covering GCC tool wrapping, error parsing, iterative self-healing loops, retry guards, and multi-phase builds.
- Each task includes a description, scenario, hint, and collapsible Solution.

#### Table of Contents

- [01. Wrap arm-none-eabi-gcc as an Agent Tool](#01-wrap-arm-none-eabi-gcc-as-an-agent-tool)
- [02. Build an Error Parser](#02-build-an-error-parser)
- [03. Implement the Healing Iteration Loop](#03-implement-the-healing-iteration-loop)
- [04. Add the Max-Retry Guard](#04-add-the-max-retry-guard)
- [05. Extend to the Linker Phase](#05-extend-to-the-linker-phase)

---

#### 01. Wrap arm-none-eabi-gcc as an Agent Tool

Wrap the `arm-none-eabi-gcc` compiler as a Python callable that:

1. Accepts: `source_code` (string), `target` (default `"cortex-m4"`)
2. Writes `source_code` to a temp file
3. Runs `arm-none-eabi-gcc -mcpu={target} -mthumb -Wall -Wextra -c {tmpfile}`
4. Returns a dict `{"success": bool, "errors": list[str], "warnings": list[str]}`
5. Cleans up the temp file after compilation

#### Scenario:

◦ The self-healing agent cannot fix code it cannot compile. Wrapping the compiler as a clean Python function is the foundation of the loop.
◦ The function must be safe: no shell injection, controlled temp-file lifecycle.

**Hint:** Use `subprocess.run()` with `capture_output=True`. Split stderr on newlines. Do NOT use `shell=True`.

<details markdown>
<summary>Solution</summary>

```python
import subprocess
import tempfile
import os
from typing import TypedDict, List

class CompileResult(TypedDict):
    success:  bool
    errors:   List[str]
    warnings: List[str]

def compile_firmware(source_code: str, target: str = "cortex-m4") -> CompileResult:
    """
    Compile source_code for the given ARM Cortex target.
    Returns a structured result dict.
    """
    # Write source to a temp .c file
    with tempfile.NamedTemporaryFile(suffix=".c", mode="w", delete=False) as f:
        f.write(source_code)
        tmp_path = f.name

    try:
        result = subprocess.run(
            [
                "arm-none-eabi-gcc",
                f"-mcpu={target}",
                "-mthumb",
                "-Wall",
                "-Wextra",
                "-c",
                tmp_path,
                "-o", os.devnull,   # discard object file
            ],
            capture_output=True,
            text=True,
        )

        errors   = []
        warnings = []

        for line in result.stderr.splitlines():
            if ": error:" in line:
                errors.append(line)
            elif ": warning:" in line:
                warnings.append(line)

        return CompileResult(
            success=result.returncode == 0,
            errors=errors,
            warnings=warnings,
        )
    finally:
        os.unlink(tmp_path)
```

</details>

---

#### 02. Build an Error Parser

Implement a `parse_gcc_errors()` function that extracts structured error records from raw GCC stderr output.

Each error record must be a dict with:

- `file`: filename (string)
- `line`: line number (int)
- `column`: column number (int)
- `severity`: `"error"` or `"warning"`
- `message`: the error message text

**Test it against this sample stderr:**

```
/tmp/fw_agent_abc.c:14:5: error: implicit declaration of function 'HAL_Delay' [-Wimplicit-function-declaration]
/tmp/fw_agent_abc.c:22:12: warning: unused variable 'timeout' [-Wunused-variable]
/tmp/fw_agent_abc.c:31:1: error: expected ';' before '}' token
```

#### Scenario:

◦ The raw stderr string from GCC is not useful to an LLM unless it is structured.
◦ A proper parser turns compiler output into a list the LLM can reason about and fix.

**Hint:** Use `re.findall` with a pattern that captures `(file, line, col, severity, message)`.

<details markdown>
<summary>Solution</summary>

```python
import re
from typing import List, TypedDict

class GCCError(TypedDict):
    file:     str
    line:     int
    column:   int
    severity: str
    message:  str

GCC_PATTERN = re.compile(
    r"^(.+?):(\d+):(\d+):\s+(error|warning):\s+(.+)$",
    re.MULTILINE
)

def parse_gcc_errors(stderr: str) -> List[GCCError]:
    records = []
    for match in GCC_PATTERN.finditer(stderr):
        file_, line, col, sev, msg = match.groups()
        records.append(GCCError(
            file=file_,
            line=int(line),
            column=int(col),
            severity=sev,
            message=msg,
        ))
    return records

# --- Test ---
sample = """\
/tmp/fw_agent_abc.c:14:5: error: implicit declaration of function 'HAL_Delay' [-Wimplicit-function-declaration]
/tmp/fw_agent_abc.c:22:12: warning: unused variable 'timeout' [-Wunused-variable]
/tmp/fw_agent_abc.c:31:1: error: expected ';' before '}' token
"""

parsed = parse_gcc_errors(sample)
for e in parsed:
    print(e)

# Expected output:
# {'file': '/tmp/fw_agent_abc.c', 'line': 14, 'column': 5,  'severity': 'error',   'message': "implicit declaration of function 'HAL_Delay' [-Wimplicit-function-declaration]"}
# {'file': '/tmp/fw_agent_abc.c', 'line': 22, 'column': 12, 'severity': 'warning', 'message': "unused variable 'timeout' [-Wunused-variable]"}
# {'file': '/tmp/fw_agent_abc.c', 'line': 31, 'column': 1,  'severity': 'error',   'message': "expected ';' before '}' token"}
```

</details>

---

#### 03. Implement the Healing Iteration Loop

Implement the core `self_healing_compile()` loop that:

1. Takes an initial `source_code` string and a `task_description`
2. Compiles using `compile_firmware()` from Task 01
3. If compilation fails, sends the errors to the LLM to obtain a fixed version
4. Repeats until build succeeds or max iterations are reached
5. Returns `{"success": bool, "final_code": str, "iterations": int}`

#### Scenario:

◦ The self-healing loop is the core value of Lab 007 - code that fixes itself.
◦ On average, firmware agents need 1-3 iterations to produce clean compilation.

**Hint:** Send the original task + current source code + error list to the LLM in each iteration. Ask it to return ONLY the fixed C code.

<details markdown>
<summary>Solution</summary>

````python
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage

llm = ChatOpenAI(model="gpt-4o", temperature=0)

FIX_SYSTEM = """
You are a firmware code repair agent.
You will receive: the original task, the current C source code, and a list of compiler errors.
Your response must be ONLY the corrected C source code, with no markdown fences, no explanation.
"""

def self_healing_compile(
    source_code: str,
    task_description: str,
    max_iterations: int = 5,
) -> dict:
    current_code = source_code
    for iteration in range(1, max_iterations + 1):
        result = compile_firmware(current_code)

        if result["success"]:
            return {
                "success":    True,
                "final_code": current_code,
                "iterations": iteration,
            }

        # Format errors for the LLM
        error_text = "\n".join(result["errors"] + result["warnings"])

        messages = [
            SystemMessage(content=FIX_SYSTEM),
            HumanMessage(content=(
                f"Original task:\n{task_description}\n\n"
                f"Current code:\n```c\n{current_code}\n```\n\n"
                f"Compiler errors:\n{error_text}"
            ))
        ]
        response = llm.invoke(messages)
        current_code = response.content.strip()

    return {
        "success":    False,
        "final_code": current_code,
        "iterations": max_iterations,
    }
````

</details>

---

#### 04. Add the Max-Retry Guard

Extend `self_healing_compile()` from Task 03 with a max-retry guard that:

1. After reaching `max_iterations`, does NOT silently return the broken code
2. Instead, calls a `notify_human(task, final_code, error_history)` function with the full history
3. Raises a `HealingExhaustedError` exception
4. The `notify_human` function should print a structured summary to stdout (or post to a webhook)

Also: add a `convergence_check` - if the same error appears 3 times in a row (detected by comparing consecutive error lists), give up immediately.

#### Scenario:

◦ A runaway self-healing loop that never converges wastes LLM API credits and delays the CI pipeline.
◦ Early exit + human notification prevents silent infinite iteration.

**Hint:** Store each iteration's error list in a `history` list. Compare the last 3 entries with `==`.

<details markdown>
<summary>Solution</summary>

````python
class HealingExhaustedError(Exception):
    """Raised when self-healing loop exhausts all retry attempts."""

def notify_human(task: str, final_code: str, error_history: list) -> None:
    print("=" * 60)
    print("SELF-HEALING EXHAUSTED - Human intervention required")
    print(f"Task: {task}")
    print(f"Iterations attempted: {len(error_history)}")
    print(f"Last errors:\n{chr(10).join(error_history[-1])}")
    print("Final (broken) code available for inspection.")
    print("=" * 60)
    # Optional: post to webhook
    # requests.post(WEBHOOK_URL, json={"task": task, "errors": error_history[-1]})

def self_healing_compile(
    source_code: str,
    task_description: str,
    max_iterations: int = 5,
) -> dict:
    current_code  = source_code
    error_history = []

    for iteration in range(1, max_iterations + 1):
        result = compile_firmware(current_code)

        if result["success"]:
            return {
                "success":    True,
                "final_code": current_code,
                "iterations": iteration,
            }

        errors = result["errors"] + result["warnings"]
        error_history.append(errors)

        # Convergence check: same errors 3 times in a row → give up early
        if len(error_history) >= 3 and (
            error_history[-1] == error_history[-2] == error_history[-3]
        ):
            notify_human(task_description, current_code, error_history)
            raise HealingExhaustedError(
                f"Errors unchanged for 3 consecutive iterations - aborting."
            )

        # Ask LLM to fix
        error_text = "\n".join(errors)
        messages   = [
            SystemMessage(content=FIX_SYSTEM),
            HumanMessage(content=(
                f"Original task:\n{task_description}\n\n"
                f"Current code:\n```c\n{current_code}\n```\n\n"
                f"Compiler errors:\n{error_text}"
            ))
        ]
        current_code = llm.invoke(messages).content.strip()

    notify_human(task_description, current_code, error_history)
    raise HealingExhaustedError(f"Max {max_iterations} iterations reached.")
````

</details>

---

#### 05. Extend to the Linker Phase

Extend the workflow to include a **linker phase** after successful compilation.

Requirements:

1. After `compile_firmware()` succeeds, call `link_firmware(object_files, linker_script)` that runs `arm-none-eabi-gcc` with `-Wl,-T` to produce an ELF
2. Parse linker errors with the same `parse_gcc_errors()` function
3. If linker errors reference undefined symbols, send the list to the LLM asking it to add missing function stubs
4. Integrate both compilation and linking into a two-phase `build_and_link()` pipeline

#### Scenario:

◦ Compilation success is not the same as link success. Unresolved symbols are a common embedded linker error.
◦ The full pipeline must handle both phases to produce an executable ELF.

**Hint:** Common linker errors: `undefined reference to 'HAL_Init'`. The LLM can add `void HAL_Init(void) {}` stubs for missing symbols.

<details markdown>
<summary>Solution</summary>

```python
import tempfile, os, subprocess

def link_firmware(
    object_files:  List[str],
    linker_script: str,
    target:        str = "cortex-m4",
) -> CompileResult:
    """Link object files with the given linker script to produce an ELF."""
    elf_path = tempfile.mktemp(suffix=".elf")
    try:
        result = subprocess.run(
            [
                "arm-none-eabi-gcc",
                f"-mcpu={target}",
                "-mthumb",
                f"-Wl,-T,{linker_script}",
                *object_files,
                "-o", elf_path,
            ],
            capture_output=True,
            text=True,
        )
        errors   = [l for l in result.stderr.splitlines() if ": error:" in l]
        warnings = [l for l in result.stderr.splitlines() if ": warning:" in l]
        return CompileResult(success=result.returncode == 0, errors=errors, warnings=warnings)
    finally:
        if os.path.exists(elf_path):
            os.unlink(elf_path)

ADD_STUBS_SYSTEM = """
You are a firmware stub generator.
Given a list of undefined symbol names from a linker error output,
add empty stub functions at the end of the provided C source code.
Return ONLY the complete modified C source with stubs appended.
No markdown fences, no explanation.
"""

def build_and_link(
    source_code:   str,
    task_desc:     str,
    linker_script: str,
    max_iter:      int = 3,
) -> dict:
    # Phase 1: compile
    heal_result = self_healing_compile(source_code, task_desc, max_iter)
    if not heal_result["success"]:
        return heal_result

    # Write final source to a temp .c and compile to .o
    with tempfile.NamedTemporaryFile(suffix=".c", mode="w", delete=False) as f:
        f.write(heal_result["final_code"])
        src_path = f.name
    obj_path = src_path.replace(".c", ".o")

    subprocess.run(
        ["arm-none-eabi-gcc", "-mcpu=cortex-m4", "-mthumb", "-c", src_path, "-o", obj_path],
        capture_output=True,
    )

    # Phase 2: link
    for iteration in range(max_iter):
        link_result = link_firmware([obj_path], linker_script)
        if link_result["success"]:
            os.unlink(src_path); os.unlink(obj_path)
            return {"success": True, "final_code": heal_result["final_code"]}

        # Extract undefined symbols
        undef_syms = [
            e.split("undefined reference to ")[-1].strip("'")
            for e in link_result["errors"]
            if "undefined reference to" in e
        ]
        messages = [
            SystemMessage(content=ADD_STUBS_SYSTEM),
            HumanMessage(content=(
                f"Source code:\n{heal_result['final_code']}\n\n"
                f"Undefined symbols: {undef_syms}"
            ))
        ]
        heal_result["final_code"] = llm.invoke(messages).content.strip()

    os.unlink(src_path); os.unlink(obj_path)
    return {"success": False, "final_code": heal_result["final_code"]}
```

</details>

---

> **Back to [Tasks Index](../index.md)** | **Next: [008 Tasks](../008-SecuritySafety-Tasks/README.md)**
