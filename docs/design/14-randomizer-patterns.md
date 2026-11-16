# 14. Randomizer patterns: what 16 projects show, and what our standard should absorb

> Internal design draft, 2026-11-16. Reviewed by a different agent on 2026-11-16 (adversarial review on PR 1); not reviewed by a human. Under [`CLAUDE.md`](../../CLAUDE.md) nothing here leaves this repo until that review and a human sign-off. Our own writing: it cites the research notes in [`../research/randomizers/`](../research/randomizers/) (00 landscape, project notes 01 to 16, batch summaries A to D) and reproduces no project's code, text or data. Every number is labeled **counted** (taken from a note, which says how it was counted, or counted by us over the notes), **estimated**, **design value** or **not verified**. Every "count of 16" below is **counted by us from the notes on 2026-11-16**; a note credits a feature only where its author read the code, so a project without credit may still have the feature in code nobody opened. Terms: [`GLOSSARY.md`](GLOSSARY.md). Decision ids (D-nn) are in [`DECISIONS.md`](DECISIONS.md); rows proposed here are D-102 to D-114.

**Project key** (note number and short name, used everywhere below): 01 OoTR, 02 SoH randomizer, 03 OoTMM, 04 wwrando, 05 Randovania, 06 RandomizerCore plus RandomizerMod (Hollow Knight), 07 wotw-seedgen (Ori), 08 LADXR, 09 ALttPR, 10 VARIA, 11 Worlds Collide, 12 z17 (ALBW), 13 UPR ZX, 14 SoulsRandomizers (read for ideas only; restrictive license), 15 KH2 Randomizer, 16 PopTracker.

**Independence rule used for admission.** 01 and 02 count as one author group, because 02's fill and hint code descends from 01 through the OoT3D lineage and carries its MIT notice (note 02 section 1). 03 is a separate codebase and team for the same two games, so it is a second author group but the same game family. The other projects are independent of each other as far as the notes show; shared contributors were **not verified**.

## 1. Executive summary

**How randomizers are built.** Strip away the game-specific writer and the 16 projects run one pipeline: resolve settings, build a world graph, pin restricted items, fill in ordered stages with a capped retry, prove the goal is reachable, compute hints from that proof, emit a spoiler or playthrough, derive a seed identity for players to compare, then hand placements to a patcher, mod or runtime. 14 of 16 place items under an access condition (all but 13, a content randomizer, and 16, a tracker; counted). The logic itself is almost always boolean over items, counts, events and options, with options folded to constants before search (01, 03, 04, 05, 08; batch A pattern 2). The shape is data even where it is spelled in code.

**Where the field is fragmented.**
- **Logic representation: five styles.** Host-language code in 7 projects (02, 08, 09, 10, 11, 12, 15), expression strings in data files in 6 (01, 03, 04, 06, 14, 16), one custom language (07), one typed graph database (05), one cost-valued rule model (10) (counted). Code-held logic can only be read by its own runtime, so trackers and other platforms rewrite it by hand: PopTracker packs re-encode rules (16 section 13), OpenTracker re-encodes ALttPR (09 section 13), SoH logic exists again in Python in an external Archipelago fork (02 section 10), and Archipelago vendors LADXR and VARIA whole (08 section 1, 10 section 1).
- **Settings strings: five encodings.** Positional bit-packed and version-bound in 6 (01, 04, 05, 06, 13, 15), difference from defaults in 2 (03, 08), named flags in 2 (11, 14), an undecodable hash slug in 1 (07), none in 2 (02; 12, which shares preset files instead); 09 and 10 key seeds by a server-side id; 16 generates nothing (counted).
- **Randomness.** Most use the language's default generator (Python `random`, .NET `System.Random`, Java `java.util.Random`); only 07 uses a named library algorithm (PCG) and tests seeds across operating systems; 03 ships its own small xorshift-family generator, not tested across operating systems; 09 uses an unseedable generator, so a seed exists only as a stored result (notes 03, 07, 09, batch B section 3).
- **Testing.** Thin or absent for logic in 02, 10, 11, 12, 13 and 14 (counted from those notes); strong in 05 (349 test files, counted) and 16 (42 test files plus fuzzing, counted).
- **Difficulty.** Named trick toggles, ordered tiers, per-trick levels, numeric costs and skill profiles all exist, each in a different spelling (section 2.3).

**What a central standard can absorb.** The common shape fits a small, closed core plus a few versioned extensions:
1. The rule tree with macros (already doc 01 D2.5 and doc 07), with the authoring rules the field converged on (parentheses when mixing and/or, options folded first).
2. A **difficulty model**: a trick catalog plus ordered tiers, both folding to option tests so rules stay monotone.
3. **Declared fill stages** with bounded, in-stream retries as part of the reference fill (CFill v1).
4. **Entrance shuffle parameters** on `rewire` (typed pools, reverse pairing, coupled, decoupled, mixed, invariants).
5. A **platform hint engine** fed by the same reachability proof.
6. **Seed identity and secrecy**: option scopes (seed, knowledge, cosmetic), an in-game seed code from the bundle hash, a spoiler policy enum, commit-reveal instead of secret build keys.
7. **One solver specification with two implementations** serving generator, validator, tracker and plando.
8. **Integrity checks and golden seeds** in the author workflow and the conformance suite.
9. A **tracker generated from the manifest**, with an optional layout file.

What stays out of the format: ROM, disc and game-file writers, runtime hooks, Lua or scripting in rules, tile-level map generation (08), and order-sensitive item effects (06). Those are the residue already named in doc 07 section 6.

**What we should not standardize**, though several projects do it: secret build keys or private race modules (04, 09, 10), drawing a new seed when generation fails (12, 15), coercing conflicting options instead of rejecting them (silently in 02 and 04; with a printed or reported change in 08 and 13), positional settings strings, and floating-point weights (13, 15).

## 2. Pattern catalog

**Evidence quality.** High: described with file citations in three or more notes from two or more author groups. Medium: cited in one or two notes, or only in one game family. Low: inferred or marked not verified in the notes.
**Verdicts.** Core: part of the core standard (CGM, CWP or the conformance suite) in v1. Ext: optional, versioned extension, Experimental (`x.` prefix) until it meets doc 07 section 2.4. Tool: a platform service or authoring tool, not part of the file format. Reject: not in the standard.

### 2.1 Logic representation and authoring

| Pattern | What it is | Projects | Count | Evidence | Verdict and reason |
|---|---|---|---|---|---|
| Logic as host-language code | Closures, lambdas, function pointers or classes per region | 02, 08, 09, 10, 11, 12, 15 | 7 | High | **Reject** for the format. One runtime reads it; every other tool re-encodes it (section 1). The world author restates it as data. |
| Expression strings in data files | Rules as strings inside JSON, YAML or text, parsed by the project | 01, 03, 04, 06, 14, 16 | 6 | High | **Core** as the authoring surface only (doc 07 section 4.4); the canonical form stays the tree. Adopt 03's and 04's rule that mixing and/or without parentheses is a parse error (notes 03 section 4, 04 section 4). |
| Custom logic language with its own parser and language server | Indentation blocks, tiers per block, parameterized built-ins | 07 | 1 | Medium | **Tool**: a language server for our authoring profile (doc 07 section 4.6). A second language is the cost 07 carries with a small team (07 section 13). |
| Typed graph database with an editor | Regions, areas, typed nodes and docks as JSON trees, edited in a GUI, with a committed text print per file | 05 | 1 | High (read in depth) | **Core** for the canonical model (already trees) plus **Tool**: the visual editor and a canonical text print for review diffs (batch B section 2 item 2). |
| Cost-valued rules | Each rule returns pass or fail, an integer difficulty and the techniques used | 10 (coarse forms: 07, 12 tiers) | 1 | Medium | **Ext** `x.cost`, for estimation only (section 3.3). |
| Rule text kept for display | Rule source shown to players as a tooltip or committed print | 02, 05, 16 | 3 | Medium | **Tool**: the canonical pretty-printer and `keep_macros` (doc 07 section 3.2) give this for free. |
| Options folded before search | Settings and tricks become constants before reachability runs | 01, 03, 04, 05, 08 | 5 | High | **Core**, already: `opt` resolves before logic (doc 01 D2.5). 03 enforces it in the parser (03 section 4). |
| Tagged non-monotone escape | Negation or "less than" in rules, flagged as unsafe | 05 (`negate`), 06 (`<`, `=` documented as unsafe) | 2 | Medium | **Core**, already: the `non_monotone` tag (doc 01 D2.7). |

