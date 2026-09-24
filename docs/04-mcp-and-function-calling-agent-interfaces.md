# MCP & Function Calling — Designing Interfaces for AI Agents

**Created Date:** 2026-09-23
**Last Updated:** 2026-09-23
**Tags:** `#AIAgents`, `#MCP`, `#FunctionCalling`, `#ToolUse`, `#AgentInterfaces`, `#LLM`, `#StructuredOutput`
**References:** [Model Context Protocol (Anthropic)](https://modelcontextprotocol.io/), [OpenAI Function Calling docs](https://platform.openai.com/docs/guides/function-calling), [JSON Schema](https://json-schema.org/), [Ollama Tools](https://docs.ollama.com/guides/function-calling)
**Status:** Published
**Difficulty:** Intermediate
**Project:** Modular Bank Operations System — the governed agent interface (MCP + CLI `--json`) through which AI agents operate bank workflows.

---

## 📌 Key Takeaways (TL;DR)

- An **agent interface** is not a CLI — it is a *machine-readable contract* (JSON Schema tools, structured output) that lets an LLM discover capabilities at runtime and call them safely. The CLI is just one *face* of that interface (the human/scripting face).
- **Function calling (tool use):** the LLM outputs a structured call (`tool: name`, `arguments: {...}`) instead of prose; the app executes it and feeds the result back. The loop is: *prompt → tool decision → execution → observation → next decision*.
- **MCP (Model Context Protocol)** standardizes how an agent *host* connects to *tool servers*: one protocol, any tool, any agent (Claude, opencode, custom). Its three primitives — **tools** (actions), **resources** (read-only data), **prompts** (templates) — map 1:1 onto a company task system.
- **The "CLI để AI Agent thao tác" requirement should be built as: protocol (MCP) + CLI with `--json` + governed permissions.** "Agents operating freely" must be replaced by "agents operating within audited, least-privilege envelopes".
- Safety is architectural, not afterthought: **tool allowlists, read-only roles, approval gates for consequential tools, immutable audit logs, and prompt-injection defenses** are part of the interface design itself.

---

## 🧠 Detailed Notes

### 1. Core Concepts

**Function calling (tool use)** — the mechanism by which an LLM *invokes your system*:
1. You declare available tools as JSON Schemas (name + description + parameter schema).
2. The model sees tool descriptions in context; if appropriate, it replies with a *structured* `tool_calls` payload — **not text** — containing the arguments.
3. Your app validates the arguments, executes the real function, sends the result back to the model.
4. The model continues (calls more tools or produces the final answer).

Key insight: **the tool schema is the API contract with an LLM.** The quality of `description` and `parameters` determines whether the model calls tools correctly. Bad schema ⇒ model invents arguments or refuses tool use.

**Structured outputs** — forcing the model's *final* answer to also be JSON (validated against a schema), so downstream systems can consume results without parsing prose. Essential when an LLM output feeds another task in your pipeline.

**Model Context Protocol (MCP)** — an open standard (Anthropic, Nov 2024) for connecting agents to tools/data. Architecture:

```
┌──────────────┐    MCP    ┌──────────────────┐
│   MCP Host   │ ◀───────▶ │   MCP Server(s)  │
│  (agent app: │  JSON-RPC │  (your tools/    │
│   Claude,    │  stdio /  │   data sources)  │
│   opencode…) │  HTTP/SSE │                  │
└──────────────┘           └──────────────────┘
   │ (multiple clients)
   ▼
 MCP Client(s)  ── connect to n servers, each exposing:
        • tools      → actions (e.g., approve_invoice)
        • resources  → read-only data (e.g., ledger:current)
        • prompts    → reusable templates (e.g., "audit a task chain")
```

**Three MCP primitives vs your system:**

| MCP primitive | Meaning | Company Task System example |
|---------------|---------|------------------------------|
| **Tool** | callable action (mutating or not) | `approve_invoice(id)`, `post_to_ledger(id)`, `generate_report(period)` |
| **Resource** | read-only data with a URI | `ledger://current`, `invoice://{id}/status`, `agent://session/{id}/log` |
| **Prompt** | reusable instruction template | `audit_chain`, `draft_response_to_customer` |

**CLI + `--json`** — your system's human/scripting face: `taskcli invoice approve --id INV-42 --json`. The CLI is a *thin client over the same contract* that MCP serves. Same engine, three consumers (human, script, agent).

### 2. How It Works (Mechanism)

**The tool-call loop (simplified):**

```
User: "Approve invoice 42 and post it to the ledger, then tell me the balance."
  │
  ▼
[Agent] ──tools: [approve_invoice, post_to_ledger, get_balance]──▶ [LLM]
  │                                                            │
  └──────────────────  tool_calls: ◀───────────────────────────┘
      [{name: approve_invoice, args: {invoice_id: "42"}}]       (structured!)
      ├─▶ [Your system: validate → execute → result "ok, approved"]
      │
      ▼  (result fed back, loop repeats)
      └── tool_calls: [{name: post_to_ledger, args: {...}}]  … etc.
```

**MCP request flow (stdio transport):**

1. **Initialize:** client ↔ server handshake, version negotiation, capability negotiation.
2. **List tools:** client asks `tools/list` → server returns JSON Schema of every tool.
3. **Call:** client sends `tools/call` with tool name + arguments → server executes → returns structured result or error.
4. **Notifications/resources:** server can push `resources/list_changed`; client fetches resources via `resources/read`.

**Why MCP beats per-vendor glue:** instead of writing N adapters (OpenAI SDK, Anthropic SDK, local models, home-grown agent frameworks), you write **one MCP server** per capability set and any MCP-capable host can drive it. This is the standardization layer the agent-economy was missing.

### 3. Implementation (Pseudocode)

**A tool definition as JSON Schema (this IS the agent contract):**

```json
{
  "name": "approve_invoice",
  "description": "Approve an invoice for payment. Requires the invoice to be in 'validated' state and the caller to hold role 'approver'. Consequential action - requires explicit consent when run by an autonomous agent.",
  "input_schema": {
    "type": "object",
    "properties": {
      "invoice_id": {"type": "string", "pattern": "^INV-[0-9]+$"},
      "reason": {"type": "string", "minLength": 10}
    },
    "required": ["invoice_id", "reason"],
    "additionalProperties": false
  }
}
```

**A minimal MCP server (FastMCP-style pseudocode):**

```python
from mcp.server.fastmcp import FastMCP   # conceptual; SDK equivalent

mcp = FastMCP("company-tasks")

@mcp.tool()
def approve_invoice(invoice_id: str, reason: str) -> str:
    """Approve an invoice for payment (role-checked, audit-logged)."""
    if not current_agent_has_role("approver"):
        return {"error": "FORBIDDEN", "audit_id": audit.begin(invoice_id, "denied")}
    result = saga_coordinator.advance("invoice.approved", invoice_id, by=current_agent())
    audit.log("approve_invoice", invoice_id, by=current_agent(), result=result)
    return result

@mcp.resource("ledger://current")
def current_balance() -> dict:
    """Read-only snapshot of the ledger (never mutates)."""
    return ledger.snapshot()

# Transport: stdio (local) or streamable HTTP — same server, either way.
```

**The governed agent-loop wrapper (safety as architecture):**

```python
async def run_with_guardrails(goal: str, allowed_tools: set[str], require_human: set[str]):
    for step in agent_loop(goal, tools=allowed_tools):        # 1. tool allowlist
        if step.tool in require_human:                        # 2. approval gates
            await human_approve(step)                          #    (consequential tools)
        if not audit.within_budget(step):                     # 3. cost/latency budget
            abort("budget exceeded")
        outcome = await execute(step)                          # 4. execute
        await audit.record(step, outcome)                      # 5. immutable trail
```

**The CLI face over the same contract:**

```bash
$ taskcli invoice approve --id INV-42 --reason "PO confirmed" --json
{"ok": true, "state": "approved", "next": ["ledger.post.requested"], "audit_id": "a-9f3"}
```

Same engine. The agent hits it through MCP; the human hits it through the CLI; a script hits it through `--json`.

### 4. Practical Application — for the Company Task System

**Turning "CLI để AI Agent thao tác tự do" into a safe design:**

1. **Every capability gets a tool schema** (MCP) — capabilities are *discoverable*, which is what makes "free operation" *capable*, not just legal.
2. **Roles map to tool sets:** `read_only_agent` → resources only; `operator_agent` → non-consequential tools; `approver_agent` → + approval tools requiring human consent. Least privilege, like the DB roles you already know.
3. **Consequential tools call the human gate:** invoice approval, payments, deletions → never executed by autonomous agents without a human ok (configurable per policy).
4. **Everything is audited:** every tool call stored with agent identity, arguments, result, timestamp → the audit trail *is* the evidence an operations desk and regulators need.
5. **Prompt-injection defense:** separate untrusted content (documents, emails being *processed*) from instructions: never let document text become system instructions; load documents as *resources/data*, keep instructions in the system prompt; allowlist tools per task; prefer structured extraction over free-form Q&A on untrusted content.

**The interface stack (the corrected structure from the parent idea):**

```
Core engine (bounded contexts, events, saga)
        │  contract layer: JSON Schema tools + MCP server + REST + CLI --json
        ├─── CLI        (human power-user, scripts)
        ├─── Web UI     (business users — thin client, not part of this note)
        └─── MCP server (agents: Claude, opencode, custom…)
```

### 5. Comparison Table

| Dimension | Function Calling (raw SDK) | MCP | Plain REST/OpenAPI | CLI without --json |
|-----------|----------------------------|-----|--------------------|--------------------|
| Agent discoverability | ✅ (schemas in prompt) | ✅ (`tools/list`) | ⚠️ (needs agent framework) | ❌ |
| Multi-agent reuse | ❌ per-vendor glue | ✅ one server, any host | ⚠️ OpenAPI tooling exists | ❌ |
| Structured output control | ✅ | ✅ | ✅ | ❌ (prose) |
| Resource/data access for agents | ❌ (custom) | ✅ native resources | ✅ but bespoke | ❌ |
| Human usability | ❌ | ❌ (dev-facing) | ⚠️ via Swagger UI | ✅ |
| Complexity | Low | Medium | Medium | Low |
| **Fit for your idea** | Starter 🔧 | **Core contract** ✅ | Companion (reg REST clients) | Face of the CLI ✅ |

Winner for the company system: **MCP as the contract + CLI with `--json` as the face** — because you need agent discoverability AND human usability AND auditability from the same engine.

---

## 📝 Research Journey

- **Why:** The parent idea's boldest sentence is "hệ thống CLI để thao tác tự do bằng AI Agent" — I needed to know whether that's a real engineering pattern or a buzzword. Answer: it's real, but the *contract* (not the CLI) is the technical core.
- **Struggle:** I originally thought agents "used" systems the way humans do (typing in a terminal). The LLM *doesn't type* — it emits structured JSON calls. Realizing the CLI is only a *rendering* of the contract for humans changed my whole design.
- **Aha moments:** (1) **Tool schema = prompt engineering**: the description field IS the model's documentation — the same care you give `README.md` must go into `description`. (2) MCP's three primitives map 1:1 to a business system (tools/resources/prompts = actions/data/templates). (3) **Prompt injection is the #1 agent-system risk**, and it's an *interface-design* problem: content must never become instructions.
- **Career link:** Every serious company building on AI will need agent-operable systems with governance. Being one of the engineers who can *design the interface layer* (MCP + tools + audit) is a direct, differentiated skill for the AI Engineer role in Economics/Business — not just "calling an API".

## 🔗 Related Topics

- [01-project-idea-and-evaluation.md](01-project-idea-and-evaluation.md) — the project idea (Insight 6: CLI is a face; the protocol is the contract; governance).
- [02-event-driven-architecture-and-task-orchestration.md](02-event-driven-architecture-and-task-orchestration.md) — the engine behind the tools (saga, events, audit) that agents drive.
- [03-modular-monolith-vs-microservices.md](03-modular-monolith-vs-microservices.md) — the bounded contexts that define which tools exist and who may call them.
- *(external, Researching Diary)* `ai_ml/05-mlops-lifecycle-and-deployment-architecture.md` — eval gates and monitoring for the *model* side of the loop (drift, errors).
- *(external, Researching Diary)* `random_ideas/04-custom-tui-project.md` — the TUI/CLI project for AGY & opencode — the human-interaction side of this same interface story.
- [SUGGESTED] **"Agentic safety & prompt-injection defense in depth"** — an entire security layer (content/instruction separation, tool allowlists, sandboxing, red-teaming) that mainstream notes skip. *Why important:* this is the difference between a demo and a system a bank can trust with money — regulatory-grade audit matters.
- [SUGGESTED] **"Evaluating agent workflows (eval harness for tool-using LLMs)"** — how to measure whether your agent actually completes task chains correctly (pass@k over real task traces). *Why important:* the MLOps eval mindset applied to agents before touching real bank data.

## 🤔 Open Questions

- [ ] MCP stdio vs streamable HTTP transport for a multi-agent corporate deployment — security implications?
- [ ] How should the tool schema describe *business constraints* (roles, states) so the model rarely attempts forbidden calls?
- [ ] What audit format satisfies both an operations desk and a potential regulator (immutable log, hash-chained)?
- [ ] How to evaluate agent task-completion quality BEFORE letting it touch real data (eval harness, shadow mode)?

---
*The CLI is the human face of a contract; the schema is the agent's face. Design the contract once, wear it both ways.*