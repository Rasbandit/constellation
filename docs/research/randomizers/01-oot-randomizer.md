# 01. OoT Randomizer (OoTR): architecture and patterns audit

> Internal research note, 2026-11-16, read at commit `526b4c8d81b7c4023ef4573ffbaf39344e270d5b` of the `Dev` branch of [OoTRandomizer/OoT-Randomizer](https://github.com/OoTRandomizer/OoT-Randomizer) (shallow sparse clone; no code was run). Not reviewed by a different agent or a human. Our own writing with citations; no code, text or data from the project is reproduced. Numbers are labeled **counted** (by our own script or `grep`/`git ls-tree` over the clone, or the GitHub API on 2026-11-16), **estimated**, or **not verified**. Paths are relative to the repo root.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. Summary
- **What it is.** The community-standard item and entrance randomizer for The Legend of Zelda: Ocarina of Time (N64). A Python generator reads region data, places items and shuffles entrances under logic, writes hints, then patches the player's own ROM.
- **Popularity.** 443 stars, 136 contributors including anonymous (both counted via the GitHub API, 2026-11-16; 00-landscape row 3). Its logic lineage feeds OoT3D Randomizer and the Ship of Harkinian randomizer (00-landscape section 3.1). Archipelago has an in-tree `oot` world (00-landscape).
- **License.** MIT at the top level with GPLv2+ subfolders (00-landscape, LICENSE opened in that step; not reopened here). For us: read and describe only; if a future decision ever reuses any data, the MIT notice must travel with it and the GPL folders are off limits.
- **Maintenance.** Active: last push 2026-11-16 (counted via the API); CI on every push and PR to `Dev` (`.github/workflows/python.yml`). `CONTRIBUTING.md` describes a labeled review and testing process, a release cycle with feature freezes, and dev team calls that accept or reject enhancements.

## 2. Architecture
- **Language and layout.** Python generator as flat top-level modules: 50,849 lines across top-level `.py` files (counted, `wc -l`). Largest are `SettingsList.py` (5,864), `SettingsListTricks.py` (3,342), `Patches.py` (2,834), `LocationList.py` (2,662), `Scene.py` (2,540), `Hints.py` (2,330) (counted).
- **Game-side code.** `ASM/` holds C and MIPS assembly built into the base patch: 132 entries under `ASM/c`, 118 under `ASM/src` (counted, `git ls-tree`), with a `Makefile` and linker script. Not opened in depth.
- **Pipeline** (`Main.py:39-211`): resolve settings (load plando file, mark tricks as booleans, seed the RNG, roll random settings) then, inside a retry loop of up to 10 attempts (`Main.py:39-58`), `generate`: build one `World` per player (load `Overworld.json` and `Bosses.json`, then dungeons as vanilla or Master Quest), compile rules (`set_rules`), build the item pool (`ItemPool.py`), shuffle entrances (`set_entrances`), fill (`Fill.py`), then build the spoiler, playthrough, hints, file hash and optional password. Finally `patch_and_output` writes the requested outputs.
- **How users run it.** The README recommends the hosted web generator at ootrandomizer.com (not opened). Locally there is a CLI (`OoTRandomizer.py`, argparse in `Settings.py:407` onward, `--seed`, settings string) and a desktop GUI (`Gui.py` launches an Angular app in `GUI/src/app`, Electron plus Node.js per the README). The same Angular code serves the web generator: `data/settings_mapping.json` marks sections `exclude_from_web` or `exclude_from_electron`.

## 3. World model
- **Regions** are JSON arrays of objects in `data/World/*.json` (26 files, counted), one file per dungeon variant plus `Overworld.json` and `Bosses.json`. Each region has a name, an optional dungeon tag, and three maps from name to rule string: `locations`, `exits`, `events` (read in `data/World/Deku Tree.json`). Counted with our script over `data/World`: 486 regions, 2,355 location entries (2,219 distinct names, since vanilla and Master Quest files share names), 903 exits, 68 event entries. `data/Glitched World` mirrors the set (26 files; 602 regions, 1,142 exits, 81 events, counted).
- **The files are not strict JSON.** They contain `#` comments and multi-line strings, read through a custom `read_logic_file` (`Utils.py`, referenced from `RuleParser.py:13`). Our counting script had to strip both first.
- **Locations** are a separate Python table, `LocationList.py`: about 2,190 tuple entries (counted by a line-pattern `grep`, so approximate), each carrying a type (Chest, Song, Shop, GS Token, and so on), scene id, flag, vanilla item and group tags such as dungeon name and "Chests" (`LocationList.py:996`). Logic and game addresses therefore live in two places joined by name.
- **Items** are in `ItemList.py`: about 285 entries (counted by pattern), each with a type, an advancement flag, a game item id from an `IntEnum`, and a details dict; progressive items declare a chain length (`ItemList.py:421`).
- **Events** are implicit: any capitalized identifier in a rule that is not an item or helper becomes a logic-only event item (`RuleParser.py:100-110`), and regions grant them through their `events` map.
- **Age and time of day** are first-class context: every compiled rule takes `age`, `spot` and `tod` keyword arguments (`RuleParser.py:23-31`), and the search tracks reachability per age.

## 4. Logic
- **Representation.** Rule strings in Python expression syntax inside the JSON (`and`, `or`, `not`, comparisons, `(item, count)` tuples, calls). `RuleParser.py` parses each string with Python's `ast` module, rewrites names (items become `state.has` calls, settings become constants, helpers are inlined), folds constants, groups plain item tests into `has_any_of` and `has_all_of`, then compiles the tree into a cached lambda (`RuleParser.py:286-431`). Rules that fold to `True` or `False` mark the spot as always or never reachable (`RuleParser.py:482-490`).
- **Helpers and macros.** `data/LogicHelpers.json` holds 156 named helpers (counted), some parameterized (for example a `can_use(item)` style helper), inlined by textual argument substitution at parse time (`RuleParser.py:41-52`, `163-184`). Aliases also express progressive levels: a named upgrade is defined as holding N copies of the chain item.
- **Local subrules.** `here(rule)` and `at(region, rule)` lift a sub-expression into an anonymous event location in the current or a named region, so it is evaluated once per region and with that region's age and time context (`RuleParser.py:360-447`). Time-of-day helpers (`at_day`, `at_night`, `at_dampe_time`) expand to a check that can call back into the search (`RuleParser.py:452-474`).
- **Tricks and glitches.** `SettingsListTricks.py` lists 187 glitchless tricks and 186 advanced tricks (counted by `name` keys), each with a setting name, tags for GUI filtering, and a tooltip. Each trick becomes a boolean setting that rule strings reference directly (`Main.py:78-89`).
- **Difficulty levels.** `logic_rules` is `glitchless`, `advanced` (switches the data directory to `Glitched World` and enables the advanced trick list) or `none` (`SettingsList.py:650-683`; `Main.py:151-155`).
- **Validation.** The parser raises on unknown names, wrong arity and bad tuples (`RuleParser.py:111-112`, `135-146`, `171-173`). Semantic validity is checked by generating seeds in tests (section 11); there is no static reachability checker over the data.
- **Example in our words.** The Deku Tree basement ledge is reachable if dungeon shortcuts are on, or if, evaluated in the current room, the player is adult, or if a specific skip trick is enabled (`data/World/Deku Tree.json`, region "Deku Tree Basement", exit to "Basement Ledge").

## 5. Fill and placement
- **Algorithm.** Assumed fill (`Fill.py:391-516`): assume every unplaced item is held, pop one item, remove it from the assumed state, compute maximum reachability, and place it in the first shuffled location that accepts it and is reachable. Failure raises and the whole generation retries (`Main.py:48-58`).
- **Staged order** (`Fill.py:30-231`): shops first, then dungeon-restricted items (boss keys before small keys, `Fill.py:261-262`), then empty-dungeon junk, then songs on song locations (own-world only, with 15 local retries, `Fill.py:348-388`), then optional one major item per dungeon (`Fill.py:271-344`), then all progression with reachability checks, then "priority" items with only item-rule checks, then filler with no checks. It ends with an explicit beatability check (`Fill.py:230-231`).
- **Beatable-only mode.** When the setting allows unreachable locations, the reachability check is skipped for items the goal does not need (`Fill.py:439-450`).
- **Constraints.** Per-location item rules compose as closures (`Rules.py:107-119`; Ganon's location only accepts the Triforce, `Rules.py:22`). Other flags: `minor_only` locations, and `disabled` locations that are filled only if the seed stays beatable (`Fill.py:482-485`).
- **Progression balancing.** No separate balancing pass. In multiworld, an item placed in another player's world must also be reachable at the same named location in its owner's world, so players are not blocked behind each other (`Fill.py:457-480`).
- **Item pools.** `ItemPool.py` (1,137 lines, counted) builds pools per settings (plentiful, ludicrous and so on). Ice traps are disguised as other items with a configurable appearance pool (`Fill.py:88-119`).
- **Plando.** `Plandomizer.py` (1,519 lines, counted) reads a distribution JSON that can pin settings, items per location, entrances, starting items, hint distribution, gossip text, file hash and password. It hooks into fill (`Fill.py:85`) and can cloak models (`Fill.py:233`). The wiki page linked from the README was not opened.
- **Starting inventory** is the `starting_items` dict setting (`SettingsList.py:3301`), with effective starting items computed after fill (`Main.py:132-133`).

## 6. Entrance randomization
- **Method.** `EntranceShuffle.py` (1,166 lines, counted). A table of 159 shufflable entrance rows (counted by pattern, `EntranceShuffle.py:89` onward), each typed (Dungeon, Interior, Overworld, Grotto, boss types, one-way types such as owl drops, spawns, warp songs) with the game's entrance index. Pools are built from settings (`EntranceShuffle.py:443-560`). Entrances are disconnected, then reconnected one by one to random targets of a compatible type.
- **Coupling.** Two-way entrances are placed as coupled pairs. A `decouple_entrances` check is left commented out (`EntranceShuffle.py:704`), so we found no decoupled mode on this branch. Boss rooms can mix child and adult pools (`full` versus `limited`, `EntranceShuffle.py:545-560`).
- **Validation.** After each placement, `validate_world` (`EntranceShuffle.py:956`) checks that every location reachable before shuffling is still reachable, plus age and time-of-day access constraints. Entrances are split into restrictive and soft by whether they are reachable as both ages at all times with all assumed entrances disconnected; restrictive ones are placed first (`EntranceShuffle.py:776-847`). Each pool has 20 retries with rollback (`EntranceShuffle.py:771-812`).
- **Priority one-way placements.** Some warp-song targets are pinned so key areas stay reachable; this logic is long hand-written conditionals over settings and enabled glitches (`EntranceShuffle.py:487-543`, `872-898`).

## 7. Options model
- **Count.** 284 setting declarations in `SettingsList.py` (counted): 107 Combobox, 86 Checkbutton, 29 Scale, 16 MultipleSelect, 7 Textinput, 6 SearchBox, plus file, directory, button and internal types. 163 are marked `shared` (counted, `shared = True`), meaning they change the seed and are encoded in the settings string. The others are local (output paths, cosmetics).
- **Declaration.** Each setting is a Python object with GUI text, default, choices, tooltip, `disable` map and GUI parameters. The `disable` map says which other settings to disable for a given value (`SettingsList.py:674-681`); this is the conflict and dependency model.
- **Random settings.** 75 settings carry a `distribution` of weighted choices (counted) used when a "randomize" key is on (`Settings.py:326-366`).
- **Presets.** `data/presets_default.json` holds 12 named presets (counted), mostly community events and tournaments. `CI.py` checks their formatting and settings (`CI.py:50-140`).
- **Settings string.** Shared settings are packed into a bit stream in declaration order, each at a fixed bit width, then mapped to text (`Settings.py:42-64`, `132-193`). Lists encode as indexes with a terminator and invert when more than half is selected. The string depends on setting order and version.
- **UI generation.** `SettingsToJson.py` turns the Python settings plus the tab and section layout in `data/settings_mapping.json` into JSON the Angular GUI consumes (`GUI/src/app/providers/GUIGlobal.ts:660`). The UI is generated from one source for desktop and web.

## 8. Seeds, spoilers and hints
- **Seed.** A user seed is sanitized to `[A-Za-z0-9_-]` or randomly generated as 10 characters. The numeric seed is SHA-256 over settings string, plando distribution, version and seed (`Settings.py:253-270`), which seeds Python's global `random` (`Main.py:123`). Per-world patch randomness is derived from that state (`Main.py:230-237`). Reproducibility is therefore tied to version and Python's PRNG.
- **Seed identity.** Five hash icons from a 32-icon set shown in game (`Spoiler.py:82-85`), and an optional password lock of ocarina notes (`Spoiler.py:87-95`).
- **Spoiler.** JSON with settings, placements, entrances, a sphere playthrough and way-of-the-hero data (consumed by `Unittest.py:766` onward).
- **Hints.** Gossip-stone hints driven by hint distribution files in `data/Hints/` (18 counted). Each file gives per hint type an order, weight, fixed count and copies, plus add and remove lists for locations and items and per-category limits (read in `data/Hints/tournament.json`). Hint types include always, goal or path, way of the hero, barren, sometimes, dual, entrance, important check, song and junk (`Hints.py:614-1304`). Allocation fills fixed counts first, then draws weighted types (`Hints.py:1614-1910`). `Goals.py` defines goal categories for path hints. Misc hints (for example a named item hinted by an in-game text source) are configured per distribution.

## 9. Game integration
- **Generation time.** All randomization happens in Python. `Patches.py` writes the item override table (`Patches.py:2185-2250`), scene and actor edits, text and models into a copy of the player's ROM. The C and ASM payload from `ASM/` adds runtime behavior such as item overrides, multiworld hooks and menus (inferred from the context notes below).
- **Outputs.** A `.zpf` patch in the project's own XOR-block format keyed on the original ROM (`N64Patch.py`), a `.zpfz` zip for multiworld (`Main.py:413-474`), compressed or uncompressed ROM, or a Wii Virtual Console WAD (`Main.py:286`). `from_patch_file` rebuilds a ROM from a patch (`Main.py:513`).
- **Cosmetics** are a separate pass that can be rerun on an existing patch (`Main.py:597`, `Cosmetics.py`).
- **Player steps** (estimated, from the README): pick settings on the website, generate, supply your own ROM to patch (in browser or locally), then load the result in one of four recommended emulators, on an EverDrive or on Wii VC. About 4 steps, plus one-time emulator setup per the wiki.

## 10. Multiworld and co-op
- **Generation.** Built in: `world_count` up to 255 (`Main.py:103-106`); all worlds share one fill with cross-world reachability (section 5).
- **Runtime.** A versioned memory layout, the "co-op context", at a fixed address, documented in `Notes/coop-ctx.md` (version 7). It has fields for incoming item and sender, outgoing item, location key and recipient, player names, and a flag for progressive-item state. Emulator plugins poll and write these fields. The README names BizHawk as the only race-legal emulator with multiworld. The network side lives outside this repo (00-landscape names midoshouse/ootr-multiworld; not opened).

## 11. Tooling and testing
- **Tests.** `Unittest.py` has 35 test methods (counted) across classes for plando, hints, entrance randomizer, valid spoilers, text, scene flags, audio, language files and scene files. `tests/` holds 278 files, 258 of them under `tests/plando/` (both counted with `git ls-tree -r` at commit `526b4c8d`, 2026-11-16; matches 00-landscape), including 18 `.sav` settings files (counted) used as generation cases, and plando files such as boss-rush variants. `TestValidSpoilers` checks way-of-the-hero, playthrough and disabled locations in generated spoilers. A fuzzer generates 10 seeds with random settings and saves the failing settings file (`Unittest.py:975-1006`). Further tests cover presets and advanced tricks with entrances (`Unittest.py:913-975`).
- **CI.** `.github/workflows/python.yml` runs `CI.py` unit tests on Python 3.13 and latest 3.x, plus other checks. `CI.py` checks preset formatting, a hell-mode preset that enables all tricks, preset settings validity, duplicate message ids, code style and table sizes (`CI.py:50-251`). `build.yml` was not read.
- **Tracker support.** `Notes/auto-tracker-ctx.md` documents a versioned memory layout (version 6) so auto-trackers can read dungeon rewards, MQ status and similar without per-version symbol files.
- **Debug and docs.** `Notes/` holds ROM modding guides (ASM, scene patching, save data, SRAM). There is no logic editor; rule strings are edited by hand.

## 12. Developer ergonomics
- **Adding a location** touches at least three places: a region entry with a rule in `data/World/*.json` (and `Glitched World` if relevant), a row in `LocationList.py` with scene and flag, and possibly hint text in `HintList.py` and item rules in `Rules.py`. Inferred from the data split; no contributor guide for this was found in the files read.
- **Adding an item:** `ItemList.py`, pool logic in `ItemPool.py`, and usually game-side C.
- **Adding a trick:** one entry in `SettingsListTricks.py` and references in rule strings.
- **Docs.** `CONTRIBUTING.md` (process, labels), `Notes/code-style.md`, the wiki (not opened). Contributors: 136 (counted, API).

## 13. Patterns worth standardizing, and pain points
**Worth standardizing**
1. Region files that list locations, exits and events with rule strings, plus a separate helper file with parameterized, inlined macros.
2. Compile-time partial evaluation: settings and tricks fold to constants, and rules that become `True` or `False` mark locations always or never reachable.
3. Context-parameterized reachability (age, time of day) as a dimension of the search instead of duplicated regions.
4. Tricks as a catalog of named, tagged, documented booleans that rules reference; difficulty presets are sets of tricks.
5. Staged fill with an explicit order for restricted pools, then assumed fill, then a final beatability check.
6. Data-driven hint distributions: per-type fixed counts, weights, copies, and add or remove lists.
7. Versioned memory ABIs for multiworld and auto-trackers (`Notes/coop-ctx.md`, `Notes/auto-tracker-ctx.md`), so external tools need no per-build symbols.
8. A settings fuzzer that saves the failing settings for replay.

**Pain points (design costs, in our words)**
1. Rule strings are Python syntax compiled with Python's `ast` and `eval`, so another-language tool must reimplement Python's expression semantics plus the name rewriting rules.
2. Non-standard JSON (comments, multi-line strings) needs a custom reader.
3. Glitched logic is a parallel copy of all 26 region files, so fixes must be made twice.
4. Location metadata and logic are joined by name across JSON and a Python table.
5. Entrance priority rules are long conditionals over settings in code.
6. Whole-generation retry (up to 10 attempts) hides how often fills fail.
7. Determinism depends on the global Python PRNG and the version string.
8. The settings string depends on declaration order and version.

## 14. Mapping to our standard
**Maps to existing doc 07 primitives and doc 01 concepts**
- Region files map to `regions`, `connections` and `locations` (01 D2.2). Logic-only events map to `events`. `LogicHelpers.json` maps to `defs` with parameterized macros (01 D2.5; 07 2.2).
- Item counts and tuples map to `has` with `n`. Progressive chains map to `progressive` and `level` (07 2.3; `ItemList.py:421`, `LogicHelpers.json` aliases).
- Tricks and logic level map to `opt` over toggles, and presets map to `presets` (01 D2.3).
- The settings `disable` map maps to option `depends`, `requires` and `conflicts` (07 2.3; `SettingsList.py:674`). Random settings with weights map to the settings-layer `random` weights (01 D2.3; `Settings.py:349-352`).
- Item rules map to `item_rule` (`Rules.py:107-119`). Plando item pins map to `place`. Ice trap and junk pools map to `weighted_filler`. Entrance pools with coupling map to `rewire` (`EntranceShuffle.py`).
- Location kinds: `excluded` and `priority` match 01 D2.7. Beatable-only mode matches our accessibility modes. Settings string and seed hash map to 01 D3.1 and D3.2.

**Needs a new primitive or concept**
- **`context` dimensions** (candidate): a world declares finite reachability contexts (for OoT: age, with two values, and time of day as a bitmask). Rules may test the current context, and `reach` takes a context. Evidence: `RuleParser.py:23-31`, `Search.py:131-171`. Without it, authors duplicate regions per age.
- **`here` and `at` local events** (candidate, possibly a compile step rather than a primitive): lift a sub-rule into an anonymous event at a region so it inherits that region's context. Evidence: `RuleParser.py:360-447`.
- **`fill_stages`** (candidate): an ordered list of (item filter, location filter, check level: full, item-rule-only, none, own-world, retries) that the platform runs before general fill. Evidence: `Fill.py:30-231`.
- **`group_quota`** (candidate): at least or at most N items of a class per location group, such as one major item per dungeon. Evidence: `Fill.py:271-344`.
- **`hint_plan`** (candidate): a weighted hint-type allocator with fixed counts, copies and per-area limits, over platform-defined hint kinds (way of the hero, barren, path to goal, item at location). Evidence: `data/Hints/*.json`, `Hints.py:1614-1910`.
- **`disguise`** (candidate): a trap item takes the appearance of an item drawn from a filtered pool. Evidence: `Fill.py:88-119`.
- **Rewire extensions:** typed entrance pools, mixed pools, a restrictive-first placement order, and pinned "priority targets" expressed as data instead of code. Evidence: `EntranceShuffle.py:487-560`, `776-847`.
- **Versioned client memory ABI** as a doc 01 CSDK concept for emulator-based games. Evidence: `Notes/coop-ctx.md`.

**Cannot be data, and why**
- ROM patching, C and ASM payloads, scene and actor rewrites, text, models and cosmetics (`Patches.py`, `ASM/`, `Cosmetics.py`) are game-binary work. They belong in a game-side patcher (01 CPK) fed by slot data, not in the world format.
- Shop price randomization and the post-fill wallet requirement derived from prices (`Fill.py:235-244`) need seeded value generation plus a derived counter. They might fit `select` plus `counter`, but that is not verified.
- Time-of-day checks that call back into the search from inside a rule (`RuleParser.py:452-474`) do not fit a pure rule tree. They need the `context` primitive above to be expressible.
