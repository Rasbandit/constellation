# Batch C summary: ROM-patch generators, solvers and web hosts

> Internal research note, 2026-11-16. Covers [09-alttpr.md](09-alttpr.md), [10-varia.md](10-varia.md), [11-worlds-collide.md](11-worlds-collide.md) and [12-albw-z17.md](12-albw-z17.md). Not reviewed by a different agent or a human. Our own writing; every claim below is cited in the per-project note it comes from. Numbers are labeled as in those notes.
>
> Analysis-side research under [`docs/notes/03-legal-notes.md`](../../notes/03-legal-notes.md) (section "If we ever want a clean-room claim"): our agents read the public code of the projects described here, so this note cannot support a clean-room claim.

## 1. What the four projects are
| Project | Game | Language | Logic form | Delivery | License |
|---|---|---|---|---|---|
| ALttPR | ALttP (SNES) | PHP + Vue | closures per region, trick flags | browser patch from a stored seed | MIT |
| VARIA | Super Metroid (SNES) | Python | lambdas returning (bool, cost, techniques) | browser patch of a server IPS | MIT |
| Worlds Collide | FF6 (SNES) | Python | character gates only, plus objectives | full ROM from CLI | MIT |
| z17 | ALBW (3DS) | Rust | function pointers per logic tier | LayeredFS mod folder | GPL-2.0 |

All four keep logic in code. None has static checks on logic. Two (ALttPR, VARIA) are web-first; two are local tools.

## 2. Cross-project patterns
- **Assumed fill is the default for item logic** (ALttPR `RandomAssumed.php`, z17 `assumed_fill`); VARIA uses forward fills with rollback or random fill plus a solver gate; Worlds Collide uses a forward gated fill because gates only unlock.
- **Seed confirmation in game.** ALttPR writes a five-symbol code from the seed id to the start screen; z17 shows a five-icon hash of seed, settings and version; Worlds Collide shows a sprite hash of seed, flags and version. Three of four projects (counted).
- **Settings bound into the seed.** Worlds Collide seeds its RNG with seed plus gameplay flags (cosmetics excluded); z17 hashes seed, settings and version for the display code. Our settings string (01 D3.1) already carries both; we should state that cosmetic options are excluded from the hash input.
- **Race secrecy kept out of the public code.** ALttPR's tournament transform is an empty function in the public repo, and VARIA imports a race module that is not in the tree (both inferred from the public code). Race integrity therefore rests on the operator, not on verifiable data.
- **Bounded retries with readable errors.** VARIA (up to 500 layout attempts and a time limit, with messages per cause), z17 (100 attempts with a fresh seed, deterministic failure when the user fixed the seed), ALttPR (HTTP 409 with the reason).
- **Playthrough as spoiler.** ALttPR computes a minimal required set by removal per sphere; z17 stores spheres including hints, goals and shuffled cracks; VARIA's solver path is its spoiler and its difficulty report.
- **Two presets beat one.** VARIA splits "what I can do" (skill preset) from "what I want" (rando preset). ALttPR and z17 fold tricks into a logic level.

## 3. The strongest ideas to adopt
1. **Difficulty-valued logic (VARIA).** Each rule leaf has an integer cost; AND sums, OR takes the cheapest branch; a per-player skill profile sets each technique's cost or disables it. This one change gives difficulty estimates, a "techniques this seed needs" list, and generation inside a difficulty window. It stays total and integer-only, so it fits 01 D2.5.
2. **Solver as a separate, reusable service (VARIA).** The same engine generates, rates uploaded seeds, powers the tracker and fills plando. For us: the independent validator in 01 D2.9 should output a canonical playthrough with per-step cost, not only "beatable: yes".
3. **Web delivery with a cached base file and a tiny seed delta (ALttPR, VARIA).** The player picks the base file once; it stays in browser storage, checked by MD5 or CRC32; a per-build base patch plus a per-seed write list is applied in the page; cosmetics are applied client-side; a short permalink reopens the seed. ALttPR's `/h/{hash}` is one click to a playable file. This is the concrete shape of our CPK (01 D4.1) and the optional browser patcher (D-58).
4. **Spoiler policy as a seed property (ALttPR).** `on`, `off`, `generate` (creator sees it once, the shared copy never has it), `mystery` (only safe metadata). Add `creator_once` to 01 D3.3.
5. **Objectives as data (Worlds Collide, VARIA).** Goal templates made of condition kinds, seeded count ranges, any-N-of-M, and a result. Fits race formats and is shown in game.
6. **Hints placed only where reachable first (z17).** A hint source must be logically reachable without the item it reveals.
7. **Fair forward gating (Worlds Collide).** Candidate slots lose weight each round they wait, countering the early-unlock bias of forward fill.
8. **Tiers as ordered alternatives (z17).** Each logic tier adds ways, never removes them; compiles to `any` plus `opt`.

