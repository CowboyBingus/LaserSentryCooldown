# Laser Sentry Cooldown v1.1

The A/LAS-98 Laser Sentry no longer blows itself up. When it reaches max heat it overheats as usual, with the
game's own overheat sounds, and stops firing. It then cools down at the same rate it already cools at between
fights, and fires again once it is cold.

The mod adds no numbers of its own and has no options: every value comes from the Laser Sentry's own data in the
game. Overheating still costs the sentry a lot. It is out of action for the full natural cooldown, about 50 seconds
from max heat on a normal planet. It just isn't lost.

**Status:** v1.0 changed only the cooling rule, and a sentry that overheated in a mission still exploded: the
game's overheat ability for the Laser Sentry, which v1.1 turns off. The first v1.1 test build then stopped the
explosion and cooled the sentry in 50 s, but its turret never came back online: its AI switches itself off at
an overheat for good. v1.1 now powers the turret up again once it has cooled. Played in a mission on
2026-10-06 with the live test build: the sentry overheated without exploding, cooled for 50.0 s and fired
again about 1 s later ([docs/TECHNICAL.md](docs/TECHNICAL.md#live-play-2026-10-05-and-2026-10-06)). The per-frame cost of the
turret watch has not been measured in game yet.

## What changes

- At max heat (250) the sentry overheats and stops firing as in the base game. It does not explode: the
  base game plays an overheat ability on it that shows an effect and, about 0.8 s later, sets off an explosion
  at the sentry itself. The mod sets that ability to 0, the value of weapons that have none.
- Instead of being destroyed, it cools at its normal rate (5 heat per second) and fires again once it is cold:
  about 50 s, longer on hot planets and shorter on cold ones, through the game's own planet modifiers.
- Its turret then powers up again. At the overheat the sentry's AI switches itself off for good (the base game
  never needed it back), so once the weapon has cooled the mod asks the game, through the AI's own state
  request, for the state it uses to power the turret up after it deploys: it looks around, then picks targets
  as usual. That happens within about 2 s of the cooldown ending (the mod looks every 120 frames).
- One side effect of the game's rule for lasers that recover: while the beam powers up or winds down (about
  0.7 s around each burst) the sentry now also cools at that normal rate, where the base game does not cool it
  at all. That is up to about 3.5 heat per burst out of 250.
- Nothing else changes: how fast it heats up while firing, how fast it cools when idle, its damage, range,
  targeting and deployment are the game's own. No other weapon is affected.

## Requirements and install

- Helldivers 2 Steam build **25480438** (EXE 1.8.46015.0). The mod checks the game.dll and EXE hashes and does
  nothing on any other build.
- [Bingus Shared Loader v18 or newer](https://github.com/CowboyBingus/BingusSharedLoader/releases/latest).

Import the release ZIP into Arsenal or HD2MM together with the loader, enable and deploy both, then restart the
game. Keep the shared loader as the winning Wwise startup replacement. The log is
`%LOCALAPPDATA%\CowboyBingus\Helldivers2\Logs\LaserSentryCooldown.log`.

## Multiplayer

The machine that owns a sentry (normally the player who called it in) decides when it overheats and recovers,
from its own data, runs its AI, and the game replicates both to everyone. A Laser Sentry owned by a player
without the mod behaves as in the base game; the mod only powers up turrets whose AI runs on its own machine.

The overheat explosion is different: every player's game plays the overheat ability on its own when a sentry
overheats. Players without the mod therefore still see and hear the explosion when a modded sentry overheats.
Whether their copy can still damage the sentry has not been tested. When every player has the mod, no game
plays it.

## How it works

The Laser Sentry's heat settings lose it at max heat in two ways. They name an overheat ability, which the game
plays the moment the sentry overheats: an effect, then an explosion at the sentry itself. And they say that after
an overheat it needs a new heat sink, which it does not have, so the engine would never cool it again and its
weapon would count as empty from then on. The engine also has the other rules. An overheat ability of 0 plays
nothing. Without the heat-sink requirement, the Quasar Cannon's rule, an overheated weapon
cools at its "overheated" cooling rate and fires again once cold. The mod sets the overheat ability to 0,
switches the Laser Sentry to that cooling rule and sets its overheated cooling rate to its own normal cooling
rate: 9 bytes in two places of one record in the game's weapon data, written once when the game starts.

The sentry's AI is a small state machine. When its weapon reports overheated, the firing state switches to a
state that has no way out: in the base game the explosion follows, so it never needed one. The mod watches for
Laser Sentries whose AI runs on this machine; when one that was overheated has cooled down and its AI is still
in that state, the mod writes the AI's pending state request (4 bytes) for the game's own power-up state,
once, and the game applies it at the sentry's next update as it applies any request. Details, offsets and
checks: [docs/TECHNICAL.md](docs/TECHNICAL.md).

The record's unused overheated cooling rate (400 per second) would have made an overheat cost less than a
second, so the mod uses the sentry's normal cooling rate instead.

The game keeps its weapon data read-only. The mod makes the record writable for its two writes only and puts
the original protection back at once, as Arc Thrower Revamped does for its record.

## Cost

Every frame: a frame counter, nothing else. Every 120th frame (about 2 s at 60 frames per second) one look for
Laser Sentries cooling down: 1 memory read in menus, 2 on the ship, 3 in a mission when no heat weapon is
overheated; one more while something is overheated, and one record read per newly overheated weapon. The look
allocates nothing and stays out of the game's shared LuaJIT code cache (interpreted). Once per overheat of your
own sentry, the turret's power-up request: a few reads, one page check and one 4-byte write. While the game is still
building its weapon data at boot, two memory reads every 30 frames. Measured in live play with v1.0 (no turret
watch): 0.002 ms per frame in missions and on the ship, the update guard alone. The turret watch has not been
measured in game yet; from the per-call costs it adds well under 0.001 ms per frame on average.

## Build and test

`python -B scripts/build.py` runs every test in a LuaJIT and in the game's own `lua51.dll`, then builds
`releases/Laser-Sentry-Cooldown-v1.1.zip`. `--test` also builds the live test build, which behaves the same and
only adds a log (never publish it). The reported cases are tests that stay: the v1.0 explosion
(`tests/test_cooldown.lua`) and the turret that never came back online (`tests/test_turret.lua`).

## License

Zero-Clause BSD (0BSD): use, copy, modify and distribute for any purpose, with no conditions. See `LICENSE`.

**AI disclosure:** Claude Opus 5.5 assisted with research, implementation, tests and documentation.
