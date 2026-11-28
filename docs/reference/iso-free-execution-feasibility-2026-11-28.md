---
layout: default
title: "ISO-free game execution — feasibility study (2026-11-28)"
---

# ISO-free game execution: feasibility study

**Date:** 2026-11-28. **Type:** investigation (reference), not a WP. Nothing in
the engine, the Android app or the source media was changed.
**Fork inspected:** `origin/scooby-engine` at `bfeaa1c7ba` (ScummVM
`2026.4.1git`). The test binary was built from that commit (`scummvm.exe`,
"Nov 28 2026 08:10:50"; TerraGlyph-only build).

**Question.** Can TerraGlyph run the supported games from extracted game-data
files, without the original CD-ROM or its ISO image?

---

## A. Executive summary

**For Gen 1, yes, and it's verified at runtime.** *Showdown in Ghost Town*
(Aug 2001 English release) runs from a plain folder of 48 files copied out of
the ISO. Detection, startup, the title movies, the menus, typed sign-in,
gameplay with voiced scenes, and save/load all worked. The engine's full
resource self-test decodes every room, cel, sound clip and Bink movie in that
folder, with no failures. Its output is **identical line for line** to the same
self-test run on the read-only-mounted original ISO. *Phantom of the Knight*
(still `ADGF_UNSTABLE`) also boots and decodes every resource from extracted
files.

**TerraGlyph has never read ISO images.** ScummVM has no ISO 9660 reader. The
engine opens ordinary files by name through ScummVM's file layer, so a mounted
disc and a copied folder look the same to it. Nothing in the engine checks
for a CD: no volume label, no marker file, no CD audio, no disc swapping.

**Gen 2/3 is blocked, but not by the ISO.** *Jinx at the Sphinx*, *Case File
#1* and *Activity Challenge* are detected from extracted files. They don't run
because the engine has no reader for their MMFW archives yet, whatever the
source.

**Savings are modest on disk and larger over the network.** Gen 1's runtime
set is about 13% smaller than the ISO. A deflate zip of it, the format the
WP-085 app already installs, is 31–34% smaller than the ISO. Most of the data
is already-compressed audio, so compression helps only partly.

| Game | Edition (ISO) | Status | One-line evidence |
|---|---|---|---|
| *Showdown in Ghost Town* | Aug 2001, v1.2 (`SHOWDOWN`, 636,268,544 B) | **VERIFIED** | Detect, boot, menus, gameplay, audio, video, save/load from extracted files; self-test identical to mounted ISO (§F) |
| *Showdown in Ghost Town* | Feb 2001 pressing (`SHOWDOWN`, 676,673,536 B) | **BLOCKED** (detection) | "Unknown game variant": its `Scooby.exe` and `TGIFILE.ART` aren't in the detection table (§F.2). Not an ISO dependency |
| *Phantom of the Knight* | (`PHANTOM`, 621,656,064 B) | **PARTIALLY VERIFIED** | Detect, boot, title movies, Sign In, every resource decodes; gameplay and save/load not driven this session; game is `ADGF_UNSTABLE` |
| *Jinx at the Sphinx* | (`JINX`) | **BLOCKED** (no MMFW reader) | Detected from extracted root-layout files; engine refuses Gen 2 at `run()` |
| *Case File #1* | (`CASE_FILE_1`) | **BLOCKED** (no MMFW reader) | Detected from extracted root-layout files; Gen 3 refused at `run()` |
| *Activity Challenge* | (`SCOOBYDOO`) | **BLOCKED** (no MMFW reader) | Detected from extracted root files; Gen 2 refused at `run()` |
| *Case File #2* | (`CASE_FILE__2`) | **UNKNOWN** (out of scope) | Not a TerraGlyph engine ([WP-005](../work-packets/WP-005-engine-family-check.md)); not detected |
| *Case File #3* | (`SCOOBY DOO! #3`) | **UNKNOWN** (out of scope) | The image holds one 181 MB MSI installer, `Scooby-Doo!.exe`; not in the detection table |

---

## B. Current engine architecture (resource loading)

All references are to the fork at `bfeaa1c7ba`.

