# GLOSSARY

Internal draft, 2026-11-16, proposed and not final. One line per term, with the doc and heading that owns it; if a definition here differs from its owner, the owner wins. Working name: Constellation. Decisions: [DECISIONS.md](DECISIONS.md). Map: [00-overview.md](00-overview.md).

## A. Product and standard names

| Term | Definition | Owner |
|---|---|---|
| CWP | Constellation wire protocol: rooms, identity, events, transports, extensions | 01 D1.1 to D1.9 |
| CGM | Game manifest format: a world as data (options, items, locations, rules, ID ledger) | 01 D2.1 |
| CPK | Patch package: an optional, third-party-facing diff, data only, never publisher bytes (D-07) | 01 D4.1 |
| CCP | Connector profile: addresses and logic for a third-party bridge, defined once | 01 D4.3 |
| CSDK | The SDK contract; "SDK" means a reference implementation of it (`sdk-core` in Rust) | 01 section 3.5, 05 section 6 |
| CRFC | Public change process for the standards | 01 D6.3 |
| Constellation Compatible | The mark, with a class suffix (Server, Client, Generator, World, Patcher, Bridge, Game); needs counsel (D-21) | 01 D6.2 |
| extension | A protocol extension on the wire (`org.cwp.<name>`, `x.<name>`, `com.<vendor>.<name>`); short names like `lobby.ready` are shorthand for the wire form | 01 D1.9 |
| world module | WASM code inside a world; reserved and disabled in native v1 (D-10); not the same as a protocol extension | 01 D2.8, 07 section 6.2 |
| primitive | A built-in building block a pure world is composed from | 07 section 2.3 |
| pure world | A native world that is data only, with no module | 07 section 2.1 |
| compat edge (`edge-compat`) | The only component that speaks Archipelago's wire behavior | 01 section 3.8 |
| compatibility adapter | The compat edge plus the legacy sidecar | 01 section 3.8 |
| legacy world | An Archipelago Python world, run in the sidecar | 01 section 3.8 |
| opaque world | The synthesized manifest of a legacy world; Valid tier at most | 01 section 3.8 |
| legacy client | An existing Archipelago client connecting through the compat edge | 01 section 3.8 |

## B. Rooms and delivery

| Term | Definition | Owner |
|---|---|---|
| room | One running multiworld session | 02 D1 |
| room key | Public 10-character base32 address (about 50 bits, counted by arithmetic), not a credential, no region inside (D-12) | 01 D1.1 |
| room id | Internal 64-bit id, never shown | 02 section 5.5 |
| join link | `https://{host}/j/{room_key}#{invite_secret}`; per-slot and spectator links are named variants | 09 section 4.2 |
| seq | Room-wide event log position, gapless | 01 D1.4 |
| item_seq | Per-slot delivery index derived from the log, carried on item events | 01 D1.4 |
| item_uid | Stable item identity, `H(room, location_id)`, the same across epochs | 01 D1.4, 06 section 11 |
| cid | Client message id, the idempotency key everywhere (carried by the `Idempotency-Key` header over HTTP) | 01 D1.4 |
| ack | Client frame `ack {applied: item_seq}`; the server's reply to a command is `result` | 01 D1.4 |
| resume | Client cursor `resume: {after, epoch?}` sent in `hello` | 01 D1.4 |
| applied_seq | Storage name of the cumulative applied cursor per slot | 02 section 5.5 |
| epoch | Room epoch (fence counter on each placement); the region epoch is a separate counter | 06 section 6.3 |
| error key | Dotted lower-case key plus numeric code, such as `room.rolled_back` | 01 D1.7 |
| product phase | The room's lifecycle from draft to archived | 09 section 7.1 |
| residency state | Whether a room is in memory (Resident) or not (Cold); orthogonal to the phase | 02 section 4.3, 05 section 8.2 |
| storage class | hot, warm or archive: where a room's data sits | 02 section 5.4 |

## C. Identity and credentials

| Term | Definition | Owner |
|---|---|---|
| account | A person or organization identity; proves nothing about any room | 10 section 2.1 |
| guest | A random participant id plus a slot token; no email | 10 section 2.2 |
| principal | A guest id or an account id | 09 section 2 |
| slot, seat | The game seat in a room: "slot" in the protocol and schema, "seat" in lobby and UI copy | 09 section 2, 10 section 2.1 |
| room role | An account's row in the room's role table: owner, co-host or moderator; "host" is informal for owner or co-host | 10 sections 2.3, 2.4 |
| platform moderator | Our staff, distinct from the room role moderator | 04 section 7.8 |
| session, step-up | Server-side browser session; a fresh proof required for dangerous actions | 10 sections 6.1, 6.5 |
| session ticket, connect ticket, command ticket | Short-lived tokens (5 min, 60 s, 60 s): browser to API, to open a socket, to send one admin command | 10 sections 6.1, 7.3 |
| slot token | `cst1_`, 256-bit, hash-only at rest, scopes `play` and `observe` only | 01 D1.2, 10 section 2.8 |
| invite secret, room claim secret | 128-bit secrets for joining and for claiming a room into an account | 10 section 2.8 |
| pairing code | 8-character code a console shows and a person enters on the web | 01 D5.2 |
| integration credential, API key | Account-bound, scoped bot credential; personal key with prefix `cstk_` | 10 section 2.8 |
| claim | Only "seat claim" (09) and "room claim" (10); never a pairing step | 09 section 4.4, 10 section 4.4 |

