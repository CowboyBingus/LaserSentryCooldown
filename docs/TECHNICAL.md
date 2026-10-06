# Laser Sentry Cooldown: implementation and validation

Steam build 25480438 (EXE 1.8.46015.0), game.dll SHA-256 `2E2C3B7C...C718F51E`, EXE SHA-256 `F5FEE03D...0D5F06`.
Both are verified (through Bingus Shared Runtime's session-wide hash cache) before anything is read.

## Why the Laser Sentry is lost at max heat

The Laser Sentry (`content/fac_helldivers/hellpod/laser_cannon_turret/laser_cannon_turret`,
`0x56070F36CFFFA8A8`) has a `WeaponHeatComponent` record of 592 bytes. Two of its entries lose it at max heat:
the cooling rule below and the overheat ability (next section). Its relevant values in this build:

| Offset | Field | Laser Sentry | Quasar Cannon |
| --- | --- | --- | --- |
| 0x54 / 0x5C | spare heat sinks at deployment / maximum | 0 / 0 | 0 / 0 |
| 0x60 | overheat temperature | 250 | 100 |
| 0x64 | recover temperature | 0 | 0 |
| 0x78 / 0x74 | heat gain per second firing / per shot | 8 / 2 | 0 / 100 |
| 0x80 | cooling per second | 5 | 6.66 |
| 0x8C | cooling per second while overheated | 400 | 6.66 |
| 0x90 | needs a new heat sink after an overheat | **1** | **0** |
| 0x248 | overheat ability | **2866** (an explosion at the sentry) | its own (not traced) |

The engine's temperature update (`game.dll+0x762F60`, run by the peer that owns the entity) cools an overheated
weapon at the 0x8C rate and clears its overheated flag at the 0x64 temperature, **unless** 0x90 is set: then it
neither cools nor recovers until a new heat sink is inserted. A heat weapon "has rounds" exactly while it is not
overheated (`0x744C20` -> `0x764EE0`), so with 0x90 set and no heat sink left, an overheated Laser Sentry is out
of ammunition for good.

### The overheat ability

What actually destroys the sentry is the explosion its overheat ability sets off, about 0.8 s after the overheat.
v1.0 changed only the cooling rule above, and a sentry that overheated in a mission still exploded. Traced
statically in this build's `game.dll`:

1. The WeaponHeat presentation `0x763780` runs for every heat entity on every machine. On the rising edge of the
   replicated overheated flag it plays the record's overheat sounds (0x1FC and 0x210, at the sound node in 0x1F4),
   the owner's overheat voice line (0xA8, on the owner's machine only), and, when 0x248 is not 0, the overheat
   ability through the ability player (`0x7CAF40`, mode 0: on that machine only). An overheat ability of 0 plays
   nothing.
2. Abilities are compiled step functions: `0x11509E0` switches on the ability id. Case 2866 is `0x1150660`. At
   step 0 it starts the sentry's effect `0x2057FE26`. At step 25 (about 0.8 s) it calls
   `0x11AD240(ability, 190, 0x9B115563, 0, 1, 0)`: the position of node `0x9B115563` (the record's sound node,
   0x1F4), snapped to the ground, then `0x13C0A80` queues explosion type 190 with the sentry as its source. The
   ability then ends, and its stop (`0x11506C0`) ends the effect.
3. The explosion manager (`game.dll+0x346D558`) launches overlap queries for every queued explosion (`0x13C5420`)
   and turns their hits into damage, impulses and destruction (`0x13C29E0`). Its effects, crater and decals come
   from `0x13C0D10`; the crater is only made where the source entity is owned.
4. Explosion types are rows of the game's explosion settings (`ExplosionInfo`, 152 bytes; 423 types in this
   build). Their values are not decoded for this build, but every row near 190 in filediver's August 2026
   snapshot (types 182 to 190 there, since types were added since then) is a damaging explosion with an inner
   radius of 0.4 to 35 m. The sentry sits at its centre.

No other ability in cases 2850 to 2867 of the same switch calls an explosion spawner. The other heat weapons'
overheat abilities sit there: the exported weapon data numbers them 2833 to 2844, 23 below this build, as it does
the Laser Sentry's (2843 there).

