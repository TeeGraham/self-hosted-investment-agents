# Self-Hosted Multi-Agent Investment Research Platform — Capability Summary

> **Purpose of this document.** A sanitized, portfolio-facing overview of a system I designed, built, and operate. It describes *what the system does and how it is engineered* without exposing infrastructure addresses, credentials, proprietary scoring methodology, or client data. Intended for sharing with prospective employers or collaborators.

---

## Overview

A self-hosted crew of cooperating AI agents that ingests investment source material, performs structured fundamental and technical analysis, answers research questions over a private knowledge base, and generates versioned, distributable research reports — orchestrated as modular workflows on a single owned server with no dependency on a managed AI platform.

**The goal:** approximate, at personal scale, the core research-and-analytics workflow that institutional professionals get from platforms like **Bloomberg Terminal** or **BlackRock's Aladdin** — consolidated market and fundamentals data, structured scoring, a research knowledge base, and generated reports — but self-built and self-hosted for **individual use at a small fraction of those platforms' licensing cost** (a Bloomberg Terminal seat runs on the order of tens of thousands of dollars per year; Aladdin is an enterprise contract). This is not a claim of feature parity with those systems — it is a deliberately scoped, personal analog of the parts that matter most to an individual analyst, running on commodity hardware.

---

## Highlights

> **Ran a crew of cooperating LLM agents *and* a full nightly financial-data pipeline on a 2-core, 8 GB, GPU-less server — and engineered it to degrade gracefully instead of falling over.**

A few numbers that tend to prompt the follow-up questions:

- **A personal-scale analog of a Bloomberg/Aladdin-style research desk** — consolidated data, scoring, a research knowledge base, and generated reports — for a small fraction of institutional licensing cost.
- **~120 modular workflows, ~60 live in production** — every capability is an independently testable agent, not a monolith. Grown roughly 3x since the platform's first public snapshot, without a rebuild.
- **Runs largely unattended at real production scale** — nightly data pipelines, independently scheduled "vertical" products, and self-healing recovery (auto-restart, memory watchdog, post-failure health check) process a full multi-asset watchlist every night with no human in the loop.
- **Zero-GPU LLM inference** — local models run entirely on CPU, with heavier reasoning selectively offloaded to a hosted model as an explicit, documented cost/quality decision.
- **Survived a real out-of-memory production crash and proved the fix live** — root-caused a genuine memory-exhaustion incident on a real overnight run, shipped a remediation (raised the process memory ceiling, tightened task concurrency, added automated failure alerting, wired an idle health-check into a real recovery loop), then proved it clean on a subsequent unattended production run with zero restarts.
- **Closes the loop on its own forecasts** — generated predictions are later checked against what actually happened and against an independent, market-based accuracy signal, and the results feed back into how the system reports going forward, rather than only ever producing new output.
- **A claim is only published if it's backed by real evidence.** If the model can't cite genuine support for something it wants to say, the system runs its own targeted search and independently checks whether what it found actually supports that *specific* claim — and drops the claim entirely rather than publish it unsupported if nothing checks out.
- **Every tracked asset gets a full scheduled refresh at least once a night, and can be refreshed on demand.** An operator can trigger an out-of-schedule update for a single asset or the whole watchlist through the same entry point the nightly run uses — useful the moment something newsworthy breaks instead of waiting for the next cycle.
- **Catalysts are tracked as ongoing threads, not one-off mentions.** A developing story — a piece of pending legislation, a corporate partnership, a macro event — is followed as a single evolving thread across nights instead of being re-reported as a disconnected new item every time coverage updates, and it's automatically resolved once the real-world outcome is confirmed.
- **Has run a full production watchlist of up to ~50 assets across equities, ETFs, and crypto in a single nightly cycle** — watchlist size is an operator setting, not a hard architectural ceiling.
- **Survived a real cascade failure** (a component that pinned every core past 1000% CPU) and turned it into a permanent architectural rule, not a patch.
- **Built in verified layers** — core capability layers plus multiple end-to-end "stability gates," each proven with a real production run before the next began.
- **Everything version-controlled and audited against reality** — living docs are periodically re-checked against the running system, and any drift — including in the data the system persists, not just its documentation — is treated as a defect and fixed, not just logged.

*(Each bullet expands into a full section below.)*

---

## System Architecture

