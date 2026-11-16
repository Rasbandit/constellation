# 04. Wind Waker Randomizer (wwrando): architecture audit

> Internal research note, 2026-11-16. Not reviewed by a different agent or a human. Read from a shallow sparse clone of [LagoLunatic/wwrando](https://github.com/LagoLunatic/wwrando) at commit `99281a37` (default branch, committed 2026-10-22). Nothing was built, installed or run. This file is our own writing with paths and links; no code, text or data from the project is reproduced. Numbers are labeled **counted** (by our own script over the clone, or the GitHub API on 2026-11-16), **estimated**, or **not verified**. Paths below are relative to the repo root.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. Summary

- **What it is.** A desktop randomizer for The Legend of Zelda: The Wind Waker (GameCube, North American release only, checked by disc ID and MD5 in `randomizer.py`). It shuffles items, and optionally dungeon, boss, miniboss, secret cave and fairy fountain entrances, sea chart destinations, the starting island, required bosses and (hidden, unfinished) enemies. It also opens the game world from the start and removes most cutscenes (`README.md`).
- **Popularity.** 427 stars, last push 2026-10-22, 16 contributors including anonymous (all counted from the GitHub API on 2026-11-16; landscape row 7 in [`00-landscape.md`](00-landscape.md)). Last tagged release is 1.10.0 from 2023-01-06 (counted, GitHub releases API); `CHANGELOG.md` shows 1.11.0 "in progress", so players of the newest features use development builds (inferred from the changelog wording).
- **License.** MIT (`LICENSE.txt`, GitHub reports `MIT`). We may read and describe it. If a future decision ever reused anything, the MIT notice would have to travel with it and be listed in `THIRD_PARTY_NOTICES.md`. Nothing is reused here.
- **Maintenance.** Active and small: one main author plus occasional contributors (credits list in `README.md`). The GameCube file-format library lives in a separate repo pulled in as a git submodule (`.gitmodules` points at `LagoLunatic/gclib`), not read here.

## 2. Architecture

- **Language and shape.** One Python 3.12 application (`.python-version`, CI sets 3.12 in `.github/workflows/test.yml`). Major parts:
  - `randomizer.py` (1107 lines, counted): the `WWRandomizer` orchestrator. It validates options, encodes the permalink, reads the vanilla ISO, builds `Logic`, runs each feature randomizer, writes files, logs and the output ISO.
  - `logic/`: `item_locations.txt` (3214 lines), `macros.txt` (1164), `enemy_locations.txt` (8150), `item_types.py`, and `logic.py` (1311), the parser and evaluator (line counts counted).
  - `randomizers/`: one class per feature (`items.py`, `entrances.py`, `charts.py`, `boss_reqs.py`, `hints.py`, `enemies.py`, `palettes.py`, `starting_island.py`, `pigs.py`, `extra_starting_items.py`, `music.py`), all subclasses of `randomizers/base_randomizer.py`.
  - `tweaks.py` (2710 lines, counted): Python edits to game data (text, events, stage files) that every seed or certain options need.
  - `asm/patches/*.asm` (22 files, counted) with precompiled byte diffs in `asm/patch_diffs/*_diff.txt` that `asm/patcher.py` applies to `main.dol` and REL modules. A few custom actors are C++ in `asm/actors/`.
  - `options/`: typed option declarations (`wwrando_options.py`) on a small dataclass extension (`base_options.py`).
  - `wwr_ui/`: a Qt GUI built from Qt Designer `.ui` files.
- **Pipeline** (`randomizer.py` constructor and `randomize()`): validate options, compute permalink and seed hash, derive the integer seed, read the ISO (skipped in dry runs), assemble the starting inventory, build `Logic`, check that the settings leave enough progression locations (fails early with a readable message), detect "only dungeons are reachable at start" situations, then run pre-randomization tweaks, then call `randomize()` on each enabled feature in a fixed, commented order (charts, required bosses, entrances, starting island, pigs, extra starting items, enemies, palettes, items, hints), then `save()` on each, then post tweaks, then export the ISO and logs.
- **Split between decision and write.** `base_randomizer.py` makes each feature declare `_randomize` (decide, may use RNG, must not touch game files) and `_save` (write decisions to files, must not use RNG). The orchestrator hands out an RNG only during `_randomize` and raises if a feature touches it elsewhere. This is what makes `--dry` mode possible: generate and log without an ISO.
- **How users run it.** A packaged GUI executable (PyInstaller spec `wwrando.spec`), or `wwrando.py` with `--noui`, `--seed`, `--permalink` and developer flags (`--dry`, `--bulk N`, `--test`, `--mapselect`, `--printflags`, `--stagesearch`, `--disassemble`, `--profile`; 18 `add_argument` calls counted). No web generator in this repo; the project site (`docs/`, built with MkDocs per `mkdocs.yml`) is documentation only.

## 3. World model

- **Locations** are YAML mappings in `logic/item_locations.txt`, keyed by a human-readable name in the form "Zone - Specific place". Each has `Need` (a requirement string), `Original item`, `Types` (comma list of categories) and `Paths` (where in the game files the item is written). Counted: **320 active locations**, 56 more present but commented out (locations considered and not randomized), **59 zones** derived from the name prefix, **29 distinct type tags**, and 511 path lines. Paths use seven forms (counted in the regex list of `change_item` in `randomizers/items.py`): REL offset, `main.dol` address, custom symbol, chest, event action, scalable object and actor, each optionally scoped to a stage layer.
- **Items** are Python lists in `logic/item_types.py`: progress (65 entries, 51 distinct), non-progress (65 entries, 14 distinct), fixed consumables (90 entries, 17 distinct), duplicatable consumables (38 entries, 5 distinct), dungeon progress (17 entries: big and small keys), dungeon non-progress (12: maps and compasses) (all counted by evaluating the list expressions in our own script). Options add 8 Triforce Charts and 41 Treasure Charts to either the progress or non-progress pool (`logic/logic.py`, `__init__`). Item names map to game IDs in `data/item_names.txt` (hex ID then name per line).
- **Regions do not exist as a graph.** There are no region nodes or connection lists. A location's zone comes from its name. Access to a zone behind an entrance is a macro named "Can Access <zone>", and entrance randomization rewrites those macros (section 6). Island-level access is expressed inside the requirement strings and macros.
- **Events** are not a separate construct. Their role is covered by macros (for example "Can Reach and Defeat Ganondorf", the goal) and by a reach-location requirement form, `Can Access Item Location "<name>"`, used for required-boss goals (`logic.py`, `update_required_bosses_macro`).
- **Enemies** are listed per stage in `logic/enemy_locations.txt`: 82 stage keys and 1275 enemy entries (counted), each group with the requirement the original enemies imposed, a "must defeat" flag, and per-enemy placement category (ground, air and others) and file path.

## 4. Logic

- **Representation.** Plain text requirement strings inside YAML, parsed by `Logic.parse_logic_expression` in `logic/logic.py`. Tokens are split on `&`, `|` and parentheses; everything between operators is a requirement name. Mixing `&` and `|` at one nesting level is a parse-time error, so authors must parenthesize. There is no `not`.
- **Requirement name forms** (`check_requirement_met`): an item name; `Progressive X xN`; `<Dungeon> Small Key xN`; `Can Access Item Location "<name>"`; `Option "<key>" Enabled|Disabled|Is "<v>"|Is Not "<v>"|Contains "<v>"|Does Not Contain "<v>"`; a macro name; and the constants `Nothing` and `Impossible`. Anything else raises "Unknown requirement name" at evaluation time.
- **Macros.** `logic/macros.txt` holds **307 named macros** (counted), each a requirement string. They are global, may reference other macros, and are re-parsed when the program rewrites them at runtime (entrances, charts, required bosses). Recursion is caught by an assertion with one whitelisted case (nested entrance macros).
- **Tricks and difficulty.** Two ordered enum options, `logic_obscurity` and `logic_precision`, each None, Normal, Hard or Very Hard (`options/wwrando_options.py`). Macros `Obscure 1..3` and `Precise 1..3` are defined as option comparisons near the end of `macros.txt`, so a trick is written inline as an alternative branch guarded by `Obscure 2` or `Precise 1`. Counted: 46 lines in `macros.txt` and 54 in `item_locations.txt` mention one of these tiers. Comments beside each trick explain it in prose, which doubles as documentation for players. There is no glitched logic.
- **Example (our words).** The Outset Island location where an old swordsman trades a sword technique for ten Knight's Crests requires all of: the Spoils Bag, a macro meaning the player can farm Knight's Crests, a macro meaning the player can do his sword training, and the Magic Meter. Nested macros carry the detail, so the location entry stays short.
- **Validation.** Load-time: blank `Need` raises. Run-time: unknown names and mixed operators raise. Settings-time: "useless" progress items are detected by collecting every item name any progression location or the goal could need, and an item that opens a location but is not declared progress raises (`make_useless_progress_items_nonprogress`). Seed-time: the fill asserts the goal macro is met at the end. Tests exercise the trick tiers directly (`test/test_dry.py`, `test_trick_logic_checks`).
- **Caching.** Requirement results are memoized per inventory state and cleared on every inventory change (`requirement_met_cache` in `logic.py`).

## 5. Fill and placement

- **Algorithm: forward fill with a usefulness heuristic** (`randomizers/items.py`, `randomize_progression_items_forward_fill`). Each step computes the progression locations reachable with the current simulated inventory, picks an item, picks a reachable valid location, and adds the item to the inventory. Item choice prefers an item that opens at least one new location (first useful item in a shuffled list). If none does, it picks among items with the best "usefulness fraction": the fewest other missing items needed before it opens something (`get_items_by_usefulness_fraction` in `logic.py`). With 17 or more reachable locations it stops insisting on useful items. With exactly one reachable location left, a useful item is mandatory and failure raises.
- **Spread heuristic.** Locations get a weight based on when they became reachable; newly opened locations are more likely to be chosen, so items are pushed outward. Duplicate copies of an item (swords, bows) are collapsed in the candidate list so they do not cluster early; dungeon keys are exempt.
- **Ordered passes.** Dungeon items first, with every non-key item temporarily owned and dungeon entrances temporarily treated as open (a local assumed-fill for keys), plus a safeguard location in the first dungeon for one start condition. Then progress items, then unique non-progress items, then consumables (`items.py`, `_randomize`).
- **Constraints** (`check_item_valid_in_location`): dungeon items stay in their own dungeon and off the boss unless "Key-Lunacy" is on; no duplicate item within one shop ship; specific vanilla code limits on two gift locations; trap chests only where every path is a chest; consumables-only locations.
- **Item pools and settings.** 22 progression category toggles (counted, `progression_*` options) decide which location types may hold progress items; others get only junk. Then the program demotes progress items that open nothing under the chosen settings, so they do not occupy scarce progress locations.
- **Starting inventory:** fixed starting tools, a chosen starting gear list, Triforce Shards, heart pieces and containers, and a number of random extra starting items (`randomizer.py` constructor, `randomizers/extra_starting_items.py`).
- **Not present:** progression balancing across players (single player only) and plando; a search for "plando" across the Python files found nothing (counted, zero hits).

## 6. Entrance randomization

- **Model** (`randomizers/entrances.py`). Typed records `ZoneEntrance` (stage, room, exit index, spawn, island or parent exit) and `ZoneExit` (stage, room, spawn, optional boss stage, owned zone). An entrance is either on an island or nested inside an exit, enforced by assertions. Lists exist for dungeons, minibosses, bosses, secret caves, inner caves and fairy fountains.
- **Method.** For each set selected by options (dungeons alone or mixed with caves and fountains), entrances are shuffled and exits assigned one by one, from the sea inward, so every assigned entrance already has a path to the sea. Non-progress exits are paired off first. Terminal exits (no further entrance inside) cannot fill the last open slot. Exit choice uses integer weights by exit class to keep the distribution pleasant, for example boss doors more often lead to dungeons or bosses than caves (`weighted_choice` in `randomizer.py`). Island-level rules keep required and banned bosses on different islands, because the in-game markers show islands, not doors.
- **Coupling.** Coupled: each entrance's warp is retargeted to the exit, and exiting returns to the outermost island entrance (`update_entrance_to_lead_to_exit`, `update_boss_warp_out_destination`).
- **Validation.** Logic sees entrances only through macros: after assignment, "Can Access <zone>" is rebound to "Can Access <entrance>". Before assignment, progress-item detection uses a worst case where every zone needs access to all candidate entrances. Beatability is proven by the item fill that follows, not by a separate entrance search. A "safety entrance" with no requirements is forced when only dungeons and caves are reachable at the start.

## 7. Options model

- **Declaration.** 77 options (counted with Python's `ast` over `options/wwrando_options.py`): 59 bool, 9 int with min and max, 4 enums (sword mode, entrance mix, two trick levels), 2 string lists (starting and randomized gear), 2 strings, 1 dict (custom colors). Each carries a description (HTML fragment), per-choice descriptions, `permalink` (13 set false: cosmetics and personal comfort), `hidden` (1) and `unbeatable` (1, the enemy randomizer, disabled in packaged builds by `wwr_ui/randomizer_window.py`).
- **UI generation.** Semi-generated. The window layout is hand-built in Qt Designer (`wwr_ui/randomizer_window.ui`), and each widget is named after its option key; the window code walks `Options.all()` and binds widgets by name, showing the option description on hover. Settings persist to a YAML `settings.txt`.
- **Presets.** No gameplay presets found in the files read; only cosmetic color presets per player model (`test_save.py` uses one).
- **Conflict validation.** Limited and mostly procedural: `validate()` coerces types and resets out-of-range values to defaults without an error; the constructor fails early when settings leave too few progression locations; one dependency (enemy palettes only matter with enemy rando) is special-cased in the permalink code.

## 8. Seeds, reproducibility, spoiler logs and hints

- **Seed names** come from word lists (`seedgen/`: 795 adjectives, 1471 nouns, 121 names by line count, counted), two adjectives plus a noun or name. Seeds are sanitized to a character set and at most 42 characters, limited by the game banner (`randomizer.py`).
- **Permalink.** Base64 of: version string, NUL, seed, NUL, then a bit-packed stream of every permalink option in declaration order (bools 1 bit, enums as an index in the minimum bit width, ints as offset from minimum in the bit width of the range, starting gear as a bitset plus 2-bit counts). Decoding rejects any other version. A round-trip test exists (`test_convert_options_to_permalink_and_back`).
- **RNG.** Python `random.Random` seeded with the MD5 of the seed string. Every feature randomizer gets a fresh RNG from the same seed (`get_new_rng`), so turning one feature on does not change another feature's stream. Hint-related options and the spoiler-log flag are "RNG-changing": they perturb the stream, so two racers with the same seed but different hint settings get different placements and cannot compare hints to learn more (comment above `RNG_CHANGING_OPTIONS`).
- **Race key.** With "do not generate spoiler log", a secret `SEED_KEY` baked into official builds (`keys/generate_key.py` makes a random hex token) is mixed into the seed, so the public source cannot regenerate the race seed. A seed hash, two character names derived from permalink plus key, is shown on the name entry screen so racers can confirm they have the same seed.
- **Logs.** A non-spoiler log lists which location categories can hold progress items; the spoiler log adds progression spheres (computed by sweeping, `calculate_playthrough_progression_spheres`), every placement, entrance paths, chart mapping and required bosses. Error logs are written on failure.
- **Hints** (`randomizers/hints.py`, 1215 lines counted). Types: path (a zone holds an item required for a named goal), barren (zone holds nothing needed), item, location, and fixed-location. Importance labels: required, possibly required, not required. Required-ness is computed per location by simulating a playthrough with that location's item removed. Hints are distributed across in-game sources chosen by option (fishmen per sea sector, a wandering NPC, the talking boat) and can be cryptic or plain. Text lives in data files: `data/location_hints.txt` (88 entries, with Standard or Remote kinds), `data/progress_item_hints.txt` (44) and `data/zone_name_hints.txt` (59) (counted).

## 9. Game integration

- **Generation time does everything.** The user supplies their own vanilla ISO; the program verifies it, reads the whole disc with gclib, rewrites stage archives, events, text, REL modules and `main.dol`, applies precompiled assembly diffs, links custom actors, and writes a complete new ISO ("WW Random <seed>.iso"). There is no runtime component beyond the patched game code.
- **Item writes** follow each location's `Paths` entries; chests also get a custom trap bit and a chest appearance that matches contents when that option is on (`items.py`, `change_chest_item`).
- **Cosmetics.** Custom player models and color masks (`customizer.py`, `assets/link_color_masks/`), excluded from the permalink so they do not change seeds.
- **Player steps** (from `docs/getting-started.md`): download and extract, pick the vanilla ISO, pick an output folder, choose options or paste a permalink, press Randomize, load the new ISO in Dolphin or Nintendont. About 6 user actions (estimated by reading the guide's four sections), plus ripping one's own disc beforehand.

## 10. Multiworld or co-op

None built in; no network code in the files read, and the docs mention none. Archipelago's in-tree `tww` world (read in our local Archipelago checkout at `4f9545f7`) points players to a separate build of this randomizer published from a fork for Archipelago use (`worlds/tww/docs/setup_en.md:12`); how that fork differs was not verified.

## 11. Tooling and testing

- **Tests:** 14 test functions in 3 files (counted: `test_dry.py` 10, `test_gui.py` 2, `test_save.py` 2), plus `test_helpers.py`. Dry tests run full generation without an ISO; save tests need the game files.
- **CI** (`.github/workflows/test.yml`): pytest with coverage on five configurations (Windows, Windows Qt 5, macOS, Linux, older Linux). Game files come from a private repo through a secret token; without it the `saving` marker is skipped. `python-app.yml` builds release artifacts; `publish-site.yml` publishes the docs site.
- **Debug tools:** `--bulk N` generates N seeds in parallel processes and reports failures, with a commented hook for counting distribution bias; `--printflags`, `--stagesearch`, `--disassemble`, `--test` (spawn in a test room), `--mapselect`, `--heap`, `--profile` (`wwrando.py`).
- **Logic editor or tracker:** none in the repo. Community trackers exist (not verified, not opened).

## 12. Developer ergonomics

- **Add a location:** one entry in `logic/item_locations.txt` with requirement, vanilla item, type tags and file paths; add hint text in `data/location_hints.txt`. If the path form is new, extend `change_item` in `items.py`.
- **Add an item:** add it to the right list in `logic/item_types.py`, its ID in `data/item_names.txt`, its hint name in `data/progress_item_hints.txt`, and any game-side support in assembly.
- **Add a feature:** subclass `BaseRandomizer`, insert it into the ordered list in `randomizer.py`, and add options. The decide-then-save split keeps it testable.
- **Docs:** a player site (`docs/`), and extensive comments inside the logic files. No contributor guide was found in the tree listing. Contributors: 16 (counted, section 1).

## 13. Patterns worth standardizing and pain points

Worth standardizing:
1. **Decide, then write.** RNG only during a decision phase, writes only during a save phase, enforced by the RNG accessor. Enables dry runs, bulk tests and spoiler-only output.
2. **Independent RNG per feature.** Each feature restarts from the seed, so feature toggles do not shift unrelated placements.
3. **Information-bearing options change the seed.** Hint settings perturb the RNG so the same seed cannot leak extra information across racers with different settings.
4. **Location categories as tags plus per-category toggles**, combined with automatic demotion of items that open nothing. This is how a large game stays playable in short races.
5. **Trick tiers on two independent axes** (obscure versus precise), referenced as named tiers in rules, with a prose explanation next to each.
6. **Fail early with a readable reason** when settings are infeasible (too few progress locations), before any search.
7. **Bulk seed testing as a first-class command.**

Pain points (design cost, not criticism):
- No region graph: zone access is encoded as macros that code rewrites at runtime, so static analysis cannot see entrance structure, and correctness of the rewrite depends on code paths.
- Requirement names are matched by string patterns (`Progressive X xN`, `Option "..." Is "..."`), and unknown names fail only when evaluated.
- Item pools and many placement rules live in Python, and several special cases are tied to named locations.
- Permalink is bound to the exact version and to option declaration order, so old permalinks cannot be reopened in newer builds.
- Python's Mersenne Twister and MD5 seeding tie reproducibility to Python.
- Option validation resets bad values silently.

## 14. Mapping to our standard

**Maps to existing primitives and concepts.**
- `Need` strings map to the D2.5 combinators in [`01-standards.md`](../../design/01-standards.md): item names to `has`, `Progressive X xN` to `level` or `has` with `n`, `Small Key xN` to `has` with `n`, `Can Access Item Location` to `reach` on a location, `Option ...` forms to `opt` and `in`, macros to `defs` with `ref`, `Nothing` and `Impossible` to the constants. The "no mixed operators without parentheses" rule fits our tree form.
- `Types` map to location `tags`; "Consumables only" and "No progression" map to the D2.7 location kind `excluded`.
- Dungeon-item placement, shop and trap rules map to `item_rule` from [`07-data-only-world-format.md`](../../design/07-data-only-world-format.md) section 2.3; chart shuffle maps to `permute`; required bosses map to `select` plus a goal over the selection; entrance shuffle maps to `rewire`; consumable fill maps to `weighted_filler`; starting gear and shards map to options of type `item_set` and `range`.
- Per-feature RNG restart matches our per-hook streams (D3.1). The permalink maps to our settings string; ours is difference-from-defaults with a checksum and pins the manifest hash instead of the version.

**Needs a new primitive or concept.**
- **`category_gate` (or `kind_when`):** a location's kind computed from an option over its tags, for example "locations tagged minigame are excluded unless option X". Here 22 toggles drive it. Close to `classification_when` but for locations.
- **Automatic progression demotion (platform analysis, not a primitive):** after options resolve, items that cannot open any non-excluded location or the goal become filler. Our static checks (doc 07 section 5.1) could own this.
- **Ordered choice comparison:** trick tiers need `opt` with `ge` over an ordered choice. Either an `ordered: true` flag on `choice` options or `ge` on choice index.
- **Same-group item rule:** "dungeon X items only in dungeon X locations" as one parameterized `item_rule` matching an item tag to a location tag, rather than one rule per dungeon.
- **`rewire` extensions:** nesting (exits that contain further entrances), terminal-exit handling, class weights, a forced safe first entrance, and constraints on groups of entrances (same island).
- **Rule rebinding from a primitive's output:** chart shuffle and entrance shuffle both work by rebinding named `defs` to the shuffled value. A primitive should be able to declare which `defs` it rebinds, so the rule tree stays static and analyzable.
- **`hints` primitive (new concept):** hint types (path, barren, item, location), importance labels from removal simulation, distribution across declared hint sources, and text keys from i18n. No hint primitive exists in doc 07 yet; the other batch A projects also ship hint systems (per the landscape file list, for example OoTR `Hints.py`; not verified in this note).
- **Information-salting options:** options that change what a player learns but not placement rules should salt the slot seed. This belongs in D3.1 and D3.3.
- **Option metadata `unsafe` or `experimental`:** an option that may produce unbeatable seeds, hidden from normal users.

**Cannot be data, and why.**
- `tweaks.py`, the assembly patches and custom actors: game-binary edits specific to one disc image; they belong in the patch format (CPK, D4.1), not the manifest.
- Location `Paths`: patch-side addresses; they should live in the patch or client package keyed by location ID, not in logic.
- Enemy randomization's "do not add requirements" check enumerates item combinations against each candidate enemy (`filter_out_enemies_that_add_new_requirements` in `logic.py`). It is computable from data, but as a generic primitive it needs a cost bound because it is exponential in relevant items; until three worlds need it, it stays in the residue (doc 07 section 6).
- The fill heuristics themselves (usefulness fraction, age weighting) are generator behavior. Under our standard they belong in CFill or a named fill variant, not in a world file.
