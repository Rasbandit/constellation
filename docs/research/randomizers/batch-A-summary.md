# Batch A summary: the Zelda N64 lineage and Wind Waker

> Internal research note, 2026-11-16. Not reviewed by a different agent or a human. Our own writing with paths and links; no code, text or data from any project is reproduced. Numbers are **counted** in the per-project notes unless labeled otherwise. Commits read: OoTR `526b4c8d` (Dev), SoH `cbfcd827`, OoTMM `29e83036`, wwrando `99281a37`. Details and citations live in [01](01-oot-randomizer.md), [02](02-soh-randomizer.md), [03](03-ootmm.md) and [04](04-wind-waker-randomizer.md).
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. The four at a glance

| | OoT Randomizer | SoH randomizer | OoTMM | Wind Waker Rando |
|---|---|---|---|---|
| Logic form | JSON region files, Python-syntax rule strings compiled through `ast` | C++ functions per rule, source text kept for tooltips | YAML areas, own expression parser in TypeScript | YAML location list plus 307 text macros, own string parser |
| Region graph | yes (486 regions) | yes (1,091 regions) | yes (868 OoT + 467 MM areas) | no; zone access is a macro rewritten by shuffles |
| Fill | staged assumed fill, whole-seed retry 10x | staged assumed fill, per stage and per seed retry 10x | named pass pipeline, assumed fill to goal then shuffle, retry 100x | forward fill with "opens the most" heuristic |
| Entrances | coupled pairs, typed pools, 20 retries per pool | coupled, decoupled, mixed pools, invariant checks | typed pools via a `reverse` field, backtracking | coupled, nested dungeons, sea-inward fill |
| Settings | 284 declared, bit-packed string | about 312, no string, spoiler JSON shares a seed | 446, deflated JSON diff from defaults | 77, base64 permalink, version-locked |
| Integration | ROM patch, own `.zpf` format | native port, generated in the game menu | ROM patch, C payload | full ISO rebuild from the user's disc |
| Multiworld | yes, up to 255 worlds, memory-layout protocol | co-op only (Anchor, JSON over TCP) | yes, binary packets with a write-ahead log | none built in |
| Tests | 35 unit tests, plando fixtures, settings fuzzer | none found for the randomizer | 34 test files, 25 whole-seed | 14 test functions, 5-OS CI |

## 2. Cross-project patterns

1. **Logic as data with expression strings is the majority choice.** Three of four keep rules in data files (OoTR `data/World/*.json`, OoTMM `data/world/{oot,mm}/**/*.yml`, wwrando `logic/item_locations.txt`). The one code-based project, SoH, still stores the rule's source text beside the compiled function so tools can show it (`soh/soh/Enhancements/randomizer/location_access.h`). The demand for a readable rule string is real even where rules are code.
2. **Settings are folded into rules before search.** OoTR folds settings to constants during `ast` rewriting (`RuleParser.py`); OoTMM allows `!` and `cond` only over settings at parse time so the search stays monotone in items (`packages/logic/src/expr/parser.ts`); wwrando option checks are macros (`logic/macros.txt`). This is the same idea as our `opt` resolved before logic (01 D2.5, D2.7).
3. **A second reachability axis beyond items.** All three Ocarina of Time projects track age (child, adult) and time of day per region: OoTR passes age and time into every rule (`RuleParser.py:23-31`, `Search.py`), SoH tracks four facets per region (`location_access.h`), OoTMM adds 46 Majora's Mask time slices with a `stay` rule for where waiting is blocked (`packages/logic/src/pathfind/pathfind.ts`, `expr/data.ts`). Doc 07 has no primitive for this today.
4. **Fill is an ordered list of named stages, not one call.** OoTR (`Fill.py:30-231`), SoH (`3drando/fill.cpp`) and OoTMM (`packages/logic/src/pipeline.ts`) each place restricted things first (shops, dungeon rewards, own-dungeon keys, songs), then progression, then filler. wwrando does keys first in a local pass (`randomizers/items.py`). Every one retries on failure with a fixed cap.
5. **Entrance shuffle uses typed pools with a pair link.** Every project groups entrances by type and links each to its reverse (OoTR `EntranceShuffle.py`, OoTMM `data/defs/entrances.yml` with 611 rows and 40 types, SoH 28 types, wwrando `randomizers/entrances.py`). Validation after each placement is common: OoTR checks against prior reachability, SoH checks that a few anchor areas stay reachable with no items (`entrance.cpp` near line 793).
6. **Hints are configured as weighted distributions over a small set of kinds.** OoTR has 18 distribution files (`data/Hints`), SoH four weighted tables, OoTMM an ordered list of type, amount and duplicates, wwrando path, barren, item and location hints. All derive "required" from a removal test over the playthrough.
7. **Tricks are named, tagged boolean settings.** OoTR 187 glitchless and 186 glitched, SoH 202, OoTMM 154; wwrando instead has two ordered difficulty settings compared as "at least level N" (`logic/macros.txt`).
8. **The settings list drives the UI.** OoTR generates the GUI from settings data (`SettingsToJson.py`, `data/settings_mapping.json`), SoH and OoTMM render menus from their settings arrays, and wwrando binds widgets named after option keys in a loop (`wwr_ui/`).

