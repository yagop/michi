---
description: GitHub-issue-driven development — plan, implement, and sync an issue task by task. Resumable.
argument-hint: <number> [owner/repo]
allowed-tools: >-
  Bash(gh auth status:*),
  Bash(gh issue view:*),
  Bash(gh issue edit:*),
  Bash(gh issue comment:*),
  Bash(gh label list:*),
  Bash(git status:*),
  Bash(git log:*),
  Bash(git branch:*),
  Bash(git checkout:*),
  Bash(git switch:*),
  Bash(git diff:*),
  Bash(git rev-parse:*),
  Bash(git add:*),
  Bash(git commit:*),
  Read, Edit, Write, Grep, Glob, TodoWrite
---

You are running **michi**: GitHub-issue-driven development. Work issue **#$1** in this
repository, driving it to completion one small task at a time. Runs are **resumable** —
the issue body is the source of truth, so you can be stopped and re-invoked at any point.

## Arguments

- `$1` — the issue number (required). If empty, ask the user for it and stop.
- `$2` — optional `owner/repo`. When present, pass `-R $2` to **every** `gh` call.
  When absent, `gh` uses the current repo. Below, `$REPO` means "`-R $2` if given, else nothing".

## Live context

- Current branch: !`git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "(not a git repo)"`
- Working tree: !`git status --short 2>/dev/null | head -30`

## Hard rules (safety)

- **Never** `git push`, force-push, open a PR, auto-merge, or close the issue. Stop and let
  the user do those explicitly. (The tool scope already forbids them — don't try to work around it.)
- **One commit per task**, message format: `T<n>: <task summary> (#$1)`.
- Make **only the change the current task requires**. Don't batch tasks into one commit.
- The checklist in the issue body is the durable state. Keep it accurate after every task.

## 0. Preconditions

1. Confirm `gh` is authenticated: `gh auth status`. If not, tell the user to run `gh auth login` and stop.
2. Confirm you're inside a git repo (see Live context). If not, stop and say so.

## 1. Read state

Fetch the issue:

```
gh issue view $1 $REPO --json number,title,body,labels,state
```

Look in the body for the michi marker block:

```
<!-- michi:plan -->
- [ ] T1: …
- [x] T2: …
<!-- /michi:plan -->
```

- **No marker block** → this is a **fresh start**. Go to §2 (Plan).
- **Marker block present** → this is a **resume**. Go to §3 (Reconcile & resume).

## 2. Plan (fresh start only)

1. Read the issue title + body. If the repo has obvious conventions (build/test commands,
   structure), skim what you need to plan well.
2. Decompose the work into **small, independently committable tasks** — each one a coherent
   change that can be a single commit and verified on its own. Prefer 3–8 tasks; split only
   where it earns its keep.
3. Write the checklist **into the issue body**, inside a fresh marker block, preserving the
   user's original body above it:

   ```
   <existing issue body>

   <!-- michi:plan -->
   - [ ] T1: <task>
   - [ ] T2: <task>
   <!-- /michi:plan -->
   ```

   Apply it with `gh issue edit $1 $REPO --body-file -` (pipe the full new body via stdin, or
   write to a temp file and use `--body-file`). Do **not** lose existing body content.
4. Ensure a working branch exists. If you're on the default branch, create one:
   `git switch -c issue-$1` (skip if already on a suitable branch).
5. Best-effort label: if `gh label list $REPO` shows an `in-progress` label, add it with
   `gh issue edit $1 $REPO --add-label in-progress`. If it doesn't exist, continue silently —
   it's nice-to-have, not required.
6. Mirror the task list into Claude Code's TodoWrite so the user sees live progress.

Then proceed to §4 (Implement).

## 3. Reconcile & resume

1. Parse the checked/unchecked state from the marker block.
2. Cross-check against reality so you never redo finished work:
   - `git log --oneline --grep "(#$1)"` — commits already made for this issue (look for `T<n>:`).
   - Treat a task as done if its checkbox is `[x]` **or** a matching `T<n>:` commit exists.
   - If they disagree (e.g. a commit exists but the box is unchecked), trust the git history,
     tick the box, and note the reconciliation.
3. Mirror current state into TodoWrite.
4. Resume at the **first genuinely unchecked** task. Proceed to §4.

## 4. Implement (loop, one task at a time)

For each unchecked task `T<n>`, in order:

1. Mark it `in_progress` in TodoWrite.
2. Implement exactly that task. Read the files you need; make the change.
3. **Verify**: run the project's test/build/lint for the touched area if one exists, or
   otherwise sanity-check the change. Don't move on if it's broken.
4. Commit just this task's changes:
   ```
   git add <the files for this task>
   git commit -m "T<n>: <task summary> (#$1)"
   ```
5. **Sync**: flip the checkbox to `[x]` for `T<n>` inside the issue's marker block via
   `gh issue edit $1 $REPO --body-file -` (re-read the current body first so you don't clobber
   concurrent edits), and mark the task `completed` in TodoWrite.
6. Move to the next unchecked task.

If a task turns out to be wrong-sized or blocked, update the plan in the issue body
(adjust/split the checklist inside the marker block) and explain why, then continue.

## 5. Wrap up

When every task is checked:

1. Post a short summary comment on the issue: `gh issue comment $1 $REPO --body "<summary>"` —
   list the tasks done and the commits (`T<n>:` … ) made.
2. Stop. Remind the user the commits are **local**; you have not pushed, opened a PR, or
   closed the issue, and will only do so on their explicit go-ahead.
