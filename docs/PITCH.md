# Constellation

> Draft for private review. Statements about law and about Archipelago have not been reviewed by a lawyer or by the Archipelago maintainers. Facts are sourced or labeled in `notes/`.

**Multiplayer randomizers that anyone can join in one click.**

Working name. Status: early research, no product yet. Items marked **[TBD]** are things we have not measured or decided.

---

## 1. The one-liner

Constellation is the easy way to play a *multiworld randomizer*: a game mode where several people play different games at once, and the items you find in your game are secretly handed to someone else's game. We turn a setup that needs config files, network ports and patch files into "enter a room key and press connect." The principle: the second a player has to move or install a file, we have lost.

We run it for players, and we sell studios a supported studio integration so new games ship with it built in.

**Direction (proposed, not final; decision D-01 in [`design/DECISIONS.md`](design/DECISIONS.md)): web-first and standard-only.** Everything we build is on the web: configuration, options, room and world setup, generation, seeds, the admin console, inbox and notifications, join links, console pairing-code entry. We build no desktop app, launcher, connector or patcher, and no game-specific clients. Games and mods embed our SDK or implement the open protocol and connect directly. Third parties may build bridges and patchers to the standard, and existing Archipelago clients keep working through a compatibility layer. The design documents start at [`design/README.md`](design/README.md).

---

## 2. The issue

Today one open-source project, Archipelago, is the main open-source project in this space. It works. Roughly 80 games are supported, and the shuffling logic is solid. It was built by volunteers, mostly for people comfortable with technical setup, so the experience around that core is hard for newcomers.

### What a player goes through

