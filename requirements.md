# Requirements — OneDrive → Google Migration CLI

## 1. Purpose

A Node.js terminal application that migrates files and folders from
Microsoft OneDrive directly to a Google account — either **Google Drive**
(preserving folder structure) or **Google Photos** (mapping a folder to an
album). The migration is a direct, streamed transfer between the two
cloud services: **no file is ever fully persisted to local disk**.

## 2. Core Mode Selection

On startup (or via CLI flag), the user chooses one destination mode for
the run:

- `drive` — move files and folders into Google Drive.
- `photos` — move photo/video files into Google Photos, using the source
  folder name to create/select an album (or, optionally, skip album
  assignment entirely — see §9).

Mode is selected once per run and determines which subsequent prompts and
validation rules apply. A run cannot mix both destinations; migrating the
same source to both targets requires two separate runs.

Source scope is always **one specific folder at a time**, chosen
explicitly by the user for that run (a single Drive-bound folder, or a
single OneDrive folder being migrated to a single Photos album/library
upload) — there is no "migrate my whole OneDrive account" mode.

Scope is limited to the signed-in user's **own OneDrive** content. Items
under "Shared with me" and SharePoint-backed document libraries are out
of scope for now; supporting them is a possible future enhancement, not
phase 1.

## 3. Authentication & Credentials

- **Microsoft Graph API** (OneDrive): OAuth 2.0 (authorization code flow
  with PKCE, or device code flow for a headless terminal). Requires an
  app registration in Azure AD/Entra ID with `Files.Read.All` (and
  `offline_access` for refresh tokens).
- **Google APIs**: OAuth 2.0 for a Google Cloud project with the
  **Drive API** (`drive.file` or `drive` scope, mode `drive`) and/or the
  **Photos Library API** (`photoslibrary.appendonly` +
  `photoslibrary.edit.appendonly` for album creation, mode `photos`)
  enabled.
- Credentials (client ID/secret, tenant ID, redirect URI, refresh tokens)
  are stored in a local `.env` file, never committed to source control
  (`.env` is already git-ignored; `.env.example` documents required keys).
- Tokens obtained during the OAuth flow are cached locally (e.g.
  `.tokens/` or in the state store, see §7) and refreshed automatically;
  the user should not need to re-authenticate on every run.
- No credential or token value is ever written to logs.
- The Photos Library API restricts read/list access to app-created
  content for most scopes; since this app only ever re-lists items it
  uploaded itself (for the verification pass in §7), the
  `photoslibrary.appendonly` + `photoslibrary.edit.appendonly` scopes are
  expected to remain sufficient. Reconfirm this against Google's current
  API terms at implementation time, since this policy area has shifted
  before.

## 4. No Local Copy Requirement

- Files are streamed directly from the OneDrive download URL into the
  Google upload request (pipe/stream chunking, e.g. Node `stream.pipeline`
  or resumable upload sessions on both ends).
- Temporary buffering in memory for a single chunk is acceptable; writing
  a complete file to a temp directory on disk is not.
- If a streamed transfer fails partway, the partial data is discarded and
  the transfer is retried (see §6) — it is not resumed from a local
  partial file.

## 5. Logging & Progress Tracking

- Structured logs (e.g. JSON lines or a library like `pino`/`winston`)
  written to a log file under `logs/` (already git-ignored) plus a
  human-readable summary on stdout.
- Log levels: `error`, `warn`, `info`, `debug`. Each file/folder operation
  logged with source path, destination path/ID, size, status, and
  duration.
- Live progress display in the terminal (e.g. via `cli-progress` or
  `ora`): overall percentage, files completed/total, bytes transferred,
  current file, current transfer speed, and ETA.
- A final run summary: total files migrated, skipped, failed, total bytes,
  elapsed time, and a path to the detailed log file.

## 6. Retry, Recovery, Pause & Restart

- **Retry**: transient failures (network errors, HTTP 429/5xx) are
  retried automatically with exponential backoff and jitter, up to a
  configurable max attempt count. Non-retryable errors (auth failure,
  permission denied, quota exceeded) stop retrying and are recorded as
  failed.
- **Concurrency**: transfers run with limited parallelism by default
  (default: 3 concurrent file transfers, configurable via `.env`/CLI
  flag) rather than fully sequential or unbounded. This default is
  conservative enough to stay under typical Graph API and Google API
  per-app rate limits; the backoff above further throttles automatically
  if a 429 is hit anyway.
- **State store**: a local manifest/database (e.g. a JSON file or SQLite
  under `.state/`) tracks, per source item: its path/ID, destination
  ID (once created), status (`pending`, `in-progress`, `done`, `failed`,
  `skipped`), byte count, and last-attempt timestamp. This state file
  contains no file contents — only metadata/paths — so it doesn't violate
  the no-local-copy rule.
- **Pause**: the user can interrupt the run (e.g. `Ctrl+C` or a `pause`
  command) and the app exits cleanly, leaving the state store consistent
  (in-progress items are re-marked `pending` on shutdown).
- **Restart/resume**: re-running the same command against the same
  source/destination automatically detects the existing state store and
  resumes from the first non-`done` item, skipping items already
  confirmed migrated. A `--fresh` flag forces starting a new state store
  and ignoring history.
