# Constellation

Working name. Early research for an easier way to play multiworld randomizers, plus a supported studio integration product for game studios. Web-first and standard-only: everything we build is on the web, and games connect directly through an open protocol and an SDK, with no desktop app, launcher, connector or patcher from us. No code yet. This repo holds our own writing. It describes Archipelago's public code and docs with citations, and builds on the community's work. It includes none of their source code, text or assets.

- [`docs/ONE-PAGER.md`](docs/ONE-PAGER.md): the same story on one page, for chat.
- [`docs/EXPLAINER.md`](docs/EXPLAINER.md): the non-technical version. Share this one with anyone new.
- [`docs/PITCH.md`](docs/PITCH.md): the technical pitch: vision, problem, fix, marketing and monetization.
- [`docs/notes/01-archipelago-findings.md`](docs/notes/01-archipelago-findings.md): what we learned from reading Archipelago, with sources.
- [`docs/notes/02-architecture-sketch.md`](docs/notes/02-architecture-sketch.md): early design thinking.
- [`docs/notes/03-legal-notes.md`](docs/notes/03-legal-notes.md): licensing and legal research. Not legal advice.
- [`docs/notes/04-socket-tier-evidence-beam.md`](docs/notes/04-socket-tier-evidence-beam.md): evidence for and against an Elixir (BEAM) socket tier.
- [`docs/notes/05-socket-tier-evidence-rust.md`](docs/notes/05-socket-tier-evidence-rust.md): evidence from the Rust side for the same question.

## Design documents

Internal drafts in [`docs/design/`](docs/design/), all proposed and not final. Start with [`docs/design/README.md`](docs/design/README.md) (reading order for a friend, a partner and an engineer), then [`00-overview.md`](docs/design/00-overview.md) and [`DECISIONS.md`](docs/design/DECISIONS.md).

- [`00-overview.md`](docs/design/00-overview.md): the map, one walkthrough, merged roadmap and top risks.
- [`01-standards.md`](docs/design/01-standards.md): the open protocol and formats.
- [`02-architecture-and-scale.md`](docs/design/02-architecture-and-scale.md): rooms, generation, capacity and cost.
- [`03-platform-and-ecosystem.md`](docs/design/03-platform-and-ecosystem.md): registry, studios, plans, roadmap.
- [`04-security-trust-safety.md`](docs/design/04-security-trust-safety.md): security, trust and safety, counsel questions.
- [`05-rust-architecture.md`](docs/design/05-rust-architecture.md): server design.
- [`06-multi-region.md`](docs/design/06-multi-region.md): regions, failover and residency.
- [`07-data-only-world-format.md`](docs/design/07-data-only-world-format.md): worlds as configuration data.
- [`08-serverless-workers-and-edge.md`](docs/design/08-serverless-workers-and-edge.md): workers and edge.
- [`09-connection-and-lobby.md`](docs/design/09-connection-and-lobby.md): joining, the staging lobby and the ready check.
- [`10-identity-and-room-ownership.md`](docs/design/10-identity-and-room-ownership.md): accounts, room roles, sign-in and recovery.
- [`11-socket-tier-language-study.md`](docs/design/11-socket-tier-language-study.md): socket tier language study and spike plan.
- [`12-start-small-ladder.md`](docs/design/12-start-small-ladder.md): the Stage 0 to Stage 3 start-small ladder below rung 1.
- [`GLOSSARY.md`](docs/design/GLOSSARY.md): the terms used across the docs.
- [`DECISIONS.md`](docs/design/DECISIONS.md): the decision log, including what is still open.

Agents and contributors: read [`CLAUDE.md`](CLAUDE.md) first. It has hard rules for how we talk about Archipelago.

Items marked **[TBD]** are not measured or decided yet.
