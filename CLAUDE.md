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
- **Folder → Album mapping in Photos mode (phase 1)**: Google Photos has
  no folder concept; the whole chosen source folder maps to **one** album
  named after that top-level folder (create if absent, reuse if present).
  Do not implement one-album-per-subfolder in phase 1 — that's an
  explicitly deferred future enhancement. Also support a "no album"
  variant where photos upload straight to the Photos library with no
  album assignment (a per-run choice, not a separate mode).
- **Migration is copy-first; deletion is a separate, later, confirmed
  step.** Never delete or modify OneDrive source items during the
  migration run itself. After a run completes, a distinct verification
  pass re-checks the destination against the state store and produces a
  report; only after the user reviews that report does the tool prompt
  to delete confirmed-migrated source items. Deletion must never be
  offered for — or applied to — items that weren't just freshly
  re-verified as present at the destination.
- **Avoid duplicate-data conflicts.** A destination naming conflict
  should only ever arise from a mistake or an interrupted/retried run,
  never from normal operation. Before any upload, check the destination
  for an existing item with the same name: if size/checksum also match,
  skip (already migrated); if they differ, don't auto-rename or upload a
  second copy — flag it as a conflict for manual resolution instead.
- Any ambiguity in requirements.md §9a (Open Questions) should be raised
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