**Where the files come from.** ScummVM sets the game's `path` (the folder the
user added) as the search root. The engine adds a `scooby` subfolder if one
exists, so either the disc root or `scooby\` works:
`engines/terraglyph/terraglyph.cpp:62-63`
(`SearchMan.addSubDirectoryMatching(gameDataDir, "scooby")`).

**What Gen 1 opens.** Everything goes through `Common::File`, which resolves
names through `SearchMan`:

| File | Opened at | If missing |
|---|---|---|
| `TGIFILE.ART` | `terraglyph.cpp:106` → `art_archive.cpp:78` (index); `art_archive.cpp:270` reopens the file for each member stream | `kNoGameDataFoundError` |
| `object.ini` | `terraglyph.cpp:110` | `kNoGameDataFoundError` |
| `Sfx.dat`, `Voice.dat`, `Music.dat` | `sound.cpp:33`, `:47`; `sound_archive.cpp:61` (index); `:150` and `:164` reopen per clip | `kNoGameDataFoundError` |
| `BK.NNN` | `movie.cpp:42` (name), `:65` (header/size check), `:78` (`Video::BinkDecoder`) | Warning, movie skipped, game continues |

**Detection.** `detection_tables.h:44-117`. Each entry is three files, MD5 of
the first 5,000 bytes plus whole-file size. Gen 1 entries use `Scooby.exe`,
`TGIFILE.ART` and `object.ini`. `Scooby.exe` is read for detection only and is
never executed. `detection.cpp:42-52` lets detection look into `scooby`,
`jinx`, `museum` and `scripts and resources`, three levels deep.

**Answers to the brief's questions.**

| | Answer | Evidence |
|---|---|---|
| A. ScummVM version / fork | `barefootbetters/scummvm`, branch `scooby-engine`, version string `2026.4.1git` | `--version` output; `base/internal_version.h` |
| B. Engines | `terraglyph` (engine id); builds here are TerraGlyph-only | `--list-engines` |
| C. Scooby games recognised | `showdown` (EN + NB), `phantomknight`, `jinxsphinx`, `casefile1`, `activitychallenge` | `detection_tables.h:24-126` |
| D. How resources are found | ScummVM `SearchMan` over the game path plus `scooby\` | `terraglyph.cpp:62-63` |
| E. Reads ISO images directly? | **No.** The fork has no ISO 9660 reader (no such file in `common/`); a Windows ISO works only because the OS mounts it as a drive | `git ls-tree` search |
| F. Expects extracted directories? | **Yes**: a directory, whether a mounted disc or a copy | as D |
| G. Custom loaders | `ArtArchive` (a `Common::Archive`), `SoundArchive`; both read plain files | `art_archive.cpp`, `sound_archive.cpp` |
| H. Hard-coded names | Yes: the file names above and `BK.%03d`; no hard-coded **paths** or drive letters | `movie.cpp:42`, `sound.cpp:33` |
| I. OS filesystem reliance | None beyond ScummVM's own. Lookups are **case-insensitive** on every platform (`common/fs.h:353`); *Phantom*'s upper-case names (`OBJECT.INI`, `MUSIC.DAT`) load at runtime (§F) | `common/fs.h:353-354` |
| J. Windows vs Android | Same engine code. App-private paths go through `AndroidPOSIXFilesystemNode`, while picker folders go through SAF (`backends/fs/android/android-fs-factory.cpp:62,68`) | §H |

The engine's test-only config keys (`start_profile`, `test_script`,
`test_shots`, `profile_check_file`) read files outside the game path. They're
for testing and play no part in normal play.

---

## C. Game media inventory

Read-only inventory, 2026-11-28. Nothing was moved, renamed or modified. The
one ISO that was mounted (§F.3) was mounted with `-Access ReadOnly` and its
timestamp was checked unchanged afterwards.

### `C:\pcloud\scooby-engine` — the Scooby disc images

Every image is a **single-track ISO 9660 + Joliet data image**. There are no
BIN/CUE or mixed-mode images and no audio tracks, so each title is one disc.

| File | Title / edition | Volume label | Size (B) | Files (dirs) | SHA-256 |
|---|---|---|---|---|---|
| `Scooby Doo Showdown in Ghost Town.iso` | *Showdown*, Aug 2001 (title screen: v1.2) | `SHOWDOWN` | 636,268,544 | 237 (31) | `698caea433a7fce9e1f2c9c74efee2ccc6fdf4e44091f93552b4b20e495652a8` |
| `Showdown.iso` | same disc + 311,296 trailing zero bytes | `SHOWDOWN` | 636,579,840 | 237 (31) | `b9cf0618c55fae6206c3e720b4a5a421e2fd06d34ca9b167271483c8f4fddaef` (first 636,268,544 B hash = the row above) |
| `Scooby-Doo! - Showdown in Ghost Town (2001).iso` | *Showdown*, Feb 2001 pressing (DX8 patch, `NonZero.exe`) | `SHOWDOWN` | 676,673,536 | 242 (30) | `5bd9d3e8854098d39a18ab02d1d67a6e471ffd8a00baf7ec2cb3d3490b326c0a` |
| `Scooby Doo Phantom of the Knight.iso` | *Phantom* | `PHANTOM` | 621,656,064 | 252 (31) | `e70345a3a5a1e65510bdffbc40c63322d020c568d0c402051d87aaf7725719ed` |
| `Scooby Doo Jinx at the Sphinx.iso` | *Jinx* | `JINX` | 581,302,272 | 230 (31) | `1a238e30555a58806094c94f6ed0d026f1950d2bbb3cb63e038357db84904bf1` |
| `ScoobyDoo.iso` | *Activity Challenge* | `SCOOBYDOO` | 231,700,480 | 199 (31) | `fa9474359ca65539bf10074964b1a186536c8b3e97ccc38f1db17fadcbf1e628` |
| `Case File 1.iso` | *Case File #1* | `CASE_FILE_1` | 426,536,960 | 272 (41) | `4bf764fb69ae78a15343a8d4dabbec8f490644757ed3eaf45289ed875f0dcc42` |
| `Case File #2.iso` | *Case File #2* (not TerraGlyph) | `CASE_FILE__2` | 613,085,184 | 278 (27) | `261b1479f1e86f4bb6d5316884148a19296c3864878ae112f36c239c2d22e7e2` |
| `Scooby Doo! #3.iso` | *Case File #3*, 2007 MSI installer (`Scooby-Doo!.exe`, 181,897,154 B) | `SCOOBY DOO! #3` | 182,718,464 | 3 (0) | `25667131446686483f95bf25ad125087eab7ef1f45ceabb40d56569fe85979a5` |

