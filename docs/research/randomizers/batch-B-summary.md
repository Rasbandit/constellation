# Batch B summary: frameworks, reusable cores and custom logic languages

> Internal research note, 2026-11-16. Not reviewed by a different agent or a human; nothing goes outside this repo before review under [`CLAUDE.md`](../../../CLAUDE.md). Our own writing; numbers are **counted**, **reported**, **estimated** or **not verified** as in the per-project notes. Projects: [05 Randovania](05-randovania.md) (`97f748d`), [06 RandomizerCore plus RandomizerMod](06-hollow-knight-randomizercore.md) (`791b025`, `0923baa`, `cad3d5a`), [07 wotw-seedgen](07-ori-wotw-seedgen.md) (`v5` at `51ad907`), [08 LADXR](08-ladxr.md) (`4325207`).
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. Cross-project patterns
| Pattern | Randovania | RandomizerCore / Mod | wotw-seedgen | LADXR |
|---|---|---|---|---|
| Logic form | JSON requirement trees, GUI editor | Infix strings in JSON, parsed to DNF and RPN | Own indentation language, LSP | Python objects |
| Graph unit | region, area, node, dock | terms, waypoints, transitions | region, anchor, conn | location node |
| Difficulty | 187 tricks with 6 levels (counted) | 21 independent skip tokens (counted) | 4 tiers inside each rule, 32 tricks (counted) | 5 logic levels as code branches |
| Consumable resources in logic | damage per traversal | state logic (soul, charms, health) | health and energy with refills | none found |
| Fill | retcon forward fill with action weights | forward fill by minimal progression steps, stages and groups | forward fill by weighted "solutions" | random plus assumed check, or forward |
| Entrance or door rando | dock lock shuffle, teleporters, docks | map area, full area, room, coupled or not | entrances with loop size groups | 8 modes, generated maps |
| Settings string | bit-packed permalink, version-bound | reflection binary code, version-bound | hash slug (names, does not decode) | diff-from-defaults short string |
| Golden output tests | expected seed hash per game, acceptance files | dated serialized logic snapshots | cross-OS seed file comparison | 26 setting-combination smoke runs |
| Game-agnostic? | yes in design, inside one app | yes, a library | no (patterns only) | no |

## 2. Strongest ideas to adopt (for a game-agnostic core)
1. **Ship the core as a library with a pluggable world format** (RandomizerCore `ILogicFormat`, `LogicManagerBuilder`; `docs/articles/logic_model.md`). Randovania shows the cost of the alternative: its `game_description/` and `generator/` are game-agnostic but only reachable inside the app. Our generator should be a crate and a WASM build with a stable API, and the CGM canonical JSON is the format.
2. **A structured logic database plus a first-class editor plus a committed text print.** Randovania's typed nodes, docks with lock weaknesses, templates and events, edited in a GUI (`gui/data_editor.py`) and dumped to `.txt` next to each JSON for review, is the best authoring loop in the batch. For us: the visual region-graph editor in doc 07 4.6 should be a launch feature, and `cgm print` should produce a stable review text.
3. **Integrity checks as part of the data workflow**: strongly connected components unreachable from the start, recursive templates, undocumented tricks, dock naming (Randovania `integrity_check.py`); cycle detection and commutativity failure (RandomizerCore `CycleDetector.cs`, `forward_fill.md`). Add an SCC check, a "trick needs documentation above level N" check and a commutativity vector to doc 07 5.1.
4. **Difficulty as structure, not scattered conditionals.** Ori's per-rule tier blocks (each tier adds alternatives) and Randovania's trick levels make "harder is a superset" checkable. LADXR's code branches show the cost of not doing it.
5. **Logic overlays for community extension.** RandomizerCore logic edits, macro edits, substitutions and the `?` coalescing operator let third-party mods extend a world without forking it (`Logic/LogicFileType.cs`, `RC/Requests/RequestBuilder.cs`).
6. **Documented fill semantics.** RandomizerCore's forward fill articles state which progression set is chosen and why; Randovania documents action weights in code (`generator/filler/weights.py`). Our CFill v1 spec should explain its guarantees the same way.
7. **Patcher boundary as a schema'd JSON document, patcher runnable standalone** (Randovania `docs/New Game.md`). This matches our slot data plus optional CPK split.
8. **One settings declaration drives CLI, UI, settings string and validation** (LADXR `settings.py` to `www/js/options.js`), with diff-from-defaults encoding, matching D2.3 and D3.1.
9. **Cross-platform determinism tested in CI** (wotw-seedgen `test.yaml` compares Linux and Windows seed files; `calculate_weights` sorts because search order is not portable). Our Reproducible tier should run the same test across OS and across implementations.
10. **Admission policy for new games** (Randovania: owner, separate logic reviewer, forkable patcher, timelines) as a template for our registry tiers.

