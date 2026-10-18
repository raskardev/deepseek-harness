# Working on the fork with Claude Code

The repository is set up for coding agents: `CLAUDE.md` links to `AGENTS.md`, and `.claude/skills` links to the 19 project skills in `.agents/skills/`. Both links need the symlink setup in [FORK.md](../FORK.md#local-environment-prerequisites); without it, Claude Code loads neither the rules nor the skills. This page says which skills apply unchanged to the fork and gives a local permission allowlist.

## Project skills

Claude Code loads each skill when its description matches the task; a skill can also be invoked by name (`/dsh-doc`). Most skills assume the upstream workflow: GitHub stacked PRs, a designer, CI on upstream runners. On the fork, use the parts that do not depend on that.

| Skill | On the fork |
| --- | --- |
| `dsh-pre-push-checks` | Use. Picks the smallest local checks for a diff. Skip the `gh stack` and `gh pr checks` sections: the fork has no stacks and no CI ([ci.md](ci.md)) |
| `dsh-doc`, `dsh-prose-standard`, `dsh-trim-cot-leakage` | Use for any change under `docs/`, package READMEs, JSDoc, or comments. `FORK/` pages need no bilingual pair, but the prose rules still help |
| `dsh-error-handling`, `dsh-ci-test-reliability`, `cordis-styling-nitpicking` | Use when writing or reviewing plugins, tests, or lifecycle code |
| `dsh-code-review` | Use for reviewing a fork branch; run `change-scope` against `master` of the fork, not upstream |
| `dsh-find-simplifications`, `agent-experience`, `dsh-speed-up-perf` | Use as written |
| `dsh-client-ui-ux`, `dsh-responsive-ui` | Use for client UI changes; ignore the designer-review steps unless you have one |
| `dsh-create-upgrade-guide` | Use only for changes you will contribute upstream. Fork-only breaking changes go in your own notes, not in `docs/upgrade-guide/` (an upstream merge-conflict hot spot) |
| `record-browser-gif` | Recording works locally; the `gh --attach` publishing step targets the PR you open, so it applies only to PRs you send upstream |
| `dsh-merging-stacked-prs` | Do not use: it requires GitHub's stacked-PR feature on the upstream repository. The fork merges ordinary PRs ([sync.md](sync.md)) |
| `dsh-archive-agent-notes` | Upstream maintenance of `.agents/notes/`; use only if you add Agent Notes to a change you contribute upstream |
| `dsh-official-plugin-artwork` | Upstream-only: official icons need upstream designer approval |
| `dsh-translate-docs` | Runs only when invoked explicitly by name (`disable-model-invocation`); upstream translation workflow |

Skills written for the fork itself belong in `.agents/skills/fork-<name>/` (they appear through the same `.claude/skills` link) or in `%USERPROFILE%\.claude\skills\` when they are not repository-specific. A `fork-` prefix keeps them apart from upstream skills during a sync.

## Local permission allowlist

`.claude/settings.json` is gitignored in this repository (`.gitignore`), so a project-level permission file stays local and never conflicts with upstream. The fork checkout uses one that:

- **allows** read-only git commands and the bounded checks: `typecheck`, `lint`, `test:docs`, `doc-sync`, `build`, `build:artifacts`, `build:check`, `build:native-system`, `hygiene`, `duplication`, `change-scope`, `verify-*`, and path-filtered `vitest run <path>` / `run test <path>`;
- **asks** before the full suite (`run test` and `exec vitest run` with no path), `test:coverage`, `test:e2e*` and `test:snapshot:record` (real API calls, paid with your key), `test:expected:refresh`, `pnpm install`, and `pnpm dsh` (starts the product);
- **denies** reading `.env`, `.env.*`, `%USERPROFILE%\.dsh\.env`, and `.credentials.yaml` files.

Each rule exists four times: for the `Bash` and `PowerShell` tools, and with `pnpm` and `corepack pnpm`, because [windows.md](windows.md) explains that `pnpm` may only be reachable through Corepack. A shortened excerpt:

```json
{
  "permissions": {
    "allow": [
      "Bash(git status*)",
      "Bash(pnpm run typecheck)",
      "Bash(pnpm run verify-*)",
      "Bash(pnpm exec vitest run *)",
      "PowerShell(corepack pnpm exec vitest run *)"
    ],
    "ask": [
      "Bash(pnpm run test)",
      "Bash(pnpm run test:e2e*)",
      "Bash(pnpm dsh*)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(~/.dsh/.credentials.yaml)"
    ]
  }
}
```

Precedence is deny, then ask, then allow, so the exact `run test` ask rule wins over the `run test *` allow rule only when no path follows. `Read` deny rules do not stop a shell command such as `Get-Content .env`; they only keep the file tools away from secrets. Check the loaded rules with `/permissions` after editing the file. A worktree under `.claude\worktrees\` is a separate checkout without the ignored file; if `/permissions` in a worktree session does not list the rules, copy the file into that worktree's `.claude\` folder.
