# 15. The settings file: one plain-language standard for player and room settings

> Internal design draft, 2026-11-16. Not reviewed by a different agent or a human; under [`CLAUDE.md`](../../CLAUDE.md) nothing here leaves this repo until that review and a human sign-off. Our own writing, with citations, and no Archipelago source code, text or assets. Numbers are labeled **counted**, **estimated**, **design value** or **not verified**; nothing here was measured. Terms: [`GLOSSARY.md`](GLOSSARY.md). Decision ids (D-nn) are in [`DECISIONS.md`](DECISIONS.md); rows proposed here are `D-TBD-S1` and onward, so they do not collide with doc 14's `D-TBD-a` series.

Builds on doc 07 (option kinds, strict YAML profile, canonical JSON), doc 01 D2.3 (typed options with human text), D2.4 (string keys), D3.1 (seed phrase and settings string) and D3.3 (mystery settings), doc 14 (option `scope`, presets, tricks and tiers, spoiler policy) and doc 09 (where options are chosen and locked).

## 0. Sources and their limits
- **Facts about the existing multiworld project's common options** come from a check of its `Options.py` made on our behalf in November 2026 (the requesting agent's verification; I did not open that repository). From that check: "Progression Balancing" is a 0 to 99 slider with default 50 and named stops at 0 (disabled), 50 (normal) and 99 (extreme), and its code describes it as a way to move progression earlier so a player is less likely to be stuck early. The other common options checked are Accessibility, Local Items, Non-local Items, Start Inventory, Start Inventory from Pool, Start Hints, Start Location Hints, Exclude Locations, Priority Locations, Item Links, Plando Items, Plando Connections and Death Link. Player files express randomness as relative weights.
- **The weight problem**: our notes record that the option template writes `option: 50` meaning a weight (`data/options.yaml:80-88`, verified in [`../notes/01-archipelago-findings.md`](../notes/01-archipelago-findings.md)).
- The **values** of those options (for example which choices Accessibility offers) were not part of the check. Where this doc maps them, the mapping is by meaning and the value names are **not verified**.
- How other randomizers name and share settings comes from the batch summaries in [`../research/randomizers/`](../research/randomizers/) (A to D). Community aliases listed in section 4 are words players use in public; they are listed so search works, not copied text.
- The existing project's design suits people comfortable with technical setup, which is who its volunteers built it for. This doc is about a different front door for a wider audience, and every mapping in section 6 exists so files keep working in both directions.

## 1. Principles

