# Extending DeepSeek Harness

This page chooses the mechanism for a new behavior before any code is written. Upstream documents each mechanism; it does not say which one to pick, how to run a plugin outside the tree at each stage of its life, when a model provider needs code, or where a Web UI plugin can live. Configuration-only changes are covered in [customizing.md](customizing.md).

## Pick the lightest mechanism

Prefer the first row that fits. Rows above the plugin rows need no code and never conflict with upstream.

| You want to | Use | Owner doc |
| --- | --- | --- |
| Change a model, provider, plugin option, or turn a plugin off | A profile patch row | [customizing.md](customizing.md#patch-rows) |
| Give the agent standing instructions | `AGENTS.md` / `CLAUDE.md` in the workspace or `%USERPROFILE%\.dsh` | [packages/context/agent-instructions/README.md](../packages/context/agent-instructions/README.md) |
| Teach a procedure the model loads on demand | A skill folder | [docs/subsystems/skills.md](../docs/subsystems/skills.md) |
| Expose an external tool or service, in any language | An MCP server | [docs/user/guide/mcp-memory.md](../docs/user/guide/mcp-memory.md) |
| Reuse Claude Code or Codex hook config files | The experimental hook bridges | [packages/experimental/hooks-claude-code/README.md](../packages/experimental/hooks-claude-code/README.md), [hooks-codex](../packages/experimental/hooks-codex/README.md) |
| Add a model-callable tool, a hook on `tools/pre-execute` or `agent/*` events, a system-prompt section, a slash command, or a service | A server-side plugin | [docs/user/develop/basic/index.md](../docs/user/develop/basic/index.md), [docs/cookbook/extension-cookbook.md](../docs/cookbook/extension-cookbook.md) |
| Add a Web UI page, settings card, or chat renderer | A client plugin (browser half) | [Web client plugins](#web-client-plugins) |
| Reach a model whose API is not OpenAI Chat Completions, OpenAI Responses, or Anthropic Messages | An LLM adapter plugin | [Do you need an LLM adapter?](#do-you-need-an-llm-adapter) |
| Change `agent-loop`, session persistence, or a core service | A fork edit of upstream code | [FORK.md](../FORK.md#rules), [compatibility.md](compatibility.md) |

The [feature → mechanism map](../docs/cookbook/extension-cookbook.md) in the extension cookbook lists which extension point each shipped product feature uses; start there before writing a server-side plugin.

## Running a plugin outside the tree

A plugin does not need to live in `packages/`. Keep fork plugins outside the tree, or in a top-level directory that upstream does not own, so syncs never touch them. Pick the mode by the plugin's stage:

| Stage | Mode | How it loads | Build step | Owner doc |
| --- | --- | --- | --- | --- |
| Prototype | A single `.ts` or `.js` file inserted by a `--patch` overlay | `- insert: [{ id: x, name: 'C:/abs/path/plugin.ts' }]`, launched with `pnpm dsh web --patch <overlay>` | None under the source launch (`tsx`); compile to `.js` for an installed `dsh` | [docs/user/develop/basic/index.md](../docs/user/develop/basic/index.md) |
| Daily use on this machine | A bundle package linked into a profile | `pnpm dsh plugin --profile web add C:/path/to/pkg` writes `link:` and appends the bundle | Build `lib/` yourself after each change | [publish.md](../docs/user/develop/basic/publish.md#install-into-a-profile) |
| Share from a git host | `github:<user>/<repo>#<sha>` | `dsh plugin add github:...` | The package's `prepare` script; the user allowlists it in the profile's `pnpm-workspace.yaml` (`allowBuilds`) | [publish.md](../docs/user/develop/basic/publish.md#installing-from-github-the-build-script-catch) |
| Share without build permissions | A `pnpm pack` tarball or an npm package with `lib/` prebuilt | `dsh plugin add ./pkg-0.1.0.tgz` or `dsh plugin add <name>` | At pack or publish time | [publish.md](../docs/user/develop/basic/publish.md#installing-from-github-the-build-script-catch) |

Facts that apply to every mode:

- **Paths.** A plugin path in a `--patch` overlay must be absolute; a patch does not change the directory the loader resolves modules from. In YAML, prefer forward slashes (`C:/Proyectos/...`), which Node accepts on Windows and which need no escaping.
- **Bundle manifest.** A package becomes a layer only when its `package.json` declares `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }`; without it, `dsh plugin add` installs a plain dependency and warns.
- **Shared instances.** Declare `@deepseek-ai/cordis` and every `@deepseek-ai/dsh-*` package whose instance must be shared with the host in both `peerDependencies` and `devDependencies`.
- **Version range.** Set `"engines": { "dsh": ">=0.2.0 <0.3.0" }` (match the fork's version line); profile boot refuses a bundle outside its range. The check includes prereleases, so `0.2.1-rc.2` satisfies that range.
- **Verify without booting.** `pnpm dsh --profile <name> --dump-config` shows a `# == <package>` layer for each installed bundle.
- **Row config replaces.** A bundle row that overrides a base row restates the whole `config`; see [customizing.md](customizing.md#patch-rows).
- **Plugins run with your full user rights.** A plugin executes inside the `dsh` process, outside the sandbox that confines the agent's shell commands, and can read credentials in `%USERPROFILE%\.dsh`. Load only code you have read.
- **`allowBuilds` runs code at install time.** Allowlisting a git package's `prepare` script lets it execute on your machine during `dsh plugin add`, outside any sandbox. Allow only sources you trust and pin a full commit (`#<sha>`) so a later push cannot change what runs.
- **Never publish under `@deepseek-ai`.** Use your own scope or an unscoped `dsh-*` name ([compatibility.md](compatibility.md#publishing-and-identity)).

## Do you need an LLM adapter?

Usually not. An adapter is code that translates the harness's provider-neutral request into a wire protocol; `dsh-llm-pi-ai` already speaks the common ones.

| Situation | What to do | Code? |
| --- | --- | --- |
| A provider in the installed pi-ai catalog (`anthropic`, `openai`, `moonshotai`, `zai`, …) | Settings → Models → **Add model provider** → **Third-party model provider** | No |
| A relay, company gateway, or self-hosted server that speaks OpenAI Chat Completions, OpenAI Responses, or Anthropic Messages | Settings → Models → **Custom model API** (`api: openai-completions`, `openai-responses`, or `anthropic-messages`) | No |
| A gateway that serves two of those protocols | Two custom providers, one per protocol | No |
| Headers, timeouts, retries, request-compatibility switches, or reasoning levels of a built-in model | A `dsh-llm-pi-ai` row in the profile patch | No |
| A keyless local server (OpenAI-compatible) | A custom provider with a placeholder credential (`apiKeyEnv`) or an `Authorization` entry in `headers` | No |
| A wire protocol none of the above speaks, or provider-native replay state (response ids, signatures) that pi-ai does not carry | A new `LlmAdapter` plugin | Yes |

Owner docs: [providers.md](../docs/user/guide/providers.md) for the no-code rows, [packages/llm/llm-pi-ai/README.md](../packages/llm/llm-pi-ai/README.md) for route fields and limits, and for the last row [docs/user/develop/practice/llm-adapter.md](../docs/user/develop/practice/llm-adapter.md) plus the protocol obligations in [docs/cookbook/adding-an-llm-adapter.md](../docs/cookbook/adding-an-llm-adapter.md). An adapter registers provider routes with `ctx.llm.registerAdapter()`; registering a route another adapter already owns fails plugin loading, so pick route names that neither `dsh-llm-deepseek` nor `dsh-llm-pi-ai` uses.

## Web client plugins

A Web UI extension is the browser half of a plugin package. The Host serves it from the package's built `./client` export; nothing in the browser runs until the plugin is first used.

What the package needs (owner: [packages/client/AGENTS.md](../packages/client/AGENTS.md#new-plugin-package-checklist), [packages/client/modules/README.md](../packages/client/modules/README.md)):

- `package.json` declares `"dsh": { "client": { "platform": "web" } }` and exports `./client`.
- The bundle is built to `lib/client.js` with the shared tsdown preset (`packages/client/tsdown.client.ts`), which externalizes the shell's shared module table (`PLATFORM_MODULES`: React, Cordis, and selected client libraries). Do not bundle a private copy of React.
- The browser half attaches only to a Loader row whose `name` is the bare package name; a row mounted from a subpath export carries no browser half.
- UI copy goes through the typed locale dictionaries and `t`; theme tokens come from [packages/client/ui-theme](../packages/client/ui-theme/README.md).

Task recipes: [adding-a-settings-card.md](../docs/cookbook/adding-a-settings-card.md) for a settings page, the "UI plugin" and "Web Client Chat business node" entries of [extension-cookbook.md](../docs/cookbook/extension-cookbook.md) for chat rendering, and [docs/subsystems/client-modules.md](../docs/subsystems/client-modules.md) for the boot graph.

Where to put it in the fork:

| Location | Works today | Cost |
| --- | --- | --- |
| `packages/experimental/<name>/`, already matched by the `packages/*/*` workspace glob and exempt from [product-package classification](../scripts/product-package-policy.ts) | Yes: same build preset, `verify-client-packages` checks, and `pnpm run build` as upstream client plugins | Every package gate applies: a `README.md` with its `README.zh.md` pair and `.i18n.yaml` sidecar, `hygiene`, and the 100% per-file coverage of `test:coverage`. A name upstream later uses becomes a merge conflict |
| A workspace directory upstream does not own (for example `fork/<name>/`, added to `pnpm-workspace.yaml`) | Untested: the shared preset and tsconfig bases should resolve by relative path, but repository gates may not cover or may reject the directory | One edited upstream file (`pnpm-workspace.yaml`) |
| A separate repository linked with `dsh plugin add` | Not documented upstream and not tested in the fork: the build preset and type configs live in this monorepo and would have to be reproduced | No upstream files touched |

Every client change needs `pnpm run build` before launch; the Host fails startup with a build instruction when a `lib/client.js` is missing, but it does not detect stale bundles.
