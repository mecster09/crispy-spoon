# CLAUDE.md

This file guides Claude Code when working in this repository.

## Project

A Node.js terminal (CLI) application that migrates files/folders from
Microsoft OneDrive to a Google account, targeting either **Google Drive**
or **Google Photos**. See `requirements.md` for the full functional spec —
read it before making product decisions; it also lists open questions
that should be resolved (with the user) before implementing the affected
behavior, rather than guessed.

Status: pre-implementation. No source code exists yet — this repo
currently only has planning docs (`CLAUDE.md`, `requirements.md`) and
`.gitignore`.

## Non-negotiable constraints

- **No local copy of file contents, ever.** Transfers must stream
  directly from the OneDrive download URL into the Google upload request.
  Never write a full file to disk as an intermediate step. The state
  store (see below) may only persist metadata (paths, IDs, status,
  sizes) — never file bytes.
- **Credentials live in `.env` only**, loaded via `dotenv` or similar.
  Never commit `.env`, print its values, or write tokens/secrets to logs.
  `.env` is already git-ignored; keep `.env.example` in sync with any new
  required variable (names/placeholders only).
- **Recursive completeness in Drive mode**: every subfolder and file
  under the chosen source folder must be visited and migrated — no
  silent truncation on deep trees or paginated API responses (both Graph
  and Drive APIs paginate; always follow `@odata.nextLink` /
  `nextPageToken` fully).
- **Folder → Album mapping in Photos mode**: Google Photos has no folder
  concept; a source folder name becomes an album name (create if absent,
  reuse if present). Don't invent Drive-style folder emulation for this
  mode.
- Any ambiguity in requirements.md §9 (Open Questions) should be raised
  with the user rather than assumed silently when it affects the feature
  being built.

## Architecture guidance (for when code is added)

Expect/aim for a structure roughly like:

- `src/auth/` — OAuth flows for Microsoft Graph and Google APIs, token
  cache/refresh.
- `src/onedrive/` — Graph API client: listing, recursive traversal,
  streamed download.
- `src/google/drive/` — Drive API client: folder create-if-missing,
  streamed/resumable upload.
- `src/google/photos/` — Photos Library API client: album create/find,
  media upload.
- `src/migration/` — orchestration: mode selection, recursion/traversal
  engine, conflict policy, checksum/size verification.
- `src/state/` — resumable state store (pending/in-progress/done/failed),
  pause/resume logic.
- `src/logging/` — structured logger + terminal progress display.
- `src/cli.ts`/`bin/` — command entrypoint and argument parsing.

Keep the Microsoft-specific and Google-specific API code isolated behind
small interfaces so retry/backoff/logging/state logic stays shared and
provider-agnostic where possible.

## Commands

No package.json exists yet. Once the project is scaffolded, update this
section with the real commands, e.g.:

```
npm install       # install dependencies
npm run build     # compile (if using TypeScript)
npm run lint      # lint
npm test          # run test suite
npm start -- ...  # run the CLI
```

Do not guess at commands not yet defined — check `package.json` scripts
first if it exists.

## Conventions

- Prefer TypeScript for a project this API-surface-heavy (typed Graph/
  Drive/Photos responses catch a lot of pagination/shape bugs early) —
  confirm with the user if plain JS is preferred instead.
- Any new required `.env` key must be added to `.env.example` in the same
  change.
- Retry/backoff, rate-limit handling, and state-store writes are
  cross-cutting — changes to one provider client should not duplicate
  this logic; put it in `src/migration/` or a shared util instead.
- No secrets, tokens, or full file paths containing personal data in
  test fixtures or logs committed to the repo.
