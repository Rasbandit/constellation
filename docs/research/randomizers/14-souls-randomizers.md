# 14. SoulsRandomizers (Dark Souls III, Sekiro, Elden Ring): architecture and patterns audit

> Internal research note, 2026-11-16, read at commit `0f4c316ac5728fed7db78e3a54bd307cfd43bd8e` of the `master` branch of [thefifthmatt/SoulsRandomizers](https://github.com/thefifthmatt/SoulsRandomizers) (shallow sparse clone of `RandomizerCommon/**/*.cs`, `dist*/Base/*.txt` and root `.md` files; no code was run, built or imported). Not reviewed by a different agent or a human. Numbers are labeled **counted** (by `grep`, `wc` or `git ls-tree` over the clone, or the GitHub API on 2026-11-16), **estimated**, or **not verified**.
>
> **License warning.** This project uses a custom source-available license (`LICENSE.md`, read in full). It allows viewing and forking the unmodified source and private personal modification, and forbids conveying derived works, hosting the software or a substantial part of its features as a service, and programmatically invoking it from distributed software; its data files are covered too (`CONTRIBUTING.md`, "Using randomizer code elsewhere"). **We read it for design ideas only.** This note describes behavior at a high level in our own words, names no algorithm details beyond what is needed to state an idea, and we must never host, wrap, invoke or port it, or derive our data from its data files.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. Summary
- **What it is.** Item and enemy randomizers for FromSoftware games, shipped as PC mods. The README links mod pages for Elden Ring, Dark Souls III and Sekiro (`README.md`); code also references Dark Souls Remastered and a newer title enum (`RandomizerCommon/Core/RandomizerOptions.cs:141`, `Randomizer.cs:187`). Beyond items and enemies it edits characters, gestures, materials and hints (`RandomizerCommon/Misc/`, `RandomizerCommon/Elden/`).
- **Popularity.** 94 stars and 2 contributors (both counted via the GitHub API, 2026-11-16; 00-landscape row 27). Popularity is mostly off GitHub, through mod sites (not measured).
- **License.** See the warning above. In short: no reuse of any kind; ideas only.
- **Maintenance.** Active: last push 2026-10-27 (counted via the API). Contributions are accepted since 2026 under a CLA, with a stated policy against substantial LLM-written code (`CONTRIBUTING.md`). We are not contributing.

## 2. Architecture
- **Language and layout.** C# (.NET 6 and 8 per `CONTRIBUTING.md`). One shared library `RandomizerCommon/` (60 `.cs` files and 50,003 lines in our sparse set, counted) plus per-game front ends (`DS3Randomizer/`, `SekiroRandomizer/`, `EldenRingRandomizer/`) and two UI toolkits (WinForms, moving to Avalonia per `CONTRIBUTING.md`).
- **Subsystems.** `Item/` (12 files: location data, annotations, key item placement, general permutation, writer, presets, external placement), `Enemy/` (13 files, the largest is `EnemyRandomizer.cs` at 9,931 lines, counted), `Core/` (game data loading, options, merge manifest, messages), `Elden/`, `Sekiro/`, `DS3/`, `Misc/`.
- **Dependencies.** Format libraries split into separately licensed repositories (named in `CONTRIBUTING.md`) handle the game's archive and parameter formats.
- **Pipeline** (high level, from `RandomizerCommon/Randomizer.cs` and file names): load the player's game files and any mods to merge; load annotation data; place key items by area; place the remaining items; randomize enemies within silos under placement checks; apply scaling; write edited game files and logs.
- **Running it.** A desktop GUI. The license permits only the GUI path for unmodified use (`LICENSE.md` grant 1).

## 3. World or content model
Per-game data lives in YAML-formatted `.txt` files: `dist/Base/` (Dark Souls III), `dists/Base/` (Sekiro), `diste/Base/` (Elden Ring) (mapping inferred from file contents and option names).
- **Annotations** (`annotations.txt`): sections for config options, item groups, starting groups, events, areas, placement restrictions, special modes, item priority and slots (9 top-level sections in Dark Souls III, counted).
- **Counts** (counted with `grep`): Dark Souls III 34 areas and 1,279 item slots; Sekiro 60 areas and 866 slots; Elden Ring 363 areas, with 4,379 slots in a separate `itemslots.txt`.
- **Slots** carry a stable key, display text, area, tags (for example boss, missable, no-randomize) and an optional event.
- **Areas and events** carry a requirement expression string over areas, events, item names and config flags with `AND`, `OR` and parentheses.
- **Enemies** (`enemy.txt`): one entry per placed enemy with map, class, tags, events, ownership (helpers attached to a main enemy), arena data and flags. Class entries counted: Dark Souls III 2,544; Sekiro 1,542; Elden Ring 23,173. Classes include basic, boss, miniboss, helper, hostile NPC and game-specific ones; Elden Ring has 7,919 wildlife and 12,399 basic entries (counted).
- **Enemy classes** are a code enum with comments that define intent (must be killable, moves with its owner, not randomized) (`RandomizerCommon/Enemy/EnemyAnnotations.cs:708-740`), plus class groups for presets (`:804-816`).
- **Hint markers** (`diste/Base/hintmarks.txt`, 931 lines counted) list in-world positions for hint placement.

## 4. Logic and constraints
- **Area graph logic.** Requirements are strings over named areas, events and options, parsed in C#. Config flags (for example skips and DLC on or off) are first-class terms in the expressions.
- **Placement windows.** Per-item restrictions say how many copies may appear up to an area, before or after an area, and separately for shops and drops; restrictions switch on options. This lets "upgrade materials stay rare early" be data rather than code.
- **Item priority.** An ordered list of item groups and tag exclusions decides what is placed first.
- **Race mode.** Special modes restrict key items to categories of "race locations" chosen by options (for example chests, rings, minibosses).
- **Enemy placement safety.** A single predicate decides whether a source enemy may go to a target (`RandomizerCommon/Enemy/EnemyRandomizer.cs:1673`). Rules include: a size tag on the source must not exceed an arena tag on the target; avoid same model or same NPC in one encounter; progression-related hard requirements. A few exceptions are keyed by entity ID in code (`:1782-1801`).
- **Explain mode.** The same predicate can print why each candidate was rejected (`:1760-1801`). That makes constraint failures debuggable.
- **Scaling.** Each game defines scaling tiers (5 in Sekiro, 20 in Dark Souls III, 34 in Elden Ring) with per-tier multipliers; a moved enemy is scaled from its source tier to its target tier (`RandomizerCommon/Enemy/ScalingEffects.cs:27-172`).
- **Example in our words.** "A boss of size 3 may replace another boss only where the target arena allows size 3 or more; when it moves from tier 4 to tier 12, its health and damage are scaled by the ratio of those tiers' multipliers."

## 5. Shuffle and placement algorithms
- **Key items by area, not by location.** Key items are first assigned to areas, using a measure of how late each area is and a difficulty bias that pushes items later; equivalent areas are merged for weighting (`RandomizerCommon/Item/KeyItemsPermutation.cs`, comments around lines 21-27, 386, 425-459). Comments acknowledge some option mixes fail often (`:459`), so success is not guaranteed in one pass.
- **Remaining items.** A general permutation fills the rest with weighted choices and a bounded attempt loop of 10,000 (`RandomizerCommon/Item/Permutation.cs:1804-1833`).
- **Enemies.** Assignment inside silos (none, DLC or region; `RandomizerCommon/Item/LocationData.cs:150-156`), with forced mappings, helper swaps and phase balancing heuristics (`RandomizerCommon/Enemy/EnemyRandomizer.cs:1396`, `:1613`, `:1965`).
- **User enemy presets.** YAML presets (`RandomizerCommon/Enemy/EnemyPreset.cs`) declare weighted pools per class, an "everything is X" mode, enemy multipliers, do-not-randomize and remove-from-pool lists, and explicit target-to-source maps. This is a user-facing content-shuffle language in data.

## 6. Entrance or area randomization
Not in this repository's item and enemy code. Code refers to a separate fog gate randomizer that can be merged, with an explicit merge-order check (`RandomizerCommon/Randomizer.cs:44-45`, `RandomizerCommon/Item/KeyItemsPermutation.cs:153`). Not read.

## 7. Options model
- **Declared in data and code.** Dark Souls III annotations declare 15 boolean options, 5 one-of groups and 5 numeric options (counted). The UI side was not read.
- **Options string.** A space-separated list of flag names, `key:value` pairs, a version token, and positional numbers for difficulty, item seed and enemy seed (`RandomizerCommon/Core/RandomizerOptions.cs:150-220`). Named flags make it order-independent, unlike a bit-packed string.
- **Presets.** Item and enemy presets as YAML files; the license explicitly allows sharing exported options files, presets and spoiler logs (`LICENSE.md` grant 2).
- **Mod merge config.** Other mods can ship a `randomizer_merge_config.yml` listing things not to randomize (`CONTRIBUTING.md`), read into a merge manifest of excluded entities, lots, shops and items (`RandomizerCommon/Core/MergeModManifest.cs:15-27`).

## 8. Seeds, reproducibility, spoiler logs, hints
- **Two seeds.** Items and enemies have separate seeds (`RandomizerCommon/Core/RandomizerOptions.cs:154-155`), so changing enemy options does not reshuffle items.
- **String seeds.** Non-numeric seeds are hashed to 32 bits (`RandomizerOptions.cs:50-60`).
- **Sub-streams.** Several enemy steps build their own RNG from the seed plus a constant (`RandomizerCommon/Enemy/EnemyRandomizer.cs:1257-1521`), keeping steps from disturbing each other. .NET's `System.Random` algorithm is a runtime detail (from memory, not verified), so cross-runtime reproduction is not guaranteed.
- **Spoiler logs.** Written to a `spoiler_logs` directory (`RandomizerCommon/Item/PermutationWriter.cs:894`); a hint section lists items by hint category (`:107-132`).
- **In-game hints.** Elden Ring places hint markers in the world (`RandomizerCommon/Elden/HintMarker.cs`, 1,861 lines counted).

## 9. Game integration and delivery
- **Method.** Reads game files from the player's own installation, writes edited files into a mod directory loaded by a third-party mod loader, and can merge other mods' files first (`RandomizerCommon/Randomizer.cs:126-233`). A mismatch between the edited parameter file and other files is detected and reported (`:34`).
- **Player steps.** Download the mod, unpack into the game folder, run the GUI, pick options, randomize, launch through the mod loader: about 5 steps (estimated from `CONTRIBUTING.md` and code paths).

## 10. Multiworld or co-op support
- **External placement input.** Behind an `ARCHIPELAGO` compile symbol (2 uses counted), an external preset maps external item and location IDs onto local slots and fully overrides the finite item pool; everything not covered stays vanilla, and unmapped locations are reported as a list of errors (`RandomizerCommon/Item/ExternalItemPreset.cs`, `RandomizerCommon/Item/ExternalLocationData.cs:1-104`). It also carries forced boss targets and shard counts.
- Archipelago's in-tree `dark_souls_3` world is separate (00-landscape row 27; directory confirmed locally, not read).

## 11. Tooling and testing
- **Tests.** No test project at the repository root (counted from the root listing). Data validators and auto-update flows for Elden Ring configs are mentioned (`CONTRIBUTING.md`, "Codebase state"); `RandomizerCommon/Core/RuntimeParamChecker.cs` exists (not read).
- **Data scraping.** `RandomizerCommon/Elden/EldenLocationDataScraper.cs` (1,337 lines) builds location data from game files (inferred from the name; not read in detail).

## 12. Developer ergonomics
- **Add an item slot or area.** Edit YAML annotations; logic strings and restrictions are data.
- **Add an enemy rule.** Usually C# in the placement predicate; tags like size and arena are data.
- **Docs.** `CONTRIBUTING.md` explains build layout (sibling repositories by relative path), runtime files not in source, and scope. No architecture doc.

## 13. Patterns worth standardizing and pain points
Patterns (ideas only, described in our words):
1. **Integer fit tags.** "Source size must not exceed target capacity" as two integer tags is a general, data-expressible placement constraint.
2. **Scale by placement tier.** Content moved to a different depth is rescaled by a tier table. This is the content analogue of progression balancing.
3. **Placement windows by area depth.** "At most N copies before area X, none in shops until area Y" as data.
4. **Silos.** Shuffle only within a declared partition (DLC, region).
5. **Explainable rejections.** Every constraint can say why it refused a candidate.
6. **Merge exclusion lists** shipped by other mods, so randomizers coexist with mods.
7. **Separate seeds per subsystem** and named flag options rather than bit positions.
8. **External placement injection**: a single-player randomizer that can accept a placement computed elsewhere.

Pain points:
1. Special cases keyed by entity ID in code; some placement heuristics are not declared.
2. Area-level key item placement with acknowledged failure rates in some settings.
3. No visible automated tests; reproducibility depends on runtime RNG details.
4. Restrictive license blocks reuse and hosting, so ideas cannot be shared as code.

## 14. Mapping to our standard
Maps to existing constructs:
- Areas with requirement strings and option flags: `regions` and rules with `opt` (01 D2.2, D2.5). Events: `events`. Slot tags: location `tags` and `kind` (`excluded` for missable).
- Do-not-randomize and merge exclusion lists: location `default_item` plus `place` (07 section 2.3). Item priority: `place` ordering, not quite.
- Race location categories: `item_rule` (07 section 2.3) on location groups chosen by options.
- Two seeds: per-hook streams (01 D3.1).
- External placement injection: our generator produces the placement and the game mod reads it, which is the normal CSDK path (01 section 3.5).

Needs a new primitive or concept:
- **`permute` with `fits`.** A constraint on a source-to-target permutation: source integer tag must be at most target integer tag (and the same-model and same-NPC "not equal" rules). Evidence: Souls arenas, UPR similar strength (13), KH2 boss compatibility via an external library (15, not read).
- **`scale_by_tier`.** A post-shuffle transform: each placed content row gets stat multipliers from a tier table indexed by (source tier, target tier). Must be integer (fixed-point ratios) for reproducibility.
- **`depth_window` item rule.** "At most N of item X with depth below D" where depth is a derived per-region number (sphere index or declared tier). Needs a normative definition of depth; doc 07 has none today. KH2 depths (15) are a second use.
- **Silo.** A grouping key that partitions any `permute`; probably an option of `permute` rather than a new primitive.
- **Explain output.** A standard diagnostic shape for primitive rejections (code, candidate, rule) to go with 01 D1.7 error codes.

Cannot be data, and why:
- Writing game files, event scripts and arena geometry fixes (`RandomizerCommon/Enemy/EnemyRandomizer.cs:2905-2964`): binary formats and per-encounter scripting, owned by the game mod, not the manifest.
- Phase-balancing heuristics and per-entity exceptions: possible as data only if the primitive exposes per-row overrides; otherwise they are code.

Not verified: UI options list, test or validator code, the fog gate integration, exact Elden Ring option counts.

## Sources read (paths in the clone, 2026-11-16)
- `LICENSE.md`, `README.md`, `CONTRIBUTING.md` (in full).
- `dist/Base/annotations.txt`, `dists/Base/annotations.txt`, `diste/Base/annotations.txt` and `diste/Base/itemslots.txt` (section names, key names and counts only).
- `dist*/Base/enemy.txt` (class counts and key names only), `diste/Base/hintmarks.txt` (structure only).
- `RandomizerCommon/Enemy/EnemyAnnotations.cs` (class enum), `EnemyPreset.cs` (preset fields), `ScalingEffects.cs` (tier fields), `EnemyRandomizer.cs` (placement predicate, RNG sub-streams; comments).
- `RandomizerCommon/Item/KeyItemsPermutation.cs` and `Permutation.cs` (comments and attempt bound), `LocationData.cs` (silo enum), `ExternalItemPreset.cs`, `ExternalLocationData.cs`, `PermutationWriter.cs` (hint and log output).
- `RandomizerCommon/Core/RandomizerOptions.cs` (options parsing), `MergeModManifest.cs`, `RandomizerCommon/Randomizer.cs` (output and merge paths).
- GitHub API: repository and contributors endpoints.