Also in this folder:
- `wp-085-showdown-copy\scooby\`: an already-extracted *Showdown* folder, 50
  files, 556,640,312 B, used for the WP-085 TV test. Its SHA-256s match file
  for file the `scooby\` extracted in §F from the Aug 2001 ISO.
- Non-Scooby images (*102 Dalmatians* ×2, *Clue Jr*, *Little Mermaid II*) and
  the BASS freeware zip.
- Non-media folders (`verification`, `wp-085-captures`, `licensing`,
  `reference`), named here and not inspected as game media.

### `P:\games` (pCloud drive)

No Scooby media. It holds a general ScummVM test library: Humongous,
Broderbund Living Books, LucasArts, Broken Sword, Simon the Sorcerer, BASS,
LEGO Island, Magic School Bus, Dungeon Siege I/II, PopCap installers, and the
`tv-test-packages` (BASS zips, manifests, negative-test zips). None of these
uses the TerraGlyph engine, so they're out of scope and weren't hashed (about
25 GB). Most are extracted directories already, which is ScummVM's normal
input.

### Disc layout of the Gen 1 discs

```
<root>\AUTORUN.INF  Play.exe  SETUP.EXE  showdown.cd | phantom.cd (0-byte marker)
<root>\AOL\  messagemate\  INSTALL\           ← installer and bundled third-party software
<root>\scooby\TGIFILE.ART  object.ini  Scooby.eng  Scooby.exe  binkw32.dll
              Music.dat  Sfx.dat  Voice.dat  BK.NNN …
