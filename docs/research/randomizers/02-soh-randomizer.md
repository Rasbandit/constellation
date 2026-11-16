# 02. Ship of Harkinian randomizer (HarbourMasters/Shipwright, randomizer code only)

> Internal research note, 2026-11-16, read at commit `cbfcd827256963df18978ce74ac41994366c1bf7` of [HarbourMasters/Shipwright](https://github.com/HarbourMasters/Shipwright) (default branch, shallow sparse clone). Not reviewed by a different agent or a human. Our own writing with paths and links; no code, text or data from the project is reproduced. Numbers are **counted** (by a small script of ours over the clone, or by `grep`), **estimated**, or **not verified**. Path shorthand: `R/` means `soh/soh/Enhancements/randomizer/`; `N/` means `soh/soh/Network/`.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. Summary
- **What it is.** Ship of Harkinian (SoH) is a native PC port of Ocarina of Time built from the decompilation. Its randomizer is part of the game executable: settings, seed generation, tracking and plando all run inside the game's own menu.
- **Popularity.** 6,077 GitHub stars and 191 contributors for the whole Shipwright repo (counted on 2026-11-16, from [00-landscape.md](00-landscape.md)). The randomizer has no separate star or contributor count.
- **License.** No license at the repo top level. `R/3drando/LICENSE.md` carries the MIT notice for code derived from OoT Randomizer (through the OoT3D randomizer lineage, per 00-landscape). We treat everything outside that notice as all rights reserved: read and describe at a high level only, never reuse. Even the MIT-covered 3drando files are only described here.
- **Maintenance.** Very active: the commit read is dated 2026-11-16, the same day as this note.

## 2. Architecture
- **Language.** C++ (randomizer) inside a C and C++ game port; UI with Dear ImGui through the port's own widget layer (`soh/soh/SohGui/SohMenuRandomizer.cpp`).
- **Components** (all in `R/` unless noted):
  - Static tables: `static_data.cpp`, `location_list.cpp`, `item_list.cpp`, plus one file per check category (`ShufflePots.cpp`, `ShuffleGrass.cpp`, `fishsanity.cpp` and about 17 more) that both registers its locations and implements the in-game behavior for them.
  - World graph: `location_access.h`/`.cpp` and 34 region files under `location_access/` (22 overworld, 12 dungeons, plus `root.cpp`, counted).
  - Logic helpers: `logic.h`/`logic.cpp` (a `Logic` object holding the item state and the named helper predicates).
  - Settings: `settings.cpp` (option catalog, UI callbacks, finalization), `option.cpp`, `tricks.cpp`.
  - Generator pipeline: `3drando/playthrough.cpp` (entry), `3drando/fill.cpp`, `3drando/item_pool.cpp`, `3drando/starting_inventory.cpp`, `3drando/shops.cpp`, `entrance.cpp`, `3drando/hints.cpp` and `hint.cpp`, `3drando/spoiler_log.cpp`.
  - Seed context: `SeedContext.cpp` holds the per-seed state, parses a spoiler file back into a playable seed and resolves what item a check gives at runtime.
  - Runtime: `hook_handlers.cpp`, `randomizer.cpp`, `savefile.cpp`, `RCToRandInf.cpp` (check to save-flag mapping), draw code.
  - Tools: check, item, entrance and hint trackers (`randomizer_*_tracker.cpp`), `Plandomizer.cpp`.
- **Pipeline** (`3drando/playthrough.cpp`, `3drando/fill.cpp` `Fill()`): seed the RNG from the seed number; finalize settings (resolve random values, coerce incompatible ones); build a settings string from option texts and hash it with the seed into the final RNG seed; then up to 10 attempts (counted in `Fill()`) of: rebuild the region table, build location and item pools, starting inventory, excluded locations, entrance shuffle, shops and prices, dungeon rewards, own-dungeon items, songs, other restricted dungeon items, the "Link's Pocket" reward, remaining progression, then junk. A successful attempt computes the playthrough, required items ("Way of the Hero"), barren areas, item overrides, entrance overrides and hints, and writes the spoiler file.
- **How users run it.** Only through the in-game menu: pick settings (or a preset), optionally type a seed, press Generate (generation runs on a background thread, `R/randomizer.cpp` around line 936), then create a new save file at the file select screen. There is no CLI, web generator or standalone library in this tree (searched, none found).

## 3. World model
- **Identifiers are C++ enums** generated from X-macro lists in `R/randomizerEnums/` (counted with our script): checks `RandomizerCheck` 3,485 entries, items `RandomizerGet` 328, regions `RandomizerRegion` 1,098, logic values and events `LogicVal` 353, setting keys 312, option values 259, tricks 276, hint text keys 1,673, save flags `RandomizerInf` 3,022. Counts include sentinels and Master Quest (MQ) variants.
- **Locations.** One table entry per check: key, quest variant (vanilla, MQ or both), check type, area, actor id, scene, actor parameter, flag, display name, hint text key, vanilla item, and a few flags. Entries are spread across 20 files: `location_list.cpp` holds 746 and the category files hold the rest; 3,478 `locationTable[` assignments in total (counted), close to the enum count.
- **Items.** One table entry per item in `item_list.cpp` (472 `itemTable[` lines, counted, some are variants): key, display name in English, French and German inline, item type, game item id, progression flag, the logic value it sets, hint key, model and chest data, category, and articles per language.
- **Regions and events.** `areaTable` is a fixed array indexed by region enum. Each region has a name, scene, whether time of day can pass there, the areas it belongs to (used for hints and barren), events, locations and exits. Region files hold 1,091 region definitions, 3,747 location access rules, 2,696 exits and 461 event rules (counted).
- **Variants.** Each dungeon is vanilla or MQ per seed; an entry region per dungeon routes to the vanilla or MQ graph based on that choice (`location_access/dungeons/deku_tree.cpp`, first region).

## 4. Logic
- **Representation: code.** Each location, exit and event rule is a C++ lambda written through a macro. The macro also stores the rule's source text as a string, which the check tracker shows as a tooltip (`R/location_access.h`, `R/randomizer_check_tracker.cpp` around line 2349). So rules are compiled code, with a human-readable echo for players.
- **Helpers.** 118 `Logic::` predicates in `logic.cpp` (counted): "can use item X" (which folds in age restrictions), "has item", small key counts per dungeon, "can pass enemy E", "has fire source", "can cut shrubs" and similar. Events are boolean logic values set when their rule first holds.
- **Age and time of day.** Each region tracks four reachability bits: child by day, child by night, adult by day, adult by night. Rules are evaluated under the age currently being explored; a helper evaluates a rule as "any age that can reach this region", so one age can open a path for the other (for example, adult breaks rocks that child then walks past). Time passes only in regions marked for it. The Spirit Temple has a dedicated per-region table of key thresholds for each age with forward and reverse entry (`SpiritLogicData`, `R/location_access.h`).
- **Tricks and glitches.** 202 active trick settings and 15 commented out (counted in `settings.cpp`), each tagged with a difficulty (novice, intermediate, advanced, expert, extreme), optional experimental and glitch tags, a quest variant, an area and a short code. On the first line of active entries: novice 102, intermediate 63, advanced 24, expert 8, extreme 3, glitch 9 (counted, approximate because tags can wrap). There is no separate glitched logic graph; glitches are tricks.
- **Levels.** Glitchless logic plus tricks, and a "No Logic" switch that makes every rule true. "All Checks Reachable" decides whether fill keeps placing with logic after the goal is beatable.
- **Validation.** Only at generation time: the fill's reachability search and the final playthrough. No unit tests for rules were found (section 11).
- **Example, in our words.** In the Deku Tree lobby, a spider web drops you to the basement. An event "web cleared" becomes true if the player has any fire source; a separate region on the third floor can also set the same event if the player can get past a big spider. The exit to the basement requires that event, so two different routes unlock the same edge.

## 5. Fill and placement
- **Algorithm.** Assumed fill in stages (`3drando/fill.cpp`). For each restricted pool: shuffle the items, then for each item assume every item not yet placed, run a reachability search over the allowed locations, and place the item at a random reachable empty spot. If nothing is reachable, clear that stage's placements and retry the stage, up to 10 times; then the whole seed retries, up to 10 times (counted). With "All Checks Reachable" off, once the game is beatable the remaining progression items are placed at random.
- **Stage order** (from `Fill()`): shop items, dungeon rewards, own-dungeon items, songs (song spots or reward spots), other dungeon items with restricted pools (any dungeon, overworld), Link's Pocket, all other progression, then a fast random fill for the rest.
- **Constraints.** Expressed as which location list each stage receives; excluded locations are filled with junk first.
- **Item pool.** Built in `3drando/item_pool.cpp` from settings; the pool scarcity option has four levels (plentiful, balanced, scarce, minimal, `settings.cpp` line 1408). Ice traps get a disguise model and fake name chosen from the seed RNG (`SeedContext.cpp` `CreateItemOverrides`).
- **Progression balancing.** None; this is a single-world generator.
- **Starting inventory.** Mostly one option per starting item (65 `RSK_STARTING_` option registrations in `settings.cpp`, counted; this includes starting age and other non-item starts), applied at save creation (`R/savefile.cpp` `SetStartingItems`).
- **Plando.** An in-game editor (`R/Plandomizer.cpp`) that loads an existing spoiler file, lets the user change items, ice trap disguises, prices and hint text per location, and writes the file back. It works on finished seeds; we found no logic re-check in it (2 mentions of "logic" or "beatable", counted), so edited seeds can be unbeatable (inferred).

## 6. Entrance randomization
- **Method** (`R/entrance.cpp`). Entrances have a type (owl drop, spawn, warp song, blue warp, dungeon, boss, interior, grotto, overworld and reverse forms; 28 values counted in `entrance.h`). Pools are built per type, optionally mixed. Placement assumes target entrances are reachable, connects pairs, validates the world, and rolls back on failure.
- **Coupling.** Coupled by default (a forward entrance and its reverse are bound two-way); a "decoupled" option breaks the binding, and "mixed pools" merges types.
- **Validation** (`ValidateWorld`, around line 793). After each placement: all locations reachable when required; a valid starting area with refills reachable with no items; time can pass for both ages with no items; the Temple of Time reachable as the other age; and, when coupled, a few named entrances must never be reachable as the wrong age. Entrance shuffle runs before item fill, with temporary shop shields in the pool so the world can be validated.
- **Options.** Separate switches for dungeons, bosses, Ganon's tower, overworld, interiors, Thieves' Hideout, grottos, owl drops, warp songs and spawns (`settings.cpp` lines 280 to 371).

## 7. Options model
- **Catalog in code.** `Settings::CreateOptions` registers each option with a key, display labels for each value, category, the console variable (CVar) that stores the user's choice, widget type and default. 157 enumerated and 155 boolean options plus 202 tricks (counted, `OPT_U8`, `OPT_BOOL`, `OPT_TRICK` lines). Excluded locations are one option per location, grouped by area.
- **UI generation.** The settings menu is drawn from that catalog. 52 per-option callbacks (counted) hide, disable or explain options when others change, for example disabling "All Checks Reachable" when "No Logic" is on.
- **Conflicts.** Handled twice: UI callbacks, and imperative coercion in `Context::FinalizeSettings` (`settings.cpp` around line 2644), which also adapts to what game data the player owns (if only MQ data exists, all dungeons become MQ).
- **Presets.** JSON files of CVar values grouped into blocks, with a built-in flag and a config version (`soh/soh/Enhancements/Presets/Presets.cpp`; 6 randomizer presets of 12 total shipped under `soh/assets/custom/presets/`, counted). A "randomize all settings" action also exists.
- **Settings string.** None. Settings travel inside the spoiler file, keyed by option display name with the chosen value's display text.

## 8. Seeds, spoilers and hints
- **Seed.** A user seed text (up to 1,023 characters, or 10 random digits by default) is hashed to a number; the final RNG seed is a hash of that number plus the concatenated display texts of every option (`3drando/playthrough.cpp`). When spoilers are disabled the build version is folded into the hash, so race seeds need the same build. Because labels are hashed, renaming an option's label would change seeds from the same input (inferred, not tested).
- **RNG.** One global PRNG state behind helpers for range, double, element and shuffle (`R/rng.h`, implementation in `ShipUtils`, not read).
- **Spoiler file as the seed artifact.** The generator writes a JSON file with version, git branch and commit, seed and final seed, file hash icons (5), settings, excluded locations, enabled tricks, MQ dungeons, trials, entrances, playthrough by sphere, locations and hints (`3drando/spoiler_log.cpp`). Loading this file (drag onto the window) is how a seed is shared and replayed; `SeedContext::ParseSpoiler` rebuilds the seed from it. Names, not numeric ids, are the keys.
- **Hints.** Gossip stone hints use one of four distribution tables (useless, balanced, strong, very strong) with weights and copies per hint type: required-item area ("Way of the Hero"), barren area ("Foolish"), song, overworld, dungeon, named item, random (`3drando/hints.cpp` near line 153). Hints go only to gossip stones reachable before the hinted item (inferred from `GetAccessibleGossipStones`). Static hints cover altar texts, bridge and Ganon requirements, and optional per-NPC hints toggled one by one (for example the skulltula house rewards).

## 9. Game integration
- **No ROM patch.** The player supplies their own ROM; the port extracts its assets into archive files (`docs/BUILDING.md`; 19 supported ROM hashes listed in `docs/supportedHashes.json`, counted). The randomizer then changes game behavior from inside the executable.
- **Generation time** produces only the spoiler JSON and in-memory context. **Runtime** does the rest: when the game would grant an item, the seed context returns the placed item's entry, with ice traps drawn as their disguise (`SeedContext.cpp` `GetFinalGIEntry`). 24 hook registrations in `hook_handlers.cpp` (counted) plus each category file's own draw and behavior hooks. Each check maps to a save flag through `RCToRandInf.cpp`; trackers persist their own sections in the save through the port's save manager.
- **Cosmetics** live in the port's general enhancements, not in the randomizer (not audited).
- **Player steps** (estimated): install the port, point it at a ROM once, open the randomizer menu, generate, start a new file. About 5 steps, all in one program.

## 10. Multiworld and co-op
- **Anchor** (`N/Anchor/`, 29 files, 3,229 lines counted): online co-op, not multiworld. Players with the same team id share items, flags, dungeon item counts, checks and discovered entrances; they must load the same spoiler file, and different teams may use different seeds (in-game instructions, `N/Anchor/Menu.cpp` around line 221).
- **Protocol.** TCP via SDL_net with JSON packets terminated by a NUL byte (`N/Network.h`). Packet types (22 files in `Packets/`, counted) include handshake (room id, room state, client state), give item (mod id and game item id), set and unset flag, set check status, entrance discovered, player position and sound effects for ghost players, damage, teleport, game complete and team state. Default host is a project-run server; the server is not in this repo.
- **Archipelago.** Not on `main`. A separate repo, [HarbourMasters/Archipelago-SoH](https://github.com/HarbourMasters/Archipelago-SoH) (70 stars, GitHub reports it as a fork of Archipelago, default branch `oot-soh`, pushed 2026-11-02, checked with `gh api`), holds a Python world `worlds/oot_soh/` (79 paths, counted) with its own regions, logic helpers, items, options and presets. That means SoH's logic exists twice, in C++ and Python. Its client writes connection details into the port's config file (`worlds/oot_soh/Client.py`, first lines read); how the game side connects was not verified.
- Crowd Control and a "Sail" remote module also exist under `N/` (not read).

## 11. Tooling and testing
- **Built in:** check tracker with the logic tooltip, item tracker, entrance tracker, hint tracker, plandomizer, performance timers per fill phase, a bulk seed generator for testing (`Playthrough_Repeat`), and a world graph dump to Graphviz with per-region age and time flags (`location_access.cpp` around line 1059).
- **Tests.** No randomizer test files found in the clone (searched for test directories and files, none). CI (`.github/workflows/`, 6 files counted) builds on several platforms and distros, checks formatting, whitespace and JSON lint; none runs seed generation (inferred from step names).

## 12. Developer ergonomics
- **Adding a check** touches, by our reading: the check enum, a location table entry (in `location_list.cpp` or a category file), a rule line in a region file, a hint text key, a save-flag enum entry and its mapping in `RCToRandInf.cpp` (which asserts if missing), tracker area data, and runtime code to intercept the vanilla reward. About 6 to 8 places (estimated).
- **Adding a setting:** a key enum, an option registration, often a callback, finalization rules, and the logic or runtime code that reads it.
- **Docs.** `docs/` covers building, modding and versioning; no randomizer developer guide was found. Comments in code are the main documentation.
- **Contributors.** 191 for the whole repo (counted, 00-landscape); randomizer-only count not measured.

## 13. Patterns worth standardizing, and pain points
**Worth standardizing**
1. **Rule text kept beside the rule.** Showing the rule to players in the tracker is useful; a data-first format gets this for free and can render it nicer.
2. **One module per check category** that owns its locations, rendering and runtime behavior; adding "grass shuffle" is one file plus a setting.
3. **Tagged trick catalog** with difficulty, area and short code; drives filtered UI and presets.
4. **Hint distributions as weighted tables** with per-type copies and location filters.
5. **ER validation invariants** stated as reachability with an empty inventory (start area, time passing, returning through time).
6. **Spoiler file as the replayable seed artifact,** including build commit, so a seed is portable without a server.
7. **Generation inside the game**, so a player never handles a patch.

**Pain points (design costs, in our words)**
1. Logic is compiled C++, so a tracker, a web generator or the Archipelago world must reimplement it; Archipelago-SoH shows the cost (a second logic in Python).
2. Identity is by enum order and display names; the spoiler and presets key on names, so renames can break old files (inferred).
3. Option labels feed the seed hash, coupling UI wording to reproducibility.
4. Conflicts are resolved by silent coercion in code, not declared constraints.
5. Retries are fixed counts with no reported cause to the user beyond logs.
6. No automated tests on generation in CI, so logic regressions surface in play.

## 14. Mapping to our standard
**Maps to doc 07 primitives and doc 01 concepts as they exist**
- Regions, exits, locations, events: `regions`, `connections`, `locations`, `events` (01 D2.2).
- Helper predicates: `defs` with `ref` (01 D2.5); "has", key counts: `has` with `n`, `count`.
- Item pool scarcity: item `count` as a rule over `opt`.
- Excluded locations: a `location_set` option plus location `kind: excluded` (01 D2.3, D2.7).
- Restricted pools (own dungeon, songs on song spots): `item_rule` plus `place` (07 section 2.3).
- Entrance shuffle by type, coupled or decoupled, mixed: `rewire` with coupling rules.
- MQ or vanilla per dungeon: `select` over a variants table plus `opt`-style conditions on region entry.
- Settings UI from a catalog, presets: options and `presets` (01 D2.3); conflicts become `requires` and `conflicts`.
- Anchor's same-seed team sharing: close to an Archipelago-style shared slot over CWP's event log (01 D1.4); whether CWP allows several clients on one slot is **not verified** in doc 01.

**Need a new primitive or concept**
- `access_facets` (concept): a world declares a small set of facets (here age and time of day) and each region tracks reachability per facet. Rules may ask "any facet that reaches here", and some regions allow moving between facets (time passing, the pedestal). Without it, worlds split every region per facet by hand.
- `fill_stages` (concept): an ordered list of fill passes, each with an item filter and a location filter, run before the general fill. Covers shops, rewards, own-dungeon keys, songs, pocket item.
- `hint_distribution@1` (primitive): weighted hint types with copies, filters over location groups and platform-computed facts (required, barren), placed only at hint spots reachable before the hinted item.
- `rewire` invariants: named rules that must hold with an empty inventory after every placement (start area, facet changes), plus forbidden facet reach for listed entrances.
- `trick` option kind: a toggle with tags (difficulty, glitch, area) and a short code, so trick catalogs and filters are uniform across worlds.
- `price_roll` (maybe composable from `select`): seeded prices from a weighted distribution per location group.

**Cannot be data, and why**
- Runtime item substitution, actor drawing, ice trap disguises and save flags: game code (doc 07 section 6.8, client side).
- The Spirit Temple key thresholds by age and entry direction are hand-derived; they could be data as tables, but deriving them is analysis, not declaration (inferred).
- Settings that adapt to what game data the player owns depend on the local install; that belongs in the client handshake, not the world.

Not read: `ShipUtils` PRNG implementation, Anchor server, Crowd Control and Sail modules, port cosmetics, and the item pool function body.
