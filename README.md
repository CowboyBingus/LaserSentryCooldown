# Laser Sentry Cooldown v1.0

The A/LAS-98 Laser Sentry no longer burns itself out. When it reaches max heat it overheats as usual, with the
game's own overheat sounds and effects, and stops firing. It then cools down at the same rate it already cools
at between fights, and fires again once it is cold.

The mod adds no numbers of its own and has no options: every value comes from the Laser Sentry's own data in the
game. Overheating still costs the sentry a lot. It is out of action for the full natural cooldown, about 50 seconds
from max heat on a normal planet. It just isn't lost.

**Status:** played in missions with the change in place throughout; an overheat followed by the cooldown has not
been captured in a log yet. The live test build captures one ([docs/TECHNICAL.md](docs/TECHNICAL.md#live-test)).

## What changes

- At max heat (250) the sentry overheats and stops firing as in the base game. Instead of being destroyed, it
  cools at its normal rate (5 heat per second) and fires again once it is cold: about 50 s, longer on hot
  planets and shorter on cold ones, through the game's own planet modifiers.
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

The game whose machine simulates a sentry decides when it overheats and recovers, from its own data. A Laser
Sentry simulated by a player without the mod behaves as in the base game. The overheat and the cooldown are
replicated by the game itself, so other players see the same sentry either way.

## How it works

The Laser Sentry's heat settings say that after an overheat it needs a new heat sink, and it has none, so the
engine never cools it again and its weapon counts as empty from then on. The engine also has the other rule,
which the Quasar Cannon uses: without the heat-sink requirement an overheated weapon cools at its "overheated"
cooling rate and fires again once cold. The mod switches the Laser Sentry to that rule and sets its overheated
cooling rate to its own normal cooling rate: 5 bytes of one record in the game's weapon data, written once when
the game starts. After that the mod does no work per frame. Details, offsets and checks:
[docs/TECHNICAL.md](docs/TECHNICAL.md).

The record's unused overheated cooling rate (400 per second) would have made an overheat cost less than a
second, so the mod uses the sentry's normal cooling rate instead.

The game keeps its weapon data read-only. The mod makes those 5 bytes writable for the write only and puts the
original protection back at once, as Arc Thrower Revamped does for its record.

## Cost

No work per frame once the change is written (normally before the first frame); while the game is still
building its weapon data at boot, two memory reads every 30 frames. Measured in live play: 0.002 ms
per frame in missions and on the ship, the update guard alone.

## Build and test

`python -B scripts/build.py` runs every test in a LuaJIT and in the game's own `lua51.dll`, then builds
`releases/Laser-Sentry-Cooldown-v1.0.zip`. `--test` also builds the live test build, which behaves the same and
only adds a log (never publish it).

## License

Zero-Clause BSD (0BSD): use, copy, modify and distribute for any purpose, with no conditions. See `LICENSE`.

**AI disclosure:** Claude Opus 5.5 assisted with research, implementation, tests and documentation.
