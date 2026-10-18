# GitHub Actions on the fork

## Current state

GitHub Actions is allowed on `raskardev/deepseek-harness`, but no workflow has been registered or run (checked 2026-10-18 with `gh run list` and `gh api repos/raskardev/deepseek-harness/actions/workflows`). GitHub keeps inherited workflows inactive on a fork until someone enables them in the **Actions** tab. Keep them inactive: the upstream workflows assume upstream runners, repository variables, secrets, and GitHub Apps that the fork does not have.

Local checks remain the verification source for fork work; see `.agents/skills/dsh-pre-push-checks/SKILL.md`.

## Why workflows break on a fork

- **Runner labels.** Most heavy jobs choose a runner with `vars.DSH_CI_FAILOVER_LINUX` / `vars.DSH_CI_FAILOVER_WINDOWS` (`blacksmith` or `selfhosted`) and otherwise fall back to upstream organization larger runners such as `dsh-ubuntu-24-04-16core` and `dsh-windows-2025-16core`. A personal fork has none of these, so the job waits in the queue until it times out.
- **Secrets.** `DEEPSEEK_API_KEY_EXTERNAL`, `NPM_TOKEN`, `CLOUDFLARE_*`, `CF_ACCESS_*`, `DSH_ISSUE_APP_PRIVATE_KEY`, and provider keys are upstream secrets.
- **Repository gates.** `release.yml` and `release-vendor.yml` reach self-hosted runners only when `github.repository == 'deepseek-harness/deepseek-harness'`; elsewhere they fall back to `ubuntu-24.04`.

## Workflow verdicts

Triggers and runners as of upstream `687f282e86`. "Stuck" means the job requests a runner label the fork cannot provide.

| Workflow | Trigger | Verdict on the fork | Action if Actions is enabled |
| --- | --- | --- | --- |
| `ci.yml` | `pull_request` | Mixed: static, coverage, snapshots/artifacts and all Windows jobs are stuck (`dsh-*-16core`); the `ubuntu-latest` matrix jobs and `python-sdk` run | Patch the fallback labels to `ubuntu-latest` / `windows-latest`, or disable |
| `ci-master.yml` | push to `master`, dispatch | Stuck: self-hosted and `dsh-*` runner matrix (benchmarks) | Disable |
| `e2e.yml` | push to `master`, PR, daily schedule | Fails: push and schedule runs need `DEEPSEEK_API_KEY_EXTERNAL`; the preflight hard-fails without it | Disable, or add your own key as that secret and drop the schedule |
| `sandbox.yml` | push to `master` | Runs on GitHub-hosted Linux and macOS (bwrap, landlock, seatbelt) | Optional; keep only if the fork changes `packages/sandbox` |
| `node-addon-system.yml` | PR/push touching `native/system/**` or lockfiles | Runs on GitHub-hosted runners | Optional |
| `release.yml`, `release-vendor.yml` | PR, push to `master`, dispatch | Run on `ubuntu-24.04`: release rehearsals, no publication | Disable unless the fork releases |
| `release-publish.yml`, `release-vendor-publish.yml`, `node-addon-system-release.yml`, `python-release.yml`, `publish-ssh-helper.yml` | dispatch | Publish to npm, PyPI, or GitHub Releases with upstream secrets and names | Disable; never run under `@deepseek-ai` (see [compatibility.md](compatibility.md#publishing-and-identity)) |
| `desktop-publish.yml`, `macos-package.yml`, `windows-package.yml` | dispatch | Stuck: self-hosted packaging and signing machines | Disable |
| `build-preview-cloudflare.yml` | `pull_request` | Fails: needs Cloudflare secrets | Disable |
| `docs-pages.yml` | dispatch | Stuck: `blacksmith-8vcpu-ubuntu-2404` | Disable |
| `issue-lifecycle.yml`, `issue-policy.yml` | issues, PRs, reviews | Fail: need the upstream issue GitHub App key | Disable |
| `weighted-approval.yml`, `weighted-approval-review-event.yml` | `pull_request_target`, comments, reviews | Upstream review-approval policy | Disable |
| `expected-filenames.yml` | PR touching files named `*golden*` | Runs on `ubuntu-latest` | Harmless |
| `build-exe-for-python-sdk.yml`, `build-exe-for-ssh-helper.yml` | `workflow_call`, dispatch | Called by release workflows; GitHub-hosted fallback | Follows its callers |
| `pi-ai-provider-e2e.yml` | dispatch | Needs Azure OpenAI and Anthropic secrets | Disable |

## Enabling a minimal CI

If the fork needs CI, enable Actions and then disable every workflow except the ones you need with `gh workflow disable <file> -R raskardev/deepseek-harness`. For `ci.yml`, replace each `'dsh-ubuntu-24-04-16core'` fallback with `'ubuntu-latest'` and each `'dsh-windows-2025-16core'` with `'windows-latest'` in a single fork commit, and expect longer runs on standard runners. That commit conflicts whenever upstream edits the same `runs-on` expressions; resolve by taking upstream and re-applying the label replacement.
