# 07. A data-only world format: how far can it go?
> Internal design draft, exploratory; not reviewed by a different agent or a human. Per `CLAUDE.md`, nothing here goes outside this repo until that review. Numbers are labeled **counted** (a line or file count from the research report), **estimated** (judgment), **design hypothesis** (a performance or benefit claim nobody has measured) or **not verified**. This repo contains our own writing, with citations, and no Archipelago source code, text or assets. Terms: `GLOSSARY.md`.

Builds on `01-standards.md` (D2.1 to D2.8: canonical JSON manifest, closed rule combinators, ID ledger, reserved world module). This doc asks whether the world-module (WASM) door in D2.8 can stay closed. Pure data is the working starting direction for native version 1, not locked (D-10, proposed); the Python sidecar handles everything else. Under D-01 (web-first, standard-only), patch packages (CPK) and connector profiles (CCP) are optional third-party extensions, not first-party flows.

## 0. Sources and their limits
- **World evidence** comes from a research report on the Archipelago repository at commit `4f9545f7` (internal file `worlds.md`, written by an earlier agent; I did not re-read that repository). File and line cites below are the report's cites into their repo. Their code changes; recheck before anything is published. Share-of-code percentages are the report's **estimates**; only raw line counts and grep counts are **counted**.
- **Language-design lessons** come from pages opened on 2026-11-16 whose bodies were read: Dhall safety guarantees (`docs.dhall-lang.org/discussions/Safety-guarantees.html`), the Starlark README (`github.com/bazelbuild/starlark`), "The Logic of CUE" (`cuelang.org/docs/concept/the-logic-of-cue/`), the Helm template guide on control structures (`helm.sh/docs/chart_template_guide/control_structures/`) and the CloudFormation `Fn::ForEach` reference. The Ansible loops, Terraform `for` expressions, Nix manual and YAML 1.2.2 pages were fetched but extraction failed (metadata only), so every statement about those four is **from memory, not verified**.
- Performance claims in section 5 are **design hypotheses**. Nothing here was benchmarked.

## 1. Verdict on feasibility
**Short answer.** A world can be pure data for what it *declares* (names, IDs, classes, options, regions, entrances, rules, goal, slot data, text). It cannot be pure data for what it *does* unless the platform ships named primitives for the recurring imperative patterns. With a modest primitive set, an **estimated** majority of sampled worlds become fully data. A residue (ROM and mod output, client code, bespoke graph generators, custom state updates) does not, and the honest design gives it a separate home.

Rough sorting of the eight sampled worlds (estimates):

| Fully data with primitives (estimated) | Data core plus separate artifact | Not pure without a large new primitive family |
|---|---|---|
| checksfinder, apquest (generation side), doom_ii, faxanadu, musedash | lingo (logic and tables yes; painting and door shuffles need the graph primitive), pokemon_emerald (logic core yes; ROM output and client no) | sc2 (mission-order engine, pool-filter pipeline) |

### 1.1 Evidence per sampled world
Counted figures are line and grep counts from the report; "blockers" are my reading of its file-level notes.

| World | Counted facts | Constructs that block pure data | Pure-data share (estimate) |
|---|---|---|---|
| checksfinder | 134 lines; tables 36 lines; rules loop 20 iterations (`Rules.py:7-9`) | A per-location rule loop with a closure default arg (a `foreach` over a numeric range removes it); a random 32-bit seed in slot data (`__init__.py:63-70`) | about 85% |
| faxanadu | 698 lines: Locations 199, Items 58, Regions 66, Rules 79, Options 107, `__init__` 189; all rules `has`, `has_all`, `has_any` (`Rules.py:21-40`) | Pre-filled potions at shops; randomly chosen wing-boots shops (`__init__.py:80-122`); weighted random filler (`:158`); Poison forbidden in shops (`:62-63`); classification promoted by an option (`:142-143`) | 70 to 75% |
| apquest (generation side) | 885 lines; about 3,200 lines of in-tree client and game simulation (not world data); rules in both the serializable builder (`rules.py:5-6, 30-33`) and lambda form (`:63-66`) | Trap chance from a random draw; arithmetic over unfilled locations to size filler | about 75% |
| musedash | about 1,535 non-test lines; `MuseDashData.py` 757 lines of song rows (`:4-6`); rules `Has(song)`, completion `Has("Music Sheet", N)` (`__init__.py:278-304`) | Song pool chosen by DLC and difficulty options; plando (`:127`); trap counts; a seed-dependent location set over static name-to-ID tables | about 60% |
| doom_ii | 6,203 lines; tables 5,166 (Locations 3,442 + Items 1,095 + Regions 590 + Maps 39, counted sum, 83%); Rules 496 (generated); `__init__` 297 | Pool-ratio arithmetic (`__init__.py:190-282`); rule variants by a `pro` flag and episode functions (parameterized macros handle these) | about 90% |
| lingo | `data/LL1.yaml` 8,764 plus `ids.yaml` 1,625 = 10,389 data lines; about 2,450 Python lines; schema in the YAML header (`LL1.yaml:1-60`); append-only ID file (`utils/assign_ids.rb:1-13`) | Custom state predicates: panel colors, progressive doors, mastery counts (`rules.py:28-48`); painting and door shuffles; requirement closure (`player_logic.py`) | about 60% overall; rules 90% |
| pokemon_emerald | 8,495 Python lines plus about 20,000 JSON lines (estimate); `rom.py` 883, client 738; JSON tables, rules via the serializable builder (`rules.py:7, 21-70`) | HM and badge shuffling with `fill_subset_with_retries` (`__init__.py:526-634`); `collect` and `remove` overrides (`:782, 796`); species and move randomization; ROM patch building (`rom.py`, `:641`, a bsdiff base patch); a game client | logic core 55 to 60%; output and client about 25% of lines never data here |
| sc2 | about 36,960 lines (estimate); `locations.py` 14,176 with 531 lambdas (counted by the report); `rules.py` 3,827; `mission_order/` about 4,900; item tables 6,078 lines | The mission-order engine (a graph DSL with layouts, entry rules, presets); the pool-filter pipeline; cached per-option derived fields (`rules.py:60-110`); phase detection in `is_item_placement` (`rules.py` about `:103-108`) | about 65% of locations, items and rules; the engine not |