## 3. Strongest ideas to adopt

- **Seed-affecting versus cosmetic settings, explicit.** OoTR marks 163 of 284 settings as seed-affecting; wwrando keeps 13 options out of the permalink and makes hint settings change the seed so race fairness holds (`randomizer.py`). Our settings layer (01 D3.1) should mark every option `affects_seed` and `affects_knowledge`.
- **Settings strings as a diff from defaults** (OoTMM `v2.` strings) survive new options better than bit-packing in declaration order (OoTR, wwrando permalinks only open in the same version). Prefer the diff form, versioned.
- **Rule source text kept with compiled rules** for tracker tooltips (SoH). Our canonical JSON tree plus a canonical pretty-printer gives this for free.
- **A versioned, fixed-address client memory layout** for emulator bridges and auto-trackers (OoTR `Notes/coop-ctx.md` v7, `Notes/auto-tracker-ctx.md` v6). A good model for CSDK connector profiles (01 D4.3).
- **Write-ahead item log with acks** in the multiworld wire (OoTMM `doc/Multi-Packets.txt`), the same shape as our ordered event log (01 D1.4). Evidence that the design matches what game-side code can carry.
- **Settings fuzzing in CI** that stores failing combinations (OoTR `Unittest.py`) and whole-seed tests over setting combinations (OoTMM `packages/logic/tests`). Our conformance suite should include a fuzz tier.
- **Automatic demotion of items that open nothing** under the chosen settings to filler (wwrando). This is a platform-side analysis we can offer every data world.

## 4. Recurring pain points

- **Name strings as the join key across files.** OoTMM joins YAML, XML checks and code by location name; OoTR joins regions, `LocationList.py` and hints by name. Renames ripple. Our ID ledger (01 D2.6) plus keys addresses this.
- **IDs from table order** (OoTMM) or enums in code (SoH), not an append-only ledger.
- **The same logic exists twice** when another platform needs it: SoH C++ plus a Python port in the external Archipelago-SoH fork; OoTR's logic was reimplemented for OoT3D and SoH. A language-neutral world format is the direct remedy.
- **World shaping in code next to data.** OoTMM `world/transform.ts` (2,338 lines), OoTR warp-song pinning conditionals (`EntranceShuffle.py:487-543`), wwrando macro rewriting after shuffles (`logic/logic.py`, `update_*_macros`).
- **Custom data readers.** OoTR region JSON allows comments and multi-line strings; OoTMM macros are text substitution with no cycle check found. Our strict profile and static cycle check (07 sections 2.2 and 4.2) avoid both.
- **Reproducibility leaks.** OoTMM draws random starting age and clock outside the seed stream (inferred, not tested); SoH hashes option display labels into the seed (inferred, not tested); OoTR and wwrando rely on Python's `random`, so reproducibility is tied to that implementation.
- **Thin tests where logic is code.** SoH has no randomizer tests found, and its in-game plando editor does not recheck logic.

