# DECISIONS: decision log for the Constellation design docs

> Internal draft, 2026-11-16. Working name: Constellation. "proposed" is a recommended default the owner authorized us to apply so the docs agree; none is final. "open" needs the owner's business or product judgement. "needs-lawyer" needs counsel before it is decided or published. Not reviewed by a different agent or a human; under `CLAUDE.md` nothing here leaves this repo until that happens. This repo contains our own writing, with citations, and no Archipelago source code, text or assets.

**How to read it.** Part A is the owner's direction. Part B resolves the 38 contradictions from the first overview (old check number shown). Part C is the register of open and lawyer items. Part D holds the decisions added when the docs were merged (D-60 to D-72). Reading order and doc summaries: [`README.md`](README.md). Terms: [`GLOSSARY.md`](GLOSSARY.md). Docs 00 to 10 are all written; rows below name the docs they changed.

**Number labels.** Every number is an estimate or a design value unless it says counted or measured. We have measured nothing.

---

## Part A. Owner direction and cross-cutting decisions

| Id | Decision | Status | Rationale | Docs changed | Blocks |
|---|---|---|---|---|---|
| D-01 | **Web-first, standard-only.** No first-party desktop app (no Tauri), no first-party connector, patcher or launcher, no game-specific clients. Games and mods embed the SDK or implement CWP and connect directly. All first-party surfaces are on the web (PWA fine): configuration, option selection, world and room setup, generation, seeds, admin console, inbox, notifications, join links, console pairing-code entry, spectator and results pages. Third-party bridges, patchers and desktop clients may exist, built to the standard, outside the first-party product and outside the native tier; the compat edge keeps old Archipelago-style clients working. Patch packages become an optional third-party-facing extension (D-07); browser-side patching is a possible later web feature, not committed. | proposed | The moment we ask a person to move or install a file, we have lost the one-minute join. Not building game-specific clients also removes the largest recurring maintenance load. Replaces the earlier launcher helper, connector architecture, local-code trust boundary and patch-on-your-machine onboarding step. | 00, 01, 02, 03, 04, 05, 07, 08, 09 | Studio program wording, SDK scope, onboarding funnel, any external text |
| D-02 | **Event cursors.** One room-wide ordered event log with gapless `seq` used to resume the room log (`resume: {after: seq}`). Per-slot `item_seq` is a derived delivery index carried on item events. One ack per slot, `applied` up to `item_seq`; the separate `recv` ack is dropped. | proposed | Keeps 01's standard wire (room log, replay) and 02's simple exactly-once delivery index. One ack is less client state and still gives the server the "what did the game apply" fact. | 01, 02, 05, 06, 09 | CWP v1 freeze, simulator invariants |
| D-03 | **Tokens and scopes.** All tokens are 256-bit random values, rotatable, hash-only at rest, never in URLs, sent in the first frame. Slot tokens carry `play` and `observe` only, never admin. Admin and moderator powers come from an account-bound room role (doc 10 sections 2.3 and 7.3), never from the token. `view` is renamed `observe`. | proposed | A leaked slot token (in a screenshot, a stream, a shared client config) must never grant kick, release or moderation. Stricter length from 01, separation from 04. | 01, 04, 05, 09, 10 | CWP v1 freeze |
| D-04 | **Legacy-world sandbox.** Day one runs no unreviewed third-party legacy worlds. Vetted (reviewed by us) legacy worlds may run under gVisor, no network, strict quotas, at rung 1. Before any unreviewed upload is accepted, a Firecracker microVM, fresh per job, never reused across jobs or tenants, is required. | proposed | Matches risk to isolation strength and removes the KVM requirement from day one. Resolves the gVisor versus Firecracker disagreement. | 02, 04, 05, 01 (3.11) | Opening unreviewed uploads; build order |
| D-05 | **Security gates by rate or count.** Gate S1: first public user. S2: 10,000 games per day sustained 7 days, or 100,000 cumulative games, whichever first. S3: 100,000 per day sustained 7 days, or 1,000,000 cumulative. S4: 1,000,000 per day sustained 7 days, or 10,000,000 cumulative. All thresholds are estimates (proposed, not measured). The old "before the first million games" items are re-keyed to S2 and S3. | proposed | At 100,000 games per day, 1,000,000 cumulative is about 10 days (counted by arithmetic), so a cumulative-only gate arrives faster than the work. | 04, 02 | Launch plan, SOC 2 timing |
| D-06 | **Public directory.** MVP rooms are link-only. A public directory of games (world pages) is allowed from MVP. A public directory of rooms opens only after moderation tooling and the report flow exist. G3 becomes "playable by link in under 1 day; listed after review". | proposed | Child safety and moderation load; aligns 03 with 04. | 03, 04 | Phase 1 scope |
| D-07 | **Patch packages and "client-side code".** Patch packages (CPK) are data (diffs) with schema, size caps and scanning, not executable code, and are an optional third-party-facing extension (D-01). "Client-side code" for the unverified tier means anything executable on a player's device (mods, plugins, bridges, scripts, native programs); banned for unverified. CPK diffs and connector profiles are allowed as data. The applying tool is third-party. | proposed | Removes the 04 versus 01 contradiction without letting code through. | 01, 04, 03 | Registry trust tiers |
| D-08 | **Race integrity.** Tournament and verified race modes require native worlds at the Reproducible tier. Legacy-world rooms may run casual races without verification, labelled "committed, not regenerable". | proposed | Independent regeneration is the only strong fairness proof. | 01, 04, 02 | Race mode scope |
| D-09 | **Web UI language.** Superseded as the working default by D-70 (Leptos first, TypeScript fallback gate); stays open until the owner confirms D-70. Under either choice the TypeScript rule evaluator is the independent conformance implementation of the CGM rule language. | **open** (superseded by D-70) | Owner direction is Rust as the main language. | 05, 07, 08 | Web build start |
| D-10 | **Data-only worlds (doc 07).** Pure data is the working starting direction for native version 1, not locked. The Python sidecar handles everything else. WASM extension modules remain a reserved, disabled hinge. | proposed | Smallest sandbox surface; keeps the door open. | 07, 01, 05 | Doc 07 build, CGM v1 |