### 1.2 What the evidence says, in order of confidence
1. **High (counted):** the world surface is mostly tables. doom_ii is 83% tables by line count; lingo already ships as 10,389 YAML lines against about 2,450 Python lines.
2. **High (counted):** rules are overwhelmingly boolean logic over items, groups and counts. Only 3 of about 85 world folders import the serializable rule builder, but about 80 files contain `lambda state` (file-level grep, not a world count), and the sampled lambdas have the same shape. The shape is data; the spelling is code.
3. **Medium (my reading):** the imperative lines cluster in five patterns: seeded selection from a table, constrained shuffles or pinned placements, derived counters, graph or layout generation, and pool arithmetic. These match the report's escape-hatch hotspots (`worlds.md` section 6). Five patterns makes primitives plausible; it does not prove five suffice.
4. **Low (not tested):** how many worlds outside the eight need a sixth or seventh pattern (the report sampled 8 of about 85 folders). Section 8 exists to test this.

## 2. Design principle: closed, total, named
### 2.1 The principle
> A world file describes **what exists and what is true**. The platform owns **how things are chosen, shuffled and checked**. Authors pick from named behaviors; they never write one.

1. **Total expression language.** Every expression terminates: no user functions, loops, recursion or unbounded iteration. Macros are non-recursive substitutions inlined at build time; tables are finite and fixed in the package.
2. **Behavior lives in named primitives.** A primitive has a name, a version, a typed input schema, a precisely specified algorithm (every PRNG call), test vectors and a declared cost class. Primitives are the only place randomness or search happens.
3. **Explicit capability surface.** The world declares the language version and primitives it needs; a generator declares what it supports. A mismatch is a clear load-time error, never a silent difference.

### 2.2 What "total" means here and what it costs
- **Expression language:** the combinators of 01 D2.5 (which owns the list: boolean logic, `has`, `group_count`, `reach`, `opt`, checked integer arithmetic, `ref`, plus `level`, `count`, `counter`, `in`, `reach_event`). Integers only, no floats, no strings except comparing to enum keys. Authoring macros compile to `defs` in the canonical form (section 3.2).
- **`foreach` is expansion, not iteration.** It produces N copies of a template at build or load time, then is gone, and cannot depend on anything it produced. CloudFormation's `Fn::ForEach` has the same shape: a collection and a fragment whose identifier is substituted, expanded into a final template, with quotas that "apply to the resultant template" (page opened). We add one rule: expansion size is bounded by a declared limit (design value to be set, for example a cap on expanded node count).
- **Termination check is static.** The build rejects macro and reference cycles. Dhall shows the benefit of totality: type-checked expressions evaluate in finite time with a canonical normal form (page opened). We take the termination property by construction (no functions), not Dhall's type system.
- **Cost:** some real worlds will find this too small (section 6). That is the price of being analyzable; the primitive set is where the pressure is released, on purpose and slowly.

### 2.3 Named primitives (initial candidate list)
Each is a candidate to be proven by section 8, not a promise.

| Primitive | What it does | Replaces (sampled evidence) |
|---|---|---|
| `select` | Seeded choice of N table rows with `where` filters, exclusions, per-group minimums, bounded retries, a named PRNG stream | musedash song pool; faxanadu random shops |
| `place` | Pin items (or by filter) to filtered locations, seeded, before fill; reportable failure | faxanadu prefill; pokemon HM placement |
| `permute` | Constrained shuffle of one table column (within groups, forbidden pairs, bounded retries) | pokemon species and moves; lingo paintings (partial) |
| `rewire` | Entrance randomization over declared sets with coupling rules | painting and entrance randomization |
| `progressive` | Ordered chain granting increasing levels; `level(chain)` in rules | `collect`-based progressive items |
| `counter` and `set` | Derived values updated only by declared monotone increments | simple `collect` overrides (about half, estimated) |
| `weighted_filler` | Fill by integer weights; exact-sum rounding (largest remainder) for traps; `unfilled` count in expressions | apquest filler; faxanadu `random.choices`; musedash traps |
| `graph_template` | Regions and entrances from a declared layout (chain, ring, grid, tiers) with entry rules | sc2 mission order (in part, 6.3) |
| option `requires`, `conflicts` | Cross-option constraints with an error string key, checked before a generation is queued | `OptionError` in `generate_early` |
| `item_rule` | Forbid or allow item groups at location groups | faxanadu Poison-in-shops |
| `classification_when` | Item class as an expression over options | faxanadu `:142-143` |

### 2.4 Rule of three: admission policy for new primitives
A new primitive is admitted only when **all** hold:

1. **Three real worlds** by at least two independent authors need it today, shown by a diff of how each does it now (one world is a feature request, two a coincidence, three a pattern).
2. **Not expressible as a composition** of existing primitives plus the expression language; a written attempt is required.
3. **Test vectors**: world fragment, seed, expected output, including failure and retry cases.
4. **Two reference implementations** in different languages that agree on all vectors: the platform's, and an independent one. The web UI's rule evaluator (the web UI is Leptos first with a TypeScript fallback, D-70 and D-09 open; the evaluator stays TypeScript either way; doc 05 section 4.1) doubles as the independent conformance implementation of the CGM rule language, so the web tracker and the generator cross-check each other on the same vectors.
5. **Cost class and bound** declared, so the publish-time checker and fill scheduler can account for it.
6. **Versioned entry** (`name@N`). Behavior changes mint `name@N+1`; old versions stay supported per the deprecation clock in 01 D1.8.

