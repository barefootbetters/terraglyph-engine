---
layout: default
title: "EC-014: Google TV package storage and download reliability run sheet"
---

# EC-014: Google TV package storage and download reliability: device checks

**Paired WP:** [WP-096](../work-packets/WP-096-tv-package-reliability.md)
**Purpose:** Verify each WP-096 behaviour on the TCL 65Q77K with evidence,
in the order the WP builds them. A failed step is fixed on the WP branch
before the next starts.
**Status:** not started.

**Evidence rule:** every checked box cites a `logcat` excerpt, a command
output or a screenshot captured during the run. No capture, no check. Jeff's
live confirmations are quoted as his. Quoted output replaces the app's id
with `<app-id>`. Raw captures stay local, as in
[EC-008](EC-008-tv-app-pipeline.md).

**How the device is driven:** Windows `adb`, as in EC-008. WSL's NAT mode
can't see the TV.

**Before and after every step that touches packages:** take a `run-as`
listing of `files/packages/`, `files/saves/` and the library entries
(snapshot S). WP-096 exit criterion 8 compares them.

---

## Pre-flight

- [x] WP-096 D1–D4 answered (2026-11-28); pre-flight verdict **READY**
      ([pre-flight-WP-096-2026-11-28](../reference/pre-flight-WP-096-2026-11-28.md)).
      Jeff chose one physical power cut (exit criterion 12): "Yes, once (Rec)".
- [ ] Any `/sdcard/wp096-filler/` left from an earlier run is removed, and
      `df /data` is recorded as the baseline
      ([filler safeguards](../work-packets/WP-096-tv-package-reliability.md#filler-safeguards-d4)).
- [ ] `tools/Invoke-TvFiller.ps1` is dry-run once: it creates a 16 MiB
      filler in `/sdcard/wp096-filler/`, is stopped with Ctrl+C, and leaves
      the folder gone and `df /data` within 1 MiB of the baseline.
- [x] Fork branch, base commit and toolchain recorded: `wp-096-tv-package-reliability`
      off `origin/scooby-engine` `6b5b02c53b`; platform `android-37.0`, NDK
      `23.2.8568313`, armeabi-v7a `liboboe.a` `399a790e…`, libpng16 v7a
      `c1a2ca89…` (pre-flight §Repo + device state).
- [x] The GitHub Releases download of `bass-cd-1.2.zip` was requested with
      `Range: bytes=1000-1999`: `302`, then `206 Partial Content`,
      `Content-Range: bytes 1000-1999/69377781`, and the 1,000 bytes match the
      same span of a full read (pre-flight, 2026-11-28).
- [ ] The synthetic package (D2) and its manifests are created as a new
      `tv-test-packages` release (Jeff's go, given at pre-flight: "Yes, at
      execution (Rec)"), and `Get-FileHash` matches them.
- [ ] The TCL is reachable over Windows `adb`, and its free internal storage
      is recorded (`adb shell df /data`). **Stop if under 2 GiB free**: the
      update test needs ~1.46 GiB. At pre-flight it was paired and had
      17,063,208 KiB (≈16.3 GiB) free ([pre-flight §Device](../reference/pre-flight-WP-096-2026-11-28.md#device-the-tcl-65q77k)).
      Re-check at the run. `adb devices` may list the TV twice (two mDNS
      names), so pin every command with `adb -t <transport>` or `-s <serial>`.
- [ ] Snapshot S's baseline includes the existing state: WP-085's `bass-cd`
      package and `installed.json`, and the saves `SKY-VM.*`, `showdown-win.001`
      and `showdown-win.002`.

## Step 1: build and install (time-box 0:45)

- [ ] The debug APK installs over the WP-085 test build. Library, packages and
      saves match snapshot S.
- [ ] The release APK passes the WP-085 D4 checks (no `advanced` package,
      `TVPKG_MANIFEST_URL` count 0).

**Outcome:**

## Step 2: storage preflight (time-box 1:00)

- [ ] Override (D4 b): with the debug free-space override set below the
      requirement, installing the synthetic package is refused at once. The
      message shows needed, free and shortfall, and `.staging/` stays empty.
- [ ] Filler (D4 a), through `Invoke-TvFiller.ps1` only: free space is left
      below the requirement but at or above the 256 MiB floor, with `df`
      recorded. The install is refused before any download, and `df` is
      unchanged after the attempt.
- [ ] Without `installedSize`, with room for the zip only (filler again): the
      download completes and verifies, the post-download check refuses, and
      `.staging/` is emptied.
- [ ] The script's cleanup has run: `/sdcard/wp096-filler/` is gone and `df`
      is within 1 MiB of the baseline.

**Outcome:**

## Step 2b: measure the headroom (time-box 0:45)

- [ ] With no filler, BASS and then the synthetic package are installed while
      `df /data` and `getAllocatableBytes` are logged after the download, at
      the extract's peak (polled once a second) and after install.
- [ ] Each package's overshoot over package size + installed size is
      recorded, with whether 64 MiB covered it (WP-096 exit criterion 11).

**Outcome:**

## Step 3: out of space mid-extract (time-box 0:45)

- [ ] An older version of the synthetic id is installed, then the newer one
      is started with a filler sized so the extract runs out (`ENOSPC`). The
      log names the cause, `.staging/` is empty, and the older version
      launches. The filler is deleted.

**Outcome:**

## Step 4: resume (time-box 1:00)

- [ ] The synthetic download is force-stopped at about 30%. The `.part` file
      and its sidecar remain.
- [ ] On retry the log shows `Range: bytes=<n>-` and `206`, and only the
      remainder is fetched. The SHA-256 matches and the package installs.
- [ ] The no-`Range` case (a `200` answer) and the changed-validator case
      restart from 0 and install, on the TV or in the JVM harness (say which).

**Outcome:**

## Step 5: verify (time-box 0:30)

- [ ] Verify passes on the fresh install.
- [ ] One byte of an installed file is changed through `run-as`. Verify names
      that file. The package is then reinstalled.

**Outcome:**

## Step 5b: interrupted installation (time-box 1:00)

- [ ] With an older synthetic version installed, start the update and
      `kill -9` the app process once the log shows extraction under way. After
      restarting: the log shows the staging sweep; `installed.json` names the
      old version; the old version launches and passes Verify; snapshot S
      matches.
- [ ] Repeat with `adb reboot` mid-extract, with the same checks after boot.
- [ ] After each interruption, retrying the update completes and passes Verify.
- [ ] Physical power cut at the wall (Jeff chose it at pre-flight). Start
      the update. When the log shows extraction under way, tell Jeff to pull
      the TV's plug, and record his confirmation. After it boots, run the
      same checks.

**Outcome:**

## Step 6: preservation and close (time-box 0:30)

- [ ] For steps 2–5b, snapshot S before vs. after matches for saves and
      library entries.
- [ ] `/sdcard/wp096-filler/` does not exist and `df /data` is within 1 MiB
      of the pre-flight baseline.
- [ ] The manifest doc is updated, the fork PR is open, and WP-096 §Result
      is written.

**Outcome:**

## Definition of done

Every step's outcome is recorded with captures. WP-096's exit criteria are
met, its §Result is written, and its WORK_INDEX row is updated.

## Out-of-scope captures

(One line per temptation, for a later WP.)
