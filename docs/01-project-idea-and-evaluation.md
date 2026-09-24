# Idea — Modular Company Task System, Operable by AI Agents

**Created Date:** 2026-09-23
**Last Updated:** 2026-09-23
**Tags:** `#ProjectIdea`, `#SystemArchitecture`, `#Modularity`, `#AIAgents`, `#CLIFirst`, `#EventDriven`, `#DomainSpecific`
**Status:** Draft — under evaluation
**Type:** Idea capture + critical evaluation (structure & reasoning quality)
**Project:** Modular Bank Operations System — anchor domain: **banking operations** (Clarification Gap #1 resolved: the "company with specific expertise" is a bank)

---

## 💡 Original Idea (Verbatim)

> *"Tôi muốn làm 1 hệ thống lớn chuyên biệt cho các tác vụ của 1 công ty có chuyên môn cụ thể, các tác vụ ấy có khả năng liên kết với nhau thông qua hệ thống back-end vững chắc, tuy nhiên các thành phần, nhân tố trong đó phải có khả năng độc lập cao, có khả năng thích ứng với thời cuộc, dễ dàng sửa đổi và phát triển dài lâu. Đồng thời có khả năng tích hợp AI và có hệ thống CLI để thao tác tự do bằng AI Agent."*

*(Original Vietnamese wording kept as a marked quote, per Language Policy A6.)*

**English rendering:** A large system specialized for the tasks of a company with specific domain expertise; those tasks can interlink through a solid back-end system; however, every component inside must have high independence, adapt to changing times, be easy to modify, and support long-term development. At the same time, it must be able to integrate AI, and it must have a CLI system through which AI Agents can operate freely.

---

## 📌 Key Takeaways (TL;DR)

- The idea describes a **domain-specific, agent-operable business platform**: vertical (specialized), modular (independent components), evolvable (long-term), and AI-native (CLI + agents).
- The three strongest instincts here are: **(1) domain specialization as the moat**, **(2) component independence as a design goal**, **(3) agents as first-class system operators** — these are exactly the three differentiators of modern platforms.
- The core tension to resolve is **independence vs. integration**: components that must stay independent *and* link together need an explicit contract layer (events/APIs/schemas). This back-end "linking" decision is where the design will live or die.
- Biggest gaps: no concrete first domain identified; "agents operate freely" has no governance story; **CLI alone is not an agent interface** (agents also need structured protocols like JSON schemas / MCP / OpenAPI).
- Verdict: a **well-directioned, strategically sound idea** with a clear-thinking architecture instinct, weakened only by abstraction (no anchor domain yet) and one unsafe assumption (ungoverned agent freedom).
- **Follow-up evaluation added (2026-09-23):** +5 insights (Conway's Law, data evolution, per-operation AI economics, data flywheel polarity, option-buying), a 12-point list of **clarification gaps** the idea must resolve, and **8 risks to acknowledge** before committing — see the expanded sections below.

---

## 🧠 Idea Deconstruction (6 Atomic Requirements)

| # | Requirement (user's words) | Engineering Interpretation | Architectural Axis |
|---|----------------------------|----------------------------|--------------------|
| 1 | "Large system specialized for a company with specific expertise" | A **vertical domain platform** built around one profession's workflows (not a generic tool) | Scope & domain |
| 2 | "Tasks interlink through a solid back-end" | An **integration backbone**: reliable messaging, data contracts, workflow choreography/orchestration | Integration |
| 3 | "Components have high independence" | **Modularity / loose coupling**: independent deployability, bounded contexts, failure isolation | Coupling |
| 4 | "Adaptable to the times, easy to modify, long-term" | **Evolutionary architecture**: refactoring discipline, versioning, backward compatibility, ADRs | Evolvability |
| 5 | "AI integration capability" | **AI as an embedded capability** inside tasks (models, agents, copilots) | Intelligence |
| 6 | "CLI so AI Agents can operate freely" | **Agent-native interface**: every capability exposed as a callable tool with structured I/O | Interface |

The idea is unusually *complete* for a first draft — it covers **scope, integration, coupling, evolvability, intelligence, and interface**, which is essentially six of the classic architecture driver categories. That is a sign of systems thinking.

---

## 💡 Insights & Clarifications (Making the Idea Sharper)

### Insight 1 — Decide: is this a *product* or an *internal platform*?
"One company with specific expertise" is ambiguous: is it **one specific company** (internal tooling) or **companies of one profession** (vertical SaaS product)?
- **Internal platform:** deep integration with existing processes, no multi-tenancy, deployment = their infra.
- **Vertical product:** needs multi-tenancy, configurable workflow engine, onboarding, billing, support — a fundamentally larger system.
*Why it matters:* this single decision changes the architecture (single-tenant vs multi-tenant), the roadmap, and the business model. Resolve it first.

### Insight 2 — The "solid back-end" *is* the idea
Requirement #2 is the load-bearing wall. "Tasks interlink" implies an **event-driven backbone** (e.g., a durable message bus + saga/choreography), not just a pile of REST calls:
- Task A finishes → emits event → Task B triggers, with retries, idempotency keys, dead-letter queues, and an audit trail.
- "Solid" should be defined as measurable properties: **durability, ordering guarantees, at-least-once/at-most-once delivery, observability**.
*Why it matters:* if this layer is hand-waved, "independent components" silently become a distributed ball of mud.

### Insight 3 — Independence ≠ isolation; independence needs *contracts*
High independence only survives if governed by **stable interfaces, volatile internals**:
- Each component owns its data (database-per-context), communicates only through versioned APIs/events (schema registry, contract tests).
- Classic trade-off: **modular monolith vs microservices**. Start modular monolith (one deployable, hard module boundaries enforced by lint rules/archunit), split only when a fitness function proves a need. This matches the evolutionary-architecture thinking already in the diary (HAL, refactoring, scaling — see Related Topics).
*Why it matters:* without contract discipline, "independence" degenerates into "each team invents its own protocol" → integration chaos.

### Insight 4 — "Adaptable / easy to modify / long-term" is a *discipline*, not a feature
You cannot design adaptability in once; you **earn** it continuously:
- CI/CD with fast test suites, ADRs (Architecture Decision Records) so the *why* survives staff turnover, semantic versioning + deprecation windows, feature flags for safe rollout, strangler pattern for replacing legacy parts.
- Fitness functions: automated checks (dependency rules, latency budgets, security scans) that *guard* the architecture as it evolves.
*Why it matters:* this requirement is the one most ideas forget, and it's the one that decides whether the system still exists in 5 years.

### Insight 5 — "AI integration" actually means *three* different things
Disambiguate before building:
1. **AI inside tasks** — models embedded in a workflow (e.g., document extraction in an approval step).
2. **AI as orchestrator** — an agent plans/executes multi-step tasks (agent = the workflow engine's client).
3. **AI as operator via CLI** — agents drive the system like a power user (the stated requirement).
Each has different reliability needs. Note that AI is **non-deterministic** — business systems need guardrails: human-in-the-loop approval, audit logs, deterministic fallbacks, and permission scoping.

### Insight 6 — "CLI for free agent operation" is 50% right — the missing half is the *protocol*
A CLI is an excellent agent interface (text in/out, composable, scriptable, JSON flags), but modern agent operation needs more:
- **Structured output:** every command supports `--json` so agents parse results, not prose.
- **Self-describing commands:** `help --json` / machine-readable schemas (think Cobra/Click/Typer + JSON schema) — agents discover capabilities at runtime.
- **A protocol layer:** MCP (Model Context Protocol) or OpenAPI tool schemas let agents call capabilities *programmatically*, without shell round-trips. The CLI is the human/agent shell; the protocol is the contract.
- **Governance:** "tự do" (free) must be bounded — least-privilege tokens, sandboxing, approval flows for destructive ops, full audit trail. An ungoverned agent with a company-system CLI is a liability, not a feature.
*Bonus insight:* **headless core + many thin clients** — one engine, exposed via CLI (agents/scripts), Web UI (business users), API (integrations). CLI-only UX would restrict adoption to technical users.

### Insight 7 — Conway's Law is your co-designer (structure mirrors the team)
The system's component boundaries will mirror the structure of whoever builds it. Built solo (or by one small team), "independent components" have **no external consumer pressure** to stay decoupled — they quietly degrade into folders with casual imports. Independence is only *real* when there are genuinely independent teams or release trains (three modules that can deploy and evolve on their own). If you build alone, be honest about your realistic independence unit: **module boundary + enforced rules** (import-linting, contract tests) — not physical services. And use this consciously: introduce real independence gradually as actual consumers appear (strangler pattern), instead of pre-building autonomy nobody needs yet.

### Insight 8 — "Solid back-end" is mostly a *data evolution* problem
Linking tasks means data/events cross component boundaries — so **every schema change is a potential breaking change** for downstream consumers. A long-lived linked system therefore needs, from day one: a schema registry, semantic versioning of contracts, and a deprecation policy ("old version supported for N months"). "Solid" is not a property you install; it is a *versioning discipline* you apply for years. Miss this, and the back-end freezes — nobody dares touch schemas, and the "adaptive, easy-to-modify" requirement (4) quietly dies.

### Insight 9 — AI integration turns one-time costs into *per-operation* costs
An agent-operated business system pays a **variable cost per action** (tokens, model calls, latency). Business systems run high volumes. You must model **unit economics per task**: cost/task, latency budget, error-rate budget, fallback cost. A beautifully modular, agent-operable system that costs $0.50/task on a $0.05-margin operation is economically dead. Also, model providers change pricing/models/availability unpredictably — abstract the AI layer at the *interface* (provider-neutral tool schemas + thin adapters), not by wrapping every obscure feature.

### Insight 10 — The AI-vs-expert flywheel can run in reverse
The edge of this idea = domain knowledge + AI. But if AI gradually replaces the experts' *doing*, the stream of new real-world/vetted data dries up, and the AI drifts from reality (concept drift). The workflow must keep domain experts as **validators** (human-in-the-loop) — because that validation signal *is* tomorrow's training/eval data. Design the human check into the workflow, not as an afterthought. Remember also: the domain knowledge lives in experts' heads; the system is hostage to whoever codified it first — plan continuous domain discovery (interviews, shadowing, log mining → new task designs).

### Insight 11 — "Adapting to the times" is option-buying, not forecasting
You cannot predict the future; the architecture should buy **options (reversibility)** instead: feature flags, strangler patterns, ADRs, and a deliberately thin core contract. Every irreversible commitment (schema, topology, tech stack) is a bet — count your bets and limit them. This is exactly the "evolutionary architecture + fitness functions" thinking already in the diary's agile/architecture notes.

---

## ❓ Clarification Gaps — Points Still Unclear (Điểm chưa được làm rõ)

The idea statement leaves these unresolved. They are *design blockers*, not style questions — each one changes the architecture:

1. **Anchoring identity:** ONE specific company vs an entire profession? (Determines multi-tenancy, per-client customization, licensing, effort.)
2. **Task granularity:** Atomic action (single command) vs long-running process (a case spanning days)? Orchestration differs hugely (simple calls vs sagas with compensation).
3. **"Liên kết" semantics:** Data flow (output of A feeds B), control flow (A triggers B), or both? And *what happens when B fails after A already committed*? (compensation/rollback behavior)
4. **AI scope:** Generative only, or also classical ML (routing, forecasting, classification)? Which tasks tolerate AI error (recommendations) vs which do not (financial posting, legal output)?
5. **CLI operator model:** Humans only, agents only, or both with different permissions? Is there a read-only agent role vs an execute role? Are all agent actions logged and approvable?
6. **Deployment & data residence:** Cloud / on-prem / hybrid? Public cloud may be off-limits for some company data (finance, healthcare, confidential business data; Vietnamese Decree 13/2023 on personal data protection).
7. **Data sensitivity class:** PII, financial, health, trade secrets? → determines compliance scope (data protection rules, sector regulations, possibly the EU AI Act if ever exposed to EU users).
8. **Existing tooling:** What do target users run today (Excel, ERP, CRM, email)? Does the system replace, complement, or import/export from them? — Integration adapters are usually the hidden 50–80% of project time.
9. **v1 success metric:** What measurable outcome proves the idea? (e.g., task time −60%, error rate −40%, cost/task below X) Without a number, "large system" stays unverifiable.
10. **Ownership of "specialization":** Is the domain expertise yours, a partner's, or to be learned? The moat is ~90% domain understanding, ~10% code.
11. **Business model (if product):** per-seat, per-task, license, outcome-based?
12. **Human role after automation:** Which steps *must* stay human, and how is that enforced? (liability, trust, regulation)

---

## ✅ Strengths (Điểm tốt)

1. **Domain specialization is a real moat.** Generic horizontal platforms are crowded and commoditized; a system tuned to one profession's workflows has deep switching costs, obvious ROI, and is hard to displace. Vertical beats horizontal for a small builder.
2. **Component independence is the right default.** It buys failure isolation, independent deployability, parallel team work, and technology freedom per component — directly countering the "big ball of mud" that kills long-lived internal systems.
3. **Integration-first ("solid back-end") is mature instinct.** Many solo ideas jump to UI first; this one correctly puts the connective tissue (data flow, task linkage) at the center — the part that determines whether the system scales beyond a demo.
4. **Agent-native CLI is ahead of the curve.** Designing every capability to be machine-operable (structured I/O, tool schemas) aligns with the 2025–2026 agentic-workflow shift (MCP, function calling). Systems designed *for agents* will be strictly more automatable than those retrofitted.
5. **Long-term evolvability is prioritized.** Explicitly valuing "adapt to the times, easy to modify, long-term" means the design starts from maintainability rather than treating it as an afterthought — the difference between a 5-year asset and a 6-month rewrite.
6. **Highly buildable with your current skill set.** Backend + AI/ML + CLI/TUI experience (see the Custom Multi-TUI project) + DevOps notes (IaC, MLOps) cover essentially every layer this idea needs. It is an ambitious but *reachable* project, not vaporware.

---

## ⚠️ Weaknesses & Risks (Điểm xấu)

1. **Scope explosion / the "large system" trap.** "Large + specialized + AI + CLI + long-term" with no concrete first domain is the classic v1 trap: everything is possible, nothing ships. Without a bounded starting slice, the project balloons or stalls.
   *Mitigation:* pick ONE real company domain + 3 concrete tasks for v0; grow via strangler pattern.
2. **The independence-vs-integration contradiction is unresolved.** Components that are "highly independent" do not magically "link through a solid back-end" — linking *is* coupling. If the contract layer (events, schemas, versioning) is not designed, you get one of two failure modes: distributed chaos (independent but unlinkable) or a hidden monolith (linkable but not independent).
   *Mitigation:* design the contract layer first: event catalog, schema versioning policy, contract tests.
3. **No defined user, buyer, or business model.** The idea describes *architecture*, not *value*: who uses it daily, who pays, what pain does it remove? A company's real workflows are messy; without an anchor domain and a real user, every design decision stays abstract and unverifiable.
   *Mitigation:* write a one-page problem statement per task: user, pain, current workaround, measurable win.
4. **"Agents operate freely" is an unsafe assumption for a business system.** Free agent operation over company tasks = autonomous writes to business data, destructive commands, prompt-injection surface. Without permissions, sandboxing, approval gates, and audit logs, one bad agent run can corrupt operations or leak data.
   *Mitigation:** least-privilege agent roles, dry-run mode by default, human approval for irreversible ops, immutable audit trail.
5. **CLI-only UX caps adoption.** Business staff (accountants, operators, managers) will not live in a terminal. CLI-only means the system serves developers and agents, excluding the humans who validate the domain value — narrowing the market to tech-savvy firms.
   *Mitigation:* headless core; CLI as power-user/agent surface, Web UI as thin client over the same API.
6. **"Long-term" raises the maintenance floor.** A modular, evolvable, observable multi-component system costs more to operate than a simple app: cross-module monitoring, version drift management, dependency upgrades, contract test upkeep. Without CI/CD and testing discipline from day one, the adaptability goal quietly dies under technical debt.
   *Mitigation:* automate fitness functions early (dependency-rule checks, smoke tests, IaC for reproducible environments).
7. **AI reliability risk in business-critical paths.** Hallucinations, non-determinism, and model drift inside linked tasks can cascade (task B trusts task A's AI output). AI needs its own pipeline: evaluation, guardrails, fallback rules (see the MLOps lifecycle note).

---

## 🚨 Risks to Acknowledge Before Committing (Rủi ro phải nhận)

These are risks you *inherit by the nature of the idea itself* — acknowledge them now, or they surface later as surprises:

1. **Bus-factor = 1 (single-builder syndrome).** A "large, long-term" system built mostly by one person concentrates all knowledge and energy in one head. If your priorities shift (graduation, first job, changing interests), the system has no one to carry it. *Acceptable only if* you treat "survivable without me for 6 months" as a real property: ADRs, module maps, small honest scope.
2. **Prompt injection at business scale.** Agents that process third-party content (invoices, emails, partner documents) open an *indirect* prompt-injection surface: malicious instructions hidden inside business documents can make the agent exfiltrate data, approve payments, or delete records. This is the #1 AI-security risk for agentic business systems — not hypothetical. Mitigations: content/instruction separation, tool-level allowlists, least privilege, human approval on consequential actions.
3. **Compliance & liability.** AI output driving real business decisions creates liability (wrong financial action, bad legal output, data leak). As a *product*, you inherit regulatory exposure (data protection, sector rules, possibly the EU AI Act). *Accept only with* a defined compliance scope, disclaimers, and an immutable audit trail.
4. **Garbage-in-garbage-out amplification.** A linked system propagates errors automatically: bad data at task A contaminates B and C — that is literally what "linked" means. Data quality is a company-culture problem you cannot fully fix in code, yet your system will be blamed for data it didn't create. Mitigations: provenance tracking + validation gates at every boundary.
5. **Vendor lock-in in the AI layer.** Providers change pricing/models/APIs under you. The "adaptable" part of your system must include a thin provider-neutral interface, or you accept recurring rework.
6. **Technology decay.** Whatever you build on today will age; "easy to modify long-term" is a promise to future-you who will pay in refactoring. A long-lived system is a *subscription of effort*, not a one-off build — maintenance compounds every year.
7. **Opportunity cost for a student.** This is a multi-year commitment. Each month on it is a month not spent on deeper ML courses, internships, or competing portfolio projects. *Accept it consciously*, or shrink v1 to one module + one customer.
8. **Premature generic-ization.** Designing "independence/adaptability" *before* a concrete set of tasks usually produces wrong abstractions — and fixing wrong abstractions is the most expensive kind of rework. *Rule of three:* build concrete first; abstract only when a third real need appears.

### Risk register summary

| # | Risk | Inherent severity | Mitigable now? | Acceptable as-is? |
|---|------|-------------------|----------------|-------------------|
| 1 | Bus-factor = 1 | High | Partial (docs, small scope) | Conditional |
| 2 | Prompt injection | Critical | Yes (design-time) | No — must be designed for |
| 3 | Compliance & liability | High (if product) | Partial (legal review) | Conditional |
| 4 | GIGO amplification | Medium | Partial (validation gates) | Conditional |
| 5 | AI vendor lock-in | Medium | Yes (thin interface) | Yes — with thin adapter |
| 6 | Technology decay | Medium | Partial (ADRs, options) | Yes — budget refactoring |
| 7 | Opportunity cost | Personal | Yes (resize v1) | You decide |
| 8 | Premature abstraction | High | Yes (rule of three) | No — must be disciplined |

---

## 🏗️ Structural Critique (Đánh giá về KẾT CẤU)

**What the structure gets right:**
- Clear layering instinct: **system → tasks → components → interface (CLI) → intelligence (AI)**. The idea separates *what the system does* (tasks) from *how it's built* (independent components) from *how it's operated* (CLI/agents) — a healthy three-axis decomposition.
- Non-functional requirements are named explicitly (independence, adaptability, modifiability, longevity). Most first-draft ideas only state functional goals; naming the *qualities* is how real architecture starts.

**Structural problems:**
1. **Priority order is missing.** Independence, adaptability, modifiability, and longevity *conflict* under pressure (e.g., ship-fast vs. clean boundaries). Architecture = making trade-offs explicit; without a ranked driver list, every future decision defaults to "whatever is fastest."
2. **The data layer is absent from the structure.** Where does state live — shared database, database-per-component, event store? Component independence is mostly a *data ownership* question; leaving it out is the single biggest structural hole.
3. **Security & identity are absent.** No mention of personas, roles, or permission boundaries — yet "agents operating freely" makes authorization the most load-bearing missing layer.
4. **Observability is absent.** A distributed, linked, long-lived system without tracing/logging/metrics cannot be debugged or evolved; "solid back-end" without observability is unverifiable.
5. **CLI sits at the wrong abstraction level.** The requirement mixes a *user interface* (CLI) with an *integration contract* (how agents call capabilities). Structurally, the clean shape is: `Core engine → API/protocol layer (MCP/OpenAPI) → {CLI, Web UI, Agent adapters}`. The CLI should be a *client of the protocol*, not the protocol itself.

**Corrected structural sketch:**
```
                    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   Thin clients →   │  CLI (human/ │  │   Web UI     │  │ Agent adapter│
                    │  agent)      │  │ (business)   │  │ (MCP/tools)  │
                    └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
                           └─────────────────┼─────────────────┘
                                             ▼
                          ┌────────────────────────────────────┐
                          │  API + Protocol layer (contract,   │
                          │  auth, versioning, audit)          │
                          └────────────────┬───────────────────┘
                                             ▼
   Independent components (bounded contexts, DB-per-context)
   [Task A] ──event──▶ [Task B] ──event──▶ [Task C]   ← integration backbone
                                             ▼
                          ┌────────────────────────────────────┐
                          │  Cross-cutting: observability,     │
                          │  AI services, identity/permissions │
                          └────────────────────────────────────┘
```

---

## 🧠 Reasoning Critique (Đánh giá về KHẢ NĂNG TƯ DUY)

**Where the thinking is strong:**
1. **Systems-level framing.** The idea reasons about a *whole system* (components, linkage, interface, evolution) rather than a single feature — genuine architectural instinct, not feature-listing.
2. **Non-functional awareness.** Valuing independence, adaptability, and longevity shows awareness of software *qualities*, which is the hallmark of mature engineering thinking (most beginners optimize only for "it works").
3. **Future-orientation.** Anchoring on agents + AI integration shows the idea is designed for where the industry is going (agent-operable systems), not where it has been.
4. **Consistency with real experience.** The multi-repo independence trade-off already chosen in the Custom TUI project (accepting code duplication for absolute independence) is the same philosophy at a larger scale — the reasoning generalizes correctly.

**Where the thinking breaks down:**
1. **Abstraction without an anchor.** The idea floats at the "large system" level with no concrete domain, tasks, or users named. Thinking that never touches a specific example cannot be tested — and untestable requirements ("adapt to the times") quietly become unachievable.
2. **A contradiction is left unexamined.** "Highly independent components" + "tasks interlinked through a solid back-end" + "one large system" pull in opposite directions. The idea *asserts* they coexist but never asks *how* — that missing "how" (the contract layer) is where 80% of the real work lives.
3. **"Free" is confused with "capable."** "Thao tác tự do" (operate freely) mistakes unconstrained access for agent capability. In reality, agents are *more* capable inside well-specified permission boundaries (they know what they may do) than in an open field (they must guess, and guessing = risk). This is a governance blind spot, not just a security one.
4. **Tool conflation:** CLI ≈ agent interface. The reasoning jumps from "agents need to operate it" to "therefore CLI," skipping the question of *what agents actually consume* (schemas, structured output, protocols). The CLI is a good *part* of the answer, not the answer.
5. **"Large" is treated as a virtue, not a cost.** Nothing in the idea weighs the carrying cost of largeness (ops burden, coordination overhead, slower refactors). Longevity thinking should first ask *"what is the smallest system that still delivers the value?"*

**Net assessment of reasoning:** directionally excellent (right concerns, right future), but currently **assertive rather than analytic** — it states desired properties without resolving their tensions. Converting each "I want X" into "X is achieved by Y, verified by Z" would lift this from a vision to an architecture.

---

## 🎯 Decisions to Make Before Writing Any Code

- [ ] Pick ONE anchor domain (specific company/profession) + 3 concrete tasks for v0.
- [ ] Product vs internal platform? (multi-tenant or not)
- [ ] Draw the task graph: which tasks must link, and what data crosses boundaries.
- [ ] Choose the integration backbone: event bus / queue / REST orchestration — and define "solid" (delivery guarantees, ordering, retries).
- [ ] Choose data ownership model: database-per-component vs shared store.
- [ ] AI mode priority: embedded AI vs orchestrator vs operator (Insight 5).
- [ ] Agent protocol: CLI with `--json` + MCP server? Define permission roles & approval gates.
- [ ] Ranking of quality drivers (independence vs speed vs cost) — the trade-off order.
- [ ] Start as modular monolith? Set the fitness functions that trigger a split later.
- [ ] Contract versioning policy: schema registry, semantic versioning, deprecation windows.
- [ ] Data classification & regulator scope: PII / financial / health? Which compliance rules apply (e.g., Decree 13/2023, sector rules)?
- [ ] Agent safety model: permission roles, read-only vs execute agent, approval gates, audit trail.
- [ ] AI-provider abstraction level (keep it thin) + unit economics: budget per task (tokens, latency, error rate, fallback cost).
- [ ] Human-in-the-loop design: which steps stay human-validated, and how validation feeds future training/eval data.

---

## 📝 Research Journey

- **Why:** I keep circling back to the same shape of project — a company-scale system where tasks interconnect but each part stays independent and AI can operate it. Capturing it properly (instead of letting it stay a vague "someday" idea) forces me to confront whether I actually understand how such systems are built, or only like how they sound.
- **Struggle:** The hardest part was the apparent contradiction between *independence* and *linkage*. For a long time I assumed "more independent = better," without realizing independence without contracts is just fragmentation — and linkage without discipline is just a monolith wearing a costume.
- **Aha moments:** (1) Independence is bought with **stable interfaces + data ownership**, not with physical separation. (2) A CLI is the *shell*, but agents really need a **protocol** (schemas/MCP) — realizing this reframed "CLI for agents" into "agent-native platform, CLI as one face of it." (3) "Adaptability" is a *discipline* (tests, CI, ADRs), not a property you can finish.
- **Career link:** This is precisely the AI Engineer shape of product I want to build in Economics/Business: a vertical business platform whose workflows are AI-augmented and whose every capability is agent-callable. Designing governed, observable, agent-operable systems is the exact intersection of my backend, AI, and DevOps notes — and the differentiator I want on my CV/portfolio.

---

## 🔗 Related Topics

**In this repo:**
- [02-event-driven-architecture-and-task-orchestration.md](02-event-driven-architecture-and-task-orchestration.md) — implements requirement #2 ("tasks interlink through a solid back-end").
- [03-modular-monolith-vs-microservices.md](03-modular-monolith-vs-microservices.md) — resolves requirement #3 (component independence vs integration).
- [04-mcp-and-function-calling-agent-interfaces.md](04-mcp-and-function-calling-agent-interfaces.md) — implements requirement #6 (CLI → governed agent protocol).

**Source of truth (Researching Diary, external):**
- `random_ideas/04-custom-tui-project.md` — the agent-interface instinct one level down (TUI/CLI for AGY & Opencode).
- `random_ideas/05-projects-review-blindsight-snakeann-joblink.md` — Joblink's feature-based modular structure + layered security as concrete precedent.
- `iot_aiot/03-hardware-abstraction-layer-hal.md` — independence through abstraction at hardware level.
- `ai_ml/05-mlops-lifecycle-and-deployment-architecture.md` — long-term operability of the AI parts of the task chain.
- `git_github/11-infrastructure-as-code-and-devops-automation.md` — reproducible environments for a long-lived system.
- `books_summaries/agile_project_management/notes/07-agile-architecture-hal-refactoring-scaling.md` — evolutionary architecture & scaling frameworks.

**[SUGGESTED] follow-ups:** event-driven backbone (→ doc 02) and bounded-context discovery for banking — KYC / payments / ledger / AML-approval before writing modules (→ doc 03).

---

## 🤔 Open Questions

- [ ] Which specific company domain anchors v0 — and what are its 3 most automatable tasks?
- [ ] Product (vertical SaaS) or internal platform? How does that change the data model?
- [ ] What delivery guarantees make the back-end "solid" enough for this domain (ordering? latency? at-least-once?)?
- [ ] How do agent permissions map to component boundaries (role = set of allowed tools)?
- [ ] What is the fallback when the AI component is wrong — and how does the audit log capture it?
- [ ] Which quality driver wins when they conflict: independence, speed of delivery, or cost?
- [ ] What are the unit economics per task (tokens, latency, cost/task) — is the margin still positive at real business volume?
- [ ] When the AI is wrong on a consequential action, who is accountable — and what exactly does the audit trail prove?

---
*Idea quality is measured by the tension it resolves, not the features it lists.*