## Part B. The 38 former contradictions

| Id | Old check | Decision | Status | Rationale | Docs changed | Blocks |
|---|---|---|---|---|---|---|
| D-02 | 1 | Cursor model: see Part A. | proposed | See above. | 01, 02 | CWP v1 |
| D-03 | 2 | Token scope model: see Part A. | proposed | See above. | 01, 04 | CWP v1 |
| D-11 | 3 | Accounts issues and revokes tokens and runs pairing (a web page where a code from a console is entered); the room shard stores only the hash. One word for the result: slot token. | proposed | One owner per secret. | 01, 02, 04, 10 | Accounts design |
| D-12 | 4 | Room key is a separate random 10-character base32 value (about 50 bits, counted by arithmetic) mapped to `room_id`; region hint lives in the hostname or a directory lookup. | proposed | Typeable key; 02's 15-character derived key is not. | 01, 02, 06, 08, 09 | CWP v1 |
| D-13 | 5 | Dedupe on `(slot, cid)` for at least 24 hours, time-bounded; bitmap kept for checks. | proposed | Follow the standard's promise. | 01, 02 | Room core |
| D-14 | 6, 7 | Exact server-recorded acks are native only; compat clients get heuristic acks and weaker guarantees. Strict reachability validation is native-world only; legacy rooms get existence and duplicate checks only. | proposed | Honest about what an old protocol can prove. | 04, 01 | Compat edge |
| D-04 | 8 | Legacy sandbox: see Part A. | proposed | See above. | 02, 04 | Rung 1 |
| D-15 | 9, 10 | Uploaded WASM that is not first-party runs inside the microVM (or vetted gVisor host): two layers. The declarative interpreter always runs in the generation fleet, never in the serving process. | proposed | Matches 02's isolation of serving from generation. | 04, 02, 05 | Worker design |
| D-16 | 11 | Job limits default to 1 vCPU, 2 GB, 300 s, with per-plan raises up to 02's 600 s and 4 GB cap; re-run the cost sensitivity for heavy legacy worlds. | proposed | Stricter default, explicit exception path. | 02, 04 | Worker config |
| D-17 | 12 | Seed cache is shared only for public presets and daily seeds; private seeds are wrapped under the room key. | proposed | Keeps per-room crypto-erase meaningful. | 02, 04 | Generation cache |
| D-08 | 13 | Race integrity: see Part A. | proposed | See above. | 01, 04 | Race mode |
| D-18 | 14, 37 | Close 02 open question 8 (answered by 01 D3.3 and 04 section 7.3); fix four cross-reference errors. | proposed | Clean-up. | 01, 02, 04 | none |
| D-19 | 15, 16 | Signing follows 04 (keyless identity, transparency log, platform countersignature; answers 01 q7). Clients check a signed revocation list wherever bytes came from; add the check to CSDK and CPK. Whose root signs after a neutral transfer stays open (company root in Stage 0, rotate at transfer). | proposed (root custody open) | One trust model; mirrors stay possible. | 01, 04 | Signing design |
| D-20 | 17 | Outer package envelope wraps CGM and an optional CPK; base-file hashes live only in the CPK; one id scheme. | proposed | One source of truth. | 01, 04 | Registry schema |
| D-21 | 19, 20 | Glossary: Plan, Trust level, Conformance tier, Pool. Badges: "Reachability tested", "Studio-verified", "Conformance-verified". Mark follows 01, "Constellation Compatible"; "Multiworld ready" is the Game class or dropped. | proposed (mark needs-lawyer) | One word, one meaning. | 01, 02, 03, 04 | Mark policy |
| D-22 | 21 | Add the missing components to 02's table (pairing as an accounts function, moderation, signing and transparency log, scanners and build workers, abuse engine, conformance register, inbox); use `edge-compat` and `room-core`. | proposed | Every doc names the same parts. | 02, 05 | none |
| D-06 | 22 | Directory timing: see Part A. | proposed | See above. | 03, 04 | Phase 1 |
| D-23 | 23 | One `config` module, one quota table, starting from 04's numbers; guests join only. | proposed | 02 already requires a single config source. | 02, 03, 04, 09, 10 | First public user |
| D-24 | 24 | Foundation trigger: first of 2 independent implementations, a written third-party request, or month 24. | **open** | Owner business and governance judgement. | none (default noted) | Spec announcement |
| D-25 | 25 | Publish protocol transcripts and vectors in Phase 1; the reference server in Phase 2. | proposed | The standard needs vectors before implementers. | 03, 01 | Phase plan |
| D-26 | 26 | Rooms follow 02 (RPO 0 for acked events); registry follows 04's numbers; reconcile backup retention. | proposed | Stricter for what loses data. | 02, 04 | SLA |
| D-05 | 27 | Gates: see Part A. | proposed | See above. | 04 | Launch plan |
| D-26b | 26 | Backup retention default set by doc 04 editor: 14 days for room WAL (02), 30 days for the registry (04). Owner to confirm. | proposed | Reconciles 14 vs 30 days. | 02, 04 | DR plan |
| D-27 | 28 | Residency: default follow 02 (EU rooms pinned to EU). | proposed, **needs-lawyer** | Retrofitting is expensive; legality is for counsel. | 02, 04, 06, 10 | EU launch |
| D-28 | 29 | Chat payloads sit under a separate key with a 30-day TTL; the log keeps the event and loses the text. | proposed | Meets 04's retention without breaking per-room erase. | 02, 04, 01 | Retention |
| D-29 | 30 | Capacity and cost inputs follow 02's model; 03 section 7.4 and 04 section 9 point at it. | proposed | Formulas are shown there. | 02, 03, 04 | Pricing |
| D-30 | 31 | Abandoned-room rule: no connection and no event for 30 days ends the room as abandoned (design value); mean lifetime stays a measurement target. | proposed | Rooms need a defined end. | 02, 03 | Storage model |
| D-31 | 32, 33 | Add `prev_hash` for tournament rooms, a ping frame to CWP, and a revocation check to CSDK. | proposed | Closes missing hooks. | 01, 02 | CWP v1 |
| D-32 | 34 | Drop SMS; channels are web push, email, Discord, webhook. | proposed | No carrier cost model, no SMS in 02. | 03, 02 | Notify scope |
| D-33 | 35 | Rooms keep running the world version they were generated with, except when revocation is for active harm. | proposed | Reconciles 03 and 02 with 04. | 02, 03, 04 | Revocation design |
| D-34 | 36 | Label words: measured, counted, estimated; map older labels (cited, quoted, A/E/C/M) when anything goes external. | proposed | Matches `CLAUDE.md`. Extended by D-68. | all | External text |
| D-35 | 38 | Chat is an extension, off by default for guests; 03 wording aligned. | proposed | Child safety. | 03, 04, 01, 10 | MVP chat |