### 2.2 Macros and templates

| Pattern | What it is | Projects | Count | Evidence | Verdict and reason |
|---|---|---|---|---|---|
| Named, parameterized macros in data | Shared rule fragments, inlined by substitution | 01 (156), 03 (992 across three files), 04 (307), 05 (670 templates), 06 (162), 07 (2 top-level macros); all counted in the notes | 6 | High | **Core**, already (`defs`). The static cycle check matters: 03's parser had none found (03 section 4). |
| Helper predicates in code | "Can light torches", "can kill most things" | 02, 08, 09, 10, 12, 15 | 6 | High | Same need as above, spelled in code; together 12 of 16 factor logic into named helpers (counted). |
| Cached reachability facts (waypoints) | A named fact granted when its rule holds, distinct from substitution | 06 (waypoints), 05 (events), 01 (`here` and `at` local events) | 3 | Medium | **Core** as `events` (exists). `here` and `at` become an authoring macro that compiles to an anonymous event. |
| Option-gated content layers | Groups of nodes switched on by settings | 05 (layers), 03 (2,338-line transform in code), 07 (snippets) | 3 | Medium | **Core** via `templates` with `when` (doc 07 section 3.1); a `layer` shorthand is **Tool** sugar. |
| Capability helpers from tables | Enemy or obstacle tables feed helpers, so a shuffle changes logic through data | 08, 04 (enemy requirements), 14 (size and arena tags) | 3 | Medium | **Core** via `tables` plus macros keyed by a shuffled column. |

### 2.3 Logic difficulty

| Pattern | What it is | Projects | Count | Evidence | Verdict and reason |
|---|---|---|---|---|---|
| Trick catalog | Named techniques with tags (area, difficulty, glitch) and docs, referenced in rules | 01 (373), 02 (202), 03 (154), 05 (187), 06 (21), 07 (32), 08, 09 (31 flags), 10 (105) (counts from notes) | 9 | High | **Core**: a `tricks` section and a `trick` test (section 3). Options alone lose the documentation, tags and UI grouping (05 section 14). |
| Ordered tiers | A world-level ordered difficulty; each tier adds alternatives | 01 (three levels), 04 (two axes of four), 07 (four tiers inside each rule), 08 (five levels), 09 (five levels as trick presets), 12 (five tiers per rule) | 6 | High | **Core** as authoring sugar over an ordered option (section 3.3). 07 and 12 show "harder is a superset" by construction; 08 shows the cost of scattered conditionals (08 section 13). |
| Per-trick ordered levels | Each trick has its own level, not one global level | 05 (six levels), 02 (difficulty tag per trick), 10 (difficulty per technique) | 3 | Medium | **Core** as optional `levels` on a trick. |
| Difficulty rubric and documentation rule | Written rubric; tricks above a level must link a video | 05; prose per trick in 04, 08; video links in 03 | 4 | Medium | **Core** static check (warning): undocumented trick above a declared level. |
| Numeric cost per rule leaf | AND sums, OR takes the cheapest branch | 10 | 1 | Medium | **Ext** with skill profiles (section 3). |
| Skill profile separate from seed settings | "What I can do" versus "what I want" | 10 (skill and rando presets), 05 (per-trick levels in a preset, partly) | 2 | Medium | **Ext**, a settings-layer object mapping tricks to enabled, level or cost. |
| Difficulty window at generation | Reject fills whose solver difficulty falls outside a range | 10 | 1 | Medium | **Tool**, parked: a generator option once `x.cost` exists. It multiplies generation work by the expected attempt count (10 allows up to 500 layout attempts, counted, 10 section 5). |

### 2.4 State dimensions

| Pattern | What it is | Projects | Count | Evidence | Verdict and reason |
|---|---|---|---|---|---|
| Finite context axes | Reachability per (region, age, time of day), with declared switch points | 01, 02, 03 | 3 (one game family) | High within the family | **Ext**, recipe first (section 3.2). |
| Timeline slices with flow and stay | 46 Majora's Mask time slices (counted), rules true within slices, areas where waiting is blocked | 03 | 1 | Medium | Merged into context axes as an ordered axis with a `flow` flag. |
| Resources spent along a path | Health, energy, soul, damage spent on edges and refilled at nodes, with dominance pruning | 05, 06, 07 | 3 (three groups) | High | **Ext**, after v1: expensive and path-dependent (section 3.2). |
| Repeatable versus one-time sources | Count an item separately by source kind | 03 | 1 | Medium | Park. |
| Conditional, order-sensitive item effects | Item effect depends on what is already held | 06 | 1 | Medium | **Reject**: breaks commutativity (06 section 13). The monotone subset (raise to max, multi-term grants) fits a `counter` extension. |

### 2.5 Logic overlays and add-on patches

| Pattern | What it is | Projects | Count | Evidence | Verdict and reason |
|---|---|---|---|---|---|
| Ordered logic patches over a base world | Edit a rule, edit a macro, substitute a term, optional references for absent add-ons | 06 | 1 | High | **Ext** as build-time overlay packages in the registry (section 3). This is how a community extends a world without forking it. |
| Option-gated layers | See 2.2 | 05 | 1 | High | Covered by templates. |
| Mod coexistence lists | Another mod ships a "do not touch" list | 14 | 1 | Medium | **Tool**, patch-side (CPK), not the world. |

### 2.6 Item pools and progressive items

| Pattern | What it is | Projects | Count | Evidence | Verdict and reason |
|---|---|---|---|---|---|
| Progressive chains | One item granting increasing levels | 01, 04, 06, 16 (only these notes describe it; others not checked) | 4 | High | **Core**, already (`progressive`, `level`). |
| Pool presets by scarcity | Plentiful to minimal pools from options | 01, 02, 09, 10 | 4 | High | **Core**, already: item `count` as an expression over options. |
| Trap disguise | A trap takes the look of an item drawn from a pool | 01, 02 | 2 (one author group) | Medium | **Core recipe**: a seeded `select` per trap written to slot data. No primitive. |
| Demote items that open nothing | After options resolve, useless progression becomes filler | 04 | 1 | Medium | **Tool**: a static analysis we run for every world (batch A section 3). |
| Location categories with toggles | Tags such as minigame, major, minor, each toggled for progression | 04 (22 toggles), 05, 10 (major and minor), 14 (race categories) | 4 | High | **Core** via tags, `item_rule` and location `kind` computed from options. |

### 2.7 Fill algorithms and stages

| Pattern | What it is | Projects | Count | Evidence | Verdict and reason |
|---|---|---|---|---|---|
| Assumed fill | Place each item where it is reachable assuming the rest is held | 01, 02, 03, 08 (check after random placement), 09, 12 | 6 | High | **Core**: the family behind CFill v1 (doc 01 D2.7). |
| Forward fill variants | Walk like a player, pick what opens the most, place in reachable slots | 04, 05, 06, 07, 08, 10, 11 | 7 | High | **Tool**: named fill variants a generator may offer at the Valid tier; CFill v1 stays the Reproducible reference. |
| Named fill stages | Shops, rewards, own-dungeon keys, songs, then progression, then filler | 01, 02, 03, 04, 06, 08, 09, 12 | 8 | High | **Core**: declared stages in CFill (section 3). |
| Bounded retries | Fixed retry caps per stage and per seed | 01, 02, 03, 07, 08, 10, 12, 13, 14, 15 (05 retries on failure; its note gives no cap) | 10 | High | **Core**, already (doc 01 D3.2 item 4): retries live in the stream, so failures reproduce. |
| New seed on failure | Draw a fresh seed after a failed attempt | 12 (only when the user gave none), 15 (always) | 2 | High | **Reject**: the result no longer matches the input the user typed (15 section 13). |
| Documented fill guarantees | Written proof of which progression set is chosen | 06 | 1 | High | **Tool**: the CFill spec states its guarantees the same way. |
| Commutativity as a checked invariant | Same items in different orders must give the same reachability | 06 | 1 | High | **Core** check, needed only for extensions; pure worlds are commutative by construction (monotone counters). |
| Local items in multiworld fill | A few items guaranteed to stay in their own world | 07 (5 per world, counted), 05 (first progression local), 01 (cross-world reachability) | 3 | Medium | **Core** CFill parameter `local_count`. |
| Explainable failures | Structured reasons per rejected candidate or failed attempt | 14 (explain mode), 06 (attempt monitor), 10 (readable messages), 09 (HTTP 409 with reason) | 4 | High | **Core**: failures carry a registered error key and a candidate list (doc 01 D1.7). |