```

*Phantom* stores its names in upper case (`OBJECT.INI`, `MUSIC.DAT`, …).

---

## D. Resource dependency findings (Gen 1)

Established from the source (§B) and confirmed at runtime: the minimal set in
§F lacks `Scooby.eng` and `binkw32.dll`, and every check still passes.

| Category | *Showdown* | *Phantom* | Basis |
|---|---|---|---|
| 1. Detection | `Scooby.exe`, `TGIFILE.ART`, `object.ini` | same | `detection_tables.h` |
| 2. Launch | `TGIFILE.ART`, `object.ini`, `Sfx.dat`, `Voice.dat`, `Music.dat` (a missing one aborts with `kNoGameDataFoundError`) | same | `terraglyph.cpp:106-116` |
| 3. Gameplay | as 2 (all art and script data) | same | self-test: every block, cel and room |
| 4. Audio | `Sfx.dat` (316 clips), `Voice.dat` (821), `Music.dat` (28); PCM, no CD audio | 282 / 768 / 17 | log; clip check |
| 5. Video | `BK.NNN`, 42 files (BK.001–023, 025–042, 058) | 56 movies | movie check |
| 6. Localization | None separate: text is in the art and voice; the language is per disc (EN; NB has its own detection entry) | same | detection table |
| 7. Installation only | `INSTALL\`, `AUTORUN.INF`, `SETUP.EXE`, `Play.exe` | same | never opened |
| 8. Unneeded at runtime | `Scooby.eng` (no code opens it), `binkw32.dll` (ScummVM decodes Bink itself), `AOL\`, `messagemate\`, `*.cd` markers | same | source grep + runtime without them |

**Runtime set (Gen 1):** `Scooby.exe`, `TGIFILE.ART`, `object.ini`, the three
`.dat` files, and every `BK.NNN`. That's 48 files and 556,351,281 B for
*Showdown*, and 62 files and 540,771,169 B for *Phantom*.

**Doc drift found.** [The status page](scummvm-wiki-scooby.md#required-game-data-files)
and the [install guide](download-and-install.md#required-game-data-files) list
`Scooby.eng` as a required engine file. The engine never opens it, and both
games run without it.

Don't drop movies to save space. A missing `BK.NNN` is skipped with a
warning, so the story would silently lose its cutscenes.

---

## E. ISO dependency analysis

| Disc characteristic | Used by TerraGlyph? | Evidence |
|---|---|---|
| Volume label | No | No label API call in `engines/terraglyph/` |
| Disc identification / marker (`showdown.cd`, `phantom.cd`) | No | Not referenced; marker absent from the minimal set |
| Disc switching | No: one disc per title | §C |
| CD audio tracks | No: none on the images, and no `AudioCD` use in the engine | grep: 0 matches |
| Mixed-mode layout | No | §C |
| Filesystem paths | Only `scooby\` relative to the game path | `terraglyph.cpp:63` |
| File ordering / sector placement | No: whole files read by name and offset | loaders in §B |
| Case-sensitive names | No: ScummVM lookups are case-insensitive | `common/fs.h:353`; *Phantom* at runtime |
| Proprietary containers | `TGIFILE.ART`, `.dat` (Gen 1, readers exist); MMFW (Gen 2/3, no reader) | [formats](../formats/) |
| Disc copy protection / runtime disc checks | None in the engine. Whatever the original `Scooby.exe` checked is moot because it never runs | §B |

**Nothing about the original media needs special handling.** For Gen 2/3,
keep the **disc-root layout** in any package. Their detection files sit in
two folders (`Mummy.MMP` at the root, `Jinx\Mummy.mms`; `Scripts and
Resources\MuseumCD.MMP` at the root, `MUSEUM\Scripts and Resources\Museum.MMS`),
and detecting from `Jinx\` alone failed (§F.4).

---

## F. Extracted-file test results

Test setup: extracted with 7-Zip 7z into this session's scratch folder, which
is outside both media locations. Original names and directories were kept.
The binary was a copy of the fork build. Runs were automated with the engine's
`test_script`, plus `--logfile` and `test_shots` screenshots. Logs and shots
stay local.

### F.1 *Showdown* (Aug 2001): VERIFIED

**Data under test.** The "minimal set" is `scooby\` with `Scooby.eng` and
`binkw32.dll` removed: 48 files, 556,351,281 B. `Scooby.exe` SHA-256 is
`1328454334cddb9518168b78ba7ffeb13ba7b772e6b52f4e2b87098d18af7478`, which
matches the identity lock in `tools/exes/README` and the WP-082 verification
note.

| Check | Result | Receipt |
|---|---|---|
| Detection, full extract root, `scooby\`, minimal set root, minimal `scooby\` | ✅ `terraglyph:showdown` all four | `--detect` |
| Startup | ✅ `Running Scooby-Doo! Showdown in Ghost Town (Windows/English)`; `TGIFILE.ART: 69 rooms, 453 objects, 1788 names`; `Sfx.dat: 316 clips` / `Voice.dat: 821` / `Music.dat: 28` | t1 log |
| Video | ✅ Title movies played in real time: `movie (27) ended after 45 frames in 3676 ms`, `movie (28) ended after 120 frames in 10023 ms` | t1 log |
| Menus | ✅ Title screen (screenshot), skip to Sign In; `type TESTER` + click Play → `sign in: play as TESTER` → `ROOM_Difficulty` drawn | t1, t4 logs + shots |
| Audio | ✅ in the mixer: music, SFX and voice clips decoded from the `.dat` files and played to completion through WASAPI (`sound Sign_In.SHAGGY.Type_Name ended at 35803 ms`). **Not verified by ear**: the runs were unattended | t1, t4 logs |
| Gameplay | ✅ Town Center from a seeded profile; hover `OBJ_P01_TO_P04`, click → `exit … going to ROOM_P04_Street2`; the room class runs its three-line voiced scene `Town_Center.100A_` | t5 log |
| Save / load | ✅ `profile saved: 8770 bytes … test save 1: No error`; `load 1` → `profile read: StartRoom ROOM_P01_Town_Center … test load 1: No error`. The save lands in the savepath, not the game folder | t3 log, `sd-min.001` |
| Resource-loading errors | ✅ none: `art check: 522 blocks, 0 bad`; `cel check: 9111 cels, 0 bad`; `room check: 69 rooms, 0 bad`; `clip check: 1165 clips, 0 bad`; `movie check: 42 movies, 0 bad`; also `exit check: 71 exits, 0 stuck`, `click check: 191 clicks, 0 stuck`, `hint check: 78 hints … 0 stuck` | t2 self-test (`--debuglevel=11`) |

**Not run this session:** a continuous start-to-credits playthrough from the
extracted copy. The same play was done on a mounted ISO on 2026-11-23 (Jeff's
firsthand report). The equivalence check in F.3 is why I expect it to carry
over, but that's an inference, not a capture.

**Earlier extracted-file evidence on Android.** In WP-085, *Showdown* was
copied out of the mounted ISO, pushed over `adb` into the TV app's private
folder, added, played, saved and loaded on the TCL 65Q77K
([EC-008](../execution-checklists/EC-008-tv-app-pipeline.md), Jeff: "Showdown
added+plays", "Showdown save+load"). That copy is byte-identical to the one
tested here (§C).

### F.2 *Showdown* (Feb 2001 pressing): BLOCKED at detection

`--detect` on its extracted `scooby\` reports an unknown variant:

```
{"Scooby.exe", 0, "b594ee8a9156b0bf3b36d7e07fbfaa70", 471105},
{"TGIFILE.ART", 0, "8269b5200634a48ba367e8d115ad49c0", 144642048},
{"object.ini", 0, "42db419bd6be75a49552d9e9bdeed880", 25196},
```

`object.ini` matches the Aug 2001 disc. `Scooby.exe` and `TGIFILE.ART` differ:
this is an earlier build, before the v1.2 disc. A detection entry plus a
runtime check would decide whether the engine plays it. That's unrelated to
the ISO question, and it wasn't attempted here.

### F.3 Extracted vs mounted ISO: identical

The same `--debuglevel=11` self-test was run against the original ISO, mounted
read-only (`G:\scooby`, CDFS), and against the extracted minimal set. With
timestamps and paths removed:

- **1,940 resource-check lines are identical** (0 differences). That includes
  138 room/art MD5 lines and 43 movie lines with per-movie frame colours.
- The **sets of warning lines are identical** (104 unique lines on each). The
  warnings, such as `Failed to locate Gummy.Music`, come from the game data,
  not from the extraction.

So the engine sees the same bytes either way. Copying the files out of the
disc changes nothing it can observe.

### F.4 Other titles

| Title | Detection from extracted files | Runtime |
|---|---|---|
| *Phantom* (minimal set: 62 files, no `SCOOBY.ENG`/`BINKW32.DLL`) | ✅ root and `scooby\` | ✅ Boot: title movies 27/28, title room, Sign In, Shaggy's voice line. Self-test: `art check: 486 blocks, 0 bad`, `cel check: 5810 cels, 0 bad`, `room check: 53 rooms, 0 bad`, `clip check: 1067 clips, 0 bad`, `movie check: 56 movies, 0 bad`, 82 exits / 174 clicks / 82 hints 0 stuck. Gameplay and save/load not driven |
| *Jinx* (signature files, disc layout) | ✅ from the root; ❌ from `Jinx\` alone | Not attempted: Gen 2 returns `kUnsupportedGameidError` (`terraglyph.cpp:102`) |
| *Case File #1* (signature files, disc layout) | ✅ from the root | Not attempted: Gen 3, same refusal |
| *Activity Challenge* (root archives) | ✅ | Not attempted: Gen 2, same refusal |

---

## G. Storage comparison

**Measured** = by this session's files and zips; **derived** = arithmetic on
measured values. Sizes are in bytes, with MiB in brackets.

| | *Showdown* (Aug 2001) | *Phantom* |
|---|---|---|
| A. Original ISO (measured) | 636,268,544 (606.8) | 621,656,064 (592.9) |
| B. Total extracted, whole disc (measured) | 635,780,510 (237 files) | 621,144,502 (252 files) |
| — of which installer / third-party (`INSTALL`, `AOL`, `messagemate`) | 79,082,687 | 80,026,791 |
| — `scooby\` folder | 556,640,312 (50 files) | 541,060,200 (64 files) |
| C. Required runtime set (measured; verified §F) | **556,351,281 (530.6)**, 48 files | **540,771,169 (515.7)**, 62 files |
| D. Package, zip deflate -mx9 (measured) | **419,557,579 (400.1)** | **430,195,502 (410.3)** |
| D′. zip store, no compression (measured) | 556,358,093 | — |
| D″. 7z LZMA2 -mx9 (measured; the app can't open it today) | 348,681,016 (332.5) | — |
| E. Installed size (= C, derived) | 556,351,281 | 540,771,169 |
| F. Temporary space during install (zip + staging extract, derived) | 975,908,860 (930.7) | 970,966,671 (926.0) |
| G. Peak during a version **update** (old + zip + new, derived) | 1,532,260,141 (1,461.3) | 1,511,737,840 (1,441.7) |
| **Storage reduction** (A − C) / A | **12.56%** | **13.01%** |
| Download reduction vs ISO (A − D) / A | 34.06% | 30.80% |
| Zip saving vs runtime set (C − D) / C | 24.59% | 20.45% |

Per-file deflate (*Showdown*, measured): `TGIFILE.ART` saves 63.9%,
`Sfx.dat` 29.5%, `Voice.dat` 18.9%, `Music.dat` 8.0%, `BK.001` (Bink) 1.9%,
`object.ini` 87.4%. The art compresses well. The PCM music and the Bink video
barely do, as expected.

Gen 2/3, whole-disc extracted sizes only (a runtime set can't be established
without the readers): *Jinx* 580,818,802; *Case File #1* 425,649,238;
*Activity Challenge* 231,250,932; *Showdown* Feb 2001 675,329,302 (its
`scooby\` is 554,560,840).

F and G follow from the WP-085 installer
(`backends/platform/android/advanced/.../PackageStore.java:170-211`). It
downloads the zip to `packages/.staging/`, extracts beside it, renames the
folder into `packages/<id>/<version>/` (a rename, so no second copy), and
deletes the zip. An update keeps the old version until the new one is in
place.

---

## H. Google TV compatibility

| Topic | Finding | Basis |
|---|---|---|
| Storage location | `getFilesDir()/packages/<id>/<version>/`: app-private, kept across app updates, deleted on uninstall | [manifest doc](tv-package-manifest.md); `PackageStore.java:35` |
| Filesystem access | App-private paths are plain POSIX for ScummVM (`AndroidPOSIXFilesystemNode`), with no SAF permission needed | `android-fs-factory.cpp:68` |
| Loading from app-private storage | **Verified for *Showdown*** on the TCL (WP-085, adb-pushed extracted copy) | EC-008 |
| Case sensitivity | Not a risk: ScummVM's lookups ignore case on Android too | `common/fs.h:353` |
| Performance | The engine reopens its archive file for every art member and sound clip (`art_archive.cpp:270`, `sound_archive.cpp:150`). That's cheap on a POSIX path. Through SAF (a user-picked folder) each open is a JNI/ContentResolver call: **expected** slower, **not measured** | inference |
| Save games | `<ScummVM base>/saves` (`android.cpp:422`), separate from packages. Removing a package keeps the saves (verified in WP-085, BASS) | WP-085 §Result |
| Install / remove / update | Built and verified in WP-085 with BASS: SHA-256 check, data-only extraction, update beside the old version, interrupted update leaves the old one playable | WP-085 Milestones 3–4 |
| Available-storage reporting | **Gap:** the installer doesn't check free space before downloading, and *Showdown* needs ~931 MiB free to install and ~1.46 GiB to update | `PackageStore.java` (no `StatFs`/`getUsableSpace`) |
| Temporary download storage | Inside `getFilesDir()` (`packages/.staging/`), counted against the same internal storage | `PackageStore.java:176-179` |
| Interrupted downloads | **Gap:** no HTTP `Range` resume. A restart re-downloads all 400 MiB (`cleanStaging()` at `:176`) | source |
| Package contents rule | The extractor refuses `.so`/`.dex`/ELF/dex-magic files. *Showdown*'s set includes `Scooby.exe` (a Windows PE, detection only), which the rule allows | [manifest doc](tv-package-manifest.md) |

**Android-specific changes needed for licensed beta packages:** a free-space
pre-check, a Range-resume download, and an authenticated URL scheme (§I). No
engine change is needed.

---

## I. Cloudflare R2 packaging considerations

**What exists.** The WP-085 manifest (format 1) has an id, version, exact
size, SHA-256 and any HTTPS URL. One zip per game, installed per id/version
folder, with removal that keeps saves. The downloader holds no host, so R2 is
a manifest change. [WP-085 D3](../work-packets/WP-085-tv-app-pipeline.md#decisions-for-jeff)
already puts licensed data on Cloudflare behind access control.

**R2 facts relevant here** (Cloudflare docs, read 2026-11-28):

- An object can be up to 5 TiB, and a single-part upload up to 5 GiB. A
  ~400 MiB package is one ordinary object, so **split archives aren't needed**.
- Egress is free. Storage is $0.015/GB-month. Class B (GET) is $0.36 per
  million requests.
- GET supports `Range` through the S3 API, so resuming is possible on R2's side.
- Presigned URLs (GET) expire between 1 second and 7 days. They **only work on
  the `<account>.r2.cloudflarestorage.com` S3 endpoint, not on a custom
  domain**. The `r2.dev` public URL isn't meant for production.

**Access.** A Cloudflare Access service token built into an APK can be
extracted from it. Presigned per-tester URLs written into a per-tester
manifest fit the existing `url` field without an app change, and they expire.
A Worker in front of the bucket, checking a tester credential, is the
alternative when URLs must outlive 7 days or use a custom domain.

| Option | Download (*Showdown*) | Pros | Cons |
|---|---|---|---|
| **A. Original ISO** | 636,268,544 | Byte-exact original; one artifact per disc; covers Gen 2/3 with no per-title curation | Largest download. Android can't mount ISOs, so the app would need an ISO reader or an on-device extract, which doubles peak storage. Ships the installer, AOL and messagemate executables. Data-only zip rules don't apply |
| **B. Whole disc, extracted and zipped** | ≈ whole-disc zip (not measured; B = 635,780,510 uncompressed) | Works with today's app. Keeps the disc-root layout Gen 2/3 detection needs. No per-title analysis | Carries ~79 MB of installer/third-party files per Gen 1 disc, including Windows executables (allowed by the extractor, but not game data) |
| **C. Verified runtime set, zipped** | **419,557,579** | Smallest that works today. Only files the engine reads. Verified for Gen 1 (§F) | Needs a per-title file list, and a re-check if engine detection changes. Not yet possible for Gen 2/3 |

**Recommended: Option C for Gen 1, Option B for Gen 2/3 until their readers
define a runtime set.**

---

## J. Technical risks

1. **Edition coverage.** Detection is per edition. The Feb 2001 *Showdown* is
   undetected (§F.2). Each packaged edition needs its own detection entry and
   test.
2. **No free-space check** before a ~400 MiB download and a ~530 MiB extract
   (§H).
3. **No download resume.** A dropped Wi-Fi link restarts the download (§H).
4. **Credentialed URLs** for licensed data: presigned URLs expire, and the
   manifest has no refresh path (§I).
5. **`Scooby.exe` in a data package.** It's needed for detection, never run,
   and allowed by the extractor. It might still draw a reviewer's question
   under the Play data-only policy. Detecting by `TGIFILE.ART` + `object.ini`
   + another data file would avoid shipping it. That's an engine change, not
   made here.
6. **SAF performance** for user-imported folders, because of per-clip file
   reopening (§H). Unmeasured.
7. **Silent movie loss** if a package omits a `BK.NNN` (§D).
8. **Gen 2/3 runtime set unknown.** *Jinx*'s root/`Jinx\` split and its
   `*_HD` files can't be sorted into runtime vs install-only until the MMFW
   reader exists.

---

## K. Recommended architecture

**Google TV storage (as built in WP-085, plus two gaps):**

```
getFilesDir()/
  packages/.staging/<id>-<ver>.zip, <id>-<ver>.d/   ← transient; needs zip + extract free
  packages/<id>/<version>/scooby/…                  ← the runtime set (game path)
  saves/                                            ← ScummVM saves, never in a package folder
