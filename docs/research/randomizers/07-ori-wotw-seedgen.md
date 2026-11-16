# 07. Ori and the Will of the Wisps seed generator (wotw-seedgen): two custom languages in Rust

> Internal research note, 2026-11-16. Not reviewed by a different agent or a human; nothing here goes outside this repo before review under [`CLAUDE.md`](../../../CLAUDE.md). Our own writing at a high level; no code, logic text or snippet text is reproduced, and logic examples are paraphrased. Numbers are **counted** (grep or our own script over a clone), **reported** (stated in the project's own files), **estimated**, or **not verified**.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

Source read: [ori-community/wotw-seedgen](https://github.com/ori-community/wotw-seedgen). The default branch (`main`, same commit as `dev`, `d94ecfc`, 2026-05-07) holds the older layout that 00-landscape described (`wotw_seedgen/areas.wotw`, `loc_data.csv`, `headers/*.wotwrh`, `src/languages/{logic,header}/`). Active work is on branch **`v5`** (`51ad907`, 2026-11-10, matching the repo's last push), a reorganized workspace. This note audits `v5` and mentions `main` where it differs. Read: `README.md`, `assets/logic/` (`paths.wotwl`, `pickups.csv`, `states.csv`), `assets/snippets/` (file list and one snippet), `assets/world_presets/`, `assets/preset compability.md`, `wotw_seedgen/src/generator/`, `wotw_seedgen_data/src/{logic_language,seed_language,settings}/` (structure and doc comments), `.github/workflows/test.yaml`, `Cargo.toml` files. Not read: the client ([ori-community/wotw-rando-client](https://github.com/ori-community/wotw-rando-client)), the multiplayer server, the LSP internals.

## 1. Summary
- **What it is.** The seed generator for the Ori and the Will of the Wisps randomizer: a Rust workspace with its own logic language (areas and paths) and its own seed language (snippets that script in-game behavior), a CLI, an HTTP server, a language server and a stats crate.
- **Game.** Ori and the Will of the Wisps (PC), played through the community client mod.
- **Popularity.** 5 stars, 12 contributors (counted, 00-landscape and the GitHub API); the client has 23 stars. Low GitHub visibility; community size not measured.
- **License.** No LICENSE file in the tree on either branch (searched). However `wotw_seedgen/Cargo.toml` and `wotw_seedgen_derive/Cargo.toml` declare `license = "MIT"` in package metadata (opened). A manifest field without a license text is ambiguous; we treat the repo as **all rights reserved** for our purposes: read for ideas only, describe at a high level, copy nothing. **Needs an owner or lawyer decision** on whether the `Cargo.toml` MIT declaration grants anything usable; until then, read for ideas only. If we ever want more, ask the maintainers.
- **Maintenance.** Active on `v5` (commits to 2026-11-10). A changelog of breaking asset changes for custom presets starts with asset version 1 for randomizer 5.0.0 (`assets/preset compability.md`).

## 2. Architecture
- **Language.** Rust; RNG is `rand_pcg` (`Pcg64Mcg`) seeded from the settings' seed string through `rand_seeder` (`wotw_seedgen/src/generator/mod.rs`).
- **Workspace members** (`README.md`): `wotw_seedgen` (high-level generate API), `wotw_seedgen_data` (data structures and parsing for both languages), `wotw_seedgen_parse` (their own parsing library: tokenizer, spans, error recovery), `wotw_seedgen_seed` (seed file packaging), `wotw_seedgen_cli`, `wotw_seedgen_http_server`, `wotw_seedgen_lsp` (language server), `wotw_seedgen_server_shared`, `wotw_seedgen_stats` (analyzes large numbers of seeds), `wotw_seedgen_derive`. About 40,000 lines of Rust on `main` (counted with `wc -l`, `main` branch; `v5` not totaled).
- **Pipeline** (`generator/`): parse and compile the logic file into a graph; compile the selected snippets (seed language) into commands and events; **simulate** the snippets' effects on game state so the generator knows what they change (`seed_language/simulate/`); choose spawn (`spawn.rs`); optionally randomize entrances (`entrances.rs`); build the item pool (`item_pool.rs`); run placement (`placement.rs`); write spoiler (`spoiler.rs`); package one seed file per world (`wotw_seedgen_seed`, a zip archive with zstd-compressed assembly, `package.rs`).
- **How users run it.** Per the README, the launcher uses an **online** seedgen and the in-game logic filter uses an interop DLL; the CLI is mainly for development. A player picks a preset or settings in the launcher, gets a seed file, and the client mod applies it at runtime.

## 3. World model
- **Anchors** are points on the map, optionally with coordinates ("at x, y"), grouped under **regions**. Each anchor lists, with requirements: `pickup` (a location), `state` (a world event or flag), `conn` (connection to another anchor), `refill` (health and energy refill points: full, checkpoint, and so on), `nospawn`, `tprestriction`, and on `v5` an `entrance` block with an id, a target anchor and an enter requirement.
- **Requirement macros** (`requirement Name:`) with per-difficulty bodies must come first in the file (the file's header comment says so).
- **Tables.** `pickups.csv` maps each pickup identifier to a game state condition (a group and member pair of the game's "uber state" ids), area, pickup type, details and map coordinates; `states.csv` does the same for logical states. The game's state catalog ships as `assets/uber_state_dump.json`.
- **External states.** A dedicated anchor holds states that the logic file cannot solve itself and that are set from outside, by snippets or by other integrations (the file's own comment names Archipelago as one user). Each defaults to impossible, so logic ignores them unless a setting switches them on (`paths.wotwl`, near the top).
- **Teleporters** sit in one anchor whose connections are gated by teleporter states and, for some, by skills unless a "remove teleporter locks" state is set.
- **Counts on `v5` (counted by grep over `paths.wotwl`, 17,582 lines):** 20 regions, 355 anchors, 469 pickup entries, 277 state entries, 884 connections, 335 refill lines, 32 entrances, 33 nospawn markers, 2 top-level requirement macros. `pickups.csv` 390 rows and `states.csv` 195 rows (counted, minus header). On `main` the single file `areas.wotw` is 789,649 bytes (counted from the tree listing).

## 4. Logic
- **Grammar (described, not reproduced).** Indentation-based blocks. Under each pickup, state or connection, lines start with a **difficulty tier** keyword (moki, gorlek, kii, unsafe) followed by requirements. Within a line, commas mean AND and `OR` means OR; separate lines under one tier are alternatives; a line ending in a colon opens a nested group whose sub-lines are ANDed with the parent line (`paths.wotwl`, near the top). Requirements are skills, items with amounts (`Keystone=2`), world states by dotted name, combat (`Combat=` with enemy names), damage (`Damage=10`), energy-costing actions, `free` and `Impossible`.
- **Parameterized helpers built into the language.** The file header documents built-in requirement keywords such as breaking a wall with a given health, sentry jumps with a count, hammer and grenade jumps, and swaps; many expand to skills plus "having energy to use it" (`paths.wotwl`, header comments). These are closer to typed built-ins than to user macros.
- **Resources.** Health and energy are tracked along paths ("orbs", `wotw_seedgen/src/orbs.rs`, `generator/solutions/`), with refill anchors restoring them, so reachability carries resource state.
- **Difficulty and tricks.** 4 difficulties in the settings enum (`Difficulty`: Moki, Gorlek, Kii, Unsafe, with doc comments describing each) and 32 trick variants (counted in `enum Trick`, `wotw_seedgen_data/src/settings/mod.rs`). Tier keyword counts in the logic file: moki 1,117, unsafe 774, gorlek 738, kii 738 (counted, indented occurrences, approximate). World presets select a difficulty (`world_presets/moki.json` and others, 6 files).
- **Tooling for the language.** Tokenizer, parser with error recovery (`wotw_seedgen_parse/src/recover.rs`), compile, decompile, an optimizer (`logic_language/optimize/`), and an LSP with go-to-definition for logic and completion, hover and semantic tokens for the seed language (`wotw_seedgen_lsp/src/`).
- **Example in our words.** At the marsh spawn anchor, a nearby life fragment needs grapple plus double jump or launch at the easiest tier; the next tier adds alternatives using dash, glide or a weapon combo after a grapple; higher tiers add sentry jumps, damage boosts of a stated amount, or bash chains that need the storm to be lifted. Each tier only adds alternatives, so higher difficulty is a superset.
- **Comments as design notes.** The logic file carries inline notes about generator behavior, for example why a teleporter path requires extra keystones so the generator does not fill newly opened spots with the keys it needs (`paths.wotwl`, teleporter anchor).

## 5. Fill and placement
- **Algorithm** (`generator/placement.rs`, 1,804 lines; `generator/solutions/`): forward fill by **solutions**. For the current reach, the generator searches combinations of not-yet-owned items ("solutions") that unlock new locations, weights them (`solution_weights`), picks one, force-places it in reachable slots, then fills some random placements, and repeats. Search limits are constants with the project's own benchmark timings in doc comments (for example a maximum solution size and a search radius, each with reported millisecond figures; reported by the project, not measured by us).
- **Portability of order.** A comment notes that the solution search order is not portable, so results are sorted before the weighted choice (`calculate_weights`). This is a deliberate step toward cross-platform determinism.
- **Special placement.** Pre-placements per area or world, forced keystones, spirit light (currency) with its own slot logic, shop slots, a fallback ore item, placeholder reservation.
- **Item pool** (`item_pool.rs`, 384 lines) and spirit light amounts (`spirit_light.rs`) are code; spirit light values come from a seeded command helper in the seed language (`seed_language/compile/function.rs`, 66 functions counted).
- **Logging does not change output.** A comment in `placement.rs` says output must be identical with logging on or off, so trace code sorts a clone instead of the live data.
- **Multiworld in the fill.** All worlds fill together; the world with the most remaining placements is tried first; `UNSHARED_ITEMS = 5` items per world are guaranteed to stay local (`placement.rs`).
- **Retries.** 10 by default (environment-overridable), mostly for items the generator cannot force-place (`generator/mod.rs` doc comment).
- **Plando.** First-class: plando sources in the seed language (`assets/plandos/src/*.wotws`, 3 files) with plando attributes (`wotw_seedgen_seed/src/plando_attributes.rs`).
- **Starting inventory** comes from spawn settings and snippets (for example spawn with sword).

## 6. Entrance, door or area randomization
On `v5`, entrance randomization exists (`generator/entrances.rs`, 378 lines; `randomize_entrances` in `WorldSettings` as an optional number greater than one, read as a maximum loop size; entrances grouped into up to 16 groups; 10 retries). 32 entrances are declared in the logic file (counted). Spawn randomization picks a start anchor (`spawn.rs`). Teleporters are connections gated by states.

## 7. Options model
- **Universe settings** hold a seed string, one `WorldSettings` per world and inline snippets; world settings hold spawn, difficulty, tricks, game difficulties, entrance randomization, the snippet set and per-snippet config key-value maps (`settings/mod.rs`).
- **Snippets as options.** 57 snippet files (counted) carry metadata attributes for category, name and description at the top (`assets/snippets/no_quests.wotws`), so the launcher can list them. Snippets can declare config values.
- **Presets.** World presets are JSON with an asset version, info, includes (preset inheritance) and a snippet list (`world_presets/moki.json`).
- **Settings identity.** A "slug": SHA-256 of the settings JSON, mapped to human-readable words from word lists (`settings/slug/mod.rs`). This names a settings set; it is not a decodable settings string.
- **Conflicts.** Settings types make some invalid states unrepresentable (a nonzero type for loop size); broader conflict rules not found.

## 8. Seeds, reproducibility, spoiler logs and hints
- **Cross-platform reproducibility in CI.** The test workflow generates a seed with a fixed seed string on Linux and Windows and compares the two seed files with `zipcmp` (`.github/workflows/test.yaml`). As written, a mismatch prints a message but the step does not appear to fail the job (our reading of the YAML, not run).
- **RNG.** One seeded PCG generator per generation, created from the seed string (`generator/mod.rs`); a named algorithm rather than a language default, which is what makes cross-OS comparison possible.
- **Seed package.** A zip with zstd-compressed content; doc comments record their own compression timing trade-offs per level (reported, `wotw_seedgen_seed/src/package.rs`).
- **Spoiler** (`generator/spoiler.rs`): spawns, entrance connections per world, preplacements, then placement groups, one per fill step, so the spoiler reads as spheres.
- **Hints** are snippets (area, key, knowledge, shrine, trial, tree hints among the 57).

## 9. Game integration
- Runtime mod: the client reads a seed file whose main content is compiled seed-language commands, triggers and events keyed to the game's state ids (`seed_language/output/`). Pickups are identified by game state conditions from `pickups.csv`, so the client can detect a pickup by watching state changes.
- Snippets change game behavior at runtime (for example a no-quests snippet sets quest states at spawn), and the generator simulates those changes before fill so logic matches the patched game.
- Steps for a player (estimated): install launcher and client, choose preset, generate online, play. About 3 to 4.

## 10. Multiworld or co-op
- Generation supports many worlds in one universe, with a counter of multiworld state ids for cross-world items (`placement.rs`). Bingo is a snippet (`bingo.wotws`, `launch_from_bingo.wotws`). The network protocol lives in the client and server repos (not read).

## 11. Tooling and testing
- **LSP** for both languages (`wotw_seedgen_lsp/`), the strongest authoring tooling in this batch.
- **Tests:** 51 `#[test]` functions on `main` (counted); large test modules for requirements and solutions on `v5` (`generator/solutions/tests.rs`, 1,904 lines; `world/tests.rs`, 575 lines).
- **Benchmarks** (`wotw_seedgen_data/benches`), with timings recorded in comments.
- **Stats crate** that generates and analyzes many seeds (`wotw_seedgen_stats/`).
- **CI** (`.github/workflows/`): tests on a matrix, cross-OS seed comparison, container image builds, crate publishing.

## 12. Developer ergonomics
- **Adding a location:** add a row to `pickups.csv` with its game state condition, then a `pickup` entry under an anchor with tiered requirements. **Adding a behavior or option:** write a snippet with metadata. The LSP helps both.
- Single-game design: anchors, refills, uber states and orbs are Ori concepts baked into the language, so the language is not game-agnostic, though the patterns are.
- Docs: README for development; logic syntax documented in the logic file's header; an external document for the logic format linked there (not opened).

## 13. Patterns worth standardizing and pain points
Worth standardizing:
1. **Difficulty tiers inside each rule** (one block per tier, each tier only adds alternatives) make "harder setting is a superset" structural and easy to review.
2. **Coordinates on graph nodes** (anchor positions, map positions in `pickups.csv`) feed trackers and maps.
3. **Pickup identity as a game state condition** in a table, so the client detects checks generically.
4. **Language server** for the logic and options languages.
5. **Simulating option effects before fill**, so logic sees what option scripts change.
6. **Cross-OS golden seed comparison in CI** and explicit sorting where iteration order is not portable.
7. **Preset inheritance** (`includes`) and an asset-version changelog for presets.
Pain points:
- Two bespoke languages plus a homebrew parser library are a large surface for a small team (inferred cost).
- The seed language is a programming language (functions, scopes, a simulator); option snippets therefore cannot be analyzed as data.
- No license text, which limits reuse by others.

## 14. Mapping to our standard
| Construct | Maps to (doc 07 / doc 01) | Status |
|---|---|---|
| Region, anchor, conn | `regions`, `connections` (D2.2) | Maps (anchors become regions) |
| Pickup, pickups.csv row | `locations` plus `tables` with CSV `source` (doc 07 4.7) | Maps |
| State | `events` | Maps |
| Comma AND, OR, alternatives, nested groups | `all`, `any` | Maps |
| Item with amount (`Keystone=2`) | `has` with `n` | Maps |
| Requirement macro with per-tier bodies | `defs` | Maps if tiers map (next row) |
| Difficulty tier blocks | `opt` comparisons | **New concept `tier`**: an ordered world-level difficulty where each rule may have per-tier alternatives, compiled to `any(all(tier >= t, body_t))`. Monotone by construction. Overlaps with the proposed `trick` concept |
| Damage, energy, refill anchors | none | **`resource_state`** primitive candidate (health and energy as bounded path state; refill nodes) |
| Built-in parameterized requirements (wall health, sentry jumps) | parameterized macros (doc 07 2.2) | Maps as macros over game tables |
| Combat requirements with enemy lists | macros over an enemy table | Maps |
| Coordinates | none | **Metadata field `pos`** on regions and locations (no logic effect) |
| Entrances with groups and loop size | `rewire` | Maps; loop size is a `rewire` parameter |
| Spawn selection, nospawn | `select` over regions with a tag | Maps |
| Pre-placements, forced keystones | `place` | Maps |
| Unshared items in multiworld | none | **Fill policy parameter `local_count`** |
| Snippets (seed language) | none | **Cannot be data**: they are runtime scripts. Their declared metadata (name, category, description, config keys) maps to options; their effects belong to the client, and their logic impact would need declaring as data (`grants_on_start`, `sets_event`) |
| Plando sources | platform plando (doc 07 6.6) | Maps if plando is a placement table, not scripts |
| Slug from settings hash | D3.1 seed phrase idea | Similar idea; ours must also be decodable (settings string) |
| Seed file with compiled commands | `slot_data`, CSDK | **Cannot be data** in the world file; it is the client contract |
