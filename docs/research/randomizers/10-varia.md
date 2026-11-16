# 10. Super Metroid VARIA Randomizer and Solver: architecture and patterns audit

> Internal research note, 2026-11-16, read at commit `5ce97e127b309b0481f1fa8b82930633dec50289` (2026-08-05) of the `master` branch of [theonlydude/RandomMetroidSolver](https://github.com/theonlydude/RandomMetroidSolver) (shallow sparse clone of `rando/`, `logic/`, `graph/`, `solver/`, `rom/`, `utils/`, `doc/`, the preset folders, `tools/autotracker/` and the Python and template parts of `web/`; no ROM patches, sprites or static assets; no code was run). Not reviewed by a different agent or a human. Our own writing with citations; no code, text or data from the project is reproduced. Numbers are labeled **counted** (`grep`, `find`, `wc`, `git ls-tree` over the clone, or the GitHub API on 2026-11-16), **estimated**, or **not verified**. Paths are relative to the repo root.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. Summary
- **What it is.** VARIA is a randomizer, solver, item tracker and plandomizer for Super Metroid (SNES), served at varia.run with a beta site for the `master` branch (`README.md`). Its distinguishing idea is that the solver comes first: every rule returns a difficulty along with a yes or no, and the generator uses the solver to accept or reject seeds by difficulty.
- **Popularity.** 58 stars, 24 contributors including anonymous (counted, GitHub API; 00-landscape row 21). Archipelago vendors a copy at `worlds/sm/variaRandomizer/` (listed in our local Archipelago checkout at `4f9545f7`).
- **License.** MIT (GitHub API). Read and describe; any reuse carries the MIT notice.
- **Maintenance.** Active: last push 2026-08-05 (counted). `production` and `master` branches back the two sites (README). No CI config in the tree (`git ls-tree` finds no `.github/`; counted).

## 2. Architecture
- **Language.** Python 3 throughout. Core modules: `rando/` 19 files, `solver/` 13, `logic/` 6, `graph/` 21 across three map variants, `rom/` 19 (counted). Largest logic files: `solver/interactiveSolver.py` 1,068 lines, `logic/helpers.py` 911, `solver/commonSolver.py` 872, `rando/Items.py` 806 (counted).
- **Logic variants.** Three graphs: `graph/vanilla/`, `graph/mirror/` (a mirrored map hack) and `graph/rotation/`, each with `graph_access.py`, `graph_helpers.py`, `graph_locations.py`.
- **Pipeline.** `randomizer.py` parses arguments and two preset files, seeds Python's `random` (`:349-360`), builds `RandoSettings` and `GraphSettings`, then `rando/RandoExec.py:59-115` loops: randomize door colors if enabled, build the area graph (`GraphBuilder`), set up the item and location container (`RandoSetup`), pick a filler by progression speed, fill, then solve the escape sequence. Output is an IPS patch plus JSON (`web/backend/randomizer.py:355-393`).
- **How users run it.** Mainly the web2py website (`web/controllers/solver_web.py`, `web/backend/*.py`, 13 files, 3,824 lines counted). The CLI is documented but the README says it is supported only with a rando preset plus a skill preset (README, CLI section). The website runs the CLI as a subprocess with a 20 second runtime limit (`web/backend/randomizer.py:302-350`).

## 3. World model
- **Items.** Python objects in `rando/Items.py` with a type string, category and class (major, minor) (about 33 `Type=` definitions, counted by `grep`; exact count not verified).
- **Locations.** Defined centrally in `graph/location.py` (110 `define_location` calls, counted) with map attributes, then given logic per graph variant in `graph_locations.py` (109 location blocks in the vanilla file, counted).
- **Access points.** The graph nodes. `graph/vanilla/graph_access.py` declares 94 `AccessPoint` objects (counted). Each has a name, an area, a solve area, a map of reachable neighbor access points to rules, and, for door nodes, ROM room and door pointers, entry coordinates and music (`:27-60`). `internal=True` marks nodes that are not door transitions. Start locations are access points with a `start` record (spawn id, doors to open, patches needed).
- **Events.** Bosses and objectives are tracked through the `SMBoolManager` item set plus objective goals (`utils/objectives.py`, about 71 goal definitions, counted as `Goal(` calls minus the class line).
- **Format.** Code: dictionaries and lambdas in Python modules. Presets are JSON (section 7).

## 4. Logic
- **Representation.** Every rule is a lambda over a state manager `sm` returning an **SMBool**: a boolean, a numeric difficulty, the list of techniques (`knows`) used and the items used (`logic/smbool.py:8-14`). AND (`wand`) fails if any part fails and otherwise **sums** difficulties; OR (`wor`) returns the **easiest** passing branch (`:78-118`); `wandmax` takes the max instead of the sum.
- **Difficulty scale.** Named integers in `utils/parameters.py:6-15`: easy 1, medium 5, hard 10, harder 25, hardcore 50, mania 100, then two more doubling steps and infinity. `diff2text` maps them to labels for display.
- **Tricks.** `class Knows` in `utils/parameters.py:89` defines 105 techniques (counted) as SMBools with a default difficulty. A **skill preset** (`standard_presets/*.json`, 11 counted) assigns each technique `[enabled, difficulty]`, for example a player profile that knows wall jumps at easy and a harder trick at hard. Rules call `sm.knowsX()`, so a technique's cost is the player's, not the author's.
- **Helpers.** `logic/helpers.py` (911 lines) and per-graph `graph_helpers.py` hold shared rules like passing a bomb wall or entering a boss room, cached through `Cache.ldeco` (`graph_access.py:18`).
- **Location rules in three parts.** `AccessFrom` (rules from specific access points), `Available` (the in-room requirement) and `PostAvailable` (what is needed to get back out after taking the item, evaluated with the item added) (`graph/vanilla/graph_locations.py:8-30`; `rando/MiniSolver.py:43-48`). This models one-way rooms and softlock risk in the logic itself.
- **Example, in our words.** The Bomb item location is reachable from the landing site; taking it needs the morph ball and passing a specific door; leaving afterwards needs either a named escape technique or a way through bomb passages (`graph_locations.py:20-30`). Its difficulty is the sum of the chosen branches' costs for this player's skill preset.
- **Validation.** No static checks. A long-running script generates and solves many random-parameter seeds and flags mismatches (`tools/test_jm.sh`, header comment).

## 5. Fill and placement
- **Fillers by progression speed** (`rando/RandoExec.py:30-52`): `FrontFiller` for basic, `FillerRandomSpeedrun` for speedrun, `FillerProgSpeed` for the other speeds, `FillerScavenger` for scavenger mode, and a two-phase Chozo wrapper for the Chozo item split.
- **Forward fill with rollbacks.** `FillerProgSpeed` alternates progression and non-progression phases, can get stuck and roll back to saved states (`rando/FillerProgSpeed.py:14-50`). Its parameters per speed (item limits, location limits, chance of a helpful minor item, softlock probability) live in `ProgSpeedParameters` (`rando/RandoSettings.py:89-200`).
- **Random fill with a solver gate.** `FillerRandom.isBeatable` runs the cheap `MiniSolver`, then the full `RandoSolver`, and rejects the attempt if the solver's difficulty is below `minDiff` or above `maxDiff` (with an exception for boss-only overruns) (`rando/FillerRandom.py:199-230`).
- **Restrictions.** `rando/Restrictions.py`: major and minor split (Full, Major, Chozo, Scavenger), early or late morph ball, suit restrictions, per-location class matching (`isItemLocMatching :106`), precomputed for speed (`:172-188`).
- **Retries.** Up to 500 attempts at a layout for area, door or scavenger modes, 1 otherwise, bounded by a wall-clock end date (`RandoExec.py:66-90`). Errors are human-readable sentences that name the failed step (for example, no area layout found for the chosen settings).
- **Pools.** Quantities of missiles, supers, power bombs, minors and energy are rando preset fields (`rando_presets/default.json`, `missileQty`, `minorQty`, `energyQty`).
- **Plando.** `--plandoRando` takes JSON with already placed items, transitions, door colors and patches (`randomizer.py:236,627-652`); the web plandomizer uses `interactiveSolver.randoPlando` to fill the rest (`solver/interactiveSolver.py:307`).

## 6. Entrance, door or area randomization
- **Area randomization.** `graph/graph_utils.py` builds transition pairs between door access points: full area rando (`createAreaTransitions :187`), a lighter variant (`:323`), boss room shuffle (`createBossesTransitions :167`) and a "minimizer" mode that opens a reduced map to a location limit (`:246`). Door ROM pointers per access point let the patcher rewrite doors (`getDoorConnections :502`).
- **Door color randomization.** `utils/doorsmanager.py`, called before graph building (`RandoExec.py:73-76`).
- **Escape.** After fill, `GraphBuilder.escapeGraph` can randomize the escape target and computes an escape timer from the path (`rando/GraphBuilder.py:72-248`).

## 7. Options model
- **Two presets.** A **skill preset** (what the player can do, `standard_presets/`, 11 counted, plus `community_presets/`) and a **rando preset** (what the seed is, `rando_presets/`, 46 counted). `rando_presets/default.json` has 56 top-level keys (counted with our own script).
- **Multi-value randoms.** Fields such as start location, progression speed, morph placement and energy quantity accept `random` plus a list of allowed values (README CLI section; `web/backend/randomizer.py:231-243`).
- **UI.** Server-rendered web2py templates (`web/views/randomizer.html`) with preset categories and descriptions hard-coded in Python (`web/backend/randomizer.py:98-158`). Objective exclusions are exported to the page as JSON (`:170`). Not generated from a schema.
- **Validation.** `validateWebServiceParams` checks switches, quantities and multi-selects (`web/backend/randomizer.py:272`); cross-setting problems surface as generator error messages, and the generator can report "forced arguments" it changed (`:368-369`).

## 8. Seeds, reproducibility, spoiler logs, hints, solver and difficulty estimation
- **Seeding.** Python `random.seed(seed)`, where the seed is a number from the user or `getRandomizerSeed()` (`randomizer.py:349-360`). Same seed, presets and version give the same game (inferred; not tested).
- **Race mode.** The web server draws a random "magic" number and passes `--race` (`web/backend/randomizer.py:246-247,309-310`); the generator seeds with `seed XOR magic` (`randomizer.py:358-360`), so the displayed seed number alone does not regenerate the race seed. The ROM reader loads a `rom/race_mode.py` module when a magic is given (`rom/romreader.py:216-218`), but that file is not in the public tree (`git ls-tree` over `rom/`; counted), so the race obfuscation code appears to be kept private (inferred).
- **Permalink.** A UUID `seedKey` is stored with the parameters in the database, the IPS is kept server-side LZMA-compressed under that key, and `/customizer/<uuid>` reloads the seed for cosmetic changes (`web/backend/randomizer.py:175-212,249-265,371-378`).
- **Solver.** `solver/` is a full playthrough simulator. `computeDifficulty` repeatedly picks the easiest available location (majors before minors on ties), checks objectives, handles "no comeback" areas (`solver/comeback.py`), and stops at the end-game location or a runtime limit (`solver/commonSolver.py:580-640`). The result is the maximum difficulty of any location visited plus whether enough items were collectible (`:795-812`). `DifficultyDisplayer` turns it into a label and a percent (`solver/difficultyDisplayer.py`).
- **Solver as a service.** The web solver accepts a ROM or its JSON, a difficulty target and a pickup strategy (`web/backend/solver.py:40-42,348-372`) and returns the path, techniques used per step, and difficulty. `rom/romloader.py` reads placements back out of a patched ROM, so it can rate seeds it did not generate (inferred from the loader design; not tested on other generators' ROMs).
- **Difficulty as a generation constraint.** `maxDifficulty` and `minDifficulty` (`randomizer.py:101-107`) bound the solver's result; the random filler rejects seeds outside the window (section 5). `progressionDifficulty` biases the progression filler. This is the clearest **difficulty estimation** we have seen in the audit so far.
- **Stats.** The site keeps generation statistics per preset (`web/backend/stats.py`, `extStats.py`, `progSpeedStats.py`, `web/database/create_extended_stats.sql`).
- **Spoilers and hints.** The solver path doubles as the spoiler. Hints are not a large system here (not verified in depth).

## 9. Game integration and delivery
- **Patch format.** IPS. `rom/ips.py` builds, merges and applies IPS records; `rom/rompatcher.py` applies many named base IPS patches per option (`applyIPSPatches :386`), the seed's data, and cosmetic patches. The repo carries 545 patch files under `patches/` (counted, not cloned).
- **Browser patching.** The generator returns the IPS base64-encoded; the page decodes it and applies IPS records, including RLE records, to the player's ROM in JavaScript (`web/views/randomizer.html:808-882`), then saves with FileSaver. The vanilla ROM is CRC32-checked and cached in localforage (`:1145-1188`).
- **Steps for a player** (estimated from the code path): load the ROM once, pick two presets, generate, save. The cosmetic customizer reuses the stored seed by UUID.

## 10. Multiworld or co-op support
- None built into VARIA (00-landscape). The solver mentions multiworld only for autotracker compatibility (`web/backend/solver.py:296`). Archipelago's `sm` world vendors this code for its own multiworld.

## 11. Tooling and testing
- No unit test suite or CI (counted: `tools/ItemsTest.py`, `boss_test.py`, `test_compress.py`, `test_patch.py`, `test_jm.sh` are the only test-named files). `tools/test_jm.sh` is a soak test: generate N seeds with random parameters, solve each, and log mismatches and failures to CSV.
- `--vcr` records a generation for replay (`utils/vcr.py`, `rando/RandoExec.py:60`), and `.dot` graph dumps aid debugging (`randomizer.py:648`).
- **Tracker.** The web tracker is the interactive solver with the same logic; `tools/autotracker/` ships map images and a `data.js` for emulator-connected tracking.
- **Hosting.** web2py, a MySQL-style schema (`web/database/create_schema.sql`), a Docker folder (`web/docker/`, not cloned), subprocess per generation.

## 12. Developer ergonomics
- Adding a location: `define_location` in `graph/location.py`, logic in each variant's `graph_locations.py`, ROM address in `rom/addresses.py`, and map tile data. Adding a trick: a `Knows` entry, a value in every skill preset (missing ones get completed from defaults, per the comment on `SMBool.toPreset` in `logic/smbool.py`), and uses in rules.
- Docs: `doc/` holds 16 design notes (counted), plain-text design notes (not opened; `idees.txt` suggests some are in French, and the README shows French git output), plus a website help section. Contributors: 24 (counted).

## 13. Patterns worth standardizing and pain points
- **Difficulty-valued logic.** A rule result that carries cost and the techniques used gives, for free: per-player skill profiles, difficulty estimates, "which tricks does this seed need", and difficulty windows at generation.
- **Skill preset separate from seed preset.** "What I can do" versus "what I want" is a split worth copying; skill profiles can be shared and reused across seeds.
- **Leave-after-taking rules** (`PostAvailable`) put softlock prevention into data.
- **One logic engine for generator, solver, tracker and plando.**
- **Generate-then-solve gate with bounded retries and readable errors.**
- **Pain points.** Logic as Python lambdas, so only Python can evaluate it (Archipelago vendors the whole tree to reuse it). Sum-of-costs is order-dependent in subtle ways (difficulty of a location depends on which OR branch was cheapest at that moment). Race secrecy depends on private code and a server-held magic number, so a race seed cannot be regenerated by a third party. Three graph variants duplicate logic. No CI.

## 14. Mapping to our standard
**Maps to existing doc 07 primitives and doc 01 concepts**
- Access points and transitions map to `regions` and `connections`; locations to `locations` with `rule` (01 D2.2). Helpers map to `defs`.
- Technique on or off maps to toggle options plus `opt`; logic variants (vanilla, mirror) map to separate worlds or a `choice` option. Rando presets map to `presets`.
- Area and boss transitions map to `rewire` with coupling (07 2.3). Major and minor split and morph placement map to `item_rule` and `place`. Ammo and energy quantities map to item `count` expressions over options. Plando maps to `place`.
- Multi-value randoms map to settings-layer weights (01 D2.3). Bounded retries with deterministic errors match 01 D3.2.

**Needs a new primitive or concept**
- **`cost` semiring for rules (candidate, a language extension, not a primitive):** an optional integer cost per rule leaf, with AND as sum (or max) and OR as min, evaluated alongside the boolean. Reachability gains "minimum cost to reach". Integers only, so it stays total and deterministic. Evidence: `logic/smbool.py:78-118`, `utils/parameters.py:6-15`. Seen in different form in z17 tiers (doc 12) and in wotw-seedgen difficulty tiers (00-landscape section 5).
- **`skill_profile` (candidate concept):** a named, shareable map from technique key to enabled plus cost, separate from seed options and resolved before logic. Evidence: `standard_presets/*.json`, `class Knows`.
- **`difficulty_window` (candidate generator feature):** reject a fill whose solver cost is outside `[min, max]`, with a bounded retry count. Evidence: `rando/FillerRandom.py:199-230`.
- **`exit_rule` on locations (candidate):** a rule that must hold after collecting the item to count the location as safely collectible. Evidence: `PostAvailable`. Could compile to an event plus a connection, which is worth trying before admitting it (07 2.4).
- **Reference solver output (concept for 01 D2.9):** a canonical playthrough with per-step cost and techniques, so trackers and race organizers can show "this seed needs these tricks".

**Cannot be data, and why**
- IPS generation, compression, palette and sprite work, door pointer rewriting and the escape timer computation are ROM work for a third-party patcher (07 6.7).
- The progression-speed fillers with rollback and probabilistic phase rules are search strategies, not world facts; they belong in the platform's fill algorithms as named options, not in a world file.

## Appendix A. Not verified, and open questions
- Whether a seed number plus the two presets regenerates byte-identical output across Python versions (Python's `random` is stable across versions for `random.seed` with an int, from memory, not verified).
- How the web solver handles ROMs from other generators, and how much of the race obfuscation the public solver can undo (the race module is not in the tree).
- Whether Archipelago's vendored copy (`worlds/sm/variaRandomizer/`) tracks this commit or an older one (not compared).
- Typical solver runtime per seed and the share of attempts rejected by the difficulty window: not measured. The web call allows 20 seconds (`web/backend/randomizer.py:303`), which suggests seconds, not minutes (inferred).
- How the hint and objective display in game is generated (`utils/objectives.py` outline only).

## Appendix B. Files read
- `README.md`, `.gitmodules`, `randomizer.py` (arguments and `:349-394`, `:540-560`, `:627-660`, `:815-895`)
- `logic/smbool.py`, `logic/smboolmanager.py` (outline), `utils/parameters.py:1-40` and class outline
- `graph/vanilla/graph_access.py:1-60`, `graph/vanilla/graph_locations.py:1-40`, `graph/location.py` (outline), `graph/graph_utils.py` (outline)
- `rando/RandoExec.py:20-120`, `rando/FillerRandom.py:195-230`, `rando/FillerProgSpeed.py:1-60`, `rando/MiniSolver.py:1-60`, `rando/Restrictions.py` (outline), `rando/RandoSetup.py` (outline), `rando/RandoSettings.py:89-200` (outline), `rando/GraphBuilder.py` (outline)
- `solver/commonSolver.py:580-640,795-815`, `solver/difficultyDisplayer.py`, `solver/randoSolver.py:1-30`, `solver/interactiveSolver.py` (outline), `solver/comeback.py` (outline)
- `rom/ips.py` (outline), `rom/rompatcher.py` (outline and `:334-420`), `rom/romloader.py:1-120,180-200`, `rom/romreader.py:205-230`
- `web/backend/randomizer.py:1-420`, `web/backend/solver.py` (outline and `:290-372`), `web/views/randomizer.html` (grep and `:800-920,1140-1190`)
- `rando_presets/default.json`, `standard_presets/regular.json` (first bytes), `tools/test_jm.sh:1-30`