```

- Add a free-space check before downloading: at least `size + extracted
  size`, plus the installed version's size on an update. The extracted size
  should come from the manifest (see below).
- Add Range-resume to the staged download, keeping the SHA-256 check on the
  complete file.

**R2 package structure (one object per game edition):**

```
r2://<bucket>/packages/<id>/<version>/<id>-<version>.zip
  zip root:  scooby/  Scooby.exe  TGIFILE.ART  object.ini  Sfx.dat  Voice.dat  Music.dat  BK.NNN…
```

- `id` names the edition (e.g. `showdown-en-aug2001`), so a second pressing
  is a second id, not a version.
- `version` is the package's own revision.
- Manifest (format 1) fields stay as they are. A format 2 could add
  `installedSize` for the free-space check, and a `files` list of
  name/size/SHA-256 so an installation can be re-verified in place.
- Deflate zip, because the app's `ZipInputStream` reads it. LZMA2 would cut
  *Showdown*'s download a further 71 MB (17%) but needs a new decoder
  dependency. That's not worth it yet.

---

## L. Open questions

1. Should the Feb 2001 *Showdown* pressing get a detection entry? It needs a
   runtime check for differences in `Scooby.exe`/`TGIFILE.ART`.
2. Should Gen 1 detection stop requiring `Scooby.exe`, so packages hold no
   Windows executables (§J.5)?
3. Presigned per-tester URLs, or a Worker gate, for licensed beta packages?
   How long should a tester's manifest stay valid?
4. Is the *Phantom* runtime set final once its room port closes (WP-087)? Its
   file list should be re-checked at closeout.
5. Gen 2/3 runtime sets, once the MMFW readers exist.
6. Does a full start-to-credits playthrough from the extracted copy need its
   own capture, or does the F.3 equivalence suffice for release evidence?
7. How fast does SAF load a user-imported folder on the TCL (§H)?

**Rights.** None of this answers the rights question. Local testing (this
study), internal development use, authorized beta distribution and public
sale each need their own permission, and owning a disc doesn't by itself
authorize redistribution. Nothing was uploaded or published in this study.

---

## M. Next implementation steps

1. Fix the doc drift: drop `Scooby.eng` from the required-files lists on the
   [status page](scummvm-wiki-scooby.md) and [install guide](download-and-install.md).
2. Write a per-title runtime file list (name, size, SHA-256) for *Showdown*
   Aug 2001, and later *Phantom*. Add a packaging script that builds the zip
   only from those files and refuses on any hash mismatch.
3. Add the free-space check and Range-resume to the WP-085 installer, test
   builds only, with a TCL re-verification. Drafted as
   [WP-096](../work-packets/WP-096-tv-package-reliability.md).
4. Choose the R2 access scheme (§L.3), then run one end-to-end package test on
   Jeff's own devices: R2 → manifest → TV. No game data goes anywhere public.
5. Optionally, add a Feb 2001 *Showdown* detection entry and test it.
6. Gen 2/3: rerun this study's §D–§G once the MMFW readers land.

---

## Verification performed (receipts kept locally)

- 7-Zip listings and PVD reads of all 9 ISOs; SHA-256 of all 9; prefix hash
  showing `Showdown.iso` = the Aug 2001 disc plus zero padding.
- SHA-256 of every extracted *Showdown* `scooby\` file, compared with the
  WP-085 copy (0 differences).
- 14 `--detect` runs across layouts and titles.
- Runtime runs on the extracted *Showdown* minimal set: boot (t1),
  sign-in to difficulty (t4), save/load (t3), exit click with a scene (t5),
  full self-test (t2).
- The same self-test on the read-only-mounted ISO; the ISO was unmounted
  afterwards and its timestamp checked unchanged.
- *Phantom*: boot and full self-test on its minimal set.
- Deflate, store and LZMA2 package builds; per-file deflate ratios.
- Source reads of the fork at `bfeaa1c7ba`: `engines/terraglyph/`,
  `common/fs.*`, `backends/fs/android/`, `backends/platform/android/`
  (advanced mode).