Rejected proposals are parked in a pattern backlog; a composable need becomes a published macro or recipe; an admitted primitive is Experimental (`x.` prefix), becomes Stable (`name@1`) after two releases with real worlds using it, or is withdrawn or merged. This mirrors the lifecycle in 01 D1.8 and the two-implementations gate in 01 D6.3 at primitive granularity.

### 2.5 Versioning and capability flags
- `format: cgm-data/1` names the **file format**; `language: N` names the expression operators and semantics; each primitive has `name@N`.
- A world declares `requires: {primitives: [select@1, progressive@1, counter@1, weighted_filler@1]}` against its `language`. A generator publishes the same shape as its capability list. A world that requires something the generator lacks fails with `capability.missing` and the missing names, the same negotiation idea as 01 D1.3 applied to worlds.
- The registry (01 section 3.7) records the required set so the web world page can say "this world needs `rewire@1`, your generator does not have it" before a room is created.
- A world with no module reference is a **pure world**; only pure worlds may claim every analysis in section 5.

### 2.6 DSL-creep: lessons from systems that put logic in config
The trap is well known: a data format gains conditionals, then loops, then functions, and becomes a language without a specification, type system or tooling.

| System | What the opened source shows | Lesson for us |
|---|---|---|
| Helm | Text templating around YAML (`if/else`, `with`, `range`); a section on whitespace chomping because "YAML ascribes meaning to whitespace" and a stray indent yields a "YAML parse error" at render time | Never template the *text* of the data file; our expansion works on the parsed tree, valid by construction |
| CloudFormation `Fn::ForEach` | Added later via the `AWS::LanguageExtensions` transform; barred from `Parameters`, `Mappings`, `Rules`, `Hooks`; quotas apply to the expanded result | Loops arrive late under pressure; decide now where expansion is allowed and bound its output (table-only, size-capped) |
| Starlark | Deterministic and hermetic ("Executing the same code twice will give the same results"), yet a Python dialect with functions and `for` loops | Safe code, not analyzable data; we do not start from a general-purpose dialect |
| Dhall | "Total": finite evaluation, no recursion, canonical normal form, imports frozen by semantic hash; finite is not fast ("can still be very long") | Keep termination, normal form and hashing; bound expansion explicitly |
| CUE | Types are values on a lattice, order does not matter, compatibility is subsumption ("a new version may not forbid what was allowed in the older version"); it declines conflicting defaults as an NP-complete-smelling problem | Take order-independent merging and a defined compatibility relation (5.2); do not adopt CUE as the authoring language, since its unification semantics exceed what worlds need |

Ansible, Terraform and Nix (all from memory, not verified) show the same slope: `when`, `loop`, `count`, `for_each`, `dynamic` blocks, or a lazy functional language with general recursion. Terraform is a useful precedent for what authors ask for first (iteration and conditional inclusion).

**Guard rails we take:** (1) no text templating of the data file; (2) expansion is a bounded, table-driven tree operation; (3) termination by construction, canonical normal form, hash of the normal form; (4) order-independent definitions and a defined compatibility relation; (5) growth only through the written, vectored, two-implementation process, because the usual failure is each new need adding a feature to the core language.

## 3. Examples
Syntax is a proposal. Expression strings use a small closed grammar (4.4) that parses to the same tree as 01 D2.5's JSON combinators.

### 3.1 A small nontrivial world: "Tidewatch" (fictional)
`tidewatch/world.yaml` (about 80 lines, estimated by eye, not counted by tool):

```yaml
format: cgm-data/1
language: 1
requires: {primitives: [select@1, progressive@1, counter@1, weighted_filler@1]}
meta: {id: dev.example.tidewatch, name: Tidewatch, version: 0.3.0, license: CC0-1.0, authors: [Example Author]}

options:
  goal: {type: choice, default: lighthouse, values: [lighthouse, deep_vault]}
  hard_tides:                        # cross-option requirement, checked before generation
    type: toggle
    default: false
    requires: {when: "opt.goal == deep_vault", error: err.hard_needs_vault}
  shell_hunt: {type: range, min: 0, max: 20, default: 8}
  bonus_isles: {type: range, min: 0, max: 4, default: 2}
  trap_percent: {type: range, min: 0, max: 50, default: 10}

tables:
  isles:
    columns: {key: key, tier: int, shells: int}
    rows: [[sandbar, 1, 1], [kelp, 1, 2], [wreck, 2, 2], [storm, 3, 3]]

items:
  lamp: {class: progression}
  boat: {class: progression, chain: [rowboat, sloop, galleon]}   # progressive chain
  shell: {class: progression, count: "opt.shell_hunt"}
  pearl: {class: filler, weight: "100 - opt.trap_percent"}
  jelly: {class: trap, weight: "opt.trap_percent"}

counters:
  sea_power: {add: {rowboat: 1, sloop: 2, galleon: 4}}        # derived, monotone increments only

macros:                              # compile to canonical `defs`
  can_sail: "level(boat) >= 1"
  deep_sea: "level(boat) >= 3 or (level(boat) >= 2 and not opt.hard_tides)"
  enough_shells: "count(shell) >= opt.shell_hunt"

regions:
  harbor: {start: true, exits: {reef: "can_sail()"}}
  reef: {exits: {lighthouse: "has(lamp)", vault: "deep_sea()"}}
  lighthouse: {}
  vault: {}

events:
  beacon_lit: {region: lighthouse, rule: "has(lamp)"}          # logic-only, never on the wire

locations:
  harbor_chest: {region: harbor}
  reef_dive: {region: reef, rule: "has(lamp)", kind: priority}
  tower_top: {region: lighthouse}
  vault_relic: {region: vault, rule: "enough_shells()"}

select:
  bonus: {from: isles, count: "opt.bonus_isles", where: "row.tier <= 2 or opt.hard_tides", stream: bonus}

templates:
  - foreach: row in isles                                      # expands over the whole table (stable IDs)
    when: "row.key in bonus"                                   # instantiated only if selected
    region: {key: "isle_{row.key}", entry: {from: reef, rule: "level(boat) >= row.tier"}}
    locations:
      - {key: "isle_{row.key}_cache", rule: "count(shell) >= row.shells"}

filler: {use: weighted_filler, fill_remaining: true}

goal:
  by: goal
  variants:
    lighthouse: "reach_event(beacon_lit)"
    deep_vault: "reach(vault) and enough_shells()"

slot_data:
  options: [goal, hard_tides, shell_hunt]
  computed: {isles: "bonus", shell_goal: "opt.shell_hunt"}
```