1. **Describe the experience, not the algorithm.** A label says what the player will notice ("Start with these items"), never how the generator gets there ("pre-collected inventory"). If the honest description is "the generator moves your key items earlier", say that, in those words.
2. **Every option carries five things**: a plain **label**, a one-sentence **what it does**, an optional **why change it**, a **default**, and at least one **example** in the help text or the file template. A missing one is a lint error (section 5.2).
3. **No jargon without a plain alternative.** A term from the jargon list (section 5.3) may appear only as an alias or in the "why" line next to its plain meaning, never alone in a label.
4. **Community names stay findable.** Every option may list **aliases**: the names players already search for, such as "keysanity", "ER", "plando", "deathlink" or "balancing". The settings screen's search matches aliases and shows "also called: keysanity" under the label. Aliases are never the label.
5. **One vocabulary for the common concepts.** Starting items, shared deaths, where keys can be, and the other concepts in section 4 have one stable key, one label and one explanation across every game. A world author cannot redefine them, only enable them and set defaults.
6. **Reading level and length targets** (design values, to be checked against real players): labels at most 32 characters, "what it does" one sentence of at most 120 characters, "why change it" at most 200 characters, all aimed at about a grade 6 to 8 reading level in English. Other languages get length limits scaled per language by the translation team (open question 3).
7. **Every string is a translation key.** No label or help text sits in the settings file or the manifest body (doc 01 D2.4). Common concepts use platform keys (`common.start_with.label`); world options use world keys (`opt.<key>.label`, `.what`, `.why`, `.values.<value>`, `.example`).
8. **Say who decides.** Each concept belongs to exactly one place: the **player file** (one seat's preferences), the **room settings** (the host decides for everyone) or the **game's own options** (defined by the world, chosen per player). Section 4 assigns each one.
9. **Positive wording.** "Never put important items here", not "Exclude non-filler from locations". No double negatives in labels or values.

## 2. The settings file format

### 2.1 Authoring and canonical form
- **Authoring**: YAML in the strict profile of doc 07 section 4.2 (schema-directed typing, no anchors or aliases, no duplicate keys, integers only, one document per file). The same file may be written as JSON. Because typing is schema-directed, a choice value `no` stays the word "no" and cannot silently turn into `false` (doc 07 section 4.3).
- **Canonical**: JSON with sorted keys, NFC strings and integers only. Defaults are filled in from the preset and then the world, aliases resolved to stable keys, and `surprise` entries kept as specs (they resolve only at generation, from the seed).
- **Settings hash**: SHA-256 of the canonical bytes. The lobby binds "options locked" to this hash (section 2.5), and two files with the same meaning always have the same hash whatever their comments, key order or spelling.
- **Scope** (doc 14): options with `scope: cosmetic` are kept in the file but left out of the settings hash input that feeds the seed, so changing a cosmetic never changes the game. `knowledge` options (hints, spoilers) are hashed and shown to racers.

### 2.2 Top-level structure

| Key | Required | Meaning |
|---|---|---|
| `format` | yes | `settings/1` (placeholder name, pending the naming review in doc 01 open question 1) |
| `game` | yes | The world id from the registry, such as `dev.example.lanternfall` |
| `game_version` | no | A SemVer range; empty means "the version the room pins" |
| `player_name` | no | Display name suggestion; the lobby may override it (doc 09 section 4.1) |
| `preset` | no | A named settings file to start from (2.6); this file's values override it |
| `options` | no | Map of stable option keys to chosen values (common vocabulary and the game's own options in one map) |
| `surprise` | no | Map of option keys to "surprise me" specs (section 3); a key may not appear in both `options` and `surprise` |

The common vocabulary keys in section 4 are **reserved**: a world may not define an option with the same key (lint error `settings.reserved_key`). New common keys are added only through CRFC (doc 01 D6.3), and only with a name no published world already uses (the registry can check this).

Room settings use the same rules in a separate file, `format: room-settings/1`, written by the host (doc 09 section 7.1, `draft` and `open` states). A room-level key in a player file is an error with a plain message (section 7).

### 2.3 A beginner example (fictional game "Lanternfall", doc 14 section 4.2)

```yaml
format: settings/1
game: dev.example.lanternfall
player_name: Mira
preset: relaxed
options:
  goal: find_the_crown
  start_with: [lantern]
  shared_deaths: off
surprise:
  traps: {one_of: [none, a_few]}
```

### 2.4 An advanced example (same fictional game)

```yaml
format: settings/1
game: dev.example.lanternfall
game_version: ">=0.1.0 <0.2.0"
player_name: Mira
preset: standard
options:
  # common vocabulary (section 4)
  keep_moving: strong          # "Keep everyone moving"
  must_reach: everything       # "What must be reachable"
  keep_home: [lantern, rope]   # "Keep these items in my game"
  send_away: [crown_piece]     # "Send these items to other players"
  start_with: {items: [boots], take_from_shuffle: true}
  reveal_items: [pickaxe]      # "Show where these items are"
  reveal_places: [vault_altar] # "Show what is at these places"
  no_important_at: [well_bucket]
  important_at: [mine_cart]
  logic_level: normal          # tier (doc 14 section 3)
  tricks: {ledge_clip: beginner, dark_room: on}
  key_shuffle: own_area        # "Where keys can be"
  door_shuffle: off            # "Shuffle doors"
  item_pool: standard
  # this game's own options
  shuffle_mine: true
  hint_count: 6
  lamp_color: teal             # cosmetic scope: never changes the game
  hand_placed_items:
    - {item: pickaxe, place: mine_cart}
surprise:
  goal: {one_of: [find_the_crown, light_every_lamp]}
  traps: {chances: {none: 50, a_few: 30, lots: 20}}
  hint_count: {anywhere_between: [3, 8]}
```

### 2.5 From file to screen, and back
- The settings screen is generated from the world's options and the common vocabulary (doc 01 D2.3; batch A item 8 and batch B item 8 record that projects which drive the UI from one settings declaration avoid drift). Each option renders as its kind dictates: toggle as a switch, choice as radio buttons (a select box above 7 values, design value), range as a number field plus slider with named stops, item and place sets as a searchable multi-select, tier as an ordered radio group.
- Groups: the screen shows **Basics** (goal, difficulty, start items), **Your game** (the world's own options), **Playing with others** (keep moving, keep home, send away), **Help and hints**, **Looks and sound** (cosmetic scope) and **Advanced** (hand placement). Common concepts always sit in the same group in every game.
- The screen reads and writes the same canonical JSON. Download gives the YAML form with each value commented by its label in the player's language; upload runs the validator first (section 7).
- In the lobby, `set_options` carries the canonical JSON and `lock_options` freezes it; the seat shows the settings hash prefix, and any change after lock bumps `lobby_rev` and revokes ready (doc 09 sections 5.2 and 7.1).
- A game may render the same screen in-game through CSDK `settings.ui` (doc 01 D5.1); it uses the same keys and strings.

### 2.6 Presets are settings files
A preset is a settings file with a `name` string key and no `player_name`, published with the world (`presets/relaxed.yaml`) or by a community (registry entry by hash). Presets may layer at most two deep (a community preset on a world preset, design value) so a player can always see where a value came from. The screen marks each value "from preset" or "changed by you". Doc 14 section 2.3 notes a split between "what I can do" and "what I want" (VARIA's skill and rando presets); we keep one preset per file but allow a `skills` preset that only sets `logic_level` and `tricks`, combined with one other preset (open question 5).

### 2.7 The settings string
- Shape from doc 01 D3.1: `cs1-<base32(CBOR)>-<crc16>`. It holds the game id, the manifest hash prefix, the preset reference (by hash), **only the options that differ from the preset or world defaults**, surprise specs as specs, the generator id and version, and the seed phrase.
- Cosmetic options are left out. Knowledge options are kept. Under a race or mystery room, the string shown to players is the public-safe form with surprise results masked (doc 01 D3.3).
- Why a difference and not positional bits: doc 14 counts 6 of 16 projects with positional bit-packed strings bound to one version, and 2 with a difference from defaults (counted from the notes); the difference form survives new options without a migration (batch A, "settings strings as a diff from defaults").
- Example (illustrative, not a real encoding): `cs1-KF3Q...7TQA-9C2E`, shown in the UI with a "Copy" button and the seed phrase beside it, `amber-heron-quill-tide`.
- A room string is the room settings string plus one string per seat (proposal; length not estimated yet).

## 3. Randomness made simple: "surprise me"

### 3.1 The problem with bare weights
In the existing project's player files a number next to a value is a relative weight. Our notes record that the template writes `option: 50` meaning a weight (verified, `data/options.yaml:80-88`). For a newcomer, a toggle written with `true` next to `50` reads like "true equals 50" or "50 percent". It actually means "weight 50 out of the total of all weights listed", so a single listed value at 50 is always chosen, and two values at 50 and 25 split two thirds and one third. The format is powerful and compact, and experienced players use it well; the cost is that the meaning of a number depends on every other number in the block.

### 3.2 Three readable forms

| Form | Meaning | Validation |
|---|---|---|
| `one_of: [a, b, c]` | Pick one, each equally likely | At least 2 distinct valid values |
| `chances: {a: 50, b: 30, c: 20}` | Pick one with these percent chances | Integers 1 to 100, total exactly 100, valid values, no duplicates |
| `anywhere_between: [min, max]` | Pick a whole number in this range, each equally likely | Range options only; `min < max`; both inside the option's own limits |

Side by side, for the same intent ("usually no traps, sometimes a few, rarely lots"), with our own illustrative values:

```yaml
# legacy weighted shape: numbers are relative, so you add them up to know the odds
traps:
  none: 5
  a_few: 3
  lots: 2

# our shape: the numbers are the odds
surprise:
  traps: {chances: {none: 50, a_few: 30, lots: 20}}
```

And for "pick a number of hints from 3 to 8": the legacy file has its own range keywords (**not verified** here which ones), while ours says `hint_count: {anywhere_between: [3, 8]}`.

### 3.3 Validation and resolution
- Percentages must add to exactly 100. The error says the actual total and how far off it is (section 7).
- Surprise values resolve at generation from the seed's own stream (doc 01 D3.1, D3.2 cumulative integer choice), so the same seed and file give the same result.
- `toggle` options accept `one_of: [on, off]`; `item_set` and `location_set` accept `chances` over named sets only (no per-item lotteries in v1, design choice to keep files short).
- A surprise on a `cosmetic` option resolves on the player's device and never enters the hash.
- The room can switch surprises off (`allow_surprise: false`), for example for races where every seat must be known; a file with surprises is then rejected with a message naming the options.
- The resolved values are written into the seed bundle and shown to the player after generation (or after the game, under mystery rooms).

### 3.4 How it appears on screen
Every option has a small "Surprise me" control beside it. Turning it on swaps the input for a compact editor: a checklist for `one_of`, a set of percent fields with a live "total: 100" counter for `chances`, two number fields for `anywhere_between`. The option's row shows a dice icon and the text "Surprise: 1 of 2" or "Surprise: 3 to 8", never color alone.

## 4. The common vocabulary

**Where it lives**: **P** player file, **R** room settings (host), **G** game's own option using the shared key and words. "Legacy name" is the closest option in the existing project, from the check in section 0; value correspondence is **not verified** unless stated.

| Stable key | Plain label | What it does (one sentence) | Kind | Default | Legacy name | Community aliases | Lives in |
|---|---|---|---|---|---|---|---|
| `keep_moving` | Keep everyone moving | Moves some of your important items earlier so you are less likely to wait a long time with nothing to do. | choice (ordered): `off`, `gentle`, `strong` | `gentle` | Progression Balancing | balancing, prog balancing | P (room may lock) |
| `must_reach` | What must be reachable | Chooses whether every place in your game must be reachable, or only what you need to finish. | choice: `everything`, `all_items`, `just_the_goal` | `everything` | Accessibility | accessibility, full, minimal | P |
| `keep_home` | Keep these items in my game | These items will be found somewhere in your own game. | item_set | empty | Local Items | local items | P |
| `send_away` | Send these items to other players | These items will be placed in other players' games. | item_set | empty | Non-local Items | non-local items | P |
| `start_with` | Start with these items | You begin the game already holding these items. | record: `items` (item list with counts), `take_from_shuffle` (toggle) | empty, `false` | Start Inventory; Start Inventory from Pool (`take_from_shuffle: true`) | start inventory, starting items | P |
| `reveal_items` | Show where these items are | At the start you are told where these items were placed. | item_set | empty | Start Hints | start hints | P (room may disable) |
| `reveal_places` | Show what is at these places | At the start you are told what item is at each of these places. | location_set | empty | Start Location Hints | location hints | P (room may disable) |
| `no_important_at` | Never put important items here | These places only hold extras, so you can skip them. | location_set | empty | Exclude Locations | excluded locations | P |
| `important_at` | Put important items here | These places will hold something you need, if the game can manage it. | location_set | empty | Priority Locations | priority locations | P |
| `shared_items` | Share items with teammates | Players in a group share one copy of chosen items, so one find helps everyone in the group. | list of records: `name`, `items`, `players` | none | Item Links | item links, linked items | R (each player opts in) |
| `hand_placed_items` | Place items by hand | You choose exactly where certain items go, and the game checks it is still finishable. | list of records: `item`, `place`, optional `player` | none | Plando Items | plando, item plando | P (own game); R (across games) |
| `hand_connected_doors` | Connect doors by hand | You choose where certain doors or entrances lead, and the game checks it still works. | list of records: `from`, `to`, `both_ways` | none | Plando Connections | connection plando, ER plando | P |
| `shared_deaths` | Shared deaths | When one player in the group dies, everyone else in the group does too. | choice: `off`, `on` | `off` | Death Link | deathlink, DL | R allows it; P opts in |
| `on_finish` | When a player finishes | What happens to a finished player's remaining items: keep waiting, send them all out, or let others collect theirs. | choice per side: `send_my_items` (`never`, `when_done`, `anytime`), `get_my_items` (same values) | `when_done` both | Release and collect behavior (room-level in the existing project's hosting, **not verified** here) | release, collect, forfeit | R |
| `hint_price` | Hint price | How many points a player spends to ask where an item is. | range (percent of the player's places, integer) | design value per room | Hint cost (room-level, **not verified** here) | hint cost, hint points | R |
| `logic_level` | How hard the game can expect you to play | Sets which harder moves and routes the game may require of you. | tier (doc 14 section 3) | world default | per-world difficulty options (no common option) | logic, glitches, difficulty, tricks | G |
| `tricks` | Special moves you know | Lets you mark individual techniques you are happy to be required to use. | map trick to `on`, `off` or a level | all off | per-world trick options | tricks, skips, glitches | G |
| `goal` | Goal | What you must do to finish your game. | choice | world default | per-world goal option | goal, win condition | G |
| `spoilers` | Spoiler log | Who can see the full list of where everything is, and when. | choice: `none`, `creator_once`, `after_completion`, `public`, `metadata_only` (doc 14 `spoiler_policy`) | `after_completion` | generator and hosting setting, not a player option (**not verified** here) | spoiler, spoiler log | R |
| `item_pool` | How many extra items | Sets how many extra items are mixed in, from very few to plenty. | choice (ordered): `scarce`, `standard`, `plentiful` | `standard` | per-world pool options | item pool, filler | G |
| `traps` | Traps | Adds items that cause a small harmless nuisance when you get them. | choice (ordered): `none`, `a_few`, `lots` | world default | per-world trap options | traps, trap percentage | G |
| `key_shuffle` | Where keys can be | Chooses whether keys, maps and similar items stay where the original game put them or can be anywhere. | choice (ordered): `original`, `own_area`, `own_game`, `anywhere` | world default | per-world options | keysanity, small keys, big keys, mapsanity | G |
| `door_shuffle` | Shuffle doors | Doors and entrances lead to different places than in the original game. | choice (ordered): `off`, `some`, `all`, plus `mode` (`paired`, `one_way`) | `off` | per-world entrance options | ER, entrance rando, door rando | G |
| (attribute) `scope` | Changes the game, changes hints, or only looks | Not an option: every option declares whether it changes the game, changes what players know, or only changes looks and sound. | attribute: `seed`, `knowledge`, `cosmetic` | `seed` | none (per-world conventions) | cosmetic, aesthetic | each option |

### 4.1 "Keep everyone moving" in detail
The legacy slider is a single number whose effect is hard to predict from the number. Our model keeps the same goal (avoid long stretches where a player has nothing reachable) and names three levels by what a player feels:

| Level | What you will notice | Generator behavior (CFill v1 parameter, doc 01 D2.7 "Balancing") |
|---|---|---|
| `off` | Fully random order. You might wait a while for another player to find your next key item. | No balancing pass. |
| `gentle` (default) | Usually something to do early, but surprises still happen. | A balancing pass that moves a bounded share of your progression earlier (design value, to be tuned with real games). |
| `strong` | You will rarely wait. Your important items tend to show up early, often in your own game or in early places. | A stronger pass, plus a few early places held for you (`local_count`, doc 14 section 2.7). |

Each level's exact algorithm is in the CFill spec with test vectors. We do not claim our levels reproduce the legacy slider's numbers; the mapping in section 6 is nearest-meaning only. Whether this belongs per player or per room is open question 1, because one player's setting changes where other players' items go.

### 4.2 Why some legacy per-player options move to the room
Item links, shared deaths, release and collect, hint price and spoilers affect other seats, so in our model the host decides (or allows) them and each player opts in where that makes sense. A player file that sets them is not silently obeyed: the validator says "The room host decides this" and the screen shows the room's value read-only.

## 5. Writing rules for world authors

### 5.1 Style guide

| Do | Do not |
|---|---|
| "Shuffle the mine" / "The mine's rooms are mixed into the rest of the world." | "Mine region randomization" / "Adds mine nodes to the rewire pool." |
| "Number of crown pieces needed" with an example "Example: 5 of 8" | "Crown piece threshold (int)" |
| Say the unit: "Hints you start with (0 to 12)" | A bare slider from 0 to 99 with no unit |
| Positive values: `on`, `off`; `none`, `a_few`, `lots` | "Disable non-shuffled keys: false" |
| Name the effect on the player in "why change it": "Turn on if you know the ledge clip and want faster routes." | "Enables trick-gated edges." |
| Put community names in `aliases` | Put "sanity" names in the label |
| Use the common key for a common concept (`traps`, `key_shuffle`) | A world-specific `trap_mode` that means the same thing |
| One idea per option | One choice that mixes two ideas ("hard_with_traps") |

### 5.2 Lint rules (enforced by `cgm check` and at publish)

| Code | Rule | Severity |
|---|---|---|
| `settings.label_too_long` | Label over 32 characters in `en` (scaled per language) | error |
| `settings.what_missing` | No "what it does" string | error |
| `settings.what_too_long` | "What it does" over 120 characters or more than one sentence | warning |
| `settings.example_missing` | No example string for range, set and record options | warning |
| `settings.jargon` | Label contains a jargon-list term (5.3) | error |
| `settings.jargon_unexplained` | "What" or "why" uses a jargon term without its plain meaning in the same sentence | warning |
| `settings.alias_missing` | Option matches a known community concept (alias registry) but lists no alias | warning |
| `settings.translation_missing` | A string key absent in a shipped language (falls back to `en`, doc 01 D2.4) | warning; error for `en` |
| `settings.reserved_key` | World option uses a common vocabulary key | error |
| `settings.duplicate_concept` | World option looks like a common concept (same values and alias) but does not use the common key | warning |
| `settings.negative_label` | Label or value starts with "disable", "no", "non" or uses a double negative | warning |
| `settings.unit_missing` | Range with no unit in label or example | warning |
| `settings.default_unsafe` | Default value is marked `experimental` (may give unbeatable games, doc 14 section 6.2) | error |

### 5.3 Jargon list (seed list, maintained by CRFC)
progression, balancing, accessibility (in the "reachable" sense), sphere, logic (alone), pool, filler, local, non-local, plando, sanity (as a suffix), ER, rewire, fill, slot, seed (alone, outside "seed phrase"), RNG, weight, trigger, meta. Each entry carries a suggested plain replacement, which the lint message shows.

### 5.4 Review at publish time
1. Automated: the lint rules above, and a readability estimate on `en` strings (the formula is open, question 4; English-only formulas are a known limit).
2. A plain-language preview: the registry renders the settings screen for the reviewer exactly as players will see it, in each shipped language.
3. Human review for worlds that ask for a "featured" listing: one reviewer who did not write the world reads only the labels and help text, then explains each option back; any option they get wrong goes back to the author (same idea as doc 07 section 8.2 item 7).
4. After publish: a "This is confusing" link on each option feeds a per-option counter visible to the author (no free text in v1, to avoid a moderation surface).

## 6. Import and export with the existing project's player files

### 6.1 Mapping table (by meaning; value names **not verified** unless stated)

| Legacy option | Our key | How values map | Warning shown on import |
|---|---|---|---|
| Progression Balancing (0 to 99, stops 0, 50, 99, verified via the check in section 0) | `keep_moving` | 0 to `off`; 1 to 74 to `gentle`; 75 to 99 to `strong` (our cut points, design value); export writes 0, 50, 99 | "Your file used a balancing number of 30. We use three levels and picked Gentle, the closest." (only when not 0, 50 or 99) |
| Accessibility | `must_reach` | By meaning: "all locations" to `everything`, "items only" to `all_items`, "minimal" to `just_the_goal` | Shown only when a value is unrecognized |
| Local Items / Non-local Items | `keep_home` / `send_away` | Names resolved through the world's name table and aliases | "These names are not items in this game: ..." |
| Start Inventory / Start Inventory from Pool | `start_with` | Counts kept; the second sets `take_from_shuffle: true` | "Both start lists were set; we merged them." when both appear |
| Start Hints / Start Location Hints | `reveal_items` / `reveal_places` | Names resolved | Room may have reveals off: "This room does not allow starting reveals." |
| Exclude Locations / Priority Locations | `no_important_at` / `important_at` | Names resolved; a name in both is an error | "A place cannot be both 'never important' and 'important'." |
| Item Links | `shared_items` (room) | Moved to a list of suggestions for the host | "Sharing items is set by the room host. We passed your request to them." |
| Plando Items | `hand_placed_items` | Simple item-to-place entries map; cross-world entries go to the host | "Some hand placements name other players' games; the host decides those." |
| Plando Connections | `hand_connected_doors` | Entrance names resolved; direction flag mapped | "This door name is not in this game: ..." |
| Death Link | `shared_deaths` | On or off; becomes the player's opt-in | "The host decides whether shared deaths are allowed in this room." |
| Weighted values on any option | `surprise` | One nonzero weight: a plain value. Two or more: `chances`, percentages by largest-remainder rounding (doc 07 `weighted_filler` rounding rule); equal weights: `one_of` | "We turned weights 5, 3, 2 into chances 50, 30, 20." and, if rounding changed anything, "Rounded to whole percents." |
| A game's own options | world option keys | By option key; aliases recorded by the world author | Per-world, from the world's import table |

### 6.2 What cannot be mapped
- **Files that choose among several games by weight** (from memory, **not verified** that this form is current): we ask the player to pick one game per file.
- **Triggers and layered override files** (our notes record a `meta.yaml` that overrides player files and triggers that add layers, inferred, notes section "Generation"): no equivalent; the importer lists them and drops them. Room-wide overrides become a host room settings file.
- **Range keywords** for random numbers: mapped to `anywhere_between` only where the meaning is clear; otherwise left at the default with a warning.
- **Options a world in the sidecar has but no import table for**: kept verbatim in a `legacy:` block that only the sidecar reads (doc 01 section 3.8.2), shown on screen as "Settings for the older generator, not shown here".
- **Exact balancing behavior**: our levels are not the legacy numbers (section 4.1).

### 6.3 Export
Export writes a legacy-shaped YAML for worlds that run in the sidecar: chances become weights equal to the percentages, `one_of` becomes weight 1 each, room-level keys are omitted with a comment telling the host to set them on their side. A comment block at the top of the exported file states our tool wrote it, in our own words.

## 7. Validation messages and accessibility

### 7.1 Message rules
Every message has an error key (doc 01 D1.7), says what is wrong in the player's terms, points to the option by its label, and says how to fix it. No stack traces, no YAML paths by default ("Show details" reveals file, line and column).

| Key | Example message |
|---|---|
| `settings.unknown_option` | "Lanternfall has no setting called 'shufle_mine'. Did you mean 'Shuffle the mine'?" |
| `settings.bad_value` | "'Traps' can be None, A few or Lots. Your file says 'many'." |
| `settings.out_of_range` | "'Hints you start with' goes from 0 to 12. Your file says 20." |
| `settings.chances_total` | "The chances for 'Traps' add up to 90. They need to add up to 100 (10 more)." |
| `settings.both_set` | "'Goal' is set and also marked Surprise me. Pick one." |
| `settings.room_decides` | "'Shared deaths' is decided by the room host. The host has it off." |
| `settings.conflict` | "'Shuffle the mine' needs 'Where keys can be' set to Own game or Anywhere." (the world's `requires` message key, doc 07 section 2.3) |
| `settings.unknown_name` | "'Lantren' is not an item in Lanternfall. Closest: 'Lantern'." |
| `settings.game_version` | "This file is for Lanternfall 0.1. The room runs 0.2, which renamed one setting; we updated it for you." |
| `settings.surprise_off` | "This room does not allow Surprise me. Choose a value for 'Goal' and 'Traps'." |

Unknown options are errors, not warnings with a silent fallback (doc 01 D2.3 records that the legacy behavior logs a warning and uses the default, from the notes, inferred line). Renamed options are migrated through world `alias` entries (doc 07 section 4.6) and the player is told.

### 7.2 Accessibility of the settings screen
- **Screen readers**: every control has a programmatic label from the label string and a description from "what it does"; groups are `fieldset` with `legend`; sliders also expose their named stops as a radio group; errors are linked to their field and announced through one polite live region with a count ("2 settings need attention").
- **Keyboard**: every control reachable in reading order; visible focus; Enter on a group heading collapses it; the search box (labels and aliases) is reachable by a shortcut shown on screen; the "Surprise me" control is a real button with a pressed state.
- **Color and contrast**: state never by color alone (icons and text for "from preset", "changed", "surprise", "error"); contrast to WCAG 2.2 AA (from memory, **not verified** which success criteria apply to each control; check the spec before build).
- **Motion and size**: no animation needed to understand state; reduced-motion respected; touch targets sized for phones (doc 09 section 4.1 assumes phone joins).
- **Language**: the screen sets the page language per string bundle, so a screen reader pronounces translated labels correctly.
- **Testing**: automated checks in CI plus a manual pass with at least one screen reader on desktop and one on a phone before the first public release (design value).

## 8. Proposed changes, decisions, open questions and risks

### 8.1 Proposed changes to other docs (not made here)
- **Doc 01 D2.3**: add to every option `what` (required string key), `why` (optional), `example` (required for range, set and record kinds), `aliases` (list), `scope` (from doc 14), and `decided_by: player|room` for common keys; add the record kind for `start_with` and hand placement (doc 07 already lists "small structured record"); reserve the common vocabulary keys. Replace the `"random": {"on": 1, "off": 3}` example with the `surprise` forms of section 3.
- **Doc 01 D3.1**: the difference is taken against the preset when one is named; surprise specs are encoded as specs; cosmetic options are excluded (doc 14 already proposes this); the settings hash of section 2.1 is a named field.
- **Doc 01 D3.3**: mystery settings use `surprise`; the room flag `allow_surprise`.
- **Doc 07 section 3.1 and 4.2**: world `options` gain `what`, `why`, `example`, `aliases` string keys and the lint rules of section 5.2; the strict YAML profile applies to settings files too; `presets/` folder in the package layout.
- **Doc 09**: `set_options` carries canonical settings JSON; `lock_options` records the settings hash per seat; the host console has a room settings page owning the room-level keys of section 4; a seat card shows "surprise options" without results under mystery rooms.
- **GLOSSARY**: add settings file, room settings file, common vocabulary, surprise (and `one_of`, `chances`, `anywhere_between`), keep everyone moving, preset, settings hash, decided by (player, room, game).

### 8.2 New decision rows (ids TBD)

| Id | Decision | Status | Rationale | Docs affected | Blocks |
|---|---|---|---|---|---|
| D-TBD-S1 | One settings file format (`settings/1`) for players and `room-settings/1` for hosts, strict YAML authoring, canonical JSON with a settings hash | proposed | One parser, one hash, same rules as worlds (doc 07) | 01, 07, 09 | Settings screen build |
| D-TBD-S2 | A common vocabulary of reserved keys with platform-owned labels and help (section 4) | proposed | Same words in every game | 01, GLOSSARY | World authoring guide |
| D-TBD-S3 | "Keep everyone moving" with three named levels replaces the 0 to 99 slider in our UI; nearest-level import | **open** (owner) | Predictable for players; loses fine control | 01 D2.7, CFill | CFill balancing vectors |
| D-TBD-S4 | `surprise` with `one_of`, `chances` (sum 100), `anywhere_between` replaces bare weights | proposed | The numbers are the odds | 01 D2.3, D3.3 | Settings string spec |
| D-TBD-S5 | Item links, shared deaths, release and collect, hint price and spoilers are room-level; players opt in | **open** (owner) | They affect other seats | 09, 01 | Lobby build |
| D-TBD-S6 | Plain-language lint rules and the jargon list are publish gates (errors block, warnings show) | proposed | Quality at the source | 07, registry | Registry publish |
| D-TBD-S7 | Unknown options and room-level keys in a player file are errors with plain messages, never a silent default | proposed | A typo must not change a game silently | 01 D2.3 | Validator |
| D-TBD-S8 | Legacy import and export tables per world, with a `legacy:` passthrough block for sidecar worlds | proposed | Files keep working in both directions | 01 section 3.8 | Compat milestone after D-100 |

### 8.3 Open questions for the owner (recommended default in brackets)
1. Is "Keep everyone moving" a player or room setting? [player preference, room may lock it, because a host running a race may want one value for all]
2. Three levels or keep a numeric "advanced" field for experts? [three levels only; the import warning covers the rest]
3. Label length limits per language: fixed scale factors or per-language limits set by translators? [per language, by translators]
4. Which readability estimate, and does it block or only warn? [warn only; no formula is reliable across languages]
5. Separate skill presets ("what I can do") combined with a normal preset? [yes, one of each at most]
6. Should a player file be allowed to name several games? [no; one file, one game]

### 8.4 Risks

| Risk | Why it matters | Mitigation |
|---|---|---|
| Experienced players dislike new names | They know the old terms well and search by them | Aliases in search and under labels; legacy import keeps their files working |
| Three balancing levels hide a real difference players relied on | Some groups tune the number carefully (not verified how many) | Import warning names the change; owner question 2 |
| Plain labels drift from what the generator does | A friendly label that is wrong is worse than jargon | Platform owns common strings; each level's text is reviewed against its CFill vector |
| Moving options to the room breaks expectations | A player file that used to work now gets "the host decides" | Clear message, value forwarded to the host as a suggestion |
| Reserved keys collide with future world needs | A world wants its own `traps` meaning | `settings.duplicate_concept` lint and CRFC for new common keys |
| Mapping claims about the legacy project go stale | Their options change | Date-stamped mapping, recheck before anything is published (`CLAUDE.md`) |
| Lint becomes busywork for authors | Friction loses authors (doc 07 section 8.1) | Few errors, most rules warnings, auto-fix suggestions in the language server |
