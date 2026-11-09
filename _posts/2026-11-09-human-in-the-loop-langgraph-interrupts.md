---
layout: "post"
title: "Human‑in‑the‑Loop with LangGraph: Using Interrupts to Build Controllable AI Agent Workflows"
date: "2026-11-09"
description: "Learn how LangGraph's interrupt, resume, and rewind primitives let you add controllable human‑in‑the‑loop checkpoints to AI agent workflows, improving safety an"
tags: ["langgraph", "hitl", "ai-agents", "workflow"]
permalink: "/posts/human-in-the-loop-langgraph-interrupts/"
---

## Why Interrupts Matter for Human‑in‑the‑Loop Agents

A **human‑in‑the‑loop (HITL)** workflow inserts explicit user oversight at one or more decision points, letting a person validate, correct, or augment the model’s output before the next step proceeds. In contrast, a fully autonomous agent runs end‑to‑end without any external check, which can be risky when the task involves regulatory, ethical, or financial consequences.

Long‑running, stateful business processes—such as multi‑day document processing pipelines or iterative data enrichment—accumulate context over many steps. A **pause‑review‑resume** cycle lets operators inspect intermediate state, resolve ambiguities, and prevent costly downstream errors, while preserving the graph’s memory for seamless continuation.

LangGraph addresses this need with a dedicated **interrupt primitive** (`interrupt()`, `resume()`, `rewind()`). It is exposed as a first‑class building block, allowing developers to declaratively embed HITL checkpoints directly into the graph definition rather than wiring ad‑hoc callbacks.

Typical scenarios where an interrupt is indispensable include:
- **Compliance approval** before releasing a contract draft.  
- **Data sanitization** to verify personally identifiable information is removed.  
- **Budget gating** that requires manager sign‑off before committing resources.

## The Core Interrupt API: interrupt(), resume(), and rewind()

```python
# Import the interrupt primitives and the message type
from langgraph.graph import interrupt, resume, rewind, Graph
from langgraph.messages import HumanMessage
```

