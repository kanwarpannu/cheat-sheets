# Creating Agents Best Practices

## Prompt Engineering
The practice of crafting inputs to guide a model toward better outputs.

- **System prompt:** Instructions given to the model before the conversation begins, setting its role, tone, or constraints.
- **User prompt:** The actual input/question from the user during a session.
- **Zero-shot:** Asking the model to complete a task with no examples — just the instruction. Works well for simple tasks where the model already has strong priors.
- **One-shot:** Providing exactly one input/output example before the task. Useful when the desired format or tone isn't obvious from the instruction alone.
- **Few-shot (multi-shot):** Providing multiple examples (typically 3–5) to establish a pattern the model follows. More examples improve consistency on complex or niche tasks.
- **Chain-of-thought:** Asking the model to "think step by step" before giving a final answer, which improves accuracy on reasoning tasks.

### <u>Prompt Engineering Techniques</u>
This works with both text and image inputs.

**Use a list for instructions**  
Break multi-part instructions into a numbered or bulleted list. Prose paragraphs cause models to miss or merge steps; a list forces each requirement to be addressed independently.

**Be specific**  
Replace vague qualifiers with concrete constraints — specify length, format, tone, audience, and scope. Vague instructions produce inconsistent outputs.
```
❌ "Write a short summary."
✓  "Write a 2-sentence summary for a non-technical audience."
```

**Use XML tags**  
Wrap distinct sections of your prompt in XML-style tags (`<instructions>`, `<context>`, `<example>`) to help the model parse each part independently. Especially effective with Claude — it reduces the model conflating instructions with input data.
```xml
<instructions>Classify the sentiment of the review below.</instructions>
<review>{{user_review}}</review>
```

**One-shot / multi-shot**  
See the one-shot and few-shot definitions above. As a technique: choose your examples deliberately — pick inputs that represent the hardest cases or the exact output format you need, not the easiest ones.

### <u>Prompt Evaluation Workflow</u>
An iterative feedback loop for improving prompts systematically rather than by intuition. Evals are effectively the test cases for your prompts — important enough to include in the codebase.

Flow: `Draft prompt → Create eval dataset → Run LLM → Grade outputs → Analyze & revise → Repeat`

- **Draft a general prompt** — Write an initial prompt capturing your intent. Keep it simple; don't over-engineer on the first pass. You'll learn what's missing from the eval results.
- **Create eval dataset** — Curate a set of representative test inputs covering common cases, edge cases, and failure modes. Typically 10–50 examples. Use an LLM to generate them.
- **Feed to LLM** — Run every eval input through the model with your current prompt and collect the full set of outputs.
- **Feed to Grader** — Score each output against expected criteria. The grader can be another LLM (LLM-as-judge), a rule-based scorer (regex, exact match), or human review. Produces a pass/fail or numeric score per example. Use an LLM to create the grader.
- **Analyze & revise** — Review where the prompt fails, identify patterns (e.g., wrong format, missing constraint), update the prompt, and run the cycle again. Generate an `output.html` file per run so results can be reviewed manually as well.

---

## Agent State Management (FSM)
A Finite State Machine (FSM) models an agent as a set of discrete states with defined transitions between them. Rather than letting control flow emerge from ad hoc logic, FSMs make an agent's lifecycle explicit and predictable.

- **State:** A distinct phase the agent can be in at any moment — e.g., `idle`, `planning`, `executing`, `waiting`, `done`, `error`. Only one state is active at a time.
- **Transition:** A rule that moves the agent from one state to another, triggered by an event or condition — e.g., `planning → executing` when a plan is approved.
- **Event / Trigger:** The signal that causes a transition — a user message, a tool result, a timeout, or a condition becoming true.
- **Guard:** An optional condition that must be true before a transition fires — e.g., only move to `executing` if the plan passes validation.
- **Entry/Exit action:** Logic that runs when entering or leaving a state — useful for initializing context or cleaning up resources.

**Why FSMs for agents:**  
LLM calls are non-deterministic — without structure, agent behavior becomes hard to trace or recover. An FSM gives you a map: you always know where the agent is, what it can do next, and why it moved.

### <u>Common Agent State Lifecycle</u>

```
[idle] ──(user input)──▶ [planning] ──(plan ready)──▶ [executing]
                                                           │
                              ┌────────────────────────────┤
                              │                            │
                         (tool result)              (needs input)
                              │                            │
                         [executing] ◀──────────── [waiting]
                              │
                         (task complete)
                              │
                           [done]
                              │
                         (any error)
                              │
                           [error] ──(retry)──▶ [planning]
```

Flow: `idle → planning → executing → (waiting ↔ executing) → done`  
On failure at any state: `→ error → retry (planning) or abort (idle)`