- **Orchestration-first, multi-agent design.** Each capability (document ingestion, retrieval-augmented Q&A, fundamental scoring, report authoring, distribution, scheduled data collection) is an independent, individually testable workflow. A lightweight router classifies each incoming request and dispatches it to the right sub-agent, so agents compose without being coupled.
- **Two execution modes.** Scheduled pipelines run unattended on cron triggers (nightly market-data collection, weekly macro refresh, daily maintenance); on-demand agents are reached through a single routed entry point (chat / webhook / form).
- **Separation of stores.** A vector database holds embeddings and semantic-search material; a relational database holds structured state — agent memory, scoring definitions, report versions, cached financial data, and durable analyst notes. Each store is used for what it is good at, and neither is a dumping ground for the other.
- **Local-first model strategy with deliberate exceptions.** Cheap, fast routing and semantic synthesis run on small local models; heavier reasoning tasks (scoring, report drafting, summarization) are routed to a hosted mid-tier model where quality justifies it. Model tier is an explicit, documented decision per task, not an accident.
- **Retrieval-augmented generation over a curated corpus.** Ingested documents are chunked, embedded locally, and stored with metadata so that question-answering is grounded in the user's own source material rather than the model's training memory, with metadata filtering to scope retrieval.

---

## Core Capabilities

**Knowledge ingestion & retrieval**
- Ingests long-form source documents (filings, earnings material, research write-ups), chunks and embeds them, and makes them queryable.
- Extracts structured signals from sources that don't offer a clean structured feed, so coverage isn't limited to what a vendor happens to expose.
- Retrieval-augmented Q&A answers research questions with citations back to the underlying corpus.

**Structured analysis**
- Applies a consistent, user-defined multi-factor scoring rubric across assets so that rankings are repeatable rather than ad hoc.
- Computes a broad set of technical and risk indicators locally from raw price/volume data (trend, momentum, volatility, drawdown, and risk-adjusted-return measures) rather than paying for pre-computed feeds.
- Maintains a macro-regime overlay derived from public macroeconomic series, so single-asset analysis is read against the broader environment.

**Report generation & distribution**
- Drafts, edits, and *versions* research reports as new information arrives — reports are living documents, not one-shot outputs.
- Renders finished reports to portable formats (PDF/DOCX) through a containerized rendering service and distributes them on demand or on a schedule.

