# Requirements — OneDrive → Google Migration CLI

## 1. Purpose

A Node.js terminal application that migrates files and folders from
Microsoft OneDrive directly to a Google account — either **Google Drive**
(preserving folder structure) or **Google Photos** (mapping folders to
albums). The migration is a direct, streamed transfer between the two
cloud services: **no file is ever fully persisted to local disk**.

## 2. Core Mode Selection

On startup (or via CLI flag), the user chooses one destination mode for
the run:

- `drive` — move files and folders into Google Drive.
- `photos` — move photo/video files into Google Photos, using folder
  names to create/select albums.

Mode is selected once per run and determines which subsequent prompts and
validation rules apply. A run cannot mix both destinations; migrating the
same source to both targets requires two separate runs.

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
- **Idempotency**: before re-uploading an item during a resume, the tool
  checks whether it already exists at the destination (by name within the
  destination folder/album) to avoid duplicates.

## 7. Mode: Files & Folders → Google Drive

- User specifies a **source folder** in OneDrive (path or folder
  picker/ID) and a **destination folder** in Google Drive (path or ID).
- If the destination folder (or any segment of its path) doesn't exist,
  it is created automatically, mirroring the source's folder name(s).
- The entire subtree is migrated **recursively** — all files and all
  nested subfolders at every depth — so nothing is missed. Empty folders
  are also created at the destination.
- Folder structure/hierarchy from the source is preserved exactly in the
  destination.
- File name collisions at the destination are handled per a configurable
  conflict policy (default: skip if identical size, otherwise rename with
  a suffix — see §9 open questions).

## 8. Mode: Files & Folders → Google Photos

- User specifies a **source folder** in OneDrive (path or folder ID).
- Google Photos has no folder concept, so each source **folder name
  becomes a Google Photos album name**:
  - If an album with that name already exists, items are added to it
    (respecting idempotency in §6).
  - If not, the album is created first, then items are uploaded into it.
- Recursion into subfolders still occurs; the default behavior maps each
  subfolder to its own album named after that subfolder (see open
  question in §9 for alternative "flatten into parent album" behavior).
- Only Google Photos-supported media types (images/videos) are uploaded
  in this mode; non-media files encountered are skipped and logged as
  such (not treated as errors).

## 9. Open Questions / Decisions Needed

These weren't fully specified and are worth pinning down before/while
building:

1. **Subfolder-to-album mapping**: when recursing in `photos` mode, should
   each nested subfolder become its own album (e.g. `Trip/Day1` →
   album "Day1"), or should all photos from the whole subtree land in one
   album named after the top-level source folder? Naming collisions
   (two different subfolders both named "Day1") also need a rule.
2. **Conflict policy** in `drive` mode: skip existing files with the same
   name, overwrite them, or always rename (e.g. `file (1).ext`)? Should
   this be user-configurable per run?
3. **Non-media files in `photos` mode**: skip-and-log is assumed above —
   confirm, or should they instead be routed to a Drive folder as a
   fallback?
4. **Delete-after-move**: is this a *move* (delete from OneDrive after
   confirmed successful upload) or a *copy* (leave OneDrive untouched)?
   The task says "migrate"/"move" — if source deletion is in scope, it
   needs its own confirmation step and safety rails (e.g. only delete
   after destination checksum/size verification).
5. **Selective vs. full account migration**: is source always a single
   folder chosen per run, or should there be a mode to migrate an entire
   OneDrive account in one go?
6. **File integrity verification**: should the tool verify checksums/size
   between source and destination after transfer (Graph API exposes
   `quickXorHash`; Drive exposes `md5Checksum`) to confirm a faithful
   copy, or is size-comparison sufficient?
7. **Rate limits/throttling**: both Graph API and Google APIs impose
   per-user/per-app rate limits and 429 responses — confirm acceptable
   concurrency (parallel file transfers) vs. strictly sequential.
8. **Shared/synced items**: does OneDrive "Shared with me" content, or
   items in a SharePoint-backed library, need to be in scope, or only the
   user's own OneDrive?
9. **Google Photos API deprecation note**: as of 2025, the Photos Library
   API restricts read/list access to app-created content only for most
   scopes — uploads/album-creation used here are unaffected, but this is
   worth confirming still matches current Google API terms before
   implementation.

## 10. Non-Functional Requirements

- Cross-platform (macOS/Linux/Windows) terminal execution via Node.js
  (LTS version, e.g. 20.x+).
- Configuration-driven (via `.env` and/or CLI flags) — no hardcoded
  tenant/account values.
- Automated tests for retry logic, state-store resume behavior, and
  folder→album name mapping.
- Clear README-level setup instructions for registering the Azure AD app
  and Google Cloud OAuth client (to live outside this file, referenced
  from it).
