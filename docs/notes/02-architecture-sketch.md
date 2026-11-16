# Architecture sketch

Early thinking, not a design. Everything here is a hypothesis to test.

## Principles
1. A room is a key. No ports.
2. One ordered event log per room is the source of truth. Everything else is a view.
3. Old and new standards share one core. Two thin edges translate.
4. Never parse untrusted pickle in the core. Convert on upload in a sandboxed service.
5. Generation is a job. Serving is the easy part.
6. Language-neutral by default: a published schema, JSON on the wire first, binary later.

## Layers

```
 Edges:     v2 native edge        AP-compat edge        Web (console, inbox, API)
                 \                    |                    /
 Core:                 Room actors + event log + subscriptions
                                      |
 Storage:               Postgres (events, snapshots, accounts)
                        Object storage (game data by checksum, results)
 Workers:     Generation queue -> sandboxed workers -> results
              Notification fan-out (push, email, Discord)
 Legacy:      Python sidecar: reads old multiworld files, runs old Python games
```

## Joining a room (compat routing options, best first)
1. Hostname: `abc123.play.example`. Works with any client that takes a hostname.
2. Path: `wss://host/r/abc123`. The reference Python client passes the full URL through. Other-language clients not checked.
3. Per-player token used as the password on a shared port. The server speaks first in the old protocol, so the gateway has to send a placeholder greeting first. Untested.

## The v2 protocol (to write)
- Room key in URL. Per-slot token, not a shared password.
- Items as a numbered log with server-recorded acks. Resume with "after N".
- Game name tables fetched by checksum from a CDN, not on every connect.
- Capabilities negotiated explicitly rather than by version number.
- First-class DeathLink, countdown with ack, release, collect.
- Published JSON schema so clients can be generated in any language.
- HTTP and push fallback for games that cannot hold a socket.

## In-game configuration flow
Game shows a settings screen from the game's manifest. Player locks settings, enters a room key, presses connect. Options go up, validated against the same schema. When all slots are locked, the host starts generation. Results come back on the same connection. For consoles, a pairing code: the game shows a code and the player enters it on a website or phone.

## Patching and emulators (open)
- Browser-side patching: ROM stays on the player's machine, a WebAssembly patcher applies the diff. Untested.
- Runtime memory writes through one local connector, no patched file. Needs per-game memory maps.
- Do not bundle emulator cores before checking licenses. Many are GPL.

## Scale math (assumptions, not measurements)
- 100k games a day is about 1.2 per second on average. Plan for about 10 per second at peak.
- Say 5 players per game, 5-10% online at once: 25-50k open connections, about 1-2k events per second at peak. One tuned Rust server and one Postgres should cope. **To benchmark.**
- Generation, at an unmeasured 10-60 seconds each, is about 12-70 cores around the clock. This is the real cost.
- One box is a single point of failure. Standby, replicated backups, tested restore.
- Idle rooms are evicted to storage and rehydrated on connect.

## Generation
- Job queue, warm workers (container jobs). Lambda is possible but jobs up to 10 minutes and 4 GB sit near its limits, and cold starts are heavy if every game is imported.
- Sandbox each job (Firecracker or gVisor). Uploaded game plugins are arbitrary code.
- Cache results by hash of options plus generator and game versions.
- Alternative for tiny storage: store inputs and regenerate on demand, if the generator version is pinned.

## World standard (to write)
Declarative manifest: game, version, items, locations, regions, connections, rules (expression tree), options (typed, with labels and defaults), per-language names. Escape hatch via WASM. Compare with one existing game that already ships its world as YAML plus a small interpreter. Use the append-only ID ledger pattern.

## Seeds
Readable word-based seed. Shareable settings string encoding options, seed and generator version. Daily and weekly seeds. Race seeds with hidden spoilers.

## Admin console
Roles (owner, moderator, player). Live board. Per-slot actions: release, collect, kick, swap, rotate token, pause. Audit log with undo. API tokens and a Discord bot.

## Open questions
- Stay protocol-compatible indefinitely, or fork the protocol once we have enough games?
- Generate with our own generator, or run theirs as a service first?
- How do we get game developers to adopt a new format?
- Which engines get the first SDK?
