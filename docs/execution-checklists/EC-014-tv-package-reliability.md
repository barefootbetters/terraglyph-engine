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

- [ ] WP-096 D1–D4 answered; pre-flight verdict **READY**
      (`pre-flight-WP-096-<date>.md`).
- [ ] Fork branch, base commit and toolchain recorded (as in EC-008).
- [ ] The GitHub Releases download of `bass-cd-1.2.zip` was requested with
      `Range: bytes=1000-`, and the status (`206` or `200`) is recorded.
- [ ] The synthetic package (D2) and its manifests exist on Jeff's go, and
      `Get-FileHash` matches them.
- [ ] The TCL's free internal storage is recorded (`adb shell df /data`).

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
- [ ] Filler (D4 a): a filler file is created under `/sdcard/` leaving less
      than the requirement, and `df` is recorded. The install is refused
      before any download. `df` is unchanged after the attempt.
- [ ] Without `installedSize`, with room for the zip only: the download
      completes and verifies, the post-download check refuses, and `.staging/`
      is emptied.
- [ ] The filler is deleted and `df` is back to the recorded value.

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

## Step 6: preservation and close (time-box 0:30)

- [ ] For steps 2–5, snapshot S before vs. after matches for saves and
      library entries.
- [ ] The manifest doc is updated, the fork PR is open, and WP-096 §Result
      is written.

**Outcome:**

## Definition of done

Every step's outcome is recorded with captures. WP-096's exit criteria are
met, its §Result is written, and its WORK_INDEX row is updated.

## Out-of-scope captures

(One line per temptation, for a later WP.)
