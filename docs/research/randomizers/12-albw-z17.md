# 12. A Link Between Worlds Randomizer (z17): architecture and patterns audit

> Internal research note, 2026-11-16, read at commit `dc1a9c945bed9da96a14e7e40c7eaf7e0636eaf2` (2026-10-02) of the `master` branch of [rickfay/z17-randomizer](https://github.com/rickfay/z17-randomizer) (shallow sparse clone of `randomizer/`, `macros/`, `rom/`, `presets/`, `docs/`, `cli/`, `game/`, `modinfo/`, `src/`, `.github/`; no code was run, built or installed). Not reviewed by a different agent or a human. Our own writing with citations; no code, text or data from the project is reproduced, and because the project is GPL-2.0 nothing from it may enter our Rust code. Numbers are labeled **counted** (`grep`, `awk`, `wc`, `git ls-tree`, or the GitHub API on 2026-11-16), **estimated**, or **not verified**. Paths are relative to the repo root.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. Summary
- **What it is.** The item randomizer for The Legend of Zelda: A Link Between Worlds (Nintendo 3DS). A Rust program reads the player's own decrypted game file and writes a LayeredFS mod folder (modified game files and a code patch) that a modded console or an emulator loads.
- **Popularity.** 47 stars, 5 contributors including anonymous (counted, GitHub API; 00-landscape row 17).
- **License.** GPL-2.0 (GitHub API). For us: read for design ideas and describe in our own words; never copy or translate its Rust into ours. Anything derived would have to be GPL-2.0.
- **Maintenance.** Active: last push 2026-10-13 to some branch; the default branch head is 2026-10-02 (counted). The README's update notes describe version 0.4 features (Quake, Cracksanity, weather vane shuffle, Keysy). CI builds release binaries for six targets on pull requests (`.github/workflows/`, build job).

## 2. Architecture
- **Language and crates.** Rust workspace: `randomizer` (logic, fill, hints, patching), `modinfo` (settings types), `rom` (3DS file formats: SARC, BYAML, MSBT and MSBF, exheader, RomFS), `game` (game constants: items, courses, hint ghosts), `macros`, `cli` (interactive prompts), `bytey` (binary helpers), and the top-level `src/main.rs` and `src/bin/plando.rs` binaries (`git ls-tree`).
- **Size.** 12,517 lines across the main `randomizer/src` logic, filler, world and patch files shown by `wc -l` (counted, partial set); largest are `filler/filler_item.rs` 1,572, `world/hyrule.rs` 1,315, `world/lorule.rs` 1,218 (counted).
- **Pipeline.** `src/main.rs:30-80`: load settings from a preset or CLI prompts, load `config.json` (ROM path, output folder), then try up to 100 times to generate. `randomizer::generate_seed` (`randomizer/src/lib.rs:380-395`) seeds a `StdRng` from a 32-bit seed, computes the seed hash, builds seed info (world graph, crack and vane maps, fill, hints, metrics), then patches and writes the spoiler.
- **How users run it.** A desktop binary: double click for prompts, or CLI with `--preset` and `--seed` (README "Running the Randomizer"). No web front end.

## 3. World model
- **Nodes.** A `Location` enum of graph nodes (241 variants counted in `randomizer/src/filler/location.rs`). Each node is a `LocationNode` with a display name, a list of checks and a list of edges.
- **Checks.** 402 `check!` macro uses across `randomizer/src/world/*.rs` (counted). A check has a name, a region and subregion for hints and spoilers, and a logic object. `goal!` (37 uses, counted) marks logic-only events such as defeating a boss or rescuing a sage. Hint ghosts appear as checks too (59 `ghost(` uses, counted).
- **Edges.** 434 `edge!` uses (counted), each a destination node with a logic object.
- **Regions.** `randomizer/src/regions/{hyrule,lorule,dungeons}/` define named regions and subregions used for hint text and spoiler grouping, separate from the graph nodes.
- **Items.** A large `Item` enum plus goals, cracks and weather vanes as other kinds of `Randomizable` (`randomizer/src/filler/filler_item.rs`; 358 enum lines counted by our `awk`, which may include non-item variants).
- **Format.** Rust code built with macros (`randomizer/src/world/mod.rs:82-170`). A code comment says the author would like to move logic out of code so users could write custom logic (`randomizer/src/filler/logic.rs:8`).

## 4. Logic
- **Tiered rules.** A `Logic` value holds up to five optional function pointers, one per tier: normal, hard, glitched, advanced glitched, hell (`randomizer/src/filler/logic.rs:10-17`). Access at a chosen tier passes if the rule of that tier **or any lower tier** passes; `NoLogic` always passes (`:61-80`). A missing tier rule means "no extra way at this tier".
- **Helpers.** 152 public methods on `Progress` (counted in `randomizer/src/filler/progress.rs`), such as `can_merge`, `can_hit_far_switch`, `has_master_sword`, `has_eastern_keys(n)`.
- **Toggles.** Separate settings adjust logic, for example lampless dark rooms and lamp-and-net-as-weapons (`presets/Example.json`), read by helpers through `progress.get_settings()`.
- **Example, in our words.** In Eastern Palace, moving from the foyer to the first floor needs, in normal logic, a ranged switch hit, merging, or the upgraded ice rod; in hard logic the master sword also works (`randomizer/src/world/eastern.rs:22-25`). A first-floor chest is free in hard logic because the player can throw a pot at the switch (`:33-36`, the inline comment).
- **Validation.** `verify_all_locations_accessible` checks that with all progression every location is reachable, which catches layouts that crack shuffle made unreachable (`randomizer/src/filler/mod.rs:669`). `validate_settings` rejects incompatible settings with plain messages (`randomizer/src/lib.rs:453-490`).

## 5. Fill and placement
- **Standard filler.** `fill_all_locations_reachable` (`randomizer/src/filler/mod.rs:35-58`): build crack and vane maps, preplace static and shop items, handle exclusions, then assumed fill.
- **Assumed fill** (`assumed_fill`, `randomizer/src/filler/mod.rs:799`): place progression items backwards, each into a random check reachable assuming all unplaced items are held, computed by an assumed search that loops until no new items are found (`assumed_search`, `:875`).
- **Constraints.** Dungeon items and prizes filtered to their dungeons (`filter_dungeon_checks :532`, `filter_dungeon_prize_checks :520`); the Maiamai limit junks remaining Maiamai upgrade checks once too many majors sit behind them (`handle_maiamai_limit :838`); shop guarantees such as an assured weapon (`preplace_items`, `:161-300`).
- **Exclusions.** `user_exclusions` in the preset force junk at named checks, and minigames can be excluded as a group (`handle_exclusions :320-375`; `presets/Example.json`).
- **Starting inventory.** Settings such as `start_with_merge` and `start_with_pouch` (`presets/Example.json`).
- **Plando.** A separate binary builds a hand-written check map, vane map and crack map and patches it (`src/bin/plando.rs:39-180`), so plando is code, not a file.
- **Retries.** On a generation error, the main loop draws a new random seed and tries again, up to 100 times, unless the user gave a seed (`src/main.rs:47-80`). The published seed is always one that succeeded, so it regenerates.

## 6. Entrance, door or area randomization
- **Cracksanity.** Wall cracks between Hyrule and Lorule are shuffled under four pairing modes: cross-world pairs, any-world pairs, and mirrored versions of each (README "Cracksanity"; `randomizer/src/filler/cracks.rs:556-760`). Six down-facing crack pairs only shuffle among themselves for technical reasons (`:559`). Cracks are graph edges whose destination comes from the crack map.
- **Weather vane pairs.** Vanes can be grouped into random pairs that unlock each other (`randomizer/src/filler/vanes.rs`; README).
- **Quake.** An item that opens the cracks; it is removed from the pool if cracks start open (README).

## 7. Options model
- **Settings.** 35 public fields on `Settings` (counted in `modinfo/src/settings/mod.rs`), with enum types per option in `modinfo/src/settings/*.rs` (logic mode, cracks, cracksanity, keysy, trials door, weather vanes, nice items, pedestal, Ravio's shop).
- **Presets.** JSON with comments: a seed, a version and a settings map (`presets/Example.json`). The README says presets simplify seed sharing.
- **UI.** CLI prompts written per option (`cli/src/lib.rs:21-40`), not generated from the types.
- **Conflict validation.** Hand-written checks in `validate_settings` (`randomizer/src/lib.rs:453-490`), for example sword-in-shop versus swordless mode.

## 8. Seeds, reproducibility, spoiler logs, hints, solver and difficulty estimation
- **Seed.** A 32-bit integer; `StdRng::seed_from_u64` (`randomizer/src/lib.rs:385`). Deterministic hash maps (`DashMap` and `DashSet` with a fixed XxHash64 hasher, `:492-500`, with a comment explaining why the default randomized hasher is avoided). The world graph itself is built as a standard `HashMap` (`randomizer/src/world/eastern.rs:11-14`); whether its iteration order can affect output was not verified.
- **Seed hash.** XxHash64 over seed, settings and version, reduced to five digits and shown as item icons on the file select screen and as text in the spoiler (`randomizer/src/lib.rs:399-440`; README "Item Hash").
- **Spoiler and playthrough.** `metrics::calculate_metrics` runs a sphere search after fill and stores the spheres (items, ghosts, goals, cracks, vanes per sphere) in the spoiler (`randomizer/src/metrics/mod.rs:12-40`).
- **Hints.** `randomizer/src/hints/mod.rs`: always hints, path hints (an item required to reach a goal, like way-of-the-hero hints in other Zelda randomizers), sometimes hints, crack hints and a Bow of Light hint (`:39-351`). Each hint lists hint ghosts **guaranteed logically reachable before the hinted item**, and duplicates are placed elsewhere at random (doc comments at `:55-59`). README "Hint System" explains this to players.
- **Difficulty estimation.** None beyond the logic tier and sphere count (not found).

## 9. Game integration and delivery
- **Output.** A folder named after the game's title id containing modified RomFS files and a code patch derived from the exheader (`randomizer/src/patch/mod.rs:593-700`, `dump`). The player copies it to the console's LayeredFS mods path or the emulator's mods folder (README "Installing Seeds").
- **Input.** A decrypted North American `.3ds` file named in `config.json` (README "Setup"). The README links community instructions for dumping one's own cartridge. The console's system language must be English (README "Running the Randomizer").
- **Steps for a player** (estimated from the README): dump and decrypt the game once, place it next to the binary, run the binary, copy the output folder to the SD card or emulator, launch. About five steps per seed after setup, and file moves each time.

## 10. Multiworld or co-op support
- None in this repo (not found by `grep`; 00-landscape: not verified).

## 11. Tooling and testing
- 11 `#[test]` functions in the workspace (counted), all in the `rom` crate and file-format code (`rom/tests/flow.rs`, `rom/src/files/byaml/de.rs`, `exheader.rs`, `sarc.rs`, `item.rs`) and one in `patch/lms/msbf.rs`. No tests for logic or fill (counted: none under `randomizer/src/filler/` or `world/`).
- CI builds release binaries for macOS, Linux (musl) and Windows on x86-64 and ARM64 (`.github/workflows/`).

## 12. Developer ergonomics
- Adding a location: add a `check!` to a node's list in the right `world/*.rs` file, with a subregion and tiered logic, and patch data for where the item sits in the game files. Adding an item: enum variant, pool entry in `filler/item_pools.rs`, and patch mapping.
- Macros keep the world files compact and readable. Docs are the README (game options, hint system, known issues) and images in `docs/`. Contributors: 5 (counted).

## 13. Patterns worth standardizing and pain points
- **Tiered rules, each tier adding alternatives** instead of separate rule sets per level. Compact and easy to read per edge.
- **Reachable-before hint sources:** a hint is only placed on a ghost the player can logically reach before the item it reveals.
- **Retry with a fresh seed, publish the one that worked.** Simple and keeps seeds reproducible.
- **In-game seed hash** from seed, settings and version (also in ALttPR, doc 09).
- **All-locations-accessible check** before fill as a guard for entrance-style shuffles.
- **Pain points.** Logic as Rust closures means a tracker or a web page cannot evaluate it without the binary. Delivery is a folder of game files that the player moves by hand per seed. No logic tests. Plando requires editing code.

## 14. Mapping to our standard
**Maps to existing doc 07 primitives and doc 01 concepts**
- Nodes, checks, edges and goals map to `regions`, `locations`, `connections` and `events` (01 D2.2). Progress helpers map to `defs`.
- Cracksanity pairing modes map to `rewire` with coupling rules (07 2.3); mirrored variants are a coupling constraint. Weather vane pairs map to `permute` over a table with a pairing rule, or `rewire` (not verified which fits).
- Dungeon item filters map to `item_rule`. Static shop placements and the assured weapon map to `place`. User exclusions map to `excluded` location kind (01 D2.7). The Maiamai limit maps to a counter-bounded rule (07 `counter`), not verified.
- Presets and settings map to typed options and `presets`; incompatible settings map to `conflicts` (07 2.3). Seed hash maps to the bundle hash (01 D3.2).

**Needs a new primitive or concept**
- **`tiers` on rules (candidate language construct):** a rule written as an ordered list of tier alternatives; at tier k, the rule is the OR of tiers 0..k. This compiles to plain `any` plus `opt` comparisons, so it may be an authoring macro rather than a primitive (07 2.4 composition test). Evidence: `randomizer/src/filler/logic.rs:61-80`. Compare VARIA's numeric costs (doc 10): tiers are a coarse, ordered cost.
- **`hint_source_reachable_before` (candidate constraint for the `hint_plan` primitive proposed in `01-oot-randomizer.md`):** a hint may only be placed at a source reachable without the hinted item. Evidence: `randomizer/src/hints/mod.rs:55-59`.
- **`retry_policy` (generator concept for 01 D3.2):** fresh seed on failure when the user did not fix one; fixed seeds fail deterministically. Evidence: `src/main.rs:47-80`.

**Cannot be data, and why**
- Writing BYAML stages, MSBT and MSBF text and flow, actor and chest changes, and ARM code patches (`randomizer/src/patch/`, `rom/`) is game-file work for a third-party patcher. Our CPK diff model (01 D4.1) would carry file-level diffs for RomFS plus a code patch; the tool that builds the folder stays third party.
- Crack spawn coordinates and blocked-crack handling (`randomizer/src/filler/cracks.rs:272-520`) are game facts that could be table data, but their use in patching is code.

## Appendix A. Not verified, and open questions
- Whether standard `HashMap` iteration in the world graph can change fill order between runs or builds (the code uses a deterministic hasher elsewhere, which suggests the author cared; not traced).
- Whether the `dev` branch named in the README differs in logic from `master` (not cloned).
- Whether an Archipelago world for this game exists outside the main repo (none in our local `worlds/` listing).
- Generation time and retry rate: not measured. The README says first-try success became much more likely after a fill change (README update notes).
- Whether the code patch is a byte diff against the player's own code binary or a full file (`patch/code/` outline only).

## Appendix B. Files read
- `README.md` (update notes, setup, running, installing, options, hint system), `config.json`, `presets/Example.json`
- `.github/workflows/` build job, `src/main.rs:30-80`, `src/bin/plando.rs` (outline)
- `randomizer/src/lib.rs` (outline and `:380-500`), `randomizer/src/filler/logic.rs:1-80`, `randomizer/src/filler/mod.rs` (outline and doc comments)
- `randomizer/src/world/eastern.rs:1-60`, `randomizer/src/world/mod.rs:80-170`
- `randomizer/src/filler/cracks.rs` (outline), `randomizer/src/metrics/mod.rs:1-40`, `randomizer/src/hints/mod.rs` (outline and doc comments)
- `randomizer/src/patch/mod.rs` (outline and `:593-720`), `cli/src/lib.rs:1-40`, `modinfo/src/settings/mod.rs` (field count)

## Appendix C. Delivery steps compared within batch C (estimated from code and READMEs)
| Project | Player supplies | Generator runs | Player receives | File moves per seed |
|---|---|---|---|---|
| ALttPR (doc 09) | ROM once, cached in browser | server | patched ROM saved from the page | one download |
| VARIA (doc 10) | ROM once, cached in browser | server | patched ROM saved from the page | one download |
| Worlds Collide (doc 11) | ROM per run (CLI) | player's machine | full ROM file | one, local |
| z17 (this doc) | decrypted game file once | player's machine | mod folder | copy folder to SD card or emulator |
