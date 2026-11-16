# Legal notes

> Early research notes. Where `docs/design/` differs, the design docs and [`DECISIONS.md`](../design/DECISIONS.md) win. Numbers are superseded by doc 02 section 5 ([`02-architecture-and-scale.md`](../design/02-architecture-and-scale.md)).

Research notes, not legal advice. Take this to an IP and software lawyer before launch.

## What we checked in the Archipelago repo
- Root license is MIT with four copyright holders listed.
- Every sub-license sampled is MIT: a Super Metroid randomizer, SMZ3, Ocarina of Time, an SC2 bot library, Civ 6, the Factorio mod, Pokemon Red and Blue.
- The website branding and background art are marked "All rights reserved". Do not reuse.
- No trademark notice and no contributor agreement found. 346 distinct author names appear in the git log.
- Other world license files were not read (e.g. cvcotm, celeste64, ffmq). A game data file extracted from a commercial ROM exists in one world. MIT does not cure that.

## Reusing code (MIT)
Allowed: copy, modify, sell, sublicense. Condition: keep the copyright and permission notice in copies and substantial portions. A port to another language that follows their code closely is a derived work, so keep the notice. Per our reading of MIT, no permission is needed from authors.

For now this repo includes none of their code or text. If we later reuse code or run their games unmodified in a sidecar, keep their notices with the code.

## Protocol compatibility
Implementing a documented protocol so existing clients can connect is generally lawful. Not verified by a lawyer.

## Name and brand
Say "compatible with Archipelago". Do not imply affiliation. Search trademark databases for a registered mark. Do not reuse their logos or art.

## Publisher IP (the main risk)
- Fan randomizers have mostly survived by distributing no game files and being free hobby projects.
- Charging money increases exposure and attention. We recall Nintendo's 2024 settlement with the Yuzu emulator makers as an example of how it can go. **Not verified; verify.**
- Rules we set: never host or distribute ROMs, patched files, keys or decryption tools. Patching, where a game needs it, is done on the player's machine by third-party tools (D-01). Do not sell anything derived from a publisher's game.
- Anti-circumvention law can apply to tools that bypass console protections. Separate risk. Ask a lawyer.

## Studio deals (cleaner path)
The studio owns its game and can authorize the integration. Points to settle in contracts:
- Who owns what, and what asset rights we get.
- Support terms and liability limits.
- Player data and privacy law, especially children's data.
- Moderation of names and chat.
- Console platform rules for online features. Not checked.

## Our own SDK license
Apache-2.0 for anything studios embed (includes a patent grant that MIT lacks). Avoid copyleft dependencies in anything studios ship. Check licenses before bundling any emulator code.

## Gambling
No entry fees, prizes or wagers on races or tournaments.

## Questions for the lawyer
1. Does selling a hosted service for fan randomizers expose us to publisher claims, and how do we structure it?
2. Is browser-side patching of a player's own ROM acceptable?
3. Trademark: can we use "compatible with Archipelago"? Is there a registered mark?
4. Terms of service, takedown process, and safe harbor setup.
5. Studio contract template: licensing, indemnity, data protection.
6. Entity structure and any open source compliance review of our dependency tree.

## If we ever want a clean-room claim
Today we cannot say we built anything without seeing their code. Our agents have read their public repo for research. A clean-room claim needs a deliberate process, set up before the work starts. Ask a lawyer to confirm the details.

1. **Two separate groups.** An analysis group may read Archipelago's code and docs. A build group must not.
2. **Specs only cross the wall.** The analysis group writes behavior specs in plain words: what the protocol does, what each packet means, what the file format contains. No code, no copied text, no pasted snippets.
3. **A reviewer checks the specs** for leaked code or text before they reach the build group.
4. **The build group works only from the specs** and from public documentation, and keeps a log of what it consulted.
5. **Anyone who has read their code is out of the build group**, including agents. A fresh agent session with no access to the Archipelago repo or these research notes counts as a new member.
6. **Keep records:** who read what, when, and which spec each module came from.

Where this matters: the native generator and the world-format interpreter. It matters less for the room server, which only needs the published network protocol. Note that this repo's research notes cite their file paths, so they belong on the analysis side of the wall.

Even without a clean room, MIT lets us reuse code if we keep their notices. The clean room is for cases where we want independence, not because the license requires it.
