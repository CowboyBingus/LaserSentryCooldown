# v1.1

- At max heat the Laser Sentry no longer explodes: the game's overheat ability for it, an effect and then an explosion at the sentry itself, is turned off.
- v1.0 changed only the cooling rule, so a sentry that overheated was still destroyed by that explosion about 0.8 s later.
- The overheat ability is set to 0, the game's own value for weapons that have none; the overheat sounds still play.
- After cooling down its turret powers up again: at an overheat the sentry's AI switches itself off for good, so once it has cooled the mod asks the game, through the AI's own state request, for its power-up state.
- The turret check runs every 120 frames, reads memory only (1 to 3 reads while no sentry cools), allocates nothing and writes 4 bytes once per overheat; every other frame only counts.
- The record change is now 9 bytes in two places of the same heat record, written together while the page is writable once.
- Other players without the mod still play the overheat explosion on their own screens; whether it can damage the sentry there has not been tested.
- Played in a mission: an overheated Laser Sentry cooled for 50 s without exploding and fired again about 1 s later.
- Measured in live play with v1.0: 0.002 ms per frame in missions and on the ship; the turret check is not measured in game yet (about 1 to 3 memory reads every 120 frames).
- Only your own sentries are powered up again, since their AI runs on your machine.

# v1.0

- At max heat the Laser Sentry overheats and stops firing as usual, then cools at its normal rate and fires again, instead of burning out.
- No options and no new numbers: the cooldown uses the sentry's own cooling rate (5 heat/s, about 50 s from max heat on a normal planet).
- One 5-byte change to the Laser Sentry's heat record when the game starts; no work per frame afterwards.
- The record and its page are checked before the write, and the page's read-only protection is put back at once.
- An error in the game's update or another mod restores the original values until 60 clean frames have passed.
- Steam build 25480438 only; requires Bingus Shared Loader v18 or newer.
- Played in missions with the change in place throughout; an overheat followed by the cooldown has not been captured in a log yet.
- Measured in live play: 0.002 ms per frame in missions and on the ship.
- Licensed under the Zero-Clause BSD license (0BSD).