`tidewatch/strings/en.yaml` (every user-visible string is a key, per 01 D2.4; a `fr.yaml` carries the same keys in French):

```yaml
opt.goal.name: "Goal"
opt.goal.lighthouse: "Light the beacon"
opt.hard_tides.name: "Hard tides"
err.hard_needs_vault: "Hard tides needs the Deep vault goal."
items.rowboat: "Rowboat"
locations.reef_dive: "Reef dive"
```

The file exercises options with string-key labels and a cross-option `requires`; classed items and a progressive chain; regions and entrances; macros; an event; option-gated content; a seeded `select`; a derived counter; goal variants; slot data; per-language strings. IDs are not in the file: they live in `ledger.json` (01 D2.6), which the build extends append-only, including the expanded `isle_<key>_cache` keys for **every table row** so IDs do not depend on which isles a seed selects.

### 3.2 Canonical compiled form of Tidewatch
`cgm build` converts authoring YAML into one canonical JSON document; a generator reads only that plus the ledger, never YAML.

| Canonical section | Contents after `cgm build` |
|---|---|
| `format`, `language`, `requires` | Same values; `requires.primitives` lists exact `name@N` pairs, sorted |
| `meta` | Same, keys sorted |
| `options` | Array in author order (order is meaningful in the UI); each has `key`, `type`, `default`, resolved `values`, `requires` as an expression tree plus `error` string key |
| `tables` | Columns with types; rows as arrays; CSV sources inlined and their SHA-256 recorded |
| `items`, `locations`, `events`, `regions` | Sorted by key; chains expanded to ordered lists with a derived `level` counter; every `id` resolved from the ledger |
| `defs` | Authoring `macros` inlined into every use site by default; kept as named nodes in `defs` if the author sets `keep_macros` (for explanations and tracker display) |
| `templates` | Static table `foreach` is **expanded** for key and ID allocation (4 isles give 4 candidate regions and locations, each with `when: {in: [row.key, {set: bonus}]}` left as an expression); the `when` stays because it depends on the seed |
| `select` | Kept as a primitive node (`select@1`) with expression-tree parameters; **not** run at build time |
| `filler`, `goal`, `slot_data` | Kept as expression trees and primitive nodes |
| `strings` | Not part of the logic hash; each language bundle has its own hash |
| `hashes` | `logic_hash` (all of the above except strings, canonical JSON), `strings_hash` per language, `ledger_hash` |

The `deep_sea` macro in canonical key order:

```json
{ "any": [
    { "ge": [ { "level": "boat" }, 3 ] },
    { "all": [
        { "ge": [ { "level": "boat" }, 2 ] },
        { "not": { "opt": "hard_tides", "eq": true } } ] } ] }
```

Two boundaries matter. **Build time** knows tables, keys, IDs and structure; it never knows options or the seed. **Load time** (the generator) resolves options, runs primitives with the seed, instantiates templates and produces a flat **instance graph**; option-only subexpressions (`opt.hard_tides`) fold to constants here, which is how `OptionFilter`-style branches disappear before fill (the report notes the same idea in the serializable builder).

### 3.3 A table-driven world: a rhythm game with hundreds of songs
The submitted artifact is a CSV exported from a spreadsheet plus a small YAML:

```yaml
format: cgm-data/1
language: 1
requires: {primitives: [select@1, weighted_filler@1]}
meta: {id: dev.example.beatrun, name: Beat Run, version: 1.0.0, license: CC0-1.0, authors: [Example Author]}
tables:
  songs: {source: songs.csv, columns: {key: key, title: text, easy: int, hard: int, pack: key}}   # about 400 rows, hash recorded
options:
  packs: {type: set, of: {column: songs.pack}, default: [base]}
  song_count: {type: range, min: 10, max: 400, default: 40}
select:
  setlist: {from: songs, where: "row.pack in opt.packs", count: "opt.song_count", stream: setlist, retries: 8}   # retries bounded by the primitive spec
templates:
  - foreach: row in songs
    when: "row.key in setlist"
    item: {key: "song_{row.key}", class: progression}
    locations:
      - {key: "song_{row.key}_clear", region: stage, rule: "has(@item)"}
      - {key: "song_{row.key}_bonus", region: stage, rule: "has(@item)"}
regions: {stage: {start: true}}
items: {music_sheet: {class: progression, count: "opt.song_count / 2"}}   # integer floor division
filler: {use: weighted_filler, fill_remaining: true}
goal: {rule: "count(music_sheet) >= opt.song_count / 2"}
```

This is the same `where` / `count` shape as the musedash pattern. Item and location IDs come from the ledger for **all** 400 rows (static name-to-ID tables, as musedash does today) while only the selected rows appear in a seed; adding a song is adding a CSV row, and the ledger assigns the next ID.

## 4. YAML safety, canonical form, tooling
### 4.1 Decision: authoring format versus canonical form
| Layer | Format | Who touches it |
|---|---|---|
| **Authoring** | A strict YAML 1.2 subset ("CGM-YAML"), or JSON, interchangeably | Humans, spreadsheets, scripts |
| **Canonical** | JSON with sorted keys, NFC strings, integers only (01 D2.1); hash = SHA-256 of canonical bytes; optional deterministic CBOR with the same data model | Generators, registries, trackers, any implementer |

