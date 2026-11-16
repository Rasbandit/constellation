# 13. Universal Pokemon Randomizer ZX (UPR ZX): architecture and patterns audit

> Internal research note, 2026-11-16, read at commit `7f00eb866ed35c8fe3963f078b6a2e0979dc2b8c` of the `master` branch of [Ajarmar/universal-pokemon-randomizer-zx](https://github.com/Ajarmar/universal-pokemon-randomizer-zx) (shallow sparse clone of `src/com/dabomstew/pkrandom/` logic, data and CLI folders plus `README.md`; no code was run, built or imported). Not reviewed by a different agent or a human. Our own writing with citations; no code, text or data from the project is reproduced. Numbers are labeled **counted** (by `grep`, `wc` or `git ls-tree` over the clone, or the GitHub API on 2026-11-16), **estimated**, or **not verified**. Paths are relative to `src/com/dabomstew/pkrandom/` unless they start with `src/` or are repo-root files.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. Summary
- **What it is.** A desktop content randomizer for the main-series Pokemon games, Generation 1 through Generation 7 (Game Boy to 3DS). It rewrites a ROM the player supplies: species stats, types, abilities, evolutions, movesets, TMs and tutors, trainers, encounters, static and gift Pokemon, trades, field items, shops and names. It has no item-location logic and no reachability solver.
- **Popularity.** 882 stars and 17 contributors including anonymous (both counted via the GitHub API, 2026-11-16; 00-landscape row 29 recorded 881 stars and its batch D table 18 contributors; the two API reads were not reconciled). It is a fork of the archived original by another author (388 stars, 00-landscape). 00-landscape section 3.5 names it the de facto Pokemon content randomizer.
- **License.** GPL-3.0 (API `license.spdx_id`, 2026-11-16). For us: read and describe only. The ROM offset tables and text tables are their data; we do not copy them. Any future interop would be by file format or by an independent implementation, not by vendoring.
- **Maintenance.** Last push 2024-11-23 and 290 open issues (both counted via the API, 2026-11-16). The README asks for an issue before any contribution and states that feature requests are not taken (`README.md`). Treat it as a stable, slow-moving codebase.

## 2. Architecture
- **Language and size.** Java with a Swing GUI. `romhandlers/` holds 13 files (counted): an interface `RomHandler.java` (660 lines), a shared `AbstractRomHandler.java` (7,563 lines) and one handler per generation from `Gen1RomHandler.java` (2,919) to `Gen4RomHandler.java` (5,842) (all counted with `wc -l`). The GUI is `newgui/NewRandomizerGUI.java` (4,012 lines, counted).
- **Shape.** A single ordered pipeline in `Randomizer.java` (1,341 lines, counted) calls `RomHandler` operations; each generation handler reads and writes game tables, and the abstract base implements the content algorithms once over a common object model (`pokemon/`, 35 files counted).
- **Pipeline order** (`Randomizer.java:67-702`): seed the RNG; limit the species pool; update and randomize move data (power, accuracy, PP, type, category); apply misc tweaks; update and randomize base stats; types; held items; evolutions; stat shuffle or randomization; abilities; evolution fix-ups (impossible, easier, time-based); starters; movesets and egg moves; TM moves and TM compatibility; tutor moves and compatibility; trainers (added Pokemon, double battles, replacement, levels, movesets, held items, names); static, totem and gift encounters; catch rates; encounter tables; in-game trades; field items; shops; pickup items; intro Pokemon; write a check value; save.
- **Running it.** GUI by default. A CLI (`cli/CliRandomizer.java:142`) accepts input ROM, output path, a settings file, a 3DS directory output flag, a 3DS update path and a log flag (6 flags counted). Batch randomization exists (`BatchRandomizationSettings.java`).
- **Ordering is semantic.** Later steps read earlier results: trainer and encounter "similar strength" use base stats after stat randomization, and TM compatibility "prefer type" uses types after type randomization. The order in the pipeline is part of the behavior.

## 3. World or content model
- **Object model.** Species (`pokemon/Pokemon.java`) with six base stats, types, abilities, evolution links and cosmetic forms; moves; trainers with team members; encounter sets (an area's slot list with levels); static encounters; trades; item lists. All are in-memory Java objects loaded per generation.
- **Game data tables.** `config/gen1_offsets.ini` to `gen7_offsets.ini`: 7 files, 166 ROM-version sections in total (20, 22, 46, 42, 28, 4, 4; counted by `grep '^\['`). A section names one ROM release and lists key-value entries: game code, version, text table, free space, table offsets and sizes, trainer counts, class indices, and so on. Gen 3 alone has 777 `StaticPokemon` lines (counted). Sections can inherit with `CopyFrom` (40 uses in Gen 3, counted).
- **Text tables.** 14 `.tbl` character maps (counted) for different languages and fan translations.
- **Constants.** `constants/` (17 files, counted) holds per-generation species, move, item and ability ID constants.
- **What this means.** The content model is a set of typed tables (species, moves, trainers, encounters, items) that every generation fills from its own binary layout. That table-shaped model is exactly what a data-only world could describe; the binary layout is not.

## 4. Logic and constraints
There is no progression logic. Constraints are balance and safety rules, expressed in Java code and toggled by settings:
- **Stat budget.** "Random within base stat total" keeps each species' total, reserves a floor (20 HP, 10 for each other stat) and splits the rest by random weights; any stat above 255 (one byte) triggers a full reroll (`pokemon/Pokemon.java:114-155`). One species has a fixed-HP exception in code.
- **Stat shuffle.** A permutation of the six stats per species (`pokemon/Pokemon.java:83-112`).
- **Follow evolutions.** Evolved forms copy the shuffle order of their base form, or scale the base form's stats by the ratio of totals, or add a random split of the difference (`pokemon/Pokemon.java:157-197`). This is a constraint across related rows.
- **Strength matching.** "Similar strength" replacement starts from a window of plus or minus 10% of the original's stat total, widening by 5% per round until at least three candidates exist or three rounds pass (`romhandlers/AbstractRomHandler.java:6966-6989`).
- **Themes.** Trainer modes include type-themed trainers and type-themed gyms and Elite Four; encounter restrictions include type-themed areas and "catch them all" (every species appears at least once) (`Settings.java:160-199`).
- **Ban lists and safety.** Legendaries, irregular alternate forms and ability-dependent forms can be removed from pools (`romhandlers/AbstractRomHandler.java:733-742`); areas carry their own banned species. TM and tutor compatibility have "sanity" passes, and an option gives full field-move (HM) compatibility so progression moves stay learnable (`Randomizer.java:357-365`). Required field TMs are always placed (`romhandlers/AbstractRomHandler.java:5246`).
- **Validation.** None as a separate pass; constraints are enforced while generating.
- **Example in our words.** "For each species, keep its stat total; give 20 to HP and 10 to each other stat first; split the remainder by six random weights; if any stat exceeds 255, redo; evolved forms keep the same proportions."

## 5. Shuffle and placement algorithms
- **Stats.** Permutation per row, or weighted redistribution of a fixed integer total (section 4). Weights are Java `double`s and results are rounded.
- **Encounters.** Four modes: unchanged, per-slot random, area one-to-one mapping and global one-to-one mapping (`Settings.java:194-196`). Encounter sets are visited in a shuffled order so modifiers do not always favor early areas (`romhandlers/AbstractRomHandler.java:725-730`).
- **Field items.** Shuffle (permute existing) or random. The "even" mode redraws up to 100 times to keep each item's placement count near the running average (`romhandlers/AbstractRomHandler.java:5260-5275`).
- **Shops.** Shuffle or random, with optional bans and price balancing (`Settings.java:301-302`).
- **Retries.** Local and bounded (100 redraws, 3 widening rounds) or unbounded recursion on reroll (stats). No global restart.

## 6. Entrance or area randomization
None. Area mapping (section 5) remaps which species live in which area, not how areas connect.

## 7. Options model
- **Size.** `Settings.java` is 2,389 lines with 22 enums and 107 boolean fields (counted by `grep`; approximate, since some fields are not options).
- **Settings string.** A fixed 51-byte bit-packed block (`LENGTH_OF_SETTINGS_DATA`, `Settings.java:52`), then the ROM name, a CRC32 of the block and a checksum of the custom names file, all Base64-encoded (`Settings.java:364-604`). Positions are fixed, so meaning depends on version.
- **Versioning.** A saved settings file starts with a 4-byte version; older versions are migrated by `SettingsUpdater.java` (14 version branches, counted), versions that are too old or newer than the program are rejected (`Settings.java:330-362`).
- **Per-ROM adaptation.** `tweakForRom` (`Settings.java:904`) turns off or rewrites settings that the loaded game cannot support and reports what changed. This is capability negotiation between options and a game variant.
- **Conflicts.** Checked in the GUI, for example race mode requires that trainers or encounters change (`newgui/NewRandomizerGUI.java:812-817`). The rules live in UI code, so the CLI path relies on the saved settings already being valid (inferred).
- **Presets.** Preset dialogs (`newgui/PresetMakeDialog.java`, `PresetLoadDialog.java`) bundle a seed, a settings string and the ROM name so another player can reproduce a game (inferred from the dialog names and the `randomize(filename, log, seed)` signature; dialogs not read in full).

## 8. Seeds, reproducibility, spoiler logs, hints
- **RNG.** A static wrapper over `java.util.Random` with two streams, main and cosmetic, both seeded from one `long` (`RandomSource.java`). Calls are counted and the count is logged at the end (`Randomizer.java:691`). Separating cosmetic draws means cosmetic choices do not shift gameplay output.
- **Check value.** While randomizing, an integer is folded from key results (stats, trainer teams, encounters, TMs) and written into the ROM (`Randomizer.java:217-225`, `:674`). Two players can compare it in game to confirm identical output.
- **Determinism risks.** `java.util.Random` has a documented algorithm (from memory, not verified), but stat splitting uses floating point and rounding; a port to another language would need the same arithmetic to match.
- **Spoiler log.** A verbose text log of every change, written alongside the ROM. Race mode exists (`Settings.java`); exactly what it hides was not verified.
- **Hints.** None; there is no progression to hint.

## 9. Game integration and delivery
- **Method.** Direct ROM rewrite. Handlers decompress, edit and recompress game files, including Nintendo DS archives and 3DS containers (`src/compressors/`, `ctr/`, `newnds/`). Code patches (`patches/`, 68 files counted) add tweaks.
- **Output.** A new ROM file, or for 3DS a LayeredFS directory for a custom firmware loader (`cli/CliRandomizer.java:228`).
- **Player steps.** Install Java, start the program, open the ROM, choose settings or a preset, randomize, then play in an emulator or on hardware: about 5 steps (estimated, from the CLI and GUI flow).

## 10. Multiworld or co-op support
None. Archipelago has in-tree `pokemon_rb` and `pokemon_emerald` worlds (directories confirmed in our local Archipelago checkout; contents not read here) that do their own content shuffles.

## 11. Tooling and testing
- **Tests.** No test files in the tree (0 paths containing `test`, counted with `git ls-tree`). No CI workflows; `.github/` holds only issue templates (counted).
- **Helpers.** Python scripts under `util/` for ROM byte reading and distribution analysis (`util/analyze_distribution.py`), not read in detail.

## 12. Developer ergonomics
- **Add a game version.** Add an `.ini` section with offsets (often `CopyFrom` plus overrides) and, if needed, a text table. A new generation needs a new handler of 3,000 to 6,000 lines (counted range of existing handlers).
- **Add a content option.** Touch `Settings.java` (field, bit position, string encoding, version bump and `SettingsUpdater` branch), the GUI, `Randomizer.java` (pipeline slot) and usually `AbstractRomHandler.java`. Five places for one option (counted from the files involved).
- **Docs.** A wiki page for building; no architecture doc in tree.

## 13. Patterns worth standardizing and pain points
Patterns worth keeping:
1. **Content as typed tables, algorithms written once.** Seven generations share one implementation of each shuffle because handlers map binary layouts to a common table model.
2. **Constrained redistribution.** "Keep the total, respect floors and a ceiling, follow related rows" is a reusable shape (stats here; also budgets like shop prices or level curves elsewhere).
3. **Widening similarity window.** Picking a replacement close to the original by a numeric key, widening until a minimum candidate count, avoids empty pools without a global retry.
4. **Theme then pick.** Choose one theme per group (a type per gym or area), then select members within it.
5. **Per-variant option adaptation** with feedback to the user, rather than failing.
6. **Check value shown in game** for "same seed, same game" confirmation, and a separate cosmetic RNG stream.

Pain points:
1. Positional bit-packed settings strings tie a string to one program version; every new option needs a migration branch.
2. Pipeline order carries meaning but is implicit in code; there is no declared dependency between steps.
3. Constraints are spread through a 7,563-line base class; no declarative list a tool could read or test.
4. Floating-point weights make cross-language reproduction hard.
5. No automated tests guard the content algorithms.

## 14. Mapping to our standard
Maps to existing constructs:
- Species, moves, trainers and encounter sets as `tables` (01 D2.2; 07 section 3.3 table-driven world).
- Stat shuffle per species: `permute` (07 section 2.3) over a row's own columns. Global one-to-one encounter mapping: `permute` of a species column. Per-slot random: `select` with `where` filters and exclusions.
- Ban lists: `select` exclusions and `where` filters. Theme constraints: `select` per-group filters, if the group key can come from an earlier choice (needs checking).
- Option conflicts (race mode needs a change): option `requires` and `conflicts` (07 section 2.3). Per-ROM adaptation: capability declaration (07 section 2.5) applied to game variants, plus option `depends` (01 D2.3).
- Two RNG streams: per-hook HKDF streams (01 D3.1). Check value: the bundle hash (01 D3.2 item 6) shown in game through CSDK.
- Settings string: our difference-from-defaults `cs1` string (01 D3.1) avoids positional fragility.

Needs a new primitive or concept:
- **`redistribute` (stat-budget primitive).** Input: a table, a set of integer columns, a total source (keep row total, or a range), per-column floor and ceiling, a weighting rule using integer draws, and a "follow chain" relation (evolution links) with modes copy-order, scale-by-total or add-difference. Output: new integer values. Must use integer arithmetic with largest-remainder rounding so it is reproducible (01 D3.2). Evidence in this batch: UPR stats (strong), KH2 level-up stat pools and weapon stat ranges (partial, see 15). Needs a third world under the rule of three (07 section 2.4).
- **`select` with a widening band.** An extension: "nearest by integer key, start at plus or minus P%, widen by Q% per round, stop at N candidates or R rounds". Could be a `select` option rather than a new primitive.
- **Step ordering.** A content world needs declared order between primitives (stats before strength matching). Doc 07 does not yet say how primitive outputs feed later primitives' filters; a small "stage" concept (ordered list, each stage reads prior outputs) is needed.
- **Related-row constraints.** "Evolutions follow their base form" is a relation between rows; neither `permute` nor `select` has it today.

Cannot be data, and why:
- ROM offsets, compression, text encoding and code patches. These belong to a patch producer outside the manifest (CPK, 01 section 3.4, third-party) because they are binary layout per release, and they are GPL data here in any case.
- The one-off species exception and other hardcoded special cases could be data (a per-row override), but only if the primitive exposes per-row floors and ceilings.

Not verified: race mode behavior, preset file contents, GUI string bundle and how the CLI validates conflicts.

## Sources read (paths in the clone, 2026-11-16)
- `README.md`; `.github/ISSUE_TEMPLATE/` (listing only).
- `src/com/dabomstew/pkrandom/Randomizer.java` (pipeline, check value, logging).
- `src/com/dabomstew/pkrandom/RandomSource.java` (RNG streams).
- `src/com/dabomstew/pkrandom/Settings.java` (enums, string encoding, `tweakForRom`), `SettingsUpdater.java` (version branches, counted only).
- `src/com/dabomstew/pkrandom/pokemon/Pokemon.java` (stat shuffle, redistribution, follow evolutions).
- `src/com/dabomstew/pkrandom/romhandlers/AbstractRomHandler.java` (encounters, strength matching, field items), `RomHandler.java` and handler sizes.
- `src/com/dabomstew/pkrandom/config/gen*_offsets.ini` (section and key counts only; contents not reproduced).
- `src/com/dabomstew/pkrandom/cli/CliRandomizer.java` (flags), `newgui/NewRandomizerGUI.java` (race mode checks).
- GitHub API: repository and contributors endpoints.
