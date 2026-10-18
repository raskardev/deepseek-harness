# Customizing DeepSeek Harness

This page routes each customization task to the upstream document that owns it and adds what those documents leave scattered: patch-row recipes, credential precedence, the `%USERPROFILE%\.dsh` layout, and PowerShell equivalents. Most customization needs no fork code change: a profile patch, a plugin outside the tree, or a file in `%USERPROFILE%\.dsh` is enough and never conflicts with upstream.

## Where each task is documented

| Task | Owner doc |
| --- | --- |
| Configuration layers and profiles | [apps/cli/README.md](../apps/cli/README.md#profiles), [apps/cli/reference/README.md](../apps/cli/reference/README.md) |
| Provider, API key, base URL, model, reasoning effort, custom OpenAI- or Anthropic-compatible provider | [docs/user/guide/providers.md](../docs/user/guide/providers.md) |
| Instructions (`AGENTS.md`, `CLAUDE.md`, `*.local.md`, 64 KiB limit) | [packages/context/agent-instructions/README.md](../packages/context/agent-instructions/README.md) |
| Skills and their discovery priority | [docs/subsystems/skills.md](../docs/subsystems/skills.md) (section "Local discovery priority") |
| MCP servers | [docs/user/guide/mcp-memory.md](../docs/user/guide/mcp-memory.md), [packages/mcp/mcp-client/README.md](../packages/mcp/mcp-client/README.md) |
| Agent presets | [packages/preset/agent-preset/README.md](../packages/preset/agent-preset/README.md) |
| Permissions, sandbox, approval | [docs/subsystems/permission-presets.md](../docs/subsystems/permission-presets.md), [sandbox.md](../docs/subsystems/sandbox.md), [approval.md](../docs/subsystems/approval.md) |
| Web Settings and the Models page | [packages/settings/settings/README.md](../packages/settings/settings/README.md), [docs/subsystems/settings.md](../docs/subsystems/settings.md) |
| Slash commands | [docs/subsystems/commands.md](../docs/subsystems/commands.md), [packages/interaction/commands/README.md](../packages/interaction/commands/README.md) |
| Client theme | [packages/client/ui-theme/README.md](../packages/client/ui-theme/README.md) |
| Plugins from the market, proxies, scheduled follow-ups | [docs/user/guide/plugin-market.md](../docs/user/guide/plugin-market.md), [network-proxy.md](../docs/user/guide/network-proxy.md), [schedule.md](../docs/user/guide/schedule.md) |
| Writing plugins | [extending.md](extending.md) (which mechanism, how to load it); [docs/user/develop/basic/index.md](../docs/user/develop/basic/index.md), [publish.md](../docs/user/develop/basic/publish.md) |
| Every plugin's config fields | [docs/config-catalog.md](../docs/config-catalog.md); `pnpm dsh web --dump-config-schema` |
| What leaves the machine, and how to stop it | [privacy.md](privacy.md) |

## Configuration layers

`dsh` composes the configuration in this order; a later layer wins per row:

1. The bundles listed in the profile's `package.json` (`dsh.profile.bundles`), in order. The `web` profile uses `base` + `web-app`, `headless` uses `base` + `headless`.
2. The profile patch: `%USERPROFILE%\.dsh\profiles\<name>\cordis.patch.yml`.
3. The home patch: `%USERPROFILE%\.dsh\cordis.patch.yml`, optional, applies to every profile.
4. Each `--patch <file>` on the command line, in argv order.

Edit `cordis.patch.yml`, not the profile's `cordis.yml` (an empty `[]` the loader owns). Web Settings and the Models page also write into the profile patch. An empty or comments-only patch file fails boot; write `[]`. With HMR on, patch edits apply live; bundle membership changes need a restart.

## Patch rows

- A patch row targets an existing row by `id`; the last layer to write a row wins. A `name:` on a targeted row only asserts which plugin it is.
- **A patch replaces the row's whole `config`.** There is no deep merge: restate every key you keep. Replacing a config with literals also drops any `!!js` expression the original had.
- `disabled: true` unmounts a row without deleting it (`true`, `false`, `null`, or `!!js`).
- New rows go under `- insert:`. Relative and absolute plugin names in inserted rows resolve beside the patch file. Row order has no load semantics.
- Find ids with `pnpm dsh web --dump-config` (the composed tree, each row annotated with the file it came from; unmatched patch targets are reported on stderr). `--dump-default-config` shows the bundles alone. A non-insert row without `id` is ignored with a warning.

### Recipes

Ids and current values come from `packages/bundle/base/cordis.patch.yml`; check them with `--dump-config` after a sync.

Disable a plugin:

```yaml
- id: web-fetch-http
  disabled: true
```

Change one field (the base row is `fetch: true`, `searchTimeoutMs: 60000`; omitting `fetch` would drop it):

```yaml
- id: tool-web
  config:
    fetch: true
    searchTimeoutMs: 90000
```

Change the default model (models: `deepseek-flash`, `deepseek-v4-pro`; keep the search and fetch providers):

```yaml
- id: agent-default-model
  config:
    provider: deepseek-official
    model: deepseek-v4-pro
    searchProvider: deepseek-official
    fetchProvider: http
```

Add a local plugin:

```yaml
- insert:
    - id: hello
      name: './plugins/hello.ts'   # relative to this patch file
      config:
        greeting: 'Hi there'
```

Add an MCP server (none is enabled by default; fields in `packages/mcp/mcp-client/README.md`):

```yaml
- insert:
    - id: mcp-github
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: github
        transport: stdio
        command: npx
        args: ['-y', '@modelcontextprotocol/server-github']
        env:
          GITHUB_TOKEN: !!js process.env.GITHUB_TOKEN
```

A custom provider replaces the `llm-pi-ai` row's whole config, which the Models page also writes; follow [providers.md](../docs/user/guide/providers.md) and keep providers already configured there.

In YAML, write Windows paths with forward slashes (`C:/plugins/hello.ts`) or in single quotes; a double-quoted `"C:\plugins"` is parsed as escape sequences. Absolute `C:/` plugin names go through Node's `isAbsolute`, which accepts both separators on Windows; no upstream test covers this.

## Credentials and environment

Provider credentials resolve in this order, first match wins:

1. `%USERPROFILE%\.dsh\.credentials.yaml` (written by the Models page; never copied into `process.env`).
2. The inherited process environment.
3. `.env` in the directory `dsh` was launched from.
4. `%USERPROFILE%\.dsh\.env`.

A key saved in the Models page therefore overrides `$env:DEEPSEEK_API_KEY`; remove it there to use the environment. `apps/cli/reference/README.md` ("Shared deployment behavior") still lists the inherited environment first; the code (`packages/credentials/credentials-local/src/index.ts`) and the v0.2.1-alpha.2 upgrade guide own the order above.

The two `.env` files are ordinary environment layers (an existing variable is never overwritten). Bootstrap variables are rejected in `.env` files and fail boot: every `DSH_*`, `DEEPSEEK_BASE_URL`, `DEEPSEEK_SEARCH_BASE_URL`, `SSL_CERT_*`, `NODE_TLS_REJECT_UNAUTHORIZED`, and similar must be set in the real environment. Proxy variables are accepted from `%USERPROFILE%\.dsh\.env` only.

| Variable | Effect |
| --- | --- |
| `DSH_HOME` | Harness home, default `%USERPROFILE%\.dsh` |
| `DSH_AGENTS_HOME` | Shared agents root, default `%USERPROFILE%\.agents` |
| `DEEPSEEK_API_KEY` | DeepSeek key for the model, search, and fetch |
| `DEEPSEEK_BASE_URL` | DeepSeek model endpoint override |
| `DEEPSEEK_SEARCH_BASE_URL` | Search endpoint override, independent of the model endpoint |
| `DSH_PERMISSION_MODE` | Sandbox mode, default `workspace-write`; `danger-full-access` also sets approval to `never` |
| `DSH_TOOLS_MODE` | `native` or `ptc`, process-wide tools mode |
| `DSH_TELEMETRY_DISABLED`, `DSH_TELEMETRY_MODE`, `DSH_TELEMETRY_OTLP_URL` | Telemetry; see [privacy.md](privacy.md) |
| `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`, `NODE_EXTRA_CA_CERTS` | Network proxy and extra CAs ([network-proxy.md](../docs/user/guide/network-proxy.md)) |

PowerShell: `$env:DSH_PERMISSION_MODE = 'workspace-write'` for the current session; `setx DSH_PERMISSION_MODE workspace-write` persists it for new terminals (not the current one).

## `%USERPROFILE%\.dsh` layout

| Path | Contents | Secret | Back up |
| --- | --- | --- | --- |
| `profiles\<name>\package.json` | `dsh.profile.bundles` and installed plugin dependencies | no | yes |
| `profiles\<name>\cordis.patch.yml` | Your profile customization, Settings and Models page writes | can hold `!!js` env reads, not keys | yes |
| `profiles\<name>\cordis.yml`, `pnpm-workspace.yaml`, `node_modules\` | Loader-owned | no | no |
| `cordis.patch.yml` | Optional patch for every profile | as above | yes |
| `.credentials.yaml` | Saved provider keys | **yes** | only to encrypted storage |
| `.env` | Optional environment layer | **can be** | only to encrypted storage |
| `AGENTS.md` | Global instructions, loaded before project ones | no | yes |
| `skills\` | User skills (priority 400, after project `.dsh/skills` 100 and `.agents/skills` 200) | no | yes |
| `sessions\--<normalized-cwd>--\<session-id>\` | zstd-compressed JSONL session logs, one folder per working directory | no keys, but full conversations, tool output, and paths | optional |
| `storages\` | JSON storage (workspace list, session projection cache) | no | optional |
| `.anonymous-user-id` | Random telemetry id; delete to reset | no | no |

## Bash in upstream docs

Upstream user docs use bash syntax. PowerShell equivalents:

| Bash | PowerShell |
| --- | --- |
| `export X=v` | `$env:X = 'v'` |
| `"$PWD"` | `"$PWD"` (same variable, a `PathInfo`; use `(Get-Location).Path` inside YAML or JSON) |
| `${DSH_HOME:-$HOME/.dsh}` | `($env:DSH_HOME ?? "$HOME\.dsh")` |
| `mkdir -p dir` | `New-Item -ItemType Directory -Force dir` |
| `$(openssl rand -hex 32)` | `-join ((1..32) \| % { '{0:x2}' -f (Get-Random -Max 256) })` |

Affected pages include `docs/user/guide/mcp-memory.md`, `github-review.md`, `network-proxy.md`, `python-sdk.md`, and `docs/user/develop/basic/index.md`.
