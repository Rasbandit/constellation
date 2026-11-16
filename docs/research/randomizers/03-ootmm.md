# 03. OoTMM (Ocarina of Time plus Majora's Mask combo randomizer)

> Internal research note, 2026-11-16, read at [OoTMM/OoTMM](https://github.com/OoTMM/OoTMM) commit `29e83036` (shallow sparse clone, nothing run or built). Not reviewed by a different agent or a human. Our own writing with paths and links; no code, text or data from the project is reproduced. Numbers are **counted** (by our own script over the clone, or from the GitHub API on 2026-11-16), **estimated**, or **not verified**. Paths are relative to the repo root.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. Summary
- **What it is.** A randomizer that merges two N64 games, Ocarina of Time (OoT) and Majora's Mask (MM), into one ROM. Items from either game can appear in the other, and the player travels between games inside one play session (`README.md`). It can also run OoT only or MM only (setting `games` in `packages/core/src/settings/data.ts`).
- **Popularity.** 152 stars, last push 2026-11-16, 33 contributors including anonymous (all counted via the GitHub API on 2026-11-16; stars and contributors match 00-landscape). The README points players to a hosted web generator at ootmm.com and a Discord server.
- **License.** MIT (`LICENSE`, and the API reports `MIT`). For us: we may read and describe freely; if a future decision ever reuses any file, the MIT notice must travel with it and go in `THIRD_PARTY_NOTICES.md`. Nothing is copied here.
- **Maintenance.** Active. `CHANGELOG.md` is 1,985 lines (counted) with an Unreleased section on top and release 33.0 dated 2026-10-13. CI runs on every push and pull request (`.github/workflows/ci.yml`), plus `nightly.yml` and `cleanup.yml`.

## 2. Architecture
- **Monorepo, pnpm and Nx** (`package.json`, `nx.json`, `pnpm-workspace.yaml`). Packages under `packages/` (directory listing): `data` (YAML, XML and CSV compiled to JSON), `core` (settings, PRNG, cosmetics, shared types), `logic` (world builder, expression language, pathfinder, solver, entrance shuffle, hints, analysis), `generator` (TypeScript ROM builder in `lib/`, plus the in-game C payload in `src/` with `CMakeLists.txt`), `gui`, `web`, `web-components`, `desktop` (a Neutralino desktop wrapper, `packages/desktop/neutralino.config.json`), `tools`.
- **Languages.** TypeScript for data build, logic, generator and UI; C and some MIPS assembly for the runtime payload (`packages/generator/src/common/*.c`, `*.S`, separate `oot/` and `mm/` trees).
- **Data build step.** `packages/data/build/run.ts` parses every source (files, scenes, NPCs, macros, world, regions, hints, entrances, get-item table, gossip stones, checks) and emits JSON under `lib/`, plus three manifests (`manifests/items.json`, `locations.json`, `entrances.json`) and a C header with flag counts. The generator only consumes the compiled JSON.
- **Logic pipeline.** `packages/logic/src/pipeline.ts` chains named passes, each taking the accumulated state and returning new fields: config, item properties, world build, fixer (decides which locations stay vanilla), world transform, then price, song events, entrances, solver, minimize, then cloak (trap disguises), analysis, path analysis, foolish analysis, hints, hash. This pass-chain shape is the clearest part of the design to reuse as an idea.
- **Generator.** `packages/generator/lib/combo/generator.ts` decompresses both base ROMs, resolves random settings, runs logic, builds one patch file per world, writes the spoiler, then (for a single world) packs the final ROM. A separate `patch` mode takes a patch file and the player's ROMs.
- **How users run it.** Web generator (recommended in `README.md`), desktop app, and a CLI (`packages/generator/lib/cli.ts`, 115 lines counted). The web build runs the generator in the browser (the code checks a browser flag before codegen in `generator.ts`; full browser execution inferred, not verified).

## 3. World model
- **Areas** are YAML maps in `data/world/{oot,mm}/**` keyed by area name. Each area may hold `exits`, `events`, `locations` and `gossip` maps from name to expression string, plus flags `boss`, `dungeon`, `region`, `age_change`, `time` and `stay` (typed in `packages/logic/src/world/types.ts`). Example read: `data/world/mm/boss/goht.yml`.
- **Counts (counted by our script over `data/world`):** OoT 57 files, 868 top-level area keys, 3,424 location entries, 481 event entries, 2,018 exits, 41 gossip entries; MM 43 files, 467 areas, 3,002 location entries, 582 events, 1,377 exits, 60 gossip entries. These include Master Quest dungeon variants (`dungeons_mq/`) and region layout variants (`mm/jp`, `mm/us`), so they exceed what one seed uses.
- **Checks** are a second, separate dataset: XML in `data/checks/{oot,mm}/**` (e.g. `data/checks/oot/dungeons/bottom_of_the_well.xml`), one element per check with its kind (chest, collectible and so on), game flag, scene and vanilla item. Counted: 3,315 OoT and 2,927 MM elements with a `location` attribute, 10 and 7 element kinds. World logic (YAML) and game binding (XML) are joined by location name.
- **Items** are defined in TypeScript (`packages/logic/src/items/defs.ts`, 1,107 lines; groups in `groups.ts`, 1,617 lines, counted). Items are game-prefixed (`OOT_`, `MM_`) with a third class `SHARED_` for items that count in both games when sharing settings are on (`items/shared.ts`).
- **Entrances** are one YAML table, `data/defs/entrances.yml`: 611 entries in 40 types (counted), each with game, in-game id, type (spawn, boss, dungeon, dungeon-exit and so on), the two maps, the two logic areas it joins, and an optional `reverse` partner.
- **Events** are logic-only flags set inside area `events` maps and read with `event(NAME)`.
- **IDs.** Location and entrance keys are names. The items manifest assigns numeric ids by position in the get-item table (`packages/data/build/run.ts`), not from an append-only ledger.

## 4. Logic
- **Representation.** Expression strings inside YAML, parsed by a hand-written recursive-descent parser (`packages/logic/src/expr/parser.ts`, 665 lines) into typed nodes (`expr/types.ts`) and compiled to closures (`expr/compile.ts`). Built-in functions (counted from the parser): `has(item[, n])`, `event`, `setting(key[, value])`, `trick`, `age(child|adult)`, `oot_time`, `mm_time(op, slices...)`, `renewable`, `license`, `masks(n)`, `special`, `price`, `has_pond_fish`, `flag_on`, `flag_off`, two song-event forms, `cond(a, b, c)`, `!`, `&&`, `||`, `true`, `false`, and integer `var(...)` with `+` and `-`.
- **Two design choices worth noting.** (1) Mixing `&&` and `||` without parentheses is a parse error, so precedence never surprises an author. (2) `cond` and `!` must resolve at parse time; the builder throws if they would be evaluated at runtime (`expr/builder.ts`). Settings and tricks are folded to constants while parsing, so runtime logic stays monotone in items.
- **Macros.** Named, parameterized, textual: `data/macros/macros_common.yml` (170 counted), `macros_oot.yml` (363), `macros_mm.yml` (459). The parser expands a macro by re-parsing its body with arguments substituted as tokens. No cycle check was found in the parser (not verified whether the data build checks it).
- **Two-game state model.** The pathfinder (`packages/logic/src/pathfind/pathfind.ts`, 861 lines) keeps a separate reachable-area map per age (child, adult). Age changes only in areas flagged `age_change` once time travel is available, and MM can use adult under a cross-age setting.
- **Time as part of reachability.** Each reached area carries a bit set: three OoT time flags (day, night, dusk) and 46 MM time slices over the three-day cycle (counted in `expr/data.ts`), plus region-state flags (swamp, mountain, ocean, valley cursed or cleared). Time expressions return "true, but only within these slices", and the pathfinder propagates and widens those sets. Areas whose `time` is `flow` let time pass; `stay` lists say in which slices the player cannot wait (example: `data/world/mm/overworld/clock_town.yml`). Tests in `packages/logic/tests/expr/time.test.ts`.
- **Renewable and license counts.** Besides normal item counts, the state tracks items from repeatable sources (shops, cows, scrubs) and items from non-shop sources (a "license", used so shop purchases only count when allowed). See `locations.ts` and the pathfinder collection code.
- **Tricks and glitches.** 154 tricks in `packages/core/src/settings/tricks.ts` (counted), 5 marked as glitches, each with game, name, tooltip and optional video or text link. World data references `trick(...)` on 250 lines (counted).
- **Logic levels.** Setting `logic`: all locations reachable, beatable only, or no logic (`settings/data.ts`).
- **Example in our words.** A boss room area grants a defeat event when a macro says the player can beat that boss; the "after boss" area is reached through an exit whose rule is that event; pots in the room need either short-range fire or a hookshot trick macro.

## 5. Fill and placement
- **Order** (`packages/logic/src/solver/solve.ts`, `run()`): plando, junk-only locations, vanilla-fixed items, dungeon rewards, semi-shuffled items, per-dungeon items (keys, maps) with retries, pre-completed dungeons, then progression.
- **Progression** uses assumed fill: pick a random item from the required pool, compute reachable locations assuming the rest of the pool is held, place it there. It loops only until the goal is reachable (or every location is, under all-locations logic); the remaining required items, then "nice", then junk are spread by plain shuffle (`fillAll`). Under beatable-only logic it tries locations one by one and keeps the first that still allows the goal.
- **Retries.** A logic failure restores a snapshot and retries, up to 100 attempts; a validation mode uses forward fill and 10 attempts (`retry`, `validate`).
- **Pools.** Items are split into required, nice, junk, nothing and extra (named triforce) pools; consumables that are progression in renewable form are split between required and junk.
- **Minimize.** With the "barren" item pool, a later pass replaces placed items with nothing one by one when the seed stays beatable without them (`solver/minimize.ts`).
- **Goals as counted conditions.** Rainbow bridge, moon access, light-arrow cutscene, Ganon boss key and Majora can be "custom": a chosen set of token groups (stones, medallions, remains, skulltula tokens, stray fairies per temple, masks, triforce pieces, four coin colors) and a count (`packages/core/src/settings/special-conds.ts`, 19 fields counted).
- **Starting items, plando and junk locations** are plain settings maps (presets show `startingItems`, `plando.locations`, `plando.songEvents` in `packages/generator/lib/combo/presets.ts`).
- **Progression balancing** in the Archipelago sense was not found.

## 6. Entrance randomization
- `packages/logic/src/solver/entrances.ts` (1,094 lines). Entrances are grouped into named pools by type (regions, overworld, indoors, grottos, warps, one-ways, bosses, dungeons, spawns), with a games-own-pool option.
- **Coupling.** Coupled by default: placing A to B also places the reverse pair from the `reverse` field. A decoupled setting exists (`erDecoupled`). 44 settings keys start with `er` (counted).
- **Method.** Recursive backtracking: pick a pool and a source, try destinations that pass constraints (polarity, self-loop, own-game), check the partial shuffle with the pathfinder on a world where unplaced pool entries are assumed open, recurse; give up after 10 backtracks. Large pools are approximated by assuming the destination set rather than every pair (cardinality over 5,000 and source size over 50).
- **Regions follow entrances.** After shuffling, region names (used by hints) are re-propagated through the new graph (`propagateRegions`).

## 7. Options model
- **Single source.** `packages/core/src/settings/data.ts` (4,614 lines) is an array of setting objects: key, name, category, type, description, default, optional `cond` (a function deciding visibility from other settings) and per-value descriptions. Counted: 446 top-level keys; 321 boolean, 101 enum, 15 number and 11 set type lines. Some entries are generated in a loop (MM age requirements).
- **UI from data.** The GUI renders from this list (`packages/gui/src/app/components/settings/Setting.tsx`, `SettingDependencies.tsx`, file names only; contents not opened).
- **Presets.** 11 named presets (counted in `presets.ts`), from Default and Beginner to Allsanity and two single-game presets.
- **Settings string.** `settings/string.ts`: only values that differ from defaults, as JSON, raw-deflated, URL-safe base64, prefixed `v2.`; v1 and v0 importers are kept.
- **Random settings.** A separate weighted random-settings module (`packages/core/src/random-settings.ts`, 1,058 lines).
- **Conflict validation** is code: `cond` functions and checks in logic passes, not declarative constraints.

## 8. Seeds, spoilers, hints
- **PRNG.** A small xorshift-family generator with a counter, seeded from SHA-512 of a key built from the seed text, its length and whether a spoiler is generated (`packages/core/src/random.ts`, `pipeline.ts`). Spoiler on and off give different seeds for the same text, which keeps race seeds from being derived from a public spoiler.
- **Not from the seed.** In create mode, "random" starting age and clock values are resolved from a fresh crypto source before logic (`randomVariants` in `generator.ts`), so the same seed text may not reproduce those two choices (inferred from reading; not tested).
- **Seed hash.** 8 characters drawn from the seeded stream (`logic/src/misc/hash.ts`).
- **Spoiler** sections (counted from `generator/lib/combo/spoiler.ts`): settings, tricks, glitches, special conditions, starting items, junk locations, plando, world flags, song events, entrances, hints, spheres, paths, locations.
- **Hints.** Gossip stones per game in CSV (`data/gossips/*.csv`), hint types way of the hero, always, sometimes, playthrough, specific item, foolish, junk (`settings/hints.ts`), configured as an ordered list of type, amount and duplicates. Path hints say which goal a region is on; foolish analysis can be probabilistic (`analysis/analysis-foolish.ts`). Always and sometimes lists grow with shuffle settings in code (`hints.ts`).

## 9. Game integration
- **Generation time.** TypeScript decompresses the two ROMs, injects custom objects, writes the item table, flags and config into a patch file (`application/vnd.ootmm.patch+zip`, `.ootmm`), then packs a combined `.z64` (`generator.ts`, `lib/combo/patch-build/`).
- **Runtime.** A C payload (`packages/generator/src/common/`, `oot/`, `mm/`) replaces item handling, text, shops, entrances, multiworld and the switch between games.
- **Cosmetics** are separate from logic (`packages/core/src/cosmetics/`, `lib/combo/cosmetics/`), with their own log file.
- **Player steps (estimated from the generator flow):** supply two ROMs in the right byte order, choose settings on the web, generate, download the ROM (single world) or a patch file (multiworld, applied in patch mode), load in an emulator. About 4 to 5 steps.

## 10. Multiworld and co-op
- **Modes.** Single, co-op (all items shared) and multiworld, with teams (`mode`, `players`, `teams` settings). Items can target `all` players (`pathfind.ts`).
- **Protocol** (`doc/Multi-Packets.txt`, `packages/generator/src/common/multi/multi.h`): binary packed messages, each with a sequence number and op code. Ops: hello (magic, session id and secret, player id and name, world id), write-ahead-log entry, log query, log ack, player position, item info, entrance info. Outgoing log entries carry a token and wait for an ack; incoming entries carry an index, the sender and the item or event. The hello reply returns game and network sequence numbers for resync.
- **Session secrets** are random per team and patched into the ROM (`MULTI_SESSION_ID`, `MULTI_SESSION_SECRET` in `generator.ts`).
- **Transport.** The game exchanges bytes through an IPC layer with a Project64 back end and a null back end (`common/ipc/`). The host-side relay server was not found in this repository (not verified where it lives).

## 11. Tooling and testing
- **Tests.** 34 `*.test.ts` files (counted), all in `packages/logic/tests`: 8 expression tests (parser, and-or, time, flags, memo, restrictions), 1 pathfinder test, 25 whole-seed tests over setting combinations (default, allsanity, several entrance shuffle mixes, MQ, Japanese layouts, no logic, shared items). Seed tests build a seed through `solvedWorldState` with merged settings (`tests/helpers.ts`).
- **Benchmarks.** `packages/logic/bench/` (compare, configs, run) and `packages/generator/scripts/pathfind-bench.ts`.
- **CI.** `pnpm test` and an Nx build in a pinned toolchain container, with dev, preview and stable deploy branches (`ci.yml`).
- **Logic editor or tracker** in this repo: none found.

## 12. Developer ergonomics
- **Add a location:** add the area entry with its rule in `data/world/...yml`, the check with its flag and vanilla item in `data/checks/...xml`, and runtime support in C if it is a new check kind. Three files in two formats plus code; names must match exactly across them.
- **Add an item:** TypeScript item definition and groups, get-item table entry in `data/defs/`, drawing data, C handling.
- **Docs.** `doc/` holds 4 short text files (44 lines total, counted); most knowledge lives in code and the changelog.
- **Contributors.** 33 (counted).

## 13. Patterns worth standardizing, and pain points
**Worth standardizing**
1. Named logic passes over an accumulating state (`pipeline.ts`), each pass testable alone.
2. Parse-time folding of settings and tricks so runtime rules stay monotone; `!` and `cond` only over options.
3. Explicit parentheses required when mixing and and or.
4. Reachability with extra dimensions (age, time slices) as bit sets carried per area, instead of duplicating regions.
5. Counted-goal conditions over a chosen set of groups, used for many goals at once.
6. Settings string as a compressed diff from defaults with a version prefix.
7. Entrance pools by type with a `reverse` field for coupling, validated by pathfinding with assumed openings.
8. A write-ahead log with acks and resync sequence numbers for item delivery.

**Pain points**
1. Location names are the join key between YAML logic, XML checks and code; a rename touches several formats.
2. Numeric item ids come from table order, so reordering changes ids (a ledger would prevent this).
3. A large amount of world shaping lives in code (`world/transform.ts`, 2,338 lines), outside the data.
4. Hint pools and some setting effects are code branches, not data.
5. Two "random" settings bypass the seed stream, which limits reproducibility.

## 14. Mapping to our standard
**Maps to doc 07 and doc 01 D2.5 as they exist**
- `has(x, n)` to `has` with `n`; `masks(n)` and the custom goals to `group_count` with `ge`; `event` to `reach_event`; `setting` and `trick` to `opt` (tricks as an `item_set` style option and `in`); `&&`, `||`, `true`, `false` to `all`, `any`, constants; macros to `defs` (OoTMM macros are parameterized, which D2.5 allows as build-time substitution).
- Areas, exits and locations to `regions`, `connections`, `locations`; area events to `events`; `logic` levels to options plus the generator's beatability mode; plando and starting items to `place`; junk locations to location `kind: excluded`; entrance pools with `reverse` to `rewire` with coupling; trap cloaking and item pools to `weighted_filler` and `classification_when`.
- Settings `cond` to option `depends`; presets to `presets`; settings string to D3.1.
- The write-ahead log with acks fits the ordered event log in 01 D1.4.

**Needs a new primitive or concept**
- **`state_dimension` (concept).** A small, finite set of player forms (here child and adult) under which reachability is computed separately, with declared switch points (areas that allow the change, and the item or event that enables it). Rules read the current form. Evidence: `pathfind.ts` age states, `age_change` in area YAML.
- **`time_window` (primitive).** A declared finite timeline (slices), per-region flags for whether time flows and where waiting is blocked, and rule results that are "true within these slices". The reachability engine carries slice sets per region. Evidence: `expr/data.ts`, `stay` in `data/world/mm/overworld/clock_town.yml`, `tests/expr/time.test.ts`.
- **`source_class` (concept).** Counting an item separately by where it was found (repeatable source, one-time source), so rules can ask for a renewable copy. Evidence: `locations.ts`, `renewable` and `license` in `expr/compile.ts`.
- **`shared_item` (concept).** One logical item that satisfies two games' rules in a combined world, enabled by option. Evidence: `items/shared.ts`. This also matters for any multi-game bundle we host.
- **`goal_threshold` (recipe, not primitive).** A count over an option-chosen set of groups compared to an option value; likely composable from `group_count`, `int` and `opt`, so a published recipe first.

**Cannot be data, and why**
- The C payload, ROM decompression, object injection and patch packing: these are game-specific binary work, which in our design is the optional patch layer (01 D4), not world data.
- `world/transform.ts`: setting-driven graph rewrites (removing or merging areas, MQ swaps). Some could become declared variants selected by options (table-driven `select` over area sets), but the volume and variety suggest a residue for the reserved module hooks (01 D2.8, `adjust_world`).
- Price randomization and song events are seeded table rolls that feed logic values; they fit `select` or `permute` in shape but need typed integer outputs readable by rules (`price(...)`), which our language would expose as `counter` or table lookups (needs checking against D2.5).
