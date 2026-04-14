# Lab 011 - Tool-Use & Function Calling

!!! hint "Overview"

    - Future of AI: Masterclass in Agentic Engineering** - Tool-Use (Function Calling) module.
    - In this lab, you will teach an LLM agent to **trigger external hardware simulators** (specifically QEMU) to autonomously verify firmware logic without flashing physical hardware.
    - You will implement the OpenAI function-calling protocol to expose QEMU, the ARM cross-compiler, and a register validator as callable tools.
    - By the end of this lab, you will have a fully autonomous FW verification agent that can write, compile, simulate, and validate firmware behavior in a closed loop.

## Prerequisites

- Completed [Lab 010 - Power-Sensitive Refactoring](../010-PowerSensitiveRefactoring/README.md)
- QEMU with ARM support installed

```bash
# macOS
brew install qemu

# Ubuntu/Debian
sudo apt-get install qemu-system-arm

# Verify
qemu-system-arm --version
```

## What You Will Learn

- The OpenAI function-calling (tool-use) protocol
- How to define tools the LLM can invoke (`compile`, `simulate`, `read_register`)
- How to implement a QEMU-based firmware simulation tool
- How to build an autonomous verify-and-fix loop using tool results

---

## Background

### Function Calling Protocol

The OpenAI function-calling API lets you define tools as JSON schemas. The LLM decides when and how to call them:

```json
{
  "type": "function",
  "function": {
    "name": "compile_firmware",
    "description": "Compile C firmware source using arm-none-eabi-gcc",
    "parameters": {
      "type": "object",
      "properties": {
        "source_code": {
          "type": "string",
          "description": "C source code to compile"
        },
        "mcu_target": {
          "type": "string",
          "description": "Target MCU (e.g., cortex-m4, cortex-m7)"
        }
      },
      "required": ["source_code", "mcu_target"]
    }
  }
}
```

The key insight: **the agent decides when to call each tool** based on its current reasoning state. You don't hard-code the sequence - the agent plans it.

---

## Lab Steps

### Step 1 - Define the Tool Set

```python
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "compile_firmware",
            "description": (
                "Compile ARM Cortex-M C firmware source code. "
                "Returns success status and compiler output including errors and warnings."
            ),
            "parameters": {
                "type": "object",
                "properties": {
                    "source_code": {
                        "type": "string",
                        "description": "Complete C source code to compile"
                    },
                    "mcu_target": {
                        "type": "string",
                        "enum": ["cortex-m0", "cortex-m3", "cortex-m4", "cortex-m7"],
                        "description": "ARM Cortex-M target core"
                    }
                },
                "required": ["source_code", "mcu_target"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "simulate_firmware",
            "description": (
                "Run a compiled firmware ELF binary in QEMU ARM simulation. "
                "Returns the program stdout output and exit code after a configurable timeout."
            ),
            "parameters": {
                "type": "object",
                "properties": {
                    "elf_path": {
                        "type": "string",
                        "description": "Path to the compiled ELF binary"
                    },
                    "machine": {
                        "type": "string",
                        "description": "QEMU machine type (e.g., lm3s6965evb, mps2-an385)"
                    },
                    "timeout_seconds": {
                        "type": "integer",
                        "description": "Maximum simulation time in seconds",
                        "default": 10
                    }
                },
                "required": ["elf_path", "machine"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "read_peripheral_register",
            "description": (
                "Look up the description, offset, and bit fields of a peripheral register "
                "from the microcontroller reference manual database."
            ),
            "parameters": {
                "type": "object",
                "properties": {
                    "peripheral": {
                        "type": "string",
                        "description": "Peripheral name (e.g., UART1, SPI2, TIM3)"
                    },
                    "register_name": {
                        "type": "string",
                        "description": "Register name (e.g., CR1, CR2, SR, BRR)"
                    }
                },
                "required": ["peripheral", "register_name"]
            }
        }
    }
]
```

### Step 2 - Implement the Tool Handlers

