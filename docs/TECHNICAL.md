# Laser Sentry Cooldown: implementation and validation

Steam build 25480438 (EXE 1.8.46015.0), game.dll SHA-256 `2E2C3B7C...C718F51E`, EXE SHA-256 `F5FEE03D...0D5F06`.
Both are verified (through Bingus Shared Runtime's session-wide hash cache) before anything is read.

## Why the Laser Sentry burns out

The Laser Sentry (`content/fac_helldivers/hellpod/laser_cannon_turret/laser_cannon_turret`,
`0x56070F36CFFFA8A8`) has a `WeaponHeatComponent` record of 592 bytes. Its relevant values in this build:

| Offset | Field | Laser Sentry | Quasar Cannon |
| --- | --- | --- | --- |
| 0x54 / 0x5C | spare heat sinks at deployment / maximum | 0 / 0 | 0 / 0 |
| 0x60 | overheat temperature | 250 | 100 |
| 0x64 | recover temperature | 0 | 0 |
| 0x78 / 0x74 | heat gain per second firing / per shot | 8 / 2 | 0 / 100 |
| 0x80 | cooling per second | 5 | 6.66 |
| 0x8C | cooling per second while overheated | 400 | 6.66 |
| 0x90 | needs a new heat sink after an overheat | **1** | **0** |

The engine's temperature update (`game.dll+0x762F60`, run by the peer that owns the entity) cools an overheated
weapon at the 0x8C rate and clears its overheated flag at the 0x64 temperature, **unless** 0x90 is set: then it
neither cools nor recovers until a new heat sink is inserted. A heat weapon "has rounds" exactly while it is not
overheated (`0x744C20` -> `0x764EE0`), so with 0x90 set and no heat sink left, an overheated Laser Sentry is out
of ammunition for good and the game removes it.

## The change

One write of 5 bytes at record + 0x8C: the cooling rate while overheated becomes the record's own idle cooling
rate (0x80, 5.0 per second, f32 `0x40A00000`, copied from the record, not chosen by the mod) and 0x90 becomes 0.
The recover temperature stays 0, so the sentry cools all the way before it fires again: 250 / 5 = 50 s at the
planet multiplier 1.0 (the engine multiplies cooling by a per-weapon factor; the record's extreme heat and cold
modifiers are 0.75 and 1.5). The mod has no options and adds no number of its own. The record's own value at
0x8C (400 per second, never used in the base game because 0x90 stops all cooling) would make an overheat cost
about 0.6 s, so it is not used.

Side effect, from the same branch of `0x762F60`: the 0x8C rate applies whenever the weapon's charge is not 0
but below its firing charge (100), overheated or not, unless 0x90 is set. The beam's charge rises at 200/s
before a burst and falls at 500/s after it, so for about 0.7 s around each burst the sentry now cools at 5 per
second, where the base game does not cool it at all: up to about 3.5 heat per burst out of 250. Cooling when
idle (5 per second, charge 0) and heating while firing are unchanged. The Quasar Cannon (0x90 = 0) works the
same way.

The engine reads the record live through `0x50E1F0` on every update, so the change reaches deployed sentries at
once. An entity spawned with its own heat override (`0x767B10`, none for the Laser Sentry in this build's data)
copies the record when it spawns; the mod writes at startup, before any sentry exists.

## Finding the record

The same way the engine's lookup `0x50DD30` does:

1. `root = [game.dll + 0x346BF98]`, `table = [root + 0xF12CC8]` (two reads). Either zero: the game has not
   built its weapon data yet; the mod looks again every 30 frames.
2. The 24 bytes before the table must be `LDLD`, version 1, type `0x4C981CD9` (WeaponHeatComponentData).
3. The 58-slot table (16 bytes per slot: resource, index) is probed linearly from `resource % 58` until the
   resource or an empty slot (the Laser Sentry is in slot 10, index 11, in this build).
4. Record = `table + 928 + 592 * index`, which must lie inside the block size from the header.
5. Every value the change relies on must be vanilla: heat sinks 0 and 0, overheat 250, recover 0, overheating
   enabled (0x50), an idle cooling rate between 0 and 1000 per second, and the 5 changed bytes must be vanilla
   (`400.0, 1`) or already this mod's.

Anything else stops the mod with a log line and nothing written.

## The write

The record is in the game's weapon data library: committed private memory that the game keeps `PAGE_READONLY`
(allocated read-write). Bingus Shared Runtime's `bingus_write.lua` refuses such pages by design, so the mod uses
its own adapter, with every Windows function declared under a private name (`lsc1_*` with an `__asm__` label):

1. `VirtualQuery` on the record: the region must be committed, private, and read-only or read-write, and the
   5 bytes must lie inside it.
2. Read-only: `VirtualProtectEx` to read-write for those bytes, `WriteProcessMemory`, `VirtualProtectEx` back to
   the protection it had. Read-write: the write alone.
3. The bytes are read back.

If the protection cannot be restored, or the bytes do not read back, the mod stops and puts the vanilla bytes
back. A pause of the update guard (an update below the mod raised) writes the vanilla bytes back the same way;
the resume, after 60 clean frames, writes the change again. A stop restores; shutdown restores nothing.

## Cost

| Path | Calls | In-game cost |
| --- | --- | --- |
| Startup (normally before the first frame) | 2 pointer reads, header, slot table and record reads, 1 `VirtualQuery`, 2 `VirtualProtectEx`, 1 `WriteProcessMemory`, 1 read-back | once per session; the query about 0.29 ms, the rest unmeasured in game |
| Weapon data not built yet (boot) | every 30th frame: up to 2 reads; other frames none | about 2-4 us per 30 frames |
| Every frame once written (ship, missions, firing or not) | none | 0 apart from the update guard |
| Pause, resume, stop | as the startup write | rare |

`tests/test_cooldown.lua` pins these counts per scenario with `tests/frame_budget.lua` and checks that frames
after the write allocate nothing and create no C types (also after a JIT flush). Only the per-frame step can be
compiled; the startup, pause and stop paths are marked interpreted (`jit.off` per function). Measured in live
play with the frame probe of Bingus Shared Loader's development build: 0.002 ms per frame in missions and on
the ship (the update guard alone).

## Tests

`python -B scripts/build.py` runs, in the workspace LuaJIT and in the game's `lua51.dll` (`tests/game_lua.py`):

- `tests/test_cooldown.lua`: on a simulated address space seeded with the live bytes of this build
  (`tests/heat_fixture.lua`, captured read-only: the header, the slot table and the record), the slot probing,
  the record checks, the write and its call order, the boot wait, every refusal (header, missing record, short
  table, changed values, an image, executable or unreadable page, refused or unrestorable protection, a failed
  write, a wrong read-back), a read-write page, an already-written record, the pause and resume, stop, shutdown,
  a missing loader or build, and hostile update-chain neighbours.
- `tests/test_adapter.lua` (three modes: nothing declared; the real Windows names declared first with a hostile
  prototype; declared first with the SDK prototypes): the real adapter on memory laid out like the game's, with
  the live bytes on a read-only private page at an unaligned address. It finds the record, writes and restores
  it with the real calls, and checks the protection afterwards, the refusals on image and executable pages, and
  zero C types and garbage per call.
- `tests/test_hooks.lua`: the live test build's log (below): a sentry that overheats and recovers, one lost while
  overheated, another heat weapon that is ignored, and the record check.
- `tests/compile_entry.lua`: the assembled release and test entries compile.

## Live check on the ship (2026-10-05)

An automated session with only Bingus Shared Loader v19 and the live test build deployed, solo, Invite Only.

- The loader started the mod (`Startup finished: 2 loaded, 0 failed`). The game had not built its weapon data
  when the mod started (`Waiting for the weapon data.`); a later look found it and wrote the change within 11 s
  of the mod starting (the record check at 11.07 s found it in place), during boot, about 40 s before the ship
  loaded.
- Read from outside on the ship with a read-only memory reader: record index 11 holds
  0x8C = 5.0 and 0x90 = 0, every other field as before, and the page is private `PAGE_READONLY` again.
- In game: the mod's and the test log's update guards `running` with 0 errors; the record check found the
  change in place; `Shutdown: active`.
- The session reached the ship and quit through the engine with exit code 0; no crash dump; restore verified.

## Live play (2026-10-05)

In play sessions with missions (release build), the change was in place from startup to shutdown and the
sentries fired normally. The player saw a sentry "recharge", but no log has captured an overheat followed by the
cooldown yet; the live test build below is the way to capture one. Not covered yet either: which player's game
simulates a sentry in multiplayer.

## Live test

The live test build (`python -B scripts/build.py --test`, `build/Laser-Sentry-Cooldown-v1.0-test.zip`, never
published) behaves exactly like the release (no options, no other change) and adds `src/test_hooks.lua`, which
only reads:

- Every 0.1 s it reads the WeaponHeat manager (`[game.dll + 0x3326D48]`) and logs each Laser Sentry when its
  temperature band (25 heat), overheated flag, firing flag or heat sinks change, when it appears and when it
  disappears, with the time since its last overheat.
- A `RESULT` line when an overheated sentry recovers (with how long it was overheated) or disappears while still
  overheated.
- Every 300 frames, whether the record still holds the change.

Protocol, solo, any mission with enemies (with the regular Bingus Shared Loader, not the frame-probe build,
which switches mods off in 10 s blocks):

1. Deploy a Laser Sentry where it keeps firing (a large group, or a target it cannot kill) until it overheats.
2. Watch: does it stop, cool for about 50 s and fire again, or is it destroyed?
3. Send `LaserSentryCooldown.log`.

Open question this answers: whether the sentry's own out-of-ammo handling (or the overheat ability the engine
plays on the sentry when it overheats, 2866 in this build) still destroys it while it cools down. The heat
rule itself is decoded; the destruction path is not. If it is destroyed, the log's timing tells which, and the
fix that stays closest to the base game is decided from there.