A generator needs **only the canonical JSON reader and the rule and primitive specs**; it never needs a YAML parser or the string-expression parser. Only `cgm build` does, and it has its own conformance vectors (authoring text in, canonical bytes out). Front-end implementers are fewer and cannot silently change a game, because the published canonical hash shows what was built.

### 4.2 The strict YAML profile
The parser enforces (violations are errors with file, line and column):
1. **Schema-directed typing.** A scalar is typed by the schema field it fills, not by guessing: a `string` field containing `no` or `NO` stays the string; a `bool` field accepts only `true` and `false`.
2. **No anchors, aliases, merge keys (`<<`) or custom tags.** Reuse is by macros and tables (aliases enable expansion attacks and hidden coupling).
3. **No duplicate keys** (error, not last-wins). 4. **One document per file**, UTF-8, spaces only.
5. **Quoted strings** for any value containing `:` or `#`, a leading special character, or looking numeric (formatter and linter warning).
6. **No floating point.** Integers only; a value above 2^53 minus 1 is a quoted decimal string (matches 01 D1.6).
7. **Size and depth limits** on the parse (design values).

### 4.3 The Norway problem class
YAML 1.1 resolvers treat bare words such as `no`, `NO`, `n`, `off`, `on`, `yes` as booleans, so a country code `NO` can load as `false`; the same family covers sexagesimal and octal-looking numbers. YAML 1.2 narrowed the core schema, but library behavior varies (**from memory, not verified**: the YAML 1.2.2 page did not extract, so confirm against the spec and our chosen parser before publishing). Our defense does not depend on the version: schema-directed typing removes the class; bare scalars that would resolve differently under 1.1 or 1.2 get a lint warning ("quote this value"); conformance vectors include the known cases (`NO`, `no`, `off`, `y`, `1_000`, `0o14`, `1e3`, `0x1F`, `null`, `~`); and canonical JSON means downstream consumers never see YAML typing.

### 4.4 The compact expression strings
Rules are strings such as `"level(boat) >= 2 and not opt.hard_tides"`, parsed by a small closed grammar (a few dozen productions, estimated) to the same tree as 01 D2.5. Identifiers and integers only; operators `and or not == != < <= > >= + - * / %` with integer floor semantics and checked overflow; calls only to built-ins (`has`, `count`, `level`, `reach`, `reach_event`, `counter`, `group_count`) and macros; no assignment, lambdas or string concatenation. The structured map form stays valid, and a formatter converts between them. Risk: a string grammar is a second parser to specify. Mitigation: it exists only in `cgm build`, is covered by vectors, and the canonical form is the tree.

### 4.5 Alternatives considered
| Format | For | Against | Verdict |
|---|---|---|---|
| Strict YAML profile | Comments, readable nesting, familiar to the existing community (player option files are YAML, per the report) | Typing pitfalls, mitigated above | Authoring format (chosen) |
| JSON | Unambiguous | No comments, noisy to hand edit | Equivalent authoring form; the canonical form |
| TOML | Clear typing, comments | Deep trees (regions with exits, rules) become verbose | Rejected for authoring |
| KDL | Readable, node oriented | Young ecosystem, fewer editors and libraries (**not verified**) | Rejected for now; revisit |
| CUE, Dhall, Starlark, Jsonnet | Constraint or programmable authoring (CUE subsumption page opened) | The submitted artifact would be code or need evaluation; every implementer would need the language | Author-side tooling only, not the submitted artifact |

### 4.6 Editor support and tooling
- **JSON Schema (2020-12)** for the canonical form and the authoring profile, published with the standard, for completion and inline errors in editors with YAML plus schema support (**from memory, not verified**).
- **Language server** (one Rust binary reusing the build front end): the same error codes as `cgm build`, go-to-definition for macro and item keys, a rename action that writes an `alias` entry (plain renames are breaking, 5.2), hover showing the expanded rule. **Formatter** `cgm fmt`: one canonical layout and quoting policy.
- **Visual region-graph editor** reading canonical JSON and writing authoring YAML with comments preserved; because the structure is data, it is a view, not a second source of truth (design hypothesis: this lowers the barrier for non-programmers; untested). **Strings** are flat per-language key-value files that translation tools already handle.

### 4.7 Thousands of locations
Authors do not hand-type thousands of rows; the submitted artifact is data and the generating tooling is the author's business: **CSV tables** via `source:` (hash recorded); **build scripts** in any language that emit YAML, CSV or JSON (the registry accepts only the output and never runs the script; `meta.built_with` may record the script hash, informational); **`foreach` over a table** (400 songs are 400 CSV rows and about 10 lines of YAML); and **ranges** as a built-in finite integer table (checksfinder's 20-step rule becomes `foreach n in range(1, 20)`, not a loop).

## 5. What the platform can do because it is data
All performance statements are **design hypotheses** unless marked; none were benchmarked.