## 4. Recurring pain points (design costs, not criticism of people)
- **Logic only one runtime can read.** PHP closures, Python lambdas, Rust function pointers. Trackers re-encode it (OpenTracker for ALttP) or vendor the whole tree (Archipelago vendors VARIA). This is the core case for a data world format (07).
- **Reproducibility gaps.** ALttPR uses an unseedable CSPRNG, so a seed exists only as a stored result. VARIA race seeds depend on a server-held magic number. Neither can be regenerated by a third party.
- **Little or no logic testing.** ALttPR has about 1,000 access test rows (estimated); VARIA relies on a soak script; Worlds Collide and z17 have no logic tests (counted).
- **Local tools mean file moves.** z17's output is a folder copied to an SD card per seed; Worlds Collide writes a full ROM.
- **Duplication across variants.** ALttPR mode class trees, VARIA's three graph variants.
- **Hosted generation as a subprocess.** VARIA and ALttPR (for entrance shuffle) shell out per request; cost and isolation are per process (not measured).

## 5. Candidate primitives and standard features, with evidence
| Candidate | Kind | Evidence | Projects |
|---|---|---|---|
| `cost` on rule leaves (sum for AND, min for OR) and `skill_profile` | language extension plus settings concept | `logic/smbool.py:78-118`, `standard_presets/` | VARIA; coarse form in z17 tiers; wotw-seedgen tiers (00 section 5) |
| `difficulty_window` with bounded retries | generator feature | `rando/FillerRandom.py:199-230` | VARIA |
| `tiers` authoring macro | macro over `any` and `opt` | `randomizer/src/filler/logic.rs:61-80` | z17, ALttPR logic levels (as flag sets) |
| `objectives` extending `goal` | content model | `constants/objectives/*.py`, `utils/objectives.py` | Worlds Collide, VARIA |
| `spoiler_policy` with `creator_once` | 01 D3.3 concept | `RandomizerController.php:39-79` | ALttPR |
| `seed_code` shown in game | 01 D3.2 concept | `World.php:1092`, `lib.rs:399-440`, `sprite_hash.py` | ALttPR, z17, Worlds Collide (three) |
| `cosmetic` option flag, excluded from seed hash | option attribute | `args/arguments.py:36-52` | Worlds Collide; ALttPR and VARIA apply cosmetics client-side |
| `always_allow` on `item_rule` | primitive extension | `DesertPalace.php:117-120` | ALttPR |
| `exit_rule` (leave after taking) | candidate, try as event plus connection first | `PostAvailable` in `graph_locations.py` | VARIA |
| `hint_source_reachable_before` | constraint for `hint_plan` | `hints/mod.rs:55-59` | z17; OoTR hint plan (01-oot-randomizer.md) |
| `fair_forward_weighting` | option on gated fill | `event/event_reward.py:58-90` | Worlds Collide |
| Reference playthrough output from the validator | 01 D2.9 addition | `PlaythroughService.php`, `metrics/mod.rs`, `solver/` | all three item-logic projects |
| Base-file cache plus seed-delta browser patch | CPK and web client pattern | `resources/js/rom.js`, `web/views/randomizer.html:808-882` | ALttPR, VARIA |

Only `seed_code` and the reference playthrough clear the rule of three (07 2.4) within this batch alone; the others need evidence from other batches before admission.

## 6. What could not be read, and why
- The ff6worldscollide.com site and its delivery path are not in the repo; not opened.
- The private race and tournament modules of VARIA and ALttPR are absent from the public code by design.
- ALttPR's vendored entrance randomizer (`vendor/`) is not in the repo tree we cloned.
- Binary patches, sprites and graphics folders were left out on purpose (asset folders).
