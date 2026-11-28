---
layout: default
title: "WP-095: Phantom's wizard's skull, the demo and the attempt played"
---

# WP-095: Phantom's wizard's skull: the demo and the attempt played (Gen 1)

**Status:** 🟡 In progress (2026-11-28). Fork PR [barefootbetters/scummvm#19](https://github.com/barefootbetters/scummvm/pull/19) is open.
**Phase:** 5 (a gap found while searching packets for setup-only checks)
**Depends on:** WP-077 ✅ (the room's class), WP-072 ✅ (the magic mirror room, the demo's way in and out), WP-081 ✅ (the bedroom a solve leads to)
**Companion EC:** none (inline checklist under Exit criteria)
**Pre-flight:** [pre-flight-WP-095-2026-11-28](../reference/pre-flight-WP-095-2026-11-28.md). Required: the packet modifies `engines/terraglyph/**`.
**Targets generation(s):** Gen 1, *Phantom of the Knight* only. Showdown has no such room.
**Estimated effort:** Half a day

---

## Goal

Check the wizard's skull room (`ROOM_Wizards_Skull`, class `0x42D7B0`)
against the decompile by playing it through the engine's tick and live, and fix
what differs from the original.

## Background

[WP-077](WP-077-wizards-skull.md) ported the room. Its self-test checked the
claims, the keys and the flow by name, and its live run started in the room and
watched it draw. Nothing clicked a skull, and the hand demo was "ported from the
reconciled trace" but never run: `start_room` doesn't come from the mirror room.
It was the next candidate from Jeff's search of the packets for setup-only
checks, after WP-090, WP-092, WP-093 and WP-094.

**The fork at `bfeaa1c` (source):**

- **The faces came from the wrong place.** The skulls showed the state the
  profile's `Clickables` section held, which every click moves on, including
  the demo's. The original's claim sets each skull's state from the room's
  `Skull1..4` keys (`0x401780`), which win.
- **The demo started from the wrong faces.** It played each skull's face-A anim
  once but didn't make A the clickable's state. Its clicks then turned the
  skulls on from wherever they were, so the demo ended on the combination only
  when every skull had started on A.
- **The hand and the exit were built at the claim.** The hand was a hidden
  sprite shown where it was placed; the exit was there on the demo too. The
  original claims both without building them. The demo builds the hand, 480 px
  below its rest and stopped on the last step of its lift. Only an attempt
  builds the exit.
- **The hand's lowering played forward.** The original lowers it onto each
  skull by playing `HANDLIFT` backwards (mode 4).
- **Smaller:**
  - The original reads the faces on entry and after any event.
  - It hides the cursor while the hand works.
  - It asks for the mirror room every tick once the demo is done; the fork
    asked once and released input.

## Scope

**In scope:** fork `wizards_rooms.{h,cpp}` (the skull room only) and
`wizards_check.cpp`; this packet, its pre-flight, a WP-095 note in
[scooby-exe](../formats/scooby-exe.md#phantoms-wizards-skull-and-library-wp-077),
a pointer in WP-077, and the index row.

**Out of scope:** the library (`0x42D1F0`), which the search didn't flag.

## Exit criteria

1. [x] **Self-test.** The new skull lines match text written from the decompile
   before the run. The rest matches the `bfeaa1c` (WP-094) self-test, with three
   exceptions: the per-run random keys, the reworded claim line, and the
   known-flaky arena line. No new warnings.
2. [x] **Live (session).** From the mirror room, the hand demonstrates and the
   gang returns through movie 31. From the library, clicking the skulls to the
   combination opens the wizard's bedroom through movie 6.
3. [x] **Process.** The fork PR against `scooby-engine` is open, and each claim
   carries its evidence label.

## Result

Every claim is Gen 1, Phantom. Evidence labels: **source** (the decompile),
**self-test**, **log** (the engine's debug log of a live run), **live —
session** (a scripted run this session drove). Nothing here is **live — Jeff**.

**Source.** `Scooby.exe` (SHA-256 `619EA687…AF807ED`), Ghidra 12.1.4 headless:
the class (`0x42D7B0`–`0x42E6C0`), its vtable (`0x45F4C4`), and the listings for
the anim calls' hidden arguments. Also:
- the helpers: the exit (`0x401850`), the clickable (`0x408DC0`), the state
  (`0x401780`) and the queued click (`0x401840`);
- the constants: 480 (`0x45F504`), 5.0 (`0x45EDA0`) and 0.0 (`0x45E504`).

The detail is in the WP-095 note under
[scooby-exe](../formats/scooby-exe.md#phantoms-wizards-skull-and-library-wp-077).
WP-077's account holds, and its strcmp polarity is confirmed: the demo is
entered from the mirror room.

**The port (source).** On `wp-095-wizards-skull` (`ab9a1d261b`). Two debug lines
help a live run:
- each skull's box on screen (level 2);
- the hand's lowerings and turns (level 2).

**Self-test (self-test).** The combination is pinned to 1, 0, 3, 2 and the
saved faces to 2, 0, 3, 1, with the profile's `Clickables` section holding `D`
for skull 1. The text of each line was written from the decompile before the run
(kept locally); all match.

- **The claim:** the skulls are clickables on their saved face; the hand and the
  exit are claimed, not built.
- **An attempt, from the library:**
  - The faces are `C A D B`, as clickables too: the keys win over the section.
  - The exit is built and there's no hand. `SkullPuzzleAttempted` is set, and
    the intro plays.
  - The first tick reads the faces, 2, 0, 3, 1, and no room follows.
- **The solve:**
  - Three clicks turn skull 1 through `D A B`, and one click turns skull 4 to
    `C`.
  - After three clicks no room follows. After the fourth settles, the bedroom
    opens through movie 6, `SkullPuzzle` is set, and the rotations read
    1, 0, 3, 2.
- **Already at the combination** on entry: the bedroom on tick 1. Play can't
  reach this, since the way back is gone once the puzzle is solved, but it's the
  original's entry read.
- **The demo, from the mirror room:**
  - The faces are `A A A A`.
  - The hand is built at depth 1, 480 px below its placement, stopped on
    `ANIM_SKULL_HANDLIFT`'s last step.
  - There's no exit; `SkullMovie` is set; input is held and the cursor hidden.
- **The demo's trace:** `lower 1, turn, lift, lower 3, turn, return, turn,
  return, turn, lift, lower 4, turn, return, turn, lift, lower rest, done`.
  - Each lowering lands on its spot, and input is held throughout.
  - Then the mirror room, through movie 31, with the faces at `B A D C`, the
    combination.
  - The rotations still read 2, 0, 3, 1, as in the original: they aren't read
    while the hand works.

The attempt line prints the exit's object name where the written text said
"built"; that's the only wording difference.

The rest of the self-test matches the WP-094 branch's run (`8bc6cf2994`, merged
as `bfeaa1c`) but for:
- the per-run random keys;
- WP-077's claim line, now `HAND skip; EXIT skip`;
- the arena pile check's level-0 line. It printed `ROOM_Brad_SPOOKY` this time
  and `-` in the baseline, the known flake.

The runtime warnings match, 18 each.

**Build (log).** Terraglyph only, in the fork worktree. No warnings in the
engine; the two shared-code warnings (gob, halestorm) were there before.

**Live (live — session, with log).** Both runs started from a test profile:
the combination 1, 0, 3, 2, the faces 2, 0, 3, 1, `IntroDone` clear, and
`Clickables` holding `D` for skull 1. Its `StartRoom` is the skull room, and
`start_room` named the room to come from.

- **From the mirror room:**
  - The log: `the hand demo (from ROOM_P08_Magic_Mirror_Room), combination 1 0 3 2`.
  - The hand lowered at skulls 1, 3 and 4 and turned them 1, 3 and 2 times; each
    turn logged `Skull_Slide`. Then it lowered at its rest, movie 31 played,
    and the mirror room came back, about 12 s in all.
  - Screenshots: at the start the hand rises from the bottom edge, all four
    skulls face front, and there's no exit icon. Mid-run the hand is on skull 3,
    with skull 1 turned.
- **From the library:**
  - The log: `an attempt (from ROOM_P25_Wizards_Library), faces 2 0 3 1`, then
    the intro `Wizards_Library.300D_`.
  - The script clicked skull 1 three times (`C→D`, `D→A`, `A→B`) and skull 4
    once (`B→C`), and the log read `solved -> the wizard's bedroom`.
  - Movie 6 (20.7 s) played, then `ROOM_P24_Wizards_Bedroom`.
  - The screenshot after the intro shows the faces `C A D B`, the exit icon and
    the cursor.

**Deviations:**

- **The hand and the exit are at the root.** The original passes them no
  parent (`0x4314C0` with 0); the fork hangs them on its root sprite.
- **The hand isn't hit-tested.** It's built with no hit type. In the original,
  input is held while it shows.

## Deliverables

- Fork: `wp-095-wizards-skull`, PR [barefootbetters/scummvm#19](https://github.com/barefootbetters/scummvm/pull/19)
  (`ab9a1d261b`). Staged by name: `wizards_rooms.{h,cpp}` and `wizards_check.cpp`.
- Docs: this packet, its pre-flight, the WP-095 note in scooby-exe, the pointer
  in WP-077, and the index row. The WP-094 index row now also records its merge.
- Local only: the decompile dumps, the test profile, the self-test and live
  logs, and the screenshots.

## Notes

- **The library is untouched.** Its spell-book reveal and taunts are as WP-077
  left them, checked by name only.
- **The packet search's last named candidate** is the hedge maze
  ([WP-074](WP-074-maze.md)), whose navigation is modelled.
