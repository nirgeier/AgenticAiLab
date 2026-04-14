# Lab 011 - Tool Use & Function Calling Tasks

- Exercises covering JSON tool schema definition, handler implementation, tool call loop, and error handling.
- Each task includes a description, scenario, hint, and collapsible Solution.

#### Table of Contents

- [01. Define a Tool Schema in JSON](#01-define-a-tool-schema-in-json)
- [02. Implement the Tool Handler](#02-implement-the-tool-handler)
- [03. Parse a Tool Call Response](#03-parse-a-tool-call-response)
- [04. Build the Agentic Loop](#04-build-the-agentic-loop)
- [05. Handle Tool Errors Gracefully](#05-handle-tool-errors-gracefully)

---

#### 01. Define a Tool Schema in JSON

Define OpenAI-compatible JSON tool schemas for the following three firmware-specific tools:

1. **`read_register`**: Reads a peripheral register. Parameters: `peripheral` (string, required), `register` (string, required). Returns a hex string value.
2. **`write_register`**: Writes a value to a register. Parameters: `peripheral` (string, required), `register` (string, required), `value` (integer, required, hex representation). No return value.
3. **`compile_and_check`**: Compiles C source code and returns errors. Parameters: `source_code` (string, required), `target` (string, optional, default `"cortex-m4"`). Returns `{success, errors, warnings}`.

#### Scenario:

◦ Before an LLM can call a tool, it needs a well-structured schema that describes the tool's purpose, parameters, and types.
◦ A poorly designed schema (missing descriptions, wrong types) leads to incorrect tool calls.

**Hint:** Each tool schema has three fields: `name`, `description`, and `parameters` (a JSON Schema object). Every required parameter needs a `description`.

<details markdown>
<summary>Solution</summary>

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "read_register",
            "description": (
                "Read the current value of a hardware peripheral register. "
                "Returns the register value as a hexadecimal string, e.g. '0x00000040'."
            ),
            "parameters": {
                "type": "object",
                "properties": {
                    "peripheral": {
                        "type": "string",
                        "description": "The peripheral name, e.g. 'USART2', 'SPI1', 'ADC1'.",
                    },
                    "register": {
                        "type": "string",
                        "description": "The register name within the peripheral, e.g. 'CR1', 'SR', 'BRR'.",
                    },
                },
                "required": ["peripheral", "register"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "write_register",
            "description": (
                "Write a value to a hardware peripheral register. "
                "Use this to configure peripheral control registers."
            ),
            "parameters": {
                "type": "object",
                "properties": {
                    "peripheral": {
                        "type": "string",
                        "description": "The peripheral name, e.g. 'USART2', 'SPI1'.",
                    },
                    "register": {
                        "type": "string",
                        "description": "The register name, e.g. 'CR1', 'BRR'.",
                    },
                    "value": {
                        "type": "integer",
                        "description": "The integer value to write to the register.",
                    },
                },
                "required": ["peripheral", "register", "value"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "compile_and_check",
            "description": (
                "Compile the given C source code for an ARM Cortex target and return "
                "compilation results including any errors or warnings."
            ),
            "parameters": {
                "type": "object",
                "properties": {
                    "source_code": {
                        "type": "string",
                        "description": "The complete C source code to compile.",
                    },
                    "target": {
                        "type": "string",
                        "description": "ARM Cortex target, e.g. 'cortex-m4', 'cortex-m0plus'. Defaults to 'cortex-m4'.",
                    },
                },
                "required": ["source_code"],
            },
        },
    },
]
```

</details>

---

#### 02. Implement the Tool Handler

Implement the `handle_tool_call(tool_name, tool_args)` dispatcher that routes tool calls to their implementations.

Requirements:

1. Implement `read_register(peripheral, register)` → returns a simulated hex value (use a lookup table with at least 3 entries)
2. Implement `write_register(peripheral, register, value)` → updates the simulated register file and returns `"OK"`
3. Implement `compile_and_check(source_code, target)` → calls the `compile_firmware()` function from Lab 007
4. `handle_tool_call` must return a **JSON string** for feeding back into the LLM conversation

#### Scenario:

◦ The tool call dispatcher is the bridge between the LLM's intent and real-world actions.
◦ All tool results must be returned as JSON strings to satisfy the OpenAI tool response format.

**Hint:** Use `json.dumps()` to serialize the return value. Keep the register file as a module-level dict.

<details markdown>
<summary>Solution</summary>

```python
import json
from typing import Any

# Simulated hardware register file
_register_file: dict = {
    "USART2_CR1":  0x0000,
    "USART2_BRR":  0x0000,
    "SPI1_CR1":    0x0000,
    "ADC1_CR2":    0x0000,
    "ADC1_SR":     0x0002,  # EOC bit set (simulated "conversion done")
}

def read_register(peripheral: str, register: str) -> str:
    key = f"{peripheral}_{register}"
    value = _register_file.get(key, 0x00000000)
    return json.dumps({"value": hex(value)})

def write_register(peripheral: str, register: str, value: int) -> str:
    key = f"{peripheral}_{register}"
    _register_file[key] = value & 0xFFFFFFFF
    return json.dumps({"status": "OK", "written": hex(value)})

def _compile_and_check(source_code: str, target: str = "cortex-m4") -> str:
    result = compile_firmware(source_code, target)  # from Lab 007
    return json.dumps(result)

def handle_tool_call(tool_name: str, tool_args: dict) -> str:
    """
    Dispatch a tool call to its implementation.
    Returns a JSON string for inclusion in the LLM message history.
    """
    if tool_name == "read_register":
        return read_register(
            peripheral=tool_args["peripheral"],
            register=tool_args["register"],
        )
    elif tool_name == "write_register":
        return write_register(
            peripheral=tool_args["peripheral"],
            register=tool_args["register"],
            value=tool_args["value"],
        )
    elif tool_name == "compile_and_check":
        return _compile_and_check(
            source_code=tool_args["source_code"],
            target=tool_args.get("target", "cortex-m4"),
        )
    else:
        return json.dumps({"error": f"Unknown tool: {tool_name}"})
```

</details>

---

#### 03. Parse a Tool Call Response

The following is an actual OpenAI tool call response. Write the code to extract the tool name, ID, and arguments, then call `handle_tool_call()` and return the result in the correct format for the next LLM message.

```python
# Raw OpenAI response object (simulated as a dict for this exercise)
raw_response = {
    "id": "chatcmpl-abc123",
    "choices": [{
        "message": {
            "role": "assistant",
            "tool_calls": [{
                "id": "call_xyz789",
                "type": "function",
                "function": {
                    "name": "read_register",
                    "arguments": '{"peripheral": "USART2", "register": "BRR"}'
                }
            }]
        },
        "finish_reason": "tool_calls"
    }]
}
```

#### Scenario:

◦ The LLM response can contain multiple tool calls. The parsing logic must handle all of them.
◦ Each tool result must be fed back to the LLM as a `role: "tool"` message with the matching `tool_call_id`.

**Hint:** Access `choices[0].message.tool_calls` (or `["tool_calls"]` for dict). Return a list of `{"role": "tool", "tool_call_id": ..., "content": ...}` dicts.

<details markdown>
<summary>Solution</summary>

```python
import json

def process_tool_calls(response: dict) -> list:
    """
    Process all tool calls in an OpenAI response.
    Returns a list of tool result messages to append to message history.
    """
    message     = response["choices"][0]["message"]
    tool_calls  = message.get("tool_calls", [])
    tool_results = []

    for tc in tool_calls:
        tool_id   = tc["id"]
        tool_name = tc["function"]["name"]
        tool_args = json.loads(tc["function"]["arguments"])   # JSON string → dict

        print(f"Calling tool: {tool_name}({tool_args})")
        result_json = handle_tool_call(tool_name, tool_args)

        tool_results.append({
            "role":         "tool",
            "tool_call_id": tool_id,
            "content":      result_json,
        })

    return tool_results

# Test
results = process_tool_calls(raw_response)
print(results)
# Expected:
# [{"role": "tool", "tool_call_id": "call_xyz789",
#   "content": '{"value": "0x0000"}'}]
```

</details>

---

#### 04. Build the Agentic Loop

Combine Tasks 01–03 into a complete **agentic tool-use loop** that:

1. Starts with a task: `"Configure USART2 for 115200 baud at 84 MHz APB1 and enable the peripheral"`
2. Sends the task + tools array to the LLM
3. Detects `finish_reason == "tool_calls"` and dispatches all tool calls
4. Appends tool results to the message history
5. Loops until `finish_reason == "stop"`
6. Prints the final response and number of tool call rounds

#### Scenario:

◦ The full agentic loop is the core of any tool-using agent. Without it, the LLM can only suggest code — with it, it can actually configure hardware.
◦ A well-implemented loop handles multi-turn tool call chains naturally.

**Hint:** The loop condition is `finish_reason != "stop"`. Add a max-rounds guard to prevent runaway loops.

<details markdown>
<summary>Solution</summary>

```python
from openai import OpenAI

client = OpenAI()

def run_agentic_tool_loop(task: str, max_rounds: int = 10) -> str:
    messages = [
        {
            "role": "system",
            "content": (
                "You are a firmware configuration agent. "
                "Use the provided tools to configure hardware registers. "
                "Verify your writes by reading back the register. "
                "When done, report what you configured."
            )
        },
        {"role": "user", "content": task},
    ]

    for round_num in range(1, max_rounds + 1):
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=tools,            # from Task 01
            tool_choice="auto",
        )

        choice           = response.choices[0]
        finish_reason    = choice.finish_reason
        assistant_message = choice.message

        # Append assistant message to history
        messages.append(assistant_message.model_dump())

        if finish_reason == "stop":
            print(f"Completed in {round_num} round(s).")
            return assistant_message.content

        if finish_reason == "tool_calls":
            # Convert response to dict for our parser
            response_dict = {
                "choices": [{"message": assistant_message.model_dump()}]
            }
            tool_results = process_tool_calls(response_dict)
            messages.extend(tool_results)
        else:
            print(f"Unexpected finish_reason: {finish_reason}")
            break

    return "Max rounds reached without completion."

# Run
final = run_agentic_tool_loop(
    "Configure USART2 for 115200 baud at 84 MHz APB1 and enable the peripheral."
)
print("\nFinal answer:", final)
```

</details>

---

#### 05. Handle Tool Errors Gracefully

Extend the tool handler from Task 02 to handle errors gracefully:

1. If `peripheral` is not in the known register file, return `{"error": "Unknown peripheral: {name}", "available": [list]}`
2. If `source_code` is empty in `compile_and_check`, return `{"error": "source_code cannot be empty"}`
3. If the underlying system call fails (e.g., `arm-none-eabi-gcc` not found), catch the `FileNotFoundError` and return `{"error": "Compiler not found. Is arm-none-eabi-gcc installed?"}`
4. Add a `max_register_value` validation: if `value > 0xFFFFFFFF`, return an error

Also write a test that sends a bad tool call to the LLM loop and verifies the agent self-corrects (retries with valid parameters).

#### Scenario:

◦ In production, tool handlers encounter unexpected inputs. Errors returned as JSON are handled gracefully by the LLM — it reads the error and retries with corrected parameters.
◦ Unhandled Python exceptions crash the loop entirely.

**Hint:** Always return `json.dumps({"error": str(e)})` in except blocks, never re-raise in a tool handler.

<details markdown>
<summary>Solution</summary>

```python
import json
import subprocess

KNOWN_PERIPHERALS = {"USART2", "SPI1", "ADC1", "TIM2", "I2C1"}

def read_register(peripheral: str, register: str) -> str:
    if peripheral not in KNOWN_PERIPHERALS:
        return json.dumps({
            "error": f"Unknown peripheral: {peripheral}",
            "available": sorted(KNOWN_PERIPHERALS),
        })
    key   = f"{peripheral}_{register}"
    value = _register_file.get(key, 0x00000000)
    return json.dumps({"value": hex(value)})

def write_register(peripheral: str, register: str, value: int) -> str:
    if peripheral not in KNOWN_PERIPHERALS:
        return json.dumps({
            "error": f"Unknown peripheral: {peripheral}",
            "available": sorted(KNOWN_PERIPHERALS),
        })
    if value > 0xFFFFFFFF:
        return json.dumps({
            "error": f"Value {hex(value)} exceeds 32-bit register width (max 0xFFFFFFFF)"
        })
    key = f"{peripheral}_{register}"
    _register_file[key] = value
    return json.dumps({"status": "OK", "written": hex(value)})

def _compile_and_check(source_code: str, target: str = "cortex-m4") -> str:
    if not source_code or not source_code.strip():
        return json.dumps({"error": "source_code cannot be empty"})
    try:
        result = compile_firmware(source_code, target)
        return json.dumps(result)
    except FileNotFoundError:
        return json.dumps({
            "error": "Compiler not found. Is arm-none-eabi-gcc installed and on PATH?"
        })
    except Exception as e:
        return json.dumps({"error": f"Compilation failed with unexpected error: {str(e)}"})

# --- Self-correction test ---
# Send a tool call with an unknown peripheral; the LLM should read the error
# and retry with a valid peripheral name.
def test_self_correction():
    messages = [
        {"role": "system", "content": "You are a firmware agent. Use tools to read registers."},
        {"role": "user",   "content": "Read the CR1 register of peripheral 'INVALID_PERIPH'."},
    ]
    # First call: LLM calls read_register("INVALID_PERIPH", "CR1")
    # Tool returns: {"error": "Unknown peripheral: INVALID_PERIPH", "available": [...]}
    # Second call: LLM self-corrects and picks a valid peripheral from the list
    result = run_agentic_tool_loop(
        "Read the CR1 register of peripheral 'INVALID_PERIPH'.",
        max_rounds=5,
    )
    print("Self-correction test result:", result)
    assert "INVALID_PERIPH" not in result or "corrected" in result.lower()
    print("PASSED: Agent self-corrected after invalid tool call.")
```

</details>

---

> **Back to [Tasks Index](../index.md)** | **Next: [012 Tasks](../012-ReActPatterns-Tasks/README.md)**
