---
layout: default
title: "WP-098: The arena pile check's flake"
---

# WP-098: The arena pile check's flake (Gen 1)

**Status:** ✅ Done (2026-11-28). Fork PR [barefootbetters/scummvm#21](https://github.com/barefootbetters/scummvm/pull/21) is open.
**Phase:** 5 (self-test hygiene)
**Depends on:** WP-069 ✅ (the second arena and its pile check)
**Companion EC:** none (inline checklist under Exit criteria)
**Pre-flight:** [pre-flight-WP-098-2026-11-28](../reference/pre-flight-WP-098-2026-11-28.md). Required: the packet modifies `engines/terraglyph/**`.
**Targets generation(s):** Gen 1, *Phantom of the Knight* only.
**Estimated effort:** An hour

---

## Goal

Make the arena pile check's level-0 line deterministic. On unchanged code it
printed `to -` in some runs and `to ROOM_Brad_SPOOKY` in others. WP-094 and
WP-095 noted it as a known flake.

## The cause (log, then probe)

The check enters the second arena at each level and waits for the room's own
click on the pile to finish. It then seeds the random source with 1, clicks
the pile and waits for the room change.

- **On entry** the room queues its own click on the pile (`0x401840`) and sets
  its ignore flag. The event of a click taken while the flag is set only
  clears the flag (`0x40E5A0`).
- **In each tick**, 1 in 500, the pile rattles. The rattle queues another of
  the room's own clicks and sets the flag again.
- **The check's wait** lasted only while the pile was playing. If a rattle
  came during the entry click, its queued click was still pending when the
  wait ended. On the next tick the check's click and the queued one were taken
  as one click, the room treated it as its own, and Brad's game never started.
  The seed came after the wait, too late to rule the rattle out.

The log of a failing run (WP-094's second self-test) shows exactly this:
1. The entry click at tick 0.
2. `Brad.Armor.2` during it.
3. The pile at rest at tick 36.
4. One click at tick 37, its event ignored.
5. No room change in the 2000 ticks after.

**The game is right.** A rattle swallowing a click made while it plays is the
original's behaviour (WP-069). Only the check needed fixing. The earlier guess,
in WP-094, that the flake came from waiting on a clip's playback was wrong.

## Scope

**In scope:** fork `arena_check.cpp`, the pile check's wait; this packet, its
pre-flight, a pointer in WP-094 and the index row.

**Out of scope:** the room (`arena_rooms.cpp`), which matches the original.

## Exit criteria

1. [x] **Reproduced and fixed.** A probe seeds the random source before entry.
   Seeds that make the old wait lose the click reach Brad's game with the new
   wait.
2. [x] **Self-test.** The pile line reads `ROOM_Brad_SPOOKY` at level 0, and
   the rest matches the WP-097 run except for the per-run random keys. No new
   warnings.
3. [x] **Process.** The fork PR against `scooby-engine` is open.

## Result

Evidence labels: **log** (a self-test's debug log), **self-test**, **probe** (a
temporary check this session ran and didn't commit).

- **The fix (source).** On `wp-098-arena-pile-check` (`0a9c5c19a3`). The wait
  also lasts while the room is ignoring a click, that is, until the room has
  no click of its own left. The change is the wait's condition and its comment in `arena_check.cpp`.
- **Probe.** A temporary block tried seeds from 2 upward, set before entering
  the second arena at level 0. It ran each failing seed again with the new
  wait:
  - seeds 11, 29 and 39 printed `to -` with the old wait;
  - with the new wait, all three printed `to ROOM_Brad_SPOOKY movie -1`.

  The probe was removed before the commit.
- **Self-test.** `arena check pile: level 0: to ROOM_Brad_SPOOKY movie -1;
  level 1: to ROOM_Brad_SPOOKIER …; level 2: to ROOM_Brad_SPOOKTACULAR …`. The
  rest matches the WP-097 run (`8f6cd09`) except for the per-run random keys,
  and the warnings are 18 in both.
- **Build (log).** Terraglyph only, in the fork worktree. No warnings in the
  engine once the probe (which shadowed a local) was gone.

## Deliverables

- Fork: `wp-098-arena-pile-check`, PR [barefootbetters/scummvm#21](https://github.com/barefootbetters/scummvm/pull/21)
  (`0a9c5c19a3`). Staged by name: `arena_check.cpp`.
- Docs: this packet, its pre-flight, the pointer in WP-094, and the index row.
- Local only: the self-test logs.
