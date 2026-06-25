# michi 😺

A Claude Code plugin for **GitHub-issue-driven development**. Write an issue, run `/issue <number>`, and michi plans the work, implements it task by task in a **draft PR**, and keeps everything in sync as it goes. Stop anytime — runs are **resumable**. 🐾

## What it does ✨

`/issue <number>` runs four steps:

1. **📖 Read state** — find any existing draft PR for the `issue-<n>` branch and decide: fresh start or resume. Your issue body is never touched.
2. **📝 Plan** — break the issue into small, independently committable tasks, open a draft PR (`Closes #<issue>`) with the checklist in its body, and drop a pointer comment on the issue.
3. **🛠️ Implement** — one task at a time, looping **execute → verify → correct** until green (never commits broken code), then commit → **push** the `issue-<n>` branch → tick the task.
4. **🚦 Wrap up** — wait for CI, push bounded fix commits until it's green, then mark the PR **ready for review**. It never merges, force-pushes, or closes the issue.

The task list lives in **one place — the PR body** — so there's nothing to keep in sync between the issue and the PR.

### Resumable by design

The task list lives in a marker block in the **draft PR body**:

```
<!-- michi:plan -->
- [ ] T1 <!--m:a1b2--> …
- [x] T2 <!--m:c3d4--> …
<!-- /michi:plan -->
```

**Git is the source of truth for what's done** — a task is complete iff a commit carries its id (`Michi-Task: <id>` trailer). The checkboxes are just a cache of that. On a new run the plugin re-derives done-ness from the commit log, refreshes the checkboxes to match, and resumes at the first not-done task. Nothing is re-done, and a failed sync self-heals on the next run.

## Requirements

- [`gh`](https://cli.github.com/) installed and authenticated (`gh auth status`).
- `git` **≥ 2.34** (for native `--trailer` and `%(trailers:…)` parsing), run from inside the target repository.

## Install

From within Claude Code:

```
/plugin marketplace add yagop/michi
/plugin install michi@michi
```

Or, to try it from a local clone:

```
/plugin marketplace add /absolute/path/to/michi
/plugin install michi@michi
```

## Usage

```
/issue 123                 # work issue #123 in the current repo
/issue 123 owner/repo      # target a specific repo
```

Run it again on the same issue at any time to resume.

## Agent Skill

michi is a Claude Code **plugin**, and its `/issue` workflow runs through Claude Code's
[Agent Skills](https://code.claude.com/docs/en/skills) system — Claude Code unifies slash
commands and skills, so once installed michi is surfaced as the **`michi:issue`** skill and
invoked as `/issue <number>`. It's a deliberately **user-invoked** workflow: you run it on a
specific issue. That's the right shape for a side-effecting command (it pushes branches and
opens PRs) — you don't want it auto-firing.

## Safety

The command is scoped via `allowed-tools` to specific `gh`/`git` subcommands. Within that scope it **will** push its own `issue-<n>` branch, open/maintain a **draft** PR, wait for CI and push bounded fix commits to make it pass, and mark that PR ready when every task is done and CI is green. It will **never** merge, force-push, push your default branch, or close the issue — those stay yours. (The issue closes automatically when *you* merge the PR, via `Closes #<issue>`.) Work is one commit per task on a dedicated branch, so it's easy to inspect, and the draft PR is easy to close if you don't want it.

> **Note:** this is a behavior change from earlier versions, which kept everything local and never pushed. michi now pushes to a non-default branch and opens a draft PR automatically.

## License

MIT — see [LICENSE](LICENSE).
