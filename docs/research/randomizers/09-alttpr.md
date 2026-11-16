# 09. ALttP Randomizer (alttpr.com, "VT"): architecture and patterns audit

> Internal research note, 2026-11-16, read at commit `219fcafd029dab597b8db400efafd8f56f8b4edb` (2024-02-18) of the `master` branch of [sporchia/alttp_vt_randomizer](https://github.com/sporchia/alttp_vt_randomizer) (shallow sparse clone of `app/`, `config/`, `tests/`, `routes/`, `database/`, `documentation/`, `resources/js/`, `strings/`; no code was run, built or installed). Not reviewed by a different agent or a human. Our own writing with citations; no code, text or data from the project is reproduced. Numbers are labeled **counted** (by `grep`, `find`, `wc` or `git ls-tree` over the clone, or the GitHub API on 2026-11-16), **estimated**, or **not verified**. Paths are relative to the repo root.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. Summary
- **What it is.** The item randomizer for The Legend of Zelda: A Link to the Past (SNES) and the web application that serves alttpr.com. Generator, seed database, permalink pages and the browser-side patcher live in one Laravel (PHP) plus Vue codebase.
- **Popularity.** 320 stars, 48 contributors including anonymous (counted via the GitHub API, 2026-11-16; 00-landscape row 14). The ALttP racing scene (00-landscape section 3) uses it as its seed host. Archipelago's in-tree `alttp` world is a separate Python lineage, not this code (inferred from language; not compared).
- **License.** MIT (GitHub API). For us: read and describe; any future reuse needs the MIT notice in `THIRD_PARTY_NOTICES.md`.
- **Maintenance.** Last push to any branch 2024-07-20; the default branch head is dated 2024-02-18 (both counted). CI config is `.travis.yml` targeting PHP 7.4 while the README asks for PHP 8.1, so the CI file looks stale (inferred; not checked whether Travis still runs). There is no `.github/workflows/` directory (counted: `.github/` holds only `ISSUE_TEMPLATE`).

## 2. Architecture
- **Languages.** PHP 8.1 on Laravel for generation and hosting; Vue 2 and TypeScript for the browser UI and patcher (`resources/js/`, 50 files counted). Largest PHP files: `app/Rom.php` 2,588 lines, `app/World.php` 1,637, `app/Randomizer.php` 1,094 (counted, `wc -l`).
- **Components.** `World` (one player's config, regions and locations; subclasses `app/World/{Standard,Open,Inverted,Retro}.php`), `Region` classes (51 files under `app/Region/`, counted), `Location` and `Item` classes, `Randomizer` (pre-fill setup and the fill call), `Filler/RandomAssumed`, `Services/{Playthrough,Hint,MultiworldFile}Service.php`, `Rom` (byte writer that records a write log), `Enemizer` and `EntranceRandomizer` (wrappers that shell out to external tools, section 6).
- **Pipeline (web).** `app/Http/Controllers/RandomizerController.php:104-223`: build a `World` from request fields, load the base ROM and apply the JSON base patch, run `Randomizer::randomize()`, write placements to the `Rom` object, check winnability (`:182-188`), build the spoiler, optionally run the enemizer, then return the write log as the seed's patch. The seed row is saved and a queued job (`app/Jobs/SendPatchToDisk.php`) writes a JSON file to storage.
- **How users run it.** Mainly the website. There is also a CLI (`php artisan alttp:randomize`, `app/Console/Commands/Randomize.php`, README "Running from the command line") and a self-hosted local web server. No desktop GUI.

## 3. World model
- **Items.** PHP objects created by name in `app/Item.php` (179 constructor calls counted, including per-dungeon key, map and compass variants such as `KeyP2`). Item subclasses carry type behavior (`app/Item/{Key,BigKey,Map,Compass,Sword,Bottle,...}.php`, 17 files plus 4 upgrade classes, counted).
- **Locations.** Declared inside each region class constructor with a location class (`Chest`, `BigChest`, `Dash`, `Drop`, `Npc`, `Prize\Pendant`, ...) and the ROM addresses it writes to. Example: `app/Region/Standard/DesertPalace.php:54-63` lists six item locations and a prize location, each with one or more ROM offsets. 234 location constructor calls under `app/Region/Standard/` (counted); `Inverted` regions mostly extend the `Standard` classes and override rules (`grep ^class` over `app/Region/Inverted/`).
- **Regions.** One PHP class per dungeon or overworld area, holding `can_enter`, `can_complete` and per-location requirement closures. Regions also own non-item data: music addresses, map reveal bits, the region's own item list for dungeon-item rules (`DesertPalace.php:18-38`).
- **Events.** Logic-only items such as `RescueZelda` are ordinary items pre-collected or placed (`app/Randomizer.php:134-136`). Win conditions are closures per goal (`World::getWinCondition`, `app/World.php:418`).
- **Format.** Code only. Item, location and region data live in PHP classes, not data files. `config/*.php` holds settings, presets, logic toggles and item pool counts as PHP arrays.

## 4. Logic
- **Representation.** PHP closures over `($locations, $items)` returning a boolean, attached with `setRequirements`, plus `setFillRules` (which items may go here) and `setAlwaysAllow` (items allowed here even when access fails). `Location::canFill` (`app/Location.php:72-85`) combines: always-allow, region-level fill rule, location fill rule, then access.
- **Helpers.** Methods on the item collection, such as `canLiftRocks`, `canLightTorches`, `canKillMostThings` (used in `DesertPalace.php:81,98,103`), and boss checks through `Boss::canBeat`.
- **Trick and glitch toggles.** `config/logic.php` maps five logic levels (`NoGlitches`, `OverworldGlitches`, `HybridMajorGlitches`, `MajorGlitches`, plus `NoLogic` in the controller) to sets of named booleans such as `canBootsClip` or `canSuperSpeed` (31 key lines counted). Rules read them with `$this->world->config('canBootsClip', false)`. So the logic level is a preset over trick flags, and the trick flags are tested inline.
- **No per-level rule files.** No file under `app/Region/` names a logic level (`grep` for the level names finds none); every difference is a trick flag tested inside the shared closures. Modes (Standard, Open, Inverted) are separate class trees instead. Tests are organized per level and mode (section 11).
- **Example, in our words.** Desert Palace's boss location needs: the region is enterable; a way up to the back entrance (lift rocks, or one of three glitch flags, or the mirror via Misery Mire); torches lit; the dungeon's big key and small key; the boss beatable; and, only when maps or compasses are shuffled outside dungeons, the dungeon's map or compass (`DesertPalace.php:96-107`). The same closure also forbids that location from holding the dungeon's own keys.
- **Validation.** No static checks; correctness comes from per-level PHPUnit tests that assert access for item sets (section 11).

## 5. Fill and placement
- **Algorithm.** Assumed fill (`app/Filler/RandomAssumed.php:25-57`). Locations are shuffled once. Dungeon items are placed first, assuming all required and nice items are held; then a random number of junk items is placed in Ganon's Tower (a range per world, `getGanonsTowerJunkFillRange`); then required items into the remaining locations in reverse order; then nice items and filler with a fast fill that skips logic.
- **Per-item step.** For each item, remove it from the assumed set, sweep all worlds to a fixed point collecting reachable items (`:88-99`), filter locations by `canFill`, then take the first fillable location (or a random one for maps and compasses, `:122-126`). Failure throws an exception saying no location is available, and the request returns HTTP 409.
- **Accessibility modes.** `accessibility` of `locations`, `items` or `none`; with `none`, access checks are skipped once the game is already winnable (`:101-106`).
- **Item pools.** `item_placement` basic or advanced, `item.pool` and `item.functionality` difficulty variants, plus a `custom` pool in the customizer. Prize (pendant and crystal) placement, medallions, shops, bosses and fountains are separate pre-fill steps in `Randomizer.php` (`placeBosses :361`, `fillPrizes :464`, `setShops :683`).
- **Starting inventory.** `World::addPreCollectedItem` (used for boots in glitched logic and `RescueZelda` outside standard mode, `Randomizer.php:122-136`) and the customizer's equipment field.
- **Plando.** The customizer (`CustomizerController.php`) accepts a location-to-item map (`l`), custom item counts and starting equipment, validates counts against the goal (`:129-140`), and fills the rest.
- **Progression balancing.** None as a separate pass (not found by `grep`).

## 6. Entrance, door or area randomization
- Entrance shuffle is delegated. `app/EntranceRandomizer.php:113-160` builds a command line and runs a vendored Python entrance randomizer (`vendor/z3/entrancerandomizer/EntranceRandomizer.py`, not in the clone) with `python3.10`, then reads its output back. The controller notes that the external tool is responsible for its own winnability (`RandomizerController.php:181-188`).
- The enemizer is also an external process (`app/Enemizer.php`, using `Symfony\Component\Process`), seeded with its own CSPRNG draw (`:60`).
- Door shuffle is not in this repo (it lives in the separate ALttP Door Randomizer, 00-landscape row 15).

## 7. Options model
- **Settings.** 16 validated request fields in `app/Http/Requests/CreateRandomizedGame.php:35-80` (counted): glitches, item placement, dungeon items, accessibility, goal, tower and Ganon crystal counts, world state, entrances, boss, enemy and pot shuffle, hints, weapons, item pool, item functionality. Enemy damage and health and a few others arrive unvalidated with defaults in the controller.
- **Presets.** 7 named presets in `config/alttp.php:55-170` (counted), each a full map of the same keys.
- **Random values.** Crystal counts accept `random` and are rolled server-side (`RandomizerController.php:106-109`).
- **UI.** Hand-built Vue components (`resources/js/components/VT*.vue`, `Customizer/*.vue`) with settings fetched from `GET randomizer/settings` (`routes/web.php:11`); not generated from a schema.
- **Conflict validation.** Ad hoc in the controller, for example forcing `advanced` placement for one pool (`:126-128`) or the completionist goal forcing full accessibility (`Randomizer.php:99-101`).

## 8. Seeds, reproducibility, spoiler logs, hints, solver and difficulty estimation
- **No reproducible seed number.** All randomness goes through `get_random_int`, which calls PHP's `random_int`, a cryptographic generator that cannot be seeded (`app/Helpers/number.php:11-14`; no `mt_srand` found by `grep`). A seed is reproducible only because the server stores its result.
- **Seed identity and permalink.** The database id is turned into a short public hash with Hashids (`app/Seed.php:14-17`). `/h/{hash}` renders a page that patches the player's own ROM in the browser; `/hash/{hash}` returns the stored JSON from cache or storage (`routes/web.php:15-38,121-140`). A five-symbol in-game code derived from the id (`hash_array`, `app/Helpers/array.php:142-158`) is written as the start screen hash (`World.php:1092`, via `Seed::hashArray`) so racers can confirm they have the same seed; a player-chosen override is allowed (`override_start_screen`).
- **Spoilers and races.** `spoilers` is `on`, `off`, `generate` (spoiler available at generation but stripped from the cached and shared copy) or `mystery` (only selected metadata) (`RandomizerController.php:39-79`). Tournament seeds call `Rom::rummageTable` (`:203-206`), which calls `Tournament::apply`; in the public repo that function returns the ROM unchanged (`app/Tournament.php`), so the tournament transform appears to be kept out of the public code (inferred). VARIA does the same with its race module (doc 10).
- **Daily seed.** A `FeaturedGame` per day, generated on demand by an artisan command (`routes/web.php:96-119`).
- **Playthrough.** `PlaythroughService::getPlayThrough` computes spheres on a copy of the world, removes junk, then walks spheres backwards pulling each item and keeping it only if the game becomes unbeatable without it (`app/Services/PlaythroughService.php:35-130`). This yields a minimal required-item path per sphere.
- **Hints.** Telepathic tiles get hints from a shuffled fixed list of hard-to-reach locations plus the boots and, when big keys are shuffled outside dungeons (`region.wildBigKeys`), the Ganon's Tower big key (`app/Services/HintService.php:51-115`).
- **Difficulty estimation.** None beyond the chosen logic level and pool difficulty (not found).

## 9. Game integration and delivery
- **Patch format.** A JSON array of `{offset: [bytes]}` write records, merged and compacted by `patch_merge_minify` (`app/Helpers/array.php:167-200`). The static part is a per-build BPS base patch (`/bps/{md5}.bps`, `routes/web.php:111-114`); the database `builds` table tracks base patch versions (`database/migrations/2019_11_30_135907_add_bps_to_builds_table.php`).
- **Browser patching.** `resources/js/rom.js` applies the BPS base patch to the player's ROM (`parseBaseBPS :526-540`, with `resources/js/bps.ts`), then the seed's write records (`parsePatch :481-524`), then cosmetic options (sprite, palette shuffle, music, quickswap) and fixes the SNES checksum (`:108-114`) before saving. The ROM is stored in the browser with localforage (`VTRomLoader.vue:80-91`) and checked by MD5 against the server's expected hash (`Randomizer.vue:437`).
- **Steps for a player** (estimated from the code path): first visit, pick the ROM file once; after that, choose settings, click generate, click save. Opening a shared `/h/{hash}` link is one click to save. No file is uploaded, and the cosmetic step re-runs on the client so the same seed can be saved with different cosmetics.

## 10. Multiworld or co-op support
- Fill is multi-world aware: `RandomAssumed` and `WorldCollection::isWinnable` sweep all worlds together (`app/Support/WorldCollection.php:34-60`). `MultiworldController` accepts a list of world configs (`:48-60`) and is routed (`routes/api.php:22`), but the multiworld page is commented out (`routes/web.php:74`) and `MultiworldFileService::createFile` returns an empty string (`app/Services/MultiworldFileService.php:33-36`). `documentation/multiworld.md` specifies a binary multiworld data file (header, item-location, text and entrance records). So the design is present, the delivery is not.

## 11. Tooling and testing
- 149 `*Test.php` files (counted), organized per logic level and mode: `NoGlitches`, `OverworldGlitches`, `MajorGlitches` with 23 each, three `Inverted*` folders with 22 each (counted). They use data providers of `[access, items, except]` rows; about 1,000 such rows across tests (estimated by `grep` for lines starting with a boolean array).
- Four JS test files including a BPS patcher test (`tests/js/bps.test.js`) and binary samples (`tests/samples/patch.bps`, `testrom.sfc`).
- Hosting stack: Laravel with queues (Horizon config), cache, Sentry, MySQL or SQLite, per-IP throttles on generation (`routes/api.php:20-28`: 150 per 360 minutes for standard seeds, 40 for multiworld; the second number is minutes per `app/Http/Middleware/ThrottleRequestsWithIp.php:16`).

## 12. Developer ergonomics
- Adding a location: add a constructor line with ROM addresses to the region class, add its requirement closure in `initalize()`, then check every mode class (Open, Inverted) that overrides that region, and add test rows. Adding an item: add an item class entry, pool counts in `config/item.php`, and its ROM byte.
- Documentation is the README (setup and CLI) and one file format doc. Contributors: 48 (counted).

## 13. Patterns worth standardizing and pain points
- **Web delivery done well.** Stored ROM in the browser, a hash-checked base patch per build, a tiny per-seed write list, client-side cosmetics, and a short permalink. This is close to our CPK shape (01 D4.1): static patch once per build, seed delta per seed.
- **Spoiler visibility as a seed property** (`on`, `off`, `generate`, `mystery`) rather than a separate file decision. Matches our race seeds (01 D3.3) and adds the useful "generate" mode: the creator sees the spoiler once, the shared copy never has it.
- **Minimal playthrough by removal**, a well-known way to compute required items.
- **In-game seed code** for racers to confirm they are on the same seed.
- **Pain points.** Unseeded randomness means seeds cannot be regenerated or verified by a third party; trust rests on the server's stored copy. Logic as closures cannot be read by a tracker (OpenTracker re-encodes it, 00-landscape row 47). Rules are duplicated across the Standard, Open and Inverted class trees, and trick flags are scattered inside closures, so "which locations does this trick open" has no index. Entrance shuffle as an external process makes winnability a contract between two programs.

## 14. Mapping to our standard
**Maps to existing doc 07 primitives and doc 01 concepts**
- Regions, locations, connections and win condition map to 01 D2.2 `regions`, `locations`, `goal`. Item helpers map to `defs`. Location requirements map to `rule`.
- Logic level as a set of trick flags maps to toggles plus `opt` in rules, and named presets map to `presets` (01 D2.3).
- `setFillRules` and dungeon-item restrictions map to `item_rule` (07 2.3). Pre-placed prizes and goal items map to `place`. Ganon's Tower junk count maps to `place` of filler with a seeded count, or `select` (07 2.3).
- `accessibility` modes map to 01 D2.7 location kinds and beatability. Plando map maps to `place`; starting equipment maps to start items.
- Spoiler modes and daily seeds map to 01 D3.3. The BPS base plus write list maps to CPK static patch plus seed delta (01 D4.1).

**Needs a new primitive or concept**
- **`always_allow` (candidate, an `item_rule` extension):** an item may be placed at a location even if the location's access rule fails for it, because collecting it is what grants access (a dungeon's own map at its boss). Evidence: `DesertPalace.php:117-120`, `Location.php:77`.
- **`spoiler_policy` (candidate concept for 01 D3.3):** an enum with a `creator_once` value matching `generate`. Evidence: `RandomizerController.php:69-75`.
- **`seed_code` (candidate concept):** a short symbol code derived from the bundle hash, shown in game and on the room page. Evidence: `hash_array`, `World.php:1092`. Also seen in z17 (doc 12), so two projects so far.
- **`junk_quota` (candidate, maybe a `select` recipe):** place a seeded number of filler items in a location group before progression fill. Evidence: `RandomAssumed.php:37-46`.

**Cannot be data, and why**
- ROM writes, text tables, compression, sprite and palette work, checksum fixing (`app/Rom.php`, `resources/js/rom.js`) are binary patching; they belong in a third-party patcher fed by slot data (01 D4.1, 07 6.7).
- The external entrance randomizer and enemizer are separate programs with their own logic; as data they would have to be re-expressed as `rewire` pools and an enemy table, not wrapped.

## Appendix A. Not verified, and open questions
- Whether alttpr.com production runs this exact branch head (the head predates the last push by five months).
- How the live site stores seeds today (cache plus storage JSON per `routes/web.php:15-38`; the storage backend was not checked).
- Whether the private tournament transform also changes anything a third-party tracker reads (inferred only from the empty `app/Tournament.php`).
- Whether the vendored entrance randomizer is the same code as the separate ALttP entrance randomizer projects (not opened; `vendor/` was not in the clone).
- Generation time per seed and server cost: not measured.

## Appendix B. Files read
- `readme.md`, `.travis.yml`, `documentation/multiworld.md`
- `app/Filler/RandomAssumed.php`, `app/Randomizer.php` (outline and `:97-140`), `app/World.php` (outline)
- `app/Location.php:72-97`, `app/Region/Standard/DesertPalace.php`, `app/Region/Inverted/*.php` (class lines)
- `app/Services/PlaythroughService.php:35-130`, `app/Services/HintService.php:51-115`, `app/Services/MultiworldFileService.php`
- `app/Support/WorldCollection.php:34-60`, `app/Helpers/number.php`, `app/Helpers/array.php:140-200`
- `app/Seed.php`, `app/Tournament.php`, `app/Rom.php:2487-2492`, `app/EntranceRandomizer.php:60-160`, `app/Enemizer.php:50-75`
- `app/Http/Controllers/{Randomizer,Customizer,Multiworld}Controller.php`, `app/Http/Requests/CreateRandomizedGame.php`, `app/Http/Middleware/ThrottleRequestsWithIp.php`, `app/Http/Kernel.php`
- `app/Jobs/SendPatchToDisk.php`, `routes/web.php`, `routes/api.php`
- `config/logic.php`, `config/alttp.php:30-170`, `config/enemizer.php`
- `resources/js/rom.js:1-120,470-540`, `resources/js/views/Randomizer.vue` (grep), `resources/js/components/VTRomLoader.vue` (grep)
- `tests/NoGlitches/DesertPalaceTest.php:1-110`, listings of `tests/`