### 2.8 Constrained and weighted placement

| Pattern | What it is | Projects | Count | Evidence | Verdict and reason |
|---|---|---|---|---|---|
| Own-group item rules | Dungeon items only in their own dungeon | 01, 02, 04, 08, 09, 12 | 6 | High | **Core** as `item_rule` with a `same_group` form (composable today as one rule per group; the form is sugar). |
| Allowed kinds per slot | A slot class lists what it may hold | 08, 10, 11, 15 | 4 | High | **Core**, already (`item_rule`). |
| Always-allow | An item may sit behind its own lock (a dungeon's map at its boss) | 09 | 1 | Medium | Park. |
| Quota per group | At most or at least N of a class per location group | 01 | 1 | Medium | Park. |
| Depth-biased placement | Weight locations by how late they open, by item rarity or class | 04, 05, 06, 08, 11, 14, 15 | 7 | High | **Core** as declared CFill bias parameters; a world-supplied integer weight table is **Ext** `x.weighted_place` (section 3). |
| Placement windows by depth | At most N copies before area X | 14, 15 | 2 | Medium | **Ext** once `depth` is normative. |

### 2.9 Content shuffles over typed tables

| Pattern | What it is | Projects | Count | Evidence | Verdict and reason |
|---|---|---|---|---|---|
| Content as typed tables, algorithms once | Species, enemies, shops, stats as tables; one shuffle per kind | 13, 14, 15, 04 (enemies), 08 (enemy table) | 5 | High | **Core**, already (`tables`, `permute`, `select`). |
| Redistribute within a budget | Keep a row total, floors and a ceiling, follow related rows | 13 (strong), 15 (partial) | 2 | Medium | **Ext**, parked for a third world. |
| Fit tags | Source tag at most target tag (enemy size against arena) | 14, 13 (strength window) | 2 | Medium | **Ext** option on `permute`. |
| Related-row following | Evolutions follow their base; helpers move with their owner | 13, 14 | 2 | Medium | **Ext** option on `permute` and `select`. |
| Widening similarity band | Nearest by integer key, widen until enough candidates | 13 | 1 | Medium | Park. |
| Scale by tier | Rescale moved content by a tier multiplier table | 14 | 1 | Medium (ideas only) | Park. |
| Silos | Shuffle only inside a partition (DLC, region, game) | 14, 03 (own-game pools), 06 (groups) | 3 | Medium | **Core**, already: `permute` within groups. |
| Ordered content stages | Later shuffles read earlier results | 13 (implicit), 04 (fixed commented order), 03 (named passes) | 3 | High | **Core**: order derived from data references between primitive nodes; cycles rejected. |
| Decide, then write | Randomness only in a decision phase; writers take no randomness | 04 | 1 | High | **Core** principle: the generator decides, the patcher or client writes (doc 01 D4.1). |

### 2.10 Entrance, door and lock randomization

| Pattern | What it is | Projects | Count | Evidence | Verdict and reason |
|---|---|---|---|---|---|
| Typed pools with reverse pairing | Entrances grouped by type, each linked to its reverse | 01, 02, 03, 04, 05, 06, 07, 08, 10, 12 | 10 | High | **Core** `rewire` parameters (section 6). |
| Decoupled and mixed modes | Break the pairing; merge types | 02, 03, 06, 08 | 4 | High | **Core** `rewire` parameters. |
| Invariants after each placement | Start area and refills reachable with no items; everything reachable before stays reachable | 01, 02, 03, 04 (safety entrance), 12 (all-locations check) | 5 | High | **Core** `rewire` `invariants`. |
| Restrictive-first order and pinned targets | Place the hardest entrances first; pin some targets | 01 | 1 | Medium | **Core** as parameters, since they change output; pinning as data replaces 01's hand conditionals (01 section 13). |
| Lock randomization | Shuffle the lock on a door, not its destination | 05 (dock weaknesses), 10 (door colors) | 2 | Medium | **Ext**, parked: `door_kind` metadata now, `relock` later. |
| Rule rebinding from a shuffle | A shuffle's outcome rewrites named rule fragments | 04, 01 (events) | 2 | Medium | Merged into `rewire` and `permute` as a declared `binds` output. |
| Generated maps | Wave-function-collapse overworlds, dungeons from room templates | 08 | 1 | Medium | **Reject** for the format; residue for the reserved module door (doc 07 section 6.3). |

### 2.11 Goals and objectives

| Pattern | What it is | Projects | Count | Evidence | Verdict and reason |
|---|---|---|---|---|---|
| Counted goals over chosen groups | "N of these token groups" for many goals at once | 01, 03, 08, 09 | 4 | High | **Core recipe** over `group_count` and `opt`. |
| Objective templates | Condition kinds, seeded count ranges, any N of M, a result key | 11, 10 | 2 | Medium | **Core recipe** with one language addition, `at_least` (section 3). |
| Selected bosses or targets | Required bosses chosen per seed | 04, 12 (goals as checks) | 2 | Medium | **Core**, already (`select` plus goal). |

### 2.12 Hints

| Pattern | What it is | Projects | Count | Evidence | Verdict and reason |
|---|---|---|---|---|---|
| Some hint system | Any in-game hint feature | 01, 02, 03, 04, 05, 07, 08, 09, 12, 14, 15 | 11 | High | **Core** platform service (section 4). |
| Weighted plans with copies | Per hint type: fixed count, weight, copies, filters | 01 (18 files), 02 (4 tables), 03, 04, 15 (7 systems) | 5 | High | **Ext** primitive `x.hint_plan` (section 3). |
| Required and barren by removal | Remove an item, re-solve, label it required or not | 01, 02, 03, 04, 09 (minimal set by removal) | 5 | High | **Core** analysis inside the shared solver. |
| Reachable-before sources | A hint is only placed where it is reachable without the item it names | 12, 02 (inferred) | 2 | Medium | **Core** default rule of the hint engine: cheap, and the safe choice. |
| Feature-based vague hints | Tags with a precision value, sampled for vagueness | 05, 07 (area hints) | 2 | Medium | **Ext**, parked. |

### 2.13 Seeds and settings

| Pattern | What it is | Projects | Count | Evidence | Verdict and reason |
|---|---|---|---|---|---|
| In-game seed code | Icons, names or a check value shown in game so players confirm the same seed | 01, 02, 04, 05, 09, 11, 12, 13, 15 (03 has an 8-character seed hash; its note does not say it is shown in game) | 9 | High | **Core** (section 3). |
| Word-based names | Seeds or settings named by words | 04, 05, 07 | 3 | High | **Core**, already (seed phrase, doc 01 D3.1). |
| Seed-affecting versus cosmetic settings | Flag on each option; cosmetics excluded from shared strings and hashing | 01, 04, 05, 08, 09, 10, 11, 13, 15 | 9 | High | **Core** option attribute `scope`. |
| Information settings change the seed | Hint and spoiler settings perturb the stream so racers cannot compare | 03, 04, 08, 15 | 4 | High | **Core** `scope: knowledge`. |
| Difference-from-defaults string | Only non-default values, versioned | 03, 08 | 2 | High | **Core**, already (doc 01 D3.1). |
| Positional bit-packed string | Bit positions in declaration order, version-bound | 01, 04, 05, 06, 13, 15 | 6 | High | **Reject**: every new option needs a migration (13 section 13). |
| Per-subsystem random streams | Items, enemies, cosmetics or each feature get their own stream | 04, 06, 13, 14 | 4 | High | **Core**, already (HKDF per hook, doc 01 D3.1). |
| Named generator with cross-OS test | A specified PRNG; seed files compared across operating systems in CI | 07 (03 has its own small generator, no cross-OS test) | 1 | Medium | **Core**, already (ChaCha8 and the Reproducible tier). |
| Secret build key or private race code | A secret mixed into race seeds, absent from public code | 04, 09, 10 | 3 | High | **Reject**: no third party can regenerate the seed. Commit-reveal covers fairness (doc 01 D3.3). |

### 2.14 Spoilers, proof, testing, tracking, delivery, plando, multiworld

| Pattern | What it is | Projects | Count | Evidence | Verdict and reason |
|---|---|---|---|---|---|
| Spoiler visibility as a seed property | On, off, creator sees once, metadata only | 09 (four modes); spoiler toggles in 04, 08, 11, 15 | 5 | High | **Core** `spoiler_policy` (section 3). |
| Sphere playthrough or solver output | Spheres or a path stored with the seed | 01, 02, 03, 04, 05 (resolver), 07, 09, 10, 12, 15 | 10 | High | **Core**: the validator emits a reference playthrough. |
| Difficulty estimation | A label and score from the solver | 10 | 1 | Medium | **Tool** after `x.cost`. |
| Data integrity checks | Unreachable clusters, recursive templates, undocumented tricks, cycles, slot-count asserts, feasibility before search | 05, 06, 04, 11, 12, 01 (preset checks) | 6 | High | **Core** static checks (doc 07 section 5.1, extended in section 6). |
| Golden seeds | Committed expected output per world and preset | 05 (expected hash), 06 (dated snapshots), 07 (cross-OS compare) | 3 | High | **Core** in the author harness and conformance (doc 01 D2.9). |
| Whole-seed smoke runs and fuzzing | Many seeds over setting combinations, failing settings saved | 01, 03 (25 seed tests), 04 (bulk), 08 (26 CI runs), 15 (not in CI), 16 (fuzz) | 6 | High | **Core** test tiers in the author harness. |
| Tracker logic tiers | Reachable, sequence break, inspectable | 16 (six levels), 06 (helper log marks out-of-logic) | 2 | Medium | **Core** for tracker display via tricks and tiers; "inspectable" parked. |
| Layouts as data | Widget tree, named roots, map positions | 16 | 1 | High | **Ext** format (CTL), with defaults generated from the manifest. |
| Coordinates on nodes | Positions for maps and editors | 05, 07, 16 | 3 | High | **Core** optional metadata `pos`. |
| Watch tables for auto-tracking | Memory or variable source mapped to item and location keys | 16, 01 (versioned memory layout), 07 (pickup as a game-state condition), 10 (autotracker data), 15 (tracker reads memory) | 5 | High | **Core** inside the existing connector profile `watch` (doc 01 D4.3). |
| Browser delivery | Base file cached in the browser, small per-seed delta applied in the page | 09, 10; generator in the browser in 08 (Pyodide) and 03 (inferred) | 4 | High | **Tool**, gated by D-58 (open). |
| Plando | Hand-pinned placements, filled and checked | 01, 02 (no logic recheck), 03, 07, 08, 09, 10, 12 (code only) | 8 | High | **Core** platform feature: a placement table checked by the shared solver. |
| Multiworld built in | Cross-world fill and a delivery protocol | 01, 03, 05, 07, 15 (swap chains), 02 (co-op), 09 (designed, not delivered), 14 (external placement input) | 8 | High | **Core** is CWP; world-side, only `local_count`. 03's write-ahead log with acks matches doc 01 D1.4. |

## 3. Admission results under doc 07's rule of three

### 3.1 How to read the table
Doc 07 section 2.4 admits a **primitive** (named behavior with randomness or search) only when all six criteria hold. Research can speak to criterion 1 (need, counted here as projects and author groups) and criterion 2 (a written composition attempt). Criteria 3 to 6 (vectors, two implementations, cost class, version) need work we have not done, so **no candidate is admitted today**. The best status is "evidence met, go to vectors". Many candidates are not primitives at all (an option attribute, a language combinator, a policy, a format); those follow the CRFC process (doc 01 D6.3) rather than the rule of three, and the table says so.

Status words: **Go**: evidence met and not composable; write vectors and two implementations as `x.name@1`. **Recipe**: composable today; publish a macro or recipe, no primitive. **Schema**: a field, option kind, combinator or policy; decide by CRFC. **Service**: platform behavior, not in the world file. **Park**: evidence short. **Reject**.

| Candidate | Projects (author groups) | Criterion 1 | Composition attempt | Status | Proposed shape in our format | Determinism and monotonicity | Merges and overlaps |
|---|---|---|---|---|---|---|---|
| `trick` | 01, 02, 03, 05, 06, 07, 08, 09, 10 (8 groups) | Met | Option toggles cover the boolean but lose docs, tags, levels and UI grouping | **Schema** (CGM v1 core) | Top-level `tricks: {key: {levels?: [..], tags: [..], area?, doc: string_key, video?: url}}`; rule test `{"trick": "k"}` or `{"trick": "k", "ge": "level"}`; settings layer holds the chosen set | Folds to a constant before logic like `opt`; monotone in items | Same object a skill profile parameterizes; per-trick levels absorb 05's model |
| `tier` | 01, 04, 07, 08, 09, 12 (6) | Met | Composable as `any(all(opt.tier >= t, body_t))` | **Recipe** plus **Schema** (an ordered option kind) | `tiers: {order: [..]}`; option type `tier`; authoring `{"tiers": {t1: rule, t2: rule}}` compiles to the OR of all tiers at or below the selected one | Monotone in items; superset across tiers holds by construction (OR of lower tiers) | Merge with "logic tiers for trackers" and 04's "Obscure N" macros; a tier is a coarse cost |
| rule `cost` plus `skill_profile` | 10 (1); coarse in 07, 12 | Not met as numeric cost | Tiers cover the ordered part; numbers do not compose | **Ext**, Park for v1 | `x.cost@1`: integer cost per trick level from the skill profile; difficulty of a rule = max over AND, min over OR (section 3.3) | Never changes the boolean result; integer only; order-independent with max | A skill profile is a map from `trick` to level or cost; one object, not two |
| `difficulty_window` | 10 (1) | Not met | Rejection sampling over a cost that does not exist yet | **Park** (generator option after `x.cost`) | `generate.difficulty: {min, max, attempts}` | Deterministic if attempts are drawn in-stream and capped | Depends on `x.cost` and the reference playthrough |
| `state_dimension` (context axes) | 01, 02, 03 (2 groups, 1 game family) | Met by number, thin by diversity | Composable as `templates` with `foreach` over a context table, duplicating regions per value with switch connections; this is what 01's authors avoid (01 section 14) | **Recipe** first; **Go** only if the spike shows the expansion too large | Recipe: `contexts: {age: [child, adult]}` expanded at build into regions `r@child`, `r@adult`; candidate `x.context@1`: reachability over (region, context) with `switch` connections and rule access `{"ctx": "age", "eq": "adult"}` | Finite product graph; monotone in items per product node | Absorbs `time_window` as an ordered axis with `flow` and `stay` flags; shares a search engine with `resource_state` |
| `time_window` | 03 (1) | Not met | Ordered context axis | **Merge** into context axes | Axis `{key: clock, ordered: true, values: 46 slices}` plus region `flow` and `stay` | Slice sets per region only widen | See above |
| `resource_state` | 05, 06, 07 (3) | Met | Expansion over bounded values is exponential in the number of resources; not a practical composition | **Go**, after CGM v1 | `x.resource@1`: `resources: {health: {max: rule, start: rule}}`; connection `spend: {health: 2}`; region `refill: [health]`; item effects only raise maxima | Reachability over (region, resource vector) with dominance pruning; monotone in items (more items never shrink the dominant set) but path-dependent; cost class high, bounded by declared maxima | Same engine as context axes; damage per traversal (05) is a spend |
| `fill_stages` | 01, 02, 03, 04, 06, 08, 09, 12 (7) | Met | `place` plus `item_rule` gives filters, but not the reachability check level per stage or stage-local retries | **Schema** inside CFill v1 (generator algorithm, not a world primitive) | `fill.stages: [{key, items: filter, locations: filter, check: full|rules_only|none, scope: own_world|any, retries}]`; unnamed items fall to the final progression and filler stages | Stage order is data; retries counted in the stream; same input gives the same failure | Absorbs `fill_group` (06), `junk_quota` (09, a stage of filler with a seeded count) and dungeon-keys-first (08) |
| `hint_plan` | 01, 02, 03, 04, 15 (4); systems in 11 projects | Met | Not composable: allocation over solver facts (required, barren, path) is search | **Go** | `x.hint_plan@1`: `hints: {sources: location filter, kinds: [{kind: path|barren|item|location|always, count?, weight?, copies?, filter?}], limits}`; platform-defined kinds; text by string keys | Facts come from the reference solver; allocation uses its own stream and integer weights; reachable-before is the default constraint | Takes `hint_source_reachable_before` (12) as a rule; `hint_features` is a later kind |
| `rebind` | 04, 01 (2) | Not met | Not needed as its own primitive | **Schema**: a declared output of `rewire` and `permute` | `binds: {def_key: output_column}`; after load, the def resolves to the chosen value's rule | A load-time fold; the tree stays static and analyzable | Part of `rewire` |
| `overlay` | 06, 05 (layers) (2) | Not met | Build-time patch, not runtime behavior | **Schema** (registry package kind), build-time only | `overlay.cgm`: `base: {id, hash}`, ordered `edits: [{target: def|rule|pool, op: replace|extend|substitute}]`, `optional_ref` for absent dependencies; own ledger range | `cgm build` flattens base plus overlays to one canonical document and hash; no overlay exists at load time | Uses doc 07 section 5.2 to classify each overlay as loosening or tightening |
| `objectives` | 11, 10 (2) | Not met | Composable: `select` over an objective table with rolled counts, goal as `at_least(N, conditions)` | **Recipe** plus **Schema** `at_least` | Combinator `{"at_least": n, "of": [rule...]}`; result keys in `slot_data` for the game to apply | Monotone (a count of true monotone rules) | Counted goals (01, 03, 08, 09) are the same recipe with `group_count` |
| `spoiler_policy` | 09 (four modes); toggles in 04, 08, 11, 15 | Met as a policy | Not a world concern | **Schema** in doc 01 D3.3 | Room or seed property `spoilers: none|creator_once|after_completion|public|metadata_only` | `creator_once` and `none` salt nothing; spoiler presence is a knowledge-scope input (next rows) | Storage is doc 04 section 7.4 |
| `seed_code` | 01, 02, 04, 05, 09, 11, 12, 13, 15 (8) | Met | Derivable from the bundle hash | **Service** plus a small **Schema** field | World declares `seed_code: {symbols: table_key, length: 5}`; code = first `length` symbols indexed from the bundle hash bits (defined extraction, vectors) | Pure function of the bundle hash | Replaces 13's check value and 15's icons with one derivation |
| `scope` (affects seed, knowledge, cosmetic) | 01, 04, 05, 08, 09, 10, 11, 13, 15 (9); knowledge salting in 03, 04, 08, 15 | Met | Option attribute | **Schema** in doc 01 D2.3 | `scope: seed` (default; in settings string and stream derivation), `knowledge` (hints, spoiler; in both, and shown to racers), `cosmetic` (never hashed, applied by the client) | Cosmetic options cannot reach `requires.primitives` inputs; the validator rejects a cosmetic option referenced by a rule | One attribute replaces `shared`, `permalink`, `aesthetic` and `cosmetic` flags |
| `redistribute` | 13, 15 partial (2) | Not met | Not composable (integer budget with floors and rounding) | **Park**; a third candidate may be pokemon_emerald's species randomization (doc 07 section 1.1; whether it redistributes stats is not verified) | `x.redistribute@1`: table, columns, total (keep or range), floor, ceiling, integer weights, largest-remainder rounding, `follow: relation` | Integer only, specified draws | `range_roll` (15 weapon stats) is a degenerate case |
| `weighted_place` | 15 (1 as a world primitive); bias weights in 04, 05, 06, 08, 11, 14 | Met as a fill parameter, not as a world primitive | Bias is generator behavior | **Schema** as CFill bias parameters; world table **Park** | `fill.bias: {by: depth|age, curve: table_key}` with integer weights per (item class, depth band) | Integer weights, cumulative choice (doc 01 D3.2) | Absorbs `fair_forward_weighting` (11), depth transform (06), forward factor (08) |
| `scale_by_tier` | 14 (1, ideas only) | Not met | Fixed-point table lookup after `permute` | **Park** | Not proposed | Would need fixed-point ratios | Uses `depth` |
| `depth` | 14, 15 declared; 04, 05, 06 internal | Met as a concept | Derived from the reference playthrough | **Schema** (normative definition in doc 01 D2.7) | `sphere(loc)`: index of the first sphere where a location is reachable in the reference playthrough; optional declared `depth` integer on regions | Deterministic from the playthrough; usable in `fill.bias` and hints, never in rules (a rule reading depth would be circular) | Underlies placement windows, bias and difficulty reports |
| `door_kind` plus `relock` | 05, 10 (2); partial 06, 08 | Not met for `relock` | `door_kind` is metadata; a lock is a connection rule today | **Schema** for `door_kind`; **Park** `relock` | Connection `kind: key` and optional `lock: {front: rule, back: rule}` | Rules stay monotone | `relock` would be a `permute` over lock columns per door kind |
| Logic tiers for trackers | 16, 06 (2) | Met through tricks and tiers | Tracker evaluates the rule at the room's settings, then with the next tier or all tricks on | **Service** | No new field; "inspectable" (16 only) parked | Same evaluator; no effect on generation | Merged into `trick` and `tier` |
| Tracker layout schema (CTL) | 16 (1) | Not applicable (a format) | Defaults derivable from items, groups, regions and `pos` | **Ext** format, owner decision (section 6) | `tracker.ctl.json`: maps, marker style, a closed widget set (grid, array, tabs, group, item, map, text, include), named roots (wide, tall, broadcast, settings) | Display only | Uses `pos`; designed in our own words, not PopTracker's schema |
| `pos` | 05, 07, 16 (3) | Met | Metadata | **Schema** | Optional `pos: {map: key, x: int, y: int}` on regions and locations | No logic effect | Feeds the editor and CTL |
| Watch table | 16, 01, 07, 10, 15 (5) | Met | Already the `watch` block of the connector profile | **Schema** extension of doc 01 D4.3 | Add `source: memory|variable`, `mask`, `compare`, `sets: {location|item|event: key}`, `interval_ms` | Client-side only | No second format |
| Reference playthrough | 01, 02, 03, 04, 05, 07, 09, 10, 12, 15 (9) | Met | Solver output | **Service** plus conformance (doc 01 D2.9) | Bundle section `playthrough: [{sphere, collected: [loc keys], required: bool}]` plus `minimal_required` by removal | Canonical order: by sphere, then ledger order | Feeds hints, depth, difficulty, spoiler |
| Golden seed | 05, 06, 07 (3) | Met | Test artifact | **Schema** for `tests/` | `tests/golden/*.json`: settings string, seed phrase, generator id and version, expected bundle hash | Requires a Reproducible-tier generator | Cross-OS and cross-implementation runs in conformance |
| `item_rule` `same_group` | 01, 02, 04, 08, 09, 12 (5) | Met | Composable as one rule per group with `foreach` | **Recipe** plus sugar | `item_rule: {items: {tag: dungeon}, locations: {same_tag: dungeon}}` | Unchanged | None |
| `always_allow` | 09 (1) | Not met | Needs item-dependent location rules, which the language lacks | **Park** | Not proposed | n/a | Try an event plus connection first |
| `disguise` | 01, 02 (1) | Not met | `select` per trap into slot data | **Recipe** | Recipe only | Seeded `select` stream | None |
| `exit_rule` (leave after taking) | 10 (1) | Not met | Event plus a connection out | **Recipe** (try first) | Recipe only | Monotone | None |
| `permute` `fits`, `related`; `select` `band` | 13, 14 (2) | Not met | Not composable | **Park** as `permute@2` and `select@2` options | Fields `fits: {source_col, target_col}`, `related: relation`, `band: {key, start_pct, widen_pct, min}` | Integer comparisons | Same "content" family as `redistribute` |
| `local_count` and fill knobs | 01, 05, 07 (3) | Met as parameters | Parameters of CFill | **Schema** in CFill | `fill.local_count`, `fill.bias` | Specified draws | With `fill_stages` |
| `hint_features` | 05, 07 (2) | Not met | A hint kind | **Park** | Future `x.hint_plan@2` kind | n/a | Part of `hint_plan` |

### 3.2 Merges, in plain words
1. **One difficulty model, not five.** Tricks, per-trick levels, world tiers, tracker logic tiers and skill profiles are one idea at different resolutions: a named technique with an ordered level, and a player's chosen level per technique. CGM v1 carries tricks and tiers; the numeric layer (`x.cost`, skill profiles, difficulty windows) is one extension on top.
2. **One state engine, two axis kinds.** Context axes (age, form, time slices) and resources (health, energy) both make reachability depend on (region, state) rather than region alone. They share a search engine but differ in cost: context axes are a small finite product; resources need dominance pruning. Admit them separately, context first (as a recipe), resources after v1.
3. **Fill stages swallow groups, quotas and junk placement.** A stage with an item filter, a location filter, a check level and a retry cap covers 06's groups, 08's keys-first pass and 09's tower junk.
4. **Placement bias is a generator parameter.** Seven projects bias placement by depth or age (section 2.8); only 15 makes it world data. The world may supply an integer curve; the algorithm stays in CFill.
5. **Rebinding is an output, overlays are a build step.** Neither runs at generation time, so neither is a primitive.
6. **Watch tables already have a home** in the connector profile; we extend it rather than add a format.

### 3.3 Tiers and costs against doc 01's monotone rules
Doc 01 D2.7 defines reachability as a unique fixed point because rules are monotone in items. Both proposals keep that.
- **Tier** compiles to `any(all(tier >= casual, A), all(tier >= normal, B), ...)`. The tier is an option, folded before logic, so at load time each rule is a plain monotone tree. Raising the tier only adds disjuncts, so the reachable set at a higher tier contains the set at a lower tier for the same items (superset by construction). The validator can prove this cheaply: after folding, each higher tier's tree is the lower tree OR something.
- **Cost** does not change any boolean. It is a second, integer evaluation over the same tree: a leaf's cost comes from the skill profile (a disabled trick folds to false, as today), AND takes the **maximum** of its children, OR the **minimum** of passing children. We propose max rather than VARIA's sum (10 section 4) because max is idempotent and order-independent: a shared sub-requirement counted twice gives the same answer, and the "difficulty to reach" fixed point is a bottleneck path over a finite integer lattice, so it is unique. Sum is the classic shortest-path semiring and also converges for nonnegative integers, but it double-counts shared leaves in DNF and VARIA's note records order-dependence (10 section 13). A sum mode can be a later `x.cost@2`. Cost class: one extra integer pass per evaluation (estimated small; not measured).
- **Difficulty window** wraps generation in rejection sampling with an in-stream attempt counter; output stays a pure function of inputs, but generation cost scales with the expected attempts (doc 13 owns the cost model; not estimated here).

## 4. The centralized author experience

### 4.1 What the author writes and what the platform provides

| The author writes (game-specific) | The platform provides (shared, once) |
|---|---|
| Tables (CSV or YAML): items, locations, regions, enemies, content | Strict parser, canonicalizer, ledger, hashing (`cgm build`) |
| Rules in the expression language, macros, events | Rule evaluator in Rust and an independent one in TypeScript |
| Options with string-key labels, presets | Generated settings form (web and, through CSDK, in game) |
| Tricks with docs and levels, tiers | Trick UI, documentation checks, skill-profile extension |
| Entrance pools and `rewire` parameters, fill stages | `rewire`, `select`, `permute`, `place`, CFill v1 with stages and retries |
| Hint sources and an optional plan | Hint engine: required, barren, path, allocation, reachable-before |
| `slot_data` fields, string bundles, optional `pos` and CTL | Seed phrase, settings string, streams, seed code, spoiler policy and storage |
| Author tests: rule vectors, golden seeds | Static checks, sampled seeds, fuzzing, golden comparison, cross-OS run |
| (Third party, optional) a patcher or game client | Reference playthrough, plando checker, generated tracker, registry |

### 4.2 End to end, for a new game
1. **Scaffold.** `cgm new` (or the web studio) writes a skeleton: manifest, empty ledger, `strings/en.yaml`, `tests/`. 05 shows the value of a scaffold that CI proves still works (05 section 11).
2. **Tables and logic.** The author fills tables, writes rules as strings or structured trees, and factors helpers into macros. The language server shows the expanded rule on hover and the same error codes as the build.
3. **Options, tricks and tiers.** Options carry labels as string keys and a `scope`. Tricks carry docs; tiers are an ordered option.
4. **Test harness.** `cgm check` runs the static checks; `cgm test` runs rule vectors, N sampled seeds per preset (design value), the golden seeds and a settings fuzzer that saves failing settings strings (01's practice, 01 section 11).
5. **Web studio.** The visual region-graph editor reads canonical JSON and writes authoring YAML with comments kept; a text pane shows the same edit as a diff against the canonical print, so reviews happen on text (05 section 13). Preview runs the Rust generator compiled to WASM in the author's own browser (the generation fleet rule, D-15, is about our servers, not the author's tab), and the TypeScript evaluator drives a live tracker preview, so every preview is also a cross-implementation check.
6. **Publish.** The registry signs and stores the canonical document by hash, classifies the change against the last version (doc 07 section 5.2), and records the required primitives (doc 07 section 2.5). The world page gets the settings form, trick list, a tracker and docs generated from the data.
7. **In a room.** The lobby's Check A compares world version and data checksum before generation (doc 09 section 5.1); the fleet generates; the validator emits the reference playthrough; hints and spoilers follow the room's policy.

A skeleton for a fictional game (our own example; syntax is a proposal; fields marked "proposed" are additions from section 6):

```yaml
format: cgm-data/1
language: 2                                  # proposed: adds trick tests and at_least
requires: {primitives: [rewire@1, x.hint_plan@1]}
meta: {id: dev.example.lanternfall, name: Lanternfall, version: 0.1.0, license: CC0-1.0, authors: [Example Author]}

tricks:                                      # proposed section
  ledge_clip: {levels: [beginner, advanced], tags: [movement], doc: tricks.ledge_clip.doc}
  dark_room:  {tags: [knowledge], doc: tricks.dark_room.doc}
tiers: {order: [casual, normal, hard]}       # proposed section

options:
  logic_tier:   {type: tier, default: normal}                        # proposed type
  shuffle_mine: {type: toggle, default: false, scope: seed}          # proposed attribute
  hint_count:   {type: range, min: 0, max: 12, default: 6, scope: knowledge}
  lamp_color:   {type: choice, default: amber, values: [amber, teal], scope: cosmetic}

macros:
  can_light: "has(lantern) or trick(dark_room)"
  cliff_up:
    tiers:                                   # each tier adds a way; compiles to any() over tiers <= chosen
      casual: "has(rope) and has(boots)"
      normal: "has(rope)"
      hard:   "trick(ledge_clip) >= beginner"

regions:
  village: {start: true, exits: {cliffs: "cliff_up()", mine: "can_light()"}, pos: {map: world, x: 120, y: 80}}
  cliffs:  {pos: {map: world, x: 200, y: 40}}
  mine:    {exits: {vault: "has(pickaxe)"}}
  vault:   {}

locations:
  well_bucket: {region: village, tags: [minor]}
  cliff_nest:  {region: cliffs, tags: [major], pos: {map: world, x: 210, y: 35}}
  mine_cart:   {region: mine, kind: priority}
  vault_altar: {region: vault, rule: "at_least(2, [has(gem_red), has(gem_blue), has(gem_green)])"}

fill:                                        # proposed: declared CFill stages
  stages:
    - {key: mine_keys, items: {tag: mine_key}, locations: {region_in: [mine, vault]}, check: full, retries: 10}
hints:                                       # proposed: x.hint_plan@1
  sources: {tag: signpost}
  kinds: [{kind: path, count: 2}, {kind: barren, weight: 2}, {kind: item, weight: 3}]
seed_code: {symbols: glyphs, length: 5}      # proposed
goal: {rule: "reach(vault) and has(crown)"}
```

`tests/golden/default.json` would hold a settings string, a seed phrase, the generator id and version, and the expected bundle hash; `tests/rules.json` holds `(state, options) -> expected` vectors for the tricky rules.

### 4.3 Author pipeline

```mermaid
flowchart LR
  A[Author files: YAML, CSV, strings] --> B[cgm build: strict parse, macros, tiers, canonical JSON, ledger]
  E[Web studio: graph editor plus text diff] --> A
  B --> C[cgm check: static and integrity checks]
  C --> D[cgm test: rule vectors, sampled seeds, fuzz, golden seeds, cross-OS]
  D --> P[Browser preview: Rust generator as WASM, TS evaluator, generated tracker]
  P --> R[Publish: signed, content-addressed registry entry, change class]
  R --> W[World page: settings form, tricks, tracker, docs]
  W --> G[Room generation: Rust generator in the fleet]
  G --> V[Independent validator in TS: reference playthrough]
  V --> H[Hint engine and spoiler service]
  V --> T[Tracker and plando use the same solver spec]
```

### 4.4 What the platform runs centrally
1. **Core library `cgm-core` (Rust):** parser for the authoring profile, canonicalizer, ledger checks, rule evaluator, primitives, CFill v1, solver. Built natively for the generation fleet and as WASM for previews and the web tracker.
2. **TypeScript conformance evaluator and validator:** written independently from the same spec (doc 07 section 2.4 item 4).
3. **One solver specification, two implementations.** Doc 01 D2.9 wants an independent validator, while authors want one solver behind generator, tracker and plando. Both hold if the solver is a specification with vectors and two implementations: Rust behind generation and plando, TypeScript behind the validator, the tracker and the browser preview. Every room then cross-checks the two (VARIA's one-engine benefit, 10 section 13, without a single point of error).
4. **Hint engine:** required, barren and path facts from the reference playthrough; allocation by `x.hint_plan`; reachable-before by default.
5. **Seed and spoiler services:** phrase, settings string with `scope`, streams, seed code from the bundle hash, commit-reveal, spoiler policy with per-room encrypted storage (doc 04 section 7.4).
6. **Tracker:** generated from the manifest (item grid per group, list per region, states from the evaluator, tiers as colors), optional CTL for maps, auto-tracking from the room event log (doc 01 D1.4) or a connector profile `watch` for games with no client (note 16 section 14 sketches this).
7. **Plando:** a placement table checked by the solver before generation completes (02's in-game editor does not recheck logic, 02 section 5, which is the case to avoid).
8. **Registry:** signatures, change classification, overlay flattening, required-primitive badges.

Cost note: the studio preview runs on the author's machine, so it adds no fleet CPU; hosted generation cost is governed by doc 13. The two-implementation rule roughly doubles the engineering for each primitive (doc 07 section 5.3); that is the price of "same answer everywhere".

## 5. Importers and bridges

### 5.1 Logic that is already data
Licenses are as recorded in the notes; none is legal advice. An importer is a tool that reads a project's files and emits a draft CGM world. Running it ourselves and committing or hosting its output would copy their data, which our rules forbid without an explicit decision and a license check (`CLAUDE.md`).

| Source | Form | Closeness to CGM | License (notes) | Path |
|---|---|---|---|---|
| OoTMM `data/world`, `data/macros` (03) | YAML areas, own expression grammar, parse-time folding of settings and tricks | High for grammar and folding; MM time slices need context axes; location binding lives in separate XML | MIT | **Partner** first (decided, D-101): the closest grammar to ours; offer an exporter in their repo |
| OoTR `data/World/*.json`, `LogicHelpers.json` (01) | JSON with comments; Python-syntax rule strings | Medium: regions, exits, events map; strings need a parser for a Python expression subset; age and time need context axes; location metadata is a Python table | MIT top level with GPLv2+ subfolders; which folder covers the data is not verified | **Partner** or **contribute** an exporter; counsel before any output is hosted |
| wwrando `logic/*.txt` (04) | YAML-like locations, 307 text macros | Medium: no region graph; entrance access is macro rewriting (our `binds`) | MIT | **Contribute** an exporter, maintainers decide |
| Randovania databases (05) | JSON requirement trees, typed nodes and docks, 13 games | High structurally | GPL-3.0; the note treats the database data as GPL-covered | **Partner only**: Randovania could emit CGM under its own terms; we neither convert nor host without counsel |
| RandomizerMod JSON (06) | Infix strings over terms, waypoints, transitions; state logic in C# | Medium to high for boolean logic; state logic not portable | LGPL-2.1 | **Partner**, counsel |
| wotw-seedgen `paths.wotwl` (07) | Custom language with tiers | Medium | No license file; `Cargo.toml` declares MIT | **Ask the maintainers**; no import until the license is clear |
| PopTracker packs (16) | JSON plus rule strings plus Lua | Rules low to medium (Lua escapes); layouts high for CTL | PopTracker GPL-3.0 with a plug-in addendum; each pack has its own license (not verified) | **Bridge** per pack license, counsel first; CTL is designed in our own words |
| SoulsRandomizers annotations (14) | YAML-format area and slot data | Medium | Custom restrictive license | **Never** import, wrap, host or invoke |

### 5.2 Code-only logic: compatibility edge or a fresh world
SoH (02, no top-level license), LADXR (08, MIT), ALttPR (09, MIT), VARIA (10, MIT), Worlds Collide (11, MIT), z17 (12, GPL-2.0), UPR ZX (13, GPL-3.0) and KH2 (15, Apache-2.0) keep logic in code. Where an Archipelago world exists, players reach them through the compatibility edge and sidecar unchanged (doc 01 section 3.8): the notes record in-tree worlds `oot`, `tww`, `hk`, `ladx` (vendored LADXR), `alttp` (a separate lineage), `sm` (vendored VARIA), `kh2` and `dark_souls_3`, plus external forks for SoH and (not opened) Worlds Collide (00 table, notes 02, 04, 08, 10, 11). A native world for these games means restating the logic in our language, which only makes sense with the original maintainers or with authors who write it fresh from game knowledge, never by translating restricted code (z17's GPL-2.0 note says this directly, 12 section 1).

### 5.3 Adoption plan that respects licenses and communities
1. **Ask first, in private.** Approach maintainers before any public mention, as D-38 does for Archipelago; name no individuals in anything critical.
2. **Contribute.** Offer an exporter that lives in their repo, under their license, run by them, emitting CGM. They keep their format; CGM is one more output.
3. **Partner.** Co-list the world on our registry with their team as owner, credit on the world page, shared conformance results, and their own release schedule (registry dependencies by hash, doc 01 section 3.7).
4. **Bridge.** Where they prefer not to change anything, run their Archipelago world through the sidecar, labeled honestly (Valid tier at most, doc 01 section 3.8.2).
5. **Never** copy or translate code under GPL, LGPL, custom or no license into our Rust or TypeScript; never host, wrap or invoke SoulsRandomizers; never bundle a pack or database whose license we have not cleared.
6. **Defects upstream first.** The notes mark several behaviors as inferred (random values drawn outside the seed stream in 03, option labels hashed into seeds in 02, a CI step that may not fail in 07). They stay internal and suspected until reproduced, then go to the maintainers before anywhere else (`CLAUDE.md` MUST 5, NEVER 2).

## 6. Proposed changes to other docs (not made here)

### 6.1 Doc 07 (primitives and checks)
- Section 2.3, add candidate rows: `x.hint_plan@1` (Go); `x.context@1` (recipe first, primitive if the spike needs it); `x.resource@1` (Go, after v1); `x.cost@1` (Park); `x.redistribute@1` (Park); `permute@2` options `fits` and `related`, `select@2` option `band` (Park).
- Section 2.3, extend `rewire`: `pools` by type, `reverse` pairing, `mode: coupled|decoupled|mixed`, `one_to_one`, `inside_outside`, `direction_match`, `loop_size` and `groups`, `nesting`, `order: restrictive_first`, `pinned`, `invariants: [rule]` evaluated with an empty inventory after each placement, and `binds` output.
- Section 2.3, extend `item_rule` with `same_tag` and `permute` with `group_by` named as a silo.
- Section 2.4, add one sentence: research evidence from outside projects counts toward criterion 1 only; the world-format criteria 2 to 6 still apply.
- Section 5.1, add checks: strongly connected components unreachable from the start unless marked (05); trick documentation required above a declared level (05); tier superset proof after folding; progression items that open nothing reported for demotion (04); infeasible settings detected before search (04); cosmetic-scope options must not appear in any rule; overlay flattening produces no dangling reference; hint sources sufficient for the plan's fixed counts.
- Section 6, add residue rows: tile-level map generation (08) and order-sensitive item effects (06).

### 6.2 Doc 01 (rule language, manifest, seeds, conformance)
- **D2.2** add sections `tricks`, `tiers`, `fill` (stages, `local_count`, `bias`), `hints`, `seed_code`; optional `pos` on regions and locations; optional `area` display grouping; connection `kind` and `lock`.
- **D2.3** add option type `tier` (ordered choice from `tiers.order`), attribute `scope: seed|knowledge|cosmetic` (default `seed`), attribute `ordered: true` on `choice` to allow `ge`, and flag `experimental` for options that may yield unbeatable seeds (04's `unbeatable` flag).
- **D2.5** add combinators `{"trick": k}` and `{"trick": k, "ge": level}` (folded like `opt`) and `{"at_least": n, "of": [r...]}`; authoring sugar `tiers:` (doc 07 section 4.4). Language version 2.
- **D2.7** define `sphere` and `depth` normatively from the reference playthrough; state that rules may not read depth.
- **D2.9** the validator emits the reference playthrough and `minimal_required`; each published world ships at least one golden seed; conformance runs golden seeds on two operating systems and both implementations.
- **D3.1** cosmetic-scope options are excluded from the settings string's hash input and from stream derivation; knowledge-scope options enter both; failed attempts retry inside the stream and never draw a new seed.
- **D3.2** item 6: the seed code is a defined extraction from the bundle hash, with vectors.
- **D3.3** replace `spoilers: "none"` with the enum `none|creator_once|after_completion|public|metadata_only`; state that private race code and secret build keys are not used.
- **D4.3** extend `watch` with `source`, `mask`, `compare`, `sets` and `interval_ms`.
- New optional extension spec **CTL** (tracker layout), if the owner agrees (D-111).

### 6.3 New rows for DECISIONS.md (D-102 to D-114, all proposed unless marked open or needs-lawyer)

| Id | Decision | Status | Rationale | Docs affected | Blocks |
|---|---|---|---|---|---|
| D-102 | CGM v1 carries a difficulty model: `tricks` and `tiers`, folding to constants; numeric costs and skill profiles are the `x.cost` extension | proposed | 9 and 6 of 16 projects (counted); keeps rules monotone | 01, 07 | CGM v1 freeze |
| D-103 | CFill v1 takes declared fill stages, `local_count` and integer bias; retries stay in the stream; no new seed on failure | proposed | 8 projects stage fills (counted); reseeding breaks input-equals-output | 01, 07 | CFill vectors |
| D-104 | The hint engine is a platform service; plans are `x.hint_plan@1`, reachable-before by default | proposed | 11 projects have hints, 5 weighted plans (counted) | 01, 07 | Hint vectors |
| D-105 | One solver specification with two implementations: Rust for generation and plando, TypeScript for validator, tracker and preview | proposed | Reconciles "independent validator" with "one engine everywhere" | 01, 05, 07 | Validator build |
| D-106 | Option attribute `scope` (seed, knowledge, cosmetic) | proposed | 9 projects separate cosmetics, 4 salt the seed with information settings (counted) | 01 | Settings string |
| D-107 | Spoiler policy enum with `creator_once`; no private race code or secret build keys | proposed | Commit-reveal gives verifiable fairness (doc 01 D3.3) | 01, 04 | Race mode |
| D-108 | In-game seed code derived from the bundle hash | proposed | 9 projects (counted; 03 has a seed hash whose in-game display its note does not state) | 01, CSDK | CSDK |
| D-109 | State axes: context axes as a recipe now with a spike on a non-Zelda world; `x.resource` after v1 | **open** (owner) | Evidence is one game family for context axes; resources are costly | 07 | CGM v2 scope |
| D-110 | Overlays are build-time registry packages flattened into one canonical document | proposed | Community extension without forks (06) | 01, 07, registry | Registry schema |
| D-111 | A generated tracker is first-party; the CTL layout format is an optional extension we maintain | **open** (owner) | One strong source (16); a format we would own forever | 01, 03 | Tracker build |
| D-112 | Importer and partnership policy: exporters live upstream under their licenses; no import of GPL, LGPL, unlicensed or restrictive sources without counsel; never SoulsRandomizers | **needs-lawyer** | License rules in the notes | 03, 04 | Any import |
| D-113 | Content primitives (`redistribute`, `fits`, `related`, `band`, `scale_by_tier`) parked until a third world; run a content-world spike | proposed | Two projects each, one ideas-only | 07 | Content worlds |
| D-114 | Language 2 adds `trick` tests and `at_least` | proposed | Objectives (11, 10) and tricks need them; both monotone | 01, 07 | Rule vectors |

### 6.4 Roadmap items (doc 03 section 9 and doc 05 section 20)
- **Phase 1:** rule vectors include `trick`, `tier` and `at_least`; CFill v1 vectors include stages and in-stream retries; seed code and scope in the settings string spec.
- **Phase 2 (registry, test harness, creator studio):** `cgm new`, `check`, `test` with golden seeds and fuzzing; web studio with graph editor and text diff; generated tracker; hint engine with `x.hint_plan@1` vectors in Rust and TypeScript; overlay flattening; first partner conversation for an upstream exporter (03 first candidate).
- **Spikes:** context axes on a non-Zelda world; one content world (stats or enemies) to test the parked content family; `x.resource` cost measurement on one world before any admission.

### 6.5 Needs a lawyer
- No license: SoH outside the `3drando` notice (02), wotw-seedgen (07, MIT only in `Cargo.toml`), and the unlicensed rows in note 00 section 3.3.
- Restrictive custom license: SoulsRandomizers (14), including whether describing its ideas in public text is safe.
- GPL-family data and output: whether a converted logic database from GPL-3.0 (05, 13 offsets, 16 and packs), GPLv2+ (01 subfolders), GPL-2.0 (12) or LGPL-2.1 (06) sources is a derivative work, and whether location and item names from games carry rights of their own.
- AGPL projects (00 lists sshd-rando and cheese-trackers) if we ever host anything derived; Apache-2.0 NOTICE handling for KH2 (15) if anything is reused.

## 7. Risks and open questions

### 7.1 Where the evidence is thin
- **Context axes come from one game family.** All three projects are Ocarina of Time and Majora's Mask (01, 02, 03). Other games have forms or day and night cycles, but no audited project outside that family showed the pattern.
- **Numeric difficulty is one project.** Cost scoring, skill profiles and difficulty windows are all VARIA (10).
- **Lock randomization is two projects** (05, 10); the ALttP door randomizer (00 row 15) was not audited.
- **Layouts as data are one project** (16), though it is the community's leading tracker.
- **Content shuffles lean on 13 and 15**, and 14 can only be described at a high level.
- **Unread code:** OoTR assembly, SoH's random generator and co-op server, ItemChanger and the HK multiworld mod, the Ori client and server, Randovania's GUI and patchers, ALttPR's vendored entrance randomizer, VARIA's private race module, KH2's external enemy library, PopTracker's pack checker (per each note's last section). Claims about those are absent here, not negative.
- **Sample bias.** The 16 were chosen for diversity (00 section 4), not at random, and stars measure visibility, not players (00 section 1). Counts "of 16" describe this sample only.
- **Research evidence is not world evidence.** Doc 07's criterion 1 asks for three worlds needing a primitive in our format. A pattern in three external projects predicts that need; the spike (doc 07 section 8.2) still has to show it.

### 7.2 Design risks
| Risk | Why it matters | Mitigation |
|---|---|---|
| The standard grows into a language by accretion | Every candidate here is reasonable alone (doc 07 section 2.6) | Recipes before primitives; Experimental status; two implementations; this doc admits nothing |
| Hollow Knight-style state logic does not fit | 22 state modifier classes in C# (counted, 06 section 4) | `x.resource` later; the sidecar meanwhile; honest "data plus residue" labels |
| Authors used to code find the language small | Friction loses authors | Excellent errors, the studio, recipes per pattern, the sidecar path |
| Partners see us as taking their work | Goodwill is where the games come from (`CLAUDE.md`) | Exporters upstream under their licenses, their team as owner, credit, no copying |
| Single-maintainer projects | 04, 06, 07, 08 depend on very few people (batch B section 3) | Partnerships do not assume they maintain our side |
| Generation cost of new features | Difficulty windows and resource state multiply work | Cost classes per primitive; doc 13 models fleet cost before enabling |
| Two implementations drift | Tracker and generator disagree | Shared vectors; every room's validator run is a live cross-check |

### 7.3 Open questions for the owner
1. Should the first-party product ship a generated tracker and maintain a layout format (D-111), or leave tracking to third parties reading the room log?
2. Context axes: accept a recipe-only v1 and a spike, or admit `x.context` early to court the Zelda N64 community (D-109)?
3. Partnership order: decided by the owner (D-101): OoTMM first; Randovania follows after a lawyer reviews GPL implications.
4. Is `max` the right default for `x.cost`, or should we match VARIA's sum to ease a future VARIA partnership?
5. Do we invest in the web studio's graph editor before Phase 2, given doc 07 section 4.6 calls its benefit an untested hypothesis?
6. How strict is the trick documentation check: warning or publish blocker above a level?