## The turret after an overheat

With the two record changes the first v1.1 test build stopped the explosion and cooled the sentry exactly as
planned (log: overheated at 250, 5 heat/s, `RESULT ... recovered` after 50.0 s), but the turret never fired
again. The sentry's AI does not read the cooldown at all. Traced statically in this build's `game.dll`:

1. AI behaviors are dispatched by behavior id: `0x472F60` (per-frame update) switches on `id - 1`, with four
   sibling dispatchers (`0x47A240`, `0x484B60`, `0x48EE50`, `0x4966D0`) for the other phases, 693 ids. The
   Laser Sentry's is **308** (306 in the 2026-08-31 data snapshot, two behaviors inserted since): its update
   is the case whose firing state (`0x32A3D0`) reads the WeaponHeat manager.
2. The update switches on the turret's state: 1 (deploying) -> 7 after 2 s; 2 and 3 idle and aware
   (`0x3267C0`, `0x327960`), 4 and 5 (`0x328AB0`, `0x3290F0`); 7 (power up) -> 8, 9 or 10, a look-around,
   -> 2 or 3; 13 firing (`0x32A3D0`). States change only through `0x32B6B0` (set_state: stores the state at
   +0, -1 at +4 and the state again at +8 of the behavior's state block, then runs the new state's enter action
   and marks it for replication).
3. The firing state reads the sentry's replicated overheated flag first: if it is set, `set_state(6)`
   (`0x32A472: mov edx, 6; jmp`). **State 6 has no update** (the switch's default, `return`): the turret
   leaves it only when something requests another state (4.), and nothing does. In the base game the overheat
   ability's explosion follows, so it never needed to. With the cooling rule the weapon recovers but the AI
   stays in state 6.
4. The state block's +4 is a **pending state request** (-1 when none). Before each behavior update the
   behavior manager (`0x843040`) applies a pending request: if it is not -1 and the behavior allows it in its
   current state (`0x4962C0`; behavior 308 refuses only in state 12), it calls `0x48EE50`, which for 308 is
   `set_state(request)`; set_state clears the request.

These mods never patch the game's code, so the fix is data: **the turret watch**. Every 120 frames it looks for
Laser Sentries whose AI runs on this machine (entity record +20: owned, bit 0, and simulated here, bit 1 clear;
the behavior manager runs a behavior only there, `0x843930`). One that is overheated is watched; once its
overheated flag has cleared, the watch finds its behavior through the behavior manager and, if the behavior is
308, still in state 6 and has no request pending, writes the request: state 7 (4 bytes at block +12). The
game applies it at the sentry's next update through its own `set_state(7)`, enter action and replication
included, and the turret's own transitions take over (7 -> 8/9/10, a look-around, -> 2/3, idle or aware).

Where things are:

- WeaponHeat manager `[game.dll + 0x3326D48]`: +20 instance count, +64 entity record pointers, +88 12-byte
  replicated states (+8 overheated).
- Behavior manager `[game.dll + 0x3326740]` (world + 0xEEBA08, stored by `0x568230`): +52 instance count, +64
  an entity id map (entries; capacity, empty key and multiplier at +72/+76/+80; slot `(k + id * multiplier) &
  (capacity - 1)`), +88 entity record pointers, +96 504-byte blocks (+0 behavior id, +8 state, +12 pending
  state request, -1 when none, +16 last state set, +488 flags).
- The watch checks before the write: the entity record is the Laser Sentry's and runs here, the map gives an
  index below the instance count whose record pointer is this sentry's, the block's behavior is 308, its state
  6 and no request pending, and the block lies in committed private read-write memory (one `VirtualQuery`).
  Anything else is logged and left alone; the addon keeps running.

## The change

Two writes into the record, inside one read-write window:

- 5 bytes at record + 0x8C: the cooling rate while overheated becomes the record's own idle cooling rate (0x80,
  5.0 per second, f32 `0x40A00000`, copied from the record, not chosen by the mod) and 0x90 becomes 0.
