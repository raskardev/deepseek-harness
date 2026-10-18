# Developing on Windows

Upstream develops on macOS and Linux and validates Windows in CI. This page records what works in a native Windows checkout of the fork, what needs Git Bash or WSL 2, and the Windows defaults a customizer meets. Facts marked "checked" were run on Windows 11 x64, Node 24.21, pnpm 11.28.5, on 2026-10-18. [docs/development.md](../docs/development.md) (section "Windows and WSL 2") owns the general WSL guidance.

## One-time setup

1. Enable Developer Mode and run `git config core.symlinks true` before cloning or repairing the checkout; see [FORK.md](../FORK.md#local-environment-prerequisites). CI enables Developer Mode the same way. The same Windows toggle also allows installing apps from any source, including loose files; that is the cost of native symlinks, so install only packages you trust while it is on.
2. Run `corepack enable` once, or prefix every command with `corepack` (`corepack pnpm install`). Without it, `pnpm` is not on `PATH` even though `package.json` pins `pnpm@11.28.5`, and `git push` fails because the Lefthook pre-push hook calls `pnpm` directly (`pnpm: command not found`). Do not skip the hook; `corepack enable --install-directory <dir> pnpm` with `<dir>` added to `PATH` also works without elevation. `corepack enable` writes shims next to the active `node.exe`: with a version manager such as fnm that needs no elevation, while a Node install under `Program Files` needs an elevated shell.
3. Install PowerShell 7 (`pwsh`). The `dsh` shell tool uses it on Windows (see [Runtime defaults](#runtime-defaults-on-windows)); Windows PowerShell 5.1 is not a substitute.
4. If git reports "Filename too long", run `git config --global core.longpaths true`. No upstream doc mentions it; worktrees under `.claude\worktrees\` lengthen every path.

### `pnpm install`

Checked: `corepack pnpm install --frozen-lockfile` succeeds in about 2.5 minutes on a cold store and compiles nothing. No Visual Studio Build Tools or Python is needed.

| Native dependency | Used for | Windows binary |
| --- | --- | --- |
| `node-pty` | Persistent terminals through ConPTY | Prebuilt `prebuilds/win32-x64/conpty.node`, plus a bundled `conpty.dll` |
| `koffi` 3.1.1 | Win32 calls such as `MoveFileExW` for JSONL durability, the ACL sandbox, process control | Prebuilt `@koromix/koffi-win32-x64` |
| `@deepseek-ai/node-addon-system` | flock and Landlock helpers | None needed: there is no Windows platform package, and `pnpm run build:native-system` is a no-op on Windows (checked) |
| `esbuild`, `lefthook` | Bundling, git hooks | Prebuilt platform packages |

Expected warnings: "Unsupported platform" for the `native/system/packages/{darwin,linux}-*` workspaces and "cyclic workspace dependencies". The install also writes worktree-local git hooks (`pre-commit`, `pre-merge-commit`, `pre-push`) through Lefthook. Each worktree needs its own install; the pnpm store is shared, so later installs are faster.

### Line endings

`.gitattributes` sets `* text=auto eol=lf`, so git checks out every text file with LF, except `*.cmd`, which gets CRLF. Configure the editor to keep LF (`"files.eol": "\n"` in VS Code). A file saved with CRLF shows as modified, and byte-level gates (translation-pairing hashes, `verify-*` comparisons, coverage offsets) can fail on it until git renormalizes it at commit. `git diff --cached --check` in the pre-commit hook also rejects trailing whitespace and a missing final newline.

## What runs where

| Command | Native Windows | Notes |
| --- | --- | --- |
| `pnpm install`, `pnpm run build:native-system` | Yes (checked) | |
| `pnpm exec vitest run <path>` | Yes (checked: `packages/util`, 614 passed, 1 timed out) | A few Windows-only specs exceed vitest's 5 s default under parallel load; CI runs them with `--testTimeout 90000 --no-file-parallelism`. Re-run a timed-out spec alone before treating it as a regression |
| `pnpm run test:docs` | Yes (checked: 21 gates, about 2 minutes) | `verify-repository-references` fails on upstream itself: its 7-40 hex-digit pattern also matches a decimal `size` value in `python/sdk/uv.lock` (line 113) that equals an upstream commit abbreviation. Ignore that one finding; any other finding is real |
| `typecheck` (the pre-push hook) | Yes (checked: 5 gates, about 4 minutes cold) | |
| `pnpm run build`, `lint`, `doc-sync` | Expected yes: CI runs `build` and the static gates on Windows | Not run in the fork yet |
| `pnpm run check:ci:windows-blocking` | Yes, it is the Windows CI gate | Runs `build` and `docs:build`; heavy |
| `test:expected:refresh`, `test:snapshot:record`, `test:snapshot:refresh`, `test:web:refresh`, `test:web:perf:built` | No, from PowerShell or cmd | They set a `VAR=value` prefix that only POSIX shells understand. Run them from Git Bash, or set the variable with `$env:VAR = '…'` and call the underlying command |
| `make <target>` | Only if GNU make is installed | Each target is `pnpm run <script>`; run the script directly instead |
| `pnpm run check:windows-wine` | No | Linux and macOS only: it emulates Windows with Wine |
| `pnpm run test:snapshot` | Not verified | [AGENTS.md](../AGENTS.md) says fixtures replay on macOS/Linux; use WSL 2 when a native run disagrees |
| `pnpm run test` (full suite), `test:coverage` | Possible but slow | The suite targets macOS/Linux plus the Windows jobs below; prefer path-filtered runs and let CI own the rest |

Upstream CI runs three Windows jobs on pull requests, all on upstream-only runners (see [ci.md](ci.md)): `windows-build` (`check:ci:windows-blocking`, then the observational gates without failing the run), `windows-coverage` (`check:ci:coverage`), and `windows-native-tests` (four shell, PTC workflow, and subprocess specs). These jobs are the reference for what upstream supports natively.

## Runtime defaults on Windows

What changes when `dsh` itself runs on Windows (owner docs: [packages/shell/README.md](../packages/shell/README.md), [docs/subsystems/sandbox.md](../docs/subsystems/sandbox.md), [docs/subsystems/shell.md](../docs/subsystems/shell.md)):

- **Shell tool.** The base bundle turns off `bash-sandbox` and `tool-bash` on win32 and turns on `pwsh-sandbox` and `tool-pwsh` (`packages/bundle/base/cordis.patch.yml`). Each command runs as a fresh `pwsh -Command` process, so the model writes PowerShell, not bash.
- **Sandbox.** The backend is the ACL restricted-token runner ([packages/sandbox/sandbox-windows-acl/README.md](../packages/sandbox/sandbox-windows-acl/README.md)). It confines writes with a Low-integrity token and leaves reads unconfined, so it reports `partial` enforcement where macOS and Linux report full confinement.
- **No silent fallback.** When no sandbox backend is usable, commands fail with `SANDBOX_UNAVAILABLE` instead of running unconfined. Only the user-selected `danger-full-access` mode skips the sandbox.
- **Default permission mode** is `workspace-write` with approval `ask`, as on other platforms; see [customizing.md](customizing.md) for changing it.
- **Paths in YAML.** Use forward slashes (`C:/Proyectos/...`) in `cordis.yml` and patch files.
