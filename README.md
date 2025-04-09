# iFetch

**Bulk-download your iCloud Drive from the command line — incremental re-runs, resumable transfers, integrity verification, version history, and a pipeline that mirrors iCloud to your NAS and Google Drive.**

[![CI](https://github.com/roshanlam/iFetch/actions/workflows/ci.yml/badge.svg)](https://github.com/roshanlam/iFetch/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/)

Apple gives you two ways to get your data out of iCloud Drive: drag files around in Finder, or wait days for a privacy export. iFetch gives you a third: a scriptable CLI that downloads exactly what you want, skips files that haven't changed since the last run, resumes interrupted transfers, and keeps a local version history so an accidental overwrite in the cloud never costs you a file.

**Try it:** `pip install "ifetch[auth]"` then follow [First successful run](#first-successful-run) — about two minutes to a first download.

## Why iFetch?

**[rclone](https://rclone.org/iclouddrive/) is the serious alternative, and for many people the better choice.** It gained an iCloud Drive backend in v1.69.0 (January 2025), it uploads as well as downloads, and it reaches 70+ other clouds. If you want one tool for everything, use rclone. iFetch is deliberately narrow — download-only, iCloud-first — which is what lets it do a few things rclone's iCloud backend doesn't.

| | iFetch | [rclone](https://rclone.org/iclouddrive/) (iCloud backend) |
|---|---|---|
| Upload / delete / rename in iCloud | **Restore only** — `ifetch uplink` uploads files iCloud is missing; nothing else here can write, and nothing can delete or rename | **Yes** |
| Other cloud destinations | Google Drive only | **70+ backends** |
| Bidirectional sync | No (deliberate — see [Roadmap](#roadmap)) | **Yes** (`bisync`) |
| Mount / serve / crypt | No | **Yes** |
| Maturity | Young, single-maintainer | Mature project, but this backend is [Tier 4 "Experimental"](https://rclone.org/tiers/) and excluded from rclone's CI |
| Local version history + point-in-time restore | **Yes** (`ifetch-restore`) | No — deletes/overwrites go to iCloud Trash; no revision support |
| Apple package bundles (`.key`, `.pages`, `.numbers`, `.xcodeproj`) | **Restored as usable directories** (`--expand-packages`, on by default) | Transfer fails as "corrupted — sizes differ" and the file is deleted; `--ignore-size` leaves a ZIP named `Deck.key` ([#8404](https://github.com/rclone/rclone/issues/8404), [#8176](https://github.com/rclone/rclone/issues/8176), open since early 2025) |
| Offline integrity proof | **Yes** — signed `.ifetch_manifest.json` of SHA-256 digests; `ifetch-verify --offline` detects bit-rot years later with no credentials | No — Apple exposes no hashes, so verification is size-only |
| Headless / cron / Docker 2FA | **Yes** — `--2fa-code`, `$IFETCH_2FA_CODE`, watched file, or webhook | Interactive config only ([#9120](https://github.com/rclone/rclone/issues/9120)) |
| Proactive session-expiry warning | **Yes** — `ifetch auth status` exits non-zero *before* the ~30-day token dies | No — you find out when a run fails |
| Authentication diagnostics | **Yes** — `ifetch auth doctor` names the cause and the fix | Raw Apple errors (`HTTP 423 Missing PCS cookies`) |
| China Mainland (GCBD) accounts | **Yes** — `--region china` | Not supported; [#8257](https://github.com/rclone/rclone/issues/8257) open since Dec 2024 with two unmerged PRs |
| Folders shared by another Apple ID | Best-effort (see caveats below) | Share root only; operations inside fail ([#9477](https://github.com/rclone/rclone/issues/9477), fix unmerged) |
| Content hashes from Apple | None available (Apple exposes none) | None — so `--checksum` silently degrades to size-only |
| Advanced Data Protection | **Replay-proven** — PCS-cookie flow with bounded, headless-safe approval polling; `auth doctor` names *which* ADP precondition failed. **Not yet live-validated** against a real ADP Apple ID (see caveat below) | **Yes**, since v1.74.3 (June 2026) |
| Files evicted by "Optimize Mac Storage" | **`ifetch guard`** — reports the bytes that exist only on Apple's servers and are therefore missing from every Time Machine / Backblaze / Arq / rsync backup, and downloads them back | Not addressed — rclone does not inspect local FileProvider state |
| Noticing files deleted *in iCloud* before Trash purges them | **`ifetch vanish`** — classifies what went missing, bounds the ~30-day purge deadline, and refuses to call a broken scan a mass deletion | No — `sync` propagates the deletion to your local copy |
| Graphical interface | **Yes** — `ifetch serve`, a local web UI, no extra dependencies | No — third-party GUIs only; a GUI is the [most-discussed request](https://forum.rclone.org/top?period=yearly) on their forum |
| Restoring files back into iCloud | **`ifetch uplink`** — uploads only what iCloud is missing; never overwrites, deletes or renames | Yes, but as general two-way sync, with the deletion semantics that implies |
| Bandwidth limiting | **Yes** (`--bwlimit`, rclone-compatible timetables) | **Yes** |
| Dead-man's-switch / push notifications | **Yes** — Healthchecks.io, ntfy, webhooks, with anomaly distinct from failure | Via external tooling |
| Container image | **Yes** — GHCR, session-persisting volume layout | **Yes** |
| iCloud → NAS → Google Drive in one command | Yes (`ifetch-mirror --watch`) | Composable, but not a single command |

Two honest caveats about the columns above:

- **ADP: implemented, but not verified against a live ADP account.** iFetch now performs Apple's PCS-cookie flow (`requestWebAccessState` → device consent → bounded `requestPCS` polling), persists the cookie in the same session store as everything else so unattended re-runs need no re-approval, and names each ADP failure specifically — "Access iCloud Data on the Web is off", "approval not tapped on a trusted device", "session has no web-auth cookie" — instead of relaying `HTTP 423`. Use `ifetch auth renew --adp` and `ifetch auth doctor --online --adp`. **Caveat:** turning ADP on requires a physical device and a recovery key, so this path is pinned by a [contract-test harness](https://github.com/roshanlam/iFetch/blob/main/tests/test_adp.py) replaying Apple's recorded responses — the same approach as shared folders — and has *not* been run against a real ADP-enabled Apple ID. Where iFetch cannot tell whether ADP is enabled it reports "could not determine" rather than guessing. Treat it as implemented-and-tested-by-replay, not field-proven.
- **Shared folders**: the root cause of the long-standing failure here is now fixed rather than worked around. Apple puts `shareID` on the share *root* and omits it from the items it returns for that root's children, while every layer that needs share scope — pyicloud's `get_children()`, iFetch's download fallback — reads it off the node it is holding. The identifier was therefore lost one level down, and from there every request went out unscoped: HTTP 400 on the next listing, `WSObjectNotFound` on every file. That is the same defect that confines rclone's iCloud backend to the share root ([#9477](https://github.com/rclone/rclone/issues/9477), fix unmerged). iFetch now carries share context down the tree and seeds it into the exact key pyicloud reads, so nested listings and downloads both work. **This is still pinned by a [contract-test harness](https://github.com/roshanlam/iFetch/blob/main/tests/test_shared_folder_contract.py) replaying recorded responses, not by a live run** — recorded responses only prove iFetch branches correctly on the payloads we believe Apple sends. [`docs/shared-folder-validation.md`](https://github.com/roshanlam/iFetch/blob/main/docs/shared-folder-validation.md) is the 15-minute procedure against a real cross-account share; until someone runs it, treat this as fixed-in-principle.

For **photos**, use [icloudpd](https://github.com/icloud-photos-downloader/icloud_photos_downloader) — it is far more mature than iFetch's own brand-new `ifetch-photos`, which has not yet been validated against a live account.

## First successful run

Goal: install → store credentials → pass 2FA once → download something small. About two minutes if Apple's prompt arrives promptly.

![iFetch first-run demo](docs/assets/ifetch-demo.gif)

*Auth-ready `ifetch Documents ~/icloud-backup`: a `.key` package expands to a usable directory, then an unchanged re-run skips on the metadata fast path (0 bytes). On a 2.8 GiB folder the same re-run measured 4.4s / 0 bytes — see [Benchmarks](#benchmarks).*

### 1. Install

```sh
pip install "ifetch[auth]"
# Optional extras later: [gdrive] for Google Drive export/mirror
```

Or from a clone:

```sh
git clone https://github.com/roshanlam/iFetch.git
cd iFetch
python3 -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -e ".[auth]"
```

`[auth]` pulls the `icloud` CLI used in the next step. Without it you will get “No stored password found” (or a missing `icloud` command) on first run.

### 2. Tell iFetch which Apple ID to use

Every command needs an account. Set it once in your shell profile, or pass `--email` every time:

```sh
export ICLOUD_EMAIL=you@example.com
```

China Mainland Apple IDs also need `export ICLOUD_REGION=china` (or `--region china` on each command).

### 3. Store the password in the OS keyring (one time)

```sh
icloud auth login --username "$ICLOUD_EMAIL"
```

iFetch never prompts for your Apple ID password itself — it reads what pyicloud stored in the keyring (or what you supply later with `--password-command` / `$IFETCH_PASSWORD_COMMAND` on boxes with no keyring, e.g. Docker).

**Linux desktop:** you need a Secret Service backend (`gnome-keyring` or KWallet). **Headless / NAS / Docker:** skip the keyring and use a password command — see [docs/docker.md](docs/docker.md) and [docs/troubleshooting.md](docs/troubleshooting.md#keyring-issues-per-os).

If `icloud auth login` fails at the 2FA step but the password was already saved, skip to step 4 — `ifetch` uses a simpler 2FA path and will ask for the code itself.

### 4. Prove auth works (interactive — enter the 2FA code when asked)

```sh
ifetch Documents --list
# or: ifetch auth doctor --online
```

Apple sends a six-digit code to your trusted devices. After a valid code, iFetch asks Apple to **trust the session** so later runs (and cron/Docker) skip 2FA until the trust token expires — on the order of **~30 days**, not forever. Renew ahead of time with `ifetch auth status` / `ifetch auth renew` (see [Authentication that survives a headless box](#authentication-that-survives-a-headless-box) and [docs/troubleshooting.md](docs/troubleshooting.md)).

Headless boxes cannot answer a prompt: supply the code with `--2fa-code`, `$IFETCH_2FA_CODE`, `--2fa-file`, or `--2fa-webhook` instead.

### 5. Download a folder

```sh
ifetch Documents ~/icloud-backup
```

Re-run the same command tomorrow: unchanged files are skipped (metadata fast path — see [How re-runs decide what to download](#how-re-runs-decide-what-to-download)).

**Prefer a browser?** After install + keyring + `ICLOUD_EMAIL`:

```sh
ifetch serve
# open the printed http://127.0.0.1:8765/?t=… URL
```

See [docs/webui.md](docs/webui.md).

### Before you blame an empty listing

| Situation | What to do |
|-----------|------------|
| **Advanced Data Protection (ADP)** enabled | Apple's web/API path needs "Access iCloud Data on the Web" on, and a PCS approval on a trusted device. Run `ifetch auth doctor --online` (add `--adp` if needed). **Honest caveat:** ADP support is implemented and pinned by [replay contract tests](tests/test_adp.py), **not** yet validated against a live ADP Apple ID — treat it as replay-proven, not field-proven. |
| Want **iCloud Photos**, not Drive | Prefer [icloudpd](https://github.com/icloud-photos-downloader/icloud_photos_downloader). `ifetch-photos` exists but is **not production-polished** and has not been validated against a live library at scale. |
| Running in **Docker / cron / a NAS** | Do the interactive first run (or `--2fa-file` renew) and persist the session volume — [docs/docker.md](docs/docker.md), [docs/scheduling.md](docs/scheduling.md). |

> **Note on pyicloud:** iFetch is built on [timlaing/pyicloud](https://github.com/timlaing/pyicloud) (`pyicloud>=2.5.0` on PyPI). It is installed automatically with iFetch.


## Three commands to know

Most of iFetch is recovery tooling you reach for when something has gone wrong. Day to day, three commands cover it.

**1. Download a folder and keep it in sync**

```sh
ifetch Documents ~/icloud-backup
```

Re-run it tomorrow and unchanged files are skipped entirely — zero bytes, and thanks to the metadata fast path, zero network round-trips per file.

**2. Mirror iCloud → NAS → Google Drive in one command**

```sh
ifetch-mirror Documents /Volumes/nas/icloud --gdrive-folder "iCloud Mirror" --watch 900
```

Delta-aware at both hops; `--watch` repeats the pipeline on an interval.

**3. Finish a download that was interrupted**

```sh
ifetch resume ~/icloud-backup      # re-fetch only the files that didn't finish
ifetch repair  ~/icloud-backup     # offline: report what was left behind
```

A run cut off mid-transfer — closed laptop, dropped connection, power cut — resumes without starting over, and without re-listing your whole drive.

Everything else — planning, auditing, snapshots, conflict detection, recovery, uploads, the web UI — is in the [full command reference](#full-command-reference).

## Features

- **Web UI** — `ifetch serve` for sign-in, browsing, downloading and the reports, without touching a terminal ([docs/webui.md](https://github.com/roshanlam/iFetch/blob/main/docs/webui.md))
- **Backup-exposure report** — `ifetch guard` names the bytes that exist only on Apple's servers because "Optimize Mac Storage" evicted them, and that every backup of your Mac is therefore silently skipping; `--materialize --apply` pulls them back ([details](#what-your-backups-are-missing))
- **Deletion alerting** — `ifetch vanish` finds files that disappeared from *iCloud* while the cloud copy is still inside the ~30-day Trash window, and refuses to mistake a failed scan for a mass deletion
- **Secure authentication** — password lives in your OS keyring (via pyicloud), full 2FA/2SA support, trusted sessions so you are not re-prompted every run
- **Advanced Data Protection** — Apple's PCS-cookie flow with bounded, headless-safe approval polling (tested by replay, not against a live ADP account)
- **Bandwidth limiting** — `--bwlimit` with rclone-compatible timetables, applied to the whole run rather than per worker
- **Monitoring** — Healthchecks.io, ntfy and webhook notifications, with anomalies kept distinct from failures ([docs/monitoring.md](https://github.com/roshanlam/iFetch/blob/main/docs/monitoring.md))
- **Container image** — multi-stage, non-root, session-persisting ([docs/docker.md](https://github.com/roshanlam/iFetch/blob/main/docs/docker.md))
- **Headless-friendly 2FA** — supply the code with `--2fa-code`, `$IFETCH_2FA_CODE`, a watched file or a webhook, so cron/Docker/NAS runs never hang on a prompt ([details](#authentication-that-survives-a-headless-box))
- **Authentication diagnostics** — `ifetch auth doctor` says *which precondition failed* and how to fix it, instead of relaying `HTTP 423 Missing PCS cookies`
- **Proactive expiry warnings** — Apple's trust token lasts ~30 days; `ifetch auth status` exits non-zero days *before* it dies rather than after a backup fails
- **Apple package bundles restored** — `.key`/`.pages`/`.numbers`/`.xcodeproj` come back as usable directories instead of ZIP blobs ([details](#apple-package-bundles))
- **Offline integrity proof** — a signed `.ifetch_manifest.json` of SHA-256 digests lets `ifetch-verify --offline` detect bit-rot years later with no credentials ([details](#proving-a-backup-is-still-intact))
- **China Mainland support** — `--region china` for Apple IDs served by iCloud.com.cn
- **Incremental sync** — unchanged files are skipped entirely and interrupted downloads resume from where they stopped (see [How re-runs decide what to download](#how-re-runs-decide-what-to-download) for exactly what "unchanged" means)
- **Metadata fast path** — a re-run skips known-unchanged files with **zero network round-trips**, using the `.ifetch_state.json` sync-state file in the destination
- **Integrity verification** — `ifetch-verify` re-checks a local mirror against iCloud without modifying a single file
- **Resume-capable** — checkpointed downloads pick up where they left off after an interruption
- **Parallel downloads** — configurable worker pool (`--max-workers`)
- **Retries with exponential backoff** — transient 5xx/timeout errors are retried, `Retry-After` headers respected
- **Version history** — before overwriting a changed file, the previous copy is archived to a local `.versions/` directory (see [Version history](#version-history--rollback))
- **Shared items** — list and download files/folders shared *with* you (`--list-shared`)
- **Profiles** — named include/exclude pattern sets for repeatable sync jobs
- **Plugins** — hook into auth, listing, and download lifecycle events ([docs/plugins.md](https://github.com/roshanlam/iFetch/blob/main/docs/plugins.md))
- **Structured JSON logging** and a per-run `download_report.json` summary
- **Google Drive export** (`ifetch-export`) and an **iCloud → NAS → Google Drive mirror** (`ifetch-mirror`)

## How re-runs decide what to download

This is the part people most often get wrong about backup tools, so here is exactly what iFetch does — no more, no less.

### Step 1: the metadata fast path (no network)

At the start of a run, iFetch reads `.ifetch_state.json` from the destination root. That file was written by previous runs and records, per file, the remote size and remote modified timestamp Apple reported, plus the local size and mtime at the moment the download completed.

A file is skipped **without touching the network at all** only when *every* one of these agrees:

- the remote node exposes both a size and a modified timestamp;
- both match what was recorded for that path;
- the recorded local size matches the remote size;
- the local file still exists with exactly that size;
- no leftover `.temp` / `.download` artifact is sitting next to it.

Any doubt at all — missing metadata, no recorded entry, a size that moved, a partial-download artifact — and iFetch falls through to the full network check. The fast path can only ever *skip* work; it can never cause a file to be skipped that a network check would have downloaded.

Why it matters: previously every file cost a full HTTPS open just to read its `content-length`, even when nothing had changed — roughly 39 ms per file, which is over an hour of pure round-trips on a 100k-file drive.

Turn it off with `--no-fast-scan` (forces a network check per file) or bypass all skip logic with `--force` (re-downloads everything).

Honest caveats:

- It trusts Apple to update `dateModified` when a file changes. If Apple doesn't, iFetch won't notice a same-size edit (step 2 only re-downloads same-size files when the token changed or SyncState is missing).
- **Local** corruption at the same file size is not detected. That is what [`ifetch-verify`](#ifetch-verify--read-only-integrity-checking) is for.
- Two concurrent `ifetch` runs against the same destination can clobber each other's state file. The consequence is extra network checks on the next run — never a false skip.

### Step 2: the network check (size comparison + prefix resume)

When a file does not qualify for the fast path, iFetch opens the remote file and compares `content-length` against the local file. **Same size alone is never enough to skip** — SyncState must also prove the modified token still matches:

| Situation | What happens |
|---|-