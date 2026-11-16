# Constellation: the one-page version

> Draft for private review. Statements about law and about Archipelago have not been reviewed by a lawyer or by the Archipelago maintainers. Facts are sourced or labeled in `notes/`.

Working name. Details: [`EXPLAINER.md`](EXPLAINER.md) (plain words), [`PITCH.md`](PITCH.md) (technical) and [`design/README.md`](design/README.md) (design documents).

**The idea.** In a multiworld randomizer, friends play different video games at once, and the treasure you find in your game is secretly swapped into a friend's game. It is a team puzzle that runs for days.

**Where it is today.** A free, volunteer-built project called Archipelago runs this for around 80 games, and the item-mixing works well. Getting started is the hard part: hand-written settings files, a patch file for your own game copy, typed server addresses and port numbers, a different helper program for each game, and no word on what happened while you were away. Our rule: the second a player has to move or install a file, we have lost.

**What we're building.** The easy front door to the same games, all on the web. We build no desktop app, launcher or patcher, and no game-specific clients.
- **Players:** open a link (no account needed), pick settings on a web page, press Connect, and the game connects directly. Consoles pair with a short code typed on the web. An inbox tells you what you got and when someone finds your item.
- **Hosts:** a one-tap account, then a web lobby with a ready check, so you know everyone is connected before launch. One screen to see who's online and fix problems, shareable seeds, a real race mode.
- **Game makers:** an SDK and an open standard to add a game (worlds described as configuration data, a starting direction), plus a test suite. Third parties can build bridges and patchers to the standard.
- **Compatible:** existing Archipelago games and programs keep working, as players, through a compatibility layer.

**How we make money.** We charge for the service, never for game files.
1. Game studios pay for a supported studio integration built in during development. The earliest income.
2. Bigger, private or longer-lasting rooms.
3. Phone and Discord alerts.
4. Tools for streamers and event runners.

Small public games stay free.

**Biggest risks.** Game publishers could object to a business around fan games, so we keep paid products to services and studio deals and we will get a lawyer's advice first. Community goodwill matters, so we stay compatible and give credit. All of this is still to be measured: prices, player numbers, team and funding.

**First step.** A new server where an existing Archipelago program connects with a simple link instead of a port number.
