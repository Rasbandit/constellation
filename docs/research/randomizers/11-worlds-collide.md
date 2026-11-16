# 11. Final Fantasy VI Worlds Collide: architecture and patterns audit

> Internal research note, 2026-11-16. Read at commit `c3e25b833e9999a700eb11cdee0fe9805e5cf1a5` (2026-05-29) of `main` of the active fork [ff6wc/WorldsCollide](https://github.com/ff6wc/WorldsCollide), compared at the top level with commit `3f3935864ea799cf53f2ad839f3f6c8106c9126c` (2022-07-01) of the original [AtmaTek/WorldsCollide](https://github.com/AtmaTek/WorldsCollide). Shallow sparse clones of `args/`, `objectives/`, `event/`, `data/`, `constants/`, `settings/`, `api/`, `metadata/`, `menus/`, `log/` and top-level files; `graphics/` (1,324 files, counted) was left out. No code was run. Not reviewed by a different agent or a human. Our own writing with citations; no code, text or data from the project is reproduced. Numbers are labeled **counted** (`grep`, `find`, `git ls-tree`, or the GitHub API on 2026-11-16), **estimated**, or **not verified**. Paths are relative to the fork's root.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. Summary
- **What it is.** An "open worlds" randomizer for Final Fantasy VI (SNES, released in the US as Final Fantasy III). The player starts with the airship and both worlds open, recruits characters and espers and finds items at randomized checks, and completes configurable objectives before the final fight (`README.md`). It is a content and event randomizer more than an item-logic randomizer.
- **Popularity.** 13 stars and 3 contributors on the original, 10 stars and 5 contributors on the fork (counted, GitHub API; 00-landscape row 30). The community uses its own site, ff6worldscollide.com, and a wiki (README links; site not opened).
- **License.** MIT on both (GitHub API). Read and describe; any reuse carries the notice.
- **Maintenance.** The original's last push was 2024-01-27; the fork's was 2026-10-23 (counted). The fork adds `api/` (10 files), `metadata/` (4 files), `agents.md`, `llms.md` and `build-wc-flag-metadata.py` that the original's tree lacks (compared by top-level listing). No CI workflows in the fork (only `.github/CODEOWNERS`).

## 2. Architecture
- **Language.** Python 3 standard library only (`agents.md` section 1.1). Main folders: `args/` 33 files, `event/` 62, `objectives/` 94, `data/` 109, `instruction/` 20, `menus/` 18 (counted, `git ls-tree`).
- **Pipeline.** `wc.py` is short and linear: parse arguments (importing `args` runs parsing and seeding), load the ROM into `Memory`, build `Data` (characters, items, espers, shops, maps, enemies), run `Events`, then menus, battle, settings and bug fixes, then write data and memory (`wc.py:1-30`). Each event module rewrites the game's own event scripts with an instruction builder (`instruction/field`), so the randomizer is largely a script rewriter.
- **How users run it.** CLI (`python3 wc.py -i <rom>`, README) or the community website (not in this repo). `-manifest`, `-url`, `-sid` and `-nro` (no ROM output) arguments exist for site integration (`args/arguments.py:20-29`).

## 3. World model
- **Checks are events.** Each event file under `event/` is one class named after its file, discovered by reflection and instantiated in sorted file order (`event/events.py:22-41`). An event declares its reward slots with a set of allowed reward kinds, plus `character_gate()` (one character who unlocks it, or none) and `characters_required()` (default 1) (`event/event.py:36-50`).
- **Reward slots counted.** 67 slots across events (counted by `grep` of `add_reward`): 33 accept character or esper or item, 6 character or esper only, 11 esper or item, 16 item only, 1 character only. A constant asserts the character-or-esper count stays 6 (`event/event_reward.py:9`, checked in `event/events.py:170-174`), a cheap guard against silent changes.
- **Items, characters, espers.** Python data modules under `data/` (109 files) and `constants/` (`items.py` declares 256 item ids, counted from `ITEM_COUNT`). Chests are separate from events and handled by `args/chests.py` and `data/chests.py`.
- **Gates.** `constants/gates.py` lists, per character, the checks they unlock (for character gating).
- **Format.** Code and Python constants. No external data files for logic.

## 4. Logic
- **Two modes, mutually exclusive** (`args/settings.py:5-9`): `-open` (no event restrictions) and `-cg` character gating (an event is available once its gate character is recruited and enough characters are present).
- **Representation.** No rule language. The only access condition is `character_gate` plus `characters_required` per event, evaluated inside the fill loop (`event/events.py:92-120`). Chests and shops have no logic; the open airship start makes everything else reachable by design.
- **Objectives instead of item logic.** The goal is a list of up to 25 objectives (`constants/objectives/__init__.py`, `MAX_OBJECTIVES = 25`), each with up to 8 conditions (`MAX_CONDITIONS = 8`), a required count range, and a result. 13 condition types (counted in `constants/objectives/conditions.py`): recruit N characters or a specific one, find N espers or a specific one, defeat N dragons or bosses or a specific one, complete N checks or a specific one, a quest, plus none and random. 73 result types (counted entries in `constants/objectives/results.py`), for example stat boosts, learning spells, auto-status effects, or unlocking the final battle.
- **Runtime evaluation.** Conditions read game event bits (`constants/objectives/condition_bits.py`) and are checked in game by generated code (`objectives/_check_complete.py`, `_conditions_complete.py`), not in the generator. This is logic that ships to the player's console.
- **Example, in our words.** An objective might be "defeat 3 to 5 of the 8 dragons, or recruit 6 characters, any 1 of those 2 conditions; result: unlock the final tower". The generator rolls the exact counts (`objectives/objective.py:19-20`), writes the condition bits and texts, and the game tracks progress in a menu.
- **Validation.** The asserted slot count and the guarantee in the fill (below); no other checks.

## 5. Fill and placement
- **Open world** (`event/events.py:151-165`): shuffle all reward slots; first fill slots that accept only one kind; then slots that accept only characters or espers; then the rest, falling back to items when characters and espers run out (`choose_reward`, `event/event_reward.py:38-56`, which asserts an item is allowed in that case).
- **Character gating** (`event/events.py:92-149`): start from the starting party; repeatedly collect every empty character-capable slot whose gate character is already recruited and whose required count is met, pick one, assign a random unrecruited character, record that character's unlock path, repeat until all characters are placed. Then fill the remaining slots as in open world.
- **Bias correction.** A slot's pick weight halves for each round it has already been available, so that slots unlocked early do not collect most characters (`event/event_reward.py:58-90`, with a long explanatory comment from the original author). This addresses the breadth-first skew of forward gated fill. We have not seen this correction elsewhere in the audit so far.
- **Pools and starting inventory.** `args/starting_party.py`, `args/starting_gold_items.py`; chest contents shuffled, tiered random, scaled by chests opened, or empty, plus monster-in-a-box options (`args/chests.py:10-23,33-48`).
- **Plando.** None found.

## 6. Entrance, door or area randomization
- None. The world map is open from the start; events are randomized in what they give, not where they lead.

## 7. Options model
- **Flags.** 238 `add_argument` calls across 31 group modules (counted, `args/`). Groups are listed in one place (`args/arguments.py:4-13`). Each module implements `parse`, `process`, `flags` (its part of the flag string) and `options` (labels for the log and menus) (`args/settings.py`).
- **Flag string as the settings string.** The concatenated short flags (for example `-cg -sl -ccrt`) are the shareable settings. Objective flags pack result, ranges and conditions into dotted integers (`-oa` to `-oy`, `args/objectives.py:4-7,40-60`).
- **UI metadata.** `build-wc-flag-metadata.py` and `metadata/flag_metadata_writer.py` walk the argparse groups, including mutually exclusive groups, and emit JSON metadata, which the website can use to render its form (inferred from purpose; site not read). `metadata/objective_metadata_writer.py` does the same for objective and condition types.
- **Conflict validation.** argparse mutually exclusive groups plus per-module `process` checks; min and max pairs are swapped if reversed (`args/arguments.py:69-75`).

## 8. Seeds, reproducibility, spoiler logs, hints, solver and difficulty estimation
- **Seed.** 12 random lowercase letters and digits from `secrets` (`seed.py:1-6`). Python `random` is seeded with **seed plus flag string** (`seed.py:8-14`, `args/arguments.py:36-52`), with graphics flags excluded, so changing a gameplay flag changes the whole game: a racer cannot quietly change settings and keep the same placements (the code comment says this is the purpose).
- **In-game hash.** A sprite hash from seed, flags and version (`args/arguments.py:54-55`; `sprite_hash.py` picks rarely seen character sprites to display it).
- **Flags in game.** The pre-game menus show flags and objectives (`menus/flags.py`, `menus/pregame*.py`); `-hf` hides the flags menu and the log (`args/arguments.py:30`).
- **Spoiler log.** `-sl` adds event rewards and other details to the generated log (`args/settings.py:12-13`; `event/events.py:53-61`). No hints system, solver or difficulty estimate found.
- **Manifest.** `api/get_manifest.py` returns version, flags, hash and seed id for the site.

## 9. Game integration and delivery
- **Output.** A full modified ROM file written next to the input (`args/arguments.py:57-66`); no patch format in the repo. The website's delivery path is not in this repo (not verified). The fork's `api/` exposes sprite, palette and portrait bytes for site previews.
- **Base file check.** `valid_rom_file.py` checks the input (the `agents.md` playbook says an unheadered 3 MiB US 1.0 ROM).
- **Steps for a player** via CLI: get Python, run one command, load the output in an emulator (estimated).

## 10. Multiworld or co-op support
- None in this repo. 00-landscape notes Archipelago forks named WorldsCollide-AP (not opened).

## 11. Tooling and testing
- No test directory or CI (counted). `agents.md` and `llms.md` instruct AI coding agents how to run seeds, isolate outputs and debug, and require updating those files when patterns change (`agents.md` header). That is an unusual and useful contributor aid.
- Logging via `log/format.py` sections.

## 12. Developer ergonomics
- Adding a check: write one event file whose class name matches the file name, declare rewards and gate, and implement `mod()` to rewrite the event script; it is picked up automatically (`event/events.py:27-41`). Adding a flag: a group module with the four functions.
- Adding an objective condition or result: a type entry in `constants/objectives/`, a class in `objectives/conditions/` or `results/`, and in-game check code.
- Contributors: 3 plus 5 (counted).

## 13. Patterns worth standardizing and pain points
- **Objectives as data: condition type, count range, any-N-of-M, result.** A goal model richer than "collect items", generated per seed and shown in game. Fits racing formats (bingo-like, "3 of 5").
- **Seed RNG keyed on seed plus gameplay settings,** with cosmetic settings excluded. A simple anti-tamper rule.
- **Reward slots with allowed kinds** (character, esper, item), filled most-constrained first.
- **Age-weighted slot choice** to keep forward gated fill fair.
- **Auto-discovered check modules,** one file per check.
- **Pain points.** No logic beyond character gates, so any future item-gated mode needs a real rule system. Output is a full ROM file, not a patch. Fill order depends on sorted file names and Python's `random` across modules. No tests or CI to catch regressions.

## 14. Mapping to our standard
**Maps to existing doc 07 primitives and doc 01 concepts**
- Events with reward slots map to `locations` with `tags` for allowed kinds; characters, espers and items map to `items` with `groups` (01 D2.2). Allowed kinds map to `item_rule` (07 2.3).
- Character gates map to location `rule` with `has` (gate character) and `group_count` (characters required).
- Open versus gating maps to a `choice` option. Flags map to typed options and the flag string to our settings string (01 D3.1). Mutually exclusive groups map to `conflicts`. Chest content modes map to `weighted_filler` and `select` over tiers.
- Seed plus settings as RNG key matches 01 D3.1, where the settings string already includes options and seed; cosmetic options must be excluded from the hash input, which we should state explicitly.

**Needs a new primitive or concept**
- **`objectives` (candidate concept in 01 D2.2, extending `goal`):** a list of objective templates, each with condition kinds over existing expressions (`group_count`, `has`, `reach_event`), a seeded count range, an any-N-of-M rule, and a result key the game applies. Rolled at generation, written to slot data, shown by the client. Evidence: `constants/objectives/*.py`, `objectives/objective.py:10-25`. VARIA has objectives too (`utils/objectives.py`, doc 10), so two projects so far.
- **`fair_forward_weighting` (candidate option on a gated fill primitive):** weight candidate slots by how many rounds they have waited. Evidence: `event/event_reward.py:58-90`.
- **`cosmetic` option flag (candidate):** marks options that never enter the seed hash. Evidence: `args/arguments.py:41-46`.

**Cannot be data, and why**
- Event script rewriting (`event/*.py` `mod()` methods), menu code, battle and AI changes, and in-game objective tracking are game code. Objective results such as "learn all spells" are behaviors the game runs; the world can name them, but their effect lives in the patcher and game.

## Appendix A. Not verified, and open questions
- How ff6worldscollide.com runs generation and delivers files (server patch, browser patch or full ROM download); the site was not opened and its code is not in this repo.
- Whether the flag metadata JSON is what the site's form is built from (inferred from the writer's purpose).
- How the Archipelago forks named WorldsCollide-AP map reward slots to multiworld locations (not opened).
- Whether the fork changed the fill algorithm since the original (only top-level listings were compared; `event/events.py` was read only in the fork).
- Generation time: not measured.

