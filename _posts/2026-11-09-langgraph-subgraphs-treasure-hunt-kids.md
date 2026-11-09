---
layout: "post"
title: "Explaining LangGraph Subgraphs with a Fun Multi‑Agent Treasure Hunt"
date: "2026-11-09"
description: "Learn how LangGraph subgraphs work through a fun multi‑agent treasure hunt. Kids see reusable rooms, supervisors, and cost tips in a simple example."
tags: ["langgraph", "tutorial", "kids", "subgraphs", "ai"]
permalink: "/posts/langgraph-subgraphs-treasure-hunt-kids/"
---

## What Is a Subgraph and Why It Makes AI Play Nice Together

A **subgraph** is a tiny “room” inside the big “house” of an AI program. Inside the room you keep a few related steps—like looking at a map, giving a clue, or solving a puzzle—so the rest of the house doesn’t need to know the details. This definition comes straight from the official LangGraph guide.

LangGraph lets you **plug** that room into a larger graph by treating the whole subgraph as a single node. You connect the node’s input and output edges to the parent graph, so agents can enter the room, run its logic, and then leave as if they had walked through one door.

Think of agents as explorers walking down a hallway of rooms. When they reach a door labeled *TreasureMap*, they step into the subgraph‑room, follow the steps inside, and emerge with a new clue. The rooms‑and‑doors picture helps kids see how agents cooperate without getting lost in a tangled maze.

