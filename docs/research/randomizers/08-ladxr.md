# 08. LADXR: Link's Awakening DX randomizer with logic as Python objects

> Internal research note, 2026-11-16. Not reviewed by a different agent or a human; nothing here goes outside this repo before review under [`CLAUDE.md`](../../../CLAUDE.md). Our own writing, with citations; no code or text from LADXR is reproduced. Numbers are **counted** (grep or our own script over a clone), **reported**, **estimated**, or **not verified**.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

Source read: [daid/LADXR](https://github.com/daid/LADXR) at `4325207` (branch `master`, 2026-11-04), sparse clone of root `*.py`, `logic/`, `locations/`, `docs/`, `mapgen/`, `dungeongen/`, `patches/*.py`, `patches/bank3e.asm/multiworld.asm`, `www/*.html`, `www/js/`, `.github/`, `.tinyci`. Not checked out: `gfx/`, `music/`, most `patches/` assembly. For comparison, the copy vendored in Archipelago at `worlds/ladx/LADXR/` in our local checkout at `4f9545f7` (their code, read only; directory listing only).

## 1. Summary
- **What it is.** A ROM randomizer for The Legend of Zelda: Link's Awakening DX (Game Boy Color, English v1.0), with item, entrance, boss and enemy shuffles, an optional procedurally generated overworld and a dungeon generator.
- **Popularity.** 61 stars, 28 contributors (counted, 00-landscape and GitHub API).
- **License.** MIT (`LICENSE` opened; it carries a 2020 copyright notice). For us: read and describe; if we ever reused anything we would keep the notice and list it in `THIRD_PARTY_NOTICES.md` after an explicit decision. Nothing is copied here.
- **Maintenance.** Active (last commit 2026-11-04). Mostly one lead author by commit history (inferred, not verified).
- **Relation to Archipelago.** LADXR's own multiworld page says multiworld moved out of mainline in favor of Archipelago (`www/multiworld.html`, and the FAQ in `docs/faq.html`). Archipelago vendors LADXR under `worlds/ladx/LADXR/` with a `LADXR_LICENSE` file; the vendored `logic/` has 13 files versus 15 upstream (`dungeonChain.py` and `logic/main.py` absent in the vendored copy, by directory listing), and no `randomizer.py`, since Archipelago uses its own fill (inferred from the listing).

## 2. Architecture
- **Language.** Python 3, plus Game Boy assembly patches assembled by LADXR's own assembler (`assembler.py`, 1,195 lines).
- **Components** (README development notes and file list): `randomizer.py` (orchestration and item placers), `logic/` (world graph per area: `overworld.py` 1,060 lines, `dungeon1.py` to `dungeon8.py`, `dungeonColor.py`, `dungeonChain.py`, `requirements.py`, `location.py`, `main.py`), `locations/` (29 location-type classes such as chest, heart piece, seashell, shop, owl statue, each knowing which items may go there and how to patch the ROM for its item), `patches/` (51 Python patch modules, counted), `worldSetup.py` (entrance and boss shuffles), `itempool.py`, `settings.py`, `hints.py`, `spoilerLog.py`, `plan.py` (plando), `mapgen/` (overworld generator using wave function collapse, `mapgen/wfc.py`), `dungeongen/`.
- **Pipeline** (`randomizer.py`): seed from input or 16 random bytes; up to 1,000 attempts to build a world setup (entrances, bosses, optional generated map) whose logic has as many item spots as the item pool needs; apply plan overrides; choose a placer (random or forward); place items; generate the ROM; write a spoiler unless race mode.
- **How users run it.** Web UI at the project site, or locally: `launch_web_ui.py` serves `www/`, and `www/js/worker.js` loads **Pyodide** (Python compiled to WebAssembly) to run the generator in the browser. The CLI takes the ROM path (`main.py`). The player supplies their own ROM; the README states the expected SHA-1.

## 3. World model
- **Locations** are Python objects (`logic/location.py`): a node with a name, a dungeon number, a list of item spots (`ItemInfo` instances from `locations/`) and connections to other nodes, each with a requirement. `connect` adds both directions by default with an optional different back requirement; repeated connections are merged with OR.
- **Item spots** are typed by class (chest, dungeon chest, dropped key, heart piece, seashell, owl statue, trade item, shop item and others) and identified by ROM room ids; the spot class lists allowed items.
- **Events** are modeled as spots holding fixed logic items (for example `KeyLocation` instances that grant a key event).
- **Counts (counted by grep over `logic/*.py`, approximate because some are option-conditional or duplicated across alternative dungeon files):** 879 `Location(` constructions, 406 `.add(` spot attachments; most common spot classes: dungeon chest 90, key location (logic events) 43, chest 38, owl statue 33, dropped key 27, seashell 22, trade sequence item 22, heart piece 19. `entranceInfo.py` declares 107 entrance records (counted).
- **Start and goal.** `logic/main.py` builds the world from the area classes and exposes a start node and a goal node (the final boss), which the placers use for accessibility checks (`randomizer.py`).
- **Goals** are settings (instruments with a count, bingo variants, maze and others in `settings.py`), so the goal is chosen per seed rather than declared as a rule.
- **Formats.** No data files for logic: the world is built by running Python. Item pools are computed in `itempool.py` (441 lines).

## 4. Logic
- **Representation** (`logic/requirements.py`): small requirement classes `OR`, `AND`, `COUNT` (at least n of one item), `COUNTS` (at least n across a set), `FOUND` (at least n found, used for small keys), plain item-name strings, and Python booleans folded in at build time.
- **Helpers.** A `RequirementsSettings` object defines named requirements once per seed: what can cut bushes, attack, hit switches, stun particular enemies, light torches, and a large **trick directory** of named techniques (super jumps, pit buffers, boots bonks, hookshot clips and so on), each with a comment describing the technique. Some helpers combine items with option values, so options fold into constants when the object is built. Enemy requirements come from a table keyed by enemy type, which is how enemy randomization changes logic.
- **Logic levels.** The `logic` setting has five values: casual, normal, hard, glitched, hell (`settings.py`). Area files add extra connections under conditions on that setting; occurrences counted by grep in `logic/*.py`: 12 checks for hard-or-above, 10 for glitched-or-above, 8 to 11 for hell, 13 for not casual (approximate).
- **Requirement API.** Every requirement class can test an inventory, report which missing items would satisfy it (used by the forward placer to find "items needed for the next location"), and copy itself with renamed items (used when item names change by setting) (`logic/requirements.py`).
- **Back rules.** Connections are two-way by default; `back=False` makes a one-way drop, and `back=None` makes the return trivially open. This keeps one-way ledges and drops visible in the source.
- **Validation.** At placement time only (section 5): with the remaining pool assumed collected, every location (or only the goal, per accessibility) must remain reachable. No static checker found.
- **Example in our words.** In the first dungeon, the main room connects to the north and east rooms only once three small keys of that dungeon have been found; one chest in the main room needs either a way to defeat a small worm enemy or the shield; the entrance's first dropped key needs either a way to beat a beetle enemy or to push it back. So rules read as "item OR enemy-capability helper", and helpers absorb enemy shuffle.

## 5. Fill and placement
- **Random placer with an assumed check** (`RandomItemPlacer`): place dungeon-restricted small keys, stone beaks and boss keys (`NIGHTMARE_KEY` items) first; then repeatedly pick a random empty spot and a random allowed item; after each placement, check that new places are still explorable, that the remaining pool still fits, and that with the remaining pool assumed collected everything (or the goal) is reachable; undo on failure; give up after 10 consecutive failures.
- **Forward placer** (`ForwardItemPlacer`, used when the forward factor setting is above 0 or for the dungeon chain overworld): explore from the start with current items, take reachable empty spots, compute the items needed for the next new location, choose one with weights (the forward factor tunes front-heavy versus back-heavy seeds), place it, and repeat; start-item spots are filled first; 30 consecutive failures abort.
- **Rupees as progression.** The forward placer treats "rupees needed" as a group requirement and offers any rupee denomination still in the pool (`ForwardItemPlacer._placeItem`), a small case of currency in logic.
- **Dungeon items as filler for progression.** When nothing is required next, the forward placer prefers dungeon maps, compasses and beaks before general items.
- **Constraints.** Per-spot allowed item lists (by class), dungeon item modes (in dungeon, keysanity, removed), accessibility (all locations or beatable only).
- **Plando.** `plan.py` reads a plan text file: forced items per location (or a list of options) and item pool adjustments; `docs/plan-template.txt` lists every location and item id, and `www/plando.html` offers a web form.
- **Starting inventory** via start item spots and settings.

## 6. Entrance, door or area randomization
- `worldSetup.py`: entrance shuffle with eight modes (setting values `none`, `simple`, `split`, `mixed`, `wild`, `chaos`, `insane`, `madness`; these are LADXR's own option keys in `settings.py`), turning on progressively looser rules: shuffling connector caves, allowing overworld-to-overworld links, decoupling entrances from exits, and finally allowing several entrances to lead to one place. The mode sets three flags (inside to outside only, keep two-way, one-on-one). Extra pools for itemless, annoying and water entrances; random start location; dungeon shuffle; boss and miniboss shuffle.
- **Generated maps.** `overworld=random` builds a new overworld with wave function collapse over tile sets and room types (`mapgen/`); `dungeongen/` assembles dungeons from room templates; a "dungeon chain" mode links dungeons. This is world generation, not shuffling.

## 7. Options model
- **Settings** are declared once in `settings.py` as `Setting` objects with key, category, a one-character short key, label, description, options (each with a short code), default, an aesthetic flag and `visible_if` conditions. 50 `Setting(` declarations (counted) in 7 categories; the shipped `www/js/options.js` has 49 (counted; the file appears generated from `toJson`, possibly older).
- **Settings string** (`getShortString`, `loadShortString`): only non-default settings are written, as short keys plus short option codes; booleans are a single letter. This is the same "difference from defaults" idea as our D3.1.
- **UI generation.** The web UI reads the JSON form of the settings (`options.js`), so categories, labels, help text and visibility rules come from one definition.
- **Categories** (counted): Main 6, Items 8, Gameplay 9, Special 8, Entrances 6, Dungeon Items 4, User options 9. "User options" and settings flagged aesthetic are cosmetic and do not change placement (inferred from the flag name).
- **Mystery settings.** A separate page (`www/mystery.html`) offers randomized settings (not read in detail).
- **Conflict handling** (`validate`): a set of rules that force or disable settings with a printed warning, for example bingo goals forcing a normal overworld and full accessibility. Conflicts are auto-adjusted, not rejected.

## 8. Seeds, reproducibility, spoiler logs and hints
- Python `random.Random` seeded with the seed bytes; the seed is stored in the ROM and in output file names. **Race mode** draws one extra random number so race seeds differ from normal seeds with the same seed value, and disables spoiler logs (`randomizer.py`).
- **Spoiler logs** in text or JSON (`spoilerLog.py`), read back from the generated ROM.
- **Hints** are written into owl statue and hint book texts (`hints.py`, text ids per dungeon).

## 9. Game integration
- Generation-time ROM patch with custom assembly, user-supplied ROM checked by hash. Item spots know how to write their item into the ROM (`locations/*`).
- Steps for a player (estimated): open the web page, load the ROM, pick settings, download, run in an emulator. About 4; all in the browser.

## 10. Multiworld or co-op
- Mainline multiworld was removed in favor of Archipelago (`www/multiworld.html`). Remnants: `patches/multiworld.py` (a multiworld shop patch) and a link handler in `patches/bank3e.asm/multiworld.asm` whose header comment says it is mostly removed but still used by one feature.

## 11. Tooling and testing
- **CI** (`.github/workflows/python-parse-test.yml`): 26 generator runs with `SKIP_ROM_GENERATION` (counted), one per notable setting combination (logic levels, entrance modes, overworld modes, boss shuffles), each with a 120 second timeout. A separate `.tinyci` config runs similar commands against a real ROM on a private runner (inferred).
- **No unit tests** found in the checked-out tree; correctness rests on the placer's assumed-reachability check plus the smoke runs.
- **Seed time budget.** Every CI run passes a 120 second timeout, which doubles as a performance regression guard.
- **Debug and editors:** `explorer.py` (reachability), `roomEditor.py`, `backgroundEditor.py`, `mapexport.py` (map rendering), `checkMetadata.py` (location metadata for trackers, read by other tools, inferred).

## 12. Developer ergonomics
- **Adding a location:** add a `Location` and spot in the right `logic/` file, using or adding a spot class in `locations/` that knows how to patch it; register metadata in `checkMetadata.py`.
- **Adding a game:** not a goal; the code is LADXR-specific.
- **Adding a setting:** one `Setting` entry gives the CLI flag, the web form field, the short-string code and the JSON export; logic and patches then read it by key.
- Docs: a short README, a FAQ and tips pages; the trick directory comments are the best logic documentation.

## 13. Patterns worth standardizing and pain points
Worth standardizing:
1. **One settings declaration drives CLI, web UI, settings string and validation** (keys, short codes, categories, visibility rules, descriptions).
2. **Difference-from-defaults settings string** with one-letter keys.
3. **Capability helpers** ("can attack", "can light torches") computed once per seed, plus enemy capability tables so enemy shuffle changes logic through data.
4. **Named trick directory** with one documented definition per technique.
5. **Assumed-reachability check after each random placement**, simple and easy to explain.
6. **Generator in the browser** (Pyodide), so players never install anything.
7. **Location type knows its allowed items** (spot classes), a clean way to express item restrictions.
Pain points:
- Logic is code, so other tools must run Python (or reimplement it) to read it; Archipelago vendors the Python for that reason (inferred).
- Logic levels are scattered conditionals in area files rather than tagged alternatives.
- Conflicts silently auto-correct with a printed warning; a web user may not see the change.
- Python's RNG ties reproducibility to Python.

## 14. Mapping to our standard
| Construct | Maps to (doc 07 / doc 01) | Status |
|---|---|---|
| `Location` node, `connect` with back rule | `regions`, `connections` with `both` (D2.2) | Maps; a different back rule is two connections |
| Item spot with room id | `locations` plus ledger IDs | Maps |
| Spot class allowed items | `item_rule` (doc 07 2.3) over location tags | Maps |
| `OR`, `AND`, `COUNT`, `COUNTS` | `any`, `all`, `has n`, `group_count` | Maps |
| `FOUND` (keys found, not held) | `count` on a key item with no consumption | Maps (keys are monotone in LADXR logic) |
| `RequirementsSettings` helpers | `defs` with `opt` folding | Maps |
| Trick directory | `defs` | Better as the proposed **`trick`** concept, gated by the `logic` tier |
| Logic levels casual to hell | `opt` comparisons | Proposed **`tier`** concept (doc 07 has no ordered tier yet) |
| Enemy requirements table | `tables` plus macros keyed by a `permute`d enemy column | Maps if enemy shuffle is a `permute` |
| Dungeon keys placed first in dungeon | `place` with a region filter or **`fill_group`** | Maps |
| Accessibility all or goal | generator option (D2.7) | Standard option candidate |
| Entrance modes and flags | `rewire` with coupling and pool rules | Maps; the three flags are `rewire` parameters (`inside_outside`, `coupled`, `one_to_one`) |
| Boss shuffle | `permute` | Maps |
| Plan file (plando) | platform plando | Maps as a placement table |
| `Setting` declarations with `visible_if` | options with `depends` (D2.3) | Maps |
| Validate with auto-adjust | option `requires`, `conflicts` | Maps; we reject instead of silently adjusting |
| Short settings string | D3.1 settings string | Maps |
| WFC overworld, dungeon generator | `graph_template` (doc 07 2.3) | **Cannot be data**: tile-level generation is an algorithm with its own outputs; at most a module |
| ROM patches, assembler, spot patch code | CPK (D4.1) | **Cannot be data** in the world file |
| Race mode extra draw | D3.3 race seeds | Ours is a separate hidden-spoiler mode, not a changed stream |