```python
import subprocess
import pathlib
import tempfile
import json

def compile_firmware(source_code: str, mcu_target: str) -> dict:
    """Tool handler: compile ARM firmware."""
    cpu_flags = {
        "cortex-m0": ["-mcpu=cortex-m0", "-mthumb"],
        "cortex-m3": ["-mcpu=cortex-m3", "-mthumb"],
        "cortex-m4": ["-mcpu=cortex-m4", "-mthumb", "-mfpu=fpv4-sp-d16", "-mfloat-abi=hard"],
        "cortex-m7": ["-mcpu=cortex-m7", "-mthumb", "-mfpu=fpv5-d16", "-mfloat-abi=hard"],
    }

    with tempfile.TemporaryDirectory() as tmpdir:
        src = pathlib.Path(tmpdir) / "firmware.c"
        obj = pathlib.Path(tmpdir) / "firmware.o"
        src.write_text(source_code)

        cmd = ["arm-none-eabi-gcc"] + cpu_flags.get(mcu_target, []) + [
            "-Os", "-Wall", "-nostdlib", "-c", "-o", str(obj), str(src)
        ]
        result = subprocess.run(cmd, capture_output=True, text=True)

        return {
            "success": result.returncode == 0,
            "output": result.stdout + result.stderr,
            "has_warnings": "warning:" in (result.stdout + result.stderr)
        }

def simulate_firmware(elf_path: str, machine: str, timeout_seconds: int = 10) -> dict:
    """Tool handler: run firmware in QEMU."""
    cmd = [
        "qemu-system-arm",
        "-machine", machine,
        "-nographic",
        "-semihosting",
        "-kernel", elf_path
    ]
    try:
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=timeout_seconds)
        return {"exit_code": result.returncode, "output": result.stdout + result.stderr}
    except subprocess.TimeoutExpired:
        return {"exit_code": -1, "output": f"QEMU timed out after {timeout_seconds}s"}

# Simulated register database (replace with real datasheet DB in production)
REGISTER_DB = {
    ("UART1", "CR1"): {
        "offset": "0x00",
        "reset": "0x00000000",
        "fields": {
            "UE": {"bit": 0, "rw": "RW", "desc": "USART Enable"},
            "RE": {"bit": 2, "rw": "RW", "desc": "Receiver Enable"},
            "TE": {"bit": 3, "rw": "RW", "desc": "Transmitter Enable"},
        }
    }
}

def read_peripheral_register(peripheral: str, register_name: str) -> dict:
    """Tool handler: look up register info."""
    key = (peripheral.upper(), register_name.upper())
    return REGISTER_DB.get(key, {"error": f"Register {peripheral}.{register_name} not found in database"})

TOOL_HANDLERS = {
    "compile_firmware": lambda args: compile_firmware(**args),
    "simulate_firmware": lambda args: simulate_firmware(**args),
    "read_peripheral_register": lambda args: read_peripheral_register(**args),
}
```

### Step 3 - Build the Autonomous Verification Agent

```python
import openai
import json

client = openai.OpenAI()

def run_fw_verification_agent(task: str) -> str:
    """Run the firmware agent with tool-calling until task is complete."""
    messages = [
        {
            "role": "system",
            "content": (
                "You are an autonomous firmware verification agent. "
                "You have access to tools: compile_firmware, simulate_firmware, "
                "and read_peripheral_register. "
                "Use them to verify firmware correctness. "
                "Always compile before simulating. "
                "Report the final verification result clearly."
            )
        },
        {"role": "user", "content": task}
    ]

    while True:
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=TOOLS,
            tool_choice="auto"
        )
        msg = response.choices[0].message
        messages.append(msg)

        if msg.tool_calls:
            for tc in msg.tool_calls:
                fn_name = tc.function.name
                fn_args = json.loads(tc.function.arguments)
                print(f"[TOOL CALL] {fn_name}({list(fn_args.keys())})")

                handler = TOOL_HANDLERS.get(fn_name)
                result = handler(fn_args) if handler else {"error": "Unknown tool"}
                print(f"[TOOL RESULT] success={result.get('success', '?')}")

                messages.append({
                    "role": "tool",
                    "tool_call_id": tc.id,
                    "content": json.dumps(result)
                })
        else:
            # No more tool calls - agent has reached a conclusion
            return msg.content

# Run the agent
result = run_fw_verification_agent(
    "Generate a minimal ARM Cortex-M3 'blinky' firmware that initializes "
    "a GPIO and toggles it in a loop using semihosting printf. "
    "Compile it for cortex-m3, simulate it on lm3s6965evb, and confirm it runs without errors."
)
print("\n=== AGENT CONCLUSION ===")
print(result)
```

---

## Summary

| Skill                       | What You Practiced                                      |
| --------------------------- | ------------------------------------------------------- |
| Tool schema definition      | JSON function definitions for compile, simulate, lookup |
| Tool handler implementation | Python wrappers for arm-gcc and QEMU                    |
| Autonomous tool selection   | Agent decides when and which tools to call              |
| Closed-loop verification    | Compile → simulate → report, all agent-driven           |

---

> **Next Lab:** [012 - ReAct Patterns for HW-SW Debugging](../012-ReActPatterns/README.md)
