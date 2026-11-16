# Batch D summary: content randomizers, PC mods and tooling

> Internal research note, 2026-11-16. Covers [13](13-universal-pokemon-randomizer-zx.md) (UPR ZX), [14](14-souls-randomizers.md) (SoulsRandomizers), [15](15-kh2-randomizer.md) (KH2 Randomizer) and [16](16-poptracker.md) (PopTracker). Not reviewed by a different agent or a human. Our own writing; every claim below is cited in the per-project note it comes from. SoulsRandomizers is under a restrictive custom license and was read for ideas only (14, license warning). Numbers carry the labels of their source note.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. What this batch adds
Batches A to C were mostly about item-location logic. This batch shows the other half of what players call "a randomizer": **content shuffles** (stats, moves, encounters, enemies, shops, level curves), **PC mod delivery** (game files from the player's install, mod loaders, merge with other mods) and **trackers** (layouts, rules for display, auto-tracking). Two of the four projects have no progression logic at all (UPR ZX) or no generation at all (PopTracker), which is why they were picked.

## 2. Cross-project patterns
1. **Content is tables; algorithms are shared.** UPR ZX runs one implementation of each shuffle across 7 generations and 166 ROM sections (counted, 13 section 3). Souls keeps per-game YAML for slots, areas and 1,542 to 23,173 enemies per game (counted, 14 section 3). KH2 keeps level, weapon and attack tables (15 section 3). All three would fit `tables` (01 D2.2).
2. **Constraints are small integer comparisons.** Stat floors and a 255 ceiling (13 section 4); "size tag at most arena tag" (14 section 4); stat-total windows that widen by 5% (13 section 4); weapon stats in a min and max range (15 section 5). Nearly all are expressible as integer tags plus a comparison.
3. **Depth is everywhere and never defined the same way.** Souls area lateness and "up to area X" windows (14 sections 4-5), KH2 named depth categories and rarity-by-depth weight curves (15 sections 4-5), Souls scaling tiers (14 section 4). Each project invents its own notion of "how far into the game".
4. **Placement follows a related row.** Evolutions follow their base form in UPR ZX (13 section 4); helpers move with their owner in Souls (14 section 3). Shuffles need relations between rows, not only filters.
5. **Retries are local and bounded, then global.** 100 redraws and 3 widening rounds (13), 10,000 attempts (14), 100 draws and 50 whole-seed attempts with a new seed name (15). The last one changes the seed the user asked for.
6. **Seeds are split by subsystem.** Cosmetic and main streams (13), item and enemy seeds (14), spoiler flag mixed into the seed (15). This matches our per-hook streams (01 D3.1).
7. **Settings strings are positional and version-locked** (13 section 7, 15 section 7), except Souls' named-flag string (14 section 7). All three need the same program version to reproduce.
8. **Floats in weighting** (13 section 8, 15 section 8). Any port to an integer-only standard must restate formulas as integers.
9. **Trackers re-encode logic by hand.** PopTracker rules are AND-OR strings over item codes plus Lua escapes (16 section 4); KH2 has its own tracker that reads the seed zip (15 section 11). The tracker never sees the generator's rules.

## 3. Strongest ideas to adopt
1. **Content shuffles as named primitives over tables.** A world declares "redistribute these columns within the row total, floor 10, ceiling 255, follow evolution links" instead of shipping code. Evidence: 13, 15.
2. **A tracker layout schema in the standard.** PopTracker's layout vocabulary (12 widget types, named roots for wide, tall, broadcast and settings views; counted, 16 section 3) is proven at scale and small. If the manifest already has items, groups, regions and rules, a layout file plus map positions is the only missing piece for a web tracker that needs no hand-written logic.
3. **Logic tiers.** PopTracker's optional terms (sequence break) and "inspectable" state (16 section 4) are what players use. Our rules need a tier tag so a tracker can show more than true or false.
4. **Integer fit tags and tier scaling for enemy shuffles** (14 section 4), stated in our own words and integer arithmetic.
5. **Explainable rejections.** Souls can print why each candidate failed (14 section 4). Our primitives should return a structured reason with an error code (01 D1.7).
6. **Data-only auto-tracking tables.** Most PopTracker Lua auto-tracking maps bits to item codes (16 section 9). A watch table schema covers that without Lua.
7. **Check value or hash icons in game** (13 section 8, 15 section 8): our bundle hash shown through CSDK.
8. **Merge exclusion lists for PC mods** (14 section 7): a CPK-side "do not touch" list another mod can ship.

## 4. Recurring pain points
- Logic or constraints as code (Java, C#, Python callables, Lua) that no other tool can read (13, 14, 15, 16).
- Implicit step ordering in content pipelines (13 section 2).
- Reproducibility tied to one program version, one runtime RNG, floats, or reseeding on failure (13, 14, 15).
- Thin automated testing: none in UPR ZX (counted), none visible in Souls, seed tests outside CI with time-based names in KH2 (15 section 11). PopTracker is the exception (42 test files, fuzzing; 16 section 11).
- Manual ID mapping between trackers and multiworld IDs (16 section 12).

## 5. Candidate primitives and standard features
Each would still need the rule of three (07 section 2.4); the evidence column says how close it is.

| Candidate | What it does | Evidence in this batch | Status |
|---|---|---|---|
| `redistribute@x` | Change integer columns of a row within a kept or ranged total, with floors, ceilings, integer weights, largest-remainder rounding, and a follow-relation mode | UPR stats (strong), KH2 level and weapon stats (partial) | Needs a third world |
| `permute` option `fits` | Source tag at most target tag; plus "not equal" rules on a column | Souls arenas, UPR strength windows | Extension, not new |
| `permute` or `select` option `related` | Copy a decision along a declared relation (evolutions, helpers) | UPR, Souls | Extension |
| `select` option `band` | Nearest by integer key with a widening window and minimum candidates | UPR similar strength | Extension |
| `scale_by_tier@x` | Rescale moved content by a tier table (fixed-point) | Souls, three games in one project | One author only |
| **Depth** (concept) | Normative per-location integer, derived (sphere) or declared (tier), usable in `item_rule`, weights and scaling | Souls, KH2 | Strong; define in 01 D2.7 |
| `weighted_place@x` | Choose a location for an item by integer weights over (item class, depth) | KH2 | Needs more worlds |
| **Stages** (concept) | Ordered primitive stages, each reading earlier outputs | UPR pipeline | Needed for any content world |
| **Logic tiers** (concept) | Tagged sub-rules for glitched or sequence-break logic and an "inspectable" state | PopTracker, OoTR tricks (batch A) | Strong |
| **Tracker layout schema (CTL)** | Maps, positions, widget tree, named roots, classes | PopTracker | Strong; a format, not a primitive |
| **Watch table** | Memory or variable source to item and location keys | PopTracker | Strong for SNES-era games |
| Option flag `affects_seed` | Separate shared from private (cosmetic) settings | KH2 | Small addition to 01 D2.3 |

## 6. What cannot be data
ROM and archive layouts, compression, text encoding, code patches, event scripts and arena geometry (13, 14, 15 section 9); Lua rules and click handlers in packs (16 section 14); art assets. These stay with the patch producer, game client or a future WASM module (01 D2.8).

## 7. Could not read, or not verified
- KH2's boss and enemy logic lives in an external library (`khbr`) that was not cloned (15 section 5).
- Souls UI options, validators and fog gate integration were not read; the license limits how much detail we should record (14).
- UPR ZX GUI string bundle, race mode effect and preset file contents (13).
- PopTracker pack checker action rules and real-world entrance tracking in packs (16).
