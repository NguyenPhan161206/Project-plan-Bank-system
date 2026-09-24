# Modular Monolith vs Microservices — A Decision Framework

**Created Date:** 2026-09-23
**Last Updated:** 2026-09-23
**Tags:** `#SystemArchitecture`, `#ModularMonolith`, `#Microservices`, `#DomainDrivenDesign`, `#FitnessFunctions`, `#Independence`
**References:** [Martin Fowler — MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html), [Sam Newman — Monolith to Microservices](https://samnewman.io/books/monolith-to-microservices/), [DDD — Eric Evans](https://www.domainlanguage.com/), [Pure vs applied modularization — Vlad Khononov](https://vladikk.com/2018/02/11/modular-monolith/)
**Status:** Published
**Difficulty:** Intermediate
**Project:** Modular Bank Operations System — module map for bank bounded contexts (KYC, payments, AML approval, ledger, reporting).

---

## 📌 Key Takeaways (TL;DR)

- The parent idea wants "components highly independent **and** tasks linked through a solid back-end" — this note resolves that tension: independence is a property of **boundaries + contracts**, not of *how many servers you run*.
- **Modular monolith** = one deployable unit with *hard module boundaries* (enforced by tests/linters) and isolated data per module. It delivers ~80% of the independence benefits at ~20% of the operational cost — the right default for a small team / one-builder company system.
- **Microservices** buy independent scaling, independent deployment, and fault isolation — but each service costs an operator's salary in monitoring, networking, versioning, and testing. Premature splitting is the most expensive mistake in this space.
- **The distributed monolith** is the anti-pattern to avoid at all costs: many services that are *tightly coupled* in data and calls — worst of both worlds.
- Split when **fitness functions / evidence** tell you to (team autonomy, scaling a hotspot, regulatory isolation), not when the architecture *feels* cleaner.
- Conway's Law is the deciding force: **the structure of the system will mirror the structure of the team** — with one builder/team, keep the module count small and celebrate the boundaries.

---

## 🧠 Detailed Notes

### 1. Core Concepts

**Monolith** — one process, one deployable: all code in one codebase unit. Simple ops, simple debugging, but boundary discipline is *manual* — over years, the "big ball of mud" forms.

**Modular monolith** — still one deployable, but the codebase is organized into **modules with enforced boundaries**:
- Each module owns its **data** (own tables/schemas; no cross-module table access).
- Modules communicate through **explicit interfaces** (Python `internal`-only exports, `from module import` restrictions, REST/function APIs).
- **Import rules are machine-checked** (e.g., `import-linter` for Python) so nobody accidentally couples modules.
- Can be extracted into microservices **later** — a modular monolith is a microservices-friendly shape, not a dead end.

**Microservices** — many deployables, each independently deployable/scalable/failure-isolated, communicating over the network (usually events/REST).

**Bounded context (DDD)** — the conceptual boundary of a domain model: the *same word* (e.g., "Invoice") may mean different things in different contexts, and each context owns its model. Bounded contexts are the *granularity* guide for both modular monoliths and microservices.

**Distributed monolith** — services that look like microservices but share a database, call each other synchronously in deep chains, and must be deployed together. Hardest system to operate, zero benefits.

**Fitness functions** — automated checks that *guard* architectural constraints over time (dependency rules, max latency, no cross-module DB queries). They convert "we value modularity" into verifiable CI gates.

### 2. How It Works (Mechanism)

**The independence-integrating recipe (same for both architectures):**

1. **Find the bounded contexts** — the real contours of your domain: `Billing`, `Inventory`, `Ledger`, `Approval`, `Reporting`…
2. **Give each context its data** — no shared tables; cross-context data flows as *events* (see [[01-event-driven-architecture-task-orchestration]]) or through explicit APIs.
3. **Enforce boundaries mechanically** — dependency rules in CI; a module may only import its own packages + published interfaces.
4. **Version the contracts** — every public interface/event schema follows SemVer; breaking change = major version + migration window.
5. **Measure with fitness functions** — e.g., "no module imports `ledger.db` directly", "event latency < 500 ms p95", "module X has ≥ 1 release train test".

**The decision pathway:**

```
Do you have a concrete second consumer / clear scaling hotspot / independent team?
        │
   ┌────┴─────────────────────────┐
   ▼                             ▼
  NO                            YES
  │                             │
  ▼                             ▼
Start MODULAR MONOLITH   Consider splitting ONLY that
(boundaries + contracts  bounded context into a service
already in place)        (strangler: one module at a time)
        │
        └──▶ Distributed monolith if you split without boundaries/contracts ❌
```

### 3. Implementation (Pseudocode / Structure)

**A modular monolith layout in Python:**

```
app/
├── billing/
│   ├── domain/            # entities, policies (no framework imports)
│   ├── application/       # use-cases (orchestration logic)
│   ├── infrastructure/    # DB adapters, external clients
│   └── public_api.py      # THE only thing other modules may import
├── ledger/                # (same shape)
├── approval/              # (same shape)
└── shared_kernel/         # truly cross-cutting: ids, money, time
```

**Enforced boundary (import-linter example):**

```python
# .importlinter
[Main]
root_packages = ["app"]
include_external_packages = false
ignore_imports = ["app.shared_kernel.* -> app.*"]

[Contracts.guard-module-data]
type = forbid
source_modules = ["app.billing.infrastructure"]
forbidden_modules = ["app.ledger.infrastructure"]
reason = "Modules must never touch each other's data layer."
```

**A fitness function (CI check):**

```python
# ci/gate_modularity.py — fails the build if boundaries are violated
from importlinter import lint_filesystem
failures = lint_filesystem(".importlinter")
if failures:
    raise SystemExit(f"Modularity gate FAILED: {failures}")
```

**When you later split a context out (strangler):**

```python
# step 1: route ONLY Billing traffic to the new service behind a feature flag
if feature_flag("billing_service"):
    return await remote("billing")  # new microservice
return billing_local()               # old module — keep until traffic drains
```

### 4. Practical Application — for the Company Task System

**My recommendation: modular monolith first.** Reasons:

1. **One builder / small team** → microservices' operational cost (monitoring, deployment, versioning, network testing, distributed debugging) is unpayable by one person.
2. The whole idea's *value* is in the domain workflows and AI integration — spend effort there, not on ops.
3. A modular monolith **already delivers** the idea's key qualities: high component independence (bounded contexts + enforced boundaries + isolated data), easy modification (one codebase to refactor, contracts versioned), long-term development (fitness functions guard evolvability).
4. When scale/team becomes real → **strangler-extract** the busiest bounded context `Billing` into a service, one at a time, without rewriting the system.

**Concrete module map for v1:**

| Bounded context | Owns | Publishes events | 
|-----------------|------|------------------|
| `billing` | invoices, payables, receipts | `invoice.approved`, `payment.captured` |
| `approval` | human/agent approval states, gates | `invoice.approved` |
| `ledger` | double-entry posts | `ledger.posted` |
| `reporting` | projections/dashboards (reads events) | — |
| `ai_ops` | model calls, eval logs, guardrails | `extraction.done` |

Every arrow between contexts is an **event on the bus** — independence inside, linkage through contracts. That is the parent idea, actualized.

### 5. Comparison Table

| Dimension | Classic Monolith | Modular Monolith | Microservices |
|-----------|------------------|------------------|---------------|
| Independent deployment | ❌ | One unit (modules share deploy) | ✅ per service |
| Independent scaling | ❌ | ❌ (scale whole app) | ✅ per hotspot |
| Fault isolation | ❌ (crash = all) | ❌ (shared process) | ✅ partial |
| Boundary enforcement | Manual (drifts) | **Machine-checked** ✅ | Network = enforced (but costly) |
| Operational complexity | Low | **Low–Medium** | High |
| Refactoring cost | Low (one repo) | Low–Medium | High (cross-service changes) |
| Data independence | ❌ shared DB | ✅ per-module schemas | ✅ per-service DB |
| Distributed-debugging pain | None | None | High |
| Team autonomy (per module) | None | Medium | High |
| Best for | Prototypes | **Small teams, domain-heavy systems** | Large orgs, real scaling need |

---

## 📝 Research Journey

- **Why:** The parent idea's central tension — "components highly independent" while "tasks interlink" — forced me to finally understand that independence and integration are *not opposites*; they're two sides of the same contract. This note is me working that out properly.
- **Struggle:** I spent a long time believing microservices = "good architecture" as an absolute. Every conference talk made monoliths sound shameful. The mental flip required separating *architecture quality* (boundaries, contracts, data ownership) from *deployment topology* (one process vs many).
- **Aha moments:** (1) The **distributed monolith is by far the worst option** — microservices without bounded contexts are just a slow monolith with a network. (2) **Conway's Law**: a solo builder's "independent services" would just be folders with extra pain — the honest independence unit for me is *module + enforced boundary*. (3) Fitness functions turn architectural values into CI gates — architecture becomes *testable*, which is the same spirit as the ML evaluation gates I already know from MLOps.
- **Career link:** Interviewers and real products alike probe this decision constantly; being able to *defend a modular-monolith-first strategy with fitness functions* is exactly the senior engineering judgment an AI Engineer in business systems needs.

## 🔗 Related Topics

- [01-project-idea-and-evaluation.md](01-project-idea-and-evaluation.md) — the project idea (Insight 3: independence needs contracts; Insight 7: Conway's Law).
- [02-event-driven-architecture-and-task-orchestration.md](02-event-driven-architecture-and-task-orchestration.md) — the contract/event layer that lets independent contexts *link*.
- [04-mcp-and-function-calling-agent-interfaces.md](04-mcp-and-function-calling-agent-interfaces.md) — how agents expose/consume the same module APIs.
- *(external, Researching Diary)* `books_summaries/agile_project_management/notes/07-agile-architecture-hal-refactoring-scaling.md` — evolutionary architecture, refactoring, and scaling frameworks that govern *when* to split.
- *(external, Researching Diary)* `iot_aiot/03-hardware-abstraction-layer-hal.md` — the same boundary discipline at hardware level (HAL = hardware's contract layer).
- [SUGGESTED] **Bridge note: "Bounded contexts in banking (KYC vs payments vs ledger vs AML)"** — how to *discover* the bank's real module boundaries before writing modules. *Why important:* boundary mistakes are the most expensive kind; DDD discovery is the antidote.

## 🤔 Open Questions

- [ ] Which bounded contexts are *provably* wrong in my proposed v1 module map — what company processes would reveal that?
- [ ] What's the cheapest fitness-function stack for a solo Python project (import-linter + structural tests + contract tests)?
- [ ] When does a *single* company system genuinely need microservices (regulatory isolation? multi-tenant burst?) — revisit with evidence.
- [ ] How do AI eval gates (from MLOps) double as modularity fitness functions in `ai_ops`?

---
*Independence is not how many processes you run; it's how few reasons your modules have to change together.*