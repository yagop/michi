# michi

A Claude Code plugin for **GitHub-issue-driven development**. You write an issue; Claude reads it, plans the work, implements it task by task in a **draft PR**, and keeps everything in sync as it goes — pushing the branch and ticking the plan after each task. Runs are **resumable** — stop anytime and pick up where you left off.

## What it does

Invoke `/issue <number>` and the plugin will:

1. **Read state** — find the work (existing draft PR → `🐱 Michi` issue comment → fresh) and detect fresh start vs. resume. The issue *body* is never touched.
2. **Plan** (fresh start) — decompose the issue into small, independently committable tasks, each with a stable id, and post the checklist as a `🐱 Michi` issue comment.
3. **Implement** — one task at a time: commit (tagged `T<n>: … (#<issue>)` with `Michi-Task` trailers) → **push** the `issue-<n>` branch → tick the task.
4. **Track in a draft PR** — after the first push, open a draft PR (`Closes #<issue>`) and move the plan into its body; from then on the PR is the live home for plan + progress, mirrored into Claude Code's TodoWrite.
5. **Wrap up** — mark the PR **ready for review** and post a summary. It does **not** merge, force-push, or close the issue.

### Resumable by design

The task list lives in a marker block — in the draft PR body once it exists, or the `🐱 Michi` issue comment before then:

```
<!-- michi:plan -->
- [ ] T1 <!--m:a1b2--> …
- [x] T2 <!--m:c3d4--> …
<!-- /michi:plan -->
```

**Git is the source of truth for what's done** — a task is complete iff a commit carries its id (`Michi-Task: <id>` trailer). The checkboxes are just a cache of that. On a new run the plugin re-derives done-ness from the commit log, refreshes the checkboxes to match, and resumes at the first not-done task. Nothing is re-done, and a failed sync self-heals on the next run.

## Requirements

- [`gh`](https://cli.github.com/) installed and authenticated (`gh auth status`).
- `git`, and you must run the command from inside the target repository.
- An `in-progress` label in the repo is nice-to-have (the run continues without it).

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

## Safety

The command is scoped via `allowed-tools` to specific `gh`/`git` subcommands. Within that scope it **will** push its own `issue-<n>` branch, open/maintain a **draft** PR, and mark that PR ready when every task is done. It will **never** merge, force-push, push your default branch, or close the issue — those stay yours. (The issue closes automatically when *you* merge the PR, via `Closes #<issue>`.) Work is one commit per task on a dedicated branch, so it's easy to inspect, and the draft PR is easy to close if you don't want it.

> **Note:** this is a behavior change from earlier versions, which kept everything local and never pushed. michi now pushes to a non-default branch and opens a draft PR automatically.

## License

MIT — see [LICENSE](LICENSE).
