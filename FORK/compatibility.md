# Compatibility, versioning, and publishing

## Persisted data

Sessions under `%USERPROFILE%\.dsh\sessions` and SQLite stores under `%USERPROFILE%\.dsh\storages` must stay readable by both the fork and upstream builds, because a sync can move the fork to any later upstream writer at any time.

- **Do not change `SESSION_FORMAT_VERSION`** (`packages/core/session/src/types.ts`, currently `4`). Upstream assigns version numbers in sequence; a fork-assigned `5` collides with upstream's own `5`, and the two formats then share one number with different meanings. Upstream's [version cookbook](../docs/cookbook/adding-a-session-format-version.md) assumes a single writer of that sequence and does not apply to the fork.
- **Do not change SQLite schema versions** (`SESSION_QUERY_SQLITE_SCHEMA_VERSION`, `STORAGE_SQLITE_SCHEMA_VERSION`, and other `*_SCHEMA_VERSION` constants). Upstream relies on a monotonic integer per database, so the same collision applies.
- **Do not add, move, or edit files under `docs/persistence-changes/`** or the finalized checkpoints. They record upstream's accepted history.
- **Prefer no new `SessionEventMap` members.** If a fork plugin must log model-visible input (required by the "model-visible ⟺ logged" rule in `AGENTS.md`), use a plugin-owned event that carries `ignorable: true`, so upstream readers that do not know it skip it instead of rejecting the session. Read the [reader-compatibility rules](../docs/persistence-changes/README.md#compatibility-rules) first.
- **Keep fork-only state in plugin-owned storage** (a separate storage name or file under `%USERPROFILE%\.dsh\storages`) instead of extending upstream tables.
- When upstream bumps a version during a sync, take upstream's change unmodified and let its adjacent migration convert existing data. Back up `%USERPROFILE%\.dsh\sessions` before the first run of a build that crosses a version.

[Session format status](../docs/session-format-status.md) owns the current writer and released versions.

## Versioning

- The root `release:*` scripts (`scripts/release/bump.ts`, `publish.ts`) bump upstream's `dsh` and `vendor` families and create `dsh-v*` tags. Do not run them in the fork: the fork's version numbers would collide with upstream releases on the next sync.
- Leave package versions as upstream sets them. Identify a fork build by commit (`git describe --always`) instead.
- If the fork needs its own tags, use a prefix upstream never uses, for example `fork-v<upstream-version>-<n>` (`fork-v0.2.1-rc.2-1`), and never push `dsh-v*`, `vendor-*-v*`, `python-v*`, or `node-addon-system-v*` tags.

## Publishing and identity

- Never publish packages under the `@deepseek-ai` scope or the upstream PyPI names, and never run the publishing workflows (see [ci.md](ci.md#workflow-verdicts)).
- The supported way to use the fork is from source: `pnpm install`, `pnpm run build:artifacts`, then `pnpm dsh ...` (see [docs/development.md](../docs/development.md)).
- Distributing the fork as packages requires a rescope of every workspace package first; [docs/rescope.md](../docs/rescope.md) documents how upstream rescoped the vendored Cordis packages.
- Model requests identify the app through `APP_IDENTITY` in `packages/llm/llm/src/attribution.ts` (`deepseek-harness`, upstream URL) in the `User-Agent` header. Attribution cannot be suppressed; a deployment that needs its own identity passes an `AppIdentity` to `attributionHeaders`. Changing the constant itself is an edit to an upstream file; keep it only if the fork is distributed under another name.
