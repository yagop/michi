# 😺 michi

A Claude Code [Agent Skill](https://code.claude.com/docs/en/skills) for **GitHub-issue-driven development**. Write an issue, run `/issue <number>`, and michi plans the work, implements it task by task in a **draft PR**, and watches it through to merge — keeping everything in sync as it goes. Stop anytime — runs are **resumable**. 🐾

## ✨ What it does

`/issue <number>` runs four steps:

1. **📖 Read state** — find any existing draft PR for the `issue-<n>` branch and decide: fresh start or resume. Your issue body is never touched.
2. **📝 Plan** — break the issue into small, independently committable tasks, set up an isolated **git worktree** for the `issue-<n>` branch (your current checkout is never touched), open a draft PR (`Closes #<issue>`) with the checklist in its body, and drop a pointer comment on the issue.
3. **🛠️ Implement** — one task at a time, looping **execute → verify → correct** until green (never commits broken code), then commit → **push** the `issue-<n>` branch → tick the task.
4. **🚦 Wrap up & watch** — wait for CI and push bounded fix commits until green, mark the PR **ready for review**, then **watch it until you merge** — turning new review comments and CI failures into tasks, and replying to reviewers (never resolving their threads). It still never merges, force-pushes, or closes the issue.

## 🔁 Resumable by design

The task list lives in **one place** — a marker block in the draft PR body — so there's nothing to keep in sync:

```
<!-- michi:plan -->
- [ ] T1 <!--m:a1b2--> …
- [x] T2 <!--m:c3d4--> …
<!-- /michi:plan -->
```

**Git is the source of truth for what's done** — a task is complete iff a commit carries its id (`Michi-Task: <id>` trailer); the checkboxes are just a cache. On each run michi re-derives done-ness from the commit log, refreshes the checkboxes, and resumes at the first unfinished task. Nothing is redone, and a failed sync self-heals next run.

## 📋 Requirements

- [`gh`](https://cli.github.com/) installed and authenticated (`gh auth status`).
- `git` **≥ 2.34**, run from inside the target repo.

## 📦 Install

michi is distributed as a Claude Code **plugin** (the packaging) and surfaces a single **Agent Skill** — `michi:issue` — once installed. In Claude Code:

```
/plugin marketplace add yagop/michi
/plugin install michi@michi
```

(Swap `yagop/michi` for a local path to try a clone.)

## 🚀 Usage

```
/issue 123                 # work issue #123 in the current repo
/issue 123 owner/repo      # target a specific repo
```

Run it again on the same issue anytime to resume.

## 🧩 Agent Skill

The plugin is just packaging; the capability is a single [Agent Skill](https://code.claude.com/docs/en/skills), **`michi:issue`**, invoked as `/issue <number>`. It's deliberately **user-invoked** — a side-effecting command (it pushes branches and opens PRs) shouldn't auto-fire.

## 🔒 Safety

The command is scoped via `allowed-tools` to specific `gh`/`git` subcommands, works in a dedicated `issue-<n>` **git worktree** (your current checkout — branch, staged changes, untracked files — is never touched; no branch switch, no stash), and makes one commit per task — easy to inspect. After marking it ready, michi **watches the PR until you merge it**, turning review comments and CI failures into tasks and replying to reviewers (never resolving threads or dismissing reviews). It will **never** merge, force-push, push your default branch, or close the issue; those stay yours. (The issue closes when *you* merge the PR, via `Closes #<issue>`.) The worktree stays in place for resumability; removing it (`git worktree remove`) is yours, like merging.

> **Note:** earlier versions stayed local and never pushed; michi now pushes to a non-default branch and opens a draft PR automatically.

## 📄 License

MIT — see [LICENSE](LICENSE).
