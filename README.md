# michi

A Claude Code plugin for **GitHub-issue-driven development**. You write an issue; Claude reads it, plans the work, implements it task by task in a **draft PR**, and keeps everything in sync as it goes — pushing the branch and ticking the plan after each task. Runs are **resumable** — stop anytime and pick up where you left off.

## What it does

Invoke `/issue <number>` and the plugin will:

1. **Read state** — find the work (existing draft PR for the `issue-<n>` branch → fresh) and detect fresh start vs. resume. The issue *body* is never touched.
2. **Plan & open a draft PR** (fresh start) — decompose the issue into small, independently committable tasks (each with a stable id), set up an isolated **git worktree** for the `issue-<n>` branch (your current checkout is never touched), open a draft PR up front (`Closes #<issue>`) with the checklist in its body, and drop a one-line pointer comment on the issue: `😺 Michi — started work in <PR> 🐾`.
3. **Implement** — one task at a time, looping **execute → verify → correct** until it's green (it won't commit broken code): then commit (tagged `T<n>: … (#<issue>)` with `Michi-Task` trailers) → **push** the `issue-<n>` branch → tick the task in the PR body. Progress is mirrored into Claude Code's TodoWrite.
4. **Wrap up** — **wait for CI** and push bounded fix commits until it's green (if it can't fix it after a few honest attempts, or the failure is flaky/unrelated, it stops and leaves the PR draft for you), then mark the PR **ready for review** and post a summary. It does **not** merge, force-push, or close the issue.

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

The command is scoped via `allowed-tools` to specific `gh`/`git` subcommands. It works in a dedicated `issue-<n>` **git worktree**, so your current checkout — branch, staged changes, untracked files — is never touched (no branch switch, no stash). Within that scope it **will** push its own `issue-<n>` branch, open/maintain a **draft** PR, wait for CI and push bounded fix commits to make it pass, and mark that PR ready when every task is done and CI is green. It will **never** merge, force-push, push your default branch, or close the issue — those stay yours. (The issue closes automatically when *you* merge the PR, via `Closes #<issue>`.) Work is one commit per task in a dedicated worktree+branch, so it's easy to inspect, and the draft PR is easy to close if you don't want it. The worktree stays in place for resumability; removing it (`git worktree remove`) is yours, like merging.

> **Note:** this is a behavior change from earlier versions, which kept everything local and never pushed. michi now pushes to a non-default branch and opens a draft PR automatically.

## License

MIT — see [LICENSE](LICENSE).