## D. Lobby

| Term | Definition | Owner |
|---|---|---|
| staging lobby | The page where seats ready up before launch; "ready check" is the feature | 09 section 5 |
| Check A, Check B | Pre-generation check (can this seat play) and post-generation check (has the seed's data) | 09 section 5.1 |
| probe p0 to p2 | Levels of liveness and round-trip test of a seat | 09 section 5.4 |
| attested, verified | Reported by the client, versus checked by the server | 09 section 11.6 |
| lobby_rev | Revision counter that binds a ready fact to a seed and lobby state | 09 section 4.4 (example frame) |
| host console | The web admin for a room (never "console" alone; "game console" means hardware) | 09 section 6 |

## E. Worlds and generation

| Term | Definition | Owner |
|---|---|---|
| seed | The secret randomness that decides the shuffle | 01 D3.1 |
| seed phrase | 4 words from a 2,048-word list, 44 bits (counted by arithmetic) | 01 D3.1 |
| seed bundle | The generated output, canonical and hashed | 01 D3.2 |
| settings string | Compact encoding of a world's options that replays a game | 01 D3.1 |
| generation | The compute job that builds the shuffled game | 02 D7 |
| conformance tier | Valid or Reproducible, for generators | 01 D2.9, D6.1 |
| trust level | unverified, community-reviewed or studio-verified, for publishers | 04 section 5.4 |
| plan | A paid or free product level | 03 section 7.2 |
| pool | A placement group of room nodes | 02 section 4 |
| lane | declarative-native, wasm-native (reserved), legacy-vetted (gVisor), legacy-unreviewed (Firecracker) | 04 section 4.1 |
| fuel | A deterministic instruction budget for WASM | 01 D2.8 |
| microVM, gVisor | Firecracker-style lightweight VM, and a user-space kernel sandbox | 04 section 4.1 |
| `world.pin_item`, `emit_slot_data` | Host API call and world hook names | 01 D2.8 |

## F. Scale and operations

| Term | Definition | Owner |
|---|---|---|
| rung | A capacity step (0 foundation, 1 to 3 by games per day) | 02 section 8 |
| phase | A product roadmap period with kill criteria | 03 section 9 |
| load-test gate | Gate 0 to 3: a measured test a rung must pass | 02 section 8 |
| launch gate S1 to S4 | Security gates by users and games (D-05) | 04 section 11.0 |
| kill criterion | A stated condition that ends or reshapes a phase | 03 section 9 |
| home region, partner region | Where a room lives, and its async replica | 06 section 2 |
| Authority | The global lease store | 06 section 8 |
| region lease, node lease | Time-bound ownership of a region's role, or of a room by a node | 06 sections 4, 8 |
| sync_partner | The partner region that replicates a room | 06 section 2 |
| edge node, edge worker | Our socket-holding node, versus CDN-side code (doc 08 only) | 02 D6, 08 section 5 |
| registry, world page, games directory, room directory | System, page, and two listings (rooms gated by D-06) | 03 section 3.2 |

## G. Labels, abbreviations and IDs

| Term | Definition | Owner |
|---|---|---|
| counted | Counted by us from a file, a page or arithmetic | `CLAUDE.md` |
| estimated | Computed from assumptions or a formula; "design value" is a starting limit, not benchmarked | `CLAUDE.md`, 02 section 5.0 |
| measured | Only if we ran it; none yet | `CLAUDE.md` |
| cited | A value stated by a page we opened | 02 header |
| not verified | From memory; check before use | `CLAUDE.md` |
| tag shorthand | Allowed in 02, 05, 06, 08: [A] estimated (assumption), [E] estimated (formula shown), [C] counted, [M] cited, [?] not verified; "dv" in 10 is estimated (design value) | D-68 |
| RPO, RTO | Data you can lose, and time to recover, after a failure | 06 section 2 |
| p95, p99 | The slowest 5 percent, or 1 percent, of requests | 02 section 6.1 |
| DAU, MAU | Daily and monthly active users | 02 section 5.0 |
| SLO | A service level objective: a target for a measured behavior | 02 section 6.1 |
| PITR | Point-in-time recovery of a database | 02 section 5.4 (backups) |
| KEK, HKDF | Key-encrypting key; HMAC-based key derivation function | 06 section 11, 01 D3.2 |
| OIDC, PKCE, WebAuthn | Sign-in via an identity provider; a safe OAuth code flow; passkeys in the browser | 10 sections 3, 2.8 |
| SBOM, DPIA | Software bill of materials; data protection impact assessment | 05 section 15, 04 section 6.6 |
| D-nn | A row in DECISIONS (cite as D-nn) | DECISIONS |
| doc NN D1 | A decision inside a doc (cite as "doc 04 D1", never bare "D1") | each doc |
| L1 to L15, W1 to W5, LD-01 to LD-18 (retired, summarized in D-73), O1 to O13 | Counsel questions (04 section 13); edge decisions (08); lobby decisions (09); ownership decisions (10) | 04, 08, 09, 10 |
| RG1.x, Q1 to Q8 | Region gates and region requirements (06) | 06 sections 12, 1 |