### <u>FSM Design Tips</u>

**Keep states coarse-grained**  
Model agent lifecycle phases, not every micro-step. If you have more than ~7 states, you likely have sub-states that should be nested or collapsed.

**Make transitions explicit**  
Define every valid `(from, event) → to` triple upfront. Any transition not in the table should be rejected — it catches unexpected model outputs before they corrupt state.
```
✓  (executing, tool_error)   → error
✓  (waiting,  user_reply)    → executing
❌  (done,     tool_result)   → ??? (invalid — reject and log)
```

**Persist state for long-running agents**  
Store the current state and its context (pending plan, tool results, conversation history) in durable storage after every transition. If the agent crashes or restarts, it resumes from the last known state rather than starting over.

**Use `error` as a first-class state**  
Don't swallow failures inside `executing`. Transition explicitly to `error` with the failure reason attached. From `error`, the agent can retry, escalate, or abort — but the decision is deliberate, not silent.

### <u>Audit & Traceability</u>
Every state transition is a natural audit checkpoint. Logging at transition boundaries gives you a full, ordered record of what the agent did and why — without instrumenting every internal line of code.

- **Trace ID:** A unique identifier assigned at the start of each agent run and attached to every event, transition, and log line. Lets you reconstruct the full execution of a single run across distributed logs.
- **Transition log:** An append-only record of `(from_state, event, to_state, timestamp, context_snapshot)`. The sequence of entries is the agent's execution history.
- **Context snapshot:** The relevant inputs and outputs captured at each transition — tool call arguments, tool results, LLM reasoning, user messages. Stored alongside the transition, not just the final output.
- **Causality chain:** Each log entry references the event that triggered it, so you can trace *why* the agent moved to a given state, not just *that* it did.

**What to log at each transition:**
```
{
  trace_id:    "run-abc123",
  from:        "planning",
  to:          "executing",
  event:       "plan_approved",
  timestamp:   "2025-08-06T10:42:00Z",
  duration_ms: 340,
  inputs:      { plan: "..." },
  outputs:     { first_tool_call: "..." },
  error:       null
}
```

**Replay & debugging**  
A complete transition log lets you replay a run deterministically — feed the same inputs back through the same state sequence to reproduce a failure. Without it, debugging a non-deterministic LLM agent is guesswork.

**Human-in-the-loop checkpoints**  
Flag specific transitions as requiring human approval before they fire — e.g., `planning → executing` for irreversible actions (sending an email, writing to a database). The audit log records whether approval was granted, by whom, and when.

---

## Tools
Tools (also called functions or actions) are external capabilities exposed to the agent — API calls, database queries, file operations, web searches — that let it take actions beyond text generation. The agent decides when to call a tool, what arguments to pass, and how to act on the result.

- **Tool schema:** A structured definition the model sees — the tool's name, a description of what it does, and the parameters it accepts. Schema quality directly determines how reliably the model invokes it correctly.
- **Deterministic tool:** Returns the same output for the same input every time — e.g., a math calculation, a database read by primary key. Prefer deterministic tools where possible: they are easier to test, debug, and reason about, and they reduce token waste from unexpected or variable results. This is not a hard requirement, but it meaningfully reduces turns and tokens per task.
- **Non-deterministic tool:** Output varies with context — e.g., a web search, a call to another LLM, a timestamp. Use when necessary, but validate results and handle errors explicitly.
- **Idempotent tool:** Can be called multiple times with the same arguments without changing the outcome beyond the first call. Critical for write operations that may be retried.

### <u>Tool Design</u>

**Name tools with a verb-first pattern**  
Use `snake_case` with an action prefix that describes what the tool does. The model infers intent from the name, so precision matters. Group related tools with a consistent prefix.
```
✓  get_customer_by_id
✓  send_slack_message
✓  search_products
❌  data
❌  helper
```

**Write descriptions like you're briefing a new teammate**  
The description is what the model reads to decide whether and how to call the tool. Explain what it does, when to use it, what the parameters mean, and any gotchas — query format restrictions, what "not found" looks like, when to prefer this tool over a similar one. Small refinements to descriptions yield the largest accuracy improvements.
```
❌  "Search for products."
✓  "Search the product catalog by name, SKU, or category. Returns up to 20 results
    ordered by relevance. Use exact SKU for precise matches. Returns an empty list if
    no results are found — this is not an error."
```

**Keep required parameters minimal**  
Mark a parameter as required only if the call will fail without it. Every extra required field is a failure point — the model may hallucinate a value rather than leave it blank. Use sensible defaults for everything optional.

