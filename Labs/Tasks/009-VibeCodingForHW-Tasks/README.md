# Lab 009 - Vibe Coding for Hardware Tasks

- Exercises covering intent-driven prompting, architectural planning, cross-cutting refactoring, and build validation.
- Each task includes a description, scenario, hint, and collapsible Solution.

#### Table of Contents

- [01. Write an Architectural Intent Prompt](#01-write-an-architectural-intent-prompt)
- [02. Get a File-Level Change Plan](#02-get-a-file-level-change-plan)
- [03. Cross-Cutting Symbol Rename](#03-cross-cutting-symbol-rename)
- [04. Intent-to-Build Validation Loop](#04-intent-to-build-validation-loop)

---

#### 01. Write an Architectural Intent Prompt

Translate the following natural language engineering intent into a well-structured **Vibe Coding prompt** that an agent can act on autonomously.

**Engineering intent (informal):**

> "Our SPI driver is a mess — it's all in one 800-line file, everything is blocking, and we're having reliability issues with the CS pin timing. I want it split up properly so the HAL layer doesn't know anything about the business logic."

Your prompt must:

1. State the **goal** in one sentence
2. List the **architectural constraints** (what must not change, dependency direction, naming conventions)
3. Describe the **expected output structure** (which files, what each contains)
4. Define the **done condition** (how do you know the refactoring is complete?)

#### Scenario:

◦ A junior engineer submits the informal intent to the agent and gets a half-baked partial refactor.
◦ A senior engineer rewrites it as a structured vibe coding prompt and the agent produces all required files in one shot.

**Hint:** Structure your prompt as: Goal | Constraints | Output files | Done condition.

<details markdown>
<summary>Solution</summary>

```
Goal:
Refactor the monolithic SPI driver (spi_driver.c, 800 lines) into a layered
architecture: HAL abstraction layer, protocol driver layer, and application
adapter — with zero blocking calls in the HAL layer.

Architectural Constraints:
- The HAL layer (spi_hal.c / spi_hal.h) must contain ONLY register-level operations:
  SPI_HAL_Init(), SPI_HAL_Transfer(), SPI_HAL_SetCS(). No FreeRTOS or application logic.
- The protocol layer (spi_protocol.c / spi_protocol.h) may use FreeRTOS primitives
  (mutexes, queues) but must never reference business objects (sensor IDs, packet formats).
- The application adapter (sensor_spi.c / sensor_spi.h) may reference application
  objects and calls only the protocol layer API.
- CS pin timing must use a hardware timer delay, not HAL_Delay().
- No global variables — all state passed through handle structs.
- All existing test cases in test_spi.c must pass unchanged.

Expected output files:
  spi_hal.h        - HAL API declarations
  spi_hal.c        - Register-level implementation
  spi_protocol.h   - Protocol API declarations
  spi_protocol.c   - Interrupt/DMA + FreeRTOS implementation
  sensor_spi.h     - Application adapter API
  sensor_spi.c     - Application adapter implementation

Done condition:
- arm-none-eabi-gcc compiles all six files with zero errors and zero warnings (-Wall -Wextra)
- No occurrence of HAL_Delay() in spi_hal.c or spi_protocol.c
- No occurrence of application-layer types in spi_hal.c
```

</details>

---

#### 02. Get a File-Level Change Plan

Using the prompt from Task 01, ask the agent to produce a **file-level change plan** (not code yet):

1. List every file that will be created, modified, or deleted
2. For each file: state the key functions it will contain and their signatures
3. Identify any cross-cutting changes (e.g., every caller of `SPI_Transfer()` needs updating)
4. Estimate the number of call sites to update (search for `SPI_Transfer` in the codebase)

Then review the plan and identify any gaps the agent missed.

#### Scenario:

◦ "Vibe coding" without a plan leads to half-finished refactors that break the build.
◦ Getting a plan first allows the engineer to catch missed dependencies before any code is changed.

**Hint:** Ask the agent explicitly: "Before writing any code, produce a file-level change plan in table format."

<details markdown>
<summary>Solution</summary>

**Prompt to add to Task 01:**

```
Before generating any code, produce a file-level change plan in this format:

| Action  | File            | Key Functions / Changes                       |
|---------|-----------------|-----------------------------------------------|
| Create  | spi_hal.h       | SPI_HAL_Init(), SPI_HAL_Transfer(), ...       |
...

Then list:
1. All cross-cutting symbol changes (function renames, struct changes)
2. Every file that calls the current SPI_Transfer() that will need updating
3. Any header include paths that change
```

**Expected agent plan (representative):**

| Action | File             | Key Changes                                                                   |
| ------ | ---------------- | ----------------------------------------------------------------------------- |
| Create | `spi_hal.h`      | `SPI_HAL_Handle_t`, `SPI_HAL_Init()`, `SPI_HAL_Transfer()`, `SPI_HAL_SetCS()` |
| Create | `spi_hal.c`      | Register-level impl, DWT-based CS timing                                      |
| Create | `spi_protocol.h` | `SPI_Proto_Handle_t`, `SPI_Proto_Send()`, `SPI_Proto_Receive()`               |
| Create | `spi_protocol.c` | FreeRTOS mutex, DMA callbacks                                                 |
| Create | `sensor_spi.h`   | `Sensor_SPI_Read()`, `Sensor_SPI_Write()`                                     |
| Create | `sensor_spi.c`   | Application adapter                                                           |
| Modify | `main.c`         | Replace direct `SPI_Transfer()` calls (3 sites)                               |
| Modify | `sensor_task.c`  | Replace 5 call sites                                                          |
| Delete | `spi_driver.c`   | Superseded by the three new layers                                            |

**Agent-missed gap (common):**
`test_spi.c` was not included in the plan. Cross-cutting renames will break existing unit tests unless test mocks are updated too. Always check for test files explicitly.

</details>

---

#### 03. Cross-Cutting Symbol Rename

The current codebase uses the global prefix `SPI_Transfer()`. After the refactoring, all 23 call sites must be updated to `SPI_Proto_Send()`.

Write a Python script (or shell one-liner) that:

1. Finds all `.c` and `.h` files under `src/`
2. Replaces `SPI_Transfer(` with `SPI_Proto_Send(` (note: only the function call prefix, not declarations)
3. Prints: filename and line number for every replacement made
4. Does NOT modify generated files in `build/`

Validate by counting occurrences before and after.

#### Scenario:

◦ A manual find-and-replace across 23 call sites in 8 files is error-prone.
◦ The agent-assisted script performs all replacements atomically and audits every change.

**Hint:** Use `pathlib.Path.rglob()` and `str.replace()`. Count matches with `str.count()` before and after.

<details markdown>
<summary>Solution</summary>

```python
import pathlib
import sys

OLD_SYMBOL = "SPI_Transfer("
NEW_SYMBOL = "SPI_Proto_Send("
SRC_ROOT   = pathlib.Path(sys.argv[1]) if len(sys.argv) > 1 else pathlib.Path("src")

total_before = 0
total_after  = 0
changed_files = []

for filepath in sorted(SRC_ROOT.rglob("*.c")) + sorted(SRC_ROOT.rglob("*.h")):
    # Skip generated files
    if "build" in filepath.parts or "generated" in filepath.parts:
        continue

    original = filepath.read_text(errors="replace")
    count_before = original.count(OLD_SYMBOL)
    if count_before == 0:
        continue

    updated = original.replace(OLD_SYMBOL, NEW_SYMBOL)
    count_after = updated.count(OLD_SYMBOL)  # Should be 0

    # Report changes with line numbers
    for lineno, line in enumerate(original.splitlines(), start=1):
        if OLD_SYMBOL in line:
            print(f"{filepath}:{lineno}: {line.strip()}")

    filepath.write_text(updated)
    changed_files.append(str(filepath))
    total_before += count_before
    total_after  += updated.count(OLD_SYMBOL)

print(f"\nSummary: {total_before} occurrences replaced across {len(changed_files)} files.")
if total_after > 0:
    print(f"WARNING: {total_after} occurrences remain — check for macro expansions.")
else:
    print("All occurrences replaced successfully.")
```

**Shell one-liner alternative (macOS/BSD):**

```bash
grep -rn "SPI_Transfer(" src/ --include="*.c" --include="*.h" | \
  grep -v build/ | \
  while IFS=: read file line rest; do
    sed -i '' "s/SPI_Transfer(/SPI_Proto_Send(/g" "$file"
    echo "Updated $file:$line"
  done
```

</details>

---

#### 04. Intent-to-Build Validation Loop

Combine the workflows from Tasks 01–03 into a complete **intent-to-build pipeline** that:

1. Accepts an informal engineering intent (plain English)
2. Generates a structured vibe coding prompt (Task 01 style)
3. Gets a change plan (Task 02 style) and displays it for human review
4. On human approval, generates all the code
5. Runs the self-healing compile loop from Lab 007 on each generated file
6. Reports: number of files generated, compilation pass/fail per file, total iterations needed

#### Scenario:

◦ The full vibe coding pipeline is: Intent → Plan → Human Review → Code → Verify → Deploy.
◦ Without the verification step, "vibe coded" firmware ships with compile errors.

**Hint:** Reuse `self_healing_compile()` from Lab 007. Wrap stage 1–3 in a LangGraph plan node. Gate on a `human_approve()` checkpoint (Lab 006 pattern).

<details markdown>
<summary>Solution</summary>

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage

llm = ChatOpenAI(model="gpt-4o", temperature=0)

STRUCTURE_SYSTEM = """
You are an expert firmware architect.
Transform the given informal engineering intent into a structured vibe coding prompt
with four sections: Goal | Constraints | Output Files | Done Condition.
Be specific and actionable. Return only the structured prompt.
"""

PLAN_SYSTEM = """
Given a structured vibe coding prompt, produce a file-level change plan as a Markdown table.
Columns: Action (Create/Modify/Delete) | File | Key Functions / Changes
Add a "Cross-Cutting Changes" section listing all symbol renames and affected call sites.
"""

CODE_SYSTEM = """
You are a firmware code generation agent.
Given a structured vibe coding prompt, generate the complete implementation for ONE file at a time.
Return only C source code. No markdown fences. No explanation.
"""

def intent_to_build_pipeline(informal_intent: str) -> dict:
    # Stage 1: Structure the intent
    print("Stage 1: Structuring intent...")
    structured_prompt = llm.invoke([
        SystemMessage(content=STRUCTURE_SYSTEM),
        HumanMessage(content=informal_intent)
    ]).content

    # Stage 2: Get change plan
    print("Stage 2: Generating change plan...")
    change_plan = llm.invoke([
        SystemMessage(content=PLAN_SYSTEM),
        HumanMessage(content=structured_prompt)
    ]).content

    print("\n=== CHANGE PLAN ===\n")
    print(change_plan)

    # Stage 3: Human review gate
    approval = input("\nApprove this plan? [y/n]: ").strip().lower()
    if approval != "y":
        print("Plan rejected. Stopping pipeline.")
        return {"approved": False}

    # Stage 4 & 5: Generate code for each file and compile
    import re
    # Extract file names from plan (crude but effective for demo)
    file_matches = re.findall(r'`([^`]+\.[ch])`', change_plan)
    files_to_create = [f for f in file_matches if "Create" in change_plan.split(f)[0].split("\n")[-1]]

    results = {}
    for filename in files_to_create:
        print(f"\nGenerating {filename}...")
        code = llm.invoke([
            SystemMessage(content=CODE_SYSTEM),
            HumanMessage(content=f"Generate {filename} based on:\n{structured_prompt}")
        ]).content

        compile_result = self_healing_compile(
            source_code=code,
            task_description=f"Compile {filename} for firmware project",
            max_iterations=5,
        )
        results[filename] = compile_result
        status = "PASS" if compile_result["success"] else "FAIL"
        print(f"  {filename}: {status} (iterations: {compile_result['iterations']})")

    passed = sum(1 for r in results.values() if r["success"])
    print(f"\n=== BUILD SUMMARY: {passed}/{len(results)} files compiled successfully ===")
    return {"approved": True, "results": results}

if __name__ == "__main__":
    intent_to_build_pipeline(
        "Our SPI driver is a mess — it's all in one 800-line file, "
        "everything is blocking, and the CS pin timing is unreliable."
    )
```

</details>

---

> **Back to [Tasks Index](../index.md)** | **Next: [010 Tasks](../010-PowerSensitiveRefactoring-Tasks/README.md)**