| Capability | How data enables it | Status |
|---|---|---|
| **Rules compile to bytecode or decision diagrams** | A closed integer-only tree lowers to flat bytecode or a reduced diagram per rule, with no interpreter for user code | Hypothesis. The report's concern is arbitrary Python lambdas in the fill hot loop (`worlds.md` section 4) |
| **Option constant folding** | After options resolve, subtrees like `opt.hard_tides` fold away before fill | Hypothesis; the serializable builder in the sampled ecosystem does a similar fold |
| **Incremental reachability** | With monotone rules, adding an item only adds reachable nodes; dependencies are explicit in data, not hand-registered | Hypothesis |
| **Precomputed region closures** | Static graph parts can have transitive closures cached per world hash | Hypothesis |
| **Parallel fill and caching** | The instance graph is immutable; per-slot instance building parallelizes and is cacheable by (`logic_hash`, options, select results) | Hypothesis; cross-slot fill is not trivially parallel |
| **Content-addressed results** | Output = f(world hash, ledger hash, primitive versions, options, seed, generator id); same key, same result, store once | Design requirement (01 D3.2); benefit hypothesis |
| **Publish-time static checks** | 5.1 | Cheap checks certain; solver ones hypothesis |
| **Beatability across option combinations** | Monotone propositional logic plus counters makes "goal reachable with every item" a closure; across the option product use SAT or pseudo-boolean encoding, or sampled search | Hypothesis. Proves the logic *can* be satisfied, not that a given fill succeeds |
| **Diffs with compatibility classification** | Canonical versions compare structurally (5.2) | Certain for structure; logic-change semantics hypothesis |
| **Automatic docs and trackers** | Option pages, lists, region graphs and rule explanations come from the same data; a web tracker evaluates the same tree | Design goal |
| **No sandbox for the native tier** | A pure world has no executable content, so upload is parse-and-validate, not run-in-a-VM. The declarative interpreter runs in the generation fleet, never the serving process (D-15) | Strong security consequence; the generator stays the trusted path |
| **Tiny files, easy translation** | Tables compress well; strings are separate | Hypothesis; lingo is 10,389 data lines (counted), no compressed size measured |

### 5.1 Publish-time static checks (certain unless noted)
1. Ledger rules (01 D2.6): duplicate keys, reused or retired IDs, ledger not a superset.
2. Every reference resolves: item, location, region, macro, table column, string key (all languages report missing keys).
3. Macro and reference cycles; expansion size over the cap; expression depth and node limits.
4. Type checks: option references, integer-only arithmetic, division by a possibly zero expression.
5. **Unreachable locations**, even with every item and option combination (the option-product part is the solver hypothesis).
6. **Dead options** that change nothing (no reference and not in `slot_data`).
7. **Impossible `requires`**: a cross-option constraint set with no satisfying assignment (likely cheap, hypothesis).
8. **Item and location counts**: for every option combination checked, `locations >= items` and the filler can balance (necessary conditions; sampled for large spaces).
9. Non-monotone rules flagged per 01 D2.7.
10. Primitive parameters: `select.count` that can exceed available rows, `retries` declared, stream names unique.

### 5.2 Compatibility classification (diffable versions)
The registry compares the new canonical document to the last published one. Inspired by CUE's subsumption view (page opened); the classes are our own proposal.

| Change | Class | Why |
|---|---|---|
| Add an item, location, region, option, choice value or string | Non-breaking | Ledger append; old settings strings still parse |
| Remove or rename a key | **Breaking** | IDs retire, old option files and trackers break; renames expressed as `alias` entries are non-breaking |
| Change a default | Behavior change | Old settings strings that omit the option mean something else (settings strings store a difference from defaults, 01 D3.1) |
| Loosen a rule | Logic non-breaking | Old placements stay beatable |
| Tighten a rule | Logic breaking | A seed from the old logic may not be beatable; seeds are pinned by hash (01 D3.2), so old rooms are safe but trackers on new logic would disagree |
| Change a `slot_data` field shape | **Breaking for clients** | Clients parse it |
| Change a primitive version | Breaking for reproducibility | Same seed, different result |

### 5.3 Costs to be aware of
Worlds become **less expressive than code**, deliberately; authors whose world does not fit and our primitive maintenance pay for it. **Two reference implementations per primitive** is real cost. A static analysis suite (SAT encoding, reachability) is itself code that can be wrong; keep it separate from the generator and cross-check with sampled generation.

## 6. The hard residue
For each pattern from the report's escape-hatch list: how it becomes a primitive or a separate artifact, and an honest verdict.

| # | Residue | Evidence | Proposed home | Verdict |
|---|---|---|---|---|
| 6.1 | Seeded selection with retries and constraints | musedash pool and plando; faxanadu shops; pokemon `fill_subset_with_retries` (`__init__.py:602`) | `select`, `place`, `permute` with bounded retries in the algorithm; failure is a deterministic error | **Primitive.** Likely covers most; the hard cases are constraints over fill outcomes (pokemon HM placement needs "reachable before"), which `place` needs an `ensure:` clause for. Not yet proven by three worlds |
| 6.2 | Derived state with custom updates | pokemon `collect` and `remove` (`:782, 796`); lingo closure (`rules.py:28-48`); sc2 per-option caches (`rules.py:60-110`) | `counter` and `set` with monotone increments; closures as a `closure` primitive over a declared graph | **Partial.** About half of `collect` overrides are plain counters (report estimate). Non-monotone updates and arbitrary cached predicates **cannot** be data without losing analyzability and go to the residue options |
| 6.3 | Graph and layout generators | sc2 `mission_order/` (about 4,900 lines, counted by the report); painting and entrance randomization | `graph_template` (chain, ring, grid, tiers); `rewire` | **Primitive for entrances; open for sc2.** Its mission order is effectively its own DSL. Paths: a larger `graph_template` family proven by porting it (maybe several primitives), or a certified module. We should not promise sc2 is pure |
| 6.4 | Phase-dependent logic | sc2 `is_item_placement` (`rules.py` about `:103-108`) infers fill versus pool filtering from a group query | Explicit data: a `pool_rules` section for pruning and a `play_rules` section for logic; no expression may ask which phase it is in | **Redesign, not port.** A bit-exact port of phase inference is neither possible nor desirable; the author restates the intent |
| 6.5 | Pre-fill, post-fill, special placement | faxanadu prefill; alttp-style stage hooks (report) | `place` covers pre-fill; post-fill becomes declarative `assert` checks and `swap_allowed` policies; no direct list mutation | **Mostly yes.** Post-fill code that rewrites placements is not expressible, the clearest case for 6.9 |
| 6.6 | Custom option types and cross-option validation | PlandoBosses-like text options; `OptionError` in `generate_early` | Closed option kinds (toggle, choice, range, text with regex, set, counter, list, small structured record); `requires` and `conflicts` with message keys; plando as a platform feature | **Yes** for the sampled cases; arbitrary validators **cannot** |
| 6.7 | ROM and mod output | pokemon `rom.py` 883 lines, adjuster 239, bsdiff base patch (`:641`) | A separate optional third-party-facing artifact (CPK, 01 D4.1; D-01, D-07): a diff-only patch plus a **seed delta**, data with a schema, size caps and scanning, never publisher data. A declarative `patch_table` primitive writes fixed-layout tables; connector profiles (01 D4.3) cover runtime-memory games | **Partly.** Fixed-layout tables are data. Compression, checksums, pointer relocation and algorithmic rewriting are code, living in a third-party patcher or bridge outside the first-party product, not in the world |
| 6.8 | Client code | pokemon client 738 lines, SNI or memory readers | Separate artifact; the world may reference an optional third-party client or connector profile by id and hash. Games embed CSDK (01 D5.1) or implement CWP; we ship no game-specific client (D-01) | **Cannot be data.** Out of scope of the world file by design |
| 6.9 | World-specific generation code that fits none of the above | Unknown count; sc2 and pokemon each have some | Options below | Needs a decision |

