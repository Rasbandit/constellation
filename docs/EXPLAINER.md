# Constellation, in plain words

Working name. This page needs no technical background. The technical version is in [`PITCH.md`](PITCH.md) and [`notes/`](notes/).

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
- **Joining means typing a server address and a "port number"**, like a door number on a building. Those numbers can change, and your internet router can block them.
- **Each game needs its own helper program**, with its own setup guide. There are about 70 guides.
- **You never hear about what happened.** The game runs for days, and nothing tells you "someone just found the item you've been waiting for."
- **Running a game for friends** means typed commands and one shared password. Nothing like a control panel.
- **For people who make games,** adding a game means getting it approved by volunteers and merged into one shared project.

Almost none of this is a flaw in the idea. It is what happens when a hobby project grows without anyone redesigning the foundation.

## What are we building?

A much easier way to play the same kind of game, with the same games, for the same people.

**For players**
1. A friend sends you a link.
2. In the game, you pick your settings from a plain-language screen, press **Connect**, and you are in.
3. No files to write, no port numbers, no patching steps for you to babysit.
4. You get an **inbox**: "While you were away you got 4 items, and Sam found your Hookshot." Optional alerts on your phone or in Discord.

**For the person running the game**
- One screen to see who is online, who is stuck, and to fix problems with a click.
- Easy-to-share "seeds", so everyone can play the exact same game again by pasting a short code.
- A real race mode with a shared countdown.

**For game makers**
- Add this to a game with a small toolkit, and describe the game's items and rules in a simple format instead of writing lots of code.
- A way to test that a game can always be finished.
- Ship updates on your own schedule.

**What stays the same**
We are not trying to replace the community. Existing Archipelago games will work on day one, and their existing programs can connect. We want to be the easy front door to what they built.

## How will it make money?

We charge for the service, never for other people's games.

- **Game studios** pay us for an official, supported version built into their game while they are making it. This is the cleanest and earliest source of income.
- **Hosts and communities** can pay for bigger, private, or longer-lasting game rooms.
- **Players and communities** can pay for extras like phone alerts and Discord tools.
- **Streamers and event runners** can pay for race tools and stream overlays.
- Small public games and the join flow stay **free**.

We will never host or sell game files. Players always use their own copy.

## What could go wrong?

- **The companies that own the games.** Free fan projects have mostly been left alone. A business could draw attention. We keep the paid side to services and official studio deals, and we will get a lawyer's advice before launch.
- **Upsetting the community.** If we look like a hostile takeover, the people who make the games walk away. So we stay compatible, give credit, and share what we build openly.
- **No games means no players.** Running all existing games from day one is how we avoid it.
- **Cost.** Building each game for a group takes real computing power. We need to measure it and price accordingly.

## What happens first?

The smallest proof: a new server where an existing Archipelago program connects using a simple link instead of a port number. If that works, everything else is built on it.

## Still to figure out

Team, funding, the first partner studio, prices and real player numbers. We have not measured or decided these yet.
