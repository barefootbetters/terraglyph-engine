---
layout: default
title: "WP-096: Google TV package storage and download reliability"
---

# WP-096: Google TV package storage and download reliability

**Status:** 📝 Drafted (2026-11-28) · D1–D4 answered 2026-11-28 · needs pre-flight.
**Numbering:** WP-086 is the Hotspot Magnet research (✅), and WP-095 is held
by the open wizard's-skull PR, so this takes WP-096 and EC-014 (EC-011 has
never been used) ([wp-lifecycle §Supersession check](../reference/wp-lifecycle.md#supersession-check-required-before-reserving-a-number)).
**Phase:** 5 (polish & distribution).
**Depends on:** WP-085 (its installer, merged into the fork in
[barefootbetters/scummvm#13](https://github.com/barefootbetters/scummvm/pull/13);
WP-085's one open item, an import route without a computer, isn't needed here);
the [ISO-free feasibility study](../reference/iso-free-execution-feasibility-2026-11-28.md)
(the sizes this WP plans for).
**Unblocks:** delivering licensed game-data packages to beta testers' TVs.
That also needs a host and access scheme, a separate WP ([§Out of scope](#out-of-scope)).
**Companion EC:** [EC-014](../execution-checklists/EC-014-tv-package-reliability.md).
**Pre-flight required:** **Yes.** It changes the fork outside
`engines/terraglyph/` (the Android advanced mode) and changes network download
code.
**Targets generation(s):** none directly. No engine code changes. The
installer is game-agnostic, and *Showdown* (Gen 1) sets the sizes it must
handle.
**Estimated effort:** 3–4 days, including the TCL runs.

---

## Goal

Make the WP-085 package installer safe on a storage-constrained Google TV.
It should know before it starts whether a package fits, refuse cleanly when it
doesn't, resume an interrupted download instead of starting over, confirm
that what it installed is what the package held, and never lose an installed
version or a save along the way. The installer is extended, not replaced.

## Background

- **What WP-085 built** (`backends/platform/android/advanced/`, test builds
  only per [WP-085 D4](WP-085-tv-app-pipeline.md#decisions-for-jeff)): a
  manifest (format 1: id, version, size, SHA-256, HTTPS URL), a download to
  `packages/.staging/`, size and SHA-256 checks, a data-only extract, a rename
  into `packages/<id>/<version>/`, an update beside the old version, and removal
  that keeps saves. Its rules are in the [manifest doc](../reference/tv-package-manifest.md).
- **What the feasibility study measured** for *Showdown*, Aug 2001
  ([§G](../reference/iso-free-execution-feasibility-2026-11-28.md#g-storage-comparison)):
  - installed size 556,351,281 B (530.6 MiB);
  - zip download 419,557,579 B (400.1 MiB);
  - peak during install ~931 MiB (zip plus extract);
  - peak during an update ~1.46 GiB (old version, zip and new version).

  A TV needs far more free space than the game's final size.
- **The gaps the study found** in `PackageStore.java` at `bfeaa1c7ba`
  ([§H](../reference/iso-free-execution-feasibility-2026-11-28.md#h-google-tv-compatibility)):
  - No free-space check: there's no `StatFs`, `getUsableSpace` or
    `getAllocatableBytes` call before the download or the extract.
  - No resume. `fetch()` calls `cleanStaging()` first (`:176`) and deletes the
    zip in `finally` (`:209`), so any interruption restarts a ~400 MiB download
    from byte 0.
  - Integrity is checked for the zip (SHA-256) and, implicitly, per entry
    (`ZipInputStream`'s CRC-32). Nothing is recorded that would let an
    installed folder be checked again later.
- **What already holds and must keep holding:**
  - An interrupted update leaves the old version playable (WP-085 Milestone 4).
  - Removal keeps saves, which live in `<base>/saves`, outside `packages/`.
- **The manifest parser ignores unknown fields.** It reads every field with
  `optString`/`optLong` (`PackageManifest.java`), so a new optional field stays
  compatible with format 1 and older test builds.

## Scope

### In scope

All of it lives in the advanced mode (test builds). The release build stays
free of it, as WP-085 D4 requires.

**1. Storage preflight**

1. Before downloading, the app computes the required space as **package size
   + installed size + headroom** ([D3](#decisions-for-jeff)). It compares that
   with the bytes it can actually allocate on the volume that holds
   `getFilesDir()`: `StorageManager.getAllocatableBytes` on API 26+, and
   `File.getUsableSpace` below that. For testing, a debug-only setting can
   override the free-space figure ([D4](#decisions-for-jeff)).
2. The installed size comes from a new **optional** manifest field,
   `installedSize` (the sum of the zip's uncompressed entry sizes). If it's
   absent, the app checks the download size before downloading. Once the zip
   is verified, it reads the zip's central directory for the exact extracted
   size and checks again **before** extracting.
3. An update needs space for the zip and the new version only. The installed
   old version is already on disk and isn't counted as free.
4. If the space isn't there, the app **refuses before writing anything**. It
   shows the bytes needed, the bytes free and the shortfall, and logs the same
   line ([D1](#decisions-for-jeff)).
5. Running out of space mid-download or mid-extract (`ENOSPC` as an
   `IOException`) deletes that package's staging files. Any installed version
   stays where it is, and the log names the cause.

**2. Interrupted-download recovery**

6. A network failure or a killed app keeps the partial download as
   `<id>-<version>.zip.part`, with a sidecar recording the URL, the bytes
   written and the server's `ETag`/`Last-Modified`.
7. A refusal still deletes the partial download: wrong size, wrong SHA-256,
   a data-only rule, or any `RefusedException`.
8. A retry of the same id and version continues the download. The app
   re-hashes the bytes it already has, then sends `Range: bytes=<n>-` with
   `If-Range`. A `206` appends. A `200`, a changed validator or a `416`
   restarts from 0. The final size and SHA-256 checks are unchanged, so a bad
   resume can never install.
9. `cleanStaging()` keeps the `.part` and sidecar for the id and version being
   fetched, and deletes everything else in `.staging/`.

**3. Installation integrity**

10. After extracting, the app writes `packages/<id>/<version>.files.json` with
    each entry's path, size and CRC-32 from the zip. It sits outside the game
    folder, so ScummVM's detection never sees it.
11. The advanced mode gains a **Verify** action per installed package. It
    re-reads every file, compares size and CRC-32 with `.files.json`, and
    reports pass, or each missing or changed file.

**4. Preservation**

12. A refusal, an interruption or a failure at any step never deletes or
    alters an installed version, its library entry or any save.
13. Documented in the [manifest doc](../reference/tv-package-manifest.md):
    `installedSize`, the space rule, resume and Verify.

**5. Interrupted installation recovery** (a power cut or a killed app during
extraction)

WP-085 already covers part of this at `bfeaa1c7ba`:
- extraction runs inside `.staging/` and is renamed into place only when
  complete;
- `installed.json` is written atomically (temp file and rename), and only in
  `applyPending()` after that rename;
- `cleanOrphans()` deletes any version folder that `installed.json` and
  `pending.json` don't name.

So a partly extracted game is never recorded as installed today. What's
missing is **durability across a power cut**: no file or directory is
`fsync`ed before those renames, so file data that hasn't reached the disk may
be lost behind a completed rename. Staging is also swept only when the next
download starts.

14. Every extracted file is `fsync`ed (`FileOutputStream.getFD().sync()`)
    before the staging folder is renamed into place. The written
    `installed.json`/`pending.json` temp file is synced before its rename.
15. On every advanced-mode start, the app runs `cleanOrphans()` and a staging
    sweep before anything else. The sweep deletes every `.d` extraction folder
    and temp file in `.staging/`, keeping only the `.part` download and
    sidecar that item 9 keeps. Each deletion is logged.
16. A package is marked installed (`installed.json`) only after its folder is
    complete, synced and renamed. An interrupted extraction leaves the
    previous version recorded, playable and untouched, along with every save.
17. After an interruption, installing the same package again starts cleanly:
    the download resumes or is reused if complete and verified, and the
    extract starts over.

### Out of scope

- **The production host and access scheme for licensed packages** (Cloudflare
  R2, presigned URLs vs. a Worker gate). This belongs in its own WP once the
  question in [the study's §L.3](../reference/iso-free-execution-feasibility-2026-11-28.md#l-open-questions)
  is answered. The downloader already takes any HTTPS URL.
- **Hosting *Showdown* or any licensed data anywhere.** It's tested here only
  through the size of a synthetic package ([D2](#decisions-for-jeff)).
- **Freeing space for the user** (offering to remove other games or clear
  caches). D1 records it as a later option.
- **Package building** (a per-title file list and a packaging script). That's
  step 2 of the study's next steps, a `tools/` WP of its own.
- **The release build, Google Play and any store UI.** WP-084 owns them.
- **Engine detection changes** (`Scooby.exe`, the Feb 2001 *Showdown*). Those
  belong to an engine WP.
- **Manifest format 2.** Everything here fits format 1 as optional fields.

## Decisions for Jeff

All four were answered on 2026-11-28, when Jeff relayed a review that
approved each recommendation and added safeguards to D3 and D4.

| # | Decision | Recommendation | Answer (2026-11-28) |
|---|---|---|---|
| D1 | **What happens when a package doesn't fit.** (a) Refuse, showing needed, free and shortfall; (b) also offer to remove other installed packages to make room | **(a)** for v1. Choosing which game to delete is a product decision, and test builds don't need it yet | **(a)**, "Approve" |
| D2 | **A large test package.** (a) A synthetic ~420 MB zip of random data, on the public `tv-test-packages` release (no rights question, and it matches *Showdown*'s real size); (b) BASS only (69 MB, too small to stress space or resume); (c) a real *Showdown* package on a private host | **(a)**. (c) waits for the host-and-access WP | **(a)**, "Approve" |
| D3 | **Headroom** added to the required space. (a) A fixed 64 MiB; (b) 5% of the installed size; (c) none | **(a)**. It covers filesystem overhead and the sidecars, and is easy to read in a refusal message | **(a)** "for initial testing". The 64 MiB must be **checked against what the TCL actually does**, not assumed to be enough everywhere (exit criterion 11) |
| D4 | **Testing a full disk on Jeff's TCL.** (a) Fill internal storage with a temporary filler file over `adb` until the package no longer fits, then delete it; (b) test only through a debug-build override of the free-space figure | **(a)** for the real `ENOSPC` path, with (b) as well for the quick refusal checks. The filler is deleted in the same step, and the TV's free space is recorded before and after | **(a) and (b)**, "Approve with safeguards". The safeguards are in [§Filler safeguards](#filler-safeguards-d4) |

### Filler safeguards (D4)

1. **One test folder only.** Filler files go in `/sdcard/wp096-filler/`.
   Nothing is written anywhere else, and the cleanup deletes only that folder.
2. **Never a full device.** Each filler is sized to leave a stated floor
   free: the package's shortfall target, and never less than 256 MiB of
   real free space, so the system and other apps keep running. The size is
   computed from a fresh `df /data` reading just before the filler is written.
3. **Automatic cleanup, failure included.** Fillers are created and removed
   by one script, `tools/Invoke-TvFiller.ps1` (PowerShell). Its `try/finally`
   deletes `/sdcard/wp096-filler/` however the step ends: a pass, a failure,
   an `adb` error or Ctrl+C. Each filler step starts and ends with `df /data`,
   and the end reading must be within 1 MiB of the start.
4. **No orphans between sessions.** EC-014's pre-flight removes any
   `/sdcard/wp096-filler/` left from a previous run before anything else.

## Engineering calls (Claude's, for pre-flight review)

1. **`getAllocatableBytes` over `getUsableSpace` on API 26+.** It counts the
   cache Android can clear on demand. That matches what an install can really
   get on a TV whose free space is mostly cache. The TCL runs Android 14.
2. **CRC-32, not SHA-256, for Verify.** The zip already carries a CRC-32 for
   every entry, so it costs no extra manifest data. The zip-level SHA-256
   still guards the download itself.
3. **The resume validator comes from the server's response, not the manifest.**
   `If-Range` with the stored `ETag` or `Last-Modified` stops a changed object
   from being appended to an old partial file. The final SHA-256 check is the
   backstop either way.
4. **Hosts that ignore `Range`** (they answer `200`) just restart. Before
   relying on resume against GitHub Releases, the pre-flight records whether
   its download redirect honours `Range`.

## Dependencies

- A fork worktree off a freshly fetched `origin/scooby-engine`
  ([wp-lifecycle](../reference/wp-lifecycle.md); `tools/New-ForkWorktree.ps1`).
- The Android toolchain and the libpng build from the
  [smart-TV guide](../reference/scummvm-smart-tv.md). The TV is driven through
  Windows `adb`.
- The TCL 65Q77K.
- Test packages on `barefootbetters/tv-test-packages`:
  - the existing BASS 1.2 and 1.2.1;
  - the D2 synthetic package and a manifest that lists it with and without
    `installedSize`.

  New release assets are created only on Jeff's go, as in WP-085.

## Execution

1. Pre-flight: `pre-flight-WP-096-<date>.md`, with D1–D4 answered and the
   GitHub Releases `Range` behaviour recorded.
2. Implement in the order of the scope sections: 1 (preflight), 2 (resume),
   3 (integrity), 5 (interrupted install), and 4 (the preservation checks,
   mostly tests). One commit per section on the fork branch.
3. Exercise the pure-Java parts on a desktop JVM, as WP-085 did for
   zip-slip: the space arithmetic, the resume decision table, the
   `.files.json` comparison.
4. Run [EC-014](../execution-checklists/EC-014-tv-package-reliability.md) on
   the TCL, write §Result, and open the fork PR.

## Exit criteria

1. [ ] D1–D4 answered (✅ 2026-11-28); pre-flight READY.
2. [ ] With too little space (a filler file, per D4), installing the synthetic
   package is refused **before** any byte reaches `.staging/`. The refusal
   shows needed, free and shortfall, and the free-space figure before and
   after the attempt is unchanged (logged).
3. [ ] With exactly enough room for the download but not the extract, and no
   `installedSize` in the manifest, the post-download check refuses before
   extracting and the staging files are gone (logged).
4. [ ] A real `ENOSPC` mid-extract leaves an installed older version of the
   same id playable, and `.staging/` empty (logged).
5. [ ] A download killed at about 30% (`am force-stop`) and retried sends
   `Range`, gets `206`, transfers only the remainder, and installs with a
   matching SHA-256. The log shows the resume offset and the bytes fetched
   (logged).
6. [ ] A host or object that answers `200` to a `Range` request restarts from
   0 and still installs. A partial download whose validator changed restarts
   from 0 too (logged; local test or JVM harness).
7. [ ] Verify passes on a fresh install. After one byte of an installed file
   is changed through `run-as`, Verify names that file (logged).
8. [ ] In each of criteria 2–7 and 12, existing saves and library entries are
   byte-identical before and after (`run-as` listings, as in WP-085
   Milestone 4).
9. [ ] The release APK still has no advanced-mode code (the WP-085 D4
   `apkanalyzer` and dex string checks, returning 0).
10. [ ] The [manifest doc](../reference/tv-package-manifest.md) documents
    `installedSize`, the space rule, resume, Verify and the
    interrupted-install recovery. The fork PR is open with the section
    commits, and §Result is written.
11. [ ] **The headroom is measured, not assumed** (D3). For BASS and the
    synthetic package, the real free-space drop on the TCL is recorded at
    three points: after the download, at the extract's peak, and after
    install (`df /data` and `getAllocatableBytes`). §Result states each
    package's overshoot over package size + installed size, and whether
    64 MiB covered it. If it didn't, the headroom is raised to the measured
    overshoot rounded up, and the change is noted.
12. [ ] **Interrupted installation recovery.** The app is killed mid-extract
    (`kill -9` of the app process) during an **update** of the synthetic
    package, and separately `adb reboot` is run mid-extract. In each case,
    after restart:
    1. the previous version is still installed, recorded in `installed.json`,
       launches and passes Verify;
    2. every save and library entry is byte-identical to snapshot S;
    3. the log shows the staging sweep removing the incomplete extraction;
    4. `installed.json` never names the interrupted version;
    5. retrying the update completes and passes Verify.

    A physical power cut at the wall is run as well only if Jeff chooses it
    at pre-flight (logged).

## Deliverables

- Fork branch and PR (Java only, under `backends/platform/android/advanced/`,
  plus any manifest-merge or Gradle line it needs). No game data.
- The updated manifest doc; `pre-flight-WP-096-<date>.md`; EC-014 filled in;
  §Result, with the measured headroom (criterion 11).
- `tools/Invoke-TvFiller.ps1`, the D4 filler script with guaranteed cleanup
  (this repo).
- On Jeff's go: the synthetic test package and its manifests on
  `tv-test-packages`.
- Never committed: game data, captures showing the TV's address or the app's
  id (they stay local, as in WP-085), signing keys.

## Notes

- **Evidence discipline.** Every device claim cites a capture from the run;
  Jeff's live confirmations are quoted as his
  ([DECISIONS](../ai/DECISIONS.md), the 2026-11-21 fabrication lesson).
- **Sizes to design for** (from the study): a ~400 MiB download, a ~531 MiB
  install, ~931 MiB free to install, and ~1.46 GiB free to update. Two Gen 1
  games installed take ~1.02 GiB.
- **Generation discipline.** No generation claim is made. Gen 2/3 packages
  will keep their disc-root layout (study §E), which this installer already
  allows.
