# michi

A Claude Code plugin for **GitHub-issue-driven development**. You write an issue; Claude reads it, plans the work, implements it task by task, and keeps the issue (and Claude Code's task list) in sync as it goes. Runs are **resumable** — stop anytime and pick up where you left off.

## What it does

Invoke `/issue <number>` and the plugin will:

1. **Read state** — fetch the issue and detect whether this is a fresh start or a resume.
2. **Plan** (fresh start) — decompose the issue into small, independently committable tasks and write them as a checklist *into the issue body*.
3. **Implement** — one task at a time, one commit per task (tagged `T<n>: … (#<issue>)`), verifying as it goes.
4. **Sync** — tick the checkbox in the issue body and mirror progress into Claude Code's TodoWrite list after each task.
5. **Wrap up** — post a summary. It does **not** push or open a PR without your confirmation.

### Resumable by design

The canonical task list lives in the issue body inside a marker block:

```
<!-- michi:plan -->
- [ ] T1: …
- [x] T2: …
<!-- /michi:plan -->
```

Checkboxes are the durable state. On a new run, the plugin parses this block, reconciles it against the git branch and commit log, and resumes at the first unchecked task. Nothing is re-done.

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

The command is scoped via `allowed-tools` to specific `gh`/`git` subcommands. It will **never** force-push, auto-merge, push, open a PR, or close the issue without your explicit go-ahead. Commits are local and one-per-task, so anything it does is easy to inspect and reverse.

## License

MIT — see [LICENSE](LICENSE).
