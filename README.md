# Project Plan — Modular Bank Operations System

> A domain-specific **bank operations platform** — modular, event-linked, evolvable, and operable by **AI Agents** through a governed CLI/MCP interface.

This repository is the *project home* for turning the **Modular Company Task System** idea (from the [Researching Diary](https://github.com/NguyenPhan161206/reaserching-diary/blob/mine/random_ideas/06-modular-company-task-system-ai-agents.md)) into a concrete banking system. The anchor domain is **banking operations** — the "company with specific expertise" in the original idea is a bank.

---

## 🎯 Vision

Build a large system specialized for **a bank's core operational tasks** (payments, approvals/AML, ledger, reporting). Tasks interlink through a **solid event-driven back-end**, while every component stays **highly independent** — adaptable to regulation and market changes, easy to modify, and built to last. The system **integrates AI** and exposes every capability through a **CLI + MCP protocol** so AI Agents can operate workflows — *within governed, audited permissions*.

## 📊 Docs Index

| Doc | Topic | Answers |
|-----|-------|---------|
| [01-project-idea-and-evaluation.md](docs/01-project-idea-and-evaluation.md) | Idea capture + critical evaluation | Why build it? Strengths, weaknesses, risks to acknowledge |
| [02-event-driven-architecture-and-task-orchestration.md](docs/02-event-driven-architecture-and-task-orchestration.md) | Event-driven backbone + saga orchestration | How do tasks *link* reliably? |
| [03-modular-monolith-vs-microservices.md](docs/03-modular-monolith-vs-microservices.md) | Modularity decision framework + fitness functions | How do components stay *independent*? |
| [04-mcp-and-function-calling-agent-interfaces.md](docs/04-mcp-and-function-calling-agent-interfaces.md) | MCP + function calling + governed agents | How do AI Agents *operate* the system? |

## 🏗️ Architecture Summary (v1 target)

```
                            ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
 Thin clients           →   │  CLI (human/ │  │  Web UI      │  │ MCP server   │
                            │  agent)      │  │ (ops desk)   │  │ (agents)     │
                            └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
                                   └─────────────────┼─────────────────┘
                                                     ▼
                            ┌──────────────────────────────────────────┐
                            │ Protocol/contract layer (JSON Schema,    │
                            │ auth, versioning, immutable audit trail) │
                            └────────────────┬─────────────────────────┘
                                              ▼
   Independent bounded contexts (modular monolith, data-per-context)
   [KYC/Onboarding] ──customer.onboarded──▶ [Payments] ──payment.requested──▶
   [Approval/AML] ──approved/rejected──▶ [Ledger] ──ledger.posted──▶ [Reporting]
                                              ▼
                            ┌──────────────────────────────────────────┐
                            │ Cross-cutting: observability, AI services,│
                            │ identity/permissions, schema registry    │
                            └──────────────────────────────────────────┘
```

- **Backbone:** event-driven with an **orchestrated saga** per business flow (e.g., `PaymentInitiated → AML Check → Approval(human) → PostToLedger → Notify`), at-least-once delivery + idempotent consumers + DLQs.
- **Modularity:** start **modular monolith** with enforced boundaries (import-linter, contract tests); split contexts into services only when fitness functions/evidence demand it (strangler pattern).
- **Agents:** every capability = an MCP **tool** (e.g., `fetch_payment_status`, `approve_payment`), CLI `--json` as the human/scripting face; roles map to tool sets; consequential tools require human approval; all calls audited.

## 🗺️ Roadmap

- **v0 — Payment approval chain (one context, end-to-end):** bounded `payments` + `approval` contexts, orchestrated saga, CLI + `--json`, audit log. Goal: prove the idea + the backbone with the 3 most automatable tasks.
- **v1 — Ledger + reporting by events:** add `ledger` and `reporting` contexts consuming events; schema registry + versioning discipline; human-in-the-loop validation gates for AI components.
- **v2 — Agent governance + AI ops:** MCP server surfaces the full capability set; role-based tool permissions; eval harness & shadow mode before autonomous execution; prompt-injection defense in depth (regulatory-grade audit).

## ⚠️ Top Risks to Acknowledge (see doc 01 for full register)

1. **Compliance & liability** — AI on consequential bank actions; see doc 04 for governance design.
2. **Prompt injection** — agents touching third-party content (documents, SWIFT messages, emails).
3. **GIGO amplification** — linked tasks propagate bad data automatically; provenance + validation gates.
4. **Bus-factor = 1** — solo-built, long-term system: ADRs + small honest scope required.
5. **Unit economics** — AI per-operation cost vs transaction margin must be modeled.

## 📚 Source of Truth

Design notes are maintained in the **[Researching Diary](https://github.com/NguyenPhan161206/reaserching-diary)** — this repo mirrors the project-relevant docs. Changes to the source notes are reflected here.

## 🔗 Related Links

- Researching Diary: <https://github.com/NguyenPhan161206/reaserching-diary>
- Repo owner: Phan Hữu Bình Nguyên (AI Engineer track, Economics/Business)

---
*Idea quality is measured by the tension it resolves, not the features it lists.*