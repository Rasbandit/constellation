# Design documents: how to read them

> Internal draft, 2026-11-16. Every document here is proposed; none is final. Nothing leaves this repo before review under [`CLAUDE.md`](../../CLAUDE.md). Working name: Constellation. We are compatible with Archipelago, a respected volunteer-built open-source project; these docs describe designs and tradeoffs, never people. Every number is estimated or a design value unless it says counted; we have measured nothing.

## Three reading paths

- **A friend who is not technical (about 20 minutes).** [`../EXPLAINER.md`](../EXPLAINER.md), then [`../ONE-PAGER.md`](../ONE-PAGER.md), then [`00-overview.md`](00-overview.md) sections 1 and 2 (the picture, and one game from link to results), then [`09-connection-and-lobby.md`](09-connection-and-lobby.md) section 0 (what a host sees).
- **A technical partner (about 2 hours).** [`../PITCH.md`](../PITCH.md), [`00-overview.md`](00-overview.md), [`12-start-small-ladder.md`](12-start-small-ladder.md) section 0 (how we start: one small server, then stages), [`03-platform-and-ecosystem.md`](03-platform-and-ecosystem.md) sections 1, 5, 6, 7.2 and 9, [`04-security-trust-safety.md`](04-security-trust-safety.md) sections 1 and 8, [`02-architecture-and-scale.md`](02-architecture-and-scale.md) sections 1, 2 and 5.10, [`13-cost-model.md`](13-cost-model.md) sections 0 and 5, [`10-identity-and-room-ownership.md`](10-identity-and-room-ownership.md) section 0, then [`DECISIONS.md`](DECISIONS.md) Part C.
- **An engineer who will build it (about two days).** 01 (standard), 02 (rooms and scale), 12 (the Stage 0 to Stage 3 on-ramp below rung 1), 05 (Rust layout), 11 (read before building the socket tier), 09 (lobby), 10 (identity), then 04 (security), 06 (regions), 08 (edge and workers), 07 (world format). Keep [`GLOSSARY.md`](GLOSSARY.md) open.

## One paragraph per document

| Doc | What it covers |
|---|---|
| [`00-overview.md`](00-overview.md) | The map: system picture, one walkthrough from invite to results, the merged roadmap and the top risks. |
| [`01-standards.md`](01-standards.md) | The open standard: wire protocol (CWP), world manifest (CGM), seeds, optional patch packages, the SDK contract, the compatibility adapter for existing Archipelago clients, conformance and governance. |
| [`02-architecture-and-scale.md`](02-architecture-and-scale.md) | How rooms work and scale: single-writer room actors over an event log, delivery rules, the capacity and cost model with its inputs, reliability, and the load-test gates. |
| [`03-platform-and-ecosystem.md`](03-platform-and-ecosystem.md) | The platform and the business: product surfaces, the community flywheel, the path to a standard, the studio program, plans, roadmap and kill criteria. |
| [`04-security-trust-safety.md`](04-security-trust-safety.md) | Threat model, sandbox lanes, supply chain, privacy, moderation, legal by design, launch checklists, and the 15 questions for counsel (L1 to L15). |
| [`05-rust-architecture.md`](05-rust-architecture.md) | The Rust build: crates, libraries, API sketches, the room actor, SDK layers, CI, tests and the first 90 days. |
| [`06-multi-region.md`](06-multi-region.md) | Multi-region: consistency per kind of data, placement, failover, time, cost and the invariants a simulator must check. |
| [`07-data-only-world-format.md`](07-data-only-world-format.md) | How far a game world can be pure data: primitives, the authoring format, migration of existing worlds, and the spike plan. |
| [`08-serverless-workers-and-edge.md`](08-serverless-workers-and-edge.md) | What runs where: the cost of sockets, generation lanes, the edge web layer, and what restricted clients need from the protocol. |
| [`09-connection-and-lobby.md`](09-connection-and-lobby.md) | Joining and the staging lobby: client classes, join links, the two ready checks, the host board, launch rules and diagnostics. |
| [`10-identity-and-room-ownership.md`](10-identity-and-room-ownership.md) | Accounts, room roles (owner, co-host, moderator), sign-in and recovery without passwords, sessions, step-up for dangerous actions, and the credential table. |
| [`11-rust-socket-tier.md`](11-rust-socket-tier.md) | The all-Rust socket and room tier: the two-process day-one shape (proposed), Rust engineering rules, building blocks we write, spike plan S1 to S4 and pass criteria T1 to T8. Decisions D-78 (decided) and D-92 (proposed). |
| [`12-start-small-ladder.md`](12-start-small-ladder.md) | The start-small ladder, Stage 0 to Stage 3: one rented machine first, bills, capacity, triggers, honest promises and doc edits. Decisions D-79 to D-91. |
| [`13-cost-model.md`](13-cost-model.md) | The single source for cost: scenarios A to E, prices opened, formulas, break-even and sensitivity. Headline: at 100,000 games a day the estimated range is about $830 to $3,452 a month depending on scenario [E]. Decisions D-93 to D-96. |
| [`14-randomizer-patterns.md`](14-randomizer-patterns.md) | What 16 popular randomizers teach us: a pattern catalog, which candidate primitives have enough evidence, the end-to-end author experience and the central services (shared core, one solver, generated tracker). Research notes in [`../research/randomizers/`](../research/randomizers/00-landscape.md). Decision ids still TBD. |
| [`DECISIONS.md`](DECISIONS.md) | The decision log. |
| [`GLOSSARY.md`](GLOSSARY.md) | The terms, label set and ID namespaces. |

## Where decisions live

[`DECISIONS.md`](DECISIONS.md): Part A (owner direction), Part B (resolved contradictions), Part C (open items with defaults and what each blocks), Part D (decisions added when the docs were merged, D-60 to D-72, D-79 to D-96). Each doc also numbers its own decisions (for example doc 01 D1.4); cite them with the doc number, never bare.

## What is still open

- **Owner decisions:** D-09 and D-70 (web UI language), D-24, D-38, D-39, D-47 to D-52, D-63 (hosting needs an account), D-64 (legacy clients are players only), D-65 (warm standby in a second region at rung 1), D-71 (no ownerless rooms), D-72 (delayed recovery), D-73 to D-78, D-93 to D-96 (cost scenarios), D-79 to D-91 (the Stage 0 to Stage 3 ladder: D-81 waive zero loss, D-84 when to charge, D-85 closed alpha, D-88 hosting location, D-89 Cloudflare Free, D-90 VAT and EUR rate).
- **Lawyer items:** D-27, D-36, D-37, D-40, D-42 to D-46, D-55 to D-57, D-59, D-91. The counsel list is doc 04 section 13 (L1 to L15).
- **Not measured:** every number. The first measurements come at load-test gates 0 and 1 (doc 02 section 8).
