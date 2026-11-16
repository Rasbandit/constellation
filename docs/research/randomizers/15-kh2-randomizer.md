# 15. Kingdom Hearts II Randomizer (KH2Randomizer): architecture and patterns audit

> Internal research note, 2026-11-16, read at commit `ac28071c1e7eff7393df54c91196fe4f43c727e9` of the `master` branch of [tommadness/KH2Randomizer](https://github.com/tommadness/KH2Randomizer) (shallow sparse clone of `Module/`, `List/`, `Class/`, `tests/`, `seedtests/`, `presets/`, `docs/`, `UI/`, `.github/` and root `.py` and `.md` files; no code was run, built or imported). Not reviewed by a different agent or a human. Our own writing with citations; no code, text or data from the project is reproduced. Numbers are labeled **counted** (by `grep`, `wc` or `git ls-tree` over the clone, or the GitHub API on 2026-11-16), **estimated**, or **not verified**.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. Summary
- **What it is.** The community seed generator for Kingdom Hearts II Final Mix on PC (Steam and Epic) and on the PS2 emulator PCSX2 (`docs/setup/`). It randomizes item locations under logic and also a lot of content: level-up stats, weapon stats, ability costs, form experience, synthesis recipes, attack data, battle levels, music, and bosses and enemies through an external library. It emits a mod package for the OpenKH Mods Manager.
- **Popularity.** 35 stars and 23 contributors including anonymous (both counted via the GitHub API, 2026-11-16; 00-landscape row 34; its batch D table lists 25 contributors, not reconciled). The community uses its own tracker, hint systems, daily seeds and a league (`docs/` has `daily-seeds`, `league`, `hints`, `trackers`).
- **License.** Apache-2.0 (API `license.spdx_id`). For us: permissive; reuse would need the license and NOTICE handling, but this note only describes.
- **Maintenance.** Active: last push 2026-09-09, 12 open issues (counted via the API). Linux support from source and as an AppImage is documented (`README.md`).

## 2. Architecture
- **Language and size.** Python 3.12 with a PySide6 desktop UI (`README.md`). `Module/*.py` and `Class/*.py` total 17,509 lines (counted). Largest: `Class/seedSettings.py` (3,824), `Module/seedmod.py` (1,879), `Module/zipper.py` (1,866), `Module/newRandomize.py` (1,523) (counted).
- **Components.** `List/location/` (25 files, one per world plus levels, bonuses and a graph module) defines checks and the logic graph; `List/inventory/` (21 files) defines items by category; `Module/newRandomize.py` does content and item assignment; `Module/seedEvaluation.py` validates; `Module/hints.py` builds hints; `Module/zipper.py` and `Module/seedmod.py` write the mod.
- **Pipeline** (`Module/generate.py:13-35`): build settings; up to 50 attempts of (randomize, validate reachability, generate hints, write zip); on a randomizer exception, pick a new random seed name and try again; after 50 failures raise the most common error.
- **Running it.** GUI (`localUI.py`), or a CLI generator (`cli_gen.py`, not read). Mutable data goes under the XDG data directory on Linux (`README.md`).

## 3. World or content model
- **Checks as Python data builders.** Each world file declares node IDs and check names as string enums and builds a graph of nodes holding checks (`List/location/agrabah.py`). Check constructors by kind (counted with `grep`, excluding definitions): 320 chests, 110 popups, 47 item bonuses, 24 stat bonuses, 15 hybrid bonuses, 3 double bonuses (519 total), plus level-up and form-level locations in their own files.
- **Edges.** 197 `RequirementEdge` uses (counted). An edge carries an optional requirement function, a strict flag and a `battle` flag (`List/location/graph.py:51-56`).
- **Items.** Defined by category files (abilities, magic, forms, proofs, reports, keyblades, and so on). Each item has a type and rarity used by weighting.
- **Content tables.** Level-up experience and stat pools, keyblade, staff and shield stats, form experience, synthesis recipes, attack data presets (`Module/attackDataPresets.py`), battle levels (`Module/battleLevels.py`), knockback types.

## 4. Logic and constraints
- **Requirements are Python callables** over an inventory list, with shared helpers like "needs fire, blizzard and thunder" or "needs N magnets" (`Module/itemPlacementRestriction.py:9-56`).
- **Invalid checks.** Each location carries a set of item types it may not hold (`Module/newRandomize.py:1100`), the equivalent of an item rule.
- **Depth restrictions.** Where certain item classes may land is set by a "depth" choice: first visit, second visit, first boss, last story boss, superbosses, and so on (`Module/depths.py:7-98`). Settings validation rejects contradictory depth choices, for example proofs and reports at the same boss category (`Module/RandomizerSettings.py:825-840`).
- **Reverse mode.** Location builders take a reverse flag that changes the world order (`List/location/agrabah.py:75-86`, `List/location/graph.py:60`).
- **Validation.** A separate validator computes spheres from the placed items and the requirement functions; failure triggers a retry (`Module/seedEvaluation.py:45-338`).
- **Example in our words.** "The Treasure Room chest needs the Chasm of Challenges edge, which is a battle; proofs must not land before the last story boss; this location never holds a form."

## 5. Shuffle and placement algorithms
- **Weighted item placement.** For each item, location weights come from the item's rarity and the location's depth under a difficulty setting; a location is drawn by weighted choice, rejected if the item type is invalid there, with up to 100 draws before failing (`Module/newRandomize.py:1078-1155`).
- **Weight curves.** Per difficulty, a weight list over depth for each rarity, built from a formula with powers and floors; easy difficulties put rare items early, hard ones late (`Module/weighting.py:37-116`).
- **Content shuffles.**
  - Level stats: at each level, draw two stat bonuses from a weighted pool without replacement and accumulate totals; excluded levels can get both (`Module/newRandomize.py:229-287`).
  - Weapon stats: uniform integers in a configured range for keyblades, fixed ranges for staffs and shields, averages for special weapons (`Module/newRandomize.py:289-326`).
  - Attack data, battle levels and music in their own modules (not read in detail).
  - Bosses and enemies via the external `khbr` library, whose spoiler output is captured into the zip (`Module/zipper.py:93-119`).
- **Retries.** Local (100 draws) and global (50 whole-seed attempts with a new seed name each time).

## 6. Entrance or area randomization
None for entrances. World order can be reversed, and some worlds can be turned off or set to vanilla through a tristate world setting (`Class/seedSettings.py:378`).

## 7. Options model
- **Typed settings.** A base class with name, group, UI label, a `shared` flag, default, tooltip, standalone label and a `randomizable` spec (`Class/seedSettings.py:80-113`). Types and declaration counts: Toggle 81, IntSpinner 65, FloatSpinner 7, SingleSelect 40, MultiSelect 15, and one each of a progression chain, a world tristate and a texture recolor setting: 211 in all (counted with `grep`).
- **Shared versus private.** The `shared` flag separates settings that affect the seed from local ones (cosmetics), so the share string can omit private settings (`Class/seedSettings.py:3618`).
- **Settings string.** Setting names sorted; toggles packed into a bit array; small selects packed one character each; other values appended with their own encoders (`Class/seedSettings.py:3618-3643`). Sorted by name, so adding a setting shifts positions; the share string therefore includes the generator version.
- **Share string.** Generator version, seed name, spoiler flag and settings string; a different local version is rejected with a message (`Module/seedshare.py:44-81`).
- **Presets.** One preset in the repo (`presets/Easy.json`, counted); users save their own. Settings are rendered into the UI from the typed declarations (inferred from `UI/` and the class fields; UI not read in detail).

## 8. Seeds, reproducibility, spoiler logs, hints
- **Seed material.** The Python global `random` is seeded with one string made of seed name, spoiler flag, UI version, settings string and boss overrides (`Module/RandomizerSettings.py:814-822`). Toggling the spoiler log therefore produces a different game, so a spoiler cannot be generated for a seed that was played without one.
- **Retry changes the seed.** A failed attempt draws a new seed name (`Module/generate.py:31-33`), so the final seed may not be the one the user typed. It is recorded in the output.
- **Floating point.** Weight curves use floats before flooring (`Module/weighting.py`), so a different language would need identical float behavior.
- **Spoiler log.** HTML spoiler plus enemy spoilers in the zip, skipped for tournament generation (`Module/zipper.py:345-468`). A hash icon sequence is written for in-game and tracker comparison (`Module/zipper.py:609-610`).
- **Hints.** Seven documented hint systems (counted under `docs/hints/`: boss, path, points, progression, spoiler, and two named after community members), written into in-game journal text and a file the tracker reads (`Module/hints.py:193-385`).
- **In-game checker.** The validator can generate a Lua script that checks locations in game (`Module/seedEvaluation.py:414-450`).

## 9. Game integration and delivery
- **Method.** The seed is a zip mod with a mod manifest, edited game archives, Lua scripts and assets, installed through the OpenKH Mods Manager on top of a required base mod and a Lua hook (`docs/contributing/index.md`, `docs/setup/Standalone-SeedGenerator/index.md`).
- **Player steps per new seed.** The setup guide lists 7 numbered steps from opening the generator to "Build and Run" (counted, `docs/setup/Standalone-SeedGenerator/index.md:19-37`), after a one-time install of the mod manager, base mod and hook.
- **Mod order matters** and conflicts resolve by list order (`docs/setup/Standalone-SeedGenerator/index.md:56`).

## 10. Multiworld or co-op support
- **Own simple mode.** Each player's seed is generated and validated alone, then a chain of swaps moves selected item types (magic, proofs, story unlocks and others) between players' chest locations, 20 items by default for 2 players (`Module/multiworld.py:13-18`, `Module/generate.py:63-104`). A `multiworld.multi` file is written into each zip (`Module/zipper.py:402`). A reachability check after the swaps was not seen in the lines read (**not verified**).
- **Co-op hints** exist (`seedtests/test_CoopHints.py`). Archipelago has an in-tree `kh2` world (directory confirmed locally, not read).

## 11. Tooling and testing
- **Tests.** `tests/` has 21 files and `seedtests/` 12, including package init files (counted). Seed tests generate 500 seeds per property with names from the current time (`seedtests/seedtest.py:8-22`), so a failing seed is not reproducible from the test alone.
- **CI.** Runs compile on all packages and 3 test files on Ubuntu and Windows (`.github/workflows/test.yml:28-38`); the seed tests are not in CI (counted from the workflow).
- **Trackers.** A separate community tracker loads the seed zip for hints and auto-tracks from PC or PCSX2 memory (`docs/trackers/index.md`).

## 12. Developer ergonomics
- **Add a check.** Add a constructor call and maybe an edge in the world file; IDs are game IDs in code.
- **Add a setting.** Declare it once with type, label, tooltip and defaults; string encoding and UI follow from the declaration (inferred).
- **Docs.** A full user website in `docs/` (28 MB in our clone, mostly images; counted with `du`). A contributing page lists the related tool repositories.

## 13. Patterns worth standardizing and pain points
Patterns:
1. **Depth as a first-class concept.** Named depth categories constrain where item classes go, and difficulty curves weight by depth.
2. **Rarity by depth weighting**, separate from logic.
3. **Spoiler flag in the seed material**, a cheap anti-cheat for unsupervised races.
4. **Shared versus private settings** in one typed declaration.
5. **Version in the share string** with a clear refusal message.
6. **Seed hash icons** shown in game and in the tracker.
7. **Accumulated content** (level stats as running totals) and range-bounded stats.
8. **Generated in-game verifier** for placed checks.

Pain points:
1. Requirements as Python callables cannot be read by trackers or other tools.
2. Retrying with a new seed name breaks "same input, same output".
3. Floats in weights; time-based test seeds; seed tests outside CI.
4. Positional, sorted-name settings string tied to version.
5. Multiworld mixing without a visible global reachability check (not verified).

## 14. Mapping to our standard
Maps to existing constructs:
- World graph nodes and edges: `regions` and `connections` (01 D2.2). Requirement helpers: `defs` with `has`, `group_count` (01 D2.5). Invalid checks: `item_rule` (07 section 2.3).
- Typed settings: 01 D2.3 types (`toggle`, `range`, `choice`, `item_set`); `shared` maps to a needed "affects seed" flag on options (small addition). Validation of contradictory depths: option `conflicts`.
- Version in share string: our settings string pins generator id and version (01 D3.1). Hash icons: bundle hash display.
- Retries: our bounded, reproducible retries inside the stream (01 D3.2 item 4) replace reseeding.
- Its own multiworld: replaced by our room model.

Needs a new primitive or concept:
- **Depth.** A normative, derived per-location integer (sphere index under a reference playthrough, or declared tiers) usable in `item_rule` and weights. Shared with Souls placement windows (14).
- **`weighted_place`.** Weighted location choice for an item by (item class or rarity, location depth) from an integer weight table. `weighted_filler` (07 section 2.3) weights items, not locations by depth; this is the transpose.
- **`accumulate` or `redistribute`.** Level-up stats are a per-step weighted draw summed into running totals; a variant of the stat-budget idea in 13.
- **`range_roll`.** Uniform integer in a range per row with per-row overrides (weapon stats). Probably an option of `select` or a tiny primitive.
- **Spoiler-bound seed.** Optional rule that spoiler availability is part of seed derivation. Our commit-reveal (01 D3.3) solves a different problem; worth a note in D3.3.

Cannot be data, and why:
- Archive editing, mod manifest writing and Lua script generation: game-file formats owned by the mod side.
- Boss and enemy logic inside the external library: not read; likely data-heavy but outside this repo.

Not verified: CLI generator, UI generation details, attack data and battle level algorithms, post-swap multiworld validity.

## Sources read (paths in the clone, 2026-11-16)
- `README.md`, `docs/contributing/index.md`, `docs/trackers/index.md`, `docs/setup/Standalone-SeedGenerator/index.md`, `docs/hints/` (listing).
- `Module/generate.py` (attempt loop, multiworld generation), `Module/newRandomize.py` (content and item assignment), `Module/weighting.py`, `Module/depths.py` (function list), `Module/itemPlacementRestriction.py` (helper list).
- `Module/seedEvaluation.py` (outline), `Module/RandomizerSettings.py` (seed string, validation), `Module/seedshare.py`, `Module/multiworld.py`, `Module/zipper.py` (outputs), `Module/hints.py` (outline).
- `Class/seedSettings.py` (setting classes, counts, settings string).
- `List/location/agrabah.py`, `List/location/graph.py`; constructor and edge counts over `List/location/*.py`.
- `seedtests/seedtest.py`, `seedtests/test_ChainLogic.py` (outline), `.github/workflows/test.yml`.
- GitHub API: repository and contributors endpoints.