### 6.1 Options for the residue
**(a) Grow primitives only.** For: worlds stay 100% data, full analysis holds, no code to sandbox, one conformance story. Against: the set may grow large; one hard world (sc2) may demand a whole family; authors who do not fit are blocked until a primitive is admitted (the deliberate rule-of-three delay); DSL-creep risk (2.6) if admission slips.

**(b) Separately versioned, certified WASM world modules referenced by name and hash; the world file stays data.** For: gives the residue a home without opening the world file to code; the module is a registry artifact with its own review, certification and fuel limits (01 D2.8 rules carry over); analyses hold except for the module's declared effects (`touches: [items, regions]`, checkable at the host API boundary). Against: it is code, so the sandbox, determinism, host API and fuel work (all **not benchmarked**) is needed after all; a non-first-party module would run inside the microVM or a vetted gVisor host as a second layer under the WASM sandbox, never in the serving process (D-15); a module defeats static analysis for what it touches (rules that call it cannot compile to diagrams); boundary-crossing cost in hot loops stays a worry (the report flags it, not benchmarked).

**(c) Allow inline code in the world file.** For: simplest for authors. Against: breaks the owner's premise; the world is no longer analyzable or hashable as data; every upload needs a sandbox; every other language's generator must run it. This is the current ecosystem's situation and the reason a non-Python generator is costly (the report lists seven blockers).

### 6.2 Recommendation
**Adopt (a) as the spine, build the door for (b) but keep it closed until evidence says open it, reject (c).** This is the working starting direction (D-10, proposed), not locked; the owner can reopen it after the section 8 spike.

1. **Native format v1 has no module reference.** `pure` is the only mode and world modules are a reserved, disabled door (01 D2.8). The conformance suite, the registry badge and every section 5 analysis apply to it.
2. The format reserves a `modules:` key and a `touches:` declaration, rejected by v1 generators with a clear message, so adding (b) later is not a breaking change.
3. Hard worlds are served **today** by the compatibility sidecar (01 section 3.8.2): they run unchanged as opaque worlds, the pressure valve that removes urgency to open (b) early.
4. Only when the section 8 plan shows a specific residue that fails the rule of three but blocks several real worlds do we open (b), for that residue, as a separately certified module family with stricter review than primitives.

**What would change this recommendation:** more than about one third (target, not a measurement) of each spiked world's declared behavior left unexpressed after honest primitive attempts (open (b) sooner); primitive proposals piling up without reaching three worlds (the bar is too high or the ecosystem more diverse than the sample); a WASM toolchain spike (01 section 7) showing low boundary cost and stable determinism on two runtimes (the argument against (b) weakens); or an owner priority of "migrate the existing community with minimum rewriting" (the sidecar or (b) outranks purity).

## 7. Migration and compatibility
### 7.1 Legacy Python worlds
Unchanged, through the sidecar (01 section 3.8.2): they are opaque worlds with IDs and names lifted into a synthesized manifest, options introspected, generation pinned to the sidecar version, Valid conformance tier at most. Native pure worlds and opaque worlds coexist in one multiworld, and the seed bundle records which is which. Policy lives elsewhere: the sandbox lanes (D-04) in doc 04 sections 4.1 and 4.2; race integrity (D-08), where a room with an opaque world runs casual races only, in 01 D3.3; validation (D-14), where strict reachability applies to native worlds and opaque worlds get existence and duplicate checks only, in 01 section 3.8.

### 7.2 A best-effort extractor
An optional tool (not a promise) that reads a legacy world and emits a draft `world.yaml` plus a report of what it could not convert. Convertible with good confidence (estimate): static item, location and region tables and name-to-ID dicts; option classes by introspection; item groups, classifications and simple counts; plain `lambda state: state.has(...)` forms that are a pure `and`/`or` of `has` calls. Needs a human: serializable-builder rules (3 sampled worlds use it; mapping is mostly renaming), weighted filler and trap logic, `create_items` loops that read options, pre-fill and `select`-like randomness. Cannot convert: arbitrary lambdas with closures and helper calls, `generate_output` and ROM code, `collect` and `remove` overrides with arbitrary logic, client code. How well the pattern matcher does on lambdas is **not tested**; the report's observation that rules are mostly boolean over `has` (faxanadu `Rules.py:21-40`, all of it) is a sample of one.

### 7.3 Gradual path for an author
(1) **Opaque**: the world runs in the sidecar as is. (2) **Data tables extracted**: names, IDs and options lifted into YAML; the sidecar still runs rules (hybrid). (3) **Rules converted**: the logic core becomes the expression tree; the sidecar runs only remaining hooks. (4) **Primitives replace hooks** one by one. (5) **Pure**: no sidecar, no module; registry badge, full analysis, native speed. A hybrid world is honestly labeled "data plus a declared residue" and cannot claim pure-world analyses.