**Tally.** 38 contradictions: 36 resolved as proposed (two with a sub-part still open: D-19 root custody, D-21 mark wording needing counsel), 1 left open for the owner (check 24, D-24), 1 proposed but needs a lawyer (check 28, D-27). Plus D-09 is open (superseded by D-70) and not from the old list.

## Part C. Remaining open decisions (old register D1 to D33)

All keep their recommended default from the old register, but none is decided. Status is open unless noted.

| Id | Old reg | Decision | Status | Default | Blocks |
|---|---|---|---|---|---|
| D-36 | D1 | Product and mark name | needs-lawyer | Trademark search first, plan a rename path | Any public text |
| D-37 | D2 | Entity, funding route, team | needs-lawyer (entity) | Stage 0 unfunded; decide before month 6 | Hiring |
| D-38 | D3 | Private heads-up to Archipelago maintainers | open | Before anything public, no named individuals | Announcement |
| D-39 | D4 | Shared fund and size | open | Commit in principle, amount later | Public commitment |
| D-40 | D5, D6 | Neutral foundation commitment and spec editor authority in Stage 0 | needs-lawyer | Small public steering group; announce triggers at launch | Spec 1.0 |
| D-41 | D12 | Registry trust root after transfer (see D-19) | open | Company root, rotate to foundation | Signing |
| D-42 | D14 | Age policy | needs-lawyer | As 04 pending counsel | First public user |
| D-43 | D15 | Data retention and deletion window | needs-lawyer | As proposed; separate chat TTL | First public user |
| D-44 | D17 | Mainland China and licensing-heavy markets | needs-lawyer | Defer | Wave 2 languages |
| D-45 | D19 | Paid worlds and creator payouts | needs-lawyer | None for 2 years, then studio-owned only | Phase 2 pricing |
| D-46 | D20 | Studio terms, mark pricing, paid review | needs-lawyer | Free for open implementations, paid review for commercial | Pilot 2 |
| D-47 | D22 | PRNG, hash input, diff format choice | open | Keep 01's choices; crypto-aware review; check diff licenses (diffs now optional, D-01) | CFill vectors |
| D-48 | D23 | Adapter scope (rule conversion, fixed ports) | open | No rule conversion at first | Rung 1 |
| D-49 | D24 | Room lifetime assumption | open | 3 days assumption, measure in Gate 1 | Pricing |
| D-50 | D25 | Native-generator investment before rung 2 | open | Port top games by measured volume | Rung 2 cost |
| D-51 | D26 | Purpose-built log earlier than rung 3 | open | Keep the `LogStore` seam | Rung 3 |
| D-52 | D27 | First engine for the SDK and accessibility minimum | open | Godot and Unity together; engine-less direct protocol also supported (D-01) | SDK scope |
| D-53 | D28 | Mobile push: own app or Web Push plus Discord | proposed | Web Push (PWA) plus Discord; no native app (D-01) | Notify scope |
| D-54 | D30 | Disaster recovery targets (see D-26) | proposed | See D-26 | SLA |
| D-55 | D31 | Privacy and mandatory-reporting mapping | needs-lawyer | Counsel before first public user | First public user |
| D-56 | D32 | Publisher posture, anti-circumvention, takedown agent, safe harbor | needs-lawyer | Never host game files; counsel before launch | Launch |
| D-57 | D33 | Console platform rules, insurance, entity, disclosure wording, license review | needs-lawyer | Before first studio | Studio pilot |
| D-58 | new | Browser-side patching as a later web feature (D-01) | open | Not committed; revisit after launch and counsel (D-56) | none |
| D-59 | new | Doc 04 added counsel question L15 (residency pinning); counsel list now L1 to L15 | needs-lawyer | Counsel before EU launch | EU launch |
| D-73 | 09 sec 13 | Lobby defaults: launch mode default (gated, `open` as host toggle), invite secret in the URL fragment (128 bits), legacy ready is "unverified" with host Accept, probe levels (p0 mandatory, p1 recommended, p2 optional), seat caps (64 soft, 256 hard), token binding to a device key (defer) | open | The default shown in doc 09 section 13; six sub-choices, one row until the owner reacts | Lobby build |
| D-74 | 10 sec 12 (O2, O3, O5, O8 to O13) | Identity details: sign-in methods at S1 (passkey, Discord, Google, email link), no own passwords, recovery delay and restriction (see D-72), session idle 14 d and absolute 90 d, teen hosting private by default (needs-lawyer, D-42), Steam linking only at S2, SSO for studios at first studio, support never grants ownership alone, dangerous-scope API keys allowed with step-up | open | The defaults in doc 10 sections 3, 6 and 11 | Accounts build |
| D-75 | 06 sec 15 (Q1, Q4, Q5, Q6) | Region pair and witness location for rung 1, RPO for free rooms (about 1 minute), first jurisdictions (US and EU) and whether EU pinning is paid, `sync_partner` default for tournament rooms | open | The defaults in doc 06 section 15 | Rung 1 |
| D-76 | 05 sec 21 | Conformance repository timing (monorepo first, publish later) and MSRV window for published crates | open | Develop in the workspace, publish from a separate repository | Conformance repo, SDK crates |
| D-77 | 08 secs 7.2, 8.1 | Vendor stance (S3-compatible storage and Postgres everywhere, one edge vendor, burst lanes in a separate account) and denial-of-wallet policy (hard caps and a daily budget breaker that pauses a lane rather than spend) | open | The defaults in doc 08 sections 7.2 and 8.1 | Rung 1 vendor choice |

