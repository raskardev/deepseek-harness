# What leaves the machine

The base bundle sends data to DeepSeek beyond the model conversation itself, some of it **on by default**. This page lists every channel found in upstream release 0.2.1-rc.2, what it sends, and how to turn it off. Rows are in `packages/bundle/base/cordis.patch.yml` (base) and `packages/bundle/web-app/cordis.patch.yml` (web-app); [customizing.md](customizing.md#patch-rows) explains the patch grammar.

## Channels

| Channel | Default | Data | Destination | Off switch |
| --- | --- | --- | --- | --- |
| Session-log upload (`session-log-deepseek`) | **On** in every base-backed profile | The full session log (header with cwd, then every event: messages, tool arguments and results, file paths), as a `dsh_session_log` field inside each official DeepSeek model request, up to 8 MiB per request | The `llm-deepseek` route's `baseURL` (default `https://api.deepseek.com/anthropic`), including a gateway configured there; not sent on `llm-pi-ai` routes | Web Settings → General → "Upload Session Log when using the official model API"; or the row's `config.enabled: false` / `disabled: true`. No env var |
| Plugin inventory (`plugin-package-inventory-deepseek`) | **On** | Names and versions of loaded plugin packages, as `dsh_plugin_packages` in the same model requests | Same as above | Row `disabled: true` or `config.enabled: false`. No UI toggle or env var found |
| Anonymous user id header | **On**, unconditional | `x-deepseek-harness-user-id`: a random UUID stored in `%USERPROFILE%\.dsh\.anonymous-user-id` | Every `llm-deepseek` request | None. Delete the file to rotate the id |
| Feedback OTel upload (`session-telemetry-otel`) | On in `FEEDBACK_ONLY` mode: sends only after you submit feedback | The session-log prefix up to the feedback (message text, tool arguments and results, paths) plus the anonymous id | `https://dsh-otel-collector.deepseeksvc.com/v1/logs`, or `DSH_TELEMETRY_OTLP_URL` | `DSH_TELEMETRY_DISABLED` set to any non-empty value (even `0`), which makes the launcher patch the row `disabled: true`; or the row `disabled: true`. `DSH_TELEMETRY_MODE=DISABLED` is weaker: the row stays mounted and only logs a warning on feedback |
| Desktop product analytics (`desktop-product-telemetry`, `product-analytics`) | **On** in the `desktop` profile only | Interaction events, device id, login user id, app version, OS, CPU model, core count, total memory; no prompts, responses, or keys | `DSH_PRODUCT_ANALYTICS_OTLP_URL` or the same collector by default | Both rows `disabled: true`. Telemetry env vars do not affect it |
| Web search (`web-search-deepseek`, `web-deepseek-api`) | On when the agent has the search tool | Queries written by the model | `https://api.deepseek.com/anthropic/v1`, `https://api.deepseek.com/v1/tools` with the new Search API switch, or `DEEPSEEK_SEARCH_BASE_URL` | See [Web tools](#web-tools) |
| Web fetch (`web-fetch-http`) | On when the agent has the fetch tool | Direct GET requests to public URLs the model picks, with a `deepseek-harness` User-Agent | The target site | See [Web tools](#web-tools) |
| Desktop auto-update | Packaged Desktop only | Platform, architecture, app and bundled `dsh` versions, locale, UTC offset; no installation id | The update feed | No off switch found; env vars only tune the check interval |
| User-Agent attribution | Always | `deepseek-harness/<version> (+upstream URL)`, no user data | The model provider | Cannot be suppressed (`packages/llm/llm/src/attribution.ts`) |

Not on by default: MCP servers (none enabled), the DeepSeek account sign-in (`deepseek-account`, contacts `platform.deepseek.com` only when you sign in), and `dsh plugin add` (npm registry on demand).

## Opt-out patch

Append to `%USERPROFILE%\.dsh\cordis.patch.yml` (every profile) or a profile's `cordis.patch.yml`:

```yaml
# Uploads that ride official DeepSeek model requests
- id: session-log-deepseek
  disabled: true
- id: plugin-package-inventory-deepseek
  disabled: true
# Feedback-gated session upload (same as DSH_TELEMETRY_DISABLED)
- id: session-telemetry-otel
  disabled: true
# Desktop product analytics (no effect outside the desktop profile)
- id: desktop-product-telemetry
  disabled: true
- id: product-analytics
  disabled: true
```

If the file did not exist, write the rows without a leading `[]`; if it is `[]`, replace the `[]`. The Settings toggle for the session log writes the same `session-log-deepseek` row into the profile patch, so a home-level `disabled: true` keeps it off whatever the toggle shows. This patch has not been boot-tested in the fork; confirm with `pnpm dsh web --dump-config` (unmatched ids are reported on stderr) and one session.

Environment-only alternative, which covers the feedback OTel channel only:

```powershell
$env:DSH_TELEMETRY_DISABLED = '1'   # current terminal
setx DSH_TELEMETRY_DISABLED 1       # new terminals and apps; quit and restart Desktop
```

## Web tools

Search and fetch send model-chosen queries and URLs out by design. In non-web profiles, disable the host tool row:

```yaml
- id: tool-web
  disabled: true
```

In the `web` and `desktop` profiles the base `tool-web` row is already disabled and the `standard`, `cordis`, and `ptc` agent presets mount their own `tool-web` children inside the preset row's `config.plugins`; the `minimal` preset has none. Upstream documents no way to patch a preset's child by id, so the options are choosing the `minimal` preset or disabling the provider rows (`web-search-deepseek`, `web-deepseek-api`, `web-fetch-http`). The second option leaves the tools visible but without a provider, and `agent-default-model` still names `searchProvider: deepseek-official`; neither has been tested, and boot may reject the missing referent.

## What stays

With every switch above off, the model requests themselves still go to the configured provider with the `x-deepseek-harness-user-id` header (official DeepSeek adapter only) and the User-Agent. To keep conversations off DeepSeek entirely, use another provider through `llm-pi-ai` ([providers.md](../docs/user/guide/providers.md#add-a-custom-model-api)); session-log and inventory fields are not attached on those routes.