- 4 bytes at record + 0x248: the overheat ability 2866 becomes 0, the value of heat weapons that have none, so
  the overheat plays no ability and sets off no explosion. The overheat sounds and voice line are not part of
  the ability and still play.

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
   enabled (0x50), an idle cooling rate between 0 and 1000 per second, and the changed bytes must be vanilla in
   both places (`400.0, 1` at 0x8C and 2866 at 0x248) or this mod's in both places. A record with only one place
   changed was written by something else and is refused.

Anything else stops the mod with a log line and nothing written.

## The write

The record is in the game's weapon data library: committed private memory that the game keeps `PAGE_READONLY`
(allocated read-write). Bingus Shared Runtime's `bingus_write.lua` refuses such pages by design, so the mod uses
its own adapter, with every Windows function declared under a private name (`lsc1_*` with an `__asm__` label):

1. `VirtualQuery` on the span the two writes cover (record + 0x8C to 0x24C, 448 bytes): the region must be
   committed, private, and read-only or read-write, and the span must lie inside it.
2. Read-only: `VirtualProtectEx` to read-write for that span, one `WriteProcessMemory` per place (5 and 4 bytes;
   the bytes between are never written), `VirtualProtectEx` back to the protection it had. Read-write: the writes
   alone. A failed first write skips the second.
3. Both places are read back in one read of the span.

If the protection cannot be restored, a write fails after an earlier one landed, or the bytes do not read back,
the mod stops and puts the vanilla bytes of both places back. A pause of the update guard (an update below the mod raised) writes the vanilla bytes back the same way;
the resume, after 60 clean frames, writes the change again. A stop restores; shutdown restores nothing.

## Cost

| Path | Calls | In-game cost |
| --- | --- | --- |
| Startup (normally before the first frame) | 2 pointer reads, header, slot table and record reads, 1 `VirtualQuery`, 2 `VirtualProtectEx`, 2 `WriteProcessMemory`, 1 read-back | once per session; the query about 0.29 ms, the rest unmeasured in game |
| Weapon data not built yet (boot) | every 30th frame: up to 2 reads; other frames none | about 2-4 us per 30 frames |
| Every frame once written | a frame counter; no call, no allocation | 0 apart from the update guard |
| Turret watch, every 120th frame: menus / ship / mission, nothing overheated | 1 / 2 / 3 reads (`u64`, `view`) | about 1-6 us per 120 frames, unmeasured in game |
| Turret watch while something is overheated | + 1 read (every record pointer), + 1 read per newly overheated weapon | about 2-10 us per 120 frames, unmeasured in game |
| Turret power-up request, once per overheat of a sentry run here | 6 reads, 1 `VirtualQuery`, 1 `WriteProcessMemory` (4 bytes), 1 read-back | once per overheat; the query about 0.29 ms |
| Pause, resume, stop | as the startup write | rare |

`tests/test_cooldown.lua` and `tests/test_turret.lua` pin these counts per scenario with
`tests/frame_budget.lua`, and check that idle frames and looks allocate nothing and create no C types (also
after a JIT flush, and while a sentry cools). The watch reads into one reused 2 KB buffer (`api.view`, decoded in
place through `api.bytes` / `api.words`) and keeps its few tables after they have grown once. Only the per-frame
step, a frame counter, can be compiled: every other function of the addon, the helpers and the adapter's calls
included, is marked interpreted (`jit.off`, recursively), so none of it becomes a trace in the code cache the
game and every mod share. LuaJIT's only work for the look is one-time per session: it tries to compile the
step's look branch four times (every 10 looks), then leaves it to the interpreter with one small stub trace
(seen with `-jv`). Measured in live play with v1.0 and the frame probe of Bingus Shared Loader's development
build: 0.002 ms per frame in missions and on the ship (the update guard alone). The turret watch has not been
measured in game yet.

## Tests

`python -B scripts/build.py` runs, in the workspace LuaJIT and in the game's `lua51.dll` (`tests/game_lua.py`):

- `tests/test_cooldown.lua`: on a simulated address space seeded with the live bytes of this build
  (`tests/heat_fixture.lua`, captured read-only: the header, the slot table and the record), the slot probing,
  the record checks, the write and its call order, the boot wait, every refusal (header, missing record, short
  table, changed values, an overheat ability or cooling rule changed by something else, a record with only one
  place changed, an image, executable or unreadable page, refused or unrestorable protection, a failed first
  write, a second write failing after the first landed, a wrong read-back), a read-write page, an already-written record, the pause and resume, stop, shutdown,
  a missing loader or build, and hostile update-chain neighbours.
