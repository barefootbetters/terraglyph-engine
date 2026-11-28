---
layout: default
title: "WP-097: Phantom's hedge maze, the original's ride"
---

# WP-097: Phantom's hedge maze: the original's ride (Gen 1)

**Status:** ✅ Done (2026-11-28). Fork PR [barefootbetters/scummvm#20](https://github.com/barefootbetters/scummvm/pull/20) is open.
**Phase:** 5 (a reconstruction found while searching packets for setup-only checks)
**Depends on:** WP-074 ✅ (the maze's class and tables), WP-075 ✅ (the centre), WP-076 ✅ (below the balcony), WP-081 ✅ (the maze map item)
**Companion EC:** none (inline checklist under Exit criteria)
**Pre-flight:** [pre-flight-WP-097-2026-11-28](../reference/pre-flight-WP-097-2026-11-28.md). Required: the packet modifies `engines/terraglyph/**`.
**Targets generation(s):** Gen 1, *Phantom of the Knight* only. Showdown has no such room.
**Estimated effort:** Half a day

---

## Goal

Port the hedge maze's navigation (`ROOM_Hedge_Maze`, class `0x42AA90`) from
the decompile, in place of WP-074's model, and prove it by riding the maze
through the engine's mouse and tick, and live.

## Background

[WP-074](WP-074-maze.md) wrote that "the navigation is modelled, not
frame-exact", and that the idle mutterings didn't play. It was the last
candidate from Jeff's search of the packets for setup-only checks, after
WP-090 and WP-092–095. (WP-096 is another session's TV-package study.)

**The fork at `6b5b02c` (source):**

- **No ways out.** A move only stepped the position and redrew the buttons.
  The fork left the maze only through what it called garden-exit cells. It had
  no way off the grid to the garden or below the balcony, and none onto the
  centre's cell. Riding off an edge left the gang in a cell whose mask is −1,
  with every side a wall, and the maze map was the only way out. The original
  picks the next room from the position (`0x42C530`).
- **No ride.** The original rides the unicycle off with its start clip, holds
  input until it has gone, and enters the maze again. The new cell lands the
  unicycle from the far side with its end clip. Coming in from a maze location,
  it enters again through movie 41, or 42 once `MazeEntered` is set.
- **The buttons and the signs.** The fork built the four buttons and the six
  path signs as clickables. In the original the buttons are exits with the
  room's handler, and the signs are hidden sprites, the cell's one shown.
- **The maze map ignored.** With the map held, the original's maze goes
  straight to the map.
- **Two transcription errors:**
  - Level 2's trap cell `(0,4)` was written 40.
  - Level 2's centre-path patch was written 32, for 29.

  The rest of the tables match the decompile. A parse of it compared all six
  tables cell by cell.
- **The mutterings** were played with `Sound::play`. In the original they are
  scenes.

## Scope

**In scope:** fork `maze_rooms.{h,cpp}` (the hedge maze; the map room is
unchanged) and `maze_check.cpp`; this packet, its pre-flight, a WP-097 note in
[scooby-exe](../formats/scooby-exe.md#phantoms-hedge-maze-and-maze-map-wp-074),
a pointer in WP-074, and the index row.

**Out of scope:**
- the pending-opening string the map route writes (`game+0x238`), which names
  no clip (as in the gallery);
- `game+0x40`, which entry clears.

## Exit criteria

1. [x] **Self-test.** The new ride lines match text written from the decompile
   before the run. The rest matches the WP-095 self-test, except for the
   per-run random keys and the two table lines this packet fixes. No new
   warnings.
2. [x] **Live (session).** Clicks through the engine's mouse ride the maze cell
   by cell and into the maze centre.
3. [x] **Process.** The fork PR against `scooby-engine` is open, and each claim
   carries its evidence label.

## Result

Every claim is Gen 1, Phantom. Evidence labels: **source** (the decompile),
**self-test**, **log** (the engine's debug log of a live run), **live —
session** (a scripted run this session drove). Nothing here is **live —
Jeff**.

**Source.** `Scooby.exe` (SHA-256 `619EA687…AF807ED`), Ghidra 12.1.4 headless:
- the class (`0x42A7D0`–`0x42C600`) and its vtable;
- the move handlers (`0x42C0F0`, `0x42C200`, `0x42C310`, `0x42C420`), which
  had to be made functions first;
- `0x42C530`, the button builder (`0x408D20`), the unicycle's landings and the
  scene player (`0x40C810`);
- the direction strings at `0x4694AC` (`L R U D`);
- the profile writer, which names `game+0x285` `MazeEntered`.

The detail is in the WP-097 note under
[scooby-exe](../formats/scooby-exe.md#phantoms-hedge-maze-and-maze-map-wp-074).
It also corrects the account's movies, which had 41 and 42 the wrong way round.

**The port (source).** On `wp-097-hedge-maze` (`146c12a343`).

**Self-test (self-test).** The text was written from the decompile before the
run (kept locally). Level 0 unless said:

- **In from the garden, the first time:** movie 41, at `(2,4)` going up. In the
  maze again, cell 24 (mask 15, path 3) shows four passages, looping, and only
  the `SW` sign. The unicycle lands (state 1) with `End_Down`, and
  `MazeEntered` is set.
- **Walls:** a step left, through the engine's mouse, reaches `(1,4)` (mask 3).
  There the back and right buttons can be hit, and the front and left can't.
  The expectation that a wall's OVERLAY isn't hit was my inference, since the
  handlers have no wall check; the run confirms it. Once landed, the unicycle
  idles with `Unicycle_Idle` playing.
- **The ride out:** front from `(2,4)` sets the garden as the next room. Input
  is held, the cursor hidden, the unicycle walking (state 0) and `Start_Down`
  playing. Once it has gone, `ROOM_P06_Garden`, with no movie. In from the
  garden again, movie 42.
- **To the centre:** left, up, up gives the maze, the maze, then
  `ROOM_P23_Maze_Center`.
- **Below the balcony:** up from `(1,0)` gives `ROOM_P27_Below_Balcony`.
- **Level 1's trap:** right from `(3,4)` gives the maze. Entered again, the
  gang goes to the garden through movie 36, reset to `(2,6)`.
- **The map held, in a slot:** from the garden, `ROOM_Maze_Map`. The map has
  left its slot and is still held.
- **The mutterings:** `Hedge_Maze.240A_` starts a scene, and no line plays.
  The text written beforehand said a line would play. That was wrong. Reading
  the original's scene player (`0x40C810`) then showed the original does the
  same: a line is said only by a character in the room, and the line ships for
  `SHAGGY`, who isn't in the maze. WP-074's "the mutterings don't play" is the
  original's behaviour.

**What failed first.** The first run printed "not hit" for every button. My
check looked for a hit only inside each button's box, and the hits lie outside
it. With the whole screen scanned, every line matched. A later run added a line
with a point on each button, for the live script.

The rest of the self-test matches the WP-095 run (`6b5b02c`), except for:
- the per-run random keys;
- level 2's patch cell, now 29;
- level 2's garden-exit list, now `41,4,58`.

The runtime warnings match, 18 each.

**Build (log).** Terraglyph only, in the fork worktree. No warnings in the
engine; the two shared-code warnings (gob, halestorm) were there before.

**Live (live — session, with log).** `start_room=ROOM_Hedge_Maze`, level 0,
starting at `(2,4)`.

- The script clicked the left button at a point the self-test found on it:
  - The log reads `OBJ_HEDGE_MAZE_L to (1,4), then ROOM_Hedge_Maze`, then
    `ANIM_UNICYCLE_STARTWALK_L1`.
  - In the maze again: `ANIM_UNICYCLE_ENDWALK_R1`, then
    `in cell (1,4)=14 mask 3 path 2`, then the idle.
- Two clicks on the back button:
  - `(1,3)`, where the unicycle walked up and landed from `D`;
  - `OBJ_HEDGE_MAZE_BACK to (1,2), then ROOM_P23_Maze_Center`, and the centre
    loaded.
- **The screenshots:**
  - At `(2,4)`, passages on all four sides and the stone path running down.
  - After the left move, at `(1,4)`, hedge walls close the left and the bottom.
- A first live attempt used guessed coordinates. One click missed, and one hit
  the unicycle, but its single back click rode up and landed in `(2,3)`. A
  random muttering scene also started during it.

**Deviations:**

- **The pending-opening string** the map route writes isn't kept.
- **The rate of the mutterings:** in the original they draw on `rand()` three
  times a tick; the fork draws on its own random source.

## Deliverables

- Fork: `wp-097-hedge-maze`, PR [barefootbetters/scummvm#20](https://github.com/barefootbetters/scummvm/pull/20)
  (`146c12a343`). Staged by name: `maze_rooms.{h,cpp}` and `maze_check.cpp`.
- Docs: this packet, its pre-flight, the WP-097 note in scooby-exe, the pointer
  in WP-074, and the index row.
- Local only: the decompile dumps, the table comparison, the self-test and live
  logs, and the screenshots.

## Notes

- **The packet search is done.** Its named candidates (WFR, the skull, the
  maze) are WP-094, WP-095 and WP-097.
- **The maze map room is unchanged.** It answers its markers through
  clickables; the original uses buttons with a callback (`0x437980`). The
  rooms they lead to are the same.