Register items D7, D8, D9, D10, D11, D13, D16, D18, D21, D29 are covered by D-03, D-02, D-04, D-04/D-05 (generation pool stays separate), D-07, D-06, D-27, D-23, D-08, D-05.

## Part D. Decisions added during the doc merge (D-60 to D-72)

Each row says where the rule is owned. "proposed" rows were picked by the cleanup plan where two docs disagreed; "open" rows need the owner.

| Id | Decision | Status | Rationale | Docs affected | Blocks |
|---|---|---|---|---|---|
| D-60 | **Epoch in the cursor.** Resume stays `resume: {after, epoch?}`; `welcome` carries the room epoch; a failover is announced by event `room.epoch_change`. Additive to D-02. | proposed | Doc 06 needs the epoch to detect a lost tail after failover; the field is optional so single-region rooms ignore it. | 01, 02, 06, 09 | CWP v1 freeze |
| D-61 | **Heartbeat.** 15 s in staging, 30 s when live, advertised in `welcome.limits.hb_ms`. Doc 02 capacity numbers are recomputed at 30 s. | proposed | Doc 09 section 5.5 owns lobby timing; fewer frames per second is cheaper. | 01, 02, 09 | Capacity numbers |
| D-62 | **Credential kinds and lifetimes.** One table in doc 10 section 2.8: slot token, session record, session ticket (5 min), connect ticket (60 s), command ticket (60 s), integration credential, API key (`cstk_`), invite secret (128-bit), room claim secret (128-bit), pairing code (8 chars). | proposed | One owner per secret; 08 and 10 had two different ticket lifetimes. | 01, 04, 08, 09, 10 | CWP v1, accounts |
| D-63 | **Hosting requires an account; joining never does.** Starting a room creates a passkey or Discord account in one tap. No anonymous host and no bearer "host link". A trial room is reconsidered only if sign-in drop-off is measured. | **open** (owner) | Fixes the cookie-bound admin design that fit a volunteer project with no accounts: a cleared cookie must not cost a host the room (doc 10 sections 1.2, 4.3). | 03, 09, 10, PITCH, EXPLAINER | Accounts build, onboarding funnel |
| D-64 | **Legacy clients are players only.** They carry `play` scope, hold no role, and chat text is never an admin command. Optional `compat_safe_commands` (default off) allows harmless read-only commands. | **open** (owner) | The compatibility edge cannot carry step-up. Product tradeoff: some hosts run admin commands from their game client today. | 01, 09, 10 | Compat edge |
| D-65 | **Rung 1 includes a warm standby in a second region.** The room's home region is chosen at creation and can be moved (doc 06). | **open** (owner) | Doc 06 sections 12 and 15 assume it; doc 02 assumed a standby beside the primary. | 02, 05, 06 | Rung 1 plan and cost |
| D-66 | **Errors are dotted keys plus a numeric code** (`room.rolled_back`, `resume.too_old`, `module.fuel`, `capability.missing`, `room.unavailable`, `auth.step_up_required`). No `E_` names. | proposed | One registry (doc 01 D1.7). | 01, 04, 06, 07, 08, 09, 10 | CWP v1 |
| D-67 | **Extension frames.** Wire form per doc 01 D1.9 (`{"op":"ext","ext":"org.cwp.lobby","type":"ready"}`). Short names such as `lobby.ready` are shorthand and say so once. | proposed | Avoids two naming schemes. | 01, 09 | CWP v1 |
| D-68 | **Label set.** CLAUDE.md labels (measured, counted, estimated) plus qualifiers: design value, assumption, cited, not verified. Shorthand tags allowed only in 02, 05, 06 and 08 with a header map: [A] estimated (assumption), [E] estimated (formula shown), [C] counted, [M] cited, [?] not verified. "dv" in doc 10 means estimated (design value). Extends D-34. | proposed | One vocabulary, honest about the fact that nothing is measured yet. | all | External text |
| D-69 | **ID namespaces.** D-nn (this log), doc NN D1 (a doc's own decisions, always cited with the doc), L1 to L15 (counsel list, doc 04 section 13), W1 to W5 (doc 08), LD-01 to LD-18 (doc 09, now summarized in D-73), RG1.x (doc 06 gates), Q1 to Q8 (doc 06 requirements). | proposed | Removes the collisions found in the audit. | all | none |
| D-70 | **Web UI: Leptos SSR plus islands first, with a TypeScript fallback gate.** If the web budgets fail at load-test gate 1, fall back to thin TypeScript. D-09 stays open until the owner confirms this default. | **open** (owner) | Owner direction is Rust as the main language. Doc 05 section 4.1 and doc 08 section 5.1 both state it; supersedes the D-09 working default. | 05, 08 | Web build start (same as D-09) |
| D-71 | **No ownerless rooms.** Every room has 1 to 3 owners (design value); the last owner cannot leave without transferring or closing; there is no "disown to nobody". Communities that want no single boss use an org with several owners. | **open** (owner) | A room with no owner cannot be moderated or recovered (doc 10 sections 2.6, 8). | 09, 10 | Accounts build |
| D-72 | **Delayed recovery.** Email-only recovery waits 48 hours with notices on every channel and a cancel link, then a 7 day restriction on dangerous actions. Support never grants ownership alone. Both are design values. | **open** (owner) | Balances friction against account hijack (doc 10 section 3.5). | 04, 10 | Accounts build |