**Benefits of using subgraphs**  
- **Reuse:** The same room can be dropped into many different houses.  
- **Clearer thinking:** Each room has one purpose, so you can focus on one puzzle at a time.  
- **Easier debugging:** If something goes wrong, you only need to check the room where the error happened, not the whole house ([Source](https://mastra.ai/articles/langgraph)).

<pre class="mermaid">
flowchart LR
    Start["Parent Start"] --&gt; Subgraph["Subgraph (room)"] --&gt; End["Parent Continue"]
</pre>

*Figure 1: Shows a parent graph treating a subgraph as a single node with input and output edges.*

## Build a Treasure‑Hunt Game with Three Tiny Agents Using Subgraphs  

- **Overall graph sketch** – The top‑level “Treasure Hunt” graph (the parent) calls three child subgraphs: **MapFinder**, **ClueReader**, and **TreasureCollector**. The parent simply routes the flow: start MapFinder, then hand the result to ClueReader, and finally to TreasureCollector. ([Source](https://mastra.ai/articles/langgraph))  

- **Data flow** –  
  1. The game starts with a `location` like “Beach”.  
  2. **MapFinder** updates `location` to the next waypoint (e.g., “Cave”).  
  3. **ClueReader** adds a `clue` such as “Look under the old oak”.  
  4. Both `location` and `clue` travel to **TreasureCollector**, which checks if the treasure can be reached and sets `found = True` when it does. ([Source](https://medium.com/@rasid2006/building-multi-agent-systems-with-langchain-langgraph-part-3-orchestration-with-langgraph-44b28cb354b8))  

- **State object** – LangGraph’s `State` is a shared list‑like container that lives across subgraphs. It stores keys like `location`, `clue`, and `found`. Each subgraph reads the keys it needs, writes its results, and the parent sees the final state. ([Source](https://shafiqulai.github.io/blogs/blog_13.html))  

- **Step‑by‑step diagram**  

```
Parent
 ├─▶ MapFinder      (location → "Cave")
 ├─▶ ClueReader     (clue → "Old oak")
 └─▶ TreasureCollector (found? → ✅)
```

- **Game outcome** – When `found` becomes `True`, the agents cheer:  
  *MapFinder*: “We reached the cave!”  
  *ClueReader*: “The clue points to the oak tree.”  
  *TreasureCollector*: “🎉 Treasure uncovered under the oak!”

<pre class="mermaid">
flowchart LR
    Start["Start (location)"] --&gt; MapFinder["MapFinder"] --&gt; ClueReader["ClueReader"] --&gt; TreasureCollector["TreasureCollector"] --&gt; Result["Result (found)"]
</pre>

*Figure 2: Parent graph routes location and clue through MapFinder, ClueReader, and TreasureCollector subgraphs.*

## Supervisor Pattern, Human‑in‑the‑Loop, and Edge‑Case Handling

The **supervisor pattern** is a parent subgraph that watches each child agent’s output and decides which node runs next. It acts like a game referee: after an agent finishes, the supervisor checks the result, then routes control to the next agent, a retry, or a human helper ([Source](https://mastra.ai/articles/langgraph)).

**Edge case – empty clue**  
If the Clue agent can’t locate a hint it may return an empty string. The supervisor catches this with an `if/else` branch and tells the Clue agent to try again, up to a limited number of attempts, before moving on.

**Human‑in‑the‑Loop (HITL) node**  
When retries are exhausted, the supervisor pauses the game and asks a kid for help. A simple prompt might be:  

```
“Oops! I’m stuck. Can you type a hint for the treasure?”  
```  

The subgraph waits for the user’s input, then feeds it back to the next agent ([Source](https://shafiqulai.github.io/blogs/blog_13.html)).

**Catch‑all `if/else` branch**  
Any unexpected output (e.g., gibberish symbols) falls into a final `else` clause that logs the problem and either restarts the hunt or ends the session gracefully.

These branches keep the treasure hunt robust: the supervisor retries harmless mistakes, invites a human when the AI is stuck, and safely aborts on truly odd data. For a deeper dive, see the official LangGraph “Supervisor Pattern” documentation ([Source](https://mastra.ai/articles/langgraph)).

<pre class="mermaid">
flowchart TD
    SupStart["Supervisor Start"] --&gt; CheckClue["Check clue output"]
    CheckClue --&gt;|empty| IsEmpty{"Is clue empty?"}
    IsEmpty --&gt;|yes| RetryCount["Retry count &lt; max?"]
    RetryCount --&gt;|yes| Retry["Retry ClueReader"]
    Retry --&gt; CheckClue
    RetryCount --&gt;|no| HumanLoop["Human‑in‑the‑Loop"]
    HumanLoop --&gt; Continue["Continue"]
    IsEmpty --&gt;|no| Continue
    Continue --&gt; End["Supervisor End"]
    CheckClue --&gt;|invalid| LogError["Log error &amp; abort"]
    LogError --&gt; End
</pre>

*Figure 3: Supervisor checks clue output, retries, invokes human assistance, or logs an error before continuing.*

## Performance and Cost: Keeping the Treasure Hunt Fast and Cheap

- **Token = a piece of a sentence.** Every token the model reads or writes adds to the bill ([Source](https://mastra.ai/articles/langgraph)).  

- **Cheap agents first.** A rule‑based Map subgraph runs without LLM tokens, then an expensive LLM subgraph generates clues only when needed ([Source](https://mastra.ai/articles/langgraph)).  

- **Cost estimate:**  
  `cost ≈ (total_tokens ÷ 1,000) × price_per_1k_tokens` — lets you predict dollars before the game starts ([Source](https://mastra.ai/articles/langgraph)).  

- **Speed tip:** Run the Map subgraph **synchronously** (it’s cheap) and start the Treasure subgraph with **async nodes** so clue agents execute together, reducing overall latency ([Source](https://medium.com/@rasid2006/building-multi-agent-systems-with-langchain-langgraph-part-3-orchestration-with-langgraph-44b28cb354b8)).  

- **Log token usage:** Attach LangGraph’s built‑in callback `on_token_usage` to any node.  
  ```python
  from langgraph import CallbackManager

  def on_token_usage(tokens):
      print(f"Tokens used this step: {tokens}")

  callbacks = CallbackManager(on_token_usage=on_token_usage)
  graph = MySubgraph(callbacks=callbacks)
  ```  
  This prints the token count for each step ([Source](https://shafiqulai.github.io/blogs/blog_13.html)).

## Debugging and Observability: Seeing Inside the Subgraph Playground

LangGraph ships with a built‑in **Tracer** that records every node’s input and output. Wrap your graph with `Tracer()` and the console will print a tidy table of state changes, so you can see exactly where the treasure‑hunt went off track ([source](https://mastra.ai/articles/langgraph)).  

If you like pictures, feed the same run data into **graphviz**; it draws the subgraph so you can spot the order of agents in a single glance.  

**Common bugs** you’ll meet:  

- **Missing state keys** – an agent looks for `clue` that was never stored.  
- **Infinite loops** – a branch condition stays true and sends the flow back forever.  
- **Mismatched return types** – a node returns a string while the next expects a dict, causing a crash.  

A fast way to peek inside is to sprinkle `print()` or `logging.debug()` statements in any node; the messages appear in the tracer output ([source](https://shafiqulai.github.io/blogs/blog_13.html)).  

**Debugging checklist**  
1. **Check state** – does every required key exist? (look for missing clues)  
2. **Look for missing clues** – are branch conditions producing the expected path?  
3. **Watch token usage** – too many tokens can truncate messages and hide errors ([source](https://latenode.com/blog/ai-frameworks-technical-infrastructure/langgraph-multi-agent-orchestration/langgraph-multi-agent-orchestration-complete-framework-guide-architecture-analysis-2025)).  

Following these steps lets kids watch the hunt unfold and fix problems before the pirates win.

<script type="module">
  import mermaid from "https://cdn.jsdelivr.net/npm/mermaid@11.4.1/dist/mermaid.esm.min.mjs";
  mermaid.initialize({ startOnLoad: true });
</script>