| Step | What happens today | What it costs the player |
|------|--------------------|--------------------------|
| Set up | Write a settings file in a text format called YAML. Option values look like `true: 50`, where 50 is a probability weight, not "true". | A typo in an option name logs a warning, then falls back to the default. |
| Gather | One person collects everyone's file by hand and runs the generator. | One organizer becomes the bottleneck. |
| Patch | Find your own copy of the exact game ROM (the game's data file), set a path in a config file, and apply a patch file. | Wrong version gives a vague "no handler found" error. |
| Install | Pick from several launch paths: separate client programs, an emulator bridge, or a scripting plugin. There are about 69 English setup guides. | Each game is a separate project to learn. |
| Connect | Type host, port, exact slot name and password. Each room runs on its own network port. | Ports change. Firewalls get in the way. A wrong slot name gives no suggestions. |
| Play | Games are slow and take days. There is no "what did I get since I last looked", no alert when someone finds your key item. | Players forget the game is happening. |

### What a game developer or host goes through

- **Developers** must get their game merged into one shared repository and approved by volunteer maintainers over Discord, or ship an add-on outside the main repository, with no registry and no update channel.
- **Hosts** run a server that is one process per room, with a text-command admin console behind one shared password.
- **Operators** hit a default limit (counted from their default configuration, November 2026): rooms get ports from a range of 16,384 per IP address, so a single address tops out at about that many live rooms unless the setup changes. Each room also polls the database every 5 seconds just to hear about commands.

### The root cause

The core was designed so that one program holds one game session on one port, with identity by slot name. Everything else, ports, files, clients, is a workaround for that. In our reading (inferred), much of the setup friction follows from that one-session-per-port design, which made sense for a volunteer project. Fixing it means a different foundation, not a patch.

Most claims above are sourced with file and line references in [`notes/01-archipelago-findings.md`](notes/01-archipelago-findings.md). A few are not sourced there and should be treated as not verified: the maintainer approval process, the lack of a registry for third-party games, and some error message wording.

---

## 3. The fix

### What the player sees

1. A friend sends a **room link**.
2. On a web page, the player picks their settings on a plain-language screen, locks them in, and presses **Connect**. A game or mod that embeds the SDK can show the same screen itself. On a console, the player types a short pairing code on a web page.
3. Everything else happens behind the scenes: the options go up, the game is built, and the game connects directly to the room.
4. Later, the player opens their **inbox**: "while you were away you received 4 items, and someone found your Hookshot." Optional push, email or Discord alerts.

No config files. No ports. Nothing for us to install on the player's machine. Patch packages, where a game needs one, are an optional extension that third-party tools can apply (D-07). No setup guide longer than a page.

### What the host sees

- A **staging lobby with a ready check** (next section).
- A web **console** with real room roles (owner, co-host, moderator; players come from the seat they hold), a live board of who is online and who is stuck, one-click release, collect, kick, swap and pause, and an audit log. Dangerous actions ask for a fresh passkey or sign-in check first.
- **Hosting needs a one-tap account; joining never does** (proposed, owner decision open: D-63). Today's web host has no accounts (from our reading of their public code, cited in doc 10 section 1.2 and not yet re-checked): it gives each visitor a random ID in a browser cookie and treats that ID as the room's owner. That was a sensible design for a volunteer project with no user database, and it has one cost for a hosted service: clear your cookies or switch device and you can lose a room that holds days of play. In ours, authority belongs to an account's role in the room, a host can sign in from any device, there are recovery options without a password, and a room always has an owner you can hand over (doc 10, [`design/10-identity-and-room-ownership.md`](design/10-identity-and-room-ownership.md)).
- Readable **seeds**: a short word-based seed and a shareable settings string that reproduces the same game exactly.
- Built-in **race mode** with a synchronized countdown.

### What the game developer sees

- A **declarative game format**: items, locations, regions, rules and options written as configuration data, not Python code. Pure data is a starting direction for the first native version, not locked. Simple games are mostly just that.
- A **reserved hook** for custom logic (a world module). It is disabled in native version 1 (D-10); unusual games run through the compatibility layer meanwhile.
- An **open protocol** to speak directly, and a small **SDK** in several languages, not only Python, that handles the connection and delivers items. The settings screen is rendered on the web from the game's manifest by default; a game may show it itself through the SDK. We build no game-specific clients; third parties can build bridges and patchers to the standard.
- A **test harness** that proves every item is reachable across many random seeds before the game is published.
- A **registry** so a game ships on its own schedule, with its own version, with no merge request into a shared repo.

### The host staging lobby and ready check

Before launch, the host opens a lobby page for the room. Each seat shows a state: invited, joined, game connected, ready. Ready needs a real connection from the game, not a click, so it means something. The host sees who is missing and why (no connection yet, options not locked, console pairing code not entered) and can resend a link, reassign a seat or start without the missing players. Launch is blocked, or needs an explicit override, until every seat is ready. A check before generation (Check A) and one after (Check B) catch problems early.

Why it matters to a non-technical host: in a multiworld game one missing player stalls everyone, and today the host finds out by asking in chat, sometimes hours in. The lobby turns that into a checklist anyone can read. The full design (client classes, join links, the two checks, launch rules, diagnostics) is in [`design/09-connection-and-lobby.md`](design/09-connection-and-lobby.md). Its numbers are design values, not measured.

### What stays compatible

We run existing Archipelago games on day one through a compatibility layer, and existing Archipelago game clients can connect to a Constellation room (as players; admin actions stay in the web console, D-64). We are a compatible front door to what the community built, to be used alongside it. That community is the reason the space exists.

---

## 4. How it works (short version)

```
 Game or mod using the SDK ──┐
 Third-party bridge ─────────┤
 Old Archipelago client ─────┼─► Edge ─► Room core ─► Event log + database
 Web console / inbox ────────┘             │
                                           ├─► Notifications (push, email, Discord)
                                           └─► Generation workers (sandboxed)
```

- **One front door.** A room is a short key in the address, such as `wss://play.example/r/abc123`. A gateway routes by key. No per-room ports.
- **Rooms are light.** Many rooms live in one process. Idle rooms are saved to the database and wake when someone connects.
- **Items are an ordered log.** Each event gets a sequence number, and the server records how far each game has applied. A reconnecting game asks "give me everything after number N." Today each game client has to remember that itself.
- **Generation is a job queue.** Self-contained workers, each run in a sandbox because a game plugin is arbitrary code. Identical inputs reuse a cached result.
- **Written in Rust.** One small, fast server for rooms and connections. Old Python games run unmodified in a separate sandboxed service, so nothing is lost, while the new format runs natively.

Design target, not yet proven: tens of thousands of open connections on one well-specced machine, with a warm standby in a second region for failover (proposed, owner decision open: D-65). Serving is the easy part. Generation is where the compute cost lives. Details in [`design/02-architecture-and-scale.md`](design/02-architecture-and-scale.md) and [`design/06-multi-region.md`](design/06-multi-region.md); the early sketch is [`notes/02-architecture-sketch.md`](notes/02-architecture-sketch.md).

---

## 5. Why this is a real opportunity

- **Proven demand.** A volunteer project has built about 80 supported games and thousands of commits. People clearly want this. **[TBD: player and room counts. We have no verified numbers yet.]**
- **A clear gap.** The shuffling is solved. The experience is not.
- **A second market that is not yet served.** Studios that want to ship a randomizer or multiworld mode in a new game have no polished, supported option today (inferred, not verified).
- **A format shift.** In our sample of 8 existing games, three were about 85-91% data and rules by line count (counted). The others are estimated lower. A new, simple format is realistic.

---

## 6. Marketing and growth

### Start here: be the easiest way to run what people already play

1. **Start inside the existing community.** Launch as a free, Archipelago-compatible host with two things we found no sign of in their repo: join links and the inbox. Players try it because it takes one minute, not an hour.
2. **Be a good citizen.** Say "works with Archipelago", support the same clients, publish our protocol openly, credit the project. The community writes the games. Goodwill is what makes this work.
3. **Win the people who organize, and make results shareable.** Hosts and streamers decide where friends play. Give them the console, race mode, stream overlays (a browser source for OBS) and a results page after each run (who sent what, a timeline, stats) that players post.
4. **Be findable.** Pages for "how to host a [game] multiworld" and a public games directory. A public room directory comes only after moderation tools and the report flow exist (D-06).

### The second wave: studios and indie developers

- Offer the SDK free and open for indie and hobby developers. It grows the catalog and builds the habit.
- Sell a **supported studio integration** to studios that want it shipped in their game. They embed the SDK or speak the protocol; we handle the rooms, web setup, notifications and moderation. They get a multiworld mode without building the networking.
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
| **Studio integration and support** | Game studios | Supported SDK integration, support, SLA, custom branding | Early, even before consumer scale | Long sales cycles, small number of buyers |
| **Paid rooms** | Hosts, communities | Bigger rooms, longer-lived or always-on rooms, private rooms, custom domain, more history | After launch | Free tier must stay generous or players leave |
| **Notifications and integrations** | Players and communities | Web push, email, Discord bot, webhooks, summaries | After inbox ships | Small willingness to pay per player |
| **Creator and event tools** | Streamers, tournament runners | Race mode, brackets, overlays, branded pages, analytics | Mid term | Prizes and entry fees raise gambling-law issues, so we avoid them |
| **Developer services** | Game authors | Hosted CI and test harness, analytics, private registry, priority review | Mid term | Free tools must stay free to keep the catalog growing |

**Free for everyone:** small public rooms, the inbox, the join flow, the SDK.

**We do not sell:** game files, patches, ROMs, or anything derived from a publisher's game. That is both a legal line and a trust line.

Prices, margins and projections: **[TBD, needs customer interviews; the cost model is [`design/13-cost-model.md`](design/13-cost-model.md), all estimates].** The working plan names are in [`design/03-platform-and-ecosystem.md`](design/03-platform-and-ecosystem.md) section 7.2. Cost drivers we can already name: compute for generation, storage and egress for game data, and notifications.

---

## 8. Roadmap (rough order)

1. **Prove the foundation.** Read existing generated files in a safe converter service. Stand up a Rust room server with room keys and the event log. *Milestone: an existing Archipelago client plays a real room on it.*
2. **The player layer.** Join links, accounts for hosts, the inbox, the web console, plus settings, generation, seeds, pairing-code entry and the staging lobby with its ready check, all on the web. *Milestone: a friend group finishes a real multiworld without anyone editing a config file.*
3. **Generation as a service.** Sandboxed worker queue running existing games, with result caching and readable seeds.
4. **The new standard.** Declarative game format, SDK, registry, test harness. *Milestone: one real game ported to the new format, with its settings chosen on the web or in a game that embeds the SDK.*
5. **Studio pilot.** One studio ships a game with the SDK built in during development.
6. **Scale and monetize.** Paid rooms, notifications, creator tools. We start small, on one inexpensive server (about $23 a month at first, about $93 once the public beta opens; estimates in doc 12, scenarios in doc 13), and grow in stages as use grows.

---

## 9. Risks and how we handle them

| Risk | Why it matters | Plan |
|------|----------------|------|
| **Publisher IP** (the big one) | Our reading (inferred, not legal advice): fan randomizers have mostly avoided trouble by being free and distributing no game files. Charging money may raise the stakes. | Never host or distribute game files. If a game needs a patch, a third-party tool applies it on the player's own machine; we ship none. Make studio deals the paid business. Get a lawyer before launch. |
| **Community backlash** | Looking like we compete with the community would lose the people who build the games. | Stay compatible, credit openly, contribute back, release the protocol and SDK openly. |
| **Chicken and egg** | No games means no players. | Run all existing games on day one through the compatibility layer (vetted worlds first, D-04). |
| **One-machine risk** | A small fast server is a single point of failure. | We start on one inexpensive server with offsite backups and a tested restore, and say plainly that the last minute or two can be lost if the whole machine dies (doc 12). Warm standby in a second region from the first paid rooms (doc 06). |
| **Abuse** | Free hosting attracts spam, and uploaded game plugins are code that runs on our servers. | Quotas, rate limits, name moderation, and sandboxing everything uploaded. |
| **Generation cost** | Old Python generation is heavy. | Cache by input, autoscale workers, and move to the native format over time. |
| **Console reality** | A stock console cannot run a randomizer at all. | Third-party bridges for modded consoles and emulators, built to the open standard, with a pairing code entered on a web page. Browser-side patching is a possible later feature, not committed. Studio games can integrate directly. |

Legal summary and the questions to bring to a lawyer: [`notes/03-legal-notes.md`](notes/03-legal-notes.md) and the 15 counsel questions in [`design/04-security-trust-safety.md`](design/04-security-trust-safety.md) section 13. This is research, not legal advice.

---

## 10. What we need

**[TBD]** Team, funding or runway, first design partner studio, first community champion.

The immediate next step is small: build the room-key server and show an existing client connecting to it. That single demo proves the foundation and is the first thing worth showing anyone.