**Use enums to reduce ambiguity**  
Constrain parameters with a fixed set of values wherever the domain allows. Enums prevent hallucinated values, eliminate disambiguation turns, and make the model's choice deterministic within that field.
```json
"response_format": {
  "type": "string",
  "enum": ["summary", "full", "json"],
  "default": "summary"
}
```

**Implement pagination and truncation with sensible defaults**  
Return a small number of results by default (e.g., 10–20) with a cursor or page parameter. The model can always request more. Returning 200 records when 5 suffice is a token tax on every call.

### <u>Reliability & Error Handling</u>

**Feed errors back to the model, not exceptions**  
When a tool fails, return a structured error with enough context for the model to self-correct — wrong parameter type, value out of range, resource not found. Most format errors resolve on the first retry when the model gets a clear error message.
```json
{
  "error": "invalid_parameter",
  "field": "date_range",
  "message": "end_date must be after start_date. Received: start=2025-09-01, end=2025-08-01"
}
```

**Make write operations idempotent**  
Tools that create or modify state should tolerate being called twice with the same inputs without double-writing. Key writes with a composite idempotency key (e.g., `run_id + step_index + action_type`) so retries are safe.

### <u>Security</u>

**Validate inputs at the tool boundary**  
Never trust agent-generated arguments. Tool calls are the primary attack surface for prompt injection — a malicious document in the context can instruct the agent to invoke a tool with crafted parameters. Validate all inputs before executing, regardless of how they arrived.

**Limit each agent's toolkit to what it needs**  
Don't expose every available capability to every agent. Give each agent the minimum set of tools required for its task. A scoped toolkit limits blast radius if the model is manipulated or makes an error (principle of least privilege applied to tools).

---

## RAG (Retrieval-Augmented Generation)
A pattern where relevant documents are fetched from an external knowledge base and injected into the prompt before the model generates a response.

**Why it matters:** Models have a training cutoff and cannot access private data. RAG gives them up-to-date or domain-specific context without retraining.

**Semantic Search**
Documents are first chunked, then each chunk is converted into a vector using an embedding model and stored in a vector database. At query time, the query is embedded and compared against stored vectors by similarity (e.g. cosine). Retrieve by meaning, not exact wording — "leave policy" finds relevant chunks even if they say "vacation entitlement". Best for conceptual or paraphrased queries.

**Lexical Search (BM25)**
A keyword-based ranking algorithm that scores chunks by term frequency and how rare the term is across all chunks (inverse document frequency). Unlike semantic search, it matches exact or near-exact terms. Best for precise lookups — product codes, proper nouns, acronyms — where semantic similarity can mislead. Use when the query contains specific identifiers that must appear verbatim.

**Hybrid Retrieval with RRF**
Running both approaches produces two independently ranked lists. Reciprocal Rank Fusion (RRF) merges them into a single ranking by combining each chunk's position in both lists — a chunk that ranks highly in both scores gets the strongest combined score. Hybrid retrieval is more robust than either method alone: semantic search catches conceptual matches while BM25 anchors precise terms.

Flow: `User query → [Embed → Semantic search] + [BM25 lexical search] → RRF merge → Inject top results into prompt → Generate response`

---

## Memory (in AI Agents)
Agents need memory to reason across steps, sessions, and knowledge bases. Without it, every call starts from scratch — the agent has no awareness of what it did before or what it already knows.

**1. Ephemeral Memory**  
Exists only during a single step or tool call, then is discarded immediately.  
- **Storage:** In-process variables (RAM only) — no persistence of any kind.  
- **Use case:** A ReAct-style agent's scratchpad — intermediate reasoning steps and tool outputs held while composing a final answer. Once the response is returned, this memory is gone.  

**2. Short-Term Memory**  
Lives for the duration of a conversation session.  
- **Storage:** In-memory list of messages (conversation history) passed in the context window each turn. Some frameworks also store it in a session cache (e.g., Redis) tied to a session ID.  
- **Use case:** Multi-turn chat where the model refers back to what was said earlier in the same conversation — e.g., "as I mentioned above…". The context window is the hard limit on how much short-term memory exists.  

**3. Long-Term Memory**  
Persists across sessions — survives when the conversation ends and is available in future ones.  
- **Storage:** Vector databases (Pinecone, pgvector, Chroma) for semantic retrieval; relational databases (Postgres, SQLite) for structured facts; plain files for simple key-value recall. Retrieved into context via RAG.  
- **Use case:** A personal assistant agent that remembers your name, preferences, and past decisions. A support bot that recalls previous tickets. An autonomous agent that logs completed steps to avoid repeating work.  

**Agent Memory Flow:**  
`User input → Read long-term memory (RAG from vector DB) → Load short-term memory (conversation history) → Process with ephemeral scratchpad (reasoning steps) → Respond → Write important outcomes back to long-term memory`
