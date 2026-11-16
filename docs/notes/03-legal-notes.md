# Legal notes

Research notes, not legal advice. Take this to an IP and software lawyer before launch.

## What we checked in the Archipelago repo
- Root license is MIT with four copyright holders listed.
- Every sub-license sampled is MIT: a Super Metroid randomizer, SMZ3, Ocarina of Time, an SC2 bot library, Civ 6, the Factorio mod, Pokemon Red and Blue.
- The website branding and background art are marked "All rights reserved". Do not reuse.
- No trademark notice and no contributor agreement found. 346 distinct author names appear in the git log.
- Other world license files were not read (e.g. cvcotm, celeste64, ffmq). A game data file extracted from a commercial ROM exists in one world. MIT does not cure that.

## Reusing code (MIT)
Allowed: copy, modify, sell, sublicense. Condition: keep the copyright and permission notice in copies and substantial portions. A port to another language that follows their code closely is a derived work, so keep the notice. Per our reading of MIT, no permission is needed from authors.

For now the plan is to copy nothing. If we later run their games unmodified in a sidecar, keep their notices with the code.

## Protocol compatibility
Implementing a documented protocol so existing clients can connect is generally lawful. Not verified by a lawyer.

## Name and brand
Say "compatible with Archipelago". Do not imply affiliation. Search trademark databases for a registered mark. Do not reuse their logos or art.

## Publisher IP (the main risk)
- Fan randomizers have mostly survived by distributing no game files and being free hobby projects.
- Charging money increases exposure and attention. I recall Nintendo's 2024 settlement with the Yuzu emulator makers as an example of how it can go. **Verify.**
- Rules we set: never host or distribute ROMs, patched files, keys or decryption tools. Patch on the player's machine. Do not sell anything derived from a publisher's game.
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