- **Idempotency / conflict handling**: a true naming conflict at the
  destination should never occur in normal operation — it can only arise
  from a retried/interrupted run re-processing the same item. Duplicate
  data that then needs manual cleanup is explicitly unwanted, so the
  default behavior is:
  - Destination item with the **same name and same size** (checksum
    comparison where available) as the pending source item → treat as
    already migrated: skip, mark `done`, do not re-upload.
  - Destination item with the **same name but different size/checksum**
    → do **not** auto-rename or upload a second copy. Flag it as a
    conflict in the log and in the verification report (§7) for the user
    to resolve manually (overwrite, rename, or ignore) — never resolved
    silently by creating `file (1).ext`-style duplicates.
  - This check happens before every upload attempt, not just on resume.

## 7. Post-Migration Verification & Source Deletion

- The migration phase is **always a copy**: source items in OneDrive are
  never modified or deleted during the initial transfer. Deletion is a
  distinct, later, explicitly user-confirmed step — never bundled into
  the migration run itself.
- Once a run's pending items all reach a terminal state (`done`,
  `failed`, or `skipped`), the tool performs a **separate verification
  pass** before any deletion is offered:
  - Re-query the destination (the Drive folder tree, or the Photos
    album/library) and confirm each item the state store marks `done` is
    actually present, with matching name and size (and checksum, where
    the API exposes one — Graph's `quickXorHash` / Drive's
    `md5Checksum`).
  - Produce a **verification report**, shown in the terminal and written
    to the log/state directory, summarizing: items expected, confirmed
    present, missing/mismatched (with the reason), skipped, and failed.
  - Items that come back missing or mismatched are excluded from
    deletion eligibility — they're surfaced for the user to investigate
    or re-run, not silently deleted or silently left as-is.
  - The report step offers an **interactive retry** right there for any
    missing/mismatched items: the user can choose to have the tool
    re-attempt just those items (re-upload, then re-verify) before
    moving on, rather than being forced to re-run the whole migration
    command separately. Declining the retry just leaves those items
    excluded from the deletion prompt below.
- **Only after** the user has seen this report does the tool prompt
  whether to delete the confirmed-migrated source items from OneDrive
  (e.g. "Delete N confirmed-migrated items from OneDrive? [y/N]"):
  - This prompt is a separate command/step from running the migration,
    never automatic.
  - Deletion only ever targets items the verification pass just
    confirmed as present and matching — never items merely marked `done`
    in the state store without a fresh re-check.
  - Declining leaves OneDrive untouched; the run stands as a pure copy.
  - Any deletion performed is itself logged (which items, when).

## 8. Mode: Files & Folders → Google Drive

- User specifies a **source folder** in OneDrive (path or folder
  picker/ID) and a **destination folder** in Google Drive (path or ID).
- If the destination folder (or any segment of its path) doesn't exist,
  it is created automatically, mirroring the source's folder name(s).
- The entire subtree is migrated **recursively** — all files and all
  nested subfolders at every depth — so nothing is missed. Empty folders
  are also created at the destination.
- Folder structure/hierarchy from the source is preserved exactly in the
  destination.
- Naming conflicts are handled per the idempotency rule in §6 (expected
  to be rare, and only from a mistake or an interrupted/retried run).

## 9. Mode: Files & Folders → Google Photos

- User specifies a **source folder** in OneDrive (path or folder ID) —
  the folder chosen for a given run is treated as the top level for that
  run; it is not itself the child of some other in-scope folder.
- **Phase 1 scope**: the entire chosen source folder maps to **one**
  Google Photos album, named after that top-level folder:
  - If an album with that name already exists, items are added to it
    (respecting the idempotency/conflict rule in §6, adapted for Photos:
    matched by filename + size within the album before upload).
  - If not, the album is created first, then items are uploaded into it.
  - All photos/videos found recursively under the source folder (across
    any subfolders) are uploaded into this single album — phase 1 does
    **not** create one album per subfolder. Per-subfolder album mapping
    is a possible future enhancement, not in scope now.
- **No-album option**: the user may instead choose to upload the source
  folder's photos/videos directly into the Google Photos library with
  **no album** created or assigned at all (a plain library upload). This
  is a per-run choice presented alongside the album name prompt in
  `photos` mode, not a separate mode. Traversal is unaffected by this
  choice: the tool still recurses fully into every subfolder under the
  source folder and uploads every media file found (same
  recursive-completeness principle as Drive mode) — only the destination
  (no album vs. one album) differs, not what gets picked up.
- Only Google Photos-supported media types (images/videos) are uploaded
  in this mode; non-media files encountered are **skipped and logged**
  (not treated as errors, not routed anywhere else).

## 10. Non-Functional Requirements

- Cross-platform (macOS/Linux/Windows) terminal execution via Node.js
  (LTS version, e.g. 20.x+).
- Configuration-driven (via `.env` and/or CLI flags) — no hardcoded
  tenant/account values.
- Automated tests for retry logic, state-store resume behavior, the
  verification pass, and folder→album name mapping.
- Clear README-level setup instructions for registering the Azure AD app
  and Google Cloud OAuth client (to live outside this file, referenced
  from it).
