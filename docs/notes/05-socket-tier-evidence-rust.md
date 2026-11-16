# 05. Socket and room tier: evidence from the Rust side

Status: internal research note, 2026-11-16. Not for external use. Written under `CLAUDE.md`: no em dashes, every number labeled, only pages we opened are cited as sources.

Purpose: evidence for and against an all-Rust socket and room tier (docs 02, 05, 09), and for the hybrid the owner is weighing (Elixir/BEAM for sockets and room processes, Rust for logic), seen from the Rust side. A companion note covers the BEAM side.

## Label key

- **[M]** measured by the source we opened, with the source's own hardware and method noted.
- **[C]** counted or read from a page we opened (a stated fact, a documented API, a quoted figure).
- **[E]** our own estimate or arithmetic from numbers above it, formula shown.
- **[V]** vendor or author claim about their own product, not independently reproduced.
- **[?]** from memory, not verified. Also used for pages surfaced by a search tool but not opened by us.

## Pages we opened (the only sources cited as evidence)

| Short name | URL | Date shown on page | Kind |
|---|---|---|---|
| lesismal tokio_tungstenite README | https://github.com/lesismal/go-websocket-benchmark/blob/main/frameworks/tokio_tungstenite/README.md | undated; names tokio-tungstenite 0.30.0 and Rust 1.98, so 2026 [E] | third-party benchmark repo, third-party framework port |
| miraclews case study | https://github.com/pikyusufaslan/miraclews/blob/main/CASE_STUDY.md | undated; cites SaaS prices "as of July 2026" | author of a competing Rust WS engine [V] |
| Genusify 1M post | https://www.genusify.com/blogs/one-million-websockets-single-server-rust | undated | agency marketing [V] |
| Pushpin c1m | https://github.com/fanout/pushpin-c1m | undated; Ubuntu 20.04, so about 2021 [E] | vendor repro instructions |
| Rust users forum thread | https://users.rust-lang.org/t/ballpark-number-of-mostly-idle-tcp-connections-that-async-server-can-handle/92910 | 2023-04-22 | forum, no data |
| Figma, Rust in production | https://www.figma.com/blog/rust-in-production-at-figma/ | 2018-05-02 | first-party post |
| Discord, Go to Rust (Read States) | https://discord.com/blog/why-discord-is-switching-from-go-to-rust | 2020-02-04 | first-party post |
| Discord, Rust NIF for Elixir | https://discord.com/blog/using-rust-to-scale-elixir-for-11-million-concurrent-users | 2019-05-17 | first-party post |
| Cloudflare, Pingora | https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/ | 2022-09-14 (metadata only; body not read) | first-party post |
| Zed collab (DeepWiki) | https://deepwiki.com/zed-industries/zed/11.1-collaboration-architecture | indexed 2026-11-06, commit e231594b | AI-generated wiki over the repo, secondary |
| Ryhl, what is blocking | https://ryhl.io/blog/async-what-is-blocking/ | 2020-12-21, revised 2021-08-22 | Tokio maintainer |
| Ryhl, actors with Tokio | https://ryhl.io/blog/actors-with-tokio/ | 2021-02-13, revised 2023-02-12 | Tokio maintainer |
| tokio::task::coop docs | https://docs.rs/tokio/latest/tokio/task/coop/index.html | tokio 1.54.1, 2026-10-21 | official docs |
| Quickwit, Chitchat | https://quickwit.io/blog/chitchat | 2022-04-28 | first-party post |
| Sarnacki, you still have to think | https://itsallaboutthebit.com/i-am-sorry-but-you-still-have-to-think/ | 2026-10-11 | practitioner blog (Elixir user, fair tone) |
| Dixon, select! cancellation | https://dixon.cx/blog/tokio-select-cancellation-safety.html | 2026-07-23 | practitioner blog |
| HN, Elixir and Rust is a good mix | https://news.ycombinator.com/item?id=35559925 | 2023-04-13 | forum thread, partly read by keyword search of the saved page |

Surfaced by Perplexity or SearXNG but NOT opened (treat as [?]): kameo and ractor docs.rs pages, foca docs, Conduit and Synapse comparison pages, sockudo-ws, initnode and pockit "1M connections" guides, Muqabla/Medium "1M with Tokio" posts, jemalloc profiling guides, async-book cancellation chapter, sunshowers "Cancelling async Rust".

Two source problems to know about:
- Firecrawl "query" mode failed on three pages (Discord NIF, Discord Read States, Pingora), so I re-read Discord in full and could not read the Pingora body. Pingora numbers below are therefore [?] except the one line in its metadata.
- The Zed page is an AI-generated summary of source files, not Zed's own writing. It names real files but I did not open them.

---

## Q1. Per-connection memory and max idle connections per node (Rust WebSocket stacks)

### Findings

