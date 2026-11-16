# Constellation

**Multiplayer randomizers that anyone can join in one click.**

Working name. Status: early research, no product yet. Items marked **[TBD]** are things we have not measured or decided.

---

## 1. The one-liner

Constellation is the easy way to play a *multiworld randomizer*: a game mode where several people play different games at once, and the items you find in your game are secretly handed to someone else's game. We turn a setup that needs config files, network ports and patch files into "enter a room key and press connect."

We run it for players, and we sell studios an official integration so new games ship with it built in.

---

## 2. The issue

Today one open-source project, Archipelago, owns this space. It works. Roughly 80 games are supported, and the shuffling logic is solid. It was built by volunteers, mostly for people comfortable with technical setup, so the experience around that core is hard for newcomers.

### What a player goes through

| Step | What happens today | What it costs the player |
|------|--------------------|--------------------------|
| Set up | Write a settings file in a text format called YAML. Option values look like `true: 50`, where 50 is a probability weight, not "true". | A typo in an option name only logs a warning and silently falls back to the default. |
| Gather | One person collects everyone's file by hand and runs the generator. | One organizer becomes the bottleneck. |
| Patch | Find your own copy of the exact game ROM (the game's data file), set a path in a config file, and apply a patch file. | Wrong version gives a vague "no handler found" error. |
| Install | Pick from several launch paths: separate client programs, an emulator bridge, or a scripting plugin. There are about 69 English setup guides. | Each game is a separate project to learn. |
| Connect | Type host, port, exact slot name and password. Each room runs on its own network port. | Ports change. Firewalls get in the way. A wrong slot name gives no suggestions. |
| Play | Games are slow and take days. There is no "what did I get since I last looked", no alert when someone finds your key item. | Players forget the game is happening. |

### What a game developer or host goes through

- **Developers** must get their game merged into one shared repository and approved by volunteer maintainers over Discord, or ship an unofficial add-on with no registry and no update channel.
- **Hosts** run a server that is one process per room, with a text-command admin console behind one shared password.
- **Operators** hit a hard ceiling: the public site gives each room a port from a range of 16,384, so one machine tops out at about that many live rooms. Each room also polls the database every 5 seconds just to hear about commands.

### The root cause

The core was designed so that one program holds one game session on one port, with identity by slot name. Everything else, ports, files, clients, is a workaround for that. It is not a bug someone can patch. It needs a different foundation.

Evidence for every claim above is in [`notes/01-archipelago-findings.md`](notes/01-archipelago-findings.md).

---

## 3. The fix

### What the player sees

1. A friend sends a **room link**.
2. In the game itself, the player picks their settings on a plain-language screen, locks them in, and presses **Connect**.
3. Everything else happens behind the scenes: the options go up, the game is built, the connection details come back.
4. Later, the player opens their **inbox**: "while you were away you received 4 items, and someone found your Hookshot." Optional push, email or Discord alerts.

No config files. No ports. No patch files to move around. No setup guide longer than a page.

### What the host sees

- A web **console** with real roles (owner, moderator, player), a live board of who is online and who is stuck, one-click release, collect, kick, swap and pause, an audit log with undo.
- Readable **seeds**: a short word-based seed and a shareable settings string that reproduces the same game exactly.
- Built-in **race mode** with a synchronized countdown.

### What the game developer sees

- A **declarative game format**: items, locations, regions, rules and options written as data, not Python code. Simple games are mostly just that.
- An **escape hatch** for custom logic, so unusual games still work.
- A small **SDK** in several languages, not only Python, that draws the settings screen, handles the connection and delivers items.
- A **test harness** that proves every item is reachable across many random seeds before the game is published.
- A **registry** so a game ships on its own schedule, with its own version, with no merge request into a shared repo.

### What stays compatible

We run existing Archipelago games on day one through a compatibility layer, and existing Archipelago game clients can connect to a Constellation room. We are a compatible front door to what the community built, not a replacement for it. That community is the reason the space exists.

---

## 4. How it works (short version)

```
 Player's game / mod / app ──┐
 Old Archipelago client ─────┼─► Edge ─► Room core ─► Event log + database
 Web console / inbox ────────┘             │
                                           ├─► Notifications (push, email, Discord)
                                           └─► Generation workers (sandboxed)
```

- **One front door.** A room is a key in the address, such as `wss://play.example/r/abc123`, or a subdomain. A gateway routes by key. No per-room ports.
- **Rooms are light.** Many rooms live in one process. Idle rooms are saved to the database and wake when someone connects.
- **Items are an ordered log.** The server records what each game acknowledged. A reconnecting game asks "give me everything after number N." Today each game client has to remember that itself.
- **Generation is a job queue.** Self-contained workers, each run in a sandbox because a game plugin is arbitrary code. Identical inputs reuse a cached result.
- **Written in Rust.** One small, fast server for rooms and connections. Old Python games run unmodified in a separate sandboxed service, so nothing is lost, while the new format runs natively.

Design target, not yet proven: tens of thousands of open connections on one well-specced machine, with a standby for failover. Serving is the easy part. Generation is where the compute cost lives. Details in [`notes/02-architecture-sketch.md`](notes/02-architecture-sketch.md).

---

## 5. Why this is a real opportunity

- **Proven demand.** A volunteer project has built about 80 supported games and thousands of commits. People clearly want this. **[TBD: player and room counts. We have no verified numbers yet.]**
- **A clear gap.** The shuffling is solved. The experience is not.
- **A second market that is not yet served.** Studios that want to ship a randomizer or multiworld mode in a new game have no polished, supported, official option today.
- **A format shift.** The most promising existing games already store their world as data (one sampled game is about 85% data). A new, simple format is realistic.

---

## 6. Marketing and growth

### The wedge: be the easiest way to run what people already play

1. **Start inside the existing community.** Launch as a free, AP-compatible host with two things AP lacks: join links and the inbox. Players try it because it takes one minute, not an hour.
2. **Be a good citizen.** Say "works with Archipelago", support the same clients, publish our protocol openly, credit the project. The community writes the games. Goodwill is the moat.
3. **Win the people who organize.** Hosts and streamers decide where friends play. Give them the console, race mode, overlays for stream (a browser source for OBS) and shareable results pages.
4. **Shareable moments.** A results page after each run (who sent what, a timeline, stats) is something players post. It markets the product for us.
5. **Be findable.** Pages for "how to host a [game] multiworld", game-specific landing pages, and a public room directory.

### The second wave: studios and indie developers

- Offer the SDK free and open for indie and hobby developers. It grows the catalog and builds the habit.
- Sell **official integration and support** to studios that want it shipped in their game. We handle the networking, rooms, notifications and moderation. They get a "multiworld ready" feature.
- Show up where developers are: game jams, engine communities (Unity, Godot, Unreal plugin), and a public showcase of games that shipped with it.

### Messaging

- To players: **"Join in one click. We'll tell you when something happens."**
- To hosts: **"Run your room from one screen."**
- To studios: **"Ship a multiplayer randomizer mode without building the networking."**

---

## 7. Monetization

We charge for the **service**, never for other people's game content. See legal notes below.

| Stream | Who pays | What they get | When | Main risk |
|--------|----------|---------------|------|-----------|
| **Studio integration and support** | Game studios | Official SDK integration, licensed assets, support, SLA, custom branding | Early, even before consumer scale | Long sales cycles, small number of buyers |
| **Paid rooms** | Hosts, communities | Bigger rooms, longer-lived or always-on rooms, private rooms, custom domain, more history | After launch | Free tier must stay generous or players leave |
| **Notifications and integrations** | Players and communities | Push, SMS, Discord bot, webhooks, summaries | After inbox ships | Small willingness to pay per player |
| **Creator and event tools** | Streamers, tournament runners | Race mode, brackets, overlays, branded pages, analytics | Mid term | Prizes and entry fees raise gambling-law issues, so we avoid them |
| **Developer services** | Game authors | Hosted CI and test harness, analytics, private registry, priority review | Mid term | Free tools must stay free to keep the catalog growing |

**Free for everyone:** small public rooms, the inbox, the join flow, the SDK.

**We do not sell:** game files, patches, ROMs, or anything derived from a publisher's game. That is both a legal line and a trust line.

Prices, margins and projections: **[TBD, needs customer interviews and a cost model].** Cost drivers we can already name: compute for generation, storage and egress for game data, and notifications.

---

## 8. Roadmap (rough order)

1. **Prove the foundation.** Read existing generated files in a safe converter service. Stand up a Rust room server with room keys and the event log. *Milestone: an existing Archipelago client plays a real room on it.*
2. **The player layer.** Join links, the inbox, the web console. *Milestone: a friend group finishes a real multiworld without anyone editing a config file.*
3. **Generation as a service.** Sandboxed worker queue running existing games, with result caching and readable seeds.
4. **The new standard.** Declarative game format, SDK, registry, test harness. *Milestone: one real game ported to the new format, with its settings screen inside the game.*
5. **Studio pilot.** One studio ships a game with the SDK built in during development.
6. **Scale and monetize.** Paid rooms, notifications, creator tools.

---

## 9. Risks and how we handle them

| Risk | Why it matters | Plan |
|------|----------------|------|
| **Publisher IP** (the big one) | Fan randomizers survive because they are free hobby projects that distribute no game files. Charging money raises the stakes. | Never host or distribute game files. Patch on the player's own machine. Make official studio deals the paid business. Get a lawyer before launch. |
| **Community backlash** | Looking like a takeover would lose the people who build the games. | Stay compatible, credit openly, contribute back, release the protocol and SDK openly. |
| **Chicken and egg** | No games means no players. | Run all existing games on day one through the compatibility layer. |
| **One-machine risk** | A small fast server is a single point of failure. | Hot standby, replicated backups, tested restore, from the first paying user. |
| **Abuse** | Free hosting attracts spam, and uploaded game plugins are code that runs on our servers. | Quotas, rate limits, name moderation, and sandboxing everything uploaded. |
| **Generation cost** | Old Python generation is heavy. | Cache by input, autoscale workers, and move to the native format over time. |
| **Console reality** | A stock console cannot run a randomizer at all. | Support modded consoles, PC and emulators, with a pairing-code flow and in-browser patching. Official studio games can integrate directly. |

Legal summary and the questions to bring to a lawyer: [`notes/03-legal-notes.md`](notes/03-legal-notes.md). This is research, not legal advice.

---

## 10. What we need

**[TBD]** Team, funding or runway, first design partner studio, first community champion.

The immediate next step is small: build the room-key server and show an existing client connecting to it. That single demo proves the foundation and is the first thing worth showing anyone.
