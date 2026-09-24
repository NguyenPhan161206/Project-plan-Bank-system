# Event-Driven Architecture & Task Orchestration

**Created Date:** 2026-09-23
**Last Updated:** 2026-09-23
**Tags:** `#EventDrivenArchitecture`, `#MessageBroker`, `#Saga`, `#TaskOrchestration`, `#SystemDesign`, `#Backend`
**References:** [Saga pattern — Microservices.io](https://microservices.io/patterns/data/saga.html), [Kafka design docs](https://kafka.apache.org/documentation/), [Event-driven architecture — AWS](https://aws.amazon.com/event-driven-architecture/)
**Status:** Published
**Difficulty:** Intermediate
**Project:** Modular Bank Operations System — the "solid back-end" design for bank task chains (payments → AML approval → ledger → reporting).

---

## 📌 Key Takeaways (TL;DR)

- **Events are facts of the past; commands are requests for the future.** "Task B must react to what Task A did" ⇒ emit an event; "Task A please do X" ⇒ send a command. Mixing them is the #1 source of coupled systems.
- A **message broker** (Kafka/RabbitMQ/Redis Streams) is the "solid back-end" of the Modular Company Task System: it gives durability, retries, and auditability — but it only *delivers*, it never guarantees your consumers are correct.
- Three guarantees make a linked system actually solid: **at-least-once delivery + idempotent consumers + dead-letter queues**. Ordering is a bonus that costs throughput — buy it only where the domain needs it.
- Tasks that span components must be wrapped in a **saga** (distributed workflow with compensation). Start with **orchestration** (a central coordinator) for business clarity; use choreography only when teams are mature and workflows are stable.
- **Every schema change is a contract change.** A schema registry + SemVer + deprecation windows is what keeps an event-driven system *evolvable* — which is exactly the "easy to modify long-term" requirement of the parent idea.

---

## 🧠 Detailed Notes

### 1. Core Concepts

**Event vs Command — the fundamental distinction.**

| | Event | Command |
|---|-------|---------|
| Semantics | *Something happened* (past tense, fact) | *Do this* (request, future intent) |
| Producer expects | No reply (fire-and-forget) | A result / acknowledgement |
| Language | `InvoiceApproved`, `PaymentCaptured` | `ApproveInvoice`, `CapturePayment` |
| Failure | Already happened — can't fail | Can be rejected/retried |
| Broker role | Distributor (notify subscribers) | Router (deliver to *one* worker) |

**Why this matters for your idea:** "Các tác vụ liên kết với nhau" (tasks link to each other) is almost always an *event* relationship — Task A completes, therefore Task B starts. If you model that linkage as synchronous command chains (`A → call B → call C`), you build a distributed monolith: one slow/failed task blocks the whole chain. Events give you **temporal decoupling** (B does not wait for A) and **spatial decoupling** (B does not need to know A's address).

**Message broker** — the durable middleman:
- **Kafka** — append-only log, partitioned topics, replayable, huge throughput; great for event streams + audit.
- **RabbitMQ** — classic queue + AMQP exchanges, routing, per-message acknowledgements; great for commands/work queues.
- **Redis Streams** — lightweight, in-memory-ish, good for small/medium systems with low ops overhead.

**Pub/Sub** — publishers don't know subscribers (each subscriber gets its own copy of the event) vs **queue** — one consumer takes each message.

**Event Sourcing** — store the *sequence of events* as the source of truth, derive current state by replaying (projection). Powerful auditability; higher complexity. Optional for v1 — only add when you need the event log as truth (finance/audit-heavy domains).

**CQRS (Command Query Responsibility Segregation)** — separate write models from read models; often pairs with event sourcing; useful for read-heavy dashboards. Optional again.

### 2. How It Works (Mechanism)

A minimal reliable pipeline:

```
[Producer] --publish--> [Broker: topic "payment.captured"] --> [Consumer Group: ledger-service]
                                                              │
                                                              └─ idempotency check → process → commit offset
```

**Guarantee stack:** a broker gives **at-least-once** (delivery retried until acknowledged). That means your consumer may see the same event twice. Therefore: **consumers MUST be idempotent** — applying the event twice has the same effect as applying once (use a unique event/business key stored at the consumer; skip if seen).

**Ordering:** Kafka guarantees order *within a partition* — keyed by the entity (e.g., `invoice_id`), so all events of one invoice land in one partition, in order. Cross-entity global order is basically impossible; the domain almost never needs it.

**Dead Letter Queue (DLQ):** after N failed attempts (poison messages), park the event in a DLQ for manual/after-hours repair instead of blocking the pipeline forever.

**Exponential backoff with jitter** (retry pacing):

$$t_n = \text{base} \cdot 2^n + \text{rand}(0, \text{jitter})$$

Prevents the "thundering herd" where all consumers retry simultaneously after a broker hiccup.

**Schema registry & evolution:**
- All events carry `schema_version`.
- **Additive changes** (new optional field) = backward compatible → safe on the same version.
- **Breaking changes** (field removed/retyped) = new major version + **dual-write window** (produce both versions for N weeks) while consumers migrate.

This is the *mechanical* basis of "thích ứng với thời cuộc / dễ dàng sửa đổi" — the thing that keeps a 5-year-old system safe to evolve.

### 3. Task Orchestration — Choreography vs Orchestration vs Saga

Three ways to link tasks in a workflow:

**A. Choreography (event-driven, no central coordinator):**
```
PaymentSvc → emits "payment.captured" → LedgerSvc reacts → emits "ledger.posted" → ReportSvc reacts
```
- ✅ Max decoupling, no single point of failure, each service owns its logic.
- ❌ The overall business flow is *implicit* — nobody can answer "what happens if payment.captured never arrives?" without reading every consumer. Debugging and business monitoring are hard.
- ⚠️ It is a **distributed transaction without a rollback**: if Ledger fails after Payment succeeded, who fixes the inconsistency?

**B. Orchestration (central workflow engine/coordinator):**
```
Orchestrator: 1. call PaymentSvc (command) → 2. on ok, call LedgerSvc → 3. on failure, call Compensation
```
- ✅ Explicit business flow (easy to read, monitor, and version); recovery is a first-class state machine.
- ❌ The orchestrator becomes a coupling point (a "god service" if overused); every step is a round-trip.
- This matches your idea's instinct: *"các tác vụ có khả năng liên kết với nhau"* with **visibility** — a company operations desk needs to *see* where a task chain is stuck.

**C. Saga = the correct way to do long, multi-component business flows.**

A saga is a sequence of local transactions, each with a **compensation** (undo):

```
PlaceOrder ──▶ ReserveStock ──▶ ChargePayment ──▶ ConfirmOrder
                  │                 │
              (fail)            (fail)
                  ▼                 ▼
           ReleaseStock ────▶ RefundPayment
```

Saga pattern has two incarnations:
- **Orchestrating saga** (central coordinator decides each step + compensations) — recommended start.
- **Choreographed saga** (each service publishes events that trigger the next; compensations via events too) — for mature, decoupled teams.

**Which one for the Company Task System?** Orchestrated saga. Reason: business workflows need *observability, versioning, and pause/resume* (a task chain may wait days for human approval). A central state machine gives all three for free; you can still *emit events from every step* for audit and analytics (events + orchestrator are complementary, not competing).

### 4. Implementation (Pseudocode)

**A minimal orchestrating saga for an invoice-processing task chain:**

```python
# saga_coordinator.py — orchestrating saga
# Workflow: InvoiceReceived → ExtractData(AI) → Validate → Approve(human) → PostToLedger

class InvoiceSagaCoordinator:
    STATES = ("received", "extracted", "validated", "approved", "posted", "compensated")

    def __init__(self, bus, store):
        self.bus = bus        # event bus / broker
        self.store = store    # saga-state store (DB with saga_id as key)

    async def start(self, invoice_id: str):
        saga = self.store.create(invoice_id)
        await self.bus.publish("invoice.received", {"invoice_id": invoice_id,
                                                    "schema_version": 1})

    async def on_event(self, event: dict):
        key = event["invoice_id"]
        saga = self.store.get(key)
        if not self._is_expected(event, saga.state):
            return  # stale/out-of-order event → ignore (idempotency)

        if event["type"] == "invoice.extracted" and event["ok"]:
            saga.state = "extracted"
            await self.bus.publish("invoice.validation.requested", {"invoice_id": key})
        elif event["type"] == "invoice.validated" and event["ok"]:
            saga.state = "validated"
            await self.bus.publish("invoice.approval.requested", {"invoice_id": key})  # → human/agent approval
        elif event["type"] == "invoice.approved":
            saga.state = "approved"
            await self.bus.publish("ledger.post.requested", {"invoice_id": key})
        elif event["type"] == "ledger.posted":
            saga.state = "posted"
            self.store.complete(key)
        elif event["type"] in ("invoice.validation.failed", "invoice.rejected"):
            await self._compensate(saga)  # e.g., notify billing + reopen case

    async def _compensate(self, saga):
        saga.state = "compensated"
        await self.bus.publish("invoice.compensated", {"invoice_id": saga.id})
        # compensation for each already-committed step lives HERE,
        # as a reversed sequence of compensating actions.
```

**The idempotent consumer side:**

```python
# ledger_consumer.py — consumes "ledger.post.requested"
async def handle_post_request(event, db):
    key = event["invoice_id"]
    if await db.dedupe_exists(key):        # 1) idempotency guard
        return
    try:
        await db.post_ledger_entry(event)  # 2) apply exactly once
    except RetryableError:
        await bus.retry_later(event, backoff=exponential_with_jitter(base=1_000, jitter=500))
    except PoisonError:
        await bus.dead_letter(event)       # 3) park in DLQ, alert human
    await db.mark_dedupe(key)
```

### 5. Practical Application — to the Company Task System

**The "solid back-end" defined operationally.** For your idea, "vững chắc" (solid) should mean these *verifiable properties*:

| Property | Implementation | How you verify it |
|----------|----------------|-------------------|
| Durability | Broker persists events (Kafka retention/compaction) | Test: restart broker, events survive |
| No silent loss | At-least-once + consumer offsets | Test: kill consumer mid-batch → no lost events |
| Exactly-once *effect* | Idempotency keys + dedupe store | Test: replay same event → same state |
| Workflow recovery | Orchestrated saga state machine | Test: crash coordinator → resume from last state |
| No infinite blocking | DLQ + alerts | Test: poison message lands in DLQ, alert fires |
| Evolvable contracts | Schema registry + SemVer + dual-write | Test: v2 event consumed by v1 consumers during window |

**Example task graph from the parent idea** (Quote → Invoice → Payment → Ledger → Report) now maps naturally:

```
[Quote task] ──quote.accepted──▶ [Invoice task] ──invoice.approved (human gate)──▶
[Payment task] ──payment.captured──▶ [Ledger task] ──ledger.posted──▶ [Report task]
```

Each link is an **event**; every component stays independent (deploys/evolves alone); the **orchestrated saga** makes the chain's health visible to the operations desk; the **event log** doubles as the audit trail an agent-operated company needs.

### 6. Comparison Table

| Dimension | Choreography | Orchestration (saga) | Sync RPC chain |
|-----------|--------------|----------------------|----------------|
| Coupling | Lowest | Medium (one coordinator) | Highest |
| Business-flow visibility | Poor (implicit) | Excellent (explicit) | Good (but single request) |
| Failure handling | Ad-hoc compensation | Structured compensation | Timeouts/retries only |
| Recovery / pause-resume | Manual | Native (state machine) | Manual |
| Scale ceiling | Excellent | Good (coordinator must scale) | Poor (request depth) |
| Best for | Stable flows, mature teams | Business-critical, multi-step, human gates | Trivial 2-hop chains |
| **Fit for your idea** | ⚠️ After Maturity | ✅ **Recommended start** | ❌ Distributed monolith trap |

---

## 📝 Research Journey

- **Why:** The parent idea says tasks must "liên kết với nhau thông qua hệ thống back-end vững chắc" — this note exists to answer *what "vững chắc" mechanically means* and how independent components can still form coherent business flows.
- **Struggle:** I kept conflating events and commands, and assuming that "more decoupling = better". Reality: choreography decouples the *code* but *hides the business process* — for a company system, hidden processes are unacceptable.
- **Aha moments:** (1) Idempotency is not a nice-to-have — "at-least-once" *forces* it; (2) the orchestrated saga is not a violation of events — you can (and should) have both: orchestrator for control flow, events for audit and analytics; (3) schema evolution discipline is literally how "easy to modify long-term" becomes true in a distributed system.
- **Career link:** Event-driven systems are the backbone of every serious fintech/logistics/business platform — the Economics/Business systems I want to engineer. Being fluent here is the difference between "integration layer" and "distributed monolith".

## 🔗 Related Topics

- [01-project-idea-and-evaluation.md](01-project-idea-and-evaluation.md) — the project idea this note implements (Insight 2: the "solid back-end" *is* the idea).
- [03-modular-monolith-vs-microservices.md](03-modular-monolith-vs-microservices.md) — how to keep the components independent *while* they exchange events (contract + boundary discipline).
- [04-mcp-and-function-calling-agent-interfaces.md](04-mcp-and-function-calling-agent-interfaces.md) — how agents consume these task capabilities (protocol layer above the same backend).
- *(external, Researching Diary)* `ai_ml/05-mlops-lifecycle-and-deployment-architecture.md` — long-term operability of the AI parts inside the task chain.
- *(external, Researching Diary)* `git_github/11-infrastructure-as-code-and-devops-automation.md` — brokers and sagas must be deployed reproducibly (IaC).
- [SUGGESTED] **Bridge note: "From BPMN/process mining to saga design"** — a bank's workflows (payment approval, AML checks) already exist as processes; mining real logs to derive saga steps grounds the whole system in reality. *Why important:* discover the bank's processes — don't invent them.

## 🤔 Open Questions

- [ ] Kafka vs RabbitMQ vs Redis Streams for the *first* company integration backbone — what's the ops cost at single-company scale?
- [ ] Event sourcing for the ledger task: is the audit value worth the complexity in v1?
- [ ] How does the human (or agent) approval step model a *pause of days* inside a saga — timeout policies, reminders, escalations?
- [ ] What is the DLQ alerting + repair workflow when an agent is the operator?

---
*Events connect what must stay connected, without owning what must stay independent.*