1. **Library defaults can cost 30 times the tuned figure.** The lesismal README states that as shipped, tokio-tungstenite keeps a 128 KiB read buffer (`read_buffer_size`) and zero-fills the part each read uses, and a write buffer that gathers frames up to 128 KiB and never shrinks [C]. At 50,000 connections their echo test used 6.85 GB and their pipeline test 8.9 GB in a 12.48 GB container [M]. With the read buffer at 4 KiB, write buffer 0, a shared per-thread read buffer and pooled write buffers, the same tests used 403 MB and 723 MB average (900 MB peak) [M]. Setup [C]: Docker, 3 CPUs for the server and 4 for the client, 50,000 connections, 1 KiB payload, three runs averaged, `panic = "abort"` in their release profile. The README says the `memset` alone cost the server about three echoes in ten, and throughput rose from 315k to 454k echo TPS with the tuning [M]. Credibility: reproducible repo with pinned `Cargo.lock`; the author is a Go-framework benchmarker who ported tokio-tungstenite, so the tuned build is theirs, not the library's. The 403 MB figure is under active echo load, not idle.
   - Derived [E]: 403 MB / 50,000 = about 8 KB of resident memory per connection with the tuned config under echo load. 6.85 GB / 50,000 = about 137 KB per connection with defaults. The README does not state either per-connection figure.
   - Consequence for us [E]: at doc 02's 75,000 open connections per region at rung 1, the default 128 KiB read buffer alone would be 75,000 x 128 KiB = about 9.6 GB. At 4 KiB it is about 300 MB. Doc 05 section 17 already says "pooled 4 KB buffers" but names no crate setting; the setting must be pinned and tested.
2. **Idle memory per connection, as claimed by one author.** The miraclews case study lists userspace idle memory of 4.0 KB for fastwebsockets and 10.0 KB for tokio-tungstenite, plus a measured kernel floor of about 3.3 KB per TCP connection, totalling about 7.3 KB and 13.3 KB [V]. It also says its own engine uses about 0.2 KB userspace, 3.5 KB total [V]. The page does not make clear that the fastwebsockets and tokio-tungstenite rows were measured by the author on the same rig, and it admits it tuned those rows "lower" than measured. Treat the stack comparison as a rough ordering, not a measurement. The kernel figure (3.3 KB, NOTRACK, tuned kernel, idle plain TCP) is the most useful number on the page.
3. **A TLS edge can dominate.** The same page reports, for Caddy terminating TLS with defaults, about 127 KB per connection and 16 to 31 core-seconds per million deliveries, versus about 4.9 KB per connection for the engine, and says in-process rustls adds about 4.9 KB per connection [V, own rig, 1 GbE, loopback softirq noted as upper bound]. Our doc 02 budget is 32 KB per connection with about 10 KB for TLS [A in doc 02]. The single design choice that moves this by 10x or more is who terminates TLS and with what buffer settings, not the WebSocket crate.
4. **Held-connection ceilings reported.**
   - miraclews: 300,000 idle connections held exactly on a 2-machine rig with a 1 GbE NIC, userspace 61,243 KB over baseline for 300,224 connections [V, measured by author]. Loader capped near 310,000 established because of a NIC connection-setup-rate ceiling on that rig, not a server limit. 500k and 1M figures on that page are labeled projections by the author. Production claim on the page: 100,000 connections in 78 MB RSS on a courier gateway [V, not checkable].
   - Pushpin: 1,000,004 connections held on one server (8 vCPU, 32 GB DigitalOcean droplet class), driven by tcpkali on 4 c-16 droplets at 250,000 each with `--connect-rate 200` [C]. Transport is HTTP long-hold/SSE through a Rust connection manager plus a C++ handler, not WebSocket, mostly quiet connections, and the README says WebSocket performance was still to be improved [C]. Kernel tuning listed [C]: `fs.file-max` and `fs.nr_open` 1,010,000; `ip_local_port_range` 10240 to 65000; `tcp_mem` 10000000 x3; `tcp_rmem` and `tcp_wmem` 1024 4096 16384; `rmem_max` and `wmem_max` 16384; systemd `LimitNOFILE=1010000`; `client_buffer_size=4096`; several listen ports to avoid ephemeral-port exhaustion in the test.
   - Genusify claims 1,000,000 connections on a 4-core server with Tokio [V]; the page gives no memory, hardware model, method, date or per-connection figure. Not usable as evidence.