## 5. Candidate primitives and standard features

| Candidate | What it is | Evidence | Rule-of-three status |
|---|---|---|---|
| `state_dimension` (a context axis) | Finite per-player state beyond items (age, time of day, form) with declared transitions; reachability is per (region, state) | OoTR `RuleParser.py:23-31`, `Search.py:131-171`; SoH `location_access.h`; OoTMM `pathfind/pathfind.ts` | Three projects here, but one lineage; needs a second-lineage project |
| `time_window` | Ordered timeline slices, rules true within slices, `stay` blocking waiting | OoTMM `expr/data.ts`, `data/world/mm/overworld/clock_town.yml` | One project; may compose from `state_dimension` |
| `fill_stages` | Ordered named fill passes, each with item filter, location filter and retry cap | OoTR `Fill.py:30-231`; SoH `3drando/fill.cpp`; OoTMM `pipeline.ts`; wwrando `randomizers/items.py` | Four projects; strong |
| `item_rule` extension: own-group | Item restricted to locations sharing its group (dungeon keys) | OoTR, SoH, wwrando `check_item_valid_in_location` | Fits existing `item_rule` if it gains a "same group as item" form |
| `rewire` extensions | Typed pools, reverse pairing, decoupled and mixed modes, nesting, pinned targets, no-item invariants after each step | OoTR `EntranceShuffle.py`; SoH `entrance.cpp`; OoTMM `solver/entrances.ts`; wwrando `randomizers/entrances.py` | Four projects; extend `rewire` rather than add a new primitive |
| `hint_plan` | Weighted hint kinds with counts and copies, over required, barren and path analyses | OoTR `data/Hints`, `Hints.py`; SoH hint tables; OoTMM hints pipeline; wwrando `randomizers/hints.py` | Four projects; strong |
| `rebind` (shuffle result as defs) | A shuffle's outcome rewrites named rule fragments | wwrando `update_entrance_connection_macros`, `update_chart_macros`; OoTR here/at events | Two projects; may be the natural output of `rewire` and `permute` |
| `source_class` | Renewable versus one-time copies of an item for counting | OoTMM `locations.ts`, `compile.ts` | One project; park |
| `shared_item` | One item satisfies rules in two linked games | OoTMM `items/shared.ts` | One project; relevant to combo and linked-slot rooms |
| Option kind `trick` and `level` | Tagged boolean with difficulty and area; ordered level compared "at least" | OoTR `SettingsListTricks.py`; SoH trick list; OoTMM tricks; wwrando `Obscure`/`Precise` | Four projects; an option-schema addition, not a primitive |
| `goal_threshold` | Count over chosen token groups | OoTMM `special-conds.ts`; OoTR bridge and Ganon boss key settings | Likely a recipe over `group_count` and `opt` |

## 6. What cannot be data, in this batch

ROM, ISO and native-port output (OoTR `ASM/`, wwrando `asm/patches/` and `tweaks.py`, SoH runtime item swaps, OoTMM C payload) stays in a third-party patcher or game build, as doc 07 section 6.7 already says. Emulator memory bridges are client artifacts (07 section 6.8). OoTMM's world transform code and OoTR's warp-song conditionals are the residue most likely to need either a richer `rewire` or the reserved module door (07 section 6.1 option b).

## 7. Not verified or not read

OoTR `ASM/` in depth, its presets folder and its website; the SoH PRNG, Anchor server and cosmetics; OoTMM's multiworld relay host and GUI internals; the gclib submodule behind wwrando's file formats and the fork that Archipelago's `tww` world uses. Per-note limits are in each note's last section.