### 1️⃣ Interrupt node  
A node that calls `interrupt()` simply returns a `HumanMessage`. The runtime treats this return value specially: it pauses the graph and emits the message to any external UI or approval dashboard ([Source](https://www.agentnotebook.dev/tutorials/langgraph-human-in-the-loop)).  

```python
def ask_human(state):
    # interrupt() returns a HumanMessage that the runtime pauses on
    return interrupt(HumanMessage(content="Approve the proposal?"))
```

### 2️⃣ Resuming execution  
When a human decision arrives, the external system calls `resume(state)` with the **full** state object (no separate payload). The graph continues from the exact checkpoint where it stopped ([Source](https://turion.ai/blog/langgraph-human-in-the-loop-interrupt-tutorial)).

```python
def continue_after_human(updated_state):
    # Pass the complete state back to the runtime
    return resume(updated_state)
```

### 3️⃣ Rewinding for time‑travel debugging  
`rewind(step_id)` rolls the execution back to a previously visited node, allowing you to replay downstream logic without rebuilding the graph. This is handy for audit trails and “what‑if” analysis ([Source](https://christianmendieta.ca/human-in-the-loop-ai-time-travel-workflows-with-langgraph)).

```python
# Jump back to node "approval_step" and re‑execute from there
rewind("approval_step")
```

### 4️⃣ Reusable helper  

```python
def add_interrupt_node(graph: Graph, node_name: str, prompt: str) -> Graph:
    """Inject an interrupt checkpoint into `graph`."""
    @graph.node(node_name)
    def interrupt_node(state):
        return interrupt(HumanMessage(content=prompt))
    return graph
```

The helper abstracts the pattern so you can drop a human‑in‑the‑loop checkpoint into any existing LangGraph with a single call ([Source](https://turion.ai/blog/langgraph-human-in-the-loop-interrupt-tutorial)).

<pre class="mermaid">
sequenceDiagram
    participant Agent as Agent Node
    participant Runtime as LangGraph Runtime
    participant UI as Approval UI
    participant Human as Human Reviewer
    participant Service as External Service
    Agent-&gt;&gt;Runtime: call interrupt()
    Runtime--&gt;&gt;UI: emit HumanMessage
    UI-&gt;&gt;Human: display request
    Human-&gt;&gt;UI: submit decision
    UI-&gt;&gt;Service: POST decision
    Service-&gt;&gt;Runtime: call resume(state)
    Runtime--&gt;&gt;Agent: continue execution
    Note over Runtime: optional rewind
    Service-&gt;&gt;Runtime: call rewind(step_id)
    Runtime--&gt;&gt;Agent: restart from step_id
</pre>

*Figure 1: Shows how an interrupt pauses execution, a human decision triggers resume, and rewind can roll back to a prior node.*

## Reusable Interrupt Patterns for Production Agents

Below are three compact, production‑ready interrupt patterns that can be dropped into any LangGraph `StateGraph`. All snippets use the documented `interrupt(HumanMessage)` signature and the `resume(state)` API.

### 1. Approval Gate  
```python
from langchain.schema import HumanMessage
from langgraph.graph import interrupt, resume

def approval_gate(state: dict):
    # Emit a HumanMessage that carries UI payload and actions
    msg = HumanMessage(
        content="Approve request?",
        additional_kwargs={
            "payload": state["request"],
            "actions": ["approve", "reject"],
        },
    )
    return interrupt(msg)

def handle_approval(state: dict, decision: str):
    # Merge the decision into the state before resuming
    state["status"] = "approved" if decision == "approve" else "rejected"
    return resume(state)  # ([Source](https://www.agentnotebook.dev/tutorials/langgraph-human-in-the-loop))
```

### 2. Timeout‑Fallback  
```python
import asyncio
from langgraph.graph import interrupt, resume

async def wait_for_human(state: dict) -> str:
    """Stub that blocks until a UI writes `state["decision"]`."""
    while "decision" not in state:
        await asyncio.sleep(0.5)
    return state["decision"]

async def timeout_fallback(state: dict, timeout: int = 30):
    try:
        decision = await asyncio.wait_for(wait_for_human(state), timeout)
    except asyncio.TimeoutError:
        decision = "reject"          # auto‑fallback
    state["decision"] = decision
    return resume(state)           # ([Source](https://activewizards.com/blog/langgraph-interrupt-patterns-beyond-the-basics-conditional-approval-batch-review-and-timeout-handling))
```

### 3. Batch Review  
```python
from threading import local
_thread_ctx = local()               # thread‑local store for safety

def collect(state: dict):
    ctx = getattr(_thread_ctx, "batch", [])
    ctx.append(state["partial"])
    _thread_ctx.batch = ctx
    return interrupt(HumanMessage(content="Collecting more items..."))

def finalize(state: dict):
    state["results"] = getattr(_thread_ctx, "batch", [])
    return resume(state)           # ([Source](https://turion.ai/blog/langgraph-human-in-the-loop-interrupt-tutorial))
```

### Composing in a `StateGraph`  
```python
from langgraph.graph import StateGraph

graph = StateGraph()
graph.add_node("gate", approval_gate)
graph.add_node("handle", handle_approval)
graph.add_node("timeout", timeout_fallback)
graph.add_node("collect", collect)
graph.add_node("final", finalize)

graph.add_conditional_edge("gate", "handle", lambda s: True)
graph.add_conditional_edge("handle", "timeout", lambda s: s["status"] == "approved")
graph.add_edge("timeout", "collect")
graph.add_edge("collect", "final")   # ([Source](https://arxiv.org/html/2607.19297v1))
```

### One‑File Template  
Save the above functions and the `graph` definition in `hitl_patterns.py`. Import it wherever you need a consistent HITL contract:

```python
from hitl_patterns import graph as my_agent_graph
```

This single module enforces the same interrupt behavior across services while keeping the codebase DRY.

<pre class="mermaid">
flowchart LR
    A[Start] --&gt; B[approval_gate]
    B --&gt; C[handle_approval]
    C --&gt;|approved| D[timeout_fallback]
    C --&gt;|rejected| Z[End]
    D --&gt; E[collect]
    E --&gt; F[finalize]
    F --&gt; Z[End]
    subgraph Batch Review
        E --&gt; G[interrupt (collect)]
        G --&gt; H[human batch decision]
        H --&gt; E
    end
</pre>

*Figure 2: Illustrates how approval, timeout, and batch‑review nodes are wired together in a StateGraph.*

## Edge Cases, Failure Modes, and Debugging Strategies

- **Missed approvals** – A `HumanMessage` can linger if the reviewer never responds. Attach a short‑lived token and invalidate it after a TTL (e.g., `expires_at = time.time() + 300`). When the token expires, discard the pending message and resend the request. ([Source](https://machinelearningplus.com/gen-ai/langgraph-human-in-the-loop-approval-steps))

- **Race conditions in parallel branches** – Simultaneous interrupts may overwrite each other’s state. Guard critical sections with `graph.acquire_lock()` or store a version number with each state snapshot, accepting only the newest version. ([Source](https://forum.langchain.com/t/interrupt-parallel-branch-execution/3800))

- **State drift after rewind** – After calling `interrupt()`, re‑hydrate the exact snapshot it returns and ensure every node is idempotent before re‑execution. This prevents hidden side effects and keeps the graph consistent. ([Source](https://pub.towardsai.net/langgraph-human-in-the-loop-pausing-reviewing-and-rewinding-your-agent-4028bd05b049))

- **Debugging tip** – Turn on tracing with `graph.set_tracing(True)` and pipe the JSONL stream to a structured logger. The trace can be replayed to locate the interrupt that caused a failure. ([Source](https://www.agentnotebook.dev/tutorials/langgraph-human-in-the-loop))

- **Observability** – Export Prometheus counters for `interrupt_count`, `resume_latency`, and `rewind_events`. Example:

```python
from prometheus_client import Counter, Histogram

interrupt_count = Counter("lg_interrupt_total", "Total interrupts")
resume_latency = Histogram("lg_resume_seconds", "Resume latency")
rewind_events = Counter("lg_rewind_total", "Rewind events")

def on_interrupt():
    interrupt_count.inc()

def on_resume(duration):
    resume_latency.observe(duration)

def on_rewind():
    rewind_events.inc()
```

Monitoring these metrics in real time highlights abnormal latency or unexpected rewind spikes. ([Source](https://activewizards.com/blog/langgraph-interrupt-patterns-beyond-the-basics-conditional-approval-batch-review-and-timeout-handling))

<pre class="mermaid">
stateDiagram-v2
    [*] --&gt; Idle
    Idle --&gt; WaitingForHuman: interrupt()
    WaitingForHuman --&gt; Resumed: resume(state)
    WaitingForHuman --&gt; Error: timeout
    Resumed --&gt; Idle: next step
    Resumed --&gt; Rewound: rewind(step_id)
    Rewound --&gt; WaitingForHuman: interrupt()
    Error --&gt; Idle: recover
</pre>

*Figure 3: Depicts possible states of a graph around interrupts, resumes, rewinds, and error handling.*

## Performance, Cost, and Operational Considerations

**Latency impact** – The human‑review pause dominates wall‑clock time. Teams typically see 5 s – 30 s for quick approvals and up to a minute for detailed checks; these numbers should be baked into SLA targets ([Source](https://christianmendieta.ca/human-in-the-loop-ai-time-travel-workflows-with-langgraph)).  

**Token cost** – `interrupt()` only snapshots the graph, so no LLM tokens are spent ([Source](https://www.agentnotebook.dev/tutorials/langgraph-human-in-the-loop)). When `resume()` re‑executes a node, the original prompt runs again, incurring the same token bill. Mitigate repeat costs by caching deterministic node outputs, e.g.:

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def run_expensive_node(input):
    return llm.invoke(prompt_template.format(input))
```

For distributed caches, swap `lru_cache` for a Redis‑backed store ([Source](https://www.agentnotebook.dev/tutorials/langgraph-human-in-the-loop)).  

**Scaling parallel branches** – Each branch keeps its own state snapshot, increasing memory linearly with branch count. Shard large graphs across multiple workers or containers to keep the footprint bounded ([Source](https://forum.langchain.com/t/interrupt-parallel-branch-execution/3800)).  

**Security & privacy** – Strip PII from any payload shown in the UI. The open‑source approval dashboard provides a sanitization hook that runs on the `interrupt` payload ([Source](https://forum.langchain.com/t/human-in-the-loop-approval-dashboard-for-langgraph-agents-open-source-free-to-deploy/3616)).  

**Operational tip** – Wrap `resume()` behind a lightweight FastAPI service that validates a signed token before proceeding, isolating the LLM runtime and creating an audit trail ([Source](https://machinelearningplus.com/gen-ai/langgraph-human-in-the-loop-approval-steps)):

```python
from fastapi import FastAPI, HTTPException
import hmac, hashlib

app = FastAPI()
SECRET = b"super‑secret"

def verify_sig(payload: bytes, sig: str):
    expected = hmac.new(SECRET, payload, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, sig):
        raise HTTPException(status_code=401, detail="Invalid signature")

@app.post("/resume")
async def resume(payload: dict, signature: str):
    verify_sig(str(payload).encode(), signature)
    graph.resume(**payload)   # invoke LangGraph
    return {"status": "resumed"}
```

These practices keep HITL workflows responsive, cost‑effective, and secure.

<script type="module">
  import mermaid from "https://cdn.jsdelivr.net/npm/mermaid@11.4.1/dist/mermaid.esm.min.mjs";
  mermaid.initialize({ startOnLoad: true });
</script>

