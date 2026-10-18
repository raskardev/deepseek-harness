# Fork guide

This checkout is a personal fork of [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) at [raskardev/deepseek-harness](https://github.com/raskardev/deepseek-harness). The upstream documentation stays authoritative for the product; this file and `FORK/` cover only what changes because the repository is a fork.

## Rules

1. **Upstream stays mergeable.** Prefer plugins, profile patches, and new files over edits to upstream files. Every edited upstream file is a future merge conflict.
2. **Never bump released persistence versions.** `SESSION_FORMAT_VERSION`, the SQLite `*_SCHEMA_VERSION` constants, and committed persistence generations belong to upstream. See [FORK/compatibility.md](FORK/compatibility.md).
3. **Never publish under `@deepseek-ai`.** Run the fork from source or rescope it first. See [FORK/compatibility.md](FORK/compatibility.md#publishing-and-identity).
4. **Keep GitHub Actions disabled** unless you have patched the workflows in [FORK/ci.md](FORK/ci.md).
5. **Fork docs live here.** `FORK.md` and `FORK/*.md` are outside every documentation gate (translation pairing, word budgets, markdown wrap and link checks, website manifest), so they need no `.zh.md` pair or `.i18n.yaml` sidecar. Do not name a file in `FORK/` `README.md`: the translation-pairing gate matches `README.md` at any depth.

## Contents

| File | Covers |
| --- | --- |
| [FORK/sync.md](FORK/sync.md) | `upstream` remote, merge policy, conflict hot spots, generated-artifact checklist |
| [FORK/ci.md](FORK/ci.md) | Verdict for each GitHub workflow when it runs on the fork |
| [FORK/compatibility.md](FORK/compatibility.md) | Session and storage compatibility, versioning, publishing, and app identity |
| [FORK/customizing.md](FORK/customizing.md) | Task → owner doc map, patch-row recipes, credentials and environment, `%USERPROFILE%\.dsh` layout |
| [FORK/privacy.md](FORK/privacy.md) | Every channel that sends data off the machine and how to turn it off |

## Local environment prerequisites

The repository tracks symbolic links (`CLAUDE.md` → `AGENTS.md`, `.claude/skills` → `.agents/skills`, several snapshot fixtures). On Windows, enable Developer Mode and set `git config core.symlinks true` before checkout; otherwise those paths become text files that contain the link target, Claude Code loads no project rules or skills, and fixture-based tests fail. Repair an existing checkout by re-checking out the affected paths after changing the setting: `git ls-files -s | grep ^120000` lists them.
