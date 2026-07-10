# Design: Overall Architecture & Project Scaffolding

Date: 2026-07-10
Status: Approved (pending implementation plan)

## Context

`requirements.md` and `CLAUDE.md` are fully resolved (no open questions
remain). This is the first implementation-facing design: it establishes
the language/toolchain, directory structure, and the module boundaries
that keep provider-specific (Microsoft/Google) code isolated from the
shared retry/state/conflict logic, per CLAUDE.md's architecture
guidance. It does not design the internals of any single provider client
or the full OAuth flow — those are follow-up specs.

## Language & Toolchain

- **Language**: TypeScript. Chosen over plain JS because the app talks
  to three different paginated/streaming external APIs (Graph, Drive,
  Photos Library) — typed responses catch shape/pagination bugs at
  compile time rather than at runtime against a real account.
- **Module system**: native ESM, Node 20+ (LTS).
- **Package manager**: npm (single package, no monorepo — the CLI's
  scope doesn't warrant one).
- **Dev/build/test**:
  - `tsx` for running the CLI directly in development (no separate
    compile step needed while iterating).
  - `tsc` for the production build.
  - `vitest` for the test suite (TS-native, fast, Jest-compatible API).
  - ESLint + Prettier for lint/format.

## CLI Interaction Model

- `commander` handles command/flag parsing.
- `@inquirer/prompts` drives the interactive guided flow (mode, source
  folder, destination folder/album, no-album toggle, etc.) when a
  required choice isn't supplied via flag.
- Every prompted value has a corresponding flag/env override, so the
  same command is fully scriptable for automation and resumed runs —
  interactive mode is the friendly default, not the only path.

## Provider Client Strategy

- **Microsoft Graph** (OneDrive): `@azure/msal-node` for the
  Authorization Code Flow with PKCE, `@microsoft/microsoft-graph-client`
  for Graph calls (listing, recursive traversal, streamed download).
- **Google Drive**: `google-auth-library` for auth, `googleapis`
  package for Drive calls (folder create-if-missing, resumable upload).
- **Google Photos Library API**: no solid current first-party Node SDK,
  so this client is hand-rolled REST + streaming `fetch`, using
  `google-auth-library` for the token. It is isolated behind the same
  `DestinationProvider` interface as Drive (see below), so this
  implementation detail never leaks into shared code.

## Directory Structure

```
bin/
  cli.ts                         # shebang entry, delegates to src/cli/index.ts

src/
  cli/
    index.ts                     # command registration (commander)
    prompts.ts                   # @inquirer/prompts flows
    commands/
      migrate.ts                 # run/resume a migration (copy phase)
      verify.ts                  # standalone re-run of the verification pass
      delete.ts                  # standalone deletion prompt against last verified report

  auth/
    msGraphAuth.ts                # PKCE flow + local redirect listener for MS
    googleAuth.ts                 # PKCE flow + local redirect listener for Google
    tokenCache.ts                 # shared cache/refresh abstraction (used by both)

  onedrive/
    client.ts                     # implements SourceProvider
    types.ts

  google/
    drive/
      client.ts                  # implements DestinationProvider (folder mode)
    photos/
      client.ts                  # implements DestinationProvider (album/library mode)

  migration/
    orchestrator.ts               # drives the copy phase
    conflictPolicy.ts              # pure decision logic (unit-testable, no I/O)
    verify.ts                       # post-migration verification pass + report + interactive retry
    retry.ts                         # shared backoff/jitter wrapper
    concurrency.ts                    # bounded worker pool (default 3)

  state/
    db.ts                          # better-sqlite3 setup, schema/migrations, WAL pragma
    repository.ts                   # typed CRUD, resume detection, pause reset

  logging/
    logger.ts                      # pino instance + secret redaction
    progress.ts                     # cli-progress bar wiring

  types/
    index.ts                       # shared MigrationItem, RunConfig, VerificationReport, etc.
```

## Module Boundaries (Provider Isolation)

Two small interfaces are the only contract the orchestrator depends on:

- **`SourceProvider`** — `listItemsRecursive()`, `streamDownload(item)`,
  `deleteItems(items)`. Implemented by `src/onedrive/`.
- **`DestinationProvider`** — `checkExisting(item)`, `upload(item,
  stream)`. Implemented separately by `src/google/drive/` (preserves
  folder structure) and `src/google/photos/` (album/library, no
  folders) — their conflict-matching rules genuinely differ (size/
  checksum vs. filename+creationTime), so they are not forced into one
  implementation.

`migration/orchestrator.ts` talks only to these interfaces plus
`state/repository.ts`. It has zero knowledge of Graph/Drive/Photos
specifics. This is what keeps retry/backoff/conflict/state logic
centralized instead of duplicated per provider, per CLAUDE.md's
"cross-cutting" guidance.

## Data Flow

1. **Entry**: `bin/cli.ts` → `src/cli/index.ts` parses flags; missing
   required choices (mode/source/dest) are collected via
   `prompts.ts`.
2. **Auth**: `auth/` ensures a valid, cached token per provider the
   chosen mode needs (MS Graph always; Google Drive or Google Photos
   depending on mode). First run opens a browser + local redirect
   listener for the PKCE exchange; later runs use the cached/refreshed
   token silently.
3. **Run identification**: the orchestrator derives a stable run key
   from (source path/ID, destination path/ID or album name, mode) and
   opens/creates the matching SQLite DB under `.state/`.
4. **Resume check**: if the DB has non-terminal rows already, resume
   from those. Otherwise (or with `--fresh`), do a full recursive
   `listItemsRecursive()` up front and write every discovered item into
   the state store as `pending`. Eager enumeration (not lazy, during
   transfer) is what guarantees recursive completeness and gives
   accurate progress totals from the start.
5. **Transfer loop**: a bounded worker pool (`concurrency.ts`, default
   3) pulls `pending` items. Per item: mark `in-progress` →
   `checkExisting()` (conflict policy decides skip/conflict/proceed) →
   if proceeding, `streamDownload()` piped directly into `upload()` (no
   disk buffering beyond a single chunk) → mark `done`/`failed`/
   `skipped` → update log + progress bar. Wrapped throughout by the
   shared `retry.ts` backoff logic.
6. **Verification pass**: once every item reaches a terminal state,
   `migration/verify.ts` re-queries the destination for every `done`
   item, compares, and produces a report (terminal + written under
   `.state/`). Mismatches get an inline interactive retry before moving
   on.
7. **Deletion prompt**: only items the verification pass *just*
   reconfirmed as present are eligible; the tool prompts once, deletion
   goes through `SourceProvider.deleteItems()`, and is logged.

## Error Handling

- **Retryable vs. not**: `migration/retry.ts` classifies errors once —
  network errors and HTTP 429/5xx retry with exponential backoff +
  jitter up to a configurable max; auth failure/403/quota-exceeded fail
  immediately. Provider clients throw typed errors so classification
  lives in one place, not duplicated per provider.
- **Per-item isolation**: a failed item (after retries exhausted) is
  caught at the worker-pool level, logged with full context (secrets
  redacted via `logging/logger.ts`), marked `failed`, and the pool
  continues — one bad item never aborts the run.
- **Graceful pause**: `SIGINT`/`SIGTERM` aborts in-flight streams, flips
  any `in-progress` rows back to `pending`, checkpoints and closes the
  SQLite handle cleanly, and prints resume instructions.
- **Fatal/setup errors**: bad auth, invalid source folder, missing
  `.env` keys, etc. are caught at the `cli/` command level with a clean
  user-facing message; raw stack traces only print with `--debug`.
- **Verification mismatches**: always surfaced in the report and
  excluded from deletion eligibility until resolved — never silently
  dropped.

## Testing Strategy

- **Unit tests** (fast, no network) for the provider-agnostic core:
  - `conflictPolicy.ts` — full decision matrix (Drive: size/checksum
    match/mismatch; Photos: creationTime match/mismatch/fallback-to-
    filename).
  - `retry.ts` — backoff/jitter timing, retryable vs. non-retryable
    classification.
  - `state/repository.ts` — resume detection, pause reset
    (`in-progress` → `pending`), state transitions.
  - `migration/verify.ts` report generation given canned "expected vs.
    actual" inputs.
  - Folder → album name derivation.
- **Provider client tests** with mocked HTTP (e.g. `msw`) rather than
  live accounts:
  - Assert pagination is followed completely (`@odata.nextLink`,
    `nextPageToken`) against multi-page fixtures.
  - Assert streaming never buffers a full file — feed a large fake
    stream, assert no temp file is written and memory stays bounded.
  - No real credentials or personal paths in any fixture.
- **E2E smoke test**: manual, documented (not CI-automated, since it
  needs live OAuth against real test accounts) — run migrate → verify →
  delete once against a throwaway OneDrive folder before releases.

## Out of Scope for This Design

- Internal design of the PKCE flow / local redirect server (follow-up
  spec).
- SQLite schema details (columns, indices) — follow-up spec or decided
  during implementation planning.
- Exact log line/report format.
- CI pipeline configuration.
