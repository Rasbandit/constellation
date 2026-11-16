# 16. PopTracker: tracker pack format and auto-tracking audit

> Internal research note, 2026-11-16, read at commit `6052adc2eba871512085e7ab2c2e970ebf488822` of the `master` branch of [black-sliver/PopTracker](https://github.com/black-sliver/PopTracker) (shallow sparse clone of `doc/`, `schema/`, `src/core/`, `src/usb2snes/`, `src/uat/`, `src/packmanager/`, `src/ap/`, `examples/`, `api/`, `test/` and root `.md` files; plus the `README.md` and `community-packs.json` of the `packlist` branch through the GitHub contents API; no code was run, built or imported). Not reviewed by a different agent or a human. Our own writing with citations; no code, text or data from the project is reproduced. Numbers are labeled **counted** (by `grep`, `find`, `wc` or the GitHub API on 2026-11-16), **estimated**, or **not verified**.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. Summary
- **What it is.** A cross-platform desktop tracker. It loads community "packs" that describe items, maps, locations with access rules and UI layouts, then updates them by hand or automatically from a console bridge, a game-side variable protocol, or an Archipelago room. It is not a randomizer; it is a widely used tool (00-landscape section 3.5, not measured) for showing what a player has and can reach.
- **Popularity.** 200 stars and 33 contributors including anonymous (both counted via the GitHub API, 2026-11-16; 00-landscape row 45; its batch D table lists 34 contributors, a read on a different call, not reconciled). The community pack list holds 18 entries (counted in `community-packs.json` on the `packlist` branch); many more packs are distributed directly (not measured).
- **License.** GPL-3.0 (API), with a plug-in addendum that allows separately installed plug-ins under other licenses when they talk to PopTracker over an open, external specification (`PLUGIN LICENSE ADDENDUM.md`). For us: read and describe only. Pack compatibility, if ever wanted, is an interface question for a separate decision (00-landscape section 4).
- **Maintenance.** Very active: last push 2026-11-14, 36 open issues (counted via the API). The docs mark each feature with the version that added it, which shows a long run of additive changes.

## 2. Architecture
- **Language.** C++ application with an embedded Lua interpreter (`src/core/scripthost.cpp`). Core model in `src/core/` (pack, item, location, section, layout, map, tracker, auto-tracker, state manager).
- **Backends.** USB2SNES-family websocket for SNES memory (`src/usb2snes/`), a Lua connector for emulators, the UAT variable protocol (`src/uat/`), and an Archipelago client (`src/ap/`).
- **Pack loading.** A pack is a folder or zip. `manifest.json` describes it; everything else is loaded through `scripts/init.lua`, which calls the tracker API to add item, map, location, layout and class JSON files (`doc/PACKS.md:3-6`, `:108-121`).
- **Evaluation.** The tracker evaluates access rules for every location section when items change, caching provider counts per item code (`src/core/tracker.cpp:573-615`). Packs can pause evaluation during bulk updates (`doc/PACKS.md:120-121`).

## 3. World or content model (the pack format)
- **Manifest.** Name, game name, unique pack ID, version, platform, variants with flags, an update URL, and minimum and target tracker versions (`doc/PACKS.md:10-68`). Variants are alternate layouts (for example map or compact) and also switch features: flags enable Archipelago tracking, sending checks, hint scouting, UAT, SNES memory mapping modes and the Lua connector (`doc/PACKS.md:48-58`).
- **Settings.** An optional `settings.json` for display behavior; user overrides merge by key (`doc/PACKS.md:71-84`).
- **Items.** Seven types (counted, `doc/PACKS.md:269-386`): `progressive` (stages, each with image and codes, optional inheritance of earlier codes), `toggle`, `static`, `consumable` (counts with min, max, increment), `progressive_toggle`, `composite_toggle` (two linked items, four images) and `toggle_badged` (an overlay on a base item). Items expose string **codes** that rules test.
- **Maps.** Named images with default marker size, border and shape (5 shapes counted: rect, diamond, trapezoid, triangle, concave kite) (`doc/PACKS.md:389-423`).
- **Locations.** A tree: location, children, sections. Each node may have access rules, visibility rules, images and a parent override; children have map positions (map name, x, y, per-marker overrides and visibility rules); sections have a check count, an optional fixed "hosted" item, and may reference a section elsewhere (`doc/PACKS.md:427-551`).
- **Layouts.** Named widget trees. Reserved roots for default, horizontal, vertical, broadcast and settings views (`doc/PACKS.md:576-587`). 12 widget types (counted, `doc/PACKS.md:609-621`): container, dock, array, tabbed, group, item, itemgrid, map, layout (include by name), recentpins, text, canvas. Common properties cover alignment, orientation, margins, item size and a `class` for inherited defaults (`doc/PACKS.md:589-654`).
- **UI hints.** Lua cannot touch widgets directly; it sends named hints (activate tab, zoom and pan a map) (`doc/PACKS.md:656-666`).

## 4. Logic and constraints (rules in packs)
- **Rule grammar** (`doc/PACKS.md:505-525`): `access_rules` is a list; each entry is a comma-joined AND of terms; entries are OR-ed. A term is an item code (count at least 1), `code:N` (count at least N), `@location/section` (another node's result), or `$func|arg|arg` (a Lua function returning a count or boolean). A leading `^` makes the term return an accessibility level directly.
- **Logic tiers in the grammar.** Terms in `[ ]` are optional: the location is reachable "with sequence break" without them. A group in `{ }` marks "inspectable but not collectable" (you can see the item but not take it). Accessibility levels are six (counted): none, partial, inspect, sequence break, normal, cleared (`doc/PACKS.md:164-167`).
- **Visibility rules** use the same grammar to show or hide locations and markers.
- **Children add to parents.** A child's rules are on top of its parent's, so location trees express region nesting.
- **Validation.** JSON Schemas for 7 pack files plus a stricter set exist; the schema README says they serve documentation and tests, while the real validation lives in the C++ code (`schema/README.md`, paraphrased). A CI workflow checks example packs against the strict schemas with a pack checker action (`.github/workflows/check-examples.yaml`).
- **Example in our words.** "The tower chest needs the hookshot and two keys, or the glider; with the boots it is only reachable as a sequence break; with the lens you can see what it holds."

## 5. Shuffle and placement algorithms
None. The tracker does not place items. Archipelago scouting lets a pack show placed items when the server allows it (`doc/AUTOTRACKING.md:155`, `:162`).

## 6. Entrance or area randomization
No native model. Packs implement entrance tracking in Lua with custom items and rules (inferred; no entrance concept appears in `doc/PACKS.md`).

## 7. Options model
- Pack options are ordinary items (toggles or progressive) shown in a settings layout, and rules test their codes like any item (inferred from the `settings_popup` layout and rule grammar).
- With Archipelago, a pack's clear handler receives the slot data and sets those option items (`doc/AUTOTRACKING.md:136`, `:152`).

## 8. Seeds, reproducibility, spoiler logs, hints
Not applicable to generation. State save and load use the pack ID and version (`doc/PACKS.md:19-20`). Archipelago room seed name is exposed to Lua (`doc/AUTOTRACKING.md:150`).

## 9. Game integration and delivery (auto-tracking)
- **Memory watches.** A pack registers a named watch on an address range with a callback and interval; the callback reads bytes from a segment (`doc/AUTOTRACKING.md:24-77`). SNES address mapping comes from flags or a built-in game table.
- **Variable watches (UAT).** A game or bridge publishes named variables; the pack watches names and reads values (`doc/AUTOTRACKING.md:80-126`). The protocol has its own JSON schemas (`schema/UAT/`, 5 files counted).
- **Archipelago.** Callback-driven: clear (with slot data), item received, location checked, scouted, bounce, data storage get and set-notify replies; with the right flags a pack can send checks, scouts and goal status (`doc/AUTOTRACKING.md:129-167`).
- **Connection states** are a small integer enum per backend (`doc/AUTOTRACKING.md:176-178`).
- **Player steps.** Install PopTracker, install a pack (from the in-app list or a zip), load it, start auto-tracking or connect to a room: about 4 steps (estimated from the docs).

## 10. Multiworld or co-op support
As a client of Archipelago rooms (section 9), including data storage for shared state and a "manual" mode where the tracker itself sends checks for games with no mod (`doc/PACKS.md:49-51`).

## 11. Tooling and testing
- **Lua sandbox.** Only base, table, string, math, utf8 and a reduced `os` (clock and time functions) are loaded; `load`, `loadfile` and `loadstring` are removed; `require` and `io` are replaced with pack-scoped versions; an instruction-count hook every 500,000 instructions lets the host abort a script when a pack unloads (`src/core/scripthost.cpp:612-676`). A debug library loads only when enabled at build and run time.
- **Editor support.** A Lua type definition file for language servers (`doc/PACKS.md:94-102`, `api/lua/`) and schema catalogs for editors (`schema/vs/`).
- **Distribution.** The `packlist` branch holds `community-packs.json` (pack ID to name, author, platform, homepage, `versions_url`); each pack's `versions.json` must be served over HTTPS and list a SHA-256 per zip (`packlist` branch `README.md`). The pack manager rejects non-HTTPS URLs except localhost and checks the hash (`src/packmanager/packmanager.cpp:113`, `:277-285`).
- **Tests.** 42 `test_*.cpp` files (counted) covering core parsing, Lua items, zip, HTTP, UI pieces and Archipelago Lua fixtures; a fuzzing workflow and guide (`doc/fuzzing.md`, `.github/workflows/fuzz.yaml`); example packs as test fixtures (`examples/`, 12 counted).
- **Pack builder.** A third-party script generates the pack skeleton, item and location JSON and Archipelago ID mappings from Archipelago's data package (`doc/PackCreation.md:240-279`).

## 12. Developer ergonomics
- **Make a pack.** Start from a template pack, write JSON for items, maps, locations and layouts, write `init.lua` to load them, add Lua for complex rules and auto-tracking, place markers on map images by pixel coordinates (`doc/PackCreation.md`). The novice guide is 281 lines (counted).
- **Pain the docs name.** Writing JSON for every item and location is tedious, and finding Archipelago item and location IDs is a hurdle (`doc/PackCreation.md:243-246`). The builder script exists to address both.

## 13. Patterns worth standardizing and pain points
Patterns:
1. **Layouts as data.** A small widget vocabulary with named roots for screen shapes and a broadcast view. Portable to a web renderer.
2. **Logic tiers in rules.** Optional terms for sequence breaks and a distinct "inspectable" state give players more than reachable or not.
3. **Hierarchical locations with sections**, and section references so one check appears on several maps.
4. **Watch-based auto-tracking**: declare what to watch, react on change, never poll from pack code.
5. **Capability flags per variant** (which backends and features a pack supports).
6. **Version gating** (minimum and target tracker version) for behavior changes.
7. **Signed-ish distribution**: HTTPS plus per-zip SHA-256 in a community index.
8. **Narrow host API with a sandbox**, and UI changes only through hints.

Pain points:
1. Logic is duplicated: the generator has rules in its language; the pack re-encodes them in strings and Lua, by hand.
2. Lua rules cannot be analyzed, cross-checked or run in a browser without an interpreter.
3. Memory decoding (bits to item codes) is written in Lua per pack, though most of it is a table.
4. Schemas are documentation; real validation lives in C++ and may drift.
5. ID mapping between tracker codes and multiworld IDs is manual.

## 14. Mapping to our standard
Maps to existing constructs:
- Item codes: item `key` (01 D2.2). `code` and `code:N`: `has` with `n` (01 D2.5). AND within an entry and OR across entries: `all` and `any`. `@location/section`: `reach` on a location. Progressive items with stage codes: `progressive` and `level` (07 section 2.3). Consumables: `count`.
- Pack options read from slot data: options in the manifest; a tracker reads them directly from the room.
- Archipelago callbacks: our ordered event log (01 D1.4) gives the same item, location and hint events.
- Version gating and variant flags: capability negotiation (01 D1.3) and `requires` (07 section 2.5).

Needs a new primitive or concept:
- **Tracker layout schema (proposed name: CTL).** A data file next to the manifest (or under `client`, 01 D2.2) with: maps (image key, marker size and shape), map positions per location key, a widget tree with a closed widget set (grid, array, tabs, group, item, map, text, include), named roots (wide, tall, broadcast, settings), and classes for shared properties. Generated defaults from the manifest (an item grid by group, a list by region) would remove most hand JSON.
- **Logic tiers.** Rules today are true or false. A tracker needs at least "normal", "with tricks or sequence breaks" and "inspectable". Proposal: tagged sub-rules (`{"tier": "glitched", "rule": ...}`) that the generator ignores unless an option enables them and the tracker shows as a separate state. Needs checking against 01 D2.7 monotonicity.
- **Watch table (data-only auto-tracking).** A schema mapping memory or variable sources to item and location keys: address or variable name, width, bit mask, compare, the key it sets, and poll interval. Covers the common case in Lua today; complex decoding stays in the game client or a future WASM module (01 D2.8).
- **Section and nesting** in locations: a display grouping (several checks behind one marker), not logic. A small `display` field on locations.

How a tracker could come almost free from a manifest (our sketch, not PopTracker's design):
1. Items: one grid per item group, ordered by ledger order; progressive chains render as one staged item; counts render as badges.
2. Locations: one list per region, with the region's entry rule as the parent rule and each location's rule on top, mirroring PopTracker's parent and child stacking.
3. State per location from the same rule evaluator the generator uses (07 section 2.4, item 4), with tiers mapped to colors: reachable, reachable with tricks, inspectable, not reachable, checked.
4. Auto-tracking from the room's event log (01 D1.4), so no per-game Lua is needed when the game runs a CSDK client.
5. An optional CTL file only for what cannot be inferred: map images, marker positions, custom grouping and the broadcast view.
6. Validation at publish time: every location key in CTL exists in the ledger, every map position names a declared map, and every widget references a known item key (a 07 section 5.1 style static check).
This keeps pack authors out of rule writing entirely; the remaining hand work is art and marker placement, which the PopTracker guide also leaves manual (`doc/PackCreation.md:266-272`).

Cannot be data, and why:
- Arbitrary Lua rules and click handlers: general computation. Our position (01 D2.5, rejected embedded Lua) keeps these out; packs that need them stay outside the standard.
- Pixel art, map images and fonts: assets, governed by licenses and our CPK rules, not by the manifest.

Not verified: pack checker action rules, how many community packs exist outside the list, Lua connector details, entrance-tracking practice in real packs.

## Sources read (paths in the clone, 2026-11-16)
- `doc/PACKS.md` and `doc/AUTOTRACKING.md` (in full); `doc/PackCreation.md` (outline and builder section).
- `schema/README.md`; `schema/packs/` and `schema/packs/strict/` (listing, start of `layouts.json`); `schema/UAT/` (listing).
- `src/core/scripthost.cpp` (sandbox setup), `src/core/rule.h` (rule parsing), `src/core/tracker.cpp` (caching).
- `src/packmanager/README.md`, `src/packmanager/packmanager.cpp` (URL and hash checks).
- `PLUGIN LICENSE ADDENDUM.md`; `.github/workflows/check-examples.yaml`; `test/` (file listing).
- `packlist` branch via the GitHub contents API: `README.md`, `community-packs.json` (entry count and first entry's keys).
- GitHub API: repository and contributors endpoints.
