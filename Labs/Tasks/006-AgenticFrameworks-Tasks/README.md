# Lab 006 - Agentic Frameworks Tasks

- Exercises covering LangGraph state design, node implementation, routing, graph wiring, and human-in-the-loop checkpoints.
- Each task includes a description, scenario, hint, and collapsible Solution.

#### Table of Contents

- [01. Define the State TypedDict](#01-define-the-state-typeddict)
- [02. Implement the Plan Node](#02-implement-the-plan-node)
- [03. Implement the Review Router](#03-implement-the-review-router)
- [04. Wire the Full Graph](#04-wire-the-full-graph)
- [05. Add a Human-in-the-Loop Checkpoint](#05-add-a-human-in-the-loop-checkpoint)

---

#### 01. Define the State TypedDict

Define a LangGraph `TypedDict` state for a firmware code generation agent that:

1. Tracks the original `task` description (string)
2. Stores the `hardware_context` retrieved from RAG (string)
3. Holds the current `generated_code` (string, optional — may not exist yet)
4. Keeps a `build_result` dict with keys `success` (bool), `errors` (list of strings), `warnings` (list of strings)
5. Stores a `review_status` from the human: `"pending"`, `"approved"`, or `"rejected"`
6. Keeps `iteration_count` (int) to guard against infinite loops

#### Scenario:

◦ Before implementing any LangGraph node, the team agrees on a shared state schema.
◦ All nodes read from and write to this state — it is the single source of truth.

**Hint:** Use `Optional[str]` for fields that may not exist at graph start. Import from `typing` and `typing_extensions`.

<details markdown>
<summary>Solution</summary>

```python
from typing import Optional, List
from typing_extensions import TypedDict

class BuildResult(TypedDict):
    success: bool
    errors:  List[str]
    warnings: List[str]

class FirmwareAgentState(TypedDict):
    task:             str                  # Original task description
    hardware_context: str                  # Retrieved hardware context (RAG)
    generated_code:   Optional[str]        # Latest generated C code
    build_result:     Optional[BuildResult]# Result of last compilation attempt
    review_status:    str                  # "pending" | "approved" | "rejected"
    iteration_count:  int                  # Current iteration (max guard)
```

</details>

---

#### 02. Implement the Plan Node

Implement a LangGraph `plan_node` that:

1. Reads `task` and `hardware_context` from the state
2. Calls the LLM to generate a **step-by-step plan** (not code yet): list the registers to configure and the exact values
3. Stores the plan as the initial value of `generated_code` (prefixed with `# PLAN:\n`)
4. Resets `build_result` to `None`
5. Increments `iteration_count`

#### Scenario:

◦ Separating the planning phase from the code generation phase improves output quality by forcing the LLM to reason before writing code.
◦ The plan node is the first node executed in the graph.

**Hint:** Use `langchain_openai.ChatOpenAI` with `temperature=0`. The plan should be in a structured list, not code.

<details markdown>
<summary>Solution</summary>

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage

llm = ChatOpenAI(model="gpt-4o", temperature=0)

PLAN_SYSTEM = """
You are a firmware planning agent.
Given a firmware task and hardware context, produce a step-by-step plan.
Each step must specify: the register name, offset, and the exact bit value to write.
Output format: numbered list, no code, just the plan.
"""

def plan_node(state: FirmwareAgentState) -> FirmwareAgentState:
    messages = [
        SystemMessage(content=PLAN_SYSTEM),
        HumanMessage(content=(
            f"Task: {state['task']}\n\n"
            f"Hardware context:\n{state['hardware_context']}"
        ))
    ]
    response = llm.invoke(messages)
    plan_text = "# PLAN:\n" + response.content

    return {
        **state,
        "generated_code": plan_text,
        "build_result":   None,
        "review_status":  "pending",
        "iteration_count": state["iteration_count"] + 1,
    }
```

</details>

---

#### 03. Implement the Review Router

Implement a LangGraph conditional edge router `review_router` that:

1. Returns `"approved"` if `review_status == "approved"`
2. Returns `"regenerate"` if `review_status == "rejected"` AND `iteration_count < 5`
3. Returns `"failed"` if `review_status == "rejected"` AND `iteration_count >= 5`
4. Returns `"waiting"` if `review_status == "pending"`

#### Scenario:

◦ The graph must route to the right next node depending on the human reviewer's decision.
◦ A max-iteration guard prevents infinite loops when the human keeps rejecting outputs.

**Hint:** In LangGraph, conditional edges return a string that maps to an edge name registered with `add_conditional_edges`.

<details markdown>
<summary>Solution</summary>

```python
from typing import Literal

def review_router(
    state: FirmwareAgentState,
) -> Literal["approved", "regenerate", "failed", "waiting"]:
    status = state["review_status"]
    count  = state["iteration_count"]

    if status == "approved":
        return "approved"
    elif status == "rejected":
        if count < 5:
            return "regenerate"
        else:
            return "failed"
    else:  # "pending"
        return "waiting"
```

</details>

---

#### 04. Wire the Full Graph

Using the state and nodes from Tasks 01–03, wire a complete LangGraph `StateGraph` with:

- **Nodes**: `plan`, `generate_code`, `build`, `human_review`, `finalize`, `fail`
- **Edges**:
  - START → `plan`
  - `plan` → `generate_code`
  - `generate_code` → `build`
  - `build` → `human_review` (if build succeeded) or → `generate_code` (if build failed and iterations < 5)
  - `human_review` → conditional edge using `review_router`
  - `approved` → `finalize` → END
  - `regenerate` → `generate_code`
  - `failed` → `fail` → END

#### Scenario:

◦ After all nodes are implemented individually, the graph wiring ties them into a complete autonomous pipeline.
◦ The graph should be runnable end-to-end from a single `graph.invoke()` call.

**Hint:** Use `add_conditional_edges` with a dict mapping router output strings to node names.

<details markdown>
<summary>Solution</summary>

```python
from langgraph.graph import StateGraph, START, END

builder = StateGraph(FirmwareAgentState)

# Add nodes
builder.add_node("plan",          plan_node)
builder.add_node("generate_code", generate_code_node)
builder.add_node("build",         build_node)
builder.add_node("human_review",  human_review_node)
builder.add_node("finalize",      finalize_node)
builder.add_node("fail",          fail_node)

# Static edges
builder.add_edge(START,           "plan")
builder.add_edge("plan",          "generate_code")
builder.add_edge("generate_code", "build")

# Conditional: build result → next node
def build_router(state: FirmwareAgentState) -> str:
    if state["build_result"] and state["build_result"]["success"]:
        return "human_review"
    elif state["iteration_count"] < 5:
        return "generate_code"
    else:
        return "fail"

builder.add_conditional_edges("build", build_router, {
    "human_review":  "human_review",
    "generate_code": "generate_code",
    "fail":          "fail",
})

# Conditional: human review → next node
builder.add_conditional_edges("human_review", review_router, {
    "approved":   "finalize",
    "regenerate": "generate_code",
    "failed":     "fail",
    "waiting":    "human_review",   # loop back to wait
})

builder.add_edge("finalize", END)
builder.add_edge("fail",     END)

graph = builder.compile()
```

</details>

---

#### 05. Add a Human-in-the-Loop Checkpoint

Extend the compiled graph with a LangGraph `interrupt_before` checkpoint at the `human_review` node so that:

1. The graph pauses before entering `human_review`
2. An external caller can inspect `state["generated_code"]` and `state["build_result"]`
3. The external caller sets `review_status` to `"approved"` or `"rejected"` and resumes
4. Show the complete run-and-resume pattern using `graph.invoke()` + a thread config

#### Scenario:

◦ Fully autonomous generation is acceptable for planning, but code that will be flashed to hardware requires a human sign-off.
◦ The LangGraph interrupt mechanism provides an audit-friendly pause point.

**Hint:** Pass `interrupt_before=["human_review"]` to `builder.compile()`. Use `graph.invoke(None, config)` to resume.

<details markdown>
<summary>Solution</summary>

```python
from langgraph.checkpoint.memory import MemorySaver

# Recompile with checkpoint and interrupt
memory   = MemorySaver()
graph    = builder.compile(
    checkpointer=memory,
    interrupt_before=["human_review"]
)

# ─── First run: graph pauses before human_review ───────────────────────────
thread = {"configurable": {"thread_id": "fw-task-001"}}
initial_state = FirmwareAgentState(
    task="Initialize SPI1 in master mode at 1 MHz",
    hardware_context="STM32F4, APB2=84MHz, SPI_CR1 at offset 0x00...",
    generated_code=None,
    build_result=None,
    review_status="pending",
    iteration_count=0,
)

# Run until interrupt
snapshot = graph.invoke(initial_state, thread)
print("Graph paused. Generated code:")
print(snapshot["generated_code"])
print("Build result:", snapshot["build_result"])

# ─── Human review (simulated) ──────────────────────────────────────────────
human_decision = "approved"   # or "rejected"
graph.update_state(thread, {"review_status": human_decision})

# ─── Resume from checkpoint ─────────────────────────────────────────────────
final_state = graph.invoke(None, thread)
print("Final state:", final_state["review_status"])
```

</details>

---

> **Back to [Tasks Index](../index.md)** | **Next: [007 Tasks](../007-SelfHealingWorkflows-Tasks/README.md)**
