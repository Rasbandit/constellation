# 06. Hollow Knight: RandomizerCore (library) plus RandomizerMod (game)

> Internal research note, 2026-11-16. Not reviewed by a different agent or a human; nothing here goes outside this repo before review under [`CLAUDE.md`](../../../CLAUDE.md). Our own writing, with citations; no code, logic strings or doc text is reproduced (logic examples below are paraphrased). Numbers are **counted** (our own script or grep over a clone), **reported** (stated by the project), **estimated**, or **not verified**.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

Sources read (full clones, all three are small, under 16 MB each):
- [homothetyhk/RandomizerCore](https://github.com/homothetyhk/RandomizerCore) at `791b025` (2026-01-16): `RandomizerCore/`, `RandomizerCoreTests/`, `docs/articles/*.md` (7 articles, 1,068 lines counted).
- [homothetyhk/RandomizerCore.Json](https://github.com/homothetyhk/RandomizerCore.Json) at `0923baa` (2026-01-11): `Converters/`, test resources.
- [homothetyhk/RandomizerMod](https://github.com/homothetyhk/RandomizerMod) at `cad3d5a` (2026-04-12): `Resources/Logic/*.json`, `Resources/Data/*.json`, `RC/`, `Settings/`, `LOGIC_README.md`, `README.md`, `RandomizerModTests/`.
Not read: ItemChanger and MenuChanger (the runtime and menu mods RandomizerMod depends on, per its README), the separate multiworld mod, and the HK Modding API.

## 1. Summary
- **What it is.** RandomizerCore is a C# library for randomizer logic and fill that grew out of the Hollow Knight randomizer and is written so other games can use it (its README calls it a library for randomizer logic and algorithms). RandomizerCore.Json adds a JSON logic format. RandomizerMod ("Randomizer 4") is the Hollow Knight item and room-transition randomizer built on both.
- **Game.** Hollow Knight (PC, patch 1.5 and later per the README).
- **Popularity.** RandomizerMod 75 stars, 15 contributors; RandomizerCore 8 stars, 4 contributors; RandomizerCore.Json 2 stars, 2 contributors (counted, 00-landscape plus GitHub API on 2026-11-16).
- **License.** LGPL-2.1 for all three (LICENSE headers opened). For us: read and describe. We would not link or ship it, so linking rules do not arise; nothing is copied.
- **Maintenance.** Active but slow: last commits 2026-01 (core) and 2026-04 (mod). A small team; the core is effectively one maintainer's design (inferred from contributor counts, not verified).

## 2. Architecture
- **Language.** C#, .NET. The mod runs inside the game through the Modding API.
- **Core library layout** (`RandomizerCore/`, 15,662 lines of C# counted): `StringParsing/` (tokenizer, expression parser, validator), `StringLogic/` (logic clauses, DNF and RPN builders), `Logic/` (terms, waypoints, macros, `LogicManager`, `LogicManagerBuilder`, `ProgressionManager`, `StateLogic/`), `StringItems/` (item effect strings), `LogicItems/`, `Randomization/` (`Randomizer`, `SphereBuilder`, stages, groups, coupled groups, placement strategies, `Validator`), `Updater/` (`MainUpdater`, which indexes subscribers by the terms they depend on so new progression re-evaluates only affected logic).
- **Pluggable formats.** `ILogicFormat` lets a caller load logic from any format; RandomizerCore.Json is the only public implementation (`docs/articles/logic_model.md`). `LogicFileType` (`Logic/LogicFileType.cs`) names the file kinds: terms, waypoints, transitions, macros, items, locations, logic edits, macro edits, logic substitutions, item templates, state data.
- **Mod pipeline** (`RandomizerMod/RC/RandoController.cs`): fire `OnBeginRun`; a `RequestBuilder` assembles stages, groups, items and locations from settings (`RC/Requests/`); fire `OnRequestBuilt`; run `Randomizer` with a seeded `System.Random`; collect placements; write spoiler and helper logs; compute a hash.
- **How users run it.** Install the Modding API, RandomizerMod, ItemChanger, MenuChanger, RandomizerCore and RandomizerCore.Json (README dependencies), start a new save, pick settings in the in-game menu; **generation runs inside the game** when the save is created. Steps: about 4 after the mods are installed (estimated).

## 3. World model
- **Terms** are global named integers (signed byte or 32-bit) or **state-valued** terms (`docs/articles/logic_model.md`). An integer term outside a comparison reads as "greater than zero".
- **Waypoints** are terms with logic: when the logic holds, the term is granted (a cached, reusable reachability fact). **Macros** are textual substitutions. **Locations** have logic and receive an item. **Transitions** are both a location and an item with the same name: reaching one grants it, and placing a transition at a source transmits state.
- **Items** modify terms through an item-effect grammar (section 4).
- **RandomizerMod data** (`Resources/Logic/`, counted by our script after stripping `//` comments, which the JSON files contain): 249 items, 605 locations, 892 transitions, 302 waypoints, 162 macros, 148 terms (132 signed byte plus 16 int), state fields in `state.json`. `Resources/Data/` holds game metadata: 606 location records, 892 transition records (scene name, gate name, direction, one-way type), 364 rooms, 29 start locations, 36 pools, 45 cost entries. Total 31,281 JSON lines across both folders (counted with `wc -l`).
- **Shop and currency costs** are separate data (`Data/costs.json`, 45 entries) and code (`RC/LogicGeoCost.cs`, `RC/CharmNotchCosts.cs`); the logic readme explains that geo (currency) is not counted exactly, and expensive locations instead require access to farming spots.
- **Item templates and placeholders.** `LogicFileType` includes item templates; the mod has placeholder and split items (`RC/PlaceholderItem.cs`, `RC/SplitCloakItem.cs`).
- **Rooms** are modeled by transition names of the form scene plus gate (for example a scene name with a bracketed gate such as "left1"), so the room graph is implicit in transition and waypoint logic.

## 4. Logic
- **Grammar** (`docs/articles/logic_model.md`, `StringParsing/`): infix strings with `+` (and), `|` (or), `>` `<` `=` (term comparison against an integer or another term), `?` (coalescing: use the left term if defined, else the right; for third-party edits), `*` (copy another def's logic), `/` (project a state term to a boolean), parentheses. Precedence lets authors write alternatives without parentheses. The docs warn that `<` and `=` are unsafe because they can make a reachable location unreachable later. Reserved and safe characters for names are listed in `docs/articles/safe_naming.md`.
- **Item effect strings** (`docs/articles/item_strings.md`): add (`+=`, `++`), raise-to-max, conditional effect guarded by a logic test, sequential chaining, and "try this else that" chaining, plus negation, references and coalescing. RandomizerMod uses this for split items: for example, the two halves of the cloak each set a left or right dash term depending on which half the player already has (`Resources/Logic/items.json`, paraphrased).
- **State logic** (`docs/articles/state.md`, `state_adv.md`): a state is a vector of bools and ints with a partial order; a location yields a set of minimal states (a "state union"). Logic is converted to DNF and the state modifiers in each clause run left to right; dominated states are pruned. RandomizerMod uses this for soul (spell resource), charm notches, health, the shade, and once-per-path flags; 22 state variable classes in `RC/StateVariables/` (counted) implement modifiers like spending soul or taking damage, written in C#.
- **Skip levels.** Not a numeric tier: 21 independent skip settings (`Data/logic_settings.json`, counted), each a token in logic (precise movement, enemy pogos, shade skips, fireball skips, damage boosts and so on), defined with examples in `LOGIC_README.md`. Macros combine them into tiers such as mild and spicy combat skips.
- **Terminal versus nonterminal logic.** The mod's logic readme distinguishes logic for things with permanent effect (locations, world events) from transitional logic (waypoints, transitions); state logic was added so nonterminal logic can spend resources consistently.
- **Compiled forms.** Logic strings compile to reverse-polish token arrays (`Logic/RPNLogicDef.cs`) or, when state is involved, to DNF clauses (`Logic/DNFLogicDef.cs`, `Logic/DNFConverter.cs`). The `LogicManager` can also be used for static analysis of a location's logic or an item's effects outside a fill (`logic_model.md`).
- **Validation.** Cycle detection (`Logic/CycleDetector.cs`), expression validation, a commutativity check at fill time (section 5), and test-time `LogicFixture` tests in the mod.
- **Example in our words.** A charm shop location needs the player to reach the shop's door term and, for one variant of that location, also to hold the shop key term; a waypoint for the shop room is reachable if either door of the room is reachable. So locations read "room access plus extra items", and room access is factored into waypoints.

## 5. Fill and placement
- **Algorithm: forward fill in progression steps** (`docs/articles/forward_fill.md`, `Randomization/SphereBuilder.cs`). Items and locations are permuted and given priorities in [0, 1). Each step adds items in priority order until a new location unlocks, then walks back and drops items not needed, giving a minimal set, then places that set at already reachable empty locations. Once everything is reachable, the rest is placed freely. The docs prove which minimal set is chosen (the colexicographic minimum by priority), so priority order directly controls intended progression.
- **Stages and groups** (`docs/articles/randomizer.md`): a stage runs the fill over its groups together; a group pairs only its own items with its own locations (for example left-to-right transitions as one group). Stages run in two passes, ascending then descending, so items can be placed assuming later stages succeed and then re-placed against the real result.
- **Coupled groups** (`Randomization/CoupledRandomizationGroup.cs`): a group whose items are the locations of a dual group, used for two-way transitions.
- **Constraints and bias.** Placement strategies with hard or soft constraints (hard ones restart the attempt), an `OnPermute` hook to move an item's priority (for example force it late), and a **depth priority transform** (`PriorityTransformUtil`) that pulls items toward locations unlocked later, with linear, quadratic, square root and log shapes and several interaction modes (README "Progression Depth Settings").
- **Failure handling.** Out-of-locations and unreachable-location errors start a new attempt; a **commutativity failure** (adding the same items in different orders gives different reachability) halts with an error, because the algorithm assumes order independence. A `Validator` per group checks the final result.
- **Monitoring.** A `RandoMonitor` broadcasts attempts, stage progress and the error that ended each failed attempt; the docs note that large attempt counts point to logic errors or over-tight constraints (`randomizer.md`). The mod logs profiling results for successful and all attempts (`RandoController.cs`).
- **Pool, starts, plando.** Pools are toggled by settings (`Data/pools.json`, 36 pools); duplicates of key items are added as items invisible to logic; start items and 29 start locations are settings. No plando format in the mod was found (not verified across connection mods).

## 6. Entrance, door or area randomization
First class. Modes: map area, full area, or room transition randomization, with options for area constraint, direction matching, no door-to-door, and coupled or decoupled transitions (`README.md` transition settings). Transitions are items in transition groups; one-way transitions map only to one-way ones. Because a transition's state depends on where it is placed, transition randomization uses location-dependent effects and requires final validation.

## 7. Options model
- **Settings** are C# classes (15 `*Settings.cs` files) with about 159 public fields (counted by regex, includes container fields; approximate), grouped into pool, skip, novelty, cost, long location, start location, start item, misc, cursed, transition, progression depth, duplicate item and split group settings.
- **Presets:** 14 preset data classes (`Settings/Presets/`, counted), one per settings group, so a user mixes presets per group.
- **Settings code.** `BinaryFormatting` serializes settings by reflection into a shareable code; the README says the code is tied to the mod version and covers only base settings, not connection mod settings.
- **UI** is the in-game MenuChanger menu, built in code (not read).
- **Conflicts** are resolved in code during request building.

## 8. Seeds, reproducibility, spoiler logs and hints
- Seed is an integer in `GenerationSettings`; the randomizer uses `System.Random(seed)` and the controller uses a second stream offset from the seed (`RC/RandoController.cs`). Same mod version and same installed connection mods are needed to reproduce (inferred from the hash hook below).
- **Hash.** A SHA-based hash of settings and placements, and an `OnCalculateHash` event so connection mods add their part (`RC/RandoController.cs`).
- **Logs** (README tips): tracker log (what the player has found), helper log (what is currently in logic, marking out-of-logic sequence breaks), spoiler log. Hints are mostly runtime features (not read).

## 9. Game integration
Runtime PC mod: generation in the mod, then ItemChanger rewrites the game's item givers and transitions at runtime. Some logic assumptions require helper mods (Benchwarp, QoL) per the README. No ROM or file patch. The generator is not usable outside the game without its host (inferred: the controller depends on mod settings and menus).

## 10. Multiworld or co-op
Not in these repos. A separate mod ([Shadudev/HollowKnight.MultiWorld](https://github.com/Shadudev/HollowKnight.MultiWorld), 00-landscape) adds it; Archipelago has an in-tree `hk` world. Not read.

## 11. Tooling and testing
- **Core tests:** 12 test files in `RandomizerCoreTests/` with 39 test attributes counted by grep (tokenizer, parsers, end-to-end logic defs and item strings, randomization).
- **JSON tests:** dated serialized logic snapshots (`RandomizerCore.JsonTests/Resources/2022-03-18.json` through `2024-01-20-stable472.json`, 8 dated files) used as regression inputs.
- **Mod tests:** 14 files, 75 test attributes counted by grep, mostly per state variable (soul, damage, charms, shade skips).
- **CI:** only a GitHub Pages docs build in each repo (`.github/workflows/github-pages.yml`); no automated test workflow found.
- **Docs:** a docfx site generated from XML doc comments plus 7 articles (core), and a 171-line logic readme (mod).
- **Trackers and editors.** No logic editor. Logic is hand-edited JSON. The helper log acts as a built-in tracker.

## 12. Developer ergonomics
- **Adding a game** with the core: define terms, waypoints, macros, items, locations and transitions (in JSON via RandomizerCore.Json or in code), build a `LogicManager`, build stages and groups, run `Randomizer`. The core does not ship a game template, an option model or an output format; those belong to the host.
- **Adding content to HK**: "connection mods" subscribe to `RequestBuilder.OnUpdate` (a priority-ordered event, `RC/Requests/RequestBuilder.cs`) to add items, locations, groups and costs, and load logic edit, macro edit and substitution files to patch base logic. The `?` coalescing operator lets edits refer to terms that may not exist when another mod is absent.
- 15 plus 4 plus 2 contributors (counted).

## 13. Patterns worth standardizing and pain points
Worth standardizing:
1. **A real library API**: logic model, parser and fill are separable from the game, with a pluggable logic file format. This is the clearest example of the core we want.
2. **Waypoints** as cached named reachability facts, distinct from macros (substitution). Waypoints help performance and readability.
3. **Logic overlays**: logic edits, macro edits and substitutions as ordered patch files, plus coalescing for optional dependencies. This is how a community extends a world without forking it.
4. **Documented fill guarantees**: the forward fill articles state exactly which progression set is picked and why; stages and groups make item and transition fills one mechanism.
5. **Commutativity as a checked invariant** with a named failure.
6. **State logic with dominance pruning** for consumable, path-dependent resources.
Pain points:
- State modifiers are C# classes, so the most expressive logic (soul, charms, shade) cannot travel as data to another runtime.
- Item effect strings allow conditional, order-sensitive effects; the core detects the resulting non-commutativity only at fill time.
- Settings codes do not cover connection mods, so a shared code may not reproduce a modded seed.
- Generation inside the game ties the generator to the mod host.

## 14. Mapping to our standard
| Construct | Maps to (doc 07 / doc 01) | Status |
|---|---|---|
| Term (int) | item `count`, `counter` (D2.5) | Maps |
| `+`, `|`, `>` with integer | `all`, `any`, `ge` on `count` | Maps |
| `<`, `=` | non-monotone tag (D2.7) | Maps with the tag |
| Macro | authoring `macros` to `defs` | Maps |
| `*Location` reference | `ref` to a named def | Maps if locations' rules are addressable defs |
| Waypoint | `events` plus `reach_event` | Maps; consider an explicit **`waypoint`** flag meaning "cached, not an item" |
| `?` coalescing | none | **New concept `optional_ref`** for overlays, resolved at build time |
| Logic edit, macro edit, substitution files | none | **New concept `overlay`**: an ordered patch package over a base world's `defs`, rules and pools, with its own ledger range. Build-time only, so the canonical form stays flat |
| Item effects `++`, `+=` | `progressive`, `counter` | Maps |
| Raise-to-max, split items (left or right half) | none exactly | **`counter` extension**: `max` updates and multi-term grants (still monotone) |
| Conditional and short-circuit item effects | none | **Cannot be data safely** in general (order-sensitive); allow only the monotone subset, checked by a commutativity test vector at publish time |
| State terms, state unions, modifiers | none | **New primitive candidate `resource_state`** (bounded ints and flags, refill and spend at nodes, dominance pruning). The 22 HK modifiers in C# show the residue risk |
| Transitions as items, coupled groups | `rewire` with coupling (doc 07 2.3) | Maps; coupled duals are the coupling rule |
| Stages and groups | `item_rule`, `place` | **New concept `fill_group`** (disjoint item-location pools) plus **`fill_stage`** ordering |
| Priority, `OnPermute`, depth transform | balancing (D2.7) | Fill policy knobs; declare as named parameters of the fill algorithm |
| Skip tokens (21) | `opt` toggles | Map to the proposed **`trick`** concept (independent tricks, no ordering needed) |
| Pools, costs, start locations | `items` counts, `select` | Maps; shop costs need a **`cost`** field on locations (seen in `Data/costs.json`) |
| ItemChanger runtime, game hooks | CSDK (D5.1) | **Cannot be data** |
