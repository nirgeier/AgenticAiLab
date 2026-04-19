# Lab 013 - Agentic Documentation Tasks

- Exercises covering documentation drift detection, API extraction, git-diff scanning, and Doxygen generation.
- Each task includes a description, scenario, hint, and collapsible Solution.

#### Table of Contents

- [01. Detect Documentation Drift](#01-detect-documentation-drift)
- [02. Extract an API Surface](#02-extract-an-api-surface)
- [03. Scan a Git Diff for Doc Impact](#03-scan-a-git-diff-for-doc-impact)
- [04. Generate Doxygen Comments](#04-generate-doxygen-comments)

---

#### 01. Detect Documentation Drift

The following C header file `spi_driver.h` and its accompanying `SPI_DRIVER.md` documentation have drifted out of sync. Identify every discrepancy between the two.

**spi_driver.h (current):**

```c
/**
 * @brief Initialize SPI1 in master mode.
 * @param baud_rate  Target baud rate in bps (must be a power of 2 divisor of APB2 clock).
 * @param mode       SPI clock mode: SPI_MODE_0 to SPI_MODE_3.
 * @return           0 on success, -1 if baud_rate is out of range.
 */
int SPI1_Init(uint32_t baud_rate, uint8_t mode);

/**
 * @brief Transfer a single byte over SPI1.
 * @param data   Byte to transmit.
 * @param cs_pin GPIO pin number for chip select (0-15).
 * @return       Received byte.
 */
uint8_t SPI1_Transfer(uint8_t data, uint8_t cs_pin);

/**
 * @brief Disable SPI1 and release the peripheral clock.
 */
void SPI1_DeInit(void);
```

**SPI_DRIVER.md (outdated documentation):**

```markdown
# SPI Driver API

## SPI1_Init(baud_rate)

Initialize SPI1. Takes one argument: the desired baud rate.
Returns 0 on success.

## SPI1_Write(data)

Write a byte to SPI1. No chip select management.

## SPI1_Read()

Read a byte from SPI1.

## SPI1_Reset()

Reset the SPI1 peripheral.
```

#### Scenario:

◦ The documentation was written 3 months ago. The API was refactored twice since then.
◦ An agent scanning for drift catches documentation debt before it confuses the next engineer.

**Hint:** Compare function names, parameter lists, and return values between the header and the doc. List additions, deletions, and mismatches.

<details markdown>
<summary>Solution</summary>

**Documentation drift findings:**

| Category                                    | Issue                                                                                                              |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| **Missing from docs**                       | `SPI1_Init()` now takes a second parameter `mode` - docs show only one parameter.                                  |
| **Missing from docs**                       | `SPI1_Init()` return value is `-1` on invalid baud rate - docs only mention `0 on success`.                        |
| **Missing from docs**                       | `SPI1_Transfer()` is the current function name - docs have `SPI1_Write()` and `SPI1_Read()` as separate functions. |
| **Missing from docs**                       | `SPI1_Transfer()` has a `cs_pin` parameter - no CS management exists in docs.                                      |
| **Missing from docs**                       | `SPI1_DeInit()` is present in header - docs call it `SPI1_Reset()` (wrong name).                                   |
| **Removed API (docs have, header doesn't)** | `SPI1_Write()` - replaced by `SPI1_Transfer()`.                                                                    |
| **Removed API (docs have, header doesn't)** | `SPI1_Read()` - replaced by `SPI1_Transfer()`.                                                                     |
| **Removed API (docs have, header doesn't)** | `SPI1_Reset()` - replaced by `SPI1_DeInit()`.                                                                      |

**Summary:** 3 functions were renamed/consolidated, 3 functions were deleted from the API, and 2 functions gained new parameters - none of these changes were reflected in `SPI_DRIVER.md`.

</details>

---

#### 02. Extract an API Surface

Write a Python function `extract_c_api(header_text: str) -> list[dict]` that extracts all public function declarations from a C header file.

Each dict must contain:

- `name`: function name (string)
- `return_type`: return type (string)
- `parameters`: list of `{"type": str, "name": str}` dicts
- `doxygen_brief`: the `@brief` comment if present (string or None)

Test it against the `spi_driver.h` content from Task 01.

#### Scenario:

◦ The documentation agent needs a machine-readable API surface before it can detect drift or generate docs.
◦ Extracting from the header (ground truth) is better than relying on outdated documentation.

**Hint:** Use regex to capture function declarations. A pattern like `(\w[\w\s*]+)\s+(\w+)\s*\(([^)]*)\)` covers most C declarations.

<details markdown>
<summary>Solution</summary>

```python
import re
from typing import List, Dict, Optional, Any

FUNC_PATTERN = re.compile(
    r'/\*\*\s*(.*?)\s*\*/\s*'      # Doxygen comment block (optional)
    r'([\w\s\*]+?)\s+'             # Return type
    r'(\w+)\s*'                    # Function name
    r'\(([^)]*)\)',                 # Parameter list
    re.DOTALL
)

BRIEF_PATTERN = re.compile(r'@brief\s+(.+?)(?:\n\s*\*|\Z)', re.DOTALL)
PARAM_PATTERN = re.compile(r'([\w\s\*]+?)\s+(\w+)\s*$')

def parse_parameters(param_str: str) -> List[Dict[str, str]]:
    params = []
    if not param_str.strip() or param_str.strip() == "void":
        return params
    for p in param_str.split(","):
        p = p.strip()
        m = PARAM_PATTERN.match(p)
        if m:
            params.append({"type": m.group(1).strip(), "name": m.group(2).strip()})
        else:
            params.append({"type": p, "name": ""})
    return params

def extract_c_api(header_text: str) -> List[Dict[str, Any]]:
    results = []
    for match in FUNC_PATTERN.finditer(header_text):
        doc_comment, return_type, func_name, param_str = match.groups()

        brief_match = BRIEF_PATTERN.search(doc_comment)
        brief = brief_match.group(1).strip().replace("\n * ", " ") if brief_match else None

        results.append({
            "name":          func_name,
            "return_type":   return_type.strip(),
            "parameters":    parse_parameters(param_str),
            "doxygen_brief": brief,
        })
    return results

# Test
SPI_HEADER = """... (spi_driver.h from Task 01) ..."""
api = extract_c_api(SPI_HEADER)
for f in api:
    print(f)

# Expected:
# {'name': 'SPI1_Init',     'return_type': 'int',     'parameters': [...], 'doxygen_brief': 'Initialize SPI1 in master mode.'}
# {'name': 'SPI1_Transfer', 'return_type': 'uint8_t', 'parameters': [...], 'doxygen_brief': 'Transfer a single byte over SPI1.'}
# {'name': 'SPI1_DeInit',   'return_type': 'void',    'parameters': [],    'doxygen_brief': 'Disable SPI1 and release the peripheral clock.'}
```

</details>

---

#### 03. Scan a Git Diff for Doc Impact

Write a Python function `scan_diff_for_doc_impact(diff_text: str) -> list[dict]` that:

1. Parses a unified git diff (`git diff HEAD~1`) of C header files
2. Identifies changed function signatures (lines starting with `+` or `-` that contain a `(` and `)`)
3. For each changed signature, returns: `{"action": "added"|"removed"|"modified", "function": str, "file": str}`
4. Ignores comment changes (lines starting with `+ *`, `- *`, `+//`, `-//`)

**Test diff:**

```diff
--- a/src/spi_driver.h
+++ b/src/spi_driver.h
@@ -1,8 +1,10 @@
-int SPI1_Init(uint32_t baud_rate);
+int SPI1_Init(uint32_t baud_rate, uint8_t mode);
-uint8_t SPI1_Write(uint8_t data);
-uint8_t SPI1_Read(void);
+uint8_t SPI1_Transfer(uint8_t data, uint8_t cs_pin);
+void SPI1_DeInit(void);
```

#### Scenario:

◦ As a CI pipeline pre-commit hook, this scanner alerts the team when a header change needs documentation updates.
◦ It blocks the merge until `--no-doc-update` is explicitly acknowledged.

**Hint:** Track removed signatures in one set and added signatures in another. Functions present in both are "modified" - reconstruct the pairing by function name.

<details markdown>
<summary>Solution</summary>

```python
import re
from typing import List, Dict

FUNC_SIG_PATTERN = re.compile(r'^\s*[\w\s\*]+\s+(\w+)\s*\([^)]*\)\s*;?\s*$')
COMMENT_PREFIX   = re.compile(r'^\s*[\+\-]\s*[\*/]')

def scan_diff_for_doc_impact(diff_text: str) -> List[Dict[str, str]]:
    current_file = None
    removed:  Dict[str, str] = {}  # func_name → signature
    added:    Dict[str, str] = {}

    for line in diff_text.splitlines():
        # Track current file
        if line.startswith("+++ b/"):
            current_file = line[6:].strip()
            continue
        if line.startswith("--- ") or line.startswith("+++ "):
            continue
        if line.startswith("@@") or line.startswith("diff "):
            continue

        # Skip comment-only changes
        if COMMENT_PREFIX.match(line):
            continue

        if line.startswith("-"):
            sig = line[1:].strip()
            m   = FUNC_SIG_PATTERN.match(sig)
            if m:
                removed[m.group(1)] = sig
        elif line.startswith("+"):
            sig = line[1:].strip()
            m   = FUNC_SIG_PATTERN.match(sig)
            if m:
                added[m.group(1)] = sig

    results = []
    # Modified: same name in both removed and added
    for func in set(removed) & set(added):
        results.append({
            "action":   "modified",
            "function": func,
            "file":     current_file or "unknown",
        })
    # Removed only
    for func in set(removed) - set(added):
        results.append({
            "action":   "removed",
            "function": func,
            "file":     current_file or "unknown",
        })
    # Added only
    for func in set(added) - set(removed):
        results.append({
            "action":   "added",
            "function": func,
            "file":     current_file or "unknown",
        })

    return sorted(results, key=lambda x: (x["action"], x["function"]))

# Test
TEST_DIFF = """
--- a/src/spi_driver.h
+++ b/src/spi_driver.h
@@ -1,8 +1,10 @@
-int SPI1_Init(uint32_t baud_rate);
+int SPI1_Init(uint32_t baud_rate, uint8_t mode);
-uint8_t SPI1_Write(uint8_t data);
-uint8_t SPI1_Read(void);
+uint8_t SPI1_Transfer(uint8_t data, uint8_t cs_pin);
+void SPI1_DeInit(void);
"""

for item in scan_diff_for_doc_impact(TEST_DIFF):
    print(item)
# Expected:
# {'action': 'added',    'function': 'SPI1_DeInit',   'file': 'src/spi_driver.h'}
# {'action': 'added',    'function': 'SPI1_Transfer',  'file': 'src/spi_driver.h'}
# {'action': 'modified', 'function': 'SPI1_Init',      'file': 'src/spi_driver.h'}
# {'action': 'removed',  'function': 'SPI1_Read',      'file': 'src/spi_driver.h'}
# {'action': 'removed',  'function': 'SPI1_Write',     'file': 'src/spi_driver.h'}
```

</details>

---

#### 04. Generate Doxygen Comments

Write a prompt for a documentation generation agent that:

1. Accepts a C function implementation (without comments)
2. Generates a complete Doxygen comment block: `@brief`, `@param` for each parameter, `@return`, and `@note` if there is a known hardware constraint
3. Preserves the function signature exactly - no code changes, only the comment block

Then run it against the following two functions and show the expected output:

**Function A:**

```c
int8_t I2C_WriteByte(uint8_t dev_addr, uint8_t reg_addr, uint8_t data)
{
    // ... implementation ...
}
```

**Function B:**

```c
void TIM3_PWM_SetDuty(uint8_t channel, uint16_t duty_ticks)
{
    if (channel > 4) return;
    TIM3->CCR[channel - 1] = duty_ticks;
}
```

#### Scenario:

◦ A developer spent a sprint adding features without comments. The documentation agent retroactively documents the undocumented API.
◦ Doxygen output must be syntactically correct enough to run `doxygen Doxyfile` without warnings.

**Hint:** The `@note` is appropriate when a function has hardware-specific constraints like maximum frequency, voltage range, or peripheral clock dependency.

<details markdown>
<summary>Solution</summary>

**Documentation generation system prompt:**

```
You are a firmware documentation agent.
Generate a complete Doxygen comment block for the following C function.
Requirements:
  - @brief: one sentence, active voice, no "This function..."
  - @param: one line per parameter, include type context and valid range if determinable
  - @return: describe every possible return value
  - @note: include only if there is a hardware-specific constraint (clock must be enabled, timing requirement, etc.)
  - Do NOT modify the function signature or body
  - Output: ONLY the Doxygen comment block followed by the original function signature (no body)
```

**Expected output for Function A:**

```c
/**
 * @brief Write a single byte to a register on an I2C device.
 *
 * @param dev_addr  7-bit I2C device address (without the R/W bit), range 0x00-0x7F.
 * @param reg_addr  Target register address within the device.
 * @param data      Byte value to write to the register.
 *
 * @return  0 on success.
 * @return -1 if the I2C bus is busy or an ACK was not received.
 * @return -2 if the operation timed out after the configured I2C timeout period.
 *
 * @note I2C peripheral clock must be enabled (RCC_APB1ENR_I2C1EN) and
 *       I2C must be initialized before calling this function.
 */
int8_t I2C_WriteByte(uint8_t dev_addr, uint8_t reg_addr, uint8_t data);
```

**Expected output for Function B:**

```c
/**
 * @brief Set the PWM duty cycle for a TIM3 output channel.
 *
 * @param channel     TIM3 output channel number, valid range 1-4.
 *                    Values outside this range are silently ignored.
 * @param duty_ticks  Duty cycle in timer ticks. Should be in range [0, TIM3->ARR]
 *                    to produce a 0%-100% duty cycle. Values exceeding ARR result in 100% output.
 *
 * @note TIM3 must be initialized and running before calling this function.
 *       The maximum value for duty_ticks is the TIM3 auto-reload register (TIM3->ARR).
 *       Channel 1 maps to PA6 (AF2), Channel 2 to PA7 (AF2) on STM32F4.
 */
void TIM3_PWM_SetDuty(uint8_t channel, uint16_t duty_ticks);
```

</details>

---

> **Back to [Tasks Index](../index.md)** | **Next: [014 Tasks](../014-ConstraintBasedGeneration-Tasks/README.md)**
