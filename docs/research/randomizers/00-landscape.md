# 00. Landscape of open-source video game randomizers (discovery and selection)

> Internal research note, 2026-11-16. Not reviewed by a different agent or a human. Nothing here goes outside this repo before review under [`CLAUDE.md`](../../../CLAUDE.md). This file is our own writing with links; no code, text or data from any project is reproduced. Every number is labeled **counted** (read from the GitHub API or a file listing on 2026-11-16), **estimated**, or **not verified**. Claims marked **(mem)** come from memory and were not verified in this step.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. Purpose and method

Goal of this step: find the larger and more influential randomizer projects, record basic facts, and pick 12 to 16 for a deep audit of how they represent logic, fill items, store data, ship tooling and integrate with games. The audit will feed the world format in [`../../design/07-data-only-world-format.md`](../../design/07-data-only-world-format.md) and the standard in [`../../design/01-standards.md`](../../design/01-standards.md).

What was done, all on 2026-11-16:

1. **Community index.** Located the community-maintained "BIG List of Video Game Randomizers" (site `randomizers.debigare.com`, source repo [video-game-randomizers/rando-list](https://github.com/video-game-randomizers/rando-list), MIT, 87 stars counted). Its data is one YAML file per series under `src/series/`. I downloaded 17 series files through the GitHub contents API (Zelda, Metroid, Pokemon, Final Fantasy, Kingdom Hearts, Dark Souls, Elden Ring, Mario, Castlevania, Mega Man, Celeste, Doom, Hollow Knight, Ori, Chrono, Donkey Kong, Banjo-Kazooie) and extracted 155 distinct GitHub repo links (counted). The site itself was not opened in a browser.
2. **Seed list.** Started from [HarbourMasters/Shipwright](https://github.com/HarbourMasters/Shipwright) and added about 65 repos I knew of or found through `gh search repos` queries (randomizer, rando, multiworld, item randomizer, randomizer tracker, emotracker, poptracker pack, and per-franchise queries). Generic search terms are noisy (most top hits are unrelated random-number libraries), so per-franchise queries plus the community list did most of the work. The search API rate limit was hit once; remaining lookups used the core API.
3. **Metadata.** Stars, last push, primary language, license (SPDX id as GitHub reports it), archived and fork flags came from the GitHub GraphQL and REST APIs. Contributor counts (REST `contributors?anon=1`, last page number) were collected only for the selected projects. Where GitHub reports `NOASSERTION`, I opened the license file and say what it contains.
4. **Web checks.** Perplexity and SearXNG searches to find the Free Enterprise source and the Ship of Harkinian license status. One Perplexity answer named a Free Enterprise repo that returns 404 on GitHub; I discarded it and used a SearXNG hit that resolves ([HungryTenor/FreeEnterprise4](https://github.com/HungryTenor/FreeEnterprise4)).
5. **Layout for selection.** For each selected project I fetched the full recursive git tree (`gh api repos/<repo>/git/trees/<branch>?recursive=1`, none truncated) and grouped paths by directory. A handful of files were opened through the contents API to confirm the logic format (cited in section 5). Nothing was cloned or built.
6. **Archipelago cross-check.** To say which projects feed Archipelago, I listed `worlds/` in our local checkout of the Archipelago repo at commit `4f9545f7` (their code, read only).

Limits: stars measure visibility on GitHub, not player count. Several large communities live on Discord, their own sites, or non-GitHub hosts, so their star counts understate them. Last push is the latest push to any branch, not the latest release.

## 2. Candidate table

Stars and last push are **counted** on 2026-11-16. "MW" is multiworld or co-op support. "AP in-tree" means a world of that name exists in Archipelago `worlds/` at `4f9545f7`; "AP external" means an outside Archipelago world or fork exists.

| # | Project | Repo | Stars | Last push | Lang | License | Category | Integration | MW / co-op | Why it matters to us |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Ship of Harkinian | [HarbourMasters/Shipwright](https://github.com/HarbourMasters/Shipwright) | 6077 | 2026-11-16 | C, C++ | none at top level; rando subfolder carries an MIT notice for OoTR-derived code | PC port with built-in rando | native port | online co-op ("Anchor", `soh/soh/Network/Anchor/`, 29 files counted); AP external ([HarbourMasters/Archipelago-SoH](https://github.com/HarbourMasters/Archipelago-SoH), 70 stars) | Most-starred project in the space by far; logic written as C++ code inside the game itself |
| 2 | 2Ship2Harkinian | [2ship2harkinian/2ship2harkinian](https://github.com/2ship2harkinian/2ship2harkinian) | 2477 | 2026-11-16 | C, C++ | CC0-1.0 | PC port with built-in rando | native port | not verified | Second port-plus-rando; `mm/2s2h/Rando/` is a newer, smaller design to compare with SoH |
| 3 | OoT Randomizer | [OoTRandomizer/OoT-Randomizer](https://github.com/OoTRandomizer/OoT-Randomizer) | 443 | 2026-11-16 | Python | MIT, with GPLv2+ subfolders (LICENSE opened) | item, entrance | ROM patch plus custom ASM | built-in multiworld (mem); companion app [midoshouse/ootr-multiworld](https://github.com/midoshouse/ootr-multiworld); AP in-tree `oot` | De facto OoT standard; JSON region files with rule expression strings |
| 4 | OoT3D Randomizer | [gamestabled/OoT3D_Randomizer](https://github.com/gamestabled/OoT3D_Randomizer) | 232 | 2026-11-03 | C++ | MIT notice for OoTR-derived code (LICENSE.md opened) | item | 3DS code patch | not verified | Likely source of SoH's `3drando` folder (inferred from the folder name and the shared notice) |
| 5 | OoTMM (OoT + MM combo) | [OoTMM/OoTMM](https://github.com/OoTMM/OoTMM) | 152 | 2026-11-16 | C payload, TypeScript generator | MIT | combo item, entrance | ROM patch | built-in multiworld (`packages/generator/src/common/multi/`, `doc/Multi-Packets.txt`) | Two games in one seed, YAML world data, TypeScript solver |
| 6 | Majora's Mask Randomizer (MMR) | [ZoeyZolotova/mm-rando](https://github.com/ZoeyZolotova/mm-rando) | 200 | 2026-11-11 | C# | GPL-3.0 | item, music, enemy | ROM patch | not verified | Active fork of [az64/mm-rando](https://github.com/az64/mm-rando) (39 stars, last push 2022-12-08) |
| 7 | Wind Waker Randomizer | [LagoLunatic/wwrando](https://github.com/LagoLunatic/wwrando) | 427 | 2026-10-22 | Python | MIT | item, entrance, enemy | disc image patch | none built in (mem); AP in-tree `tww` docs reference it | Plain-text logic with named macros; compact, readable codebase |
| 8 | TWW HD Randomizer | [SuperDude88/TWWHD-Randomizer](https://github.com/SuperDude88/TWWHD-Randomizer) | 75 | 2026-11-07 | C++ | MIT | item | Wii U game file patch | not verified | Same game in C++, useful contrast to the Python original |
| 9 | Twilight Princess Rando | [zsrtp/Randomizer](https://github.com/zsrtp/Randomizer) | 63 | 2026-10-08 | C++ | GPL-3.0 | item | runtime code on GameCube/Wii (mem) | not verified | Generator is a separate web project (mem) |
| 10 | Skyward Sword Rando | [ssrando/ssrando](https://github.com/ssrando/ssrando) | 46 | 2026-11-01 | Python | MIT | item, entrance | Wii image patch | AP external (an `SS_APWorld` repo appeared in search) | Strong tracker culture around it |
| 11 | SSHD Rando | [mint-choc-chip-skyblade/sshd-rando](https://github.com/mint-choc-chip-skyblade/sshd-rando) | 51 | 2026-10-13 | Python | AGPL-3.0 | item | Switch mod (mem) | not verified | AGPL; note for anything we host |
| 12 | Minish Cap Rando | [minishmaker/randomizer](https://github.com/minishmaker/randomizer) | 145 | 2025-11-29 | C# | GPL-3.0 | item | ROM patch | not verified | Custom logic text format (mem) |
| 13 | LADXR | [daid/LADXR](https://github.com/daid/LADXR) | 61 | 2026-11-04 | Python | MIT | item, entrance, map generation | ROM patch | built-in multiworld (`patches/multiworld.py`, `www/multiworld.html`) | Vendored inside Archipelago (`worlds/ladx/LADXR/`) |
| 14 | ALttP Randomizer (alttpr.com) | [sporchia/alttp_vt_randomizer](https://github.com/sporchia/alttp_vt_randomizer) | 320 | 2024-07-20 | PHP | MIT | item, plus enemizer and entrance hooks | ROM patch; also the web seed host | none built in (mem); AP in-tree `alttp` | Web app and generator in one Laravel codebase; class-per-region logic |
| 15 | ALttP Door Randomizer | [aerinon/ALttPDoorRandomizer](https://github.com/aerinon/ALttPDoorRandomizer) | 58 | 2026-11-16 | Python | MIT | door and entrance | ROM patch | not verified | Door shuffle is a hard graph problem |
| 16 | z3randomizer base patch | [KatDevsGames/z3randomizer](https://github.com/KatDevsGames/z3randomizer) | 52 | 2026-10-22 | Assembly | MIT | shared base ROM patch | ROM patch | n/a | One base patch serving several ALttP generators (mem) |
| 17 | ALBW Randomizer (z17) | [rickfay/z17-randomizer](https://github.com/rickfay/z17-randomizer) | 47 | 2026-10-13 | Rust | GPL-2.0 | item | 3DS file and code patch | not verified | A Rust generator with logic as Rust code |
| 18 | Oracles Randomizer | [jangler/oracles-randomizer](https://github.com/jangler/oracles-randomizer) | 81 | 2025-12-28, archived | none reported | none | item | ROM patch | not verified | Archived; a sign of single-maintainer risk |
| 19 | Randovania | [randovania/randovania](https://github.com/randovania/randovania) | 221 | 2026-11-16 | Python | GPL-3.0 | multi-game framework, logic editor | per-game exporters (ROM, ISO, mod) | built-in multiworld server (`randovania/server/`, `network_client/`) | 13 game folders plus a `blank` template (counted); GUI data editor |
| 20 | open-prime-rando | [randovania/open-prime-rando](https://github.com/randovania/open-prime-rando) | 6 | 2026-11-10 | Python | GPL-3.0 | patcher backend | game file patch | n/a | Split between generator and per-game patcher |
| 21 | VARIA | [theonlydude/RandomMetroidSolver](https://github.com/theonlydude/RandomMetroidSolver) | 58 | 2026-08-05 | Python | MIT | item, area, boss; solver; tracker | ROM patch; web | none built in (mem); vendored in Archipelago (`worlds/sm/variaRandomizer/`) | Solver-first design with difficulty ratings |
| 22 | SMZ3 (Tracker Council) | [TheTrackerCouncil/SMZ3Randomizer](https://github.com/TheTrackerCouncil/SMZ3Randomizer) | 39 | 2026-10-06 | C# | MIT | combo item | ROM patch | multiplayer client and server projects in tree; AP in-tree `smz3` | Original at [tewtal/SMZ3Randomizer](https://github.com/tewtal/SMZ3Randomizer) (82 stars, 2024-11-11) |
| 23 | Super Metroid Item Rando | [Dessyreqt/smrandomizer](https://github.com/Dessyreqt/smrandomizer) | 60 | 2024-09-16 | C# | none | item | ROM patch | not verified | Early, widely forked (mem) |
| 24 | Hollow Knight RandomizerMod | [homothetyhk/RandomizerMod](https://github.com/homothetyhk/RandomizerMod) | 75 | 2026-04-12 | C# | LGPL-2.1 | item, entrance | PC mod | separate multiworld mod ([Shadudev/HollowKnight.MultiWorld](https://github.com/Shadudev/HollowKnight.MultiWorld), 16 stars); AP in-tree `hk` | JSON logic over a reusable core |
| 25 | RandomizerCore (+ .Json) | [homothetyhk/RandomizerCore](https://github.com/homothetyhk/RandomizerCore), [RandomizerCore.Json](https://github.com/homothetyhk/RandomizerCore.Json) | 8, 2 | 2026-06-19, 2026-01-11 | C# | LGPL-2.1 | reusable core library | n/a | n/a | The clearest game-agnostic randomizer library found |
| 26 | Ori WotW seed generator | [ori-community/wotw-seedgen](https://github.com/ori-community/wotw-seedgen) | 5 | 2026-11-11 | Rust | no LICENSE file in tree, but `Cargo.toml` declares MIT (see note 07; owner or lawyer decision needed) | item, doors | PC mod via [wotw-rando-client](https://github.com/ori-community/wotw-rando-client) (23 stars, MIT) | multiplayer and bingo (mem) | Rust generator with its own logic language and tokenizer |
| 27 | SoulsRandomizers (DS1, DS3, Sekiro, Elden Ring) | [thefifthmatt/SoulsRandomizers](https://github.com/thefifthmatt/SoulsRandomizers) | 94 | 2026-10-27 | C# | custom source-available (LICENSE.md opened) | item and enemy | PC game file rewrite | none (mem); AP in-tree `dark_souls_3` is separate | Enemy randomization at scale; restrictive license |
| 28 | Dark Souls Item Randomizer | [HotPocketRemix/DarkSoulsItemRandomizer](https://github.com/HotPocketRemix/DarkSoulsItemRandomizer) | 86 | 2021-06-08 | Python | none | item | PC game file rewrite | no (mem) | Early Souls randomizer |
| 29 | Universal Pokemon Randomizer ZX | [Ajarmar/universal-pokemon-randomizer-zx](https://github.com/Ajarmar/universal-pokemon-randomizer-zx) | 881 | 2024-11-23 | Java | GPL-3.0 | content and stat | ROM rewrite | no | Most-starred content randomizer; fork of archived [Dabomstew/universal-pokemon-randomizer](https://github.com/Dabomstew/universal-pokemon-randomizer) (388) |
| 30 | Worlds Collide (FF6) | [AtmaTek/WorldsCollide](https://github.com/AtmaTek/WorldsCollide), active fork [ff6wc/WorldsCollide](https://github.com/ff6wc/WorldsCollide) | 13, 10 | 2024-01-27, 2026-10-23 | Python | MIT | objective-driven open world, content | ROM patch | AP external (forks named WorldsCollide-AP appeared in a search answer; not opened) | Objectives as conditions and results, not item logic alone |
| 31 | Free Enterprise 4 (FF4) | [HungryTenor/FreeEnterprise4](https://github.com/HungryTenor/FreeEnterprise4) | 12 | 2024-01-27 | JavaScript (primary as reported) | MIT | open world, key items, characters | ROM patch | AP external (an AP wiki page appeared in search; not opened) | Large community, small GitHub footprint |
| 32 | FF1 Randomizer (FFR) | [FiendsOfTheElements/FF1Randomizer](https://github.com/FiendsOfTheElements/FF1Randomizer) | 91 | 2026-10-12 | C# | GPL-3.0 | item and content | ROM patch | AP in-tree `ff1` | Very wide flag set (mem) |
| 33 | Beyond Chaos (FF6) | [abyssonym/beyondchaos](https://github.com/abyssonym/beyondchaos) | 33 | 2023-05-14 | Python | none | content | ROM patch | no | Pure chaos content randomizer, no item logic (mem) |
| 34 | KH2 Randomizer | [tommadness/KH2Randomizer](https://github.com/tommadness/KH2Randomizer) | 35 | 2026-09-09 | Python | Apache-2.0 | item, stat, content | PC mod via OpenKH tools (mem) | `Module/multiworld.py` in tree; AP in-tree `kh2` | Seed evaluation and weighting modules |
| 35 | OpenKH | [OpenKH/OpenKh](https://github.com/OpenKH/OpenKh) | 466 | 2026-11-15 | C# | Apache-2.0 | modding toolchain | mod manager | n/a | Shows how a mod platform carries randomizer output (mem) |
| 36 | SotN Randomizer | [sotnrando/sotnrando](https://github.com/sotnrando/sotnrando) | 106 | 2026-11-10 | JavaScript | WTFPL | item, content | disc image patch, runs in browser (mem) | not verified | Browser-side generation |
| 37 | DSVania Randomizer | [LagoLunatic/dsvrandom](https://github.com/LagoLunatic/dsvrandom) | 96 | 2024-10-29 | Ruby | MIT | item, enemy, map | ROM patch | no (mem) | Three games from one tool |
| 38 | Mega Man 2 Randomizer | [duckfist/MM2Random](https://github.com/duckfist/MM2Random) | 30 | 2022-12-08 | C# | none | stage, weapon, content | ROM patch | no; AP in-tree `mm2` is a separate design (mem) | Non-item randomization in a linear game |
| 39 | Level-Headed (SMB1) | [Coolcord/Level-Headed](https://github.com/Coolcord/Level-Headed) | 132 | 2024-10-13 | C++ | GPL-3.0 | procedural level generator | ROM patch | no | Generation, not shuffling |
| 40 | TTYD Infinite Pit | [jdaster64/ttyd-infinite-pit](https://github.com/jdaster64/ttyd-infinite-pit) | 63 | 2025-03-03 | C++ | GPL-3.0 | content (roguelike mode) | runtime code mod (mem) | no | Roguelike mode as a randomizer |
| 41 | SM64 Randomizer | [andrebarks/sm64-randomizer](https://github.com/andrebarks/sm64-randomizer) | 59 | 2022-10-28, archived | Python | MIT | content, warps | ROM patch | no | Archived; AP in-tree `sm64ex` targets a PC port instead |
| 42 | Celeste Randomizer | [rhelmot/CelesteRandomizer](https://github.com/rhelmot/CelesteRandomizer) | 48 | 2026-08-18 | C# | MIT | room and map generation | PC mod (mem) | not verified | Map generation from room pieces |
| 43 | Tunic Randomizer | [silent-destroyer/tunic-randomizer](https://github.com/silent-destroyer/tunic-randomizer) | 50 | 2026-11-15 | C# | MIT | item, entrance, enemy | PC mod (mem) | AP in-tree `tunic` | A mod that plays standalone and with AP |
| 44 | Banjo-Tooie AP | [jjjj12212/Archipelago-BanjoTooie](https://github.com/jjjj12212/Archipelago-BanjoTooie) | 31 | 2026-10-05 | Python | MIT | AP world plus emulator side | emulator Lua and memory (mem) | AP external | Example of an AP-only randomizer |
| 45 | PopTracker | [black-sliver/PopTracker](https://github.com/black-sliver/PopTracker) | 200 | 2026-11-14 | C++ | GPL-3.0 | tracker, auto-tracker | reads game state via usb2snes and UAT (`src/usb2snes/`, `src/uat/`) | AP client (mem) | Pack format: JSON data plus Lua rules, with a JSON schema |
| 46 | EmoTracker (community) | [EmoTracker-Community/EmoTracker](https://github.com/EmoTracker-Community/EmoTracker) | 17 | 2026-10-16 | C# | MIT | tracker | auto-tracking (mem) | n/a | Older pack format PopTracker is compatible with (mem) |
| 47 | OpenTracker | [trippsc2/OpenTracker](https://github.com/trippsc2/OpenTracker) | 40 | 2026-02-19 | C# | MIT | tracker (ALttP) | auto-tracking (mem) | n/a | Tracker that re-encodes generator logic |
| 48 | racetime.gg | [racetimeGG/racetime-app](https://github.com/racetimeGG/racetime-app) | 48 | 2026-02-23 | Python | GPL-3.0 | race and seed-rolling host | web | n/a | Race rooms with bots that roll seeds |
| 49 | SahasrahBot | [tcprescott/sahasrahbot](https://github.com/tcprescott/sahasrahbot) | 28 | 2026-07-07 | Python | MIT | seed-rolling bot | Discord and race bot | n/a | Calls several generators' web APIs (mem) |
| 50 | Mido's House | [midoshouse/midos.house](https://github.com/midoshouse/midos.house) | 1 | 2026-11-16 | Rust | MIT | event and seed host (OoTR) | web | n/a | Rust web service for a rando community |

Also seen and not tabled: [cdhowie/cheese-trackers](https://github.com/cdhowie/cheese-trackers) (28 stars, Rust, AGPL-3.0 per LICENSE opened, async room tracker for Archipelago), [MultiworldGG/MultiworldGG](https://github.com/MultiworldGG/MultiworldGG) (30 stars, GPL-3.0, Python; relationship to Archipelago not verified), [StripesOO7/poptracker-pack-builder](https://github.com/StripesOO7/poptracker-pack-builder) (17 stars, MIT), and Doom: Archipelago has `doom_1993` and `doom_ii` in-tree, and the only dedicated Doom generator with stars found was the archived level generator [caligari87/ObAddon](https://github.com/caligari87/ObAddon) (81 stars, GPL-3.0). No standalone Doom item randomizer with a notable GitHub presence was found (not exhaustive).

## 3. Landscape analysis

### 3.1 Clusters
- **Zelda N64 lineage (the largest cluster).** OoT Randomizer, then OoT3D Randomizer, then SoH's built-in randomizer (the 3DS code carries the same MIT notice), plus OoTMM, MMR and 2Ship2Harkinian. Rows 1 to 6 hold 9,581 of the table's stars (counted sum), most of it from the two ports.
- **SNES ROM-patch communities.** ALttPR, VARIA, SMZ3, Free Enterprise, Worlds Collide, FFR. Mature, racing-driven, generators with web front ends.
- **PC mods.** Hollow Knight, Souls games, Celeste, Tunic, KH2 via OpenKH. The game is modified at runtime or its files are rewritten; no ROM.
- **Multi-game frameworks.** Randovania (13 games in one generator) and Archipelago (excluded here). MultiworldGG appears to be a related distribution (not verified).
- **Content randomizers.** UPR ZX, Beyond Chaos, Level-Headed, Infinite Pit. No reachability logic, or very little; the value is in game-data editing tables.
- **Tooling.** Trackers (PopTracker, EmoTracker, OpenTracker, cheese-trackers) and race and seed hosts (racetime.gg, SahasrahBot, Mido's House, alttpr.com itself).

### 3.2 Languages (counted over the 50 rows, primary language as reported, two-language rows counted once by generator language)
Python 17, C# 14, C or C++ 8, JavaScript or TypeScript 3, Rust 3 (z17, wotw-seedgen, Mido's House), Java 1, PHP 1, Ruby 1, Assembly 1, none reported 1. Python and C# dominate generators; C and C++ dominate ports and runtime mods. Rust appears in generators (z17, wotw-seedgen) and community infrastructure (ootr-multiworld, Mido's House, cheese-trackers), which supports our Rust choice without implying anyone else's code is reusable.

### 3.3 Licenses (counted over table rows)
MIT is the most common (23 rows, counting OoT3D's MIT notice), then GPL-3.0 (11), then no license (7 rows: Shipwright top level, oracles, smrandomizer, DS1 item rando, Beyond Chaos, MM2Random, wotw-seedgen, which declares MIT only in `Cargo.toml`), then LGPL-2.1 (2), Apache-2.0 (2), and one each of CC0-1.0, WTFPL, GPL-2.0, AGPL-3.0 and a custom source-available license. "No license" means all rights reserved by default, which still lets us read public code but not reuse it. Our rule is the same for every row: read and describe, never copy.

### 3.4 Integration methods
ROM or disc patch is the majority (about 28 rows). PC mod or file rewrite next (about 9). Native port (2, the two most-starred). Runtime code injection on console (TP, Infinite Pit, OoT3D-style patching, about 3). Memory reading over USB or emulator bridges is the trackers' method. Patch packaging is a solved, repeated problem (IPS, BPS, bsdiff, xdelta and custom formats appear across projects; formats not verified per project), and every project re-implements it.

### 3.5 De facto standards in their communities
- OoT Randomizer for OoT (its logic lineage reaches OoT3D and SoH).
- ALttPR (alttpr.com) for ALttP, with z3randomizer as the shared base patch (mem).
- VARIA for Super Metroid (vendored in Archipelago).
- Randovania for the Metroid Prime series and several 2D Metroids.
- RandomizerMod for Hollow Knight.
- UPR ZX for Pokemon content randomization.
- Worlds Collide and Free Enterprise for FF6 and FF4 (mem, based on community size, not counted).
- PopTracker for trackers, with EmoTracker pack compatibility (mem).
- racetime.gg for races.

### 3.6 Reusable cores and data-driven logic that already exist
- **Data files with expression strings:** OoTR (JSON regions with boolean expression strings), OoTMM (YAML world, macros and events), RandomizerMod (JSON logic with string expressions over terms, macros and waypoints), wwrando (text logic with macros), PopTracker packs (JSON with Lua-named rules).
- **Own logic language with a parser:** wotw-seedgen (`areas.wotw`, tokenizer, parser and emitter in Rust; separate header language for seed tweaks).
- **Structured graph database with an editor:** Randovania (per-game JSON databases, requirement trees, a GUI data editor and integrity checks).
- **Logic as code:** SoH (C++ per-area files), ALttPR (PHP class per region), z17 (Rust per region), VARIA (Python graph helpers).
- **Game-agnostic library:** RandomizerCore is the only standalone core library found; Randovania's `game_description` and `generator` packages are game-agnostic in design but live inside one app.

## 4. Selection for deep audit (16 projects, 4 batches of 4)

Selection criteria: one strong example of each logic representation (expression strings in data, structured graph DB, custom DSL, code), each fill approach we know of (assumed fill, retcon or backtracking, solver-verified, sphere-based), each integration method (ROM patch, native port, PC mod, file rewrite, tracker memory reading), sizes from 2 to 191 contributors, and licenses from CC0 to restrictive. Contributor counts are **counted** (REST contributors including anonymous, 2026-11-16).

### Batch A: Zelda N64 lineage and the port model
| Project | Contributors | Where to look (from tree listing) | License implications |
|---|---|---|---|
| OoT Randomizer | 136 | `data/World/*.json` and `data/Glitched World/` (26 files each, counted); `RuleParser.py`, `Rules.py`, `RulesCommon.py`; `Fill.py`, `Search.py`, `EntranceShuffle.py`, `Hints.py`, `Plandomizer.py`; `ItemPool.py`, `SettingsList.py`; `ASM/` (patch code); `tests/` (278 files, of which 258 under `tests/plando/`; counted with `git ls-tree -r` at commit `526b4c8d`, 2026-11-16) | MIT plus GPLv2+ subfolders. Read and describe; keep any notice if a future decision ever reuses data |
| Ship of Harkinian randomizer | 191 | `soh/soh/Enhancements/randomizer/` (`location_access/overworld/` 22 files, `location_access/dungeons/` 12, counted), `3drando/fill.cpp`, `item_pool.cpp`, `playthrough.cpp`, `hints.cpp`, `static_data.cpp`, `Shuffle*.cpp`; `soh/soh/Network/Anchor/` for co-op | No top-level license; the `3drando/LICENSE.md` notice covers OoTR-derived code only. Treat everything else as all rights reserved. Read only |
| OoTMM | 33 | `data/world/` (100), `data/checks/` (96), `data/macros/`, `data/defs/`; `packages/logic/src/` (`solver/solve.ts`, `pathfind/`, `world/builder.ts`, `pipeline.ts`); `packages/core/src/settings/`; `packages/generator/src/common/multi/` and `doc/Multi-Packets.txt` | MIT. Read and describe |
| Wind Waker Randomizer | 16 | `logic/item_locations.txt`, `logic/macros.txt`, `logic/logic.py`, `logic/item_types.py`; `randomizers/*.py` (items, entrances, enemies, hints, boss_reqs); `asm/patches/`, `data/`, `options/`, `test/` | MIT. Read and describe |

### Batch B: frameworks, cores and custom logic languages
| Project | Contributors | Where to look | License implications |
|---|---|---|---|
| Randovania | 109 | `randovania/games/<game>/logic_database/*.json` (with `.txt` pretty prints), `randovania/games/blank/` (template game); `game_description/` (`db/`, `requirements/`, `integrity_check.py`, `pretty_print.py`); `generator/filler/` (`retcon.py`, `runner.py`, `weights.py`), `generator/pickup_pool/`; `resolver/`; `gui/data_editor.py`, `gui/dialog/connections_editor/`; `server/`, `network_client/`, `exporter/` | GPL-3.0. Read and describe only; no code or database content in our repo |
| Hollow Knight RandomizerCore + RandomizerMod | 4 + 15 | Core: `RandomizerCore/Logic/`, `StringLogic/`, `StringParsing/`, `Randomization/` (`Randomizer.cs`, `SphereBuilder.cs`, `Validator.cs`, `CoupledRandomizationGroup.cs`), `docs/articles/`; JSON: `RandomizerCore.Json/Converters/`; Mod: `RandomizerMod/Resources/Logic/{terms,macros,state,waypoints,transitions,locations,items}.json`, `Resources/Data/*.json`, `RandomizerMod/RC/`, `LOGIC_README.md` | LGPL-2.1. Read and describe; linking rules do not matter because we do not link |
| Ori WotW seedgen | 12 | `wotw_seedgen/areas.wotw`, `loc_data.csv`, `headers/*.wotwrh`; `src/languages/logic/` (tokenizer, parser, emitter), `src/languages/header/`; `src/generator/` (`placement.rs`, `doors.rs`, `seed.rs`, `spoiler.rs`); `benches/bench.rs` | No LICENSE file in the tree, though `Cargo.toml` declares MIT (note 07): treat as all rights reserved until an owner or lawyer decides. Read for ideas only, describe at a high level |
| LADXR | 28 | `logic/` (15 files), `locations/` (29), `randomizer.py`, `worldSetup.py`, `settings.py`, `spoilerLog.py`; `patches/` including `multiworld.py` and `bank3e.asm/multiworld.asm`; `mapgen/`, `dungeongen/`; `www/` | MIT. Read and describe. Also useful to compare with Archipelago's vendored copy |

### Batch C: ROM-patch generators, solvers and web hosts
| Project | Contributors | Where to look | License implications |
|---|---|---|---|
| ALttP Randomizer (alttpr.com) | 48 | `app/Randomizer.php`, `app/Filler/RandomAssumed.php`, `app/World.php`, `app/Region/**` (51), `app/Location/**` (23), `app/Item/**`, `app/EntranceRandomizer.php`, `app/Enemizer.php`; `config/`; `tests/{NoGlitches,MajorGlitches,OverworldGlitches,Inverted,...}/`; `resources/js/` and `database/migrations/` (seed hosting) | MIT. Read and describe |
| VARIA | 24 | `rando/` (`Filler*.py`, `GraphBuilder.py`, `RandoSetup.py`, `Restrictions.py`, `MiniSolver.py`); `logic/` (`smbool.py`, `smboolmanager.py`, `helpers.py`); `graph/{vanilla,mirror,rotation}/graph_access.py` and `graph_locations.py`; `solver/` (`standardSolver.py`, `interactiveSolver.py`, `randoSolver.py`); `rando_presets/` (46); `tools/autotracker/`; `web/` | MIT. Read and describe |
| Worlds Collide (FF6) | 3 (AtmaTek) + 5 (ff6wc fork) | `wc.py`, `seed.py`; `args/` (31); `objectives/conditions/`, `objectives/results/` (56); `event/` (60); `data/` (91) | MIT. Read and describe. Audit the active fork, cite both |
| ALBW Randomizer (z17) | 5 | `randomizer/src/regions/{hyrule,lorule,dungeons}/`, `randomizer/src/world/*.rs`, `randomizer/src/lib.rs`, `randomizer/src/patch/`, `macros/src/`, `rom/src/`, `presets/`, `docs/` | GPL-2.0. Read and describe only; never paste Rust from it into our Rust code |

### Batch D: content randomizers, PC mods and tooling
| Project | Contributors | Where to look | License implications |
|---|---|---|---|
| Universal Pokemon Randomizer ZX | 18 | `src/com/dabomstew/pkrandom/Randomizer.java`, `romhandlers/` (13), `config/gen*_offsets.ini` and `*.tbl`, `pokemon/`, `constants/`, `cli/CliRandomizer.java`; `asm/` | GPL-3.0. Read and describe; offsets and tables are their data, do not copy |
| SoulsRandomizers | 2 | `RandomizerCommon/Item/` (`Permutation.cs`, `KeyItemsPermutation.cs`, `AnnotationData.cs`, `LocationData.cs`, `README.md`), `RandomizerCommon/Enemy/`, `RandomizerCommon/Elden/`, `RandomizerCommon/Core/`; `diste/Base/` (`itemlocations.txt`, `annotations.txt`, `enemy.txt`, `events.txt`); `dist/`, `dists/`; `CONTRIBUTING.md` | Custom license: viewing and forking the unmodified source is allowed, but derived works, redistribution and hosting it as a service are restricted. Read for design ideas only, describe at a high level, and never host or wrap it |
| KH2 Randomizer | 25 | `Module/` (`newRandomize.py`, `generate.py`, `itemPlacementRestriction.py`, `weighting.py`, `seedEvaluation.py`, `multiworld.py`, `hints.py`), `List/location/` (25), `List/inventory/` (21), `seedtests/`, `tests/` | Apache-2.0. Read and describe |
| PopTracker | 34 | `doc/PACKS.md`, `doc/AUTOTRACKING.md`, `doc/PackCreation.md`; `schema/packs/strict/` (7); `src/core/` (`pack`, `location`, `locationsection`, `rule.h`, `tracker`, `autotracker`, `scripthost`); `src/usb2snes/`, `src/uat/`, `src/packmanager/` | GPL-3.0. Read and describe. Pack compatibility, if ever wanted, is an interface question for a separate decision |

Left out on purpose: SMZ3 (combo design covered by OoTMM), 2Ship2Harkinian (port model covered by SoH; good follow-up), Celeste and Tunic (PC mod model covered by HK and Souls), Free Enterprise (small GitHub footprint; Worlds Collide covers the RPG objective style), MMR and the SS randomizers (same lineage as batch A).

## 5. Files opened to confirm formats (2026-11-16)
Read through the GitHub contents API, first lines only, not reproduced here:
- OoTR `data/World/Deku Tree.json`: list of regions, each mapping location names to boolean expression strings over items and named helpers.
- wwrando `logic/item_locations.txt`: per-location entries with a requirement field, original item and file paths inside the game.
- wotw-seedgen `areas.wotw`: a commented custom syntax with difficulty tiers and parameterized requirement keywords.
- RandomizerMod `Resources/Logic/locations.json`: name plus logic string combining room transitions and state terms.
- OoTMM `data/world/mm/boss/goht.yml`: areas with events, exits and locations as expression strings, using named macros.
- Randovania `logic_database/Chozo Ruins.txt`: human-readable dump of nodes, docks and requirements with engine asset ids.
- PopTracker `doc/PACKS.md`: manifest in JSON, logic and behavior through Lua, data in JSON.
- License files: OoTR `LICENSE`, OoT3D `LICENSE.md`, SoH `3drando/LICENSE.md`, SoulsRandomizers `LICENSE.md`, cheese-trackers `LICENSE`, wotw-rando-client `LICENSE`.

## 6. Not verified
- All **(mem)** items, including most multiworld claims for MMR, TP, SS, Minish, and how alttpr.com and z3randomizer share code.
- Free Enterprise's canonical repo: the one found has 12 stars and a 2024 push; the active source may live elsewhere.
- Whether MultiworldGG and Archipelago-SoH are forks of Archipelago (GitHub reports Archipelago-SoH as a fork; its parent was not checked).
- Community size beyond GitHub stars (Discord member counts, seeds generated) was not measured.
- The BIG List site was not opened; only its repo data was read. 17 of its series files were read, so projects outside those series may be missed.
