# Archipelago findings

Research notes from reading the Archipelago repo (read-only, nothing copied). Paths are relative to that repo, for anyone who wants to check. "Verified" means checked in code. "Inferred" means an agent's or my reading. Snapshot taken November 2026.

## Scale
- About 83 folders under `worlds/` (about 80 games). About 694k lines of Python in total, of which about 655k are games. Core and web host are small by comparison (about 4.6k web, about 9k tests).
- About 8.6k commits, about 346 distinct commit authors.

## Generation
- Fixed sequence of hooks run in lockstep across every game (`Main.py`). Nothing can be added or removed after item creation (`docs/world api.md`).
- Fill places an item, then checks it is reachable, and swaps on dead ends (`Fill.py:766`). Failures show up after a full run.
- Player YAML: an unknown option logs a warning and falls back to the default (inferred line `Generate.py:~620`).
- `meta.yaml` overrides every player file. Triggers add more layers.
- Option template writes `option: 50` as a weight (`data/options.yaml:80-88`, verified).
- Games are metaclass-registered Python plugins running in-process (`worlds/AutoWorld.py:50`).
- Rules are mostly lambdas. A newer `rule_builder/` is serializable but used by 3 of the worlds sampled.
- Output: one zip containing a zlib-compressed pickle plus per-game patch files. The `--race` flag reseeds nondeterministically, so a race seed is not reproducible from its number.

### Suspected, not verified
- `Fill.py:1099-1126` may use an item after clearing it in the accessibility correction step.

## Runtime
- One process, one multiworld, one port. No room ID exists in the protocol. Login carries slot name, game, password, no room (`docs/network protocol.md`, verified).
- Identity is the slot name plus one room password. No per-slot secret.
- The server resends the full item list on connect. Each game client must remember its own "last processed index". The server never learns what the game applied.
- DeathLink is a client convention over generic `Bounce` packets. The server has no code for it.
- Save is a periodic full pickle every 60s, written with a plain overwrite (`MultiServer.py:685-770`, inferred risk of corruption on crash).
- Reconnect delay doubles forever with no cap (`CommonClient.py:941`, verified).
- Five different ways a game attaches: client subclass, SNI bridge, BizHawk Lua bridge, external-language libraries, text client. About 10 client scripts at repo root and about 69 English setup guides.
- Patching: players supply their own ROM, a hash is verified, a patched copy is written next to the patch file.
- Install from source prompts interactively and installs every game's requirements together, all or nothing.

## Web host and operations
- Flask plus Pony ORM, SQLite by default (`WebHostLib/__init__.py:61`).
- Seed is stored as one blob column. Room save is a whole pickled blob rewritten on each save (`WebHostLib/customserver.py:166`).
- Each room process polls a command table every 5 seconds (`customserver.py:110-126`).
- Room ports are chosen from 49152-65535 (`__init__.py`), so about 16,384 maximum rooms per IP.
- Generation: default 8 workers, 600 second limit, 4 GB memory cap per job (`__init__.py:46-53`).
- Rooms shut down after 2 hours idle by default, with a 3-day cap.
- No player notification, subscription, email or webhook feature found. The tracker API is poll-only snapshots.

## File size sample (one real file, from our own other project, single player, 61 locations)
- 130 KB on disk, 1.0 MB expanded.
- About 99% by text size is the game's name-to-ID tables, identical for every seed of that game. The seed-specific part is about 11 KB of text.
- Takeaway: store the name tables once, keyed by checksum. Estimate 2-5 KB per seed. Not measured. One sample, treat as directional.

## World format (8 sampled worlds)
Full details in the agent report. Key points:
- The wire contract is two name-to-int dicts plus groups. IDs are namespaced per game.
- A good pattern to copy: an append-only ID ledger (Lingo).
- About 80 world files use `lambda state` rules. Three use `rule_builder`.
- Legacy worlds need CPython: metaclasses, global monkey-patching of the logic state, direct list edits, Python `random`, per-world pip requirements.
- Declarative share (counted): about 91% for one game, about 85% for two others. Estimated 60-90% for small games. About 55-65% for large ones. ROM output and client code are never declarative.
- Escape-hatch hotspots: seeded shuffles with retries, generated mission graphs, phase-dependent logic, patch output.
- Not benchmarked: WASM speed for hot rule loops.

## Governance
- World changes need a world maintainer plus a core maintainer. Maintainers are added by majority vote on Discord over a week.
- Contributing guide admits test coverage is incomplete.
- Docs are inconsistent across games. 69 setup guides in English, fewer than a dozen in each other language.

## What we did not check
- Git history or chat for the original reasons behind decisions.
- Per-game setup guides, one by one. Other-language client libraries.
- Real-world player or room counts, or how often ports change for real players.