- `tests/test_adapter.lua` (three modes: nothing declared; the real Windows names declared first with a hostile
  prototype; declared first with the SDK prototypes): the real adapter on memory laid out like the game's, with
  the live bytes on a read-only private page at an unaligned address. It finds the record, writes both places
  and restores them with the real calls (every other byte of the record untouched), and checks the protection afterwards, the refusals on image and executable pages, and
  zero C types and garbage per call.
- `tests/test_turret.lua`: on simulated WeaponHeat and behavior managers, the reported case (a cooled sentry
  that stayed off: nothing written while it cools, a request for state 7 once on the first look after, applied
  by the simulated manager as the game does), the calls of every kind of look, sentries run on another machine,
  other heat weapons, a sentry removed while it cools, every refusal (another state, a pending request, another
  behavior or entity, an unexpected map, an image page), zero allocation, and the live test build's turret
  log. The behavior map's slots are computed with an exact 32-bit product, independently of
  the addon's arithmetic.
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

## Live play (2026-10-05 and 2026-10-06)

In play sessions with missions (v1.0 release build), the change was in place from startup to shutdown and the
sentries fired normally. The player saw a sentry "recharge" (it had not reached max heat). In a later mission a
sentry reached max heat, overheated and exploded: the overheat ability above, which v1.0 left in place.

The first v1.1 test build (overheat ability 0) was played the same evening: no explosion, the sentry cooled in
exactly 50 s and its heat state recovered (`RESULT sentry 721 recovered: overheated for 50.0 s`), but the turret
never came back online: the AI state 6 above.

The turret watch was played on 2026-10-06 (live test build, regular loader v19, a multiplayer mission). Sentry
8389524, owned here, overheated at 1065.27 s (turret state 6), cooled at 5 heat/s, `RESULT ... recovered` at
1115.28 s after 50.0 s; the watch logged `Laser Sentry 8389524 cooled down: its turret powers up again.`, the
next sample showed turret state 5 (1115.79 s) and it fired again at 1116.31 s (state 13), then fought normally
until its payload lifetime ran out (removed 182.5 s after it appeared: 180 s life plus 2.5 s retract, as in
its data). Another player's sentry (owned 0, turret state `?`) was left alone. No error, pause or stop; the
cost was not measured (no frame probe).

## Live test

The live test build (`python -B scripts/build.py --test`, `build/Laser-Sentry-Cooldown-v1.1-test.zip`, never
published) behaves exactly like the release (no options, no other change) and adds `src/test_hooks.lua`, which
only reads:

- Every 0.1 s it reads the WeaponHeat manager (`[game.dll + 0x3326D48]`) and logs each Laser Sentry when its
  temperature band (25 heat), overheated flag, firing flag or heat sinks change, when it appears and when it
  disappears, with the time since its last overheat.
- A `RESULT` line when an overheated sentry recovers (with how long it was overheated) or disappears while still
  overheated.
- Every 300 frames, whether the record still holds the change in both places.
- With each Laser Sentry line, its turret's AI state (13 firing, 6 switched off after an overheat, 7 powering
  up), or `?` when its AI runs on another machine.

Protocol, solo, any mission with enemies (with the regular Bingus Shared Loader, not the frame-probe build,
which switches mods off in 10 s blocks):

1. Deploy a Laser Sentry where it keeps firing (a large group, or a target it cannot kill) until it overheats.
2. Watch: does it stop, cool for about 50 s and fire again, or is it destroyed?
3. Send `LaserSentryCooldown.log`.

What it confirms: the sentry overheats (turret state 6), does not explode, cools for about 50 s (a `RESULT`
line saying it recovered), its turret powers up (`turret state 7`, then 8 to 10, 2 or 3) and it fires again
(firing 1, turret state 13). Still open: in multiplayer, players without the mod play the overheat ability on
their own machines (step 1 above); whether their copy of the explosion can damage the sentry was not traced (the
hits become damage events on every machine, `0x129DD30`, and where those are applied was not followed).
