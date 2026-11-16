# Constellation, in plain words

> Draft for private review. Statements about law and about Archipelago have not been reviewed by a lawyer or by the Archipelago maintainers. Facts are sourced or labeled in `notes/`.

Working name. This page needs no technical background. The technical version is in [`PITCH.md`](PITCH.md). The design documents start at [`design/README.md`](design/README.md); early research is in [`notes/`](notes/).

## What is a multiworld randomizer?

Picture a group of friends, each playing a different video game at the same time. The twist: the treasure you find in your game is secretly swapped with items from your friends' games. You open a chest in your Zelda game and find a key that your friend needs in their Pokemon game. They can't progress until you find it, and you can't progress until someone else does.

It is part game, part team puzzle. Games can take days or weeks, because everyone plays at their own pace.

## Who does this today?

One free, volunteer-built project called **Archipelago**. People love it. It supports around 80 games. The part that mixes up the items works well.

## What's the problem?

Everything around that part is hard. Getting a game started feels like setting up a home network.

- **Settings are written in a text file** with odd rules. You will see lines like `true: 50`, where "50" does not mean "true". It is a probability. Mistype a setting and it quietly gets ignored.
- **One person has to gather everyone's files** and run a program to build the game for the group.
- **You need your own copy of the game file**, in exactly the right version, and a patch file applied to it. If anything is off you get an unhelpful error.
- **Joining means typing a server address and a "port number"**, like a door number on a building. Those numbers can change (we think, not yet confirmed), and a router or firewall can block them.
- **Each game needs its own helper program**, with its own setup guide. There are about 70 guides.
- **You never hear about what happened.** The game runs for days, and nothing tells you "someone just found the item you've been waiting for."
- **Running a game for friends** means typed commands and one shared password. Nothing like a control panel. On the web host, the person who made a room is remembered by a random ID in their browser, a design that fit a volunteer project with no accounts. If that browser's cookies are cleared, the admin controls can be lost (from our reading of their public code, cited in design doc 10 section 1.2 and not yet re-checked; we do not know how any real site is set up).
- **For people who make games,** adding a game means getting it approved by volunteers and merged into one shared project.

Almost none of this is a flaw in the idea. These are design tradeoffs that suit technical users. Newcomers pay for them.

## What are we building?

A much easier way to play the same kind of game, with the same games, for the same people. Everything we build lives on the web. We do not make a desktop app, a launcher or a patching tool. Our rule is simple: the second a player has to move or install a file, we have lost.

**For players**
1. A friend sends you a link. You do not need an account to join.
2. On a web page, you pick your settings from a plain-language screen (a game that includes our toolkit can show the same screen itself), press **Connect**, and you are in. The game connects straight to the room. On a console, you type a short pairing code into the web page instead.
3. No files to write, no port numbers, and nothing from us to download or install. Other people may build helper tools that patch games, but you will not need one from us.
4. You get an **inbox**: "While you were away you got 4 items, and Sam found your Hookshot." Optional alerts on your phone or in Discord.

**For the person running the game**
- A **waiting room** (the staging lobby) with a **ready check**. Running a game needs a quick account (a passkey or a Discord sign-in, one tap), so you can sign in from any device and never lose the room by clearing your browser. Joining never needs one. You see who has joined, who has connected their game, and who is not ready yet. You press Start only when everyone is green, so nobody finds out ten minutes in that a friend never got connected.
- One screen to see who is online, who is stuck, and to fix problems with a click. Helpers can be given limited powers (a co-host who can run the game, or a moderator who can only keep order). Risky actions, like handing out every remaining item, ask you to confirm again first.
- Easy-to-share "seeds", so everyone can play the exact same game again by pasting a short code.
- A real race mode with a shared countdown.

**For game makers**
- Add this to a game with a small toolkit (an SDK), or speak our open protocol directly. Describe the game's items and rules as plain configuration data instead of writing lots of code (a starting direction, not final).
- A way to test that a game can always be finished.
- Ship updates on your own schedule.

**What stays the same**
We are not trying to replace anyone. Existing Archipelago games keep working through a compatibility layer, and their existing programs can connect as players. Admin actions stay on the web page. We want to be the easy front door to what they built.

## What is the waiting room?

Before a game starts, the host opens a web page that lists every player. Each name shows a status: invited, joined, game connected, ready. Think of a pre-flight checklist. The host starts the game only when everyone is ready, or chooses to start without the people who are not. In a multiworld game one friend who never got connected can hold up the whole group, and a host who is not technical should not have to find that out in chat. The page also shows the fix, such as "send Sam the link again" or "Sam's console needs its pairing code." Details: [`design/09-connection-and-lobby.md`](design/09-connection-and-lobby.md).

## How will it make money?

We charge for the service, never for other people's games.

- **Game studios** pay us for a supported version built into their game while they are making it (a studio integration). This is the cleanest and earliest source of income.
- **Hosts and communities** can pay for bigger, private, or longer-lasting game rooms.
- **Players and communities** can pay for extras like phone alerts and Discord tools.
- **Streamers and event runners** can pay for race tools and stream overlays.
- Small public games and the join flow stay **free**.

We will never host or sell game files. Players always use their own copy.

## What could go wrong?

- **The companies that own the games.** Free fan projects seem to have mostly been left alone (our impression, not checked). A business could draw attention. We keep the paid side to services and studio deals, and we will get a lawyer's advice before launch.
- **Upsetting the community.** If we look like we compete with the community, the people who make the games walk away. So we stay compatible, give credit, and share what we build openly.
- **No games means no players.** Running all existing games from day one is how we avoid it.
- **Cost.** Building each game for a group takes real computing power. We need to measure it and price accordingly.

## What happens first?

The smallest proof: a new server where an existing Archipelago program connects using a simple link instead of a port number. If that works, everything else is built on it. We also start small: one inexpensive server, growing in stages as more people play.

## Still to figure out

Team, funding, the first partner studio, prices and real player numbers. We have not measured or decided these yet.