## 3. Recurring pain points (design costs)
- **Expressive logic escapes into code**: HK state modifiers in C# (22 classes, `RC/StateVariables/`), Ori snippets in a full scripting language, LADXR's whole world in Python, Randovania's per-game bootstrap classes. Each project hit a need beyond boolean item logic (consumable resources, conditional item effects) and solved it in its host language.
- **Version-bound share codes**: Randovania permalinks and HK settings codes only work on the same version; Ori's slug cannot be decoded. Only LADXR's string is a plain diff from defaults.
- **Order sensitivity**: RandomizerCore detects non-commutative items at fill time; Randovania negated requirements and per-traversal damage need a separate resolver with a 600 second timeout.
- **Language-default RNGs** (Python `random`, .NET `System.Random`) tie reproducibility to a runtime; only Ori uses a named PRNG (PCG).
- **Single-maintainer cores** (RandomizerCore 4 contributors, wotw-seedgen 12, LADXR mostly one lead) carry continuity risk.

## 4. Candidate primitives and standard features (with evidence)
Proposed under doc 07 2.4 rule of three; evidence counts independent projects in this batch only.
| Candidate | What it is | Evidence | Rule-of-three status |
|---|---|---|---|
| **`trick` concept** | Named, documented technique; optional ordered levels; `trick(name) >= level` in rules; docs links; UI section | Randovania `resource_database.tricks` and `layout/base/trick_level.py`; HK `Data/logic_settings.json` (21 skips) and `LOGIC_README.md`; wotw `enum Trick` (32); LADXR trick directory in `logic/requirements.py` | 4 projects |
| **`tier` concept** | Ordered world difficulty; rules may carry per-tier alternatives; monotone by construction | wotw tier blocks in `paths.wotwl`; LADXR `logic` setting (5 levels); Randovania trick levels (per trick) | 3 projects |
| **`resource_state` primitive** | Bounded path-dependent integers and flags (health, energy, soul); spend on edges, refill at nodes; dominance pruning of states | HK state logic (`docs/articles/state.md`); wotw orbs and `refill`; Randovania damage resources | 3 projects; expensive, Experimental first |
| **`door_kind` and `relock`** | Connection kinds with lock semantics (front and back behavior); primitive shuffles locks within allowed sets | Randovania dock weaknesses and `dock_weakness_distributor.py`; HK transition direction groups; LADXR entrance flags | 1 strong, 2 partial |
| **`rewire` parameters** | coupled, one-to-one, inside-to-outside, direction matching, loop size, groups | HK transition settings; LADXR three flags; wotw loop size; Randovania docks | 4 projects (extends existing primitive) |
| **`fill_group` and `fill_stage`** | Disjoint item-location pools and staged fills (items, then transitions, then items again) | RandomizerCore stages and groups; LADXR dungeon keys first; Randovania major/minor | 3 projects |
| **`overlay` package** | Ordered build-time patches to a base world (edit rule, edit macro, substitute, optional refs) | RandomizerCore logic edits and coalescing; Randovania layers (option-gated content) | 2 projects; also requested by our registry model |
| **`hint_features`** | Tags with integer precision on locations and items for vague hints | Randovania `docs/Hints.md`; wotw area hints | 2 projects |
| **`pos` metadata** | Coordinates on regions and locations for trackers and editors | wotw anchors and `pickups.csv`; Randovania node coordinates and editor map | 2 projects |
| **`waypoint` flag** | Event that is a cached reachability fact, not an item | RandomizerCore waypoints; Randovania events | 2 projects |
| **Golden seed per world** | Committed expected output hash for a fixed seed and preset | Randovania `GameTestData`; wotw cross-OS compare; RandomizerCore.Json dated snapshots | 3 projects |
| **Fill knobs** | Declared parameters of CFill: local item count, depth bias, forward factor | wotw `UNSHARED_ITEMS`; HK depth priority transform; LADXR forward factor; Randovania action weights | 4 projects |

### 4.1 Changes these suggest to our drafts
- Doc 07 2.3: add `trick`, `tier` and `resource_state` (Experimental) to the candidate list, and extend `rewire` with the parameters above.
- Doc 07 5.1: add the strongly connected component check, the trick documentation check and a commutativity vector.
- Doc 07 4.6: make the region-graph editor and `cgm print` review text part of the first tooling release, and add an LSP for the authoring profile (wotw shows a small team can maintain one).
- Doc 01 D2.2: add optional `pos` metadata and an `area` display grouping; consider `overlay` packages in the registry model (doc 01 section 3.7).
- Doc 01 D2.9 and D6.1: require a golden seed per world and a cross-OS run in the conformance suite.
- Doc 01 D3.1: keep the decodable diff-from-defaults string; avoid binding share codes to one generator version where a manifest hash plus generator id suffices.

## 5. What stays out of the world file
ROM and file patchers (Randovania exporters, LADXR assembly), runtime hooks (ItemChanger, the Ori client and its seed language, Randovania game connectors), tile-level world generation (LADXR wave function collapse), and conditional order-sensitive item effects. These match doc 07 section 6 residue items 6.3, 6.7 and 6.8; the conditional-effects case is a new residue entry.

## 6. Not read or not verified
ItemChanger, MenuChanger and the HK multiworld mod; the Ori client, server and LSP internals; Randovania's GUI preset tabs, server internals and patchers; LADXR graphics, music and most assembly. Ori's license status (no LICENSE file, MIT declared in `Cargo.toml`) needs an owner or lawyer decision before anything beyond high-level description (see 07 section 1).