5. **What bounded these tests** (as stated or implied by the pages): memory per connection when buffers were left at defaults (lesismal); NIC connection-setup rate and ephemeral ports in the load generator (miraclews, Pushpin's extra ports); TLS terminator memory and CPU (miraclews); file descriptor limits (Pushpin). None reports being bounded by the Tokio scheduler at idle.
6. **Forum baseline.** A 2023 Rust forum answer says literally idle sockets are memory-bound, mostly kernel memory, and guesses a layer-7 proxy can do 500k to 1M [?, opinion, no data] (users.rust-lang.org, 2023-04-22).

### Unknown

- No independent (non-author) measurement of axum WebSocket upgrade per-connection memory with TLS and a 30 s ping timer. [?]
- No published figure for connection-count where tokio's timer wheel or one-task-per-socket scheduling costs show up with 30 s heartbeats at 150k to 300k connections on one node.
- fastwebsockets vs tokio-tungstenite on equal tuning with our message mix: not found in a source we opened.
- Mass-reconnect (TLS handshake storm) cost per node: no Rust-specific number found.

---

## Q2. Production Rust realtime and room-like systems

| System | What we opened | What it says | Relevance |
|---|---|---|---|
| Figma multiplayer (2018-05-02) | first-party post | TypeScript single-threaded server had unpredictable latency spikes: one slow encode locked a whole worker. They moved the performance-sensitive part into a Rust child process per document, talking over stdin and stdout, kept Node for WebSockets, serialization more than 10x faster [C]. They tried async Rust for network handlers in 2018, hit futures-API ergonomics and error-type problems, and dropped the plan to rewrite everything [C]. Two Rust compression crates had subtle correctness bugs, so they called a C library [C]. Each document lives on one specific worker; fixed workers per machine [C]. | Same shape as ours (one owner per document or room). Caveat: written before async/await was stable, so the async complaints are dated; the process-per-document isolation lesson is not. |
| Discord Read States (2020-02-04) | first-party post | Go service had latency spikes about every 2 minutes from forced GC scanning a large LRU cache [C]. Rust port had no spikes and beat Go on latency, CPU and memory; cache raised to 8 million Read States [C]. They ran on nightly Rust until async stabilised [C]. Tokio 0.2 upgrade gave CPU gains "for free" [C]. | Evidence for tail latency without a GC. It is a cache service, not a socket tier, and not a room actor. Discord kept Elixir for its core real-time services (see below). |
| Discord Rust NIF (2019-05-17) | first-party post | Member list needed a mutable sorted set at 100k+ entries. Pure Elixir best case at 250,000 items was 640 us for an insert; a Rustler NIF gave best 0.61 us and worst 3.68 us over 5,000 to 1,000,000 items [M, author benchmark]. Operations stay under 1 ms so they did not need to worry about yielding. "We can still keep our core real-time communications logic in the higher-level Elixir ... while dropping down into Rust when needed." [C] | The best primary evidence for the hybrid from the Rust side. It is CPU data-structure work, not a room log or delivery cursor. |
| Cloudflare Pingora (2022-09-14) | metadata only | "an HTTP proxy ... in-house using Rust that serves over 1 trillion requests a day" [C, page description]. | Proves a Rust multi-threaded proxy at huge request rates. Body not read; it is a stateless proxy, not stateful rooms. Everything else I remember about it is [?]. |
| Zed collab (DeepWiki, 2026-11-06) | AI-generated wiki | Central `collab` server on Axum and WebSockets, binary protobuf RPC, a `ConnectionPool`, Postgres in production via sea-orm and SQLx, SQLite in tests [C via secondary source]. Per-room and per-project async mutexes held in `DashMap` entries wrap DB transactions so "broadcasted RPC messages are strictly serialized with database state commits" [C via secondary source]. Rooms are ephemeral call sessions, channels are persistent [C]. | Closest published analogue to our shape: Rust, Axum, WebSocket, Postgres, one serialization point per room. Notably it uses a per-room lock, not an actor mailbox. Load numbers and incidents: not found. |
| Pushpin | repo | Rust connection manager rewrite enabled "millions of concurrent TCP connections" (their wording) [V]. A C++ handler needed timer-wheel work to match [C]. | Rust at the connection layer is credible; their lesson was a timer wheel instead of per-connection timers, which doc 02 already adopts. |
| Matrix homeservers (Conduit and others), Materialize, Rust game servers | not opened | Perplexity gave memory comparisons for Conduit vs Synapse but I did not open a primary source. | [?]. Nothing from these is used in the assessment. |

### Unknown

- Any first-party postmortem of a Rust room-per-actor socket tier at 100k+ connections, including what broke. Not found.
- Discord's current gateway language and internals beyond the two posts above. [?]

---

## Q3. Actor and supervision patterns in Rust

### Findings

1. **Plain Tokio actors are the documented baseline.** Alice Ryhl (Tokio maintainer) describes the task plus handle recipe: a spawned task owns the state and a cloneable handle wraps an `mpsc::Sender` [C]. Points that map directly onto doc 05 section 8:
   - Use bounded channels. `send` can wait, or use `try_send` and "simply killing the actor"; for an actor managing a `TcpStream` that cannot keep up, "you might want to just close the connection" [C]. This is our "slow subscriber overflows its bounded queue, session resyncs from its cursor" rule.
   - Shutdown is dropping all senders so `recv` returns `None` [C].
   - Cycles of bounded channels deadlock; a `oneshot` send or a `try_send` cannot be part of a deadlock cycle [C]. Our room mailbox, publisher and session queues must be drawn as a DAG.
   - Split a connection into a reader task and a writer task that only do IO and pass messages to an actor [C].
2. **Panic isolation is by task boundary, with a caveat that matters to us.** From memory and a Perplexity summary [?]: a panic in a `tokio::spawn`ed task is caught and returned as `JoinError` (`is_panic()`), only if the build unwinds; with `panic = "abort"` the process dies and no supervisor can restart the task. Doc 05 section 17 item 7 already rejects `panic = "abort"` for the server. Evidence that this is a live trap: the lesismal benchmark's release profile uses `panic = "abort"` (opened, [C]). Anyone copying a benchmark config into the server loses room-level isolation. I did not open the tokio `JoinError` docs; verify in the spike.
3. **Frameworks.** Perplexity summarised (not opened, [?]): Ractor is OTP-inspired with supervisor-child links and panic events; Kameo is Tokio-native with bounded mailboxes (default capacity 64) and link-based supervision; Actix is mature but tied to its own context; Xtra is minimal. No production-use references for Ractor or Kameo were found in pages we opened. Doc 02 D2 already rejects framework actors; nothing found contradicts that, but the evidence base on frameworks is thin, not negative.
4. **Restart is reconstruction.** For us this is the rehydrate path: restart reads snapshot plus log tail. The sans-IO `room-core` makes this clean [E]. The supervisor must also count restarts and back off, otherwise a poisoned log entry crash-loops a room [E].
5. **Room-per-actor with eviction.** Not found as a documented Rust pattern in sources we opened beyond the Zed per-room-lock model and Figma's fixed worker assignment. Our directory plus epoch fencing (doc 02 D5) is more careful than either; it is unproven in Rust by anything we read.

### Unknown

- Real restart behaviour of Ractor or Kameo under panic in `handle`, and mailbox-full semantics, from primary docs.
- A published eviction/rehydration design for a few hundred thousand idle rooms in Rust.

---

## Q4. Cooperative scheduling hazards, honest comparison with BEAM

### Findings

1. **The rule and the number.** "Async code should never spend a long time without reaching an `.await`." Rule of thumb: no more than 10 to 100 microseconds between awaits, depending on the application [C, Ryhl 2020, revised 2021]. The multi-thread runtime defaults to one worker per CPU core, typically about 8, so a few stuck tasks stall everything and you can miss it in local testing [C].
2. **Where blocking goes.** Ryhl's cheat sheet [C]: `spawn_blocking` suits synchronous IO (pool limit about 500 threads) and is "suboptimal" for CPU-bound work; `rayon` suits CPU-bound work, never synchronous IO, and you must not block waiting for it (use a `oneshot`); a dedicated thread suits anything that runs forever. Doc 05's "bounded blocking pool" for heavy CPU should be a rayon-style or dedicated pool with a few threads, not Tokio's default blocking pool, per this guidance. Better still, our design already puts heavy CPU in separate worker pools (task statement), which is the stronger isolation.
3. **Tokio's coop budget is opt-in and limited.** Official docs [C]: it works through "explicit yield points in a number of library functions"; a future that is always ready can starve others; "Since Rust does not have a runtime, it is difficult to forcibly preempt a long-running task." `unconstrained` disables it and "exposes your service to starvation". It does not help with plain synchronous compute inside a poll [C by implication; the doc describes only library yield points]. `consume_budget` exists for loops you write.
4. **A recent practitioner example of the failure.** The 2026 AI-generated Campfire rewrite in Rust ran some database operations as blocking calls inside async tasks and held sync locks across slow work, which the author explains stalls every task on that worker thread [C, Sarnacki 2026-10-11]. The author's point: model-written code does not reliably know this rule.
5. **Detection tooling.** Named in the Perplexity summary and not opened [?]: tokio-console (per-task busy and scheduled time), `tokio-metrics` (`busy_duration`, `budget_forced_yield_count`), `RuntimeMetrics` (worker busy duration), and the unstable `tokio_unstable` flag requirement for some. Verify before relying on them; doc 05 section 17 item 8 lists tracing and profiles but not these.
6. **BEAM comparison, honestly.**
   - BEAM preempts by reduction count, so no Erlang or Elixir process can monopolise a scheduler through ordinary code [?, well known]. One HN commenter describes it as yielding about every 1,000 function calls [C that the comment says so; the true budget is an implementation detail I have not verified].
   - BEAM is not immune. NIFs "have to be extremely fast (< 1ms) because the VM can't preempt them" and dirty schedulers exist for longer work [C, HN commenter quoting Rustler docs, 2023-04-13]. Discord's NIF stayed under 1 ms for that reason [C].
   - Rust's trade is the reverse: more raw speed, no GC, but the discipline is a code-review and tooling problem. Under our workload (small rooms, sparse events, heavy CPU already in other pools) the exposure is mainly: TLS handshake storms, large snapshot encode and zstd, JSON/CBOR encode of a 64 to 256 seat roster, and any accidental sync call (logging to a pipe, `std::sync::Mutex` held across work, sync DNS) [E].
   - One independent data point where a Rust server dropped events while an Elixir one did not (Sarnacki, 2026-10-11) turned out to be a bounded-channel setting, not the scheduler: Rust version used a `tokio::sync::broadcast` capacity of 256 and got a 1% delivery rate in a closed-loop benchmark; raising capacity to 16384 moved delivery to about 90% but raised worst-case latency from about 11 s to over 130 s [M, author rerun, 100 requests per second]. Under constant arrival rate, about 14% of events reached clients in Rust and about 60% in Elixir, with Elixir timing out about 23% of POSTs, worst latency about 180 s, and 1.8 GB of memory [M, same author]. Lesson: lagging-receiver semantics are a design decision in both stacks. Our design (bounded queue, drop, resync from cursor) makes the "lost" events recoverable, which neither benchmark did.

### Unknown

- No source we opened gives tail-latency numbers for Tokio under mixed TLS handshake plus compute load on one node.
- tokio-console's overhead in production: not found.

---

## Q5. Cross-node routing, pub/sub and presence in Rust

### Findings

1. **Membership is a library; presence is not.** Chitchat (Quickwit) implements Scuttlebutt anti-entropy gossip plus a phi-accrual failure detector [C]. Parameters stated [C]: gossip every second to 3 randomly chosen nodes (plus a seed and a dead node sometimes), failure detector window 1,000 heartbeat intervals, phi threshold 8.0 (paper suggests 8.0 to 12.0). Each node writes only its own key-value namespace [C]. Problems they had to solve [C]: a restarted node must be a new identity (`{node_unique_id}/{generation}` with a start timestamp), the node list grows and needs garbage collection, clock skew can reuse a generation, and dead nodes can be falsely resurrected by indirect heartbeats so they stopped gossiping about dead nodes. Chitchat gives liveness of nodes and a small key-value map; it does not give per-user join/leave diffs, multi-device merge or presence TTLs. Foca (SWIM) is the other option [?, not opened].
2. **What Phoenix.PubSub/Presence provide that we would build.** [?, from memory] a CRDT-based presence with per-session metas and join/leave diffs, replicated through the BEAM distribution; a process-group pub/sub. In Rust we would build: node discovery and failure detection (Chitchat), a room directory with epoch fencing (already designed, doc 02 D5), cross-node fan-out transport (gRPC streams, doc 05 section 5, or NATS), and a presence data structure with leases (Constellation's presence is heartbeat-driven per doc 09, so a lease table in the room owner may suffice).
3. **Important shape point.** Our rooms have one owner and presence for a room lives at that owner (doc 09, doc 10), so we do not need a cluster-wide CRDT presence. We need: locate the owner (directory lookup), route a session to it (edge to room node stream), and rebuild presence on failover from heartbeats [E]. That is much less than Phoenix.Presence.
4. **Zed and Figma precedent.** Both use a single serialization point per room or document on one node with no cluster-wide presence layer [C, from the pages above]. Figma pins documents to workers; Zed uses per-room locks.
5. **Durable-Objects-like designs.** Doc 02 D1 already references Cloudflare Durable Objects. workers-rs was not opened; nothing from it is used here. [?]

### Unknown

- Real cost, in engineer-weeks, of the directory plus fencing plus edge routing plus failover in Rust. No published number found. Our own build-order estimate lives in doc 05 section 20.
- gRPC bidirectional stream behaviour with 100k logical streams multiplexed per node pair: not found in a source we opened.

---

## Q6. Operations: draining, reconnect storms, profiling, async pain

### Findings

1. **Graceful shutdown and draining.** I did not open the Tokio shutdown topic page. From the actor article: shutdown by dropping senders and `recv` returning `None` [C]. For a rolling deploy we need more: stop accepting, send a "moving, reconnect to X after jittered delay" frame, flush logs, release room epochs. All [E], design work, not library features.
2. **Reconnect storms.** Pushpin's test ramps at 200 connections per second per client machine [C], which shows how long a test fleet takes to fill; real clients reconnect together. Miraclews found its rig's setup-rate ceiling (about 310,000 established) was a NIC limit, not the server [V]. Doc 02 already plans a hydration token bucket and TLS handshake CPU as the limit; no Rust-specific storm data found.
3. **Profiling and memory.** jemalloc heap profiling and fragmentation guidance came back from Perplexity only [?]. The one thing I can say from opened evidence: the lesismal README showed allocator-adjacent buffer behaviour (zero-fill of 128 KiB per read) dominating CPU and memory, which a counting allocator in `sim` (doc 05 section 17 item 3) would have caught [E].
4. **Cancellation safety is the Rust-specific correctness hazard.** `tokio::select!` drops the losing futures. `read_exact` and `write_all` are documented as not cancellation-safe; `mpsc::recv`, `oneshot`, `Mutex::lock` and `Interval::tick` are safe [C, Dixon 2026-07-23, quoting Tokio's docs]. A keepalive branch winning a race against a half-read frame desyncs the byte stream "only under load, and only sometimes" [C]. Fixes [C]: keep one pinned future across iterations, or use `tokio_util::codec` `FramedRead` whose buffer lives on the framed value. For us: the session task (heartbeat timer plus reader plus writer plus shutdown) is exactly this shape. Use the tungstenite/axum message-level API (which buffers frames internally) and forbid `read_exact` in session code; add a deterministic simulator test that injects timer wins at every byte boundary [E].
5. **Async Rust friction, as reported.** Figma in 2018 dropped async Rust for networking because of futures ergonomics and long errors [C, dated]. Discord ran nightly Rust for async until stabilisation [C, 2019 to 2020]. Both predate stable async/await. Newer opened evidence about Send bounds or compile times: not found. I will not claim any number for compile times.
6. **AI-assisted development.** The only opened evidence is the 2026 Campfire rewrite comparison: the Rust version "dropped the CSRF token", replaced Redis with in-process queues, had blocking DB calls in async tasks and sync locks held across slow work, and the author concludes that when a prompt is not specific, "many decisions are a coin flip" [C, Sarnacki, 2026-10-11]. That is a single anecdote about generated Rust and generated Elixir alike; the Elixir version had a single process serialising all SQL reads [C]. It supports "specify the constraints and review the concurrency code" for either stack, not a verdict on either language.
7. **Deploy on one small box.** Rung 1 is one box plus standby (doc 05 section 14). Nothing opened contradicts a single Rust binary here; Pushpin's c1m guide needed 8 vCPU and 32 GB for 1M connections [C], far beyond our 75k [E]. Static binary, no VM, no cluster daemon is an advantage over BEAM at this rung [E, my judgement].

### Unknown

- Crash reporting practice for Rust servers (panic hook to Sentry, core dumps): not researched in an opened source.
- tokio-console in production at scale: not found.

---

## Q7. Migrations and hybrids (seen from the Rust side)

### Findings

1. **Hybrid, evidenced.** Discord kept Elixir for core real-time and used a Rustler NIF for one hot data structure; benchmarks in the table above, operations under 1 ms so no yielding concerns [C]. Figma kept Node for sockets and ran a Rust child process per document over stdin and stdout [C]. Both are "keep the existing socket tier, put the hot logic in Rust".
2. **NIF risk, in practitioners' words.** HN 2023-04-13 commenters: NIFs run embedded in the BEAM, "always a risk to deploy"; ordinary NIFs must return in under about 1 ms; Rustler's dirty-CPU flag moves long work to dirty schedulers [C, comments]. Discord's own post says Rustler makes a NIF "guaranteed not to crash the VM or leak memory" [C, their claim]. These two statements are in tension; a panic or a bug in unsafe code inside a NIF is a VM-level event, whereas a panic in a Tokio task is a task-level event [E, my reading].
3. **Direction of migration.** I found no first-party account of a team moving a room-style socket tier from Elixir to Rust or from Rust to Elixir. The Discord Go to Rust post (2020) is a cache service. The 2026 Campfire experiment is a one-off comparison whose author says the two rewrites were not functionally equivalent. The HN thread has opinions only (including one that a preemptive Go scheduler reduced the need for Elixir, [C that a commenter said it, not evidence]). What practitioners regret: not found in a source we opened.
4. **Sidecar from the Rust side.** Figma's stdin/stdout child process is the sidecar precedent, and they chose it because Rust's low memory made a process per document affordable [C].
5. **What a hybrid would cost us** [E, judgement]: a second toolchain and deploy artifact; the sans-IO leaf crates are shared with the client SDK, WASM and conformance suite (doc 05), so the room state machine would need to cross a NIF or port boundary and hydrate state from the log on every restart; Rust-side supervision and Postgres writer would still be needed for the log writer.

### Unknown

- Any measured NIF boundary cost for passing a 64 to 256 seat room state or event batch. Not found.
- Regret reports for BEAM-to-Rust or Rust-to-BEAM migrations of stateful socket tiers. Not found.

---

## Q8. Counter-evidence: where all-Rust realtime struggled

Opened evidence only; absence of items means absence of what I could open, not absence of problems.

1. **Defaults and cheap-looking libraries cost memory and CPU** (lesismal, 137 KB per connection at defaults [E from M]). It is a configuration trap, solved once known.
2. **A bounded channel is a policy, and the wrong constant destroys delivery** (Sarnacki: 256 vs 16384 changed delivery from 1% to 90% in the Rust rewrite, [M]). Solved for us by cursors and resync, but the bound must be chosen deliberately, with a latency budget.
3. **Cooperative scheduling is a standing hazard.** The Tokio maintainer's own guidance is a 10 to 100 us budget between awaits; the official docs concede there is no forced preemption [C]. A generated or hurried change that adds a long synchronous section stalls every room on that worker [C by Sarnacki example, E for our rooms].
4. **Cancellation bugs are silent** and appear "only under load" [C, Dixon].
5. **`panic = "abort"` in a copied release profile removes per-room isolation** [C: benchmark profile uses it; E: consequence].
6. **Library gaps early on.** Figma found two Rust compression crates with data-loss bugs in 2018 and fell back to C [C, dated; they say the bugs were fixed].
7. **Tooling gaps.** No opened source documents Rust equivalents of BEAM's live introspection (attach to a running node, inspect a process mailbox, hot code load). Doc 05 plans flight recorders and sampled spans to compensate [C from our docs]. Whether that is enough is a spike question, not evidence.
8. **Hot spots and fragmentation.** A glibc-vs-jemalloc RSS growth discussion was surfaced but not opened [?]. Our workload (few, long-lived rooms, sparse events) is not the classic churn pattern, but snapshot encode buffers and TLS handshakes are.

---

## What this means for our workload

### Strengths of all-Rust (supported by opened evidence)

1. **Connection count is not the hard part.** Our target is about 75,000 open connections at rung 1 [E, doc 02], and 150k per edge node is planned. Opened data: 1,000,004 held on 8 vCPU and 32 GB (Pushpin, HTTP/SSE, quiet connections) [C]; 300,000 idle WebSocket-like connections on a lab rig [V]; a tuned tokio-tungstenite at about 8 KB per active-echo connection [E from M]. At 75,000 connections and 16 KB each including kernel and TLS, memory is about 1.2 GB [E: 75,000 x 16 KB]; at 100x (7.5M) it is a fleet question, not a one-node question. The budget of 32 KB per connection in doc 02 has roughly 2x margin against the numbers above [E].
2. **Tail latency without a GC** is documented at Discord and at Figma (the Figma problem was a single-threaded runtime plus a GC, which Rust plus a process per document solved) [C].
3. **One language for sans-IO leaf crates, SDK, WASM and conformance** is a structural advantage of all-Rust that no opened source weakens. It is the core of doc 05's design and the main thing a hybrid would cost [E].
4. **Panic containment per room** is available with unwinding enabled [?] and fits doc 05's supervisor; Figma-style child processes are the fallback if a room's code is distrusted [C].
5. **Fan-out by pointer** (`Arc<EncodedEvent>`) and bounded per-session queues have a direct Tokio-maintainer recipe [C].
6. **Simple rung 1 deploy** (one static binary on one box) [E].

### Risks

1. **Cooperative scheduling plus small team plus AI-assisted code** is the top operational risk. The opened evidence is thin but pointed (Ryhl's rule; Sarnacki's example of generated code blocking in async). Mitigation: heavy CPU in separate pools (already designed); lints and review rule banning blocking calls in session and room code; runtime metrics wired from day one; a synthetic "poison task" test that blocks a worker for 50 ms and shows the p99 impact on other rooms [E].
2. **Cancellation and select! bugs** in the session task. Mitigation: message-level WebSocket API only; simulator tests injecting timer wins; code review checklist [E].
3. **We build what OTP gives free:** supervision with backoff and escalation, the room directory and fencing, cross-node routing, drain protocol, introspection. Chitchat's write-up lists the sort of edge cases (identity generation, dead-node resurrection, clock skew) that cost real time [C]. Our design needs less than Phoenix.Presence, because room presence lives at the room owner [E], but it is still engineering we own.
4. **TLS and buffer defaults** can silently move memory 10x to 30x (lesismal, miraclews) [C, V].
5. **Thin evidence base for Rust actor frameworks**; stay with plain Tokio tasks as doc 02 D2 chose, and reassess only if supervision code grows.
6. **Evidence is mostly vendor or author claims** for connection density. Nothing here is a measurement on our workload, with our TLS, our message mix and our hardware.

### Deal-breakers

None found. No opened source shows an all-Rust socket tier failing at or near our scale. Conditions that would turn a risk into a deal-breaker: (a) the spike shows p99 stalls from scheduling that cannot be fixed by moving work off the sockets' runtime; (b) per-room panic isolation cannot be made reliable; (c) the failover and drain protocol proves too costly for the team to build before launch.

### The hybrid, from the Rust side

- **Where the evidence supports a hybrid:** the Discord NIF post and the Figma child process show Rust used inside a non-Rust realtime stack for CPU-bound hot data structures, both under about 1 ms per operation or in a separate process [C]. That would fit our heavy compute, which already runs in separate worker pools in either option.
- **Where it does not:** none of the opened sources is about a room log with per-slot delivery cursors, and none shows a BEAM tier hosting a Rust state machine that must be rehydrated from Postgres. The sans-IO crates make that technically possible [E], but we would be the first data point.
- **Cost the hybrid adds** (judgement, [E]): two toolchains, two deploy and observability stacks, two failure models (VM crash from a NIF versus task panic), and a boundary that all room commands cross.
- **What would make the hybrid clearly better:** if the spike shows the Rust tier needs significant new code for supervision, directory, routing and drain that OTP provides, and the team lacks Rust async depth. That is a staffing and risk judgement the evidence cannot settle.

### Honest overall assessment

On the evidence I could open, all-Rust is viable for the socket and room tier at the first rung and likely at 10x, with the main costs being engineering discipline and the code we write ourselves for supervision, routing and draining. The evidence that BEAM would be materially better on tail latency or robustness for this workload is, from the Rust side, not found; the one head-to-head I opened (2026-10-11) is dominated by a bounded-channel setting and by generated-code defects, and says little about either runtime. I cannot give a confidence number; every claim above is labeled for how far it was verified.

---

## Open questions a spike must answer

Run all on a throwaway rig, not on the shared box (CLAUDE.md of the user: no heavy runs on this box); targeted, with a counting allocator and `tokio-metrics`.

1. **Memory per connection with our real stack:** axum upgrade, `read_buffer_size` and `write_buffer_size` set to 4 KiB and 0 (or equivalent), rustls in process, 30 s ping on a shared timer wheel. Measure RSS and `smaps_rollup` at 10k, 50k and 150k idle connections, then with 5% sending. Compare against doc 02's 32 KB and the 8 KB (echo, tuned) and 4.9 KB (rustls) figures above. Include the default-buffer run to confirm the 30x gap on our crate versions.
2. **Where TLS terminates:** in-process rustls versus a separate terminator at a 150k-connection node. Compare memory (the 127 KB per connection Caddy figure is a default, not a law) and CPU on a simulated reconnect storm.
3. **Scheduler isolation:** inject a 5 ms, 50 ms and 500 ms synchronous stall on one room and plot p99 and p999 for other rooms on the same runtime, with one runtime versus a separate runtime or thread for the room tier versus the edge. Confirm which metrics (busy duration, poll duration, scheduled delay) catch it.
4. **Panic isolation:** panic inside `room-core::apply`; confirm unwinding is caught at the task boundary, the room freezes, other rooms continue, and a restart with backoff rehydrates from log; confirm that a `panic = "abort"` build fails that test.
5. **Cancellation safety:** deterministic simulator injecting heartbeat and shutdown wins at every byte boundary of a frame; confirm no desync with the chosen WebSocket API.
6. **Bounded queue policy:** per-session queue of 256 under a slow reader; measure time to detect, drop, resync from cursor, and the memory ceiling, versus a 16384 queue (the Sarnacki contrast).
7. **Drain and deploy:** rolling restart of a node holding 75k connections; time to move rooms, reconnect jitter needed to keep handshake CPU under budget, and message loss (should be zero by cursor).
8. **Directory, fencing and failover:** kill a room node mid-append; time to takeover, no double-apply; the effort and line count of the routing layer we built, as input to the build-versus-OTP decision.
9. **Hybrid probe (only if the owner wants it evaluated):** a thin Elixir session tier calling `room-core` as a NIF, and as a port, with a 64 and a 256 seat room; measure per-command boundary cost, rehydrate cost and what a NIF panic does. Without this the hybrid case rests on analogy, not evidence.
10. **Allocator and long-run behaviour:** 24 h soak with glibc and with jemalloc under realistic churn; RSS drift; heap profile on a canary.

## Not found or not verified (summary)

- Pingora internals beyond one metadata line; Matrix and Materialize primary sources; Zed's own writing.
- Ractor and Kameo production references; foca, workers-rs.
- tokio-console and `tokio-metrics` pages (names only, from a search summary).
- Any independent measurement of axum WebSocket per-connection memory with TLS.
- Any first-party migration or regret story for a stateful socket tier in either direction.
- Compile-time and Send-bound pain with numbers; crash reporting practice; jemalloc production fragmentation data.
