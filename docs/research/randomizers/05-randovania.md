# 05. Randovania: a multi-game framework with a structured logic database and editor

> Internal research note, 2026-11-16. Not reviewed by a different agent or a human; nothing here goes outside this repo before review under [`CLAUDE.md`](../../../CLAUDE.md). Our own writing, with citations; no code, text or database content from Randovania is reproduced. Numbers are **counted** (by our own script over a sparse clone), **reported** (a number the project states in its own files), **estimated**, or **not verified**.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

Source read: [randovania/randovania](https://github.com/randovania/randovania) at commit `97f748d` (branch `main`, 2026-11-16), shallow sparse clone of `randovania/game_description/`, `generator/`, `resolver/`, `layout/`, `network_common/`, `games/*/{logic_database,pickup_database,presets,generator,layout,exporter}`, `games/blank/`, `docs/`, `.github/workflows/`. Not read: `gui/` (except `gui/data_editor.py` and `gui/dialog/connections_editor/` listed), `server/` internals, `game_connection/` bodies, any patcher package. Paths below are relative to the repo root.

## 1. Summary
- **What it is.** A desktop application plus server that randomizes several games from one generator, one requirement model and one data editor. 13 game folders plus a `blank` template (counted, `randovania/games/`): AM2R, Cave Story, Metroid Dread, Factorio, Metroid Fusion, Planets Zebeth, Metroid Prime, Prime 2 (two folders, `prime2` and `prime2_opr`), Prime Hunters, Samus Returns, Yoku's Island Express, Zero Mission.
- **Popularity.** 221 stars, 109 contributors (counted, from 00-landscape and the GitHub contributors API on 2026-11-16).
- **License.** GPL-3.0 (`LICENSE` header opened). For us: read and describe only. No database JSON, code or doc text enters our repo; even the data is GPL-covered work.
- **Maintenance.** Very active: HEAD commit dated 2026-11-16. A formal admission policy for new games exists (`docs/New Game.md`): an owner, a separate logic reviewer, an open-source patcher with a license allowing a fork, and test coverage; development games may be removed if they stall.

## 2. Architecture
- **Language and runtime.** Python, Qt GUI, an optional Cython build of hot paths (`generator/generator_native.py` plus `.pxd`, `resolver/resolver_native.py`; CI toggles an `enable-cython` flag in `.github/workflows/build-test-publish.yml`).
- **Components (game-agnostic core).** `game_description/` (database model: `db/` nodes, areas, regions, docks; `requirements/` requirement tree; `integrity_check.py`; `pretty_print.py`; `data_reader.py`, `data_writer.py`; `game_description_migration.py`). `generator/` (pool creation, retcon filler, dock and teleporter distributors, hint distributor). `resolver/` (a separate solver that replays a finished layout to prove beatability). `layout/` (presets, permalinks, layout description, migrations). `network_common/` and `server/` (multiworld and async races). `game_connection/` (per-game connectors and executors for emulators and mods).
- **Per-game plug-in.** Each game ships a `game_data.py` that builds one `GameData` record wiring factories for options, GUI, generator (pool creator, bootstrap, base patches factory, action weights), hints, patch data factory, exporter, hash words and test data (`games/blank/game_data.py`). Games are lazy-loaded through these factory functions.
- **Pipeline** (`generator/generator.py`, `generator/filler/runner.py`): per player, filter the database for the preset (`layout/filtered_database.py`), patch the resource database, create base patches (starting location, teleporters), distribute dock locks, place pre-fill hints, compute the pickup pool, optionally check that the base state is beatable, retry this setup up to 10 times; then run the retcon filler across all players; then distribute remaining items; then run the resolver (solo games only; the code comments that multiworld skips it) with a 600 second default timeout. Whole-generation retries: `DEFAULT_ATTEMPTS = 15` on `UnableToGenerate`.
- **How users run it.** Download the app, pick a game and preset, generate (locally or in an online session), then export: the exporter reads the player's own game files and writes a patched ISO, ROM or mod folder through a separate patcher package.

## 3. World model
- **Three location tiers:** region, area (usually a room), node (`docs/Database Format.md`). Nodes are typed: generic, pickup, event, dock, hint, teleporter network, configurable (counted node types in the JSON). Connections are one-way edges between nodes inside one area, each with a requirement.
- **Docks** pair two dock nodes across adjacent areas. A dock has a **type** (door, portal, hole) and a **weakness** (the lock: missile door, blast door). Weaknesses carry an "open" requirement and a lock requirement with a lock type that defines front and back behavior (`games/blank/logic_database/header.txt`, dock weakness section). Long-distance links are teleporter nodes.
- **Resource database** (`header.json`): `items`, `events`, `tricks`, `damage`, `versions`, `misc`, `requirement_template` (named reusable requirements), `damage_reductions`. Also `layers` (option-gated node groups), `starting_location`, `minimal_logic`, `victory_condition` (a requirement), `dock_type_database` (with per-type "dock rando" rules), `hint_feature_database`, `used_trick_levels`. `schema_version` is 35 in every game (counted).
- **Pickups** live in a separate `pickup_database/pickup-database.json`: categories, standard pickups (each granting one or more items, optionally progressive), ammo pickups, generated pickups; each carries hint features and generator parameters such as a probability offset.
- **Layers.** Each node lists the layers it belongs to (`default` plus optional ones such as `extra_pickups` in the blank game), so a preset can switch a group of nodes on or off without editing the graph.
- **Minimal logic.** The header declares which items and events to ignore and a description for a "minimal logic" mode, where the generator only guarantees a narrower set of requirements (`header.json`, `minimal_logic`).
- **Area metadata.** Areas carry hint features and a default node; nodes carry a heal flag and a spawn flag (`Intro.txt`).
- **Files and formats.** One JSON file per region plus a header, with a committed human-readable `.txt` pretty print next to each JSON for review diffs (`games/blank/logic_database/Intro.txt`). Engine-specific data goes into an `extra` field on nodes.
- **Counts across all 14 databases (counted by our script):** 105 regions, 2,760 areas, 11,814 nodes (1,927 pickup, 981 event, 7,000 dock, 1,646 generic, 125 hint, 76 teleporter network, 59 configurable), 17,894 connections, 815 items, 894 events, 187 tricks, 670 requirement templates. Largest by nodes: Dread 1,857; Samus Returns 1,509; AM2R 1,132. Factorio is an outlier: 1 region, 208 nodes, 427 templates, 0 tricks, so the same model carries a tech tree.

## 4. Logic
- **Representation.** A requirement is a JSON tree, not a string. Node kinds (counted in the data): `and`, `or`, `resource` (type plus name plus amount plus a `negate` flag), `template` (reference to a named template), `node` (requires having visited another node). Empty `and` is trivial and empty `or` is impossible (`docs/Database Format.md`). Every `and` or `or` block may carry a comment, and comments with video links serve as trick documentation.
- **Requirement node counts across all 14 databases** (counted): `resource` 29,959; `and` 22,713; `or` 7,958; `template` 8,772; `node` 344 (332 of them in Factorio, where tech prerequisites point at other nodes).
- **Resource types** (`game_description/resources/resource_type.py`): item, event, trick, damage, version, misc, node identifier. A trick requirement's amount is a **trick level**; damage resources are consumed each time a connection is crossed (the docs advise using an event when the cost is one-time).
- **Trick levels** (`layout/base/trick_level.py`): disabled, beginner, intermediate, advanced, expert, ludicrous. The preset sets a level per trick; the docs give a written rubric for assigning difficulty (consistency, penalty, retry time, discoverability).
- **Evaluation.** Requirements are simplified and flattened into lists of alternatives; each alternative uses a bitmask for single-copy items and a side list for counts, negations and damage (`game_description/requirements/requirement_list.py`). That is a DNF (an OR of ANDs) compiled to bit operations, the same idea as our "rules compile to bytecode" hypothesis (doc 07 section 5).
- **Validation** (`game_description/integrity_check.py`): dock naming conventions, duplicate pickup indices, recursive templates, inconsistent dock configuration, undocumented tricks (videos required above a configured level), unnormalized hint features, and a strongly connected component check that flags any node cluster not mutually reachable with the start unless explicitly marked.
- **Example in our words.** In the blank game, the spawn room has a door to the boss arena whose connection needs the weapon item and the missile ammo item; the boss door's own weakness is an explosive lock that needs the explosive item before the dock can be used from the front. So reaching the boss area is "connection rule AND dock lock rule", stored in two places that the generator combines.

## 5. Fill and placement
- **Algorithm: "retcon" forward fill** (`generator/filler/retcon.py`). The filler walks the game like a player: at each step it lists possible actions (collect a reachable event or unsafe resource node, or assume a pickup that unlocks new nodes), weights them by what they open (pickup nodes, events, hints, victory; dangerous actions get a 0.75 multiplier, `generator/filler/weights.py`), picks one with a seeded weighted choice, then places that pickup at an already reachable empty location, chosen with weights that favor locations that have waited less long ("index age") and regions with fewer placements. "Retcon" because the pickup is retroactively written into a location the player could already visit.
- **Multiplayer inside the filler.** `_get_next_player` chooses which player's world advances next; a pickup for one player can land in another player's location. Remaining non-progression items are spread after the main loop (`_distribute_remaining_items`).
- **Constraints.** Major/minor location categories (pickup node "Major" flag, `layout/base/available_locations.py`), "first progression must be local", per-pickup probability multipliers and offsets, excluded locations, bulk placement of like pickups.
- **Location weighting detail.** Each location's weight falls with its "age" (steps since it became reachable) by an inverse square rule capped at age 10, and with how many placements its group (a region, respecting exclusions) already holds (`_calculate_uncollected_index_weights`). This spreads items across regions and favors newly opened spots without forcing them.
- **Victory weighting.** An action that unlocks the victory condition gets a large multiplier (1000), so the filler finishes once progression is complete (`weights.py`).
- **Starting inventory and pool** come from the preset's standard and ammo pickup configuration; the filler can also decide to make a pickup a starting item (`should_be_starting_pickup`).
- **Plando.** No hand-placement feature found: a grep for "plando" over the checked-out Python found no files; the glossary describes a plandomizer only as a hypothetical hand-made layout. Not verified for the GUI folders we did not check out.
- **Validation after fill.** The resolver replays the layout to the victory condition; failure triggers a new attempt.

## 6. Entrance, door or area randomization
- **Dock lock randomization** (`generator/dock_weakness_distributor.py`, 566 lines): per dock type, the database declares which weaknesses may be changed from and to, and which are "unlocked" and "locked" defaults (`header.txt`). This randomizes the *lock on a door*, not the door's destination.
- **Teleporter and elevator randomization** (`generator/teleporter_distributor.py`).
- **Dock (door destination) randomization** exists per dock type ("Dock Rando" block in the dock type database); the details of two-way forcing are checked by `find_mismatching_dock_types_randomizable_with_forced_two_way`. Algorithm bodies not read in full.

## 7. Options model
- **Configuration** is a dataclass per game inheriting `BaseConfiguration` (`layout/base/base_configuration.py`), with shared fields: trick levels, starting locations, available locations, standard and ammo pickup configuration, damage strictness, dock weakness distribution, logical pickup placement (minimal, majors, all), hints, and several generator flags. 15 shared fields (counted) before game-specific ones. Cosmetic patches are a separate class that does not affect generation.
- **Presets** are versioned `.rdvpreset` files with migrations (`layout/preset_migration.py`); 27 shipped presets (counted).
- **Permalink** (`layout/permalink.py`): one bit-packed binary blob containing the seed, a 4-byte app version hash, an optional seed hash and each player's preset, encoded with a byte rotation and base64. Only valid for the same version (or same stable major.minor, per the glossary).
- **UI generation.** The Qt GUI has hand-written preset tabs per game (`gui/preset_settings`, not read); the UI is not generated from data.
- **Game-specific options** extend the base class per game (`games/blank/layout/blank_configuration.py`), so every game carries the shared trick, pool and location settings for free; a preset describer turns a configuration into a readable summary (`layout/preset_describer.py`).
- **Conflict validation** happens in code (`InvalidConfiguration` on pool size and starting state). Not a declarative layer.

## 8. Seeds, reproducibility, spoiler logs and hints
- RNG is Python's `random.Random` (glossary). Reproducible within one version only.
- The **layout description** (`.rdvgame`) is the spoiler log: every player's patches plus the permalink; it is versioned and migrated.
- A short **seed hash** shown as words (`hash_words.py` per game) lets players confirm they have the same game.
- Each game's test data pins an **expected seed hash** (`games/blank/game_data.py`, `GameTestData`), a golden-seed regression check.
- **Featural hints** (`docs/Hints.md`): areas, pickup nodes and pickups get "features" (for example "underwater"); each feature has a precision, and the hint distributor picks a feature by sampling a Gaussian over precision so hint vagueness is tunable per game.

## 9. Game integration
- Generation-time patching, with a strict split: a `PatchDataFactory` turns the layout into one JSON document, and a separate patcher package (pinned with `==` in `pyproject.toml`) applies it to the user's own files. The docs recommend each patcher also run standalone from a CLI and publish a JSON Schema for its input (`docs/New Game.md`).
- Runtime connection for multiworld: `game_connection/` has connector builders for Dolphin, Nintendont, sockets and several games (file list counted from the tree, bodies not read).
- Steps for a player (estimated from docs): install app, select game and preset, generate, export with own game files, launch in emulator or console. About 5 steps, plus a connector step for multiworld.

## 10. Multiworld or co-op
- Built-in server (`server/`, 43 files in tree, counted) with Discord login (`server/discord_auth.py`), sessions and async race rooms (`network_common/async_race_room.py`).
- Sync shape (`network_common/world_sync.py`): the client sends per world (keyed by UUID) its connection status, collected location indices, an encoded inventory and a beaten flag; the server answers with world and session details and per-world errors. The server, not the client, computes what each world must receive.
- The pool is shared across players at generation (retcon handles all players at once).

## 11. Tooling and testing
- **Data visualizer and editor** (`gui/data_editor.py`, 978 lines; `gui/dialog/connections_editor/` with model, view, controller and a requirement tree; `docs/Database Editor.md`): edit one area at a time, a map view for some games, create docks in pairs, rename with automatic reference updates, edit the resource database. The same tool ships read-only to players as a visualizer.
- **Tests:** 349 `test_*.py` files and 76 committed log files under `test/test_files/log_files` (counted). Acceptance tests compare outputs with committed reference files, regenerated by one command (`docs/Testing.md`).
- **CI** (`.github/workflows/`): build-test-publish (which also scaffolds a throwaway game with `add-new-game` to prove the template), a weekly per-game "daily generate" on a self-hosted runner, a trick documentation report, a benchmark workflow, linting.
- **Profiling and performance docs** (`docs/Profiling.md`), a benchmark workflow, and the optional Cython build show that generation speed is a maintained concern.
- **Debug output.** The filler prints weighted actions and player state at higher debug levels (`generator/filler/filler_logging.py`, `resolver/debug.py`); failures include a per-player state report (`runner.py`).
- **Migrations:** 29 database migration functions toward schema 35 (counted in `game_description_migration.py`), plus preset and layout migrations; the docs advise migrations not to import live code.

## 12. Developer ergonomics
- Adding a game: a CLI command scaffolds a new game folder from `blank` with naming checks (`docs/New Game.md`); then build the database (the docs recommend scripting import from the game's own files and putting engine data in `extra`), the pickup database, a configuration class, a patch data factory and an exporter.
- Adding a location: in the editor, add a pickup node, connect it, give it a pickup index; integrity checks catch naming and index mistakes.
- Docs are good on concepts (glossary of 46 entries, counted, database standards, trick rubric); multiworld integration for a new game is marked TODO in `docs/New Game.md`.
- 109 contributors (counted).

## 13. Patterns worth standardizing and pain points
Worth standardizing:
1. **Typed graph data with a first-class editor**: nodes, typed docks with lock weaknesses, templates, events, tricks and victory condition all in one data model that a GUI edits and a text dump makes diffable.
2. **Trick as a resource with ordered levels** and a written difficulty rubric, plus an integrity rule that tricks above a level must carry a video link.
3. **Golden seed hash per game** and acceptance files regenerated by one command.
4. **Patcher boundary as a JSON document with a schema**, patcher runnable standalone.
5. **Admission policy for new games** (owner, independent logic reviewer, forkable patcher, timelines).
6. **Featural hints** driven by tags with a precision value.
Pain points (design costs, not faults):
- The core is game-agnostic in design but lives inside one GPL app; there is no library API that another generator could call.
- Per-game Python classes (bootstrap, base patches, pool creator) are needed even for simple games, so the "data only" path stops early.
- Damage tracking per traversal and negated requirements make reachability path-dependent, which is why a separate resolver with a timeout exists.
- Permalinks bind to one app version, so old links stop working across releases.
- Logic for a node pair lives in two places (connection rule and dock lock), which the editor hides but a text reader must combine.
- Multiworld integration for a new game is undocumented (marked TODO), so it is learned from existing games.

## 14. Mapping to our standard
| Randovania construct | Maps to (doc 07 / doc 01) | Status |
|---|---|---|
| Region, area, node, connection | `regions`, `connections` (D2.2) | Maps, but we have one tier. Areas become regions; nodes inside an area become sub-regions. Proposed concept **`area` grouping tag** (display and hint grouping, no logic effect) |
| Pickup node, pickup index | `locations` plus ledger ID (D2.6) | Maps directly |
| Event node | `events` (D2.2) | Maps |
| `and`, `or`, `resource`, `template` | `all`, `any`, `has`, `ref` (D2.5); templates to `defs` | Maps |
| `negate` on a resource | Non-monotone rule tagged (D2.7) | Maps with the tag; should stay rare |
| Trick with level | `opt` comparison | **New concept `trick`**: a named, documented skill with ordered levels (`trick(name) >= level`), its own UI, docs links and a "documentation required above level" check. Options alone lose the docs and the rubric |
| Damage resource consumed per traversal | none | **New primitive candidate `resource_state`** (shared with HK state logic and Ori energy; see batch summary). Not expressible in D2.5 |
| Dock type and weakness with lock semantics | `connections` with rules | **New concept `door_kind`**: a connection carries a kind and a lock with front and back behavior; plus **primitive candidate `relock`** (shuffle locks within allowed sets). `rewire` covers destination shuffle only |
| Layers | `templates` `when` (doc 07 3.1) | Maps; a named **`layer`** shorthand would be friendlier |
| Victory condition | `goal` | Maps |
| Major/minor categories, logical placement modes | location `tags` plus `item_rule` | Maps |
| Starting location choice | `select` over start regions | Maps |
| Featural hints | none | **New concept `hint_features`**: tags with an integer precision on locations and items; the hint primitive samples them |
| Golden seed hash | D2.9 golden bundles, D6.1 | Maps; adopt per-world golden seeds |
| `.txt` pretty print | doc 07 4.6 formatter | Add **canonical text print** for review diffs |
| Retcon weights, action weights | CFill v1 (D2.7) | Fill policy; parameters could be declared knobs |
| Exporter, patcher, game connectors | CPK (D4.1), CSDK (D5.1) | **Cannot be data**: they rewrite files or memory. Only the patch-data JSON with its schema maps to our `slot_data` plus a schema |