### 7.4 Conformance tests that prove equivalence
Bit-identical output between a legacy world and its port is **impossible** in general: its randomness comes from Python's generator (the report notes a non-Python generator cannot match it without reimplementing CPython's algorithms exactly) and ours is ChaCha8 (01 D3.2). So equivalence is layered:

| Layer | Test | Strength |
|---|---|---|
| Names and IDs | Name-to-ID tables identical, ledger superset | Exact |
| Options | Same names, types, defaults, accepted values; the same player YAML parses | Exact |
| Pool composition | For N sampled option sets, the item multiset equals the legacy pool | Exact for deterministic parts; distribution-level where random |
| Rules | The legacy world exports a reachability oracle for sampled states (the sidecar evaluates its rules); the port agrees on every sampled (state, location) pair | Sampled |
| Randomized primitives | Chi-square or coverage test on selection frequencies against the legacy world's observed frequencies, plus invariants (counts, uniqueness, constraints) | Statistical |
| Beatability | All generated seeds pass the independent validator (01 D6.1) | Exact per seed |
| Slot data | Same keys and value shapes for the same options | Exact for deterministic fields |

These tests double as the extractor's acceptance harness.

## 8. Risks, evaluation plan, open questions
### 8.1 Risks
| Risk | Why it matters | Mitigation |
|---|---|---|
| Primitive set grows into a programming language by accretion | The ordinary failure mode (2.6) | Rule of three, vectors, two implementations, named versions, no user functions ever |
| Pure format excludes the biggest worlds | sc2 and pokemon_emerald may stay in the sidecar | Be honest: the sidecar stays first class; the badge describes the pure world |
| Primitive semantics diverge between implementations | Same world, different seeds | Vectors, two reference implementations, Reproducible tier (01 D2.9) |
| String expression grammar becomes a language | Authors ask for features | Integers, built-in calls and macros only; no change without the admission process |
| Static proofs mislead | "Beatable across options" is a necessary condition, not a fill promise | Name exactly what is proven (reachable with all items); run the independent bundle validator on real seeds |
| YAML library differences | Different parsers read the same file differently | Schema-directed typing, our own `cgm build` vectors, canonical JSON as the contract |
| Author friction | Authors who know Python may resent a closed language | Excellent errors, a formatter, the extractor, recipes per primitive, the sidecar path |
| Compatibility claims | Statements about the legacy ecosystem come from a dated snapshot | Date-stamp; recheck before publication (`CLAUDE.md`) |

### 8.2 Evaluation plan: port three real worlds by hand (spike)
Done by hand on a branch, not shipped, to find the constructs that break the model.

| World | Why this one | What it stresses |
|---|---|---|
| **musedash** | Table-driven; 757-line song table; seeded song selection by options; two locations per song; trap counts | `select`, `foreach` over a table, CSV source, `weighted_filler`, count arithmetic; plando as a platform feature |
| **faxanadu** | Small, hand-written; seeded shop placement, prefill, weighted random filler, an item rule, option-driven classification | `place`, `item_rule`, `classification_when`, `weighted_filler`, rule macros |
| **lingo** | Already data (10,389 YAML lines) with a Python interpreter, painting and door shuffles, an existing ID ledger | `rewire`, custom state predicates (closure, mastery counts), whether `counter` and `set` suffice, scale (about 10,000 data lines to canonical form) |

Held back on purpose: doom_ii is expected to be easy (about 90% tables, estimate) and would inflate the success rate; pokemon_emerald and sc2 stress the *residue* (section 6) and belong to a second spike. A one-day look at the sc2 mission-order engine is a reasonable add-on to size the graph family.

**What success looks like** (targets, not measurements): (1) **Expressiveness:** an itemized list per world of every hook and rule construct, each marked "data", "needs new primitive (does it meet the rule of three?)" or "residue"; target at least two of three worlds with no residue. (2) **Equivalence:** names, IDs and options identical; pool multisets identical across at least 50 sampled option sets per world (target count); rule-oracle agreement; statistical agreement for randomized primitives. (3) **Beatability:** generated seeds pass the independent validator on all sampled seeds. (4) **Size and clarity:** counted lines of data versus original Python plus data, reported in both directions. (5) **Primitive count:** distinct primitives the ports used, and how many were used by two or more worlds; a set that keeps growing per world is a warning sign. (6) **Static checks earn their keep:** run the publish-time checks on each port and report real defects (any found in the legacy world are reported upstream first, per `CLAUDE.md`). (7) **Author experience:** one non-author reads a ported file and explains the logic back; note where they stumbled.

### 8.3 Open questions for the owner
Not yet in DECISIONS Part C; the doc 07 rows need adding there. Recommended defaults in brackets.
1. Is `pure` the only native mode in v1, with the sidecar as the escape for everything else? [yes, with a reserved, disabled `modules:` door; D-10, proposed, not locked]
2. Do you accept that some worlds (sc2 mission order, ROM writing) may never become pure, and that the honest position is "native format for the tractable majority, sidecar for the rest"?
3. Authoring syntax: compact expression strings (a small grammar, only in the build tool) or structured trees only, at a cost in readability? [strings plus formatter]
4. How slow should primitive admission be? Should a studio paying for a primitive be able to skip the rule of three (a risk to neutrality, 01 D6.4)?
5. Should every primitive be Reproducible-tier from day one (exact PRNG calls specified)? [yes, since the primitive spec includes the algorithm]
6. Spike scope and owner: the three ports need an agent or a contributor; first pass by us or opened to the community?
7. Naming: `cgm-data/1` is a placeholder; format and capability names need the trademark review flagged in 01 open question 1.
8. Does the hosted product show a "pure" badge on worlds and prefer pure worlds in fast paths (cache, parallel fill)? That is product policy, not spec.
