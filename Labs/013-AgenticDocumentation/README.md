# Lab 013 - Agentic Documentation

!!! hint "Overview"

    - Applied Agents for Hardware-Software R&D** - Agentic Documentation module.
    - In this lab, you will build an agent that **keeps firmware source code and technical design documents synchronized**.
    - As firmware evolves, documentation drifts. This agent detects the drift and either regenerates the outdated sections or flags them for engineer review.
    - By the end of this lab, you will have a living-documentation agent that runs as a pre-commit hook or CI step to ensure docs never fall behind code.

## Prerequisites

- Completed [Lab 012 - ReAct Patterns for HW-SW Debugging](../012-ReActPatterns/README.md)
- Python 3.10+ and `gitpython` installed

```bash
pip install gitpython openai
```

## What You Will Learn

- How to detect drift between a header file and its corresponding design document
- How to prompt agents to regenerate specific documentation sections from code
- How to build a git-diff-aware documentation agent
- How to integrate the agent as a pre-commit or CI check

---

## Background

### The Documentation Drift Problem

Firmware documentation has a short half-life:

```
Week 1:  SPI_Init() takes 3 parameters       ← docs correct
Week 3:  SPI_Init() now takes 4 parameters   ← code updated, docs forgotten
Week 8:  New engineer reads docs, wastes 2h debugging why 3-param call fails
```

A documentation agent solves this by:

1. Watching the diff of changed `.c` / `.h` files
2. Identifying which sections of the design document reference those symbols
3. Regenerating the affected documentation sections automatically

---

## Lab Steps

### Step 1 - Detect Stale Documentation

Given a header file and its design document, ask the agent to find mismatches:

**Header file (`spi.h`):**

```c
/**
 * @brief Initialize SPI1 in Master mode.
 * @param config   Pointer to SPI configuration struct
 * @param dma_mode Enable DMA TX/RX (true/false)
 * @param cs_gpio  GPIO port for chip select
 * @param cs_pin   Chip select pin number
 * @return HAL_StatusTypeDef - HAL_OK on success, HAL_ERROR on failure
 */
HAL_StatusTypeDef SPI1_Init(SPI_Config_t *config, bool dma_mode,
                             GPIO_TypeDef *cs_gpio, uint16_t cs_pin);
```

**Design document section (`TDD-SPI.md`):**

```markdown
## 2.3 SPI1 Initialization

The SPI1 peripheral is initialized by calling `SPI1_Init(config, dma_mode)`.
The function takes two parameters:

- `config`: pointer to the SPI configuration structure
- `dma_mode`: boolean flag to enable DMA transfers

Returns `HAL_OK` on success.
```

**Agent Prompt:**

```
You are a firmware documentation sync agent.
Compare the following C header declaration with its corresponding design document section.

Identify:
1. Every parameter in the code that is NOT mentioned in the document
2. Every parameter in the document that no longer exists in the code
3. Any differences in return type, return values, or behavior description

Then generate a corrected version of the document section that accurately reflects
the current function signature. Use the same Markdown heading style and structure.

[paste header and design doc sections]
```

### Step 2 - Git-Diff-Aware Documentation Scanner

```python
import git
import openai
import pathlib
import re

client = openai.OpenAI()

def get_changed_files(repo_path: str, base_ref: str = "HEAD~1") -> list[str]:
    """Get list of changed .c and .h files between HEAD and base_ref."""
    repo = git.Repo(repo_path)
    diffs = repo.head.commit.diff(base_ref)
    return [
        diff.a_path for diff in diffs
        if diff.a_path and (diff.a_path.endswith(".c") or diff.a_path.endswith(".h"))
    ]

def extract_public_api(header_content: str) -> str:
    """Extract function declarations from a header file."""
    prompt = (
        "Extract all public function declarations from the following C header file. "
        "Return them as a structured list with: function name, parameters, return type, "
        "and Doxygen comment if present.\n\n"
        + header_content
    )
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content

def check_doc_for_function(doc_content: str, function_api: str) -> str:
    """Check if the design document accurately describes the current API."""
    prompt = f"""
You are a documentation accuracy checker.
Given the current C function API and an existing design document,
identify any outdated, missing, or incorrect information in the document.

Current API:
{function_api}

Design Document:
{doc_content}

Return:
1. A list of specific inaccuracies (or "ACCURATE" if no issues found)
2. If there are inaccuracies, provide a corrected replacement for the affected section only.
"""
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content

def scan_documentation_drift(repo_path: str, docs_dir: str) -> None:
    """Scan changed header files for documentation drift."""
    changed = get_changed_files(repo_path)
    headers = [f for f in changed if f.endswith(".h")]

    if not headers:
        print("No header files changed - documentation check skipped.")
        return

    root = pathlib.Path(repo_path)
    docs = pathlib.Path(docs_dir)

    for header_path in headers:
        header_file = root / header_path
        if not header_file.exists():
            continue

        header_name = header_file.stem
        doc_file = docs / f"TDD-{header_name.upper()}.md"

        if not doc_file.exists():
            print(f"[MISSING DOC] No design document found for {header_path}")
            print(f"  Expected: {doc_file}")
            continue

        print(f"\n[CHECKING] {header_path} vs {doc_file.name}")
        api = extract_public_api(header_file.read_text())
        result = check_doc_for_function(doc_file.read_text(), api)

        if "ACCURATE" in result:
            print(f"  [OK] Documentation is up to date.")
        else:
            print(f"  [DRIFT DETECTED]\n{result}")

if __name__ == "__main__":
    scan_documentation_drift(
        repo_path=".",
        docs_dir="docs/design"
    )
```

### Step 3 - Auto-Generate Doxygen from Implementation

Ask the agent to generate missing Doxygen comments from function implementations:

```
You are a firmware documentation agent.
The following C functions are missing Doxygen comments.
For each function, generate a complete Doxygen block comment that includes:
- @brief (one sentence)
- @param for each parameter (name, type context, expected range if applicable)
- @return describing all possible return values
- @note for any important timing, side-effect, or hardware constraint
- @warning if the function is not interrupt-safe or has reentrancy restrictions

[paste function implementations here]
```

### Step 4 - Pre-Commit Hook Integration

Create `.git/hooks/pre-commit`:

```bash
#!/bin/bash
# Pre-commit hook: check for documentation drift

echo "[pre-commit] Checking documentation sync..."
python scripts/check_doc_drift.py .

if [ $? -ne 0 ]; then
    echo ""
    echo "[pre-commit] BLOCKED: Documentation drift detected."
    echo "  Run 'python scripts/check_doc_drift.py --fix .' to auto-update docs."
    exit 1
fi

echo "[pre-commit] Documentation is up to date."
```

```bash
chmod +x .git/hooks/pre-commit
```

---

## Summary

| Skill                   | What You Practiced                                        |
| ----------------------- | --------------------------------------------------------- |
| Drift detection         | Comparing function signature against design doc section   |
| Git-diff-aware scanning | Only checking files changed in the current commit         |
| Auto-regeneration       | Agent rewrites outdated doc section to match current code |
| Pre-commit integration  | Blocking commits when docs are out of sync                |

---

> **Next Lab:** [014 - Constraint-Based Code Generation](../014-ConstraintBasedGeneration/README.md)