**Evidence verification & false-information guards**
- No claim is published on the model's say-so alone: every catalyst and forecast branch must cite specific supporting evidence, or the system runs an independent, targeted search and only attaches what it finds if a separate adjudication step confirms the result genuinely supports that *specific* claim — not just that it's topically related. If nothing checks out, the claim is dropped. "No evidence for this" is a valid, expected outcome, not a gap papered over with plausible-sounding filler.
- A second, independent validation pass runs after generation and checks the output against hard rules — required fields present, only valid category/outcome values used, internal narrative-vs-number consistency (a claimed "increase" can't be attached to a cited decrease), and no duplicate or unsupported claims slipping through.
- Source relevance is enforced per asset class, so a generic public data source is only cited when it's actually relevant to the asset being analyzed, rather than getting attached indiscriminately.
- Ingested market and fundamentals data is checked for plausibility on the way in — an implausible day-over-day jump in a reported figure is flagged and suppressed rather than silently propagating into every downstream calculation that depends on it.

<img width="1026" height="1560" alt="image" src="https://github.com/user-attachments/assets/8ae538bf-0644-40f0-bef2-68b3cbe1b65f" />

*A real dossier's full citation trail — every source either a primary SEC filing or a dated, named publication. Nothing is asserted without a link back to where it came from.*

**Catalyst tracking & cross-referencing**
- Tracks multiple distinct categories of market-moving events per asset — political/regulatory, macro-economic, pending legislation, and corporate/ecosystem developments (partnerships, competitive ties, M&A) — plus a separate global/macro layer for events that affect many assets at once rather than just one.
- A developing story is tracked as a single evolving thread carrying its own running confidence level and source history, so continuing coverage updates the existing thread instead of spawning a new, disconnected entry every time the story moves.
- Each catalyst carries an explicit duration classification (how long the underlying dynamic is expected to matter) and is eventually resolved — marked as having played out, stalled, or been disproven — rather than just aging out silently.
- When a catalyst names another tracked asset (a partnership, a supply-chain tie, a competitor), that reference links directly to the other asset's own report, so a reader can follow the connection instead of re-researching it themselves.

<img width="1047" height="828" alt="image" src="https://github.com/user-attachments/assets/7854a99b-dae4-4cc4-9fa8-cea5d37f5bee" />

*Real cross-asset ties, not a static "related tickers" widget — each connection carries its own relationship type (competitor dependency, strategic partner) and a cited reason, and clicking through opens that asset's own report.*

**Closed-loop forecast evaluation**
- Every generated forecast is later compared against what actually happened, rather than the system only ever producing new predictions with no feedback path.
- Cross-checks its own predictive claims against an independent, market-based signal (a public prediction-market feed) as an outside sanity check on forecast quality, alongside the system's own internal accuracy tracking.
- A structured event-resolution process determines whether a previously flagged catalyst played out, stalled, or was disproven, and folds that outcome back into the historical record that later reports and analysis draw on.

<img width="1021" height="1675" alt="image" src="https://github.com/user-attachments/assets/08b3a084-3df0-4d2a-a816-102492c2027b" />

*Every forecast is scenario-based and time-boxed, not a single point prediction — three horizons, three outcomes each, and every one of them traceable back to a specific cited source.*

<img width="918" height="1747" alt="image" src="https://github.com/user-attachments/assets/58c3fca5-7f8b-4b8e-8c8f-756ff2fb00f5" />

*A real revision, explained in plain language: this version's changes from the last one, why they happened, plotted against a benchmark, with the full version history underneath it.*

**Automation & delivery**
- Runs most of its work unattended on schedules, and packages finished output as automatically delivered "vertical" products. *(Covered in depth in the next section.)*

---

## Automation & Orchestration

Automation is not a feature bolted onto this system — it is the operating model. The platform is designed to do most of its work with no human in the loop, and to route the work that does need a human through a single, predictable entry point.

**Two classes of trigger.**

- **Scheduled (unattended).** Time-based pipelines run on their own schedules and need no interaction:
  - a **nightly market-and-fundamentals run** that walks the active watchlist one asset at a time — pulling price, fundamentals, and filing data, computing derived indicators locally, and writing everything to cache;
  - a **weekly macroeconomic refresh** that scores the prevailing market regime;
  - a **daily maintenance pass** that prunes transient content, enforces retention on the vector store, and compacts storage when needed.
- **Event-driven / on-demand (routed).** Chat messages, webhooks, and form submissions all arrive at a single orchestrator that classifies intent and dispatches to the correct agent — so supporting a new request type is a routing rule, not a new front door. The same entry point also lets an operator trigger a full, out-of-schedule refresh for one asset or the entire watchlist — the identical pipeline the nightly run uses, just invoked on demand rather than waiting for the clock.

<img width="1043" height="957" alt="image" src="https://github.com/user-attachments/assets/6e0c1e52-a8cc-47e1-a16d-6633a38d62ae" />

*A real version history — daily refreshes as the baseline, plus genuine same-day updates (two versions in one day here) proving the on-demand path is actually used, not just theoretically available.*

**Orchestrated, not merely scheduled.** The orchestrator does more than dispatch. A concurrency guard checks whether a heavy job is already running before starting another; when the system is busy, work is queued and the caller receives an estimated wait rather than an overload. Heavy tasks execute strictly sequentially — which is precisely what makes the system safe to leave running unattended on modest hardware.

**Automated "verticals" — the productization layer.** On top of the core engine sits a registry of independently scheduled jobs — digests, monitors, and data exports — each on its own cadence (daily, weekly, or event-driven). Each composes existing agents into a finished, delivered output with zero manual steps. This is the substrate for turning a personal research engine into repeatable products: a new offering is a newly registered job, not a rebuilt platform. Several are live today and deliver on their own schedules.

**Safe to re-run (idempotent by design).** The data pipeline is incremental and freshness-checked: re-running it does not duplicate work or waste API budget, because each step inspects what is already cached and fetches only what changed. Jobs that share a rate-limited credential are scheduled in non-overlapping windows so they never collide.

**Self-healing operation.** The system is built to recover on its own: containers restart automatically after a failure, a memory watchdog raises an alert under pressure, and a post-restart health check verifies database connectivity and data completeness before the system is trusted again. A single reusable notification component centralizes all operational alerts — errors, threshold breaches, and health-check results. The design goal is that a transient failure in the middle of the night resolves itself and leaves an audit trail, rather than waiting for someone to notice. **This was proven against a real incident, not just designed on paper**: a genuine out-of-memory crash on a live overnight run was root-caused, remediated (higher memory ceiling, tighter concurrency, automated error alerting, a health-check wired to an actual recovery action instead of just logging), and then verified clean across a subsequent full unattended production run with zero container restarts.

**Role-gated multi-user access.** The operator-facing dashboard supports authenticated accounts with standard/admin roles, so reporting surfaces can be shared with more than one person without giving everyone the same level of access. This replaced an earlier standalone, bring-your-own-backend demo page as the platform matured — the dashboard is now the one interface, not a public demo plus a separate real product.

---

## Engineering Practices

- **Resilience engineering under hard resource limits.** The system was originally designed to run reliably on a small, memory-constrained single server. That drove a layered mitigation stack — immediate model unloading, strictly sequential heavy-task execution, per-container memory limits, a guard that prevents concurrent heavy jobs from overlapping, and bounded context growth — all so the box degrades gracefully instead of failing under load. (The hardware has since been upgraded, but the discipline remains and the design still holds.)
- **Intellectual-property abstraction layer.** Where reasoning tasks touch a hosted model, the work is decomposed so that no single external call ever sees the proprietary methodology as a whole — sensitive synthesis is kept local, and external calls receive only abstracted, decomposed sub-tasks. Data sourcing follows the same principle.
- **Modular, gated build methodology.** The platform was built in progressive layers, each with an explicit stability "gate" that had to be verified end-to-end before the next layer began — a Modular Open Systems Architecture approach adapted to an AI-agent stack. Core capability layers plus multiple verification gates are all published and verified end-to-end.
- **Verification is first-class.** Every gate and vertical was proven with a real production run, not just a unit test — recurring classes of latent bugs (silent input-truncation, stale references, silent partial-batch processing that only touched the first item in a multi-item run, database type-coercion edge cases) were caught precisely because "done" meant "observed working in production," including a full data-integrity recovery effort that root-caused why a subset of forecast data had been silently failing to persist for weeks and rebuilt the pipeline to recover it.
- **Documentation-as-source-of-truth, kept honest.** The system carries a living technical report, an operations/maintenance plan, and a deferred-items register — and these are periodically audited *against the running system* rather than trusted blindly, with drift between docs and reality treated as a defect to be logged and corrected.
- **Everything version-controlled.** Workflow definitions, migrations, the rendering service, and all supporting documentation live in git; nothing important exists as a loose, unrecoverable file.
- **Defense in depth on data quality, not one gate.** No single check is trusted to catch everything: evidence-gating at generation time, a separate rules-based validation pass after generation, per-asset-class source-relevance filtering, and a recurring scheduled audit sweep that keeps re-checking already-published data all run independently — so a failure in one layer doesn't mean nothing catches the problem.
- **Root-cause discipline over quick patches.** Fixes are traced to their actual mechanism against real captured data — reading the underlying data-layer behavior directly rather than guessing from symptoms — even when that takes longer than a plausible-looking patch would have.
- **Fixes are proven at small scale before being trusted at full scale.** A change is verified against a minimal real sample first and only rolled out to the full watchlist once that sample is clean, rather than validated once and assumed to hold at scale.
- **Forward-only correction, not silent rewriting of history.** When a fix changes what a report would say, already-published reports are left as they were rather than silently rewritten; the fix takes effect on the next natural regeneration, so historical output stays an honest record of what was actually published at the time.
- **A recurring, scheduled self-audit — not just one-time testing.** A dedicated automated sweep re-checks already-published output on an ongoing basis, catching drift (a source going stale, a citation rule regressing) after launch, not only during initial build-and-verify.

---

## Engineering Under Hardware Constraints

The design decisions above are best understood against the machine this was originally built on — because almost every architectural choice was a response to it.

**The original hardware was deliberately modest:** a **2-core CPU, 8 GB of RAM, roughly 96 GB of disk, and — critically — no GPU.** Every piece of AI inference ran on that CPU. There was no accelerator to fall back on, no managed autoscaling group, no spare node to fail over to. One box, always on, doing everything.

At that size, RAM was the binding constraint, and the margins were genuinely thin:

- The permanent base services (workflow engine, relational DB, vector DB, OS, and control plane) already consumed roughly **1.6 GB before any AI work happened.**
- The mid-tier reasoning model needed roughly **4.4 GB more** the moment it loaded.
- That put a *realistic* working load around **85% of the 8 GB ceiling**, with worst-case simultaneous scenarios pushing past **90%** — leaving on the order of a single gigabyte of headroom to absorb any spike.

That envelope wasn't just tight on paper — it bit. Early on, a local audio-transcription component spun up and **saturated the CPU past 1000% (all cores pinned), starving the database and cascading into a system-wide stall.** The fix wasn't a bigger tune — the component was removed from the stack entirely, and the incident became a permanent design rule: nothing is allowed to contend for the whole machine at once.

Everything in the "resilience engineering" section above exists because of this envelope:

- **Models unload the instant they finish**, so two large models can never be resident together.
- **Heavy work runs strictly one item at a time**, in a single bounded loop, so memory and execution context can't balloon.
- **Vector storage is bounded by design** (one summary vector per document, on-disk payloads, tuned index parameters) so the search DB doesn't grow without limit.
- **Every container has a hard memory cap** with a small safety buffer, so no single service can starve the others.
- **A guard prevents concurrent heavy jobs**, queueing work and returning an estimated wait rather than risking an overlap.
- **Automatic OOM recovery** (restart-on-failure, a memory-monitoring watchdog with email alerts, and a post-restart health check that verifies data integrity) means the system heals itself without a human present.

The result is a platform that was engineered to **degrade gracefully instead of falling over** on hardware most people would consider too small for a single local LLM, let alone a crew of cooperating agents plus a full data pipeline.

> *The server has since been upgraded (to a 4-core CPU, ~16 GB RAM, ~193 GB disk), so memory is no longer the tight constraint it was — but the discipline that the original envelope forced is still in place, and the system is far more robust for having been built under real pressure rather than on comfortable hardware.*

---

## Current Limitations

- **Single-node deployment.** The platform runs on one server. There is no horizontal scaling or high-availability failover today; it is sized for a single power-user / small-team research workload, not public multi-tenant scale. (See *Engineering Under Hardware Constraints* above for how much this shaped the design.)
- **CPU-only inference.** There is no GPU, so local model choice is bounded by what runs acceptably on CPU — which is exactly why the local tier is deliberately small models, with heavier reasoning selectively offloaded to a hosted model.
- **Data-access ceilings.** External data access is rate- and volume-limited, which imposes daily ceilings and drove an incremental, cached, nightly collection design. Higher-frequency or intraday coverage would require a different access tier.
- **Video transcription is partial.** The caption-based transcription path is available; a full audio-fallback transcription path is deferred pending a replacement service (an earlier local-transcription approach was retired after it proved too CPU-heavy for a shared box).
- **Output distribution is currently owner-scoped.** The automated vertical jobs deliver to a single owner mailbox today; external/subscriber-facing delivery is designed but not yet wired.
- **No live-trading loop.** The system analyzes, scores, and reports; it does not place trades. An entry/exit execution framework is intentionally gated behind establishing a real track record first.

---

## Roadmap & Future Integrations

- **Full video-transcription path** — replacing the retired local transcription component with a sustainable service so audio-only sources are covered.
- **External / subscriber delivery** — extending the automated vertical jobs from owner-only mailboxes to external recipients and free-funnel distribution, turning research output into products.
- **Additional structured-signal coverage** — bringing more chart-only and otherwise non-feed sources into the pipeline.
- **Entry/exit decision framework with paper trading** — a risk-managed execution layer validated against a paper-trading broker, activated only after a multi-month analytical track record exists.
- **Latency-optimized nightly processing** — restructuring the nightly batch job so reports compose progressively per-asset instead of waiting on the single slowest item in the run, as the watchlist has scaled up.

*(Since the prior version of this document: a macro-event and prediction-market-verification layer, and a closed-loop forecast-accuracy evaluation system, both moved from roadmap into the live "Core Capabilities" above.)*

---

## Technology Stack

| Layer | Technology |
|---|---|

| Local LLM runtime | Ollama (CPU inference) |
| Embeddings | Local embedding model, no external exposure |
| Vector database | Qdrant |
| Relational database | PostgreSQL |
| Containerization / deployment | Coolify + Docker |
| OS / host | Ubuntu on an owned VPS |
| Document rendering | Pandoc + HTML-to-PDF engine (containerized service) |
| OCR | Tesseract via a lightweight HTTP service |
| Hosted model (selective) | A hosted mid-tier model for heavier reasoning tasks |
| Self-hosted search | Self-hosted metasearch, used to backfill supporting evidence for evidence-thin claims before they're published |


---

*Prepared 2026-07-09, updated 2026-08-05, from the internal system report and project index. Infrastructure identifiers, credentials, proprietary scoring methodology, and personal data are deliberately omitted.*

---

## License

Copyright © 2026 Teejay Graham.

The text and images in this repository are licensed under the **Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International License (CC BY-NC-ND 4.0)** — you may share this work with attribution, for non-commercial purposes, without modifications. Full terms in [LICENSE](LICENSE); summary at <https://creativecommons.org/licenses/by-nc-nd/4.0/>.