## Appendix B. Files read
- `README.md`, `agents.md` (first 60 lines), `wc.py`, `seed.py`, `sprite_hash.py` (first 20 lines)
- `args/arguments.py`, `args/settings.py`, `args/objectives.py:1-60`, `args/chests.py:1-50`
- `event/events.py`, `event/event_reward.py`, `event/event.py:30-50`, `event/zozo.py:1-40`
- `objectives/objective.py:1-60`, `objectives/_check_complete.py:1-15`
- `constants/objectives/__init__.py`, `constants/objectives/conditions.py:1-40`, `constants/objectives/results.py:1-20`, `constants/gates.py:1-20`, `constants/items.py:1-5`
- `metadata/flag_metadata_writer.py:1-40`, `metadata/objective_metadata_writer.py:1-30`, `build-wc-flag-metadata.py`, `api/get_manifest.py`
- Directory listings of `data/`, `event/`, `objectives/`, `menus/`, `api/` in the fork and the top level of the original

## Appendix C. The character-gated fill, step by step (our words)
1. Shuffle all reward slots across all events.
2. Fill every slot that allows exactly one kind (this includes the starting party slots).
3. Record the characters now available (the starting party).
4. Collect the slots that can hold a character, are still empty, whose gate character is available, and whose required character count is met.
5. Give each such slot an age: how many rounds it has already been a candidate.
6. Pick one slot with weight halving per round of age, place a random unrecruited character there, and mark that character available.
7. Repeat from step 4 until every character is placed.
8. Fill the remaining character-or-esper slots, then every remaining slot that allows items.
9. Each step draws from the one global Python `random` stream seeded by seed plus flags.
This is a forward fill: it never needs to undo, because characters only unlock and never block, and the open airship makes the rest reachable.